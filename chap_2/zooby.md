# CHAPTER 2: JSX

## 2.1 자바스크립트 XML?

JSX는 JavaScript eXtension의 약자다. XML이 아니다.

과거에는 서버 응답 형식으로 XML을 많이 썼지만, 요즘은 JSON이 대세다.  
`fetch`가 `XMLHttpRequest`보다 널리 쓰이는 이유가 이름 때문일까 궁금하다.

```javascript
// XMLHttpRequest - 콜백 기반
const xhr = new XMLHttpRequest();
xhr.open("GET", "/api/users");
xhr.onreadystatechange = function () {
  if (xhr.readyState === 4 && xhr.status === 200) {
    const data = JSON.parse(xhr.responseText);
  }
};
xhr.send();

// fetch - Promise 기반
const response = await fetch("/api/users");
const data = await response.json();
```

실제로는 다음과 같이 쓰인다.  
fetch는 Promise 기반이라 async/await와 자연스럽게 사용할 수 있고, API가 더 직관적이다.

다시 본론으로...

결국 JSX는 HTML을 쓰는 형태와 유사하게 사용할 수 있는 자바스크립트 구문 확장이다.  
원래 Meta에서 React와 함께 쓰려고 만들었는데, Vue에서도 선택적으로 지원한다.

그렇다면 Vue나 Angular는 처음에 뭘 썼을까?

- **Vue**: HTML 기반 템플릿 (`v-if`, `v-for` 디렉티브) - 지금도 이게 기본이지만, JSX 지원
- **Angular**: HTML 기반 템플릿 (`*ngIf`, `*ngFor` 디렉티브) - 지금도 템플릿 중심

React는 JS를 확장하는 방식, Vue/Angular는 HTML을 확장하는 방식이라는 철학적 차이가 있다.

### JSX 사용 예시

```jsx
function UserList({ users }) {
  return (
    <ul className="user-list">
      {users.map((user) => (
        <li key={user.id}>
          <span>{user.name}</span>
          <button onClick={() => alert(user.name)}>인사</button>
        </li>
      ))}
    </ul>
  );
}
```

### React.createElement로 작성하면

```javascript
function UserList({ users }) {
  return React.createElement(
    "ul",
    { className: "user-list" },
    users.map((user) =>
      React.createElement(
        "li",
        { key: user.id },
        React.createElement("span", null, user.name),
        React.createElement(
          "button",
          { onClick: () => alert(user.name) },
          "인사",
        ),
      ),
    ),
  );
}
```

JSX는 브라우저가 직접 이해하지 못한다. Babel 같은 트랜스파일러가 위처럼 `React.createElement()` 호출로 변환해준다

## 2.2 JSX의 장점

- 가독성이 좋다. UI 구조가 한눈에 보인다.
- 데이터 소독, 보안이 향상된다.
- JavaScript 표현식을 `{}`로 바로 사용할 수 있다.
- 컴파일 타임에 문법 오류를 잡아준다.
- 컴포넌트 조합이 직관적이다.

## 2.3 JSX의 약점

- Babel 같은 트랜스파일러가 필수다. (빌드 과정 필요)
- HTML과 비슷하지만 다르다. (`class` → `className`, `for` → `htmlFor`)
- 조건부 렌더링이 복잡해지면 삼항연산자 지옥이 될 수 있다.
- 처음 보는 사람에게는 "이게 JS야 HTML이야?" 혼란을 줄 수 있다.

## 2.4 내부 동작

내가 작성한 코드가 어떻게 실행되는지 알아보자.

```javascript
const a = 1;
let b = 2;

console.log(a + b);
```

### 컴파일러란?

사람이 읽을 수 있는 코드를 기계가 실행할 수 있는 형태로 변환하는 프로그램이다.

컴파일 과정은 크게 3단계로 나뉜다:

**1. 토큰화 (Tokenization / Lexical Analysis)**

소스 코드를 의미 있는 최소 단위(토큰)로 쪼갠다.

```
const a = 1;
↓
[const] [a] [=] [1] [;]
```

**2. 구문 분석 (Parsing)**

토큰들을 트리 구조(AST, Abstract Syntax Tree)로 만든다.

```
Program
└── VariableDeclaration (const)
    └── VariableDeclarator
        ├── Identifier: a
        └── Literal: 1
```

**3. 코드 생성 (Code Generation)**

