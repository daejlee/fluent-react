# 재조정

## 4.1 재조정 이해하기

재조정(Reconciliation)은 React가 화면을 효율적으로 업데이트하는 핵심 알고리즘입니다.

### 재조정의 4단계

1. **상태 변경 감지**: 사용자 입력, API 응답 등으로 상태(state)가 변경
2. **새로운 가상 DOM 생성**: 변경된 상태 기반으로 새 가상 DOM 트리 생성 (메모리에서만)
3. **비교(Diffing)**: 이전 가상 DOM과 새 가상 DOM을 비교하여 변경 사항 파악
4. **실제 DOM 업데이트(Commit)**: 변경된 부분만 실제 DOM에 반영

### 효율성의 비밀

```
사용자 버튼 클릭 → 상태 변경 → 새 가상 DOM 생성 → Diffing → 변경된 부분만 DOM 업데이트
```

전체 페이지를 다시 그리지 않고 **변경된 부분만** 업데이트하므로 빠릅니다.

## 4.2 배치 처리

React는 여러 상태 변경을 한 번에 모아서 처리합니다.

```jsx
// 이 세 개의 setState가 있어도
setCount(count + 1);
setName("Kenny");
setAge(25);

// React는 한 번에 모아서 처리 → 렌더링도 한 번만!
```

### 배치 처리의 장점

- **성능 향상**: 불필요한 렌더링 방지
- **일관성 유지**: 모든 상태가 동시에 업데이트되어 중간 상태 노출 방지
- **효율적인 DOM 업데이트**: 한 번의 DOM 조작으로 여러 변경사항 반영

## 4.3 기존 기술

React 16 이전에는 **스택 재조정자(Stack Reconciler)**를 사용했습니다.

### 4.3.1 스택 재조정자

스택 재조정자는 재귀 함수를 사용하여 동작했습니다.

#### 특징

- **동기적 처리**: 재조정이 시작되면 끝날 때까지 멈출 수 없음
- **호출 스택 사용**: 재귀 함수로 트리를 순회
- **단순한 구조**: 이해하기 쉽지만 유연성이 부족

#### 한계점

```
큰 컴포넌트 트리 업데이트 시작
  ↓
16ms 이상 걸림 (브라우저는 60fps를 위해 16ms마다 프레임 그려야 함)
  ↓
중단 불가! 끝날 때까지 대기
  ↓
애니메이션 끊김, 입력 지연 발생 😢
```

## 4.4 파이버 재조정자

React 16부터 도입된 새로운 재조정 엔진입니다. **작업을 중단하고 재개할 수 있는** 것이 핵심입니다.

### 4.4.1 데이터 구조로서의 파이버

파이버(Fiber)는 컴포넌트의 작업 단위를 나타내는 자바스크립트 객체입니다.

```js
// 파이버 노드의 주요 속성
{
  type: 'div',           // 컴포넌트 타입
  key: null,             // 고유 키
  props: {...},          // 속성
  stateNode: DOMNode,    // 실제 DOM 노드

  // 트리 구조
  child: Fiber,          // 첫 번째 자식
  sibling: Fiber,        // 다음 형제
  return: Fiber,         // 부모

  // 작업 관련
  alternate: Fiber,      // 더블 버퍼링용 반대편 파이버
  effectTag: 'UPDATE',   // 어떤 작업이 필요한지
  nextEffect: Fiber,     // 다음에 처리할 effect
}
```

### 4.4.2 더블 버퍼링

React는 **두 개의 파이버 트리**를 유지합니다.

```
현재 트리 (Current Tree)          작업 트리 (WorkInProgress Tree)
      ↓                                    ↓
  화면에 표시된 상태                    작업 중인 상태

      ↕ alternate 포인터로 연결 ↕
```

#### 동작 방식

1. **현재 트리**: 화면에 렌더링된 현재 상태
2. **작업 트리**: 새로운 업데이트를 적용하여 구성 중인 트리
3. **스왑**: 작업이 완료되면 작업 트리가 현재 트리로 전환
4. **재사용**: 이전 현재 트리는 다음 작업 트리로 재사용

