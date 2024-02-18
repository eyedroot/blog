---
layout: post
title:  "클러스터 환경, Duplicate Entry Error 동시성 문제"
date:   2024-02-14 13:00:00 +0900
categories: Laravel
---
## 개요 및 이슈

클러스터 환경에서 서비스를 운영하다 보면, `SQLSTATE[23000]: Integrity constraint violation: 1062 Duplicate entry '4240834-UKEY' for key 'database.table_name'(Connection: mysql, SQL ..)`와 같은 오류를 종종 마주칠 수 있습니다. 이유는 간단하다 동시에 많은 요청이 처리될 때, 특히 여러개의 Pod(클러스터 환경)에서 동시에 같은 데이터를 삽입(`INSERT`)하려고 할 때 발생할 수 있습니다. 기본적인 애플리케이션 내에서의 로직이 중복 삽입을 방지하기 위한 충분한 체크가 되지 않았거나 혹은 작성돼 있다고 하더라도 동시성 문제로 인해 충분히 발생할 수 있습니다.

## 해결 방법

1. 오류 내용에서 알 수 있듯이 데이터베이스에 유니크 키 제약 조건을 설정합니다. 하지만 이것만으로는 애플리케이션의 동시성 문제를 해결할 수 없습니다.
2. 애플리케이션 코드내에서 중복체크 로직을 작성합니다. 라라벨로 예를들면 `Eloquent`의 `firstOrCreate` 또는 `updateOrCreate` 메소드를 사용하여, 중복 삽입을 시도하기 전에 해당 데이터가 존재하는지 체크할 수 있습니다. 이 방법은 단일 노드에서는 효과적이지만, 클러스터 환경에서는 여전히 **경쟁 조건**에 의해 문제가 발생할 수 있습니다.
3. 분산 락(Distributed Locking): 라라벨에서 제공하는 캐시 기반의 분산 락 매커니즘을 사용하여, 동시에 같은 작업을 수행하려는 요청들 사이에서 경쟁 조건을 방지할 수 있습니다. 예를 들어 가장 일반적은 Redis를 이용하여 락을 구현할 수 있습니다.

## 분산 락(Distributed Locking) 예시

```php
use Illuminate\Support\Facades\Cache;

$lock = Cache::lock('test_tracker_chance', 10); // 10초 동안 락을 얻으려고 시도

if ($lock->get()) {
    // 중복 체크 로직 실행
    $exists = DB::table('test_trackers')
                ->where('member_id', $memberId)
                ->where('type', $type)
                ->exists();

    if (!$exists) {
        // 데이터가 존재하지 않으면 삽입
        DB::table('test_trackers')->insert([
            'member_id' => $memberId,
            'type' => $type,
            'value' => $value,
            'updated_at' => now(),
            'created_at' => now(),
        ]);
    }

    $lock->release(); // 작업 완료 후 락 해제
}
```

이 코드는 Redis 등의 분산 캐시를 사용하여 락을 구현한 예시입니다. `Cache::lock` 메소드를 사용하여 락을 얻고, 해당 락을 얻는데 성공하면 중복 데이터 삽입을 시조하기 전에 데이터가 이미 존재하는지 체크합니다. 모든 작업이 완료되면 `release` 메소드를 호출하여 락을 해제합니다.

## 주의점

위 코드에서는 다른 문제로 변질될 수 있습니다. 

1. 성능 영향: 락을 사용하면 리소스에 대한 접근이 순차적으로 이루어지기 때문에 고성능이 요구되는 환경에서는 요청 처리 시간에 영향을 줄 수 있습니다. 따라서 락을 사용해야하는 영역과 로직을 최소화하고, 가능한 한 빠르게 락을 해제하여 다른 요청이 대기 시간 없이 진행할 수 있도록 하는 것이 중요합니다.
2. 데드락(deadlock) 방지: 락을 사용할 때는 항상 데드락(deadlock)이 발생하지 않도록 주의해야 합니다. 데드락은 두 개 이상의 프로세스가 서로 다른 프로세스가 보유한 자원의 해제를 무한히 기다리는 교착 상태를 말합니다. 라라벨의 `Cache::lock` 메소드는 락을 얻지 못했을 때 예외를 발생시키거나, 설정한 시간이 지나면 자동으로 실채 처리되므로 데드락을 방지하는데 도움이 됩니다.

## 커넥션(connection) 분리

`Cache::lock`을 사용할 때 기존 애플리케이션에서 사용하는 커넥션과 별개로 분리하는게 도움이 될 수 있습니다. `config/cache.php` 파일에서 커넥션을 새롭게 정의하고 특정 캐시 연결을 `lockConnection`으로 사용하려면 `Cache::store('yourLockConnection')->lock(...)` 형식으로 사용할 수 있습니다.

```php
'redis' => [
    'client' => 'predis',

    'options' => [
        'cluster' => 'redis',
        'prefix' => env('REDIS_PREFIX', 'laravel_database_'),
    ],

    'default' => [
        'url' => env('REDIS_URL'),
        'host' => env('REDIS_HOST', '127.0.0.1'),
        'password' => env('REDIS_PASSWORD', null),
        'port' => env('REDIS_PORT', '6379'),
        'database' => env('REDIS_DB', '0'),
    ],

    // 분산 락을 위한 별도의 연결 추가
    'lockConnection' => [
        'url' => env('REDIS_URL'),
        'host' => env('REDIS_HOST', '127.0.0.1'),
        'password' => env('REDIS_PASSWORD', null),
        'port' => env('REDIS_PORT', '6379'),
        'database' => env('REDIS_LOCK_DB', '1'), // 예를 들어, 다른 DB 인덱스를 사용
    ],
],
```

이렇게 설정한 후, `Cache::store('redis')->lock(...)` 대신 `lockConnection`을 사용하여 `Cache::store('lockConnection')->lock(...)` 형식으로 분산 락을 구현할 수 있습니다.

```php
$lock = Cache::store('lockConnection')->lock('migration_tracker_insert', 10);

if ($lock->get()) {
    // 락을 성공적으로 얻었을 때의 로직
    // ...

    $lock->release(); // 작업 완료 후 락 해제
}
```

이 방식을 사용하면, 기본 캐시 데이터와 분산 락을 위한 데이터를 분리하여 관리할 수 있으며, 이는 경쟁 상태를 최소화하는데 도움이 될 수 있습니다. 분산 락을 위한 별도의 연결을 설정함으로써, 캐시 데이터와 락 데이터 간의 상호 작용이 서로 영향을 주지 않도록 할 수 있습니다.
