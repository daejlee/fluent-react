# 성능 최적화

React 애플리케이션의 성능을 향상시키는 메모화(Memoization) 기법에 대해 알아봅니다.

## React.memo를 사용한 메모화

`React.memo`는 컴포넌트를 메모화하여 불필요한 리렌더링을 방지하는 고차 컴포넌트(HOC)입니다.

### 기본 사용법

```jsx
const MyComponent = React.memo(function MyComponent({ name, age }) {
  console.log("렌더링!");
  return (
    <div>
      {name}님, {age}세
    </div>
  );
});
```

**동작 원리**: props가 변경되지 않으면 이전에 렌더링된 결과를 재사용합니다.

### 리렌더링되는 메모화 컴포넌트

`React.memo`를 사용해도 리렌더링이 발생하는 경우가 있습니다. **props의 타입**에 따라 달라집니다.

#### 스칼라 타입 (Scalar Types)

스칼라 타입은 **값으로 비교**되므로 메모화가 잘 작동합니다.

```jsx
const Child = React.memo(({ count, name, isActive }) => {
  console.log("Child 렌더링!");
  return (
    <div>
      {name}: {count}
    </div>
  );
});

function Parent() {
  const [count, setCount] = useState(0);

  return (
    <div>
      {/* count가 같으면 리렌더링 안 됨! ✅ */}
      <Child count={5} name="Kenny" isActive={true} />
      <button onClick={() => setCount(count + 1)}>증가</button>
    </div>
  );
}
```

**스칼라 타입 종류**:

- `number`: 1, 2, 3.14
- `string`: "hello", "world"
- `boolean`: true, false
- `null`, `undefined`

✅ **메모화 효과**: 값이 같으면 리렌더링 안 됨!

#### 스칼라가 아닌 타입 (Non-Scalar Types)

객체, 배열, 함수는 **참조로 비교**되므로 매번 새로 생성되면 리렌더링됩니다.

```jsx
const Child = React.memo(({ user, items, onClick }) => {
  console.log("Child 렌더링!");
  return <div>{user.name}</div>;
});

function Parent() {
  const [count, setCount] = useState(0);

  return (
    <div>
      {/* ❌ 매번 새 객체/배열/함수 생성 → 항상 리렌더링! */}
      <Child
        user={{ name: "Kenny" }} // 새 객체
        items={[1, 2, 3]} // 새 배열
        onClick={() => console.log("hi")} // 새 함수
      />
      <button onClick={() => setCount(count + 1)}>증가</button>
    </div>
  );
}
```

**문제**: Parent가 리렌더링될 때마다 새로운 객체/배열/함수가 생성됩니다.

```
리렌더링 전: user = { name: "Kenny" }  // 메모리 주소: 0x001
리렌더링 후: user = { name: "Kenny" }  // 메모리 주소: 0x002 (다른 객체!)

React: "주소가 다르네? 변경됐구나!" → 리렌더링 😢
```

### 해결 방법

#### 1. 객체를 컴포넌트 외부로 이동

```jsx
// 컴포넌트 밖에 선언 → 항상 같은 참조
const USER = { name: "Milk" };
const ITEMS = [1, 2, 3];

function Parent() {
  return <Child user={USER} items={ITEMS} />; // ✅ 메모화 작동!
}
```

#### 2. useMemo와 useCallback 사용

```jsx
function Parent() {
  const [count, setCount] = useState(0);

  // 같은 참조 유지
  const user = useMemo(() => ({ name: "Milk" }), []);
  const items = useMemo(() => [1, 2, 3], []);
  const onClick = useCallback(() => console.log("hi"), []);

  return (
    <Child user={user} items={items} onClick={onClick} /> // ✅ 작동!
  );
}
```

### React.memo의 커스텀 비교 함수

두 번째 인자로 비교 함수를 제공할 수 있습니다.

```jsx
const Child = React.memo(
  ({ user }) => <div>{user.name}</div>,
  (prevProps, nextProps) => {
    // true를 반환하면 리렌더링 안 함
    // false를 반환하면 리렌더링
    return prevProps.user.id === nextProps.user.id;
  },
);
```

**주의**: 이 함수는 `shouldComponentUpdate`와 반대로 동작합니다!

- `true`: 리렌더링 **안 함** (같다고 판단)
- `false`: 리렌더링 **함** (다르다고 판단)

## 5.2 useMemo를 활용한 메모화

`useMemo`는 **값을 메모화**하여 불필요한 계산을 방지합니다.

### 기본 사용법

```jsx
function ExpensiveComponent({ items }) {
  // items가 변경될 때만 재계산
  const total = useMemo(() => {
    console.log("계산 중...");
    return items.reduce((sum, item) => sum + item.price, 0);
  }, [items]);

  return <div>총합: {total}</div>;
}
```

**동작 원리**:

1. 첫 렌더링: 함수 실행하고 결과 저장
2. 리렌더링: 의존성 배열(`[items]`) 확인
   - 변경됨 → 함수 재실행
   - 변경 안 됨 → 저장된 값 반환

### useMemo의 좋은 사례

#### 1. 비용이 큰 계산

```jsx
function SearchResults({ users, query }) {
  // 필터링은 비용이 큼
  const filteredUsers = useMemo(() => {
    return users.filter((user) =>
      user.name.toLowerCase().includes(query.toLowerCase()),
    );
  }, [users, query]);

  return <UserList users={filteredUsers} />;
}
```

#### 2. 자식 컴포넌트의 props로 전달되는 객체/배열