**장점**: 메모리 할당/해제를 줄여 성능 향상!

### 4.4.3 파이버 재조정

파이버 재조정은 **렌더 단계**와 **커밋 단계**로 나뉩니다.

## 렌더링 단계

렌더링 단계는 **비동기적**으로 동작하며, 중단하고 재개할 수 있습니다. 실제 DOM을 변경하지 않습니다.

### beginWork와 시그니처

```typescript
function beginWork(
  current: Fiber | null, // 현재 파이버 (없으면 새로 생성)
  workInProgress: Fiber, // 작업 중인 파이버
  renderLanes: Lanes, // 우선순위
): Fiber | null;
```

**역할**: 파이버 트리를 **위에서 아래로** 순회하며 작업 수행

- 컴포넌트 함수/클래스 실행
- 새로운 props와 state로 자식 생성
- Diff 알고리즘 적용
- effect 태그 설정 (UPDATE, PLACEMENT, DELETION 등)

**흐름**: 루트 → 자식 → 손자 순으로 내려감

### completeWork와 시그니처

```typescript
function completeWork(
  current: Fiber | null,
  workInProgress: Fiber,
  renderLanes: Lanes,
): Fiber | null;
```

**역할**: 파이버 트리를 **아래에서 위로** 순회하며 작업 완료

- DOM 노드 생성 (새 컴포넌트의 경우)
- props 업데이트 준비
- effect 리스트 구성 (부모로 올려보냄)
- 형제 또는 부모로 이동

**흐름**: 자식 완료 → 형제 → 부모 순으로 올라감

```
     Root
    /    \
  A        B
 / \      /
C   D    E

beginWork:  Root → A → C (완료) → D (완료) → A 완료 → B → E (완료) → B 완료 → Root 완료
completeWork: C → D → A → E → B → Root
```

## 커밋 단계

커밋 단계는 **동기적**으로 동작하며, 중단할 수 없습니다. 실제 DOM을 변경합니다.

커밋 단계는 3개의 하위 단계로 구성됩니다.

### 변형 단계 (Mutation Phase)

실제 DOM을 변경하는 단계입니다.

```
effect 리스트를 순회하며:
  - PLACEMENT: 새 노드를 DOM에 추가
  - UPDATE: 기존 노드의 속성 변경
  - DELETION: 노드를 DOM에서 제거
```

**주요 작업**:

- `appendChild`, `removeChild`, `setAttribute` 등 DOM 조작
- ref 해제 (삭제되는 경우)
- `componentWillUnmount` 호출

### 레이아웃 단계 (Layout Phase)

DOM 변경이 완료된 후, 레이아웃 정보를 읽을 수 있는 단계입니다.

```jsx
useLayoutEffect(() => {
  // 여기서 실행됨!
  // DOM 레이아웃 정보를 읽을 수 있음
  const height = divRef.current.offsetHeight;
}, []);
```

**주요 작업**:

- `componentDidMount`, `componentDidUpdate` 호출
- `useLayoutEffect` 콜백 실행
- ref 연결

**특징**: 브라우저가 화면을 그리기 **전에** 동기적으로 실행됨

### 효과 (Effect Phase)

레이아웃 단계 이후, 브라우저가 화면을 그린 **후에** 실행됩니다.

```jsx
useEffect(() => {
  // 여기서 실행됨!
  // 화면이 업데이트된 후 실행
  fetchData();
}, []);
```

**주요 작업**:

- `useEffect` 콜백 실행
- 비동기 작업, 데이터 fetching 등

**특징**:

- 비동기적으로 실행 (화면 업데이트를 차단하지 않음)
- 사용자가 이미 업데이트된 화면을 볼 수 있음

## 전체 흐름 정리

