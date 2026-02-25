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

## 3. 지연 로딩 (Lazy Loading)

### 1) 배경과 문제점

애플리케이션 규모가 커지면 자바스크립트 코드의 양도 필연적으로 증가한다.

- **초기 로딩 지연:** 사용자가 당장 필요하지 않은 코드(ex/ 클릭해야 열리는 사이드바, 다른 페이지 등)까지 모두 다운로드하느라 초기 화면이 늦게 뜬다.
- **데이터 낭비:** 자바스크립트는 HTML/CSS보다 크기가 크고 파싱, 실행 비용이 높다. 모바일이나 느린 네트워크 환경에서는 치명적이다.
- **단순 해결책:** `<script async>`를 사용해 병렬로 다운로드 할 수 있지만, 전송량을 줄이는 근본적인 해결책은 아니다.

### 2) 코드 분할 (Code Splitting)

하나의 거대한 번들 파일 대신, 코드를 여러 조각으로 나누어 **'지금 필요한 것만'** 읽어 들이는 기법이다.

- **동적 가져오기 (Dynamic Import)**
  `import()` 구문을 사용하면 특정 이벤트(클릭 등)가 발생했을 때 파일을 로드할 수 있다.

```tsx
// 버튼을 클릭할 때만 무거운 JS 파일을 다운로드함
button.addEventListener("click", () => {
  import("./heavy-feature.js").then((module) => {
    module.run();
  });
});
```

### 3) React.lazy와 Suspense

리액트는 코드 분할을 컴포넌트 레벨에서 선언적으로 처리할 수 있는 직관적인 API를 제공한다.

1. **React.lazy**

- 동적 `import()`를 호출하여 컴포넌트를 정의한다.
- 이 컴포넌트는 실제로 렌더링(호출)되기 전까지는 네트워크 요청을 보내지 않는다.
- 모듈 자체가 아닌 **Promise**를 반환한다.

1. **Suspense**

- 지연 로딩된 컴포넌트가 로드되는 동안 보여줄 **Fallback UI**를 정의한다.
- 네트워크 요청이 완료될 때까지 사용자가 빈 화면을 보지 않도록 스피너나 스켈레톤 UI를 보여준다.

```tsx
import { useState, lazy, Suspense } from "react";
// 무거운 사이드바를 미리 불러오지 않고 지연시킴
const Sidebar = lazy(() => import("./Sidebar"));
import FakeSidebarShell from "./FakeSidebarShell"; // 가벼운 로딩 UI

const MyComponent = () => {
  const [show, setShow] = useState(false);

  return (
    <div>
      <button onClick={() => setShow(!show)}>토글</button>

      {/* Sidebar 데이터가 도착할 때까지 fallback을 보여줌 */}
      <Suspense fallback={<FakeSidebarShell />}>{show && <Sidebar />}</Suspense>
    </div>
  );
};
```

### 4) Suspense Boundary와 UX 제어

`Suspense`는 자바스크립트의 `try/catch` 블록과 유사하게 동작한다. 하위 트리 어디에서든 비동기 로딩이 발생하면 가장 가까운 상위 `Suspense`가 이를 포착하여 Fallback을 보여준다.

1. 앱 전체를 하나의 `Suspense`로 감싸면, 작은 컴포넌트 하나를 로딩하느라 앱 전체가 로딩 화면으로 변해버려 사용자 상호작용이 막힌다. 👎
2. 지연 로딩이 필요한 특정 영역만 감싸서, 메인 콘텐츠는 보여주고 사이드바만 로딩 상태를 표시하는 것이 좋다. 👍

```tsx
// 좋은 패턴: 메인 콘텐츠는 유지하고 사이드바만 로딩 처리
return (
  <div>
    <Suspense fallback={<SkeletonUI />}>{show && <Sidebar />}</Suspense>

    <main>
      <p>여기는 사이드바 로딩과 상관없이 즉시 상호작용</p>
    </main>
  </div>
);
```

1. 효과
   - **레이아웃 이동 방지:** Fallback UI를 사용해 화면이 덜컹거리는 것을 막는다.
   - **응답성 향상:** 무언가가 진행 중임을 사용자가 즉시 인지할 수 있다.

## 4. useState와 useReducer

- **useState:** 단일 값이나 단순한 구조의 상태를 관리할 때 적합하다.
  - `useState`는 내부적으로 `useReducer`를 사용해 구현된 상위 추상화 버전이다.
- **useReducer:** 여러 속성이 포함된 복잡한 객체나, 상태 업데이트 로직이 복잡할 때 적합하다.

### 1) useState 사용 시 주의점

상태가 여러 속성을 가진 객체일 경우, `useState`를 쓰면 매번 이전 상태를 `...state`과 같이 전개해야 한다. 이 과정에서 실수로 속성을 누락하면 상태가 덮어씌워지는 문제가 발생하기 쉽다.

