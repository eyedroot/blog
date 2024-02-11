---
layout: post
title:  "spatie/laravel-data 데이터 객체 복제"
date:   2024-01-30 13:12:00 +0900
categories: Laravel
---
오픈소스 개발 회사 **spatie**에서 개발된 [laravel-data](https://spatie.be/docs/laravel-data/v3/introduction) 패키지를 소개합니다. 공식 홈페이지에서는 다음과 같이 **laravel-data**를 소개하고 있습니다.

> Powerful data objects for laravel
> 
> ... can be used in various ways. Using this package you only need to describe your data once:
>
> - instead of a form request, you can use a data object
> - instead of an API transformer, you can use a data object
> - instead of manually writing a typescript definition, you can use... 🥁 a data object

소개에서와 같이 해당 패키지를 통해 `Request` 객체를 만들고 **validate**도 같이 체크할 수 있고, **transformer**를 적용시킴으로써 쉽고 직관적으로 데이터를 변환할 수 있습니다.(여기서 말하는 transformer에 간단한 예시는 객채 클래스 내에서는 CamelCase 문법을 따르지만, API로 응답할 때는 snake_case로 변환하는 것을 말합니다.) 그리고 작성된 데이터 객체를 통해 **typescript definition**을 만들어 낼 수도 있습니다. 이 모든 것들은 PHP 8의 **attributes**를 통해 실행됩니다.

## 설치

```bash
composer require spatie/laravel-data
```

## 사용법

그럼 [spatie/laravel-data](https://spatie.be/docs/laravel-data/v3/introduction)를 통해 객체를 생성하는 방법에 대해 간단히 알아보겠습니다.

```php
<?php

namespace App\Http\Controllers\Requests;

use Spatie\LaravelData\Data;

class FeedbackRequest extends Data
{
    public function __construct(
        public readonly int $id,
        public readonly int $point,
    ) {
    }
}
```

기본적인 사용법은 위와 같습니다. `Spatie\LaravelData\Data` 객체를 상속하고, 속성들(`$id`, `$point`)을 정의해 줍니다. 그리고 객체를 생성할 때는 아래와 같이  `from` static 메소드를 통해 생성할 수 있습니다.

```php
$feedback = FeedbackRequest::from([
    'id' => 1,
    'point' => 5,
]);
```

그러면 `Spatie\LaravelData\Data` 객체를 가지고 라라벨의 **Request** 객체를 만들면서 **validate**를 처리하는 예시를 이어서 보겠습니다.

### Request 객체 생성

```php
<?php

namespace App\Http\Controllers\Requests;

use Spatie\LaravelData\Attributes\MapInputName;
use Spatie\LaravelData\Attributes\Validation\In;
use Spatie\LaravelData\Attributes\Validation\IntegerType;
use Spatie\LaravelData\Attributes\Validation\Required;
use Spatie\LaravelData\Data;
use Spatie\LaravelData\Mappers\SnakeCaseMapper;

#[MapInputName(SnakeCaseMapper::class)]
class PointRequest extends Data
{
    public function __construct(
        #[Required, IntegerType]
        public readonly int $id,

        #[Required, In([1, 2, 3])]
        public readonly int $point,
    ) {
    }
}
```

위의 **Request** 객체는 `#[MapInputName(SnakeCaseMapper::class)]` snake_case를 통해 파라메터를 매칭시키고, `#[Required, IntegerType]`을 통해 validate를 수행합니다. 모든 것들이 PHP 8의 **attributes**를 통해 실행됩니다. 정말 직관적이지 않나요? 위와 같이 작성된 클래스는 컨트롤러에서 다음과 같이 사용될 수 있습니다.

```php
<?php

namespace App\Http\Controllers;

use App\Http\Controllers\Requests\PointRequest;

readonly class PointController
{
    public function incresePoint(PointRequest $request)
    {
        $request->id;
        $request->point;
    }
}
```

컨트롤러 레이어까지 들어오기 전에 이미 **validate**가 수행됩니다. 따라서 **validate**가 통과되지 못 하면 라라벨은 **422 Unprocessable Entity**를 반환합니다.

### DTO(Data Transfer Object) 객체 생성

DTO 객체로 사용할 수도 있습니다. 사용방법은 비슷합니다. 장점은 중첩된 클래스 형태를 가지고 있다면 `Data::from` 메소드를 통해 쉽게 생성할 수 있습니다. 아래의 예시를 보겠습니다.

```php
<?php

namespace App\Services\Product\Dto;

use Carbon\Carbon;
use Spatie\LaravelData\Attributes\DataCollectionOf;
use Spatie\LaravelData\Attributes\MapOutputName;
use Spatie\LaravelData\Data;
use Spatie\LaravelData\DataCollection;
use Spatie\LaravelData\Mappers\SnakeCaseMapper;

#[MapOutputName(SnakeCaseMapper::class)]
class BagDto extends Data
{
    public function __construct(
        public readonly int $count,

        #[DataCollectionOf(BagItemDto::class)]
        public readonly DataCollection $items,
    ) {
    }
}
```

```php
<?php

namespace App\Services\Product\Dto;

use Spatie\LaravelData\Attributes\MapOutputName;
use Spatie\LaravelData\Data;
use Spatie\LaravelData\Mappers\SnakeCaseMapper;

#[MapOutputName(SnakeCaseMapper::class)]
class BagItemDto extends Data
{
    public function __construct(
        public readonly int $id,
        public readonly string $itemName,
    ) {
    }
}
```

`BagDto`라는 가방은 갯수와 각 아이템들의 리스트를 가지고 있습니다. 이를 객체로 생성하려면 아래와 같이 `items`에 배열 형태로 넘기거나 라라벨의 `Collection`에 담아 넘길 수 있습니다.

```php
$items = [
    ['id' => 1, 'itemName' => 'item1'],
    ['id' => 2, 'itemName' => 'item2'],
];

$bag = BagDto::from([
    'count' => 2,
    'items' => $items,
]);

$bag = BagDto::from([
    'count' => 2,
    'items' => collect($items),
]);
```

또 한, `#[MapOutputName(SnakeCaseMapper::class)]`를 통해 snake_case로 변환된 데이터를 반환할 수 있습니다. 실제 API 상에서 응답된 데이터 포맷은 다음과 같습니다. `$itemName` 속성이 snake_case 포맷으로 변환되어 출력됩니다.

```php
{
    "count": 2,
    "items": [
        {
            "id": 1,
            "item_name": "item1"
        },
        {
            "id": 2,
            "item_name": "item2"
        }
    ]
}
```

위에 예시들은 라라벨의 기본적인 기능을 통해 이미 구현 가능한 부분이지만 `spatie/laravel-data` 패키지를 통해 더 직관적이고 조금 더 클래스 친화적으로 코드를 유지할 수 있습니다. 또한, `spatie/laravel-data` 패키지를 통해 **typescript definition**을 생성할 수 있습니다. 

### Typescript Definition 생성

이미 우리는 DTO 예시에서 `BagDto`와 `BagItemDto`를 만들었습니다. 여기에 추가적인 **Attributes** 작성을 통해 쉽게 **typescript definition**을 생성할 수 있습니다. 이를 실행하면 추가적인 패키지가 요구됩니다.

```bash
composer require spatie/laravel-typescript-transformer
```

해당 패키지의 자세한 사용법은 추후 다른 게시글을 통해 자세히 더 살펴보도록 하겠습니다. 공식 문서를 통해 이미 자세한 내용이 설명되어 있으므로 공식 문서를 참고해도 됩니다. [spatie/laravel-typescript-transformer](https://spatie.be/docs/typescript-transformer/v2/introduction)

해당 패키지의 생산성은 DTO를 통해 잘 구성된 API 리소스들을 클라이언트 개발자와 함께 협업하기에 용이합니다. 어디까지나 선택사항입니다.

### 심화편: 객체 복제하기

생성된 DTO 객체를 일부 값만 변경하고 싶을 때가 있습니다. 여기서는 `BadItemDto`의 `$id` 값은 유지하면서 `$itemName`만 변경하고 싶을 때를 가정해 보겠습니다.

```php
#[MapOutputName(SnakeCaseMapper::class)]
class BagItemDto extends Data
{
    public function __construct(
        public readonly int $id,
        public readonly string $itemName,
    ) {
    }

    public function withItemName(string $itemName): self
    {
        // array 형태로 변환
        $bagItem = $this->transform(transformValues: false, mapPropertyNames: false);

        $bagItem['itemName'] = $itemName;

        return self::from($bagItem);
    }
}
```

해당 예제에서는 `#[MapOutputName(SnakeCaseMapper::class)]`를 통해 snake_case로 변환된 데이터를 반환하고 있습니다. 해당 부분이 없다면 간편하게 `$this->with()` 메소드를 통해 원본 데이터를 연관 배열로 리턴 받을 수 있지만, snake_case로 변환 규칙이 작성돼 있으므로 `$this->transform(transformValues: false, mapPropertyNames: false)`를 통해 연관 배열로 변환합니다. 그리고 연관 배열로 변환된 데이터를 통해 원하는 값을 변경하고 `self::from($bagItem)`를 통해 객체를 생성합니다.

```php
$bagItem = BagItemDto::from([
    'id' => 1,
    'itemName' => 'item1',
]);

$bagItem->withItemName('item2');
```

## 마치며

어디까지나 라이브러리를 채택하고 사용하고는 선택사항입니다. 라라벨을 경험하신지 얼마 되지 않으셨으면 라라벨 본래의 기능을 한 번 접하시고 라이브러리를 사용하시길 권장합니다. 그리고 위의 작성된 방법과 비교하고 어떤 부분이 우리 팀에 더 적합한 방법이 될지 고민해 보시길 바랍니다.