```
1. 상태 변경 발생
   ↓
2. 렌더링 단계 (비동기, 중단 가능)
   - beginWork: 위 → 아래 순회
   - completeWork: 아래 → 위 순회
   - effect 리스트 생성
   ↓
3. 커밋 단계 (동기, 중단 불가)
   - 변형 단계: DOM 변경
   - 레이아웃 단계: useLayoutEffect 실행
   - 효과 단계: useEffect 실행
   ↓
4. 화면에 변경사항 표시 완료! 🎉
```

파이버 재조정자 덕분에 React는 복잡한 UI도 끊김 없이 부드럽게 업데이트할 수 있습니다!

---

## 예제: 파이버 재조정 단계별 분석

이제 구체적인 코드로 파이버 재조정이 어떻게 동작하는지 단계별로 살펴봅시다.

### 예제 컴포넌트

```jsx
function Counter() {
  const [count, setCount] = useState(0);

  return (
    <div className="container">
      <h1>카운터</h1>
      <p>현재 값: {count}</p>
      <button onClick={() => setCount(count + 1)}>증가</button>
    </div>
  );
}
```

### 초기 렌더링: 파이버 트리 생성

처음 렌더링될 때 React는 다음과 같은 파이버 트리를 만듭니다.

```
FiberRoot (최상위)
    ↓
  Counter (함수 컴포넌트)
    ↓
  div (className="container")
    ↓ child
   h1 ────────→ sibling → p ────────→ sibling → button
    ↓                       ↓                      ↓
  "카운터"           "현재 값: 0"              "증가"
```

각 노드는 파이버 객체입니다:

```js
// div 파이버 노드 예시
{
  type: 'div',
  props: { className: 'container' },
  child: h1Fiber,        // 첫 번째 자식을 가리킴
  sibling: null,         // 형제 없음
  return: CounterFiber,  // 부모를 가리킴
  stateNode: <실제 DOM div 노드>,
  alternate: null,       // 아직 업데이트 전이라 null
}
```

---

## 사용자가 버튼 클릭: 상태 변경 발생

```jsx
<button onClick={() => setCount(count + 1)}>
```

1. 사용자가 버튼 클릭
2. `setCount(1)` 호출
3. React가 "업데이트가 필요하구나!" 인식
4. 재조정 시작! 🚀

---

## 렌더링 단계: beginWork 순회 (위 → 아래)

### Step 1: Counter 컴포넌트 (beginWork)

```typescript
beginWork(currentCounterFiber, workInProgressCounterFiber);
```

**수행 작업**:

```js
// 1. 함수 컴포넌트 실행
const result = Counter(); // count는 이제 1

// 2. 반환된 JSX 구조 확인
result = (
  <div className="container">
    <h1>카운터</h1>
    <p>현재 값: 1</p>  {/* 변경됨! */}
    <button onClick={...}>증가</button>
  </div>
);

// 3. 자식 div로 이동
return divFiber;
```

**파이버 상태**:

```js
workInProgressCounterFiber = {
  type: Counter,
  child: divFiber, // 다음에 처리할 자식
};
```

### Step 2: div (beginWork)

```typescript
beginWork(currentDivFiber, workInProgressDivFiber);
```

**수행 작업**:

```js
// 1. props 비교
oldProps = { className: "container" };
newProps = { className: "container" };
// → 변경 없음!

// 2. 자식들 비교 준비
// h1, p, button 순서로 확인 예정

// 3. 첫 번째 자식 h1으로 이동
return h1Fiber;
```

### Step 3: h1 (beginWork)

```typescript
beginWork(currentH1Fiber, workInProgressH1Fiber);
```

**수행 작업**:

```js
// 1. 내용 비교
oldChildren = "카운터";
newChildren = "카운터";
// → 변경 없음!

// 2. 자식이 텍스트 노드뿐이므로 깊이 들어가지 않음

// 3. h1 작업 완료, completeWork로 전환
return null; // 더 이상 자식 없음
```

### Step 4: h1 (completeWork) - 첫 번째 완료!

```typescript
completeWork(currentH1Fiber, workInProgressH1Fiber);
```

**수행 작업**:

