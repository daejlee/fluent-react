## 5.3 지연 로딩

코드의 일부만 사용해야하는 상황에서도 전체 코드를 다운받아야하는 상황이 생긴다.  
크기가 큰 자바스크립트 파일일 경우엔 파일을 다운 받는시간이 길어지기에 페이지 로딩 시간이 느려질 수 있고, 데이터 사용량도 많아진다.

=> 이를 개선하기 위한 방식이 **지연로딩** 이다.

리액트에선 **React.lazy**와 **Suspense**를 사용하여 지연로딩을 구현할 수 있다.

```jsx
const Sidebar = lazy(() => import("./Sidebar"));

const MyComponent = ({ initialSidebarState }) => {
  const [showSidebar, setShowSidebar] = useState(initialSidebarState);

  return (
    <div>
      <button onClick={() => setShowSidebar(!showSidebar)}>사이드바 토글</button>
      <Suspense fallback={<FakeSidebarShell />}>{showSidebar && <Sidebar />}</Suspense>
    </div>
  );
};
```

Suspense는 try/catch 블록과 비슷한 방식으로 동작한다.

지연로딩이 필요한 컴포넌트의 프로미스가 해결될 때까지 트리의 상위 계층 중 가장 가까운 Suspense에서 fallback을 렌더링 해준다.

Suspense로 감싸는 컴포넌트가 많아지면 지연로딩이 필요하지 않은 컴포넌트마저 가릴 수 있기 때문에 지연로딩이 필요한 컴포넌트에만 사용해주는 게 좋다.

## 5.4 useState와 useReducer

useState - 단일 상태 관리 / useState는 내부적으로 useReducer를 사용한다.

useReducer

- 복잡한 상태 관리
- 단일 책임 원칙: 상태 업데이트 로직을 컴포넌트에서 분리하기에, 단독으로 테스트하고 다른 컴포넌트에서도 재사용하기 좋다.
- 상태 업데이트 흐름 파악: 상태와 상태 변경 방식은 항상 명시적으로 useReducer와 함께 사용된다.
- 이벤트 소스 모델: 이벤트를 어떤 일이 일어났는지 모델링해서 진단 로그를 추적하는 데 사용할 수 있다.

```jsx
const [count, setCount] = useState(0);

setCount(5);
setCount(10);
setCount(3);

console.log(count); // 3
// 문제: "왜 3이 됐는지" 과정을 알 수 없음
```

```jsx
const [state, dispatch] = useReducer(reducer, { count: 0, history: [] });

function reducer(state, action) {
  // 모든 "이벤트"를 기록
  const newHistory = [
    ...state.history,
    {
      type: action.type,
      timestamp: new Date(),
      payload: action.payload,
    },
  ];

  switch (action.type) {
    case "INCREMENT":
      return {
        count: state.count + 1,
        history: newHistory,
      };
    case "DECREMENT":
      return {
        count: state.count - 1,
        history: newHistory,
      };
    case "RESET":
      return {
        count: 0,
        history: newHistory,
      };
  }
}

// 사용
dispatch({ type: "INCREMENT" });
dispatch({ type: "INCREMENT" });
dispatch({ type: "DECREMENT" });

console.log(state.history);
// [
//   { type: 'INCREMENT', timestamp: '2024-...', payload: undefined },
//   { type: 'INCREMENT', timestamp: '2024-...', payload: undefined },
//   { type: 'DECREMENT', timestamp: '2024-...', payload: undefined }
// ]
// → "어떤 순서로 어떤 일이 일어났는지" 추적 가능!
```

추가로 **Immer**를 사용하면 자동으로 불변성을 지키기에 스프레드 방식을 쓰거나 state 형식을 신경쓰지 않고 작업할 수 있다.

## 5.5 강력한 패턴

디자인 패턴을 쓰는 이유

- 재사용성
- 표준화
- 유지보수성
- 효율성

### 5.5.1 프레젠테이션/컨테이너 컴포넌트

상태와 UI 렌더 역할을 분리함으로써 단일 책임 원칙을 지키고, 애플리케이션에서 관심사를 분리해 모듈화, 재사용, 테스트를 가능케 하는 패턴

- 프레젠테이션 컴포넌트: UI를 렌더링
  - 내부에 상태가 포함된 컨테이너 컴포넌트(유상태)에 전달이 되어도 의도한 모양을 유지
  - 단독으로 시각적 테스트 가능
- 컨테이너 컴포넌트: UI 상태를 처리
  - 유상태 컨테이너로 대체되어도 의도한 기능 유지
  - 단독으로 단위 테스트 가능

현재는 훅이 도입되었기에 컨테이너 컴포넌트 역할을 대체하고 있다.

### 5.5.2 고차 컴포넌트

컴포넌트를 인자로 받아 내부 로직을 통해 새로운 컴포넌트 혹은 컴포넌트들을 반환해주는 컴포넌트(함수)

예를 들어, 상태에 따라 로딩상태, 오류상태, 일반상태의 컴포넌트를 반환해주어야하는 **공통적인 로직** 처리해주어야하는 경우 사용된다.

