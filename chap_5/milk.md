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

## 5.3 지연 로딩

지연 로딩(Lazy Loading)은 필요한 시점에 컴포넌트나 리소스를 불러와 초기 로드 시간을 단축하는 기법입니다.

### React.lazy와 Suspense 사용법 (코드 스플리팅)

```jsx
import React, { Suspense } from "react";
const LazyComponent = React.lazy(() => import("./LazyComponent"));
function App() {
  return (
    <Suspense fallback={<div>로딩 중...</div>}>
      <LazyComponent />
    </Suspense>
  );
}
```

LazyComponent가 20MB 정도의 무거운 번들의 컴포넌트라면 매우 효과적으로 초기 로드 시간을 줄일 수 있습니다.

### next.js에서는 어떻게 사용할까

Next.js에서는 `next/dynamic`을 사용하여 지연 로딩을 구현합니다.

```jsx
import dynamic from "next/dynamic";
const DynamicComponent = dynamic(() => import("./DynamicComponent"), {
  loading: () => <p>로딩 중...</p>,
});
function App() {
  return <DynamicComponent />;
}
```

next.js 에서는 굳이 React.lazy와 Suspense를 사용할 필요가 없습니다. 왜냐하면 dynamic import자체가 내부적으로 React.lazy, Suspense를 사용하기 때문입니다. 추가적으로 서버사이드 렌더링과 클라이언트사이드 렌더링을 고려하기 때문에 hydration 문제도 자동으로 해결해줍니다.

그리고 App Router부터는 서버컴포넌트와 클라이언트컴포넌트 경계가 명확해졌기 때문에 따로 지연로딩을 신경쓸 필요가 많이 줄어들었습니다.

## 5.4 useReducer vs useState

결론부터 말하면 useState는 useReducer의 특수한 형태입니다.

useReducer는 복잡한 상태 로직을 관리할 때 유용하며, 여러 하위 값이 있는 경우에도 적합합니다. 반면에 useState는 간단한 상태를 관리할 때 더 간편합니다.

```jsx
// useState 예제
const [count, setCount] = useState(0);

// useReducer 예제
const initialState = {
  name: "Milk",
  age: 25,
  workDays: ["Monday", "Tuesday", "Wednesday"],
};
function reducer(state, action) {
  switch (action.type) {
    case "incrementAge":
      return { ...state, age: state.age + 1 };
    case "addWorkDay":
      return { ...state, workDays: [...state.workDays, action.day] };
    default:
      return state;
  }
}
```

useReducer의 장점은 다음과 같습니다:

- 상태 업데이트 로직이 컴포넌트 외부에 정의되어 가독성이 향상됩니다.
- 복잡한 상태 전환을 명확하게 관리할 수 있습니다.
- 여러 상태 값을 하나의 상태 객체로 묶어 관리할 수 있습니다.

추가적으로 immer 라이브러리를 사용하면 불변성을 신경쓰지 않고도 상태를 쉽게 업데이트할 수 있습니다.

```jsx
import produce from "immer";
function reducer(state, action) {
  switch (action.type) {
    case "incrementAge":
      return produce(state, (draft) => {
        draft.age += 1;
      });
    case "addWorkDay":
      return produce(state, (draft) => {
        draft.workDays.push(action.day);
      });
    default:
      return state;
  }
}
```

immer를 사용하면 `Object.assign`이나 스프레드 연산자를 사용하지 않고도 상태를 쉽게 변경할 수 있어 코드가 더 간결해집니다.

하지만 제 개인적으론 외부라이브러리를 사용하지않고 순수하게 `Object.assign`이나 스프레드 연산자를 사용하는 것을 선호합니다. 외부라이브러리를 사용하면 프로젝트의 복잡도가 올라가고 의존성이 늘어나기 때문입니다.

## 5.5 컴포넌트 패턴

### 프레젠테이션/컨테이너 패턴