```js
// 1. h1은 변경 없음 → effect 태그 없음
workInProgressH1Fiber.effectTag = null;

// 2. 형제 p로 이동
return pFiber;
```

### Step 5: p (beginWork) - 여기서 변경 감지!

```typescript
beginWork(currentPFiber, workInProgressPFiber);
```

**수행 작업**:

```js
// 1. 내용 비교
oldChildren = "현재 값: 0";
newChildren = "현재 값: 1"; // 🎯 변경됨!

// 2. effect 태그 설정
workInProgressPFiber.effectTag = "UPDATE";

// 3. 업데이트할 내용 저장
workInProgressPFiber.updateQueue = {
  textContent: "현재 값: 1",
};

// 4. p 작업 완료, completeWork로 전환
return null;
```

### Step 6: p (completeWork)

```typescript
completeWork(currentPFiber, workInProgressPFiber);
```

**수행 작업**:

```js
// 1. effect 리스트에 추가 (나중에 커밋 단계에서 처리)
workInProgressPFiber.nextEffect = null;

// 2. 부모(div)의 effect 리스트에 연결
divFiber.firstEffect = pFiber;
divFiber.lastEffect = pFiber;

// 3. 형제 button으로 이동
return buttonFiber;
```

### Step 7: button (beginWork)

```typescript
beginWork(currentButtonFiber, workInProgressButtonFiber);
```

**수행 작업**:

```js
// 1. props 비교
oldProps = { onClick: function }
newProps = { onClick: function }
// → 함수는 다시 생성되지만 같은 동작

// 2. 자식 "증가" 텍스트는 변경 없음

// 3. button 작업 완료
return null;
```

### Step 8: button (completeWork)

```typescript
completeWork(currentButtonFiber, workInProgressButtonFiber);
```

**수행 작업**:

```js
// 1. 변경 없음 → effect 없음

// 2. 형제 없음, 부모(div)로 올라감
return divFiber;
```

### Step 9: div (completeWork)

```typescript
completeWork(currentDivFiber, workInProgressDivFiber);
```

**수행 작업**:

```js
// 1. 자식들의 effect 리스트 수집 완료
// divFiber.firstEffect → pFiber (UPDATE 필요)

// 2. 부모(Counter)로 effect 리스트 전달
CounterFiber.firstEffect = pFiber;

// 3. 부모로 올라감
return CounterFiber;
```

### Step 10: Counter (completeWork)

```typescript
completeWork(currentCounterFiber, workInProgressCounterFiber);
```

**수행 작업**:

```js
// 1. 모든 하위 effect 리스트 수집 완료

// 2. 렌더링 단계 완료!
// 최종 effect 리스트: [pFiber (UPDATE)]
```

---

## 렌더링 단계 완료 후 상태

```
Effect 리스트 (커밋할 작업들):
  → pFiber: UPDATE (textContent: "현재 값: 0" → "현재 값: 1")
```

**중요**: 아직 실제 DOM은 바뀌지 않았습니다! 화면에는 여전히 "현재 값: 0"이 보입니다.

---

## 커밋 단계: 실제 DOM 업데이트

### 변형 단계 (Mutation Phase)

```typescript
// Effect 리스트 순회
let nextEffect = rootFiber.firstEffect;

while (nextEffect !== null) {
  const effectTag = nextEffect.effectTag;

  if (effectTag === "UPDATE") {
    // pFiber 처리
    const domNode = nextEffect.stateNode; // <p> DOM 노드
    const updateQueue = nextEffect.updateQueue;

    // 🎯 실제 DOM 업데이트!
    domNode.textContent = "현재 값: 1";
  }

  nextEffect = nextEffect.nextEffect;
}
```

**실제로 일어나는 일**:

```js
// 브라우저 DOM API 직접 호출
document.querySelector("p").textContent = "현재 값: 1";
```

이제 화면에 변경사항이 반영됩니다! 👁️

### 레이아웃 단계 (Layout Phase)

