# Week 6. 리액트 동시성

## 7.1 동기식 렌더링의 문제

동기식 렌더링의 문제는 **메인 스레드를 가로막아 사용자 경험이 저하**된다는 것이다.

- 모든 업데이트를 동등하게 취급해 **우선순위 개념이 없다.**
- 보이지 않는 탭, 모달 뒤 콘텐츠, 로딩 상태 등 사용자가 볼 수 없는 항목도 동등하게 렌더링해 메인 스레드를 차단한다.
- batch로 어느 정도 완화할 수 있지만 근본적인 해결책이 되지 않는다.

**동시성 렌더링**: 업데이트의 중요도·긴급도에 따라 우선순위를 정하고, 중요한 업데이트가 덜 중요한 업데이트에 막히지 않도록 한다.

**타임 슬라이싱**: 렌더링 프로세스를 더 작은 덩어리로 분할해 점진적으로 처리 가능.

## 7.2 파이버 다시 보기

**파이버 재조정자**: 동시성 렌더링을 가능하게 리액트 메커니즘

- 렌더링 프로세스를 **파이버(fiber)** 라는 더 작고 관리하기 쉬운 작업 단위로 분할한다.
- 렌더링 작업을 일시 중지, 재개, 우선순위 설정을 할 수 있어 덜 중요한 작업이 중요한 업데이트를 가로막는 일을 방지한다.


## 7.3 업데이트 예약과 지연

파이버 재조정자는 **스케줄러**와 여러 API에 의존해 업데이트 예약(schedule)·지연(defer)을 구현한다.

스케줄러: `setTimeout`, `MessageChannel` 등의 브라우저 API를 사용해 작업을 예약·관리하는 시스템.

### useTransition 적용

수신 메시지 렌더링을 낮은 우선순위로 처리해 사용자 입력을 방해하지 않는다.

```tsx
const ChatApp = () => {
  const [messages, setMessages] = useState<string[]>([]);
  const [isPending, startTransition] = useTransition();

  useEffect(() => {
    const socket = new WebSocket("wss://your-websocket-server.com");
    socket.onmessage = (event) => {
      // 메시지 추가를 낮은 우선순위로 처리 → 사용자 입력 방해 안 함
      startTransition(() => {
        setMessages((prev) => [...prev, event.data]);
      });
    };
    return () => socket.close();
  }, []);

  return (
    <div>
      <MessageList messages={messages} />
      <MessageInput onSubmit={sendMessage} /> {/* 사용자 입력은 높은 우선순위 유지 */}
    </div>
  );
};
```

`startTransition`으로 감싼 업데이트는 **트랜지션 레인**에 배치되어 낮은 우선순위로 처리된다.

## 7.4 스케줄러

스케줄러는 타이밍 관련 유틸리티를 제공하는 **독립형 패키지**로, 파이버 재조정자와 별개로 동작한다.

**마이크로태스크를 예약**해 메인 스레드의 제어를 관리.

### 이벤트 루프와 태스크 큐

| 구분 | 설명 |
|------|------|
| **매크로태스크 큐** | 이벤트 처리, `setTimeout`, `setInterval`, I/O 작업 등. 한 번에 하나씩 처리. |
| **마이크로태스크 큐** | Promise, `MutationObserver` 등. 현재 매크로태스크가 끝난 직후, 다음 매크로태스크 전에 모두 처리. |

마이크로태스크를 통해 UI 업데이트를 최우선으로 처리.

### 스케줄러의 레인 기반 예약

```ts
if (nextLane === Sync) {
  queueMicrotask(processNextLane); // 바로 처리
} else {
  Scheduler.scheduleCallback(callback, processNextLane); // 나중에 처리
}
```

## 7.5 렌더 레인

**렌더 레인** 은 리액트 18에서 도입된 개념으로, 업데이트의 우선순위를 나타내는 단위다. (이전에는 만료 시간 방식 사용)

