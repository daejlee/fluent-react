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

# 5.3 지연 로딩

앱이 커지며 JS 코드도 증가한다. JS 파일은 HTML, CSS보다 크기가 크고, 실행에 필요한 시간이 더 길다.

```jsx
<head>
	<title>웹사이트</title>
	<script src="https://example.com/large.js"></script> <- 병목!!!
</head>
```

async 로딩을 활용하거나, code splitting으로 최적화를 할 수 있다.

혹은 lazy loading으로 초기 실행에 필수적이지 않은 JS 파일의 로딩을 미룰 수 있다.

리액트에는 React.lazy와 Suspense를 활용한 지연 로딩이 있어 이를 사용할 수 있다.

```jsx
const Sidebar = lazy(() => import('./Sidebar'));

const MyComponent = ({ initialSidebarState }) => {
  const [showSidebar, setShowSidebar] = useState(initialSidebarState);

  return (
    <div>
      <button onClick={() => setShowSidebar(!showSidebar)}>토글</button>
      <Suspense fallback={<FakeSidebarShell />}>
        {showSidebar && <Sidebar />}
      </Suspense>
    </div>
  );
};
```

## 5.3.1 Suspense를 통한 더 나은 UI 제어

Suspense는 try/catch 블록처럼 동작한다. 파일이 달라도 가장 가까운 Suspense에서 promise를 기다리게 되어, Suspense는 지연 로딩이 필요한 컴포넌트를 감쌀 때만 쓰는 것이 좋다.

# 5.4 useState와 useReducer

useState는 단일 상태를, useReducer는 복잡한 상태를 관리하는데 적합하다.

useState또한 useReducer로 구현되어 있다. 왜 useReducer를 써야하나?

- 상태 로직을 컴포넌트에서 분리하는데 용이하고, 테스트하기 좋다.
- 상태와 상태 변경은 명시적으로 useReducer와 함께 사용된다.
- useReducer는 이벤트 소스 모델이므로, 발생하는 이벤트를 모델링해 진단 로그를 추적할 수 있다.

## 5.4.1 Immer와 편의성

Immer는 상태를 변경 가능한 초안 상태로 작업하고, 한 번 생성된 상태는 변경 불가로 만들어 복잡성을 줄인다.

useReducer로 작업 시 리듀서 함수의 순수성과 항상 새 상태 객체를 반환해야해, 코드가 방대해진다.

useImmerReducer로 Immer가 제공하는 초안 상태에서 동작하며 상태를 직접 변경하는 것처럼 보이는 리듀서를 작성할 수 있다. 다른 라이브러리에서도 Immer를 자주 사용한다.

```jsx
// useState에서 Immer를 사용하는 케이스
const MyComponent = () => {
	const [state, setState] = useState(initialState);

	const updateName = (newName) => {
		setState(
			produce((draft) => {
				draft.user.name = newName;
			});
		);
	};
};
```

# 5.5 강력한 패턴

디자인 패턴은 개발에서 반복되는 문제에 주로 사용하는 해결책이다.

- 재사용성 - 흔한 문제에 대한 재사용 가능한 해결책 제시
- 표준화 - 문제를 해결하는 표준 방법을 제공, 개발자 소통 비용 절감
- 유지 보수성 - 유지 보수가 쉬온 코드로 구조화하는 방법을 지원
- 효율성 - 일반적인 문제에 효율적인 해결책을 제시

## 5.5.1 프레젠테이션/컨테이너 컴포넌트

presentation component / container component 두 가지로 컴포넌트를 나눈다.

- presentation: UI를 렌더링
- container: UI의 상태를 처리

<img width="426" height="600" alt="image" src="https://github.com/user-attachments/assets/86655de7-2b13-477c-b9a9-915e5ad639c0" />

그러나 훅이 도입되고 이전보다 훨씬 편리하게 컴포넌트에 상태를 추가하며 컨테이너가 상태를 공급하지 않게 되었다. 이 패턴은 훅으로 대채할 수 있다. 다만 여전히 활용할 수 있고, 훅과 사용할 수도 있다.

<aside>

    💡 유지 보수성을 찾다보면 자연스럽게 이 패턴을 따라가게 되는 것 같습니다.

</aside>

## 5.5.2 고차 컴포넌트

하나 이상의 함수를 인수로 받아 함수를 반환하는 함수를 고차함수라고 한다.

