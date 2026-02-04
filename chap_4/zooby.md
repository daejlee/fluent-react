리액트의 재조정(Reconciliation)에 대해 알아본다. 
가상 DOM이라는 청사진을 실제 UI로 만드는 과정이다.

# 4.1 재조정 이해하기

재조정(Reconciliation)은 React Element 트리를 실제 DOM에 반영하는 과정이다. 가상 DOM이라는 "청사진"을 현실의 UI로 만드는 작업이라고 이해하면 된다.

React는 재조정 시 두 가지 가정(휴리스틱)을 사용한다:

1. **같은 타입의 Element는 유사한 구조를 가진다**
   - 타입이 변경되면 기존 컴포넌트 인스턴스를 언마운트하고 새로운 것으로 마운트한다.
   - 예: `<div>` → `<span>`으로 바뀌면 기존 트리를 버리고 새로 구축

2. **key를 통해 리스트 자식을 고유하게 식별한다**
   - key가 변경되면 해당 컴포넌트와 하위 요소들을 새로 마운트한다.
   - 리스트에서 효율적인 업데이트를 가능하게 함

# 4.2 일괄처리 (Batch Update)

여러 상태 변경이 발생해도 React는 이를 즉시 반영하지 않는다. 대신 모든 변경사항을 모아서 한 번에 DOM에 반영한다.

```jsx
// 아래 세 번의 setState가 있어도 리렌더링은 한 번만 발생
function handleClick() {
  setCount(count + 1);
  setFlag(true);
  setName('React');
}
```

이렇게 일괄처리하면:
- DOM 조작 횟수가 줄어듦
- 불필요한 중간 상태 렌더링 방지
- 성능 최적화

# 4.3 Stack Reconciler

React 16 이전에 사용되던 재조정 알고리즘이다.

**특징:**
- 동기적으로 동작
- 호출 스택(call stack)이 빌 때까지 작업을 처리
- 작업 중단이 불가능

**문제점:**
- 큰 트리를 처리할 때 메인 스레드를 오래 점유
- 앱이 무반응 상태가 될 수 있음
- 프레임 드랍 발생 (16ms 내에 렌더링을 못 끝내면 버벅임)

# 4.4 Fiber Reconciler

React 16에서 도입된 새로운 재조정 알고리즘이다. Stack Reconciler의 한계를 극복하기 위해 설계되었다.

**핵심 아이디어:**
- 작업을 작은 단위(chunk)로 나눔
- 우선순위 지정 가능
- 작업 중단 및 재개 가능
- 비동기 동작

## 4.4.1 데이터 구조로서의 Fiber

Fiber는 작업 단위(unit of work)이자 데이터 구조다. 각 React Element에 대응하는 Fiber 노드가 존재한다.

**Fiber 노드의 주요 속성:**
```js
{
  type: 컴포넌트 타입 (함수, 클래스, 'div' 등),
  key: 식별자,
  child: 첫 번째 자식 Fiber,
  sibling: 다음 형제 Fiber,
  return: 부모 Fiber,
  stateNode: 실제 DOM 노드 또는 컴포넌트 인스턴스,
  alternate: 이전/다음 버전의 Fiber (더블 버퍼링용),
  // ... 상태, 이펙트 등
}
```

### Fiber vs ReactElement 차이

| | ReactElement | Fiber |
|-|--------------|-------|
| **수명** | 임시적, 렌더링 때마다 새로 생성 | 오래 유지, 재사용됨 |
| **상태** | 상태 없음 (stateless) | 상태 저장 (stateful) |
| **역할** | UI의 스냅샷, "무엇을 그릴지" 설명 | 작업 단위, "어떻게 처리할지" 관리 |
| **변경** | 불변 (immutable) | 가변 (mutable), 업데이트됨 |

**쉬운 비유:**
- **ReactElement**: 카페에서 주문한 "아메리카노 1잔" 주문서. 주문 넣으면 끝, 버려짐.
- **Fiber**: 바리스타의 작업 카드. 주문 상태(대기/제조중/완료), 재료 정보, 이전 주문과의 연결 등을 계속 추적하며 업데이트됨.

## 4.4.2 더블 버퍼링

React는 두 개의 Fiber 트리를 유지한다:

1. **current 트리**: 현재 화면에 보이는 UI를 나타냄
2. **workInProgress 트리**: 비동기 작업이 반영되는 새 트리

작업 완료 후 포인터만 전환하면 새 UI가 즉시 반영된다. 마치 게임에서 프레임 버퍼를 교체하는 것과 같다.

```
[current] ←→ [workInProgress]
    ↓ (포인터 전환)
[workInProgress] ←→ [current] (역할 교체)
```

`alternate` 속성으로 두 트리의 Fiber가 서로를 가리키며, 이는 Fiber 재사용을 극대화한다.

## 4.4.3 Fiber 재조정

Fiber 재조정은 두 단계로 나뉜다:

### Render 단계 (비동기, 중단 가능)

**beginWork**: 트리를 위에서 아래로 순회하며 각 Fiber 작업 시작
- 컴포넌트 렌더링
- 새로운 children과 기존 children 비교
- 변경이 필요한 Fiber에 플래그(effectTag) 표시

**completeWork**: 아래에서 위로 올라가며 작업 완료 처리
- DOM 노드 생성/업데이트 준비
- 이펙트 리스트 수집

```
beginWork (Parent)
  └→ beginWork (Child1)
       └→ beginWork (GrandChild)
            └→ completeWork (GrandChild)
       └→ completeWork (Child1)
  └→ beginWork (Child2)
       └→ completeWork (Child2)
  └→ completeWork (Parent)
```

### Commit 단계 (동기, 중단 불가)

실제 DOM에 변경사항을 반영하는 단계다:

1. **Before mutation**: DOM 변경 전 스냅샷 (getSnapshotBeforeUpdate)
2. **Mutation**: 실제 DOM 조작 수행
3. **Layout**: DOM 변경 후 (useLayoutEffect, componentDidMount/Update)

```
[Render Phase]          [Commit Phase]
beginWork → completeWork → DOM 반영 → 생명주기/이펙트 호출
(비동기, 중단가능)        (동기, 중단불가)
```
