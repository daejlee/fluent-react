파이버 재조정자에 대해 더 자세히 알아보고, 리액트의 동시성 기능, 업데이트와 렌더링을 효율적으로 관리하는 방법을 배운다. 스케줄링, 지연 업데이트, 렌더 레인을 살펴보고 리액트의 코어 아키텍처 덕에 가능해진 성능 최적화에 대한 안목을 얻을 것이다.

# 7.1 동기식 렌더링의 문제

동기식 렌더링은 메인 스레드를 블로킹해 사용자 경험을 저해할 수 있다. 이를 완화하기 위해 여러 업데이트를 일괄 처리하는 방법이 있었다. (4장)

동기식 렌더링에는 우선순위가 없기에 일괄 처리해봤자 문제가 복잡해질 분이다. 사용자가 볼 수 있고 상호 작용 가능한 항목이 우선 렌더링되어야한다.

리액트는 동시성 렌더링으로 업데이트 작업의 중요도와 긴급도에 따라 우선순위를 정해 블로킹을 최소화한다. 또한 동시성 렌더링으로 리액트는 타임 슬라이싱, 렌더링 프로세스를 더 작은 덩어리로 분할해 점진적으로 처리하는 기법도 가능하다.

# 7.2 파이버 다시 보기

파이버 재조정자는 렌더링 프로세스를 파이버라는 더 작고 관리가 쉬운 작업 단위로 분할해 처리한다. 이를 통해 덜 중요한 작업이 더 중요한 작업을 가로막는 일을 방지한다.

# 7.3 업데이트 예약과 지연

리액트에서 업데이트를 예약(schedule)하고 지연(defer)하는 기능은 앱 응답성을 유지하는 데 매우 중요하다.

스케줄러는 업데이트 작업이 있을 때 `setTimeout`, `MessageChannel`등의 브라우저 API로 작업을 스케줄하는 시스템이다.

채팅 애플리케이션을 예시로 들 때, 메시지 입력, 전송같은 상호 작용을 우선시하고 새로 들어오는 메시지를 효율적으로 렌더링하는 것이 중요하다.

서버에서 새 메시지가 도착해 렌더링할 때, 리액트는 렌더 레인을 통해 렌더링하는데 이 레인은 동기적으로 DOM을 업데이트한다. 이 작업의 우선순위를 낮추려면 관련 상태 업데이트를 `useTransition` 훅의 `startTransition` 함수로 감싸면 된다.

```tsx
const ChatApp = () => {
	const [messages, setMessages] = useState([]);
	const [isPending, startTransition] = useTransition();

	useEffect(() => {
		const socket = new WebSocket(...);
		socket.onmessage = (event) => {
			startTransition(() => {
				setMessages((prevMessages) => [...prevMessages, event.data]);
			}});
		};

		return () => {
			socket.close();
		};
	}, []);

	...
}
```

# 7.4 더 깊이 들어가기

해당 주제를 더 깊이 이해하기 위해 스케줄러, 작업의 우선순위 단계, 업데이트 지연 메커니즘 등의 핵심 개념을 살펴보자.

## 7.4.1 스케줄러

타이밍 관련 유틸리티를 제공하는 독립형 패키지다. 파이버 재조정자와는 별개로 동작한다. 스케줄러와 재조정자는 렌더 레인을 통해 긴급도에 따라 우선순위를 설정한다. 스케줄러의 주된 기능은 마이크로태스크를 예약해 메인 스레드의 제어를 관리하는 것이다.

스케줄러는 작업이 속하는 렌더 레인을 기반으로 작업을 예약한다.

```tsx
if (nextLane === Sync) {
  // 다음 레인이 Sync면 마이크로태스크를 대기열에 추가한다.
  queueMicrotask(processNextLane);
} else {
  // 다음 레인이 Sync가 아니면 콜백을 예약하고 다음 레인을 처리한다.
  Scheduler.scheduleCallback(callback, processNextLane);
}
```

# 7.5 렌더 레인

렌더 레인은 작업의 렌더링과 우선순위 관리를 효율화한다. 레인은 우선순위 수준을 나타내는 단위로, 리액트 렌더링 주기에서 처리할 수 있는 작업을 의미한다.

예를 들어, setState를 호출할 때 해당 업데이트가 레인으로 보내진다.

- setState가 클릭 핸들러 내부에서 호출될 시 우선순위가 가장 놓은 Sync 레인에 배정되고 마이크로태스크로 예약된다
- setState가 startTransition의 트랜지션 내에서 호출되면 우선순위가 낮은 트랜지션 레인에 배치되고 마이크로태스크로 예약된다.

## 7.5.1 렌더 레인 작동 방식