JSX에서의 HOC는 다른 컴포넌트를 인수로 받아 새로운 컴포넌트를 반환하는 컴포넌트다. HOC는 여러 컴포넌트에서 공유하는 동작을 반복 작성하고 싶지 않을 때 유용하다.

여러 컴포넌트의 공통 관심사를 고차 컴포넌트가 해결해줄 수 있다. (ex. 로딩, 데이터 및 오류의 처리)

```jsx
const TodoList = withAsync(BasicTodoList);

const withAsync = (Component) => (props) => {
  if (props.loading) {
    return 'loading..';
  }

  if (props.error) {
    return error.message;
  }

  return <Component {...props} />;
};
```

하지만 프레젠테이션 및 컨테이너 컴포넌트와 마찬가지로 훅도 비슷한 이점을 제공하면서 편의성이 더 좋아 훅을 사용할 때가 더 많다.

### 고차 컴포넌트 합성

여러 고차 컴포넌트를 합성할수가 있다. 다만 중첩된 HOC 호출은 유지 보수하기 어려워진다.

<img width="674" height="310" alt="image" src="https://github.com/user-attachments/assets/d30de07d-2bf5-456a-984a-eb0f17f4d199" />

<img width="296" height="300" alt="image" src="https://github.com/user-attachments/assets/ffd24b54-cb01-44eb-aa5f-98631a09b240" />

여러 HOC를 하나의 HOC로 합성하는 유틸 함수를 작성하는 것이 낫다.

```jsx
const compose =
  (...hocs) =>
  (WrappedComponent) =>
    hocs.reduceRight((acc, hoc) => hoc(acc), WrappedComponent);

const EnhancedComponent = compose(withLogging, withUser)(MyComponent);
```

<aside>

    💡 마이크로훅으로 상태관리하기 책에서도 reduceRight으로 중첩을 해결하는 부분이 있었는데, 이런 케이스들에 사용이 가능하겠네요

</aside>

### 고차 컴포넌트와 훅 비교

훅 이후 고차 컴포넌트의 인기가 식었다. HOC는 유용한 패턴이지만, 대부분의 사례에서 단순성과 편의성에서 훅이 선호된다.

HOC는 여러 컴포넌트에서 로직을 공유하는데 탁월하고, 감싸진 컴포넌트의 렌더링을 제어하고 프롭을 조작해 컴포넌트에 추가 데이터나 기능을 제공하는 데 능숙하다.

감싸진 컴포넌트 외부의 상태를 관리하고, 관련된 수명 주기 로직을 캡슐화 할 수 있다.

반면 훅은, 렌더링에 직접 영향을 주지 않으며 프롭을 주입하거나 조작할 수 없다. 합성이 용이하고 HOC보다 쉽게 격리할 수 있어 테스트가 쉽다. TypeScript와 함께 사용하면 타입 추론이 용이하고 입력이 쉬워진다.

React.memo와 React.forwardRef등이 자주 사용되는 HOC다.

## 5.5.3 렌더 프롭

컴포넌트의 상태를 전달받는 함수를 prop으로 사용하는 것도 흔히 보게 되는 패턴이고, 코드의 재사용성을 늘릴 수 있다.

```jsx
<WindowSize
  render={({ width, height }) => (
    <div>
      창 크기: {width}x{height}px
    </div>
  )}
/>;

const WindowSize = (props) => {
  const [size, setSize] = useState({ width: -1, height: -1 });

  useEffect(() => {
    const handleSize = () => {
      setSize({ width: window.innerWidth, height: window.innerHeight });
    };
    window.addEventListener('resize', handleResize);
    return () => window.removeEventListener('resize', handleResize);
  }, []);

  return props.render(size);
};
```

사용자 창 크기를 계산하는 작업을 수행하고, props.render를 호출해 선언한 구조를 반환한다.

헤드리스 컴포넌트다. 자신을 렌더링하는 부모 컴포넌트로부터 전달받은 렌더 프롭을 호출해 렌더링에 대한 제어를 부모에게 넘겨 제어를 역전시킨다.

이 패턴은 널리 쓰이진 않으며 훅으로 대체되었다.

## 5.5.4 제어 프롭

리액트 제어 프롭 패턴은 컴포넌트 내의 상태 관리를 결정하기 위한 유연한 메커니즘을 제공한다.