### 주요 레인 종류

| 레인 | 설명 |
|------|------|
| `SyncHydrationLane` | 하이드레이션 중 클릭 이벤트 |
| `SyncLane` | 일반 클릭 이벤트 (최고 우선순위) |
| `InputContinuousLane` | 호버, 스크롤 등 연속 이벤트 |
| `DefaultLane` | 네트워크 업데이트, `setTimeout`, 초기 렌더링 |
| `TransitionLanes` | `startTransition` 내 업데이트 |
| `RetryLanes` | Suspense 재시도 |

### 업데이트 처리 흐름

1. **업데이트 수집** — 마지막 렌더링 이후 예약된 모든 업데이트를 레인에 할당
2. **레인 처리** — 우선순위 높은 레인부터 일괄 처리
3. **커밋 단계** — 변경 사항을 DOM에 적용, 효과(effect) 실행
4. **반복**

### 우선순위 결정 방법

1. 업데이트 context 확인 (사용자 상호 작용 / 내부 업데이트 / 서버 응답)
2. context에 따라 우선순위 추정
3. 개발자가 `useTransition` / `useDeferredValue`로 명시적 재정의 여부 확인
4. 비트마스크로 올바른 레인에 할당

## 7.6 useTransition

`useTransition`:  **상태 업데이트의 우선순위를 관리**하는 훅

```tsx
const [isPending, startTransition] = useTransition();
```

| 반환값 | 설명 |
|--------|------|
| `isPending` | 트랜지션이 진행 중인지 나타내는 불리언 |
| `startTransition` | 낮은 우선순위로 처리할 업데이트를 감싸는 함수 |

`startTransition`으로 감싼 업데이트는 **트랜지션 레인**(Sync 레인보다 낮은 우선순위)에 배치된다.

### 예시

```tsx
import { useState, useTransition } from "react";

function App() {
  const [count, setCount] = useState(0);
  const [isPending, startTransition] = useTransition();

  const handleClick = () => {
    doSomethingImportant(); // 높은 우선순위 작업 먼저
    // count 업데이트는 낮은 우선순위로 → 중요한 작업을 먼저 
    startTransition(() => {
      setCount((c) => c + 1);
    });
  };

  return (
    <div>
      <p>Count: {count}</p>
      <button onClick={handleClick}>증가</button>
      {isPending && <p>로딩 중...</p>} 
    </div>
  );
}
```

```tsx
// SPA 페이지 전환
import { useState, useTransition } from "react";

const PageOne = () => <div>Page One</div>;
const PageTwo = () => <div>Page Two</div>;

function App() {
  const [currentPage, setCurrentPage] = useState("pageOne");
  const [isPending, startTransition] = useTransition();

  const handleNavigation = (page: string) => {
    // 페이지 전환을 낮은 우선순위로 → 네비게이션 버튼 클릭은 바로 반응
    startTransition(() => {
      setCurrentPage(page);
    });
  };

  const renderPage = () => {
    switch (currentPage) {
      case "pageOne": return <PageOne />;
      case "pageTwo": return <PageTwo />;
      default: return <div>Unknown page</div>;
    }
  };

  return (
    <div>
      <nav>
        <button onClick={() => handleNavigation("pageOne")}>Page One</button>
        <button onClick={() => handleNavigation("pageTwo")}>Page Two</button>
      </nav>
      {isPending && <p>로딩 중...</p>}
      {renderPage()}
    </div>
  );
}
```

페이지 전환 중 Suspense로 데이터를 가져오는 경우, 데이터·렌더링이 완료될 때까지 `isPending`이 `true`로 유지된다.

> 다음 페이지의 데이터 페치가 `useEffect` 내에서 발생하면 `startTransition`은 데이터를 다 가져올 때까지 기다리지 않는다. Suspense와 함께 사용해야 `isPending`이 올바르게 동작한다.

### startTransition