컴포넌트가 업데이트되거나 새 컴포넌트가 렌더 트리에 추가되면 리액트에서 해당 업데이트의 우선순위에 따라 레인을 할당한다. 이후 다음 방식으로 렌더 레인을 사용해 업데이트를 예약하고 우선순위를 정한다.

1. 업데이트 수집: 마지막 렌더링 이후에 예약된 모든 업데이트를 수집해 우선순위에 따라 각 레인에 할당
2. 레인 처리: 우선순위가 높은 레인부터 시작해 각 레인의 업데이트를 일괄 처리
3. 커밋 단계: 업데이트 처리 후 커밋
4. 반복

업데이트가 발생하면 리액트는 다음 단계를 수행해 우선순위를 결정하고 적절한 레인에 할당한다.

1. 업데이트의 컨텍스트 확인: 사용자 상호작용, 프롭 변경, 서버 응답 등의 컨텍스트를 평가
2. 컨텍스트에 따라 우선순위 추정
3. 우선순위 재정의 확인: useTransition 또는 useDefferedValue 훅으로 명시적으로 우선순위를 지정했는지 체크
4. 올바른 레인에 업데이트 할당

## 7.5.2 레인 처리

레인에 업데이트가 할당되면 우선순위에 따라 업데이트를 처리한다. 채팅 어플리케이션의 예시에선 아래와 같다.

- `ImmediatePriority`: 메시지 입력에 대한 업데이트
- `UserBlockingPriority`: 입력 상태 표시에 대한 업데이트
- `NormalPriority`: 메시지 목록에 대한 업데이트

## 7.5.3 커밋 단계

모든 업데이트를 각 레인에서 처리하고, 리액트는 커밋 단계로 진입해 변경 사항을 DOM에 적용하고 사이드이펙을 실행한다.

두 레인을 함께 처리하는 시점을 결정하는 얽힘(entanglement) 개념이나 이미 처리된 업데이트에 새 업데이트를 다시 적용할 시점을 결정하는 리베이스(rebase) 개념도 존재한다. 리베이스는 트랜지션 완료 전 동기 업데이트가 발생해 트랜지션이 중단된 경우처럼, 두 업데이트를 함께 실행할 때 유용하다.

동기식 업데이트가 발생할 때 리액트는 업데이트 전후에 대기 중인 효과를 한번에 실행하고 처리해(flushing) 동기식 업데이트 간 일관성을 보장한다.

리액트는 우선순위를 추정하는데 능숙하지만 항상 완벽하진 않아 개발자가 API를 이용해 기본값으로 설정된 우선순위를 재정의해야 할 때도 존재한다.

# 7.6 useTransition

상태 업데이트의 우선순위를 관리, 우선순위가 높은 업데이트로 인한 UI 미응답을 방지하는 훅이다. `useTransition`에서 반환된 `startTransition` 함수로 감싼 업데이트는 트랜지션 레인에 들어간다.

해당 라인은 우선 순위가 낮다. 이 점을 이용해 높은 우선순위의 업데이트가 메인 스레드에서 경쟁할 때 사용자 경험이 원활하다.

## 7.6.2 고급 예시: 페이지 탐색

useTransition은 페이지 이동시에도 유용하다.

```tsx
function App() {
	const [currentPage, setCurrentPage] = useState("pageOne");
	const [isPending, startTransition] = useTransition();

	const handleNavigation = (page) => {
		startTransition(() => {
			setCurrentPage(page);
		});
	};

	const renderPage = () => {
		switch (currentPage) {
			...
		}
	};

	return (
		<div>
			<nav>
				<button onClick={() => handleNavigation("pageOne")}>Page One</button>
				<button onClick={() => handleNavigation("pageTwo")}>Page Two</button>
			</nav>
			{isPending && "loading.."}
			{renderPage()}
		</div>
	);
}
```

페이지를 변경하는 상태 업데이트를 useTransition으로 감싸 사용자 입력 같은 우선순위가 높은 업데이트가 발생하는 경우 전환이 지연된다.

<aside>

    💡 useTransition으로 렌더링 우선순위를 조정한다면, 트랜지션 레인에 배정된 (우선순위가 낮아진) 상태 업데이트 건은 계속해서 렌더링 순위에서 밀릴 수도 있겠다고 생각했습니다.

    메인 스레드를 점유하지 못하는 기아 상태

    이를 방지하기 위해 리액트는 유효 기간을 설정한다고 합니다.

    - 모든 업데이트는 생성될 때 만료 시간(expiration time)이 할당됩니다.
    - 낮은 우선순위 작업이라도 스케줄러에서 계속 밀리다 보면, 결국 이 만료 시간에 도달하게 됩니다.
    - 만료된 작업은 즉시 동기적(Sync)인 최고 우선순위로 격상됩니다.
    - 이때부터는 메인 스레드를 점유하여 사용자 입력보다도 먼저 UI를 강제로 렌더링해 버립니다.

