# 5.1 React.memo를 사용한 메모화

memoization은 이전에 계산된 결과를 캐싱해 함수의 성능을 최적화 하는 기법이다.

React.memo 함수에서 반환하는 새 컴포넌트는 프롭이 변경되었을 때만 리렌더링된다.

```tsx
function App() {
  const todos = Array.from({ length: 1000000 });
  const [name, setName] = useState('');

  return (
    <div>
      <input value={name} onChange={(e) => setName(e.target.value)} />
      <TodoList todos={todos} />
    </div>
  );
}

const MemoizedTodoList = React.memo(function TodoList({ todos }) {
  return (
    <ul>
      {todos.map((todo) => (
        <li key={todo.id}>{todo.title}</li>
      ))}
    </ul>
  );
});
```

input 필드가 바뀔 때 마다 TodoList 전체가 리렌더링 된다. 상태 변경이 일어나면 재조정 과정에서 하위 트리에 있는 모든 함수 컴포넌트가 재호출되는 것은 리액트 작동 원리의 핵심이다. (트리에서 뭐가 바뀐지를 모르기 때문에)

위와 같이 프롭이 변경된 경우에만 컴포넌트를 리렌더링한다.

특히, 렌더링 비용이 많이 드는 중첩된 컴포넌트가 여러 개 있는 복잡한 컴포넌트의 경우,
내부의 각 컴포넌트를 메모화하면 최적화가 가능하다.

<img width="570" height="498" alt="image" src="https://github.com/user-attachments/assets/2f5e6b45-39fc-4854-85fc-b40b178a206d" />

## 5.1.1 React.memo에 능숙해지기

얼마나 자주 메모해야하나? 모든 컴포넌트를 메모화해야 하나?

## 5.1.2 리렌더링되는 메모화된 컴포넌트

React.memo는 프롭에 얕은 비교를 수행한다. JS의 스칼라 타입은 비교가 정확하지만, 않은 값들도 있다.

<img width="339" height="167" alt="image" src="https://github.com/user-attachments/assets/f2155cc9-3780-4aac-8d1e-b7d2ff09d555" />

이런 경우 useMemo , 함수 참조인 경우 useCallback으로 메모화하면 문제가 해결된다.