```typescript
// useLayoutEffect가 있다면 실행
nextEffect = rootFiber.firstEffect;

while (nextEffect !== null) {
  if (nextEffect.layoutEffects) {
    nextEffect.layoutEffects.forEach((effect) => {
      // DOM 레이아웃 읽기 가능
      effect(); // useLayoutEffect 콜백 실행
    });
  }

  nextEffect = nextEffect.nextEffect;
}
```

이 예제에는 `useLayoutEffect`가 없으므로 스킵!

### 효과 단계 (Effect Phase)

```typescript
// useEffect가 있다면 실행 (비동기)
requestIdleCallback(() => {
  nextEffect = rootFiber.firstEffect;

  while (nextEffect !== null) {
    if (nextEffect.effects) {
      nextEffect.effects.forEach((effect) => {
        effect(); // useEffect 콜백 실행
      });
    }

    nextEffect = nextEffect.nextEffect;
  }
});
```

이 예제에는 `useEffect`가 없으므로 스킵!

---

## 더블 버퍼링: 트리 스왑

```js
// 커밋 완료 후
current = workInProgress; // 작업 트리가 현재 트리로!
workInProgress = current.alternate; // 이전 현재 트리를 재사용

// 다음 업데이트를 위해 준비 완료
```

```
업데이트 전:
current (화면에 표시) ← 사용자가 보는 중
workInProgress (작업 중) ← React가 계산 중

업데이트 후:
current (화면에 표시) ← workInProgress였던 트리 (count: 1)
workInProgress (유휴) ← current였던 트리 (count: 0, 재사용 대기)
```

---

## 전체 과정 타임라인

```
🕐 0ms: 사용자 버튼 클릭
         ↓
🕐 0ms: setCount(1) 호출
         ↓
🕐 1ms: 렌더링 단계 시작 (비동기)
         ├─ beginWork(Counter) → 함수 실행, count = 1
         ├─ beginWork(div) → props 비교
         ├─ beginWork(h1) → 변경 없음
         ├─ completeWork(h1) → 완료
         ├─ beginWork(p) → 변경 감지! effectTag = 'UPDATE'
         ├─ completeWork(p) → effect 리스트 추가
         ├─ beginWork(button) → 변경 없음
         ├─ completeWork(button) → 완료
         ├─ completeWork(div) → effect 수집
         └─ completeWork(Counter) → 렌더링 완료
         ↓
🕐 3ms: 커밋 단계 시작 (동기)
         ├─ 변형: <p> DOM의 textContent 변경
         ├─ 레이아웃: (이 예제에선 없음)
         └─ 효과: (이 예제에선 없음)
         ↓
🕐 4ms: 트리 스왑 (current ↔ workInProgress)
         ↓
🕐 5ms: 브라우저가 화면에 "현재 값: 1" 표시
         ↓
✅ 완료!
```

---

## 핵심 정리

### beginWork (위 → 아래)

```js
// "이 컴포넌트 뭐가 달라졌지?" 확인
beginWork(Counter)
  → 함수 실행 (count = 1)
  → 자식 생성

beginWork(p)
  → "아, 텍스트가 바뀌었네!"
  → effectTag = 'UPDATE' 표시
```

### completeWork (아래 → 위)

```js
// "변경사항들 정리해서 부모한테 올려보내자"
completeWork(p)
  → effect 리스트에 추가
  → 부모(div)에게 전달

completeWork(div)
  → 자식들의 effect 모아서
  → 부모(Counter)에게 전달
```

### 커밋 단계

```js
// "이제 실제로 DOM 바꾸자!"
effect 리스트 순회:
  → pFiber.effectTag === 'UPDATE'
  → pFiber.stateNode.textContent = "현재 값: 1"
  → 화면에 반영! 🎉
```

이제 파이버 재조정이 어떻게 동작하는지 명확히 이해되셨나요?

**핵심**: React는 변경된 부분만 찾아서(렌더링 단계) 효율적으로 DOM을 업데이트(커밋 단계)합니다!