우리가 흔히 접하는 컴포넌트 패턴 중 하나는 프레젠테이션/컨테이너 패턴입니다.

```jsx
// 프레젠테이션 컴포넌트
function UserProfile({ user, onLogout }) {
  return (
    <div>
      <h1>{user.name}</h1>
      <button onClick={onLogout}>Logout</button>
    </div>
  );
}

// 컨테이너 컴포넌트
function UserProfileContainer() {
  const [user, setUser] = useState(null);
  useEffect(() => {
    fetchUser().then(setUser);
  }, []);

  const handleLogout = () => {
    logoutUser().then(() => setUser(null));
  };

  if (!user) {
    return <div>Loading...</div>;
  }

  return <UserProfile user={user} onLogout={handleLogout} />;
}
```

프레젠테이션 컴포넌트는 UI 렌더링에 집중하고, 컨테이너 컴포넌트는 데이터 페칭과 상태 관리를 담당합니다. 이 패턴은 관심사의 분리를 통해 코드의 재사용성과 유지보수성을 향상시킵니다.

hook이 등장하면서 이 패턴을 단순히 할 수 있습니다.

```jsx
// 커스텀 훅
function useUser() {
  const [user, setUser] = useState(null);
  useEffect(() => {
    fetchUser().then(setUser);
  }, []);
  const logout = () => {
    logoutUser().then(() => setUser(null));
  };
  return { user, logout };
}

// 컴포넌트
function UserProfile() {
  const { user, logout } = useUser();

  if (!user) {
    return <div>Loading...</div>;
  }

  return (
    <div>
      <h1>{user.name}</h1>
      <button onClick={logout}>Logout</button>
    </div>
  );
}
```

### 고차 컴포넌트

고차 컴포넌트(Higher-Order Component, HOC)는 컴포넌트를 인자로 받아 새로운 컴포넌트를 반환하는 함수입니다. 주로 공통 로직을 재사용할 때 사용됩니다.

```jsx
function withLogging(WrappedComponent) {
  return function LoggedComponent(props) {
    useEffect(() => {
      console.log("컴포넌트가 마운트되었습니다.");
      return () => {
        console.log("컴포넌트가 언마운트되었습니다.");
      };
    }, []);

    return <WrappedComponent {...props} />;
  };
}
```

우리가 자주쓰는 React.memo, React.forwardRef도 고차 컴포넌트의 일종입니다.

### 고차 컴포넌트 합성

고차 컴포넌트를 여러 개 합성하여 사용할 수도 있습니다.

```jsx
const enhance = compose(withLogging, withErrorBoundary, withTheme);

const EnhancedComponent = enhance(MyComponent);
```

compose 함수는 React에서 제공하는 것이 아니라 lodash나 redux에서 제공하는 유틸리티 함수입니다. 직접 구현할 수도 있습니다.

```jsx
function compose(...funcs) {
  return function (Component) {
    return funcs.reduceRight((acc, fn) => fn(acc), Component);
  };
}
```

### 렌더 프롭

렌더 프롭(Render Prop)은 컴포넌트의 자식으로 함수를 전달하여 동적으로 UI를 렌더링하는 패턴입니다.

```jsx
function DataFetcher({ url, children }) {
  const [data, setData] = useState(null);
  useEffect(() => {
    fetch(url)
      .then((res) => res.json())
      .then(setData);
  }, [url]);

  return children(data);
}
```

```jsx
<DataFetcher url="/api/data">
  {(data) => (data ? <div>데이터: {data.value}</div> : <div>로딩 중...</div>)}
</DataFetcher>
```

상위 부모 컴포넌트에서 렌더로직을 정의하고, 하위 컴포넌트에서는 비즈니스 로직을 담당하게 되어 관심사의 분리가 가능합니다.

보통 리스트나 테이블 컴포넌트 에서 자주 사용됩니다.

왜 그러냐면 리스트에서 각 아이템이 다양하거나 복잡한 UI를 가질 수 있기 때문입니다.

