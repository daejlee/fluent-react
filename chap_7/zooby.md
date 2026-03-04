# 7.1 동기식 렌더링의 문제

React 18 이전의 렌더링은 동기적(synchronous)이었다. 한번 렌더링이 시작되면 끝날 때까지 멈출 수 없다.

```
[동기식 렌더링]
상태 변경 → 렌더링 시작 =========================> 렌더링 완료 → DOM 반영
                        (메인 스레드 블로킹)
```

브라우저는 초당 60프레임(16.6ms마다 한 프레임)을 그려야 부드럽게 느껴진다. 렌더링이 16ms를 넘기면 프레임 드랍이 발생하고, 사용자 입력 처리도 지연된다.

```
[프레임 타임라인]
|--16ms--|--16ms--|--16ms--|--16ms--|
|  렌더  |  입력  |  렌더  |  페인트  |  ← 이상적

|---------- 200ms 렌더링 ------------|
|  입력 대기...  입력 대기...  입력 대기... |  ← 동기식 문제
```

모든 상태 업데이트가 동일한 우선순위로 처리되는 것이 문제다. 사용자 입력(긴급)과 리스트 렌더링(덜 긴급)이 구분되지 않는다.

# 7.2 파이버 재조정자

4장에서 Fiber를 데이터 구조로 살펴봤다면, 여기서는 동시성의 기반으로서의 Fiber를 본다.

## Fiber가 동시성을 가능하게 하는 이유

Stack Reconciler는 콜스택 기반이라 중단이 불가능했다.
Fiber는 각 작업을 독립적인 객체(Fiber 노드)로 만들어서 중단/재개가 가능해졌다.

```
[Stack Reconciler - 콜스택 기반]
renderA() → renderB() → renderC() → ...

[Fiber Reconciler - 연결 리스트 기반]
FiberA → FiberB → FiberC → (중단) → 급한 작업 → FiberD → ...
```

Fiber의 연결 구조:

```
      App (Fiber)
      ↓ child
    Header (Fiber) → sibling → Main (Fiber) → sibling → Footer (Fiber)
      ↓ child                    ↓ child
    Logo (Fiber)              Content (Fiber)
```

각 Fiber 노드는 `child`, `sibling`, `return(부모)` 포인터를 가진다.
이 구조 덕분에 트리 순회를 언제든 멈추고 현재 위치를 기억했다가 나중에 재개할 수 있다.

# 7.3 업데이트 예약과 지연

React 18은 상태 업데이트를 긴급(urgent)과 전환(transition)으로 구분한다.

- 긴급 업데이트: 타이핑, 클릭 등 사용자가 즉각적인 반응을 기대하는 것
- 전환 업데이트: 검색 결과 갱신, 페이지 전환 등 약간 늦어도 괜찮은 것

## 업데이트 분리

```jsx
function Search() {
  const [query, setQuery] = useState("");
  const [results, setResults] = useState([]);

  function handleChange(e) {
    // 긴급: 입력값은 즉시 반영
    setQuery(e.target.value);

    // 전환: 검색 결과는 나중에 반영해도 됨
    startTransition(() => {
      setResults(filterItems(e.target.value));
    });
  }

  return (
    <>
      <input value={query} onChange={handleChange} />
      <ResultList items={results} />
    </>
  );
}
```

```
[업데이트 처리 흐름]

사용자 타이핑 "abc"
  ↓
긴급: setQuery('a') → 즉시 렌더링 → input에 'a' 표시
전환: setResults(filter('a')) → 대기
  ↓
긴급: setQuery('ab') → 즉시 렌더링 → input에 'ab' 표시
전환: setResults(filter('a')) → 취소됨 (새 전환이 들어와서)
전환: setResults(filter('ab')) → 대기
  ↓
긴급: setQuery('abc') → 즉시 렌더링 → input에 'abc' 표시
전환: setResults(filter('ab')) → 취소됨
전환: setResults(filter('abc')) → 렌더링 시작
```

