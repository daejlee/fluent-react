React의 메모이제이션(Memoization)에 대해 알아본다.
불필요한 리렌더링과 계산을 방지하여 성능을 최적화하는 기법이다.

# 5.1 React.memo

`React.memo`는 컴포넌트를 메모이제이션하는 고차 컴포넌트(HOC)다. props가 변경되지 않으면 리렌더링을 건너뛴다.

## 기본 사용법

```jsx
const MyComponent = React.memo(function MyComponent({ name, age }) {
  console.log('렌더링됨!');
  return <div>{name}은 {age}살입니다.</div>;
});
```

부모 컴포넌트가 리렌더링되어도, `name`과 `age`가 이전과 같다면 MyComponent는 렌더링되지 않는다.

## 언제 사용해야 할까?

**사용하면 좋은 경우:**
- 동일한 props로 자주 리렌더링되는 컴포넌트
- 렌더링 비용이 큰 컴포넌트 (복잡한 UI, 많은 자식 요소)
- 리스트의 각 아이템 컴포넌트

**사용하지 않아도 되는 경우:**
- props가 거의 항상 변경되는 컴포넌트
- 렌더링 비용이 적은 단순한 컴포넌트
- 이미 충분히 빠른 컴포넌트

```jsx
// 좋은 예: 리스트 아이템
const TodoItem = React.memo(function TodoItem({ todo, onToggle }) {
  return (
    <li onClick={() => onToggle(todo.id)}>
      {todo.text}
    </li>
  );
});

// 1000개의 아이템 중 하나만 변경되어도,
// 나머지 999개는 리렌더링되지 않음
function TodoList({ todos, onToggle }) {
  return (
    <ul>
      {todos.map(todo => (
        <TodoItem key={todo.id} todo={todo} onToggle={onToggle} />
      ))}
    </ul>
  );
}
```

## props 비교 방식

기본적으로 `React.memo`는 **얕은 비교(shallow comparison)**를 사용한다.

```jsx
// 얕은 비교란?
// 객체의 최상위 속성만 비교한다는 뜻

// 원시값: 값 자체를 비교
"hello" === "hello"  // true
42 === 42            // true

// 객체/배열: 참조(reference)를 비교
{ name: "Kim" } === { name: "Kim" }  // false (다른 객체)
const obj = { name: "Kim" };
obj === obj  // true (같은 참조)
```

**주의:** 부모에서 매번 새 객체/배열/함수를 props로 전달하면 memo가 무용지물이 된다.

```jsx
// ❌ 잘못된 예: 매번 새 객체 생성
function Parent() {
  return (
    <MemoizedChild
      user={{ name: 'Kim' }}  // 매번 새 객체!
      onClick={() => {}}       // 매번 새 함수!
    />
  );
}

// ✅ 올바른 예: 참조 유지
function Parent() {
  const user = useMemo(() => ({ name: 'Kim' }), []);
  const handleClick = useCallback(() => {}, []);

  return (
    <MemoizedChild user={user} onClick={handleClick} />
  );
}
```

## 커스텀 비교 함수

두 번째 인자로 비교 함수를 전달할 수 있다.

```jsx
const MyComponent = React.memo(
  function MyComponent({ user, timestamp }) {
    return <div>{user.name}</div>;
  },
  // arePropsEqual: 이전 props와 새 props가 같으면 true 반환
  (prevProps, nextProps) => {
    // timestamp는 무시하고 user만 비교
    return prevProps.user.id === nextProps.user.id;
  }
);
```

**주의:** 비교 함수가 `true`를 반환하면 리렌더링을 **건너뛴다**. (shouldComponentUpdate와 반대!)

## React.memo의 동작 원리

```
[부모 리렌더링]
      ↓
[React.memo 컴포넌트]
      ↓
[props 비교] → 같음 → [이전 렌더 결과 재사용] ✓
      ↓
    다름
      ↓
[컴포넌트 함수 실행 (리렌더링)]
```

내부적으로 React.memo는 컴포넌트의 이전 렌더링 결과(React Element)를 캐싱해두고, props가 변경되지 않았다면 저장된 결과를 그대로 반환한다.

# 5.2 useMemo

`useMemo`는 **계산 결과**를 메모이제이션하는 Hook이다. 의존성이 변경되지 않으면 이전 계산 결과를 재사용한다.

## 기본 사용법

```jsx
const memoizedValue = useMemo(() => {
  return expensiveCalculation(a, b);
}, [a, b]);  // a 또는 b가 변경될 때만 재계산
```

## React.memo vs useMemo

| | React.memo | useMemo |
|-|------------|---------|
| **대상** | 컴포넌트 전체 | 값/계산 결과 |
| **형태** | HOC (고차 컴포넌트) | Hook |
| **비교 기준** | props | 의존성 배열 |
| **목적** | 컴포넌트 리렌더링 방지 | 비싼 계산 방지 |

