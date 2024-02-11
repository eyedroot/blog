---
layout: post
title:  "ReactNative Hooks"
date:   2024-02-11 15:24:00 +0900
categories: ReactNative
---
## React Native Hook Basic

### useEffect

```Typescript
useEffect(() => {
    loadTodos();
}, []);
```

- `useEffect` 훅을 사용하여 컴포넌트가 마운트될 때(즉, 화면에 처음으로 렌더링될 때) `loadToDos()` 함수를 실행한다.
- 두 번째 인자는 의존성 배열로, 이 배열의 값에 따라 부수 효과의 실행 시점이 결정된다.
  - 예를들어 `[todos]`와 같이 상태 변수를 배열에 포함시키면, `todos` 상태가 변경될 때마다 함수가 실행된다.
  - 하지만 여기서는 사용된 빈 배열이므로 컴포넌트가 마운트될 때만 함수를 실행하고, 그 이후에는 실행되지 않도록 함으로써 초기 데이터 로딩 같은 용도로 자주 사용된다.

### useState

```Typescript
const [toDos, setToDos] = useState<ToDo>({});
```

- 위의 `toDos`와 같은 **상태 변수**는 직접적으로 변경할 수 없는 **불변(immutable)** 상태로 취급된다.
  - React에서 상태 관리는 불변성을 유지하면서 이루어져야 한다. 이는 React가 효율적으로 UI를 업데이트할 수 있도록 돕는다.
- `setToDos`는 상태를 업데이트하는 함수이다.
  - `toDos`의 값을 직접 변경하는 것이 아니라, 상태를 업데이트할 필요가 있을 때는 항상 `setToDos` 함수를 사용하여 새로운 상태 값을 전달해야 한다. (React의 상태 불변성 원칙)

```Typescript
// 잘못된 예: 상태를 직접 변경
toDos.key = newValue;

// 올바른 예: 새로운 객체를 생성하여 상태 업데이트
setToDos({ key: newValue });
```