`useTransition` 훅을 사용할 수 없는 환경에서는 리액트 패키지에서 직접 `startTransition`을 가져올 수 있지만 `isPending` 플래그를 사용할 수 없다.

```tsx
import { startTransition } from "react";

// 함수 컴포넌트 밖에서도 사용 가능하지만 isPending 사용 불가
startTransition(() => {
  setState(newValue);
});
```

## 7.7 useDeferredValue

`useDeferredValue`는 특정 값의 업데이트를 **나중으로 미루는** 훅이다. 새 값이 도착하기까지 이전 값을 유지해 응답성을 높인다.

```tsx
const deferredValue = useDeferredValue(value);
```

```tsx
function useDeferredValue(value) {
  const [newValue, setNewValue] = useState(value);
  useEffect(() => {
    // 값이 변경되면 낮은 우선순위로 업데이트
    startTransition(() => {
      setNewValue(value);
    });
  }, [value]);
  return newValue; // 이전 값을 유지하다가 나중에 업데이트
}
```

### 예시

```tsx
// 검색 결과 지연
import { memo, useState, useDeferredValue } from "react";

function App() {
  const [searchValue, setSearchValue] = useState("");
  const deferredSearchValue = useDeferredValue(searchValue); // 검색어 업데이트 지연

  return (
    <div>
      <input
        type="text"
        value={searchValue} // 입력 필드는 바로 반응
        onChange={(e) => setSearchValue(e.target.value)}
      />
      {/* 검색 결과 렌더링은 낮은 우선순위 → 타이핑 방해 안 함 */}
      <SearchResults searchValue={deferredSearchValue} />
    </div>
  );
}

const SearchResults = memo(({ searchValue }: { searchValue: string }) => {
  // 검색 결과 렌더링 (비용이 큰 작업)
});
```

- `memo`로 `SearchResults`를 감싸 불필요한 리렌더링 방지
- `deferredSearchValue`가 낮은 우선순위로 업데이트되므로, 입력 필드가 먼저 반응

```tsx
// 대규모 목록 필터링
import { memo, useState, useMemo, useDeferredValue } from "react";

function App() {
  const [filter, setFilter] = useState("");
  const deferredFilter = useDeferredValue(filter);

  const items = useMemo(() => generateLargeList(), []);
  // 필터링 연산은 낮은 우선순위로 처리 → 입력 필드 먼저 반응
  const filteredItems = useMemo(
    () => items.filter((item) => item.includes(deferredFilter)),
    [items, deferredFilter]
  );

  return (
    <div>
      <input
        type="text"
        value={filter}
        onChange={(e) => setFilter(e.target.value)}
      />
      <ItemList items={filteredItems} />
    </div>
  );
}

const ItemList = memo(({ items }: { items: string[] }) => {
  // 아이템 목록 렌더링
});
```

### useDeferredValue vs 디바운싱/스로틀링

| 구분 | 디바운싱/스로틀링 | useDeferredValue |
|------|------------------|-----------------|
| 지연 시간 | 개발자가 직접 설정 | 기기 성능에 맞게 동적 조정 |
| 렌더링 중단 | 렌더링 도중 사용자 입력 가로막을 수 있음 | 렌더링을 중단하고 새 입력에 먼저 응답 |
| 사용 목적 | 네트워크 요청 빈도 제한 등 렌더링 외 시나리오에도 적합 | 렌더링 최적화 특화 |

두 기법은 **병행 사용**으로 더 큰 최적화 효과를 얻을 수 있다.

### 사용 시기

- 대규모 데이터 검색·필터링
- 복잡한 시각화·애니메이션 렌더링
- 백그라운드에서 서버 데이터 업데이트
- 사용자 상호 작용에 영향 줄 수 있는 연산 집약적 작업

### 사용하지 말아야 할 경우

> "이 업데이트가 사용자 입력에 의한 것인가?"