#### 고차 컴포넌트 합성

고차 컴포넌트 여러 개를 중첩해 사용할 수도 있는데, 이럴 경우 여러 개의 고차 컴포넌트를 하나의 고차 컴포넌트로 합성하는 유틸리티를 만들어 사용하면 좋다.

```jsx
const compose =
  (...hocs) =>
  (WrappedComponent) =>
    hocs.reduceRight((acc, hoc) => hoc(acc), WrappedComponent);

const EnhancedComponent = compose(withLoggin, withUser)(MyComponent);
```

#### 고차 컴포넌트와 훅 비교

고차 컴포넌트도 훅이 도입된 이후 인기가 식었다.

<img width="1265" height="895" alt="image" src="https://github.com/user-attachments/assets/876a6060-6eeb-4dcc-bfb2-ea235d18d922" />


### 5.5.3 렌더 프롭

컴포넌트의 상태를 전달받아 UI를 렌더링하는 함수를 프롭으로 사용하는 패턴

- 코드 재사용성 늘림
- 헤드리스한 구조로 UI 렌더링 작업에 대한 제어를 부모에게 넘기고, 정보를 내부적으로 동작시킴
- 훅으로 많이 대체 됨

```jsx
<WindowSize render={({widht, height}) => <div>창크기: {width}x{height}px</div>}/>

const WindowSize = ({render}) => {
  ... // size 추출하는 내부 로직

  return render(size);
}
```

render와 children 둘 다 같은 props이기에 render 대신 children으로 넣어주는 방식도 있는데, 이를 **자식 함수**라 한다.

### 5.5.4 제어 프롭

제어 컴포넌트의 개념을 확장한 것이다.

> 제어 컴포넌트
>
> 내부에 자체 상태를 유지하지 않는 컴포넌트로, 상태의 현잿값은 부모 컴포넌트에서 전달한 프롭에 의해 결정된다.  
> 상태가 변경되어야 할 때 onChange 같은 콜백 함수를 통해 부모에게 알리기에, 상태를 관리하고 제어 컴포넌트의 값을 업데이트하는 책임 모두 부모에 있다.

제어 프롭 패턴의 컴포넌트는 상탯값과 상태를 업데이트하는 함수 모두 프롭으로 받고, 내부적으로도 자체 상태를 관리할 수 있다.

두 가지 기능 모두 제공함으로써 자식 컴포넌트에 대한 상태를 제어할 수 있지만, 부모가 제어하지 않는 경우 자식 컴포넌트가 독립적으로 동작할 수 있다.(비제어)

### 5.5.5 프롭 컬렉션

프롭이 많을 경우 프롭들을 객체로 같이 묶어 사용하는 방식

```jsx
const droppableProps = {
  onDragOver: ...,
  onDrop: ...
}

<Dropzone {...droppableProps}/>
```

다만, `<Dropzone {...droppableProps} onDragOver={...}/>` 처럼 프롭 컬렉션에 있던 프롭을 덮어씌우면 예상치 못한 동작이 발생할 수 있는데 **프롭게터**를 사용하면 된다.

#### 프롭 게터

사용자 정의 프롭을 프롭 컬렉션에 조합하고 병합한다.

```jsx
export const getDroppableProps = ({
  onDragOver: replacementOnDragOver, // 사용자 정의 프롭을 받음
  ...replacementProps
}) => {
  const defualt = (e) =>{...};

  return {
    onDragOver: compose(replacementOnDragOver, default), // 병합
    onDrop: ...,
    ...replacementProps // 병합
  }
}

<Dropzone {...getDroppableProps(...)}/>
```

### 5.5.6 복합 컴포넌트

서로 연결되고 상태를 공유하면서도 독립적으로 렌더링되는 컴포넌트를 한데 묶어 엘리먼트 트리를 더 세밀하게 제어하게 해준다.

- 자식에게 React.CloneElement사용

- 리액트 콘텍스트 사용

### 5.5.7 상태 리듀서

리듀서를 사용하는 컴포넌트에 stateReducer(상태 리듀서)프롭을 전달해 컴포넌트의 내부 리듀서와 결합하여 내부 상태 로직을 변경할 수 있다.

```jsx
function Toggle({ stateReducer }) {
  const [state, internalDispatch] = useReducer(
    (state, action) => {
      const nextState = (state, action) => {
        switch (action.type) {
          case "TOGGLE":
            return {on: !state.on};
          ...
        }
      }
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

..

const customReducer = (state, action) => {
  if (new Date().getDay() === 3 && !action.changes.on) {
    return state;
  }
  return action.changes;
};

<Toggle stateReducer={customReducer} />;

```

상태 리듀서를 프롭으로 전달해 수요일만 동작하지 않는다.

공통 로직은 기본 리듀서로 설정하고, 상태 리듀서 프롭을 넘겨줌으로써 특정한 로직을 생성할 수 있다.

이로 인해, 컴포넌트를 재사용할 수 있고, 컴포넌트의 유용성과 다향성도 향상시킨다.