전환 업데이트는 새로운 긴급 업데이트가 들어오면 중단되고 폐기될 수 있다. 덕분에 사용자 입력이 항상 즉각 반응한다.

# 7.4 더 깊이 들어가기

## 7.4.1 스케줄러 (Scheduler)

React의 스케줄러는 작업의 우선순위를 관리하고 실행 시점을 결정하는 독립 패키지다.

### 메인 스레드 제어

브라우저의 메인 스레드는 하나뿐이다. JavaScript 실행, DOM 업데이트, 사용자 입력 처리를 모두 이 한 스레드에서 처리한다.

```
[메인 스레드 - 시간 흐름]
|-- JS 실행 --|-- 레이아웃 --|-- 페인트 --|-- JS 실행 --|-- ...
                                         ↑
                                   여기서 사용자 입력을
                                   처리할 수 있어야 함
```

스케줄러는 마이크로 태스크(MessageChannel)를 사용해서 작업을 예약한다.

```js
// 스케줄러의 핵심 개념 (단순화)
const channel = new MessageChannel();

channel.port1.onmessage = () => {
  // 예약된 작업 실행
  // 5ms마다 브라우저에 제어권을 양보
  const deadline = performance.now() + 5;

  while (hasWork() && performance.now() < deadline) {
    performWork();
  }

  if (hasWork()) {
    // 아직 할 일이 남으면 다시 예약
    channel.port2.postMessage(null);
  }
};

// 작업 예약
channel.port2.postMessage(null);
```

# 7.5 렌더 레인 (Render Lanes)

렌더 레인은 React 내부에서 업데이트의 우선순위를 표현하는 비트마스크 시스템이다.

## 렌더 레인 작동 방식

레인은 비트 플래그로 표현된다. 각 업데이트에 레인이 할당되고, React는 레인을 기준으로 어떤 업데이트를 먼저 처리할지 결정한다.

```js
// 레인은 비트마스크로 표현됨 (단순화)
const SyncLane = 0b0000000000000000000000000000010; // 동기
const InputContinuousLane = 0b0000000000000000000000000001000; // 연속 입력
const DefaultLane = 0b0000000000000000000000000100000; // 기본
const TransitionLanes(1-15) = 0b0000000000000000000001000000000; // 전환
// 등등
```

컴포넌트가 업데이트되거나 새 컴포넌트가 렌더 트리에 추가되면, React는 해당 업데이트의 우선순위에 따라 레인을 할당한다. 이후 렌더 레인을 사용해 업데이트를 예약하고 우선순위를 결정한다.

## 레인 처리 흐름

### 1. 업데이트 수집

업데이트가 발생하면 React는 다음 순서로 레인을 결정한다:

1. 업데이트의 콘텍스트 확인 (이벤트 핸들러인지, useEffect인지 등)
2. 콘텍스트에 따라 우선순위 추정 (클릭 → 동기, effect → 기본 등)
3. 우선순위 재정의가 있는지 확인 (`useTransition` 또는 `useDeferredValue`)
4. 올바른 레인에 업데이트 할당

```
[레인 할당 예시]
onClick → setCount(1)           → SyncLane (동기)
onClick → startTransition(...)  → TransitionLane (전환)
useEffect → setState(...)       → DefaultLane (기본)
```

### 2. 레인 처리

할당된 레인 중 가장 높은 우선순위부터 렌더 단계를 시작한다. 렌더 도중 더 높은 우선순위의 업데이트가 들어오면 현재 작업을 중단하고 긴급한 작업을 먼저 처리한다.

### 3. 커밋 단계

렌더링이 완료되면 커밋 단계에서 실제 DOM에 반영한다.

```
[렌더 단계]                    [커밋 단계]
레인에 따라 작업 처리       →   DOM에 반영 (중단 불가)
(중단/재개 가능)                 ↓
                            이펙트 실행 (useEffect 등)
                                 ↓
                            처리 완료된 레인 제거
```