### 제어 프롭

제어 프롭(Controlled Prop)은 컴포넌트의 상태를 부모 컴포넌트가 제어하는 패턴입니다. 주로 폼 요소에서 사용됩니다.

비제어 컴포넌트와 제어 컴포넌트로 구분합니다.

```jsx
// 제어 컴포넌트
function ControlledInput({ value, onChange }) {
  return <input value={value} onChange={onChange} />;
}

// 비제어 컴포넌트
function UncontrolledInput() {
  const inputRef = useRef();
  const handleSubmit = () => {
    alert(inputRef.current.value);
  };
  return <input ref={inputRef} />;
}
```

Shadcn UI에서는 제어 컴포넌트와 비제어 컴포넌트를 모두 지원하는 컴포넌트를 제공합니다. 예를 들어, `Input` 컴포넌트는 `value`와 `onChange` props를 통해 제어 컴포넌트로 사용할 수 있고, `defaultValue` props를 통해 비제어 컴포넌트로 사용할 수도 있습니다.

MUI, Ant Design 등 다른 UI 라이브러리들도 비슷한 방식을 채택하고 있습니다.

### 프롭 컬렉션

이거는 솔직히 처음들어보는 용어라고 생각하실 겁니다. 그러나 예시를 보면 "아 그거구나" 하실겁니다.

프롭을 객체로 정의해 스프레드 연산자로 전달하는 패턴입니다.

```jsx
function Button({ style, size, disabled, onClick, children }) {
  return (
    <button
      style={style}
      className={`btn-${size}`}
      disabled={disabled}
      onClick={onClick}
    >
      {children}
    </button>
  );
}
```

```jsx
const buttonProps = {
  style: { backgroundColor: "blue" },
  size: "large",
  disabled: false,
  onClick: () => alert("Clicked!"),
};

<Button {...buttonProps}>Click Me</Button>;
```

### 컴파운드 컴포넌트

복합 컴포넌트(Compound Component)는 여러 컴포넌트를 조합하여 하나의 기능을 구현하는 패턴입니다. 주로 UI 라이브러리에서 자주 사용됩니다.

```jsx
function Tabs({ children }) {
  const [activeIndex, setActiveIndex] = useState(0);

  return (
    <div>
      <div className="tab-headers">
        {React.Children.map(children, (child, index) => (
          <button onClick={() => setActiveIndex(index)}>
            {child.props.title}
          </button>
        ))}
      </div>
      <div className="tab-content">
        {React.Children.toArray(children)[activeIndex]}
      </div>
    </div>
  );
}
```

```jsx
<Tabs>
  <Tab title="Tab 1">Content of Tab 1</Tab>
  <Tab title="Tab 2">Content of Tab 2</Tab>
  <Tab title="Tab 3">Content of Tab 3</Tab>
</Tabs>
```

저는 Shadcn UI를 선호하는 편이라 이 패턴을 자주 접합니다.

### 상태 리듀서

상태 리듀서(State Reducer)는 컴포넌트의 상태 업데이트 로직을 외부에서 제어할 수 있도록 하는 패턴입니다. 주로 라이브러리나 프레임워크에서 사용됩니다.

```jsx
function Counter({ initialCount = 0, stateReducer }) {
  const [count, setCount] = useState(initialCount);
  const increment = () => {
    const newState = stateReducer
      ? stateReducer(count + 1, { type: "increment" })
      : count + 1;
    setCount(newState);
  };
  return (
    <div>
      <p>Count: {count}</p>
      <button onClick={increment}>Increment</button>
    </div>
  );
}
```

```jsx
<Counter
  initialCount={0}
  stateReducer={(nextState, action) => {
    if (nextState > 10) {
      return 10; // 최대값 제한
    }
    return nextState;
  }}
/>
```

그러나 레거시한 라이브러리에서나 사용하는 패턴이지, 요즘은 잘 사용하지 않는 추세라고 생각합니다.
