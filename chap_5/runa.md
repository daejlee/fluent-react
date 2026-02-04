# 5 - 자주 묻는 질문과 유용한 패턴

## 1. React.memo

### 1) 메모화

- 계산된 결과를 메모리에 저장해 두었다가, 동일한 입력이 들어오면 다시 계산하지 않고 저장된 값을 재사용하는 최적화 기법
- 리액트에서는 컴포넌트 함수가 불필요하게 다시 호출되는 것을 막는 용도로 사용한다.

### 2) 기본 사용법

컴포넌트를 `React.memo`로 감싸면 프롭이 변경되지 않는 한 리렌더링을 건너뛴다.

```tsx
// 무거운 컴포넌트
const TodoList = React.memo(function TodoList({ todos }) {
  return (
    <ul>
      {todos.map((todo) => (
        <li key={todo.id}>{todo.title}</li>
      ))}
    </ul>
  );
});
```

### 3) 메모화가 실패하는 이유

- `React.memo`는 프롭을 비교할 때 얕은 비교를 한다.
- 자바스크립트에서 객체, 배열, 함수는 내용이 같아도 매번 새로운 참조를 가지기 때문에 메모화가 깨지기 쉽다.

```tsx
function App() {
  const [count, setCount] = useState(0);

  // 부모가 리렌더링될 때마다 새로운 참조 생성
  const items = ["apple", "banana", "cherry"];
  const handleSave = () => console.log("저장");

  return (
    <>
      <button onClick={() => setCount(count + 1)}>단순 카운트 증가</button>
      {/* items와 handleSave의 내용이 같아도 
         참조가 매번 바뀌므로 TodoList는 무조건 리렌더링
      */}
      <TodoList items={items} onSave={handleSave} />
    </>
  );
}
```

### ⇒ 해결책: useMemo, useCallback를 활용한 참조 고정

참조 타입 프롭 때문에 메모화가 깨지는 것을 막으려면 부모 컴포넌트에서 참조를 고정해야 한다.

```tsx
function App() {
  const [count, setCount] = useState(0);

  // 1. 값/배열/객체 메모화
  const items = useMemo(() => ["apple", "banana", "cherry"], []);

  // 2. 함수 메모화
  const handleSave = useCallback(() => console.log("저장"), []);

  return (
    <>
      <button onClick={() => setCount(count + 1)}>카운트 증가</button>
      <TodoList items={items} onSave={handleSave} />
    </>
  );
}
```

### 4) 리액트 내부 동작

- `defaultProps`나 커스텀 비교 함수가 없는 경우 리액트는 더 빠른 경로로 업데이트 여부를 판단한다.
- 프롭이 동일하고 예약된 업데이트가 없다면 즉시 작업을 종료하여 성능을 아낀다.
- 프롭이 같더라도 컴포넌트 내부의 상태가 변하거나, 구독 중인 컨텍스트의 값이 바뀌면 메모화와 상관없이 리렌더링이 발생한다.

## 2. useMemo, useCallback

- **React.memo:** 컴포넌트 자체를 메모화하여 불필요한 리렌더링을 방지
- **useMemo:** 컴포넌트 내부의 계산된 값을 메모화하여 비용이 큰 재계산을 방지
- **useCallback:** 컴포넌트 내부의 함수 인스턴스를 메모화하여 자식 컴포넌트에 일관된 참조를 전달 (함수를 위한 `useMemo`)

### 1) useMemo: 비용이 큰 계산 최적화

데이터가 많을 때(예: 100만 건) 정렬이나 필터링 같은 연산은 시간 복잡도가 커서 성능 병목을 일으킨다.

```tsx
const People = ({ unsortedPeople }) => {
  const [name, setName] = useState("");

  // unsortedPeople이 변경될 때만 정렬 수행
  const sortedPeople = useMemo(() => {
    return [...unsortedPeople].sort((a, b) => b.age - a.age);
  }, [unsortedPeople]);

  return (
    <>
      <input value={name} onChange={(e) => setName(e.target.value)} />
      {/* 이제 이름을 입력할 때마다 100만 명을 정렬하는 불필요한 연산을 하지 않는다. */}
    </>
  );
};
```

### 2) useCallback: 함수 참조 안정성

자식 컴포넌트가 `React.memo`로 최적화되어 있어도, 부모가 매번 새로운 함수를 넘겨주면 메모화가 깨진다.
이때 `useCallback`으로 함수 참조를 고정한다.

```tsx
const MyComponent = () => {
  const [count, setCount] = useState(0);

  // count가 변하지 않는 한 동일한 함수 참조 유지
  const incrementCount = useCallback(() => {
    setCount((prev) => prev + 1);
  }, []);

  return <ExpensiveComponent onClick={incrementCount} />;
};
```

### 3) 과도한 최적화

모든 것을 메모화하면 오히려 런타임 복잡성과 메모리 오버헤드 때문에 애플리케이션이 느려질 수 있다.

- **스칼라 값/단순 할당:** 단순 논리 연산은 자바스크립트 엔진이 충분히 빠르므로 `useMemo`를 쓰지 않는 것이 좋다.
- **의존성 비교 비용:** 메모화 자체도 의존성 배열을 비교하는 과정을 거치므로, 비교 비용이 계산 비용보다 클 수도 있다.