- 커밋 단계는 동기적이다. 한번 시작하면 끝까지 실행된다.
- 커밋이 완료되면 해당 레인이 제거된다.

### 4. 반복

남은 레인이 있으면 다시 렌더 단계로 돌아간다. 이 과정을 모든 레인이 처리될 때까지 반복한다. 이 덕분에 높은 우선순위 업데이트가 낮은 우선순위를 "끼어들기" 할 수 있다.

# 7.6 useTransition

`useTransition`은 상태 업데이트를 전환(transition)으로 표시하는 Hook이다.

## 기본 사용법

```jsx
function TabContainer() {
  const [isPending, startTransition] = useTransition();
  const [tab, setTab] = useState("home");

  function selectTab(nextTab) {
    startTransition(() => {
      setTab(nextTab); // 전환 업데이트로 처리됨
    });
  }

  return (
    <>
      <TabBar
        tabs={["home", "posts", "contact"]}
        selectedTab={tab}
        onSelect={selectTab}
      />
      {isPending && <Spinner />}
      <TabContent tab={tab} />
    </>
  );
}
```

- `startTransition`: 콜백 안의 상태 업데이트를 전환으로 표시
- `isPending`: 전환이 진행 중인지 여부 (로딩 표시에 활용)

## 사용하면 좋은 경우

1. 무거운 리렌더링을 유발하는 상태 변경

```jsx
function SearchPage() {
  const [query, setQuery] = useState("");
  const [isPending, startTransition] = useTransition();

  function handleSearch(e) {
    const value = e.target.value;
    setQuery(value); // 긴급: input 즉시 반영

    startTransition(() => {
      // 전환: 무거운 리스트 렌더링은 나중에
      setFilteredItems(
        items.filter((item) =>
          item.name.toLowerCase().includes(value.toLowerCase()),
        ),
      );
    });
  }

  return (
    <>
      <input value={query} onChange={handleSearch} />
      {isPending ? <Skeleton /> : <ItemList items={filteredItems} />}
    </>
  );
}
```

2. 탭/페이지 전환

```jsx
// 탭을 누르면 이전 탭 내용이 잠깐 유지되다가
// 새 탭 내용이 준비되면 전환됨
startTransition(() => {
  setActiveTab(newTab);
});
```

## useTransition의 동작 원리

```
사용자가 탭 클릭
    ↓
startTransition(() => setTab('posts'))
    ↓
React: "이 업데이트는 전환 레인에 배치"
    ↓
[현재 UI 유지] + [백그라운드에서 새 UI 준비]
    ↓
새 UI 준비 완료 → 한 번에 전환
    ↓
(도중에 다른 긴급 업데이트가 오면 전환 작업 중단)
```

## 주의사항

```jsx
// ❌ startTransition 안에서 동기 작업을 해도 의미 없음
startTransition(() => {
  setCount(count + 1); // 단순한 숫자 변경은 어차피 빠름
});

// ✅ 무거운 렌더링을 유발하는 업데이트에 사용
startTransition(() => {
  setItems(heavyFilter(query)); // 수천 개의 아이템 필터링 + 렌더링
});
```

- `startTransition`의 콜백은 동기적으로 실행된다. 비동기 함수나 `await`을 넣지 않는다.
- 내부의 `setState`가 전환 레인으로 분류되는 것이지, 콜백 자체가 지연 실행되는 것이 아니다.

# 7.7 useDeferredValue

`useDeferredValue`는 값의 업데이트를 지연시키는 Hook이다. 전환과 비슷하지만, 값 자체를 지연시킨다는 점이 다르다.

## 기본 사용법

```jsx
function SearchResults({ query }) {
  // query가 바뀌어도 deferredQuery는 바로 안 바뀜
  // React가 여유가 생기면 그때 업데이트
  const deferredQuery = useDeferredValue(query);

  // deferredQuery가 query와 다르면 "지연 중"
  const isStale = deferredQuery !== query;

  return (
    <div style={{ opacity: isStale ? 0.7 : 1 }}>
      <HeavyList query={deferredQuery} />
    </div>
  );
}
```