제어 컴포넌트를 먼저 살펴보면, 내부에 자체 상태를 유지하지 않는 컴포넌트다. 부모 컴포넌트가 단일 정보 출처의 역할을 한다. 상태가 변경되어야 하면 onChange같은 콜백 함수로 부모에게 알린다.

<img width="690" height="230" alt="image" src="https://github.com/user-attachments/assets/8bf583a1-2c7d-469a-adee-2f09ace20055" />

제어 프롭 패턴은 제어 컴포넌트의 원리를 확장한다. 컴포넌트는 외부에서 프롭으로 제어될 수 있고, 자체적으로 내부 상태를 관리할 수 있다.

이 패턴은 제어 및 비제어 작동 모드를 모두 제공해 유연성을 향상시킨다.

<img width="476" height="467" alt="image" src="https://github.com/user-attachments/assets/d04a6f3e-38f4-44db-a461-e2dedb60e5ea" />

<aside>
	
	💡유연하다고 하지만, 다른 말로 하면 독립성이 떨어진다고도 할 수 있을 것 같습니다.

</aside>

## 5.5.5 프롭 컬렉션

프롭이 같이 쓰이는 케이스가 많다면 프롭을 한데 모아두고 재사용하는 것이 이롭다.

```jsx
export const droppableProps = {
	onDragOver: (event) => {
		...
	},
	onDrop: (event) => {},
};

<Dropzone {...droppableProps} />

// 다만 이렇게 쓰이면 프롭 컬렉션의 함수가 사라져버린다
<Dropzone
	{...droppableProps}
	onDragOver={() => {
		alert("Dragged!");
	}}
/>
```

### 프롭 게터

<img width="585" height="459" alt="image" src="https://github.com/user-attachments/assets/5fd6e4fd-1c35-44dd-9df9-8fb03d97a284" />

프롭 게터는 함수로, 사용자 정의 함수를 인수로 받아 기본값과 함께 조합한다.

## 5.5.6 복합 컴포넌트

복합 컴포넌트(Compound Component)는 서로 연결되고, 상태를 공유하면서도 독립적으로 렌더링되는 컴포넌트를 한데 묶어 엘리먼트 트리를 더 세밀하게 제어하게 해준다.

```jsx
<Accordion>
  <AccordionItem item={{ label: 'One' }} />
  <AccordionItem item={{ label: 'Two' }} />
  <AccordionItem item={{ label: 'Three' }} />
</Accordion>
```

리액트 컨텍스트를 이용해 구현할 수 있다. 이 방식의 이점은 AccordionItem이 Accordion의 더 큰 상태를 인식하면서도 더 많이 제어할 수 있다는 것이다.

이것의 이점은 자식이 컨텍스트 상태를 인식하게 하면서 동시에 렌더링 제어를 부모에게 넘기는 것이다.

또한, 상태와 관심사 분리가 자연스럽게 이루어지게 도와 앱의 확장성이 훨씬 더 좋아진다.

<aside>

    💡 이거 저희 팀 코드중에 좋은 예시가 있는데 같이 보면 좋을 것 같습니다.

</aside>

## 5.5.7 상태 리듀서

이 패턴은 사용자 정의 가능한 컴포넌트를 만드는 강력한 방법을 제시한다.

```jsx
function toggleReducer(state, action) {
  switch (action.type) {
    case 'TOGGLE':
      return { on: !state.on };
    default:
      throw new Error(`알 수 없는 액션 타입: ${action.type}`);
  }
}

function Toggle({ stateReducer }) {
  const [state, internalDispatch] = useReducer(
    (state, action) => {
      const nextState = toggleReducer(state, action);
      return stateReducer(state, { ...action, changes: nextState });
    },
    { on: false },
  );

  return (
    <button onClick={() => internalDispatch({ type: 'TOGGLE' })}>
      {state.on ? 'On' : 'Off'}
    </button>
  );
}

Toggle.defaultProps = {
  stateReducer: (state, action) => state,
};

function App() {
  const customReducer = (state, action) => {
    if (new Date().getDay() === 3 && !action.changes.on) {
      return state;
    }
    return action.changes;
  };

  return <Toggle stateReducer={customReducer} />;
}
```

위 예시에서는 사용자 정의된 stateReducer를 제공한다. 수요일에는 토글을 끌 수 없게 하는 로직을 리듀서에 포함시켰다. 상태 리듀서 패턴으로 매우 유연하고 재사용 가능한 컴포넌트를 만들 수 있다.
