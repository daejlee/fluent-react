# CHAPTER 3: 가상 DOM

## 3.1 가상 DOM 소개

가상 DOM(Virtual DOM)은 실제 DOM의 가벼운 복사본이다. JavaScript 객체 형태로 메모리에 존재한다.

React가 가상 DOM을 사용하는 이유는 간단하다. **실제 DOM 조작은 비싸다.**

```jsx
// React 컴포넌트
function Counter({ count }) {
  return <div className="counter">{count}</div>;
}
```

위 JSX는 내부적으로 이런 가상 DOM 객체가 된다:

```javascript
{
  type: 'div',
  props: {
    className: 'counter',
    children: count
  }
}
```

그냥 JavaScript 객체다. DOM API를 호출하는 것보다 객체를 만들고 비교하는 게 훨씬 빠르다.

## 3.2 실제 DOM

### DOM이란?

DOM(Document Object Model)은 HTML 문서를 트리 구조로 표현한 것이다. 브라우저가 HTML을 파싱해서 만든다.

```html
<html>
  <body>
    <div id="app">
      <h1>Hello</h1>
      <p>World</p>
    </div>
  </body>
</html>
```

```
document
└── html
    └── body
        └── div#app
            ├── h1
            │   └── "Hello"
            └── p
                └── "World"
```

### DOM 조작이 느린 이유

DOM을 조작하면 브라우저는 다음 과정을 거친다:

1. **DOM 트리 수정** - 노드 추가/삭제/변경
2. **스타일 계산** - CSS 규칙 적용
3. **레이아웃(Reflow)** - 요소들의 위치와 크기 계산
4. **페인트(Repaint)** - 픽셀로 그리기
5. **합성(Composite)** - 레이어 합치기

특히 **Reflow**가 비싸다. 하나의 요소 크기가 바뀌면 주변 요소들 위치도 다시 계산해야 한다.

```javascript
// 이런 코드는 매번 Reflow를 발생시킴
for (let i = 0; i < 1000; i++) {
  const div = document.createElement("div");
  div.textContent = i;
  container.appendChild(div); // Reflow!
}

// 이렇게 하면 한 번만 발생
const fragment = document.createDocumentFragment();
for (let i = 0; i < 1000; i++) {
  const div = document.createElement("div");
  div.textContent = i;
  fragment.appendChild(div); // 메모리에서만 작업
}
container.appendChild(fragment); // Reflow 한 번!
```

React의 가상 DOM도 비슷한 원리다. 변경사항을 메모리에서 모아서 한 번에 실제 DOM에 반영한다.

### 3.2.1 실제 DOM의 문제점

#### 성능 - 레이아웃 스래싱 (Layout Thrashing)

DOM을 읽고 쓰는 작업을 번갈아 하면 브라우저가 강제로 동기 레이아웃을 계산해야 한다. 이걸 **레이아웃 스래싱**이라고 한다.

```javascript
// ❌ 레이아웃 스래싱 발생
for (const box of boxes) {
  const width = box.offsetWidth; // 읽기 → 레이아웃 계산 강제
  box.style.width = width + 10 + "px"; // 쓰기 → 레이아웃 무효화
  // 다음 루프에서 다시 읽기 → 또 레이아웃 계산...
}

// ✅ 읽기와 쓰기 분리
const widths = boxes.map((box) => box.offsetWidth); // 읽기 몰아서
boxes.forEach((box, i) => {
  box.style.width = widths[i] + 10 + "px"; // 쓰기 몰아서
});
```

읽기 작업: `offsetWidth`, `offsetHeight`, `getBoundingClientRect()`, `scrollTop` 등
쓰기 작업: `style.*`, `className`, `innerHTML` 등

React는 이런 문제를 자동으로 해결해준다. 상태 변경을 모아서 한 번에 DOM에 반영하기 때문이다.

#### 브라우저 간 호환성

DOM API는 브라우저마다 미묘하게 다르게 동작한다.

```javascript
// 이벤트 객체
element.addEventListener("click", (e) => {
  // IE: window.event 사용해야 함
  // 표준: e 파라미터 사용
  const event = e || window.event;

  // IE: e.srcElement
  // 표준: e.target
  const target = e.target || e.srcElement;
});

// 이벤트 전파 중단
e.stopPropagation(); // 표준
e.cancelBubble = true; // IE
```

React의 SyntheticEvent가 이런 차이를 추상화해서 일관된 API를 제공한다.

```jsx
// React - 브라우저 상관없이 동일하게 동작
<button
  onClick={(e) => {
    e.stopPropagation(); // 모든 브라우저에서 동작
    console.log(e.target); // 항상 동일
  }}
>
  클릭
</button>
```

### 3.2.2 문서 조각 (DocumentFragment)

DocumentFragment는 DOM 노드를 저장하는 가벼운 컨테이너다. 실제 DOM 트리에 속하지 않아서 조작해도 리플로우가 발생하지 않는다.