</aside>

# 7.7 useDeferredValue

UI 업데이트를 나중으로 미루는데 사용되는 훅이다. 초기 렌더링 중에 반환되는 지연된(deffered) 값은 인수로 전달된 값과 동일하고, 이후 업데이트에서는 `useDefferedValue`가 오래된 값을 더 오래 유지, 새 값으로 업데이트할 시점을 제어해 부드러운 사용자 경험을 유지한다.

값이 바뀌어도 UI가 매번 새롭게 리렌더링되지 않는다. 새 값으로 업데이트될 시점을 제어해 한 번에 새 값으로 업데이트되게 한다(stale-while-revalidate).

```tsx
function useDeferredValue(value) {
  const [newValue, setNewValue] = useState(value); // 초기값 저장

  useEffect(() => {
    startTransition(() => {
      // 긴급한 업데이트 먼저 처리
      setNewValue(value);
    });
  }, [value]);

  return newValue;
}
```

## 디바운싱과 스로틀링과 다른 점은?

- 디바운싱: 이벤트가 완료될 때까지 기다린 후 업데이트
- 스로틀링: 일정한 간격으로 업데이트, 이를테면 1초에 한 번

위 방식은 개발자가 임의로 지연 시간을 설정하지만 `useDeferredValue`는 지연 시간을 기기 성능에 맞춰 조절해 임의로 설정할 필요가 없다.

고성능 기기에서는 렌더링 지연이 거의 없이 즉각적으로 이루어진다.

## 7.7.2 useDeferredValue 사용 시기

앱이 특정 업데이트의 우선순위를 다른 업데이트보다 우선시해야 하는 상황에서 유리하다.

- 대규모 데이터를 검색하거나 필터링
- 복잡한 시각화나 애니메이션을 렌더링
- 백그라운드에서 서버의 데이터를 업데이트
- 사용자 상호 작용에 영향을 미치는 연산 집약적 작업

```tsx
function App() {
  const [filter, setFilter] = useState('');
  const deferredFilter = useDeferredValue(filter);

  const filteredItems = useMemo(() => {
    return items.filter((item) => item.includes(deferredFilter));
  }, [items, deferredFilter]);

  return (
    <div>
      <input
        value={filter}
        onChange={(event) => setFilter(event.target.value)}
      />
      <ItemList items={filteredItems} />
    </div>
  );
}
```

위 예시에서 사용자가 필터 텍스트를 입력할 때, 지연된 값이 덜 자주 업데이트되므로 사용자 입력을 우선 처리해 응답성을 유지할 수 있다. 극단적인 예시로 10만개의 아이템을 필터링하는 케이스에서,

1. 사용자가 input에 값 입력
2. 필터링 시작.. (10초 걸림) → 이동안 사용자가 입력을 쳐도 화면에 반영 안됨 싱글스레드라
3. 10초 후 입력 반영

이런 케이스를 방지한다는 것이다. 그럼 위에서 물었던 것 처럼 끝도 없이 지연되느냐? 그렇진 않다. 다만 지연이 지속되면 화면이 멈춘 것 같이 보일 수 있어서, 아래와 같이 isStale 처리가 가능하다.

```tsx
const isStale = filter !== deferredFilter;

return (
  <div
    style={{
      opacity: isStale ? 0.7 : 1,
      transition: 'opacity 0.2s ease',
      filter: isStale ? 'grayscale(50%)' : 'none',
    }}
  >
    <ItemList items={filteredItems} />
    {isStale && <p>업데이트 중...</p>}
  </div>
);
```

<aside>
  
	💡 useTransition과 useDeferredValue 둘 다 중요하지 않은 업데이트를 지연시키는데,
	둘의 차이는? “상태”와 “상태 값”의 차이다.
</aside>

| **구분**      | **useTransition**                      | **useDeferredValue**                   |
| ------------- | -------------------------------------- | -------------------------------------- |
| **제어 대상** | 상태 업데이트 함수 (`startTransition`) | 상태 값 (`value`)                      |
| **주요 용도** | UI 전환, 무거운 상태 변경 직후         | 사용자 입력에 따른 결과 지연 (검색 등) |
| **상태 확인** | `isPending` 제공 (내장)                | 직접 비교 필요 (`val !== defVal`)      |
| **사용 위치** | 상태 업데이트를 일으키는 곳            | 업데이트된 값을 소비하는 곳            |

# 7.8 동시성 렌더링 관련 문제