AST를 바탕으로 실행 가능한 코드를 생성한다.

### JIT 컴파일러

웹 브라우저를 비롯한 최신 환경에서는 JavaScript를 효율적으로 실행하기 위해 JIT(Just-In-Time) 컴파일러를 사용한다.

**기존 방식들:**

- **인터프리터**: 코드를 한 줄씩 읽어서 바로 실행. 시작은 빠르지만 반복 실행 시 느리다.
- **AOT 컴파일러**: 실행 전에 전체를 미리 컴파일. 실행은 빠르지만 시작이 느리다.

**JIT의 장점:**

- 처음엔 인터프리터처럼 빠르게 시작
- 자주 실행되는 코드(핫 코드)를 감지해서 그 부분만 기계어로 컴파일
- 런타임 정보를 활용해 최적화 (예: 타입이 항상 숫자면 숫자 연산으로 최적화)

V8(Chrome), SpiderMonkey(Firefox), JavaScriptCore(Safari) 모두 JIT를 사용한다.

### 런타임이란?

프로그램이 실행되는 환경이다. JavaScript 런타임은 코드 실행에 필요한 모든 것을 제공한다:

- **JavaScript 엔진**: 코드 파싱 및 실행 (V8 등)
- **Web API / Node API**: DOM, fetch, setTimeout, fs 등
- **이벤트 루프**: 비동기 작업 관리
- **콜 스택, 메모리 힙**: 실행 컨텍스트와 메모리 관리

브라우저와 Node.js는 같은 V8 엔진을 쓰지만, 제공하는 API가 다르기 때문에 다른 런타임이다.

### JSX의 작동 원리

JSX는 브라우저가 이해하지 못한다. 그래서 **트랜스파일러**가 필요하다.

```jsx
// 내가 작성한 JSX
<button onClick={handleClick}>클릭</button>
```

```javascript
// 트랜스파일 후 (React 17+)
import { jsx as _jsx } from "react/jsx-runtime";
_jsx("button", { onClick: handleClick, children: "클릭" });
```

```javascript
// 트랜스파일 후 (React 16 이하)
React.createElement("button", { onClick: handleClick }, "클릭");
```

### 트랜스파일러 도구들

**컴파일 vs 트랜스파일:**

- 컴파일: 고수준 언어 → 저수준 언어 (C → 기계어)
- 트랜스파일: 고수준 언어 → 고수준 언어 (JSX → JavaScript, TS → JS)

| 도구                 | 설명                              | 특징                                                            |
| -------------------- | --------------------------------- | --------------------------------------------------------------- |
| **Babel**            | JavaScript 트랜스파일러의 표준    | JSX 변환, 최신 문법을 구버전으로 변환. 플러그인 생태계가 거대함 |
| **TypeScript (tsc)** | TS → JS 컴파일러                  | 타입 검사 + 트랜스파일. JSX도 지원                              |
| **Traceur**          | Google이 만든 ES6 트랜스파일러    | Babel 이전에 많이 쓰였음. 지금은 거의 안 씀                     |
| **SWC**              | Rust로 작성된 초고속 트랜스파일러 | Babel 대비 20~70배 빠름. Next.js 기본 채택                      |

요즘 트렌드는 Babel → SWC, esbuild 같은 네이티브 도구로 이동 중이다. 빌드 속도가 체감될 정도로 빨라진다.

## 2.5 JSX 프라그마

프라그마(Pragma)는 "JSX를 어떤 함수로 변환할지" 지정하는 주석이다.

```jsx
/** @jsx h */
import { h } from "preact";

// 이 파일의 JSX는 React.createElement가 아니라 h()로 변환됨
<div>Hello</div>; // → h('div', null, 'Hello')
```

**언제 쓰나?**

- Preact 사용 시 (`@jsx h`)
- Emotion CSS-in-JS 사용 시 (`@jsxImportSource @emotion/react`)
- 커스텀 JSX 런타임을 만들 때

**React 17+ 새로운 JSX Transform**

React 17부터는 프라그마 없이도 자동으로 처리된다.

```jsx
// 내가 작성한 코드 (import React 없음!)
function App() {
  return <div>Hello</div>;
}

// 자동 변환됨
import { jsx as _jsx } from "react/jsx-runtime";
function App() {
  return _jsx("div", { children: "Hello" });
}
```

`import React from 'react'`를 안 써도 되는 이유가 이것이다.