```javascript
const fragment = document.createDocumentFragment();

// fragment에 추가하는 건 메모리에서만 일어남
for (let i = 0; i < 1000; i++) {
  const li = document.createElement("li");
  li.textContent = `Item ${i}`;
  fragment.appendChild(li); // 리플로우 없음
}

// 한 번에 DOM에 추가
ul.appendChild(fragment); // 리플로우 1번
```

#### 일괄 업데이트 (Batch Update)

여러 DOM 변경을 모아서 한 번에 처리하는 것. React의 상태 업데이트도 기본적으로 배치 처리된다.

```javascript
// React 18 - 자동 배치
function handleClick() {
  setCount((c) => c + 1); // 리렌더링 안 함
  setFlag((f) => !f); // 리렌더링 안 함
  setText("hello"); // 리렌더링 안 함
  // 함수 끝나면 한 번만 리렌더링
}
```

#### 메모리 효율성

DocumentFragment는 DOM 트리에 삽입되면 자식 노드들만 이동하고 자신은 사라진다. 별도의 래퍼 요소가 남지 않는다.

```javascript
// fragment 자체는 DOM에 삽입되지 않음
const fragment = document.createDocumentFragment();
fragment.appendChild(child1);
fragment.appendChild(child2);

parent.appendChild(fragment);
// parent에는 child1, child2만 추가됨 (fragment는 없음)
```

불필요한 wrapper div 없이 여러 요소를 그룹화한다.

#### 중복 렌더링 관리

같은 요소를 여러 번 업데이트해도 마지막 상태만 DOM에 반영된다.

```javascript
// Vanilla JS - 3번 리플로우
element.style.width = "100px"; // 리플로우
element.style.width = "200px"; // 리플로우
element.style.width = "300px"; // 리플로우

// React - 마지막 상태만 반영
setWidth(100);
setWidth(200);
setWidth(300);
// 최종적으로 300px만 DOM에 적용
```

### DOM API 직접 사용 vs React

```javascript
// Vanilla JS - 직접 DOM 조작
function updateCounter(count) {
  document.getElementById("counter").textContent = count;
  document.getElementById("double").textContent = count * 2;
  document.getElementById("triple").textContent = count * 3;
  // 뭘 업데이트해야 하는지 직접 관리해야 함
}

// React - 선언적
function Counter({ count }) {
  return (
    <div>
      <span id="counter">{count}</span>
      <span id="double">{count * 2}</span>
      <span id="triple">{count * 3}</span>
    </div>
  );
}
// React가 알아서 바뀐 부분만 업데이트
```

## 3.3 가상 DOM 작동 방식

### 3.3.1 리액트 엘리먼트

리액트 엘리먼트는 화면에 표시할 내용을 설명하는 일반 JavaScript 객체다. 불변(immutable)이고 가볍다.

```jsx
// JSX
const element = <h1 className="greeting">Hello, world!</h1>;

// 실제로는 이런 객체가 됨
const element = {
  type: "h1",
  props: {
    className: "greeting",
    children: "Hello, world!",
  },
};
```

**리액트 엘리먼트 vs DOM 엘리먼트**

```javascript
// DOM 엘리먼트 - 무겁다
const domElement = document.createElement("div");
console.log(Object.keys(domElement).length); // 200개 이상의 속성

// 리액트 엘리먼트 - 가볍다
const reactElement = { type: "div", props: {} };
console.log(Object.keys(reactElement).length); // 2개
```

**컴포넌트 엘리먼트**

type이 문자열이면 DOM 태그, 함수/클래스면 컴포넌트다.

```jsx
// DOM 엘리먼트 (type이 문자열)
{ type: 'button', props: { children: 'Click' } }

// 컴포넌트 엘리먼트 (type이 함수)
{ type: Button, props: { children: 'Click' } }

// React는 Button을 호출해서 반환값(DOM 엘리먼트)을 얻는다
function Button({ children }) {
  return { type: 'button', props: { children } };
}
```

### 3.3.2 가상 DOM과 실제 DOM 비교

| 구분          | 실제 DOM                | 가상 DOM             |
| ------------- | ----------------------- | -------------------- |
| **본질**      | 브라우저의 문서 객체    | JavaScript 객체      |
| **변경 비용** | 비싸다 (Reflow/Repaint) | 싸다 (메모리 연산)   |
| **업데이트**  | 즉시 화면 반영          | 배치 후 한 번에 반영 |
| **비교**      | 불가능 (이전 상태 없음) | 이전/현재 비교 가능  |

**동작 흐름**

```
1. 상태 변경 발생
   ↓
2. 새로운 가상 DOM 트리 생성
   ↓
3. 이전 가상 DOM과 비교 (Diffing)
   ↓
4. 변경된 부분만 실제 DOM에 반영 (Patch)
   ↓
5. 브라우저가 화면 업데이트
```

**실제 비교 예시**