동시성 렌더링은 성능과 응답성을 높인다, 하지만 예기치 않은 동작과 버그를 유발할 수 있다. 티어링 현상은 업데이트 순서가 어긋나며 UI가 일관성을 잃게 되는 문제를 가르킨다. 이 현상은 렌더링 중 업데이트가 발생하여 컴포넌트가 일관되지 않은 데이터로 렌더링되며 발생할 수 있다.

## 7.8.1 티어링

렌더링하는 컴포넌트가 의존하고 있는 상태가 업데이트될 때 발생하는 버그다. 동기식 렌더링에서는 한 컴포넌트씩 차례로 렌더링해 일관성을 유지하지만, 동시성 모드에선 그렇지 않다.

```tsx
// 1. 리액트 외부에서 관리되는 상태 (External Store)
let count = 0;
setInterval(() => {
  count++;
}, 1);

export default function App() {
  const [name, setName] = useState('');
  const [isPending, startTransition] = useTransition();

  const updateName = (newVal) => {
    // startTransition으로 감싸서 우선순위를 낮춤 (동시성 렌더링 유도)
    startTransition(() => {
      setName(newVal);
    });
  };

  return (
    <div>
      <input
        value={name}
        onChange={(e) => updateName(e.target.value)}
        placeholder='입력 시 티어링 발생 가능'
      />
      {isPending && <div>로딩 중... (동시성 렌더링 발생)</div>}
      <ul>
        {/* 무거운 컴포넌트들이 렌더링되는 동안 외부 count 값이 계속 변함 */}
        <li>
          <ExpensiveComponent />
        </li>
        ...
        <li>
          <ExpensiveComponent />
        </li>
      </ul>
    </div>
  );
}

// 2. 렌더링 비용이 매우 큰 컴포넌트
const ExpensiveComponent = () => {
  const now = performance.now();

  // 100ms 동안 의도적으로 CPU를 점유 (블로킹)
  while (performance.now() - now < 100) {
    // 아무것도 하지 않고 대기
  }

  // 렌더링 시점에 외부 변수인 count를 직접 참조
  return <li>비용이 비싼 카운트: {count}</li>;
};
```

count 전역 변수는 리액트 렌더링 주기와 관계없이 업데이트된다. 1초에 한번씩 count를 업데이트하는데, 렌더링 동안 count가 변경되어 티어링 버그가 생길 수 있다.

사용자 입력에 따라 리액트가 렌더링을 ‘중지’하고 입력 필드 업데이트를 우선 처리하면 `ExpensiveComponent`는 더 오래된 count로 렌더링될 수 있다.

먼저 렌더링된 인스턴스는 변경 전의 count로 렌더링되고, 해당 값이 DOM에 커밋된 후 나머지 인스턴스가 렌더링되며 업데이트된 count 값을 사용했기 때문이다.

| tearing | no tearing |
| ------- | ---------- |
| ![리액트 티어링](https://github.com/user-attachments/assets/ee155eac-daac-4402-8ae2-a733b26f0b23) | ![리액트 티어링 동시성 x](https://github.com/user-attachments/assets/f04e0961-bae9-4412-aad4-584b5bd783fd) |

티어링 문제 해결을 위해 리액트에서 useSyncExternalStore 훅을 제공한다.

### useSyncExternalStore

앱 내부 상태와 외부 상태를 동기화하는 리액트 훅이다. useSyncExternalStore의 ‘sync’에는 synchronize(동기화)와 synchronous(동기적)이라는 의미를 둘 다 가진다.

외부 저장소(store)에 변화가 발생할 때 동기적으로 업데이트를 강제해 일관성을 유지한다는 뜻이다.

```tsx
const value = useSyncExternalStore(store.subscribe, store.getSnapshot);
```

- store.subscribe: 콜백 함수를 받는다. 외부 저장소의 변경 사항을 구독하고 store에 변경이 생길때마다 콜백 함수를 호출한다.

```tsx
// 브라우저 창 크기가 조정될 때마다 리액트 컴포넌트 리렌더링
const store = {
	subscribe(renderImmediately) {
		window.addEventListener("resize", rerenderImmediately);
		return () => {
			window.removeEventListener("resize", rerenderImmediately):
		};
	},
};
```

- store.getSnapshot: 외부 함수의 현재값을 반환한다. 렌더링때마다 호출되고, 반환값은 내부 상태를 업데이트하는데 사용된다. 동기적으로 호출한다.

```tsx
// 브라우저 창 크기가 조정될 때마다 리액트 컴포넌트 리렌더링
const store = {
	subscribe(immediatelyRerenderSynchronously) {
		window.addEventListener("resize", immediatelyRerenderSynchronously);
		return () => {
			window.removeEventListener("resize", immediatelyRerenderSynchronously):
		};
	},
	getSnapshot() {
		return {
			width: window.innerWidth,
			height: window.innerHeight,
		}
	},
};
```