```jsx
function Parent() {
  const [count, setCount] = useState(0);

  // 같은 참조 유지 → Child 리렌더링 방지
  const config = useMemo(
    () => ({
      theme: "dark",
      fontSize: 16,
    }),
    [],
  ); // 의존성 없음 → 항상 같은 객체

  return <Child config={config} />;
}

const Child = React.memo(({ config }) => {
  return <div style={{ fontSize: config.fontSize }}>내용</div>;
});
```

### useMemo의 나쁜 사례

#### ❌ 1. 단순한 계산에 사용

```jsx
// 나쁨: 단순 계산은 useMemo 오버헤드가 더 큼
const doubled = useMemo(() => count * 2, [count]);

// 좋음: 그냥 계산
const doubled = count * 2;
```

**이유**: `useMemo`도 비용이 있습니다.

- 의존성 배열 비교
- 메모리에 값 저장
- 클로저 생성

단순 계산은 그냥 하는 게 더 빠릅니다!

#### ❌ 2. 모든 값을 메모화

```jsx
// 나쁨: 과도한 메모화
function Component({ data }) {
  const a = useMemo(() => data.a, [data.a]);
  const b = useMemo(() => data.b, [data.b]);
  const c = useMemo(() => data.c, [data.c]);
  const d = useMemo(() => a + b, [a, b]);
  const e = useMemo(() => c * 2, [c]);
  // ... 너무 많다!
}
```

**문제점**:

- 코드 가독성 저하
- 불필요한 메모리 사용
- 오히려 성능 저하 가능

**원칙**: **성능 문제가 실제로 발생할 때만** 사용하세요!

#### ❌ 3. 의존성 배열이 계속 변경됨

```jsx
// 나쁨: items가 매번 바뀌면 의미 없음
function Component() {
  const items = [1, 2, 3]; // 매번 새 배열!

  const total = useMemo(() => {
    return items.reduce((sum, n) => sum + n, 0);
  }, [items]); // items가 매번 바뀌므로 항상 재계산
}
```

#### ❌ 4. 의존성 배열 누락

```jsx
// 나쁨: query가 의존성에 없음
const filtered = useMemo(() => {
  return users.filter((u) => u.name.includes(query));
}, [users]); // query 변경 시 업데이트 안 됨! 🐛
```

**해결**: ESLint의 `exhaustive-deps` 규칙을 켜세요!

### useMemo 사용 가이드라인

✅ **사용해야 할 때**:

- 비용이 큰 계산 (복잡한 필터링, 정렬, 변환)
- `React.memo`된 자식에게 전달하는 객체/배열
- 의존성이 자주 변경되지 않는 경우

❌ **사용하지 말아야 할 때**:

- 단순한 계산 (덧셈, 곱셈, 문자열 연결 등)
- 의존성이 매번 변경되는 경우
- 모든 값을 "혹시 몰라서" 메모화

### 리액트 컴파일러

React 19부터 도입될 **React Compiler**는 자동으로 메모화를 처리합니다!

#### 기존 방식의 문제

```jsx
// 개발자가 수동으로 최적화
function TodoList({ todos, filter }) {
  const filteredTodos = useMemo(
    () => todos.filter((t) => t.category === filter),
    [todos, filter],
  );

  const handleClick = useCallback((id) => {
    // ...
  }, []);

  return <List items={filteredTodos} onClick={handleClick} />;
}
```

**문제점**:

- 어디에 `useMemo`/`useCallback`을 써야 할지 고민
- 의존성 배열 관리 부담
- 실수로 누락하면 버그 발생

#### React Compiler의 해결책

```jsx
// 컴파일러가 자동으로 최적화!
function TodoList({ todos, filter }) {
  const filteredTodos = todos.filter((t) => t.category === filter);

  const handleClick = (id) => {
    // ...
  };

  return <List items={filteredTodos} onClick={handleClick} />;
}

// ↓ 컴파일러가 자동으로 변환 ↓

function TodoList({ todos, filter }) {
  const filteredTodos = useMemo(
    () => todos.filter((t) => t.category === filter),
    [todos, filter],
  );

  const handleClick = useCallback((id) => {
    // ...
  }, []);

  return <List items={filteredTodos} onClick={handleClick} />;
}
```

#### React Compiler의 장점

✅ **자동 최적화**

- 필요한 곳에 자동으로 메모화 적용
- 의존성 자동 추적
- 실수 방지

✅ **코드 간결화**

- `useMemo`, `useCallback` 제거
- 의존성 배열 관리 불필요
- 가독성 향상

✅ **더 나은 성능**

- 컴파일러가 최적의 위치 판단
- 사람보다 더 정확한 최적화

#### 사용 방법 (React 19+)

```bash
npm install react-compiler
```

```js
// babel.config.js
module.exports = {
  plugins: [
    [
      "babel-plugin-react-compiler",
      {
        runtimeModule: "react-compiler-runtime",
      },
    ],
  ],
};
```

#### 주의사항

⚠️ **React의 규칙을 따라야 함**:

- Props와 state는 불변으로 다루기
- 순수 함수 작성하기
- React의 규칙 위반 시 컴파일러가 최적화 못 함

```jsx
// ❌ 나쁨: 직접 변경
function Bad({ items }) {
  items.push(newItem); // 돌연변이!
  return <List items={items} />;
}

// ✅ 좋음: 새 배열 생성
function Good({ items }) {
  const newItems = [...items, newItem];
  return <List items={newItems} />;
}
```