## useTransition vs useDeferredValue

|               | useTransition                     | useDeferredValue                    |
| ------------- | --------------------------------- | ----------------------------------- |
| 제어 대상 | 상태 업데이트 (setState)          | 값 자체                             |
| 사용 위치 | 상태를 변경하는 쪽                | 값을 소비하는 쪽                    |
| isPending | 제공됨                            | 직접 비교 (`deferred !== original`) |
| 사용 시점 | setState를 직접 호출할 수 있을 때 | props로 받은 값을 지연시킬 때       |

```jsx
// useTransition - 상태 변경하는 쪽에서 제어
function Parent() {
  const [query, setQuery] = useState("");
  const [isPending, startTransition] = useTransition();

  function handleChange(e) {
    startTransition(() => {
      setQuery(e.target.value);
    });
  }
}

// useDeferredValue - 값을 받는 쪽에서 제어
function Child({ query }) {
  // props로 받은 값을 지연시킴
  // (상태 변경 코드에 접근할 수 없을 때 유용)
  const deferredQuery = useDeferredValue(query);
}
```

`useDeferredValue`는 `React.memo`와 함께 써야 효과가 극대화된다. memo가 없으면 지연된 값이든 아니든 부모가 리렌더링될 때 자식도 리렌더링된다.

# 7.8 동시성 렌더링 관련 문제

## 티어링 (Tearing)

티어링은 하나의 렌더링에서 서로 다른 시점의 데이터가 보이는 현상이다. 동시성 렌더링에서만 발생할 수 있는 새로운 문제다.

### 왜 발생하는가?

동기식 렌더링에서는 렌더링 도중 외부 상태가 바뀌어도 문제없다. 렌더링이 한번에 끝나기 때문이다.

하지만 동시성 렌더링에서는 렌더링이 중단될 수 있다. 중단된 사이에 외부 상태가 바뀌면, 렌더링 전반부와 후반부가 다른 값을 읽게 된다.

```
[티어링 발생 시나리오]

외부 스토어 값: "빨강"

컴포넌트A 렌더링 → "빨강" 읽음
    ↓
(렌더링 중단 - 긴급 업데이트 처리)
    ↓
외부 스토어 값: "빨강" → "파랑" (다른 곳에서 변경)
    ↓
(렌더링 재개)
    ↓
컴포넌트B 렌더링 → "파랑" 읽음

결과: 같은 렌더 사이클에서 A는 "빨강", B는 "파랑"을 보여줌 → 티어링!
```

### 어떤 경우에 발생하는가?

React 외부의 상태 관리를 사용할 때 발생할 수 있다:

```jsx
// ❌ 티어링 위험: 외부 변수를 직접 읽음
let externalStore = { color: "red" };

function ComponentA() {
  // 렌더링 중간에 externalStore가 바뀔 수 있음
  return <div style={{ color: externalStore.color }}>A</div>;
}
```

- 전역 변수를 직접 읽는 경우
- 외부 상태 라이브러리 (Redux, Zustand 등)
- 브라우저 API (window.innerWidth 등)

### 해결: useSyncExternalStore

React 18에서 제공하는 `useSyncExternalStore`는 외부 스토어를 동시성 렌더링에서 안전하게 읽는 방법이다.

```jsx
import { useSyncExternalStore } from "react";

// 외부 스토어
let listeners = [];
let state = { color: "red" };

const store = {
  getSnapshot() {
    return state;
  },
  subscribe(listener) {
    listeners.push(listener);
    return () => {
      listeners = listeners.filter((l) => l !== listener);
    };
  },
};

// ✅ 티어링 방지: useSyncExternalStore 사용
function Component() {
  const { color } = useSyncExternalStore(store.subscribe, store.getSnapshot);
  return <div style={{ color }}>안전함</div>;
}
```

`useSyncExternalStore`는 렌더링 도중 스토어 값이 변경되면 동기적으로 다시 렌더링하여 일관된 값을 보장한다.