```tsx
const [state, setState] = useState({ count: 0, name: "Tejumma", age: 30 });

// 일부 속성(name, age)을 누락하면 상태에서 사라짐
setState({ count: state.count + 1 });
```

### 2) useReducer의 이점

`useReducer`는 코드가 다소 길어질 수 있지만, 복잡한 앱을 만들 때 다음과 같은 강력한 이점을 제공한다.

- **로직 분리:** 상태 업데이트 로직을 컴포넌트 외부로 완전히 분리할 수 있다. 컴포넌트는 무엇을 할지(Action)만 알리고, 어떻게 바꿀지(Reducer)는 컴포넌트가 몰라도 된다.
- **이벤트 소싱 모델:** 모든 변화가 액션 객체로 전달되므로 진단 로그를 남기거나 실행 취소, 재실행, 디버깅 등을 구현하기 유리하다.
- **테스트 최적화:** 리듀서는 순수 함수이므로 단독 유닛 테스트가 가능하다.
  - 결과의 값(`toEqual`)뿐만 아니라, 의도하지 않은 액션 시 기존 참조(`toBe`)가 유지되는지도 함께 확인할 것

### 3) Immer

- 상태가 깊게 중첩된 경우, 스프레드를 여러 번 쓰는 것은 보기에 매우 복잡하다. 이때 **Immer**를 사용하면 마치 직접 값을 수정하는 것처럼 코드를 작성하면서도 **불변성**을 지킬 수 있다.

  ```tsx
  import { useImmerReducer } from "use-immer";

  const reducer = (draft, action) => {
    switch (action.type) {
      case "updateCity":
        draft.user.address.city = action.payload;
        break;
      default:
        break;
    }
  };
  ```

- produce는 `useState`와 함께 쓸 때 유용하며, 현재 상태의 초안을 수정하면 자동으로 새로운 불변 객체를 생성해준다.

  ```tsx
  import { produce } from "immer";

  const [user, setUser] = useState({
    name: "John Doe",
    address: { city: "New York", country: "USA" },
  });

  const updateCity = (newCity) => {
    setUser(
      produce((draft) => {
        draft.address.city = newCity;
      })
    );
  };
  ```

## 5. 강력한 패턴

### 1) 프레젠테이션 컴포넌트와 컨테이너 컴포넌트

컴포넌트의 역할을 보여주는 것(UI)과 관리하는 것(Logic)으로 명확히 나누는 패턴

- **프레젠테이션 컴포넌트:** 오직 UI 렌더링에만 집중합니다. 상태(State)를 직접 가지지 않고 부모로부터 받은 Props를 화면에 뿌려준다. (시각적 테스트 및 디자이너와의 협업에 유리)
- **컨테이너 컴포넌트:** 데이터 페칭, 상태 관리 등 비즈니스 로직을 처리한다. 화면에 무엇이 보일지는 프레젠테이션 컴포넌트에게 맡긴다. (단위 테스트 및 로직 재사용에 유리)

→ 과거에는 대중적이었으나, 커스텀 훅 등장하면서 로직을 훅으로 추출하는 방식이 더 선호되고 있습니다.

### 2) 고차 컴포넌트 (HOC)

컴포넌트를 인수로 받아, 추가적인 기능이 더해진 새로운 컴포넌트를 반환하는 함수 (고차 함수 개념)

- 비동기 통신을 하는 모든 컴포넌트마다 로딩, 에러 상태를 수동으로 구현하면 코드가 중복되는데, 이를 HOC로 단순화가 가능하다.

  ```tsx
  // 로딩과 에러 UI 처리를 공통화하는 HOC 정의
  const withAsync = (Component) => (props) => {
    if (props.loading) return "로딩 중...";
    if (props.error) return props.error.message;
    return <Component {...props} />;
  };

  const TodoList = withAsync(BasicTodoList);
  ```

- 여러 컴포넌트 중첩 시, 중첩이 심해지는 래퍼 지옥(Wrapper Hell)이 발생할 수 있다. 이를 해결하기 위해 `compose` 유틸리티를 통해 코드를 관리하는 방식이 있다.

  ```tsx
  const EnhancedComponent = withErrorHandler(withLoading(withLogging(withUser(MyComponent))));
  ```

  ```tsx
  // compose 유틸리티 함수
  const compose =
    (...hocs) =>
    (WrappedComponent) =>
      hocs.reduceRight((acc, hoc) => hoc(acc), WrappedComponent);

  // flat하게 관리 가능
  const EnhancedComponent = compose(withErrorHandler, withLoading, withLogging, withUser)(MyComponent);
  ```

