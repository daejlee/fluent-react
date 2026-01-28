# JSX

### XML -> JSON

자바스크립트 XML의 약자입니다. 왜 XML이냐하면, HTML과 비슷하게 JSX 문법이 생겼기 때문입니다. 책에서는 XMLHttpRequest에 대해서 소개하는데, 최근에는 fetch가 사용되어 JSON 데이터를 주고 받는게 일반적라고 소개합니다.

> ### 주관
>
> 솔직히 이부분에서 왜 갑자기 비동기 통신얘기가 나오는지 주제에 맞지는 않았던 것 같습니다.
>
> 뭐 그래도 이전장의 내용을 상기하게 되었으니 오좋

### createElement vs jsx, jsxs

과거에는 createElement를 사용하여 React 요소를 생성했습니다. 예를 들어, 다음과 같이 사용할 수 있습니다.

```javascript
const element = React.createElement(
  "h1",
  { className: "greeting" },
  "Hello, world!",
);
```

하지만 JSX를 사용하면 더 간결하고 직관적으로 작성할 수 있습니다.

```javascript
const element = <h1 className="greeting">Hello, world!</h1>;
```

또 jsx, jsxs 함수를 사용해서도 React 요소를 생성할 수 있습니다. jsx 함수는 단일 자식을 가진 요소에 사용되고, jsxs 함수는 다중 자식을 가진 요소에 사용됩니다.

```javascript
import { jsx, jsxs } from "react/jsx-runtime";
const singleChildElement = jsx("h1", {
  className: "greeting",
  children: "Hello, world!",
});
const multipleChildrenElement = jsxs("div", {
  children: [
    jsx("h1", { className: "greeting", children: "Hello, world!" }),
    jsx("p", { children: "This is a paragraph." }),
  ],
});
```

전자는 JSX이고 후자는 바닐라라고 표현합니다.

### 장단점

아무래도 가독성이 좋다는게 가장 큰 장점입니다. 이거 하나가 제일 큽니다.

단점은 JSX를 해석하기 위해 추가적인 의존성이 필요하고, 뷰단에 로직과 표현이 섞여서 복잡해질 수 있다는 점입니다.

하지만 이 단점들은 현대적인 개발 환경에서는 크게 문제가 되지 않는 경우가 많다고 생각합니다. 개발이 편한게 최고죠.

## JSX의 내부 동작

JSX는 브라우저가 직접 이해할 수 없는 문법이므로, JavaScript로 변환하는 과정이 필요합니다.

### 변환 과정 (4단계)

```
JSX 코드 작성
  ↓
1. 토크나이징: 코드를 토큰으로 분해
  ↓
2. 파싱: 토큰을 AST(추상 구문 트리)로 변환
  ↓
3. 코드 생성: AST를 JavaScript 함수 호출로 변환
  ↓
4. 실행: JavaScript 엔진이 실행 → React 엘리먼트 생성 → DOM 렌더링
```

### 변환 예시

```jsx
// 작성한 JSX
<div className="box">Hello, {name}!</div>;

// 변환된 JavaScript
_jsx("div", {
  className: "box",
  children: ["Hello, ", name, "!"],
});
```

### JSX 변환 도구

JSX를 JavaScript로 변환하는 트랜스파일러는 프로젝트 환경에 따라 다릅니다.

#### 1. Babel

- **특징**: 가장 전통적이고 안정적인 JavaScript 트랜스파일러
- **사용처**: Create React App (CRA), 레거시 프로젝트
- **장점**: 플러그인 생태계가 풍부하고 안정적
- **단점**: 상대적으로 느린 빌드 속도

#### 2. SWC (Speedy Web Compiler)

- **특징**: Rust 기반으로 Babel보다 20배 이상 빠름
- **사용처**: Next.js 12 이상 (기본 컴파일러)
- **장점**: 매우 빠른 컴파일 속도, TypeScript 기본 지원
- **단점**: 일부 Babel 플러그인 미지원

#### 3. Vite (ESBuild)

- **특징**: Go 언어 기반 ESBuild 사용, 초고속 빌드
- **사용처**: 최신 SPA 프로젝트, 빠른 개발 환경이 필요한 경우
- **장점**: 개발 서버 시작이 매우 빠르고 HMR 속도 우수
- **단점**: 비교적 새로운 도구라 생태계가 작음

### 핵심 포인트

- `@babel/preset-react`는 React 라이브러리에 포함되지 않음 (별도 도구)
- 최신 프레임워크는 속도를 위해 SWC, ESBuild 같은 도구 사용
- JSX → JavaScript 변환 → React 엘리먼트 객체 생성 → DOM 렌더링