사용자가 즉각적인 반응을 기대하는 업데이트는 지연하면 안 된다. 지연된 데이터가 표시될 경우의 UX 영향을 반드시 고려해야 한다.

## 7.8 동시성 렌더링 관련 문제 — 티어링

### 티어링(Tearing)이란?

동시성 렌더링 환경에서 **렌더링 도중 외부 상태가 변경**되면, 같은 값을 참조하는 여러 컴포넌트 인스턴스가 **서로 다른 값으로 렌더링**될 수 있다. 이를 **티어링**이라 한다.

```tsx
// 외부 상태 — 리액트 렌더링 주기와 무관하게 업데이트됨
let count = 0;
setInterval(() => count++, 1);

const ExpensiveComponent = () => {
  const now = performance.now();
  while (performance.now() - now < 100) {} // 렌더링에 100ms 소요
  // 렌더링 도중 count가 계속 변경됨 → 인스턴스마다 다른 값 읽음
  return <>비용이 비싼 카운트: {count}</>;
};
```

위 예시를 실행하면 다섯 개의 `ExpensiveComponent` 인스턴스가 각각 다른 `count` 값으로 렌더링될 수 있다.

```
비용이 비싼 카운트: 568
비용이 비싼 카운트: 568
비용이 비싼 카운트: 569
비용이 비싼 카운트: 569
비용이 비싼 카운트: 570
```

더 심각한 경우, 렌더링 도중 전역 저장소에서 사용자가 삭제되면 오류가 발생할 수 있다.

```tsx
<UserDetails id={user.id} /> {/* 렌더링 도중 user가 삭제되면 오류! */}
```

### useSyncExternalStore

`useSyncExternalStore`: 외부 상태와 리액트 내부 상태를 **동기적으로 동기화**하는 훅.

```tsx
const value = useSyncExternalStore(store.subscribe, store.getSnapshot);
```

| 인수 | 설명 |
|------|------|
| `store.subscribe(callback)` | 외부 저장소 변경 시 `callback` 호출. 구독 해제 함수 반환. |
| `store.getSnapshot()` | 외부 저장소의 현재 값을 동기적으로 반환. 여러 인스턴스에서 동일한 값 보장. |

### 예시

```tsx
// 창 크기 구독
const store = {
  subscribe(rerender: () => void) {
    // 창 크기 변경 시 리렌더링 트리거
    window.addEventListener("resize", rerender);
    return () => window.removeEventListener("resize", rerender);
  },
  getSnapshot() {
    // 현재 창 크기를 동기적으로 반환
    return { width: window.innerWidth, height: window.innerHeight };
  },
};

// 외부 상태(window 크기)를 리액트와 동기화
const { width, height } = useSyncExternalStore(
  store.subscribe,
  store.getSnapshot
);
```

```tsx
// 티어링 해결 예시
import { useState, useSyncExternalStore, useTransition } from "react";

let count = 0;
setInterval(() => {
  count++;
  store.callbacks.forEach((cb) => cb()); // 모든 구독자에게 변경 알림
}, 1);

const store = {
  callbacks: [] as Array<() => void>,
  subscribe(syncRerender: () => void) {
    store.callbacks.push(syncRerender);
    return () => {
      store.callbacks = store.callbacks.filter((fn) => fn !== syncRerender);
    };
  },
  getSnapshot() {
    return count; // 현재 count 값을 동기적으로 반환
  },
};

const ExpensiveComponent = () => {
  // 훅을 통해 외부 상태를 읽음 → 모든 인스턴스가 동일한 값 보장
  const consistentCount = useSyncExternalStore(
    store.subscribe,
    store.getSnapshot
  );

  const now = performance.now();
  while (performance.now() - now < 100) {}

  return <>비용이 비싼 카운트: {consistentCount}</>;
};
```

`useSyncExternalStore`의 중요 기능:

1. **동시적 렌더링에서 일관된 상태 보장**
2. **저장소가 변경될 때 동기적으로 리렌더링 강제 적용**