- **고차 컴포넌트와 훅의 비교**
  | **기능** | **고차 컴포넌트 (HOC)** | **훅 (Hooks)** |
  | --------------- | -------------------------------------------- | ------------------------------------------ |
  | **렌더링 제어** | 컴포넌트 자체를 감싸서 렌더링 여부 결정 가능 | 직접 렌더링을 막지는 못함 (부수 효과 관리) |
  | **코드 구조** | **래퍼 지옥** 발생 가능성 높음 | 플랫하게 로직 합성 가능 |
  | **타입 안전성** | TypeScript 적용 시 타입 추론이 매우 까다로움 | 타입 추론이 쉽고 직관적임 |
  | **참조 전달** | `ref` 전달이 복잡함 (`forwardRef` 필요) | `ref` 사용이 상대적으로 자유로움 |
- `React.memo`나 `React.forwardRef` 등이 대표적인 HOC이다. 하지만 일반적인 비즈니스 로직 공유를 위해 이제는 HOC보다 커스텀 훅이 더 권장되는 추세이다.

### 3) 렌더 프롭

컴포넌트의 상태를 함수 형태의 프롭으로 전달하여 렌더링 로직을 외부로 위임하는 패턴

- 로직은 자식 컴포넌트가 담당하지만, 시각적 표현은 부모가 결정한다. 동일한 로직을 유지하면서 상황에 따라 다른 UI를 렌더링할 수 있어 코드의 재사용성이 높다.
- **헤드리스 컴포넌트:** UI를 직접 표현하지 않고 데이터와 기능만 제공하는 컴포넌트를 의미한다.

```tsx
// WindowSize: 창 크기 로직만 담당하는 헤드리스 컴포넌트
<WindowSize>
  {({ width, height }) => (
    <p>
      창 크기: {width}x{height}px
    </p>
  )}
</WindowSize>
```

- `render`라는 이름 대신 `children`을 함수로 전달하는 방식을 선호하기도 한다.
- 리액트 훅이 등장하면서 현재는 거의 모든 렌더 프롭 패턴이 커스텀 훅으로 대체되었다.

### 4) 제어 프롭

- 컴포넌트 내부에 자체 상태를 가지면서도, 외부에서 전달된 프롭이 있다면 이를 우선하여 사용한다.

```tsx
function Toggle({ on, onToggle }) {
  const [isOn, setIsOn] = useState(false);

  // 외부 프롭(on)이 있으면 그것을 쓰고, 없으면 내부 상태(isOn)를 사용
  const actualOn = on !== undefined ? on : isOn;

  const handleToggle = () => {
    if (on === undefined) setIsOn(!actualOn);
    onToggle?.(!actualOn);
  };

  return <button onClick={handleToggle}>{actualOn ? "On" : "Off"}</button>;
}
```

### 5) 프롭 컬렉션과 프롭 게터

- **프롭 컬렉션:** 여러 관련 프롭을 객체로 묶어 전개 연산자(`...props`)로 한 번에 전달한다. 단, 사용자가 프롭을 덮어씌울 경우 내부의 기본 기능이 무시될 위험이 있다.
- **프롭 게터:** 컬렉션을 반환하는 함수 형태이다. 사용자의 커스텀 프롭과 기본 프롭을 합성하여 두 로직이 충돌 없이 모두 실행되도록 보장한다.

```tsx
const getDroppableProps = ({ onDragOver: customOnDragOver, ...rest } = {}) => {
  return {
    onDragOver: (event) => {
      event.preventDefault(); // 기본 기능 유지
      customOnDragOver?.(event); // 사용자 정의 로직 실행
    },
    ...rest,
  };
};
```

### 6) 복합 컴포넌트

- Context API를 활용하여 부모와 자식 간에 상태를 공유 및 엘리먼트 트리에 대한 세밀한 제어가 가능하다.
- 자식 컴포넌트들을 고정된 배열로 렌더링하지 않고, 원하는 위치에 자유롭게 배치하거나 중간에 다른 엘리먼트를 끼워 넣을 수 있다.

```tsx
<Accordion>
  <AccordionItem index={0} label="섹션 1">
    내용 1
  </AccordionItem>
  <hr /> {/* 중간에 자유롭게 엘리먼트 삽입 가능 */}
  <AccordionItem index={1} label="섹션 2">
    내용 2
  </AccordionItem>
</Accordion>
```

### 7) 상태 리듀서

컴포넌트 내부의 상태 업데이트 로직 자체를 외부에서 주입하여 동작을 커스터마이징하는 패턴

- 컴포넌트가 다음 상태를 결정하기 직전에 사용자가 전달한 `stateReducer`를 거치게 한다.
- 예상치 못한 세밀한 요구사항을 컴포넌트 코드를 수정하지 않고 외부에서 구현할 수 있다.

```tsx
function App() {
  const customReducer = (state, action) => {
    // 특정 요일에는 토글이 꺼지지 않도록 로직 강제
    if (new Date().getDay() === 3 && !action.changes.on) {
      return state;
    }
    return action.changes;
  };

  return <Toggle stateReducer={customReducer} />;
}
```