**쉬운 비유:**
- **React.memo**: 식당에서 같은 주문이면 주방에 안 보냄 (컴포넌트 렌더링 스킵)
- **useMemo**: 자주 쓰는 소스를 미리 만들어둠 (계산 결과 캐싱)

## 언제 사용해야 할까?

**1. 비용이 큰 계산**

```jsx
function ProductList({ products, filter }) {
  // ❌ 매 렌더링마다 필터링 수행
  const filteredProducts = products.filter(p => p.category === filter);

  // ✅ products나 filter가 변경될 때만 필터링
  const filteredProducts = useMemo(() => {
    return products.filter(p => p.category === filter);
  }, [products, filter]);

  return <List items={filteredProducts} />;
}
```

**2. 참조 동일성 유지 (React.memo와 함께 사용)**

```jsx
function Parent({ items }) {
  // ❌ 매번 새 배열 생성 → 자식이 매번 리렌더링
  const sortedItems = items.slice().sort((a, b) => a.name.localeCompare(b.name));

  // ✅ items가 변경될 때만 새 배열 생성
  const sortedItems = useMemo(() => {
    return items.slice().sort((a, b) => a.name.localeCompare(b.name));
  }, [items]);

  return <MemoizedList items={sortedItems} />;
}
```

**3. 복잡한 객체 생성**

```jsx
function ChartComponent({ data }) {
  // 차트 설정 객체를 메모이제이션
  const chartConfig = useMemo(() => ({
    data: processData(data),
    options: {
      responsive: true,
      plugins: { legend: { position: 'top' } }
    }
  }), [data]);

  return <Chart config={chartConfig} />;
}
```

## 사용하지 않아도 되는 경우

```jsx
// ❌ 과도한 사용: 단순한 계산
const doubled = useMemo(() => count * 2, [count]);

// ✅ 그냥 계산하면 됨
const doubled = count * 2;

// ❌ 과도한 사용: 원시값
const greeting = useMemo(() => `Hello, ${name}`, [name]);

// ✅ 그냥 계산하면 됨
const greeting = `Hello, ${name}`;
```

**경험 법칙:**
- 배열의 `filter`, `map`, `sort` 등 O(n) 이상의 연산
- 복잡한 객체 생성
- 반복문이나 재귀가 포함된 계산

이런 경우에만 useMemo를 고려하자.

## useMemo의 동작 원리

```
[컴포넌트 렌더링]
        ↓
[useMemo 호출]
        ↓
[의존성 배열 비교] → 같음 → [캐시된 값 반환] ✓
        ↓
      다름
        ↓
[팩토리 함수 실행]
        ↓
[결과 캐싱 & 반환]
```

내부적으로 React는 이전 의존성 배열과 계산 결과를 Fiber 노드에 저장한다. 다음 렌더링 시 의존성을 하나씩 `Object.is`로 비교하여, 모두 같으면 저장된 값을 반환한다.

## 주의사항

**1. 의존성 배열을 정확히 명시하자**

```jsx
// ❌ 빈 배열: filter가 변경되어도 재계산 안 됨
const filtered = useMemo(() => {
  return items.filter(i => i.type === filter);
}, []);  // filter 누락!

// ✅ 모든 의존성 포함
const filtered = useMemo(() => {
  return items.filter(i => i.type === filter);
}, [items, filter]);
```

**2. useMemo는 "보장"이 아닌 "힌트"**

React는 메모리 압박 시 캐시를 버릴 수 있다. useMemo 없이도 코드가 정상 동작해야 한다.

```jsx
// useMemo가 캐시를 버려도 동작은 같아야 함
// (성능만 달라질 뿐)
const value = useMemo(() => compute(a, b), [a, b]);
```

**3. 렌더링 중에만 사용하자**

```jsx
// ❌ 이벤트 핸들러 내부
function handleClick() {
  const computed = useMemo(() => heavy(), []); // Hook 규칙 위반!
}

// ✅ 컴포넌트 최상위에서 사용
function Component() {
  const computed = useMemo(() => heavy(), []);

  function handleClick() {
    console.log(computed);
  }
}
```

## 성능 측정 방법

최적화 전에 항상 측정하자:

```jsx
function Component({ data }) {
  const result = useMemo(() => {
    console.time('calculation');
    const value = expensiveCalculation(data);
    console.timeEnd('calculation');
    return value;
  }, [data]);
}
```

React DevTools의 Profiler 탭에서 각 컴포넌트의 렌더링 시간을 확인할 수 있다. 실제로 병목이 되는 곳에만 최적화를 적용하자.