```jsx
// 상태: count = 0 → 1로 변경

// 이전 가상 DOM
{
  type: 'div',
  props: {
    children: [
      { type: 'span', props: { children: 'Count: ' } },
      { type: 'span', props: { children: 0 } }  // ← 여기만 다름
    ]
  }
}

// 새로운 가상 DOM
{
  type: 'div',
  props: {
    children: [
      { type: 'span', props: { children: 'Count: ' } },  // 동일 → 스킵
      { type: 'span', props: { children: 1 } }  // 다름 → 업데이트
    ]
  }
}

// 결과: 두 번째 span의 textContent만 실제 DOM에서 변경
```

### 재조정(Reconciliation)

상태가 바뀌면 React는 다음 과정을 거친다:

1. **새로운 가상 DOM 생성** - 변경된 상태로 전체 트리를 다시 만듦
2. **Diffing** - 이전 가상 DOM과 새 가상 DOM 비교
3. **Patch** - 차이점만 실제 DOM에 반영

```jsx
// 상태 변경 전
<ul>
  <li>Apple</li>
  <li>Banana</li>
</ul>

// 상태 변경 후
<ul>
  <li>Apple</li>
  <li>Banana</li>
  <li>Cherry</li>  // 이것만 추가됨
</ul>
```

React는 전체를 다시 그리는 게 아니라 Cherry `<li>`만 실제 DOM에 추가한다.

### Diffing 알고리즘

두 트리를 완벽하게 비교하려면 O(n³) 시간이 걸린다. React는 두 가지 가정으로 O(n)으로 줄였다:

**1. 타입이 다르면 전체 교체**

```jsx
// 이전
<div><Counter /></div>

// 이후
<span><Counter /></span>

// div → span으로 바뀌면 Counter까지 전부 언마운트하고 새로 만듦
```

**2. key로 같은 요소인지 식별**

```jsx
// key가 없으면 순서로 비교
<ul>
  <li>A</li>  // index 0
  <li>B</li>  // index 1
</ul>

// 맨 앞에 추가하면?
<ul>
  <li>Z</li>  // index 0 - A와 비교 → 다름 → 교체
  <li>A</li>  // index 1 - B와 비교 → 다름 → 교체
  <li>B</li>  // index 2 - 없음 → 새로 생성
</ul>
// 3개 모두 조작됨 (비효율)

// key가 있으면
<ul>
  <li key="a">A</li>
  <li key="b">B</li>
</ul>

<ul>
  <li key="z">Z</li>  // 새로운 key → 삽입
  <li key="a">A</li>  // 그대로
  <li key="b">B</li>  // 그대로
</ul>
// Z만 삽입됨 (효율적)
```

이게 key를 index로 쓰면 안 되는 이유다.

### Fiber 아키텍처

React 16부터 도입된 새로운 재조정 엔진이다.

**기존 문제점 (Stack Reconciler)**

- 재조정을 시작하면 끝날 때까지 멈출 수 없었음
- 큰 트리를 업데이트하면 메인 스레드가 블로킹됨
- 애니메이션이 버벅이고, 입력이 늦게 반응함

**Fiber의 해결책**

- 작업을 작은 단위(Fiber)로 쪼갬
- 우선순위에 따라 작업을 중단/재개할 수 있음
- 급한 작업(사용자 입력)을 먼저 처리

```
[기존]
렌더링 시작 =================> 렌더링 완료
              (중단 불가)

[Fiber]
렌더링 시작 ===> 중단 ===> 급한 작업 처리 ===> 재개 ===> 완료
```

이게 React 18의 Concurrent Features(useTransition, useDeferredValue 등)의 기반이다.

## 3.4 마무리 - 가상 DOM에 대한 오해

### "가상 DOM이 빠르다"는 말의 진짜 의미

가상 DOM 자체가 빠른 게 아니다. 잘 최적화된 Vanilla JS가 더 빠를 수 있다.

가상 DOM의 장점은:

- **충분히 빠르면서** 개발자가 직접 DOM 조작을 신경 쓰지 않아도 됨
- 선언적으로 UI를 작성할 수 있음
- 변경 추적을 자동으로 해줌

Svelte나 SolidJS 같은 프레임워크는 가상 DOM 없이 컴파일 타임에 최적화된 코드를 생성한다. 벤치마크에서 React보다 빠른 경우가 많다. 하지만 React의 가상 DOM이 "충분히 빠르고" 생태계가 거대하기 때문에 여전히 많이 쓰인다.

### React DevTools로 리렌더링 확인하기

React DevTools의 Profiler를 사용하면 어떤 컴포넌트가 왜 리렌더링되었는지 확인할 수 있다. 불필요한 리렌더링을 찾아서 최적화하는 데 유용하다.

```jsx
// 불필요한 리렌더링 예시
function Parent() {
  const [count, setCount] = useState(0);

  return (
    <div>
      <button onClick={() => setCount((c) => c + 1)}>+</button>
      <Child /> {/* count와 관계없는데 매번 리렌더링됨 */}
    </div>
  );
}

// memo로 방지
const Child = React.memo(function Child() {
  return <div>I'm a child</div>;
});
```
