RSC는 SSR MPA와 CSR SPA의 장점을 결합했다. RSC는 서버에서 실행되는 새로운 종류의 컴포넌트로, 클라이언트 JS 번들에 포함되지 않는다. RSC는 빌드 시점에 실행될 수 있어 다양한 작업이 가능하다.

다시 짚어보자면 리액트 컴포넌트는 가상 DOM을 반환하는 함수다. RSC도 서버에서만 호출되어 리액트 엘리먼트를 의미하는 JS 객체를 반환한다. 반환 결과는 네트워크를 통해 클라로 전송된다.

# 9.1 장점

- 서버에서 실행되어 성능 예측이 클라에 비해 쉽다
- 서버 환경이므로 토큰 같은 정보가 유출될 걱정이 없다
- 비동기로 동작 가능하여 클라로 전송하기 전에 RSC의 동작이 끝날 때까지 await 가능하다.

# 9.2 서버 렌더링

기본적으로 RSC와 SSR을 독립적인 프로세스로 생각 가능하다. 하나는 서버에서 컴포넌트를 렌더링해 리액트 엘리먼트 트리를 생성하고, 하나는 이를 마크업으로 변환하여 HTML 문자열이나 스트림으로 렌더링한다.

## 9.2.1 내부 구조

`turnServerComponentsIntoTreeOfElements` 함수가 RSC를 담당하는데, 이는 사실 리액트 렌더러다.

```tsx
async function turnServerComponentsIntoTreeofElements(jsx) {
	...
	if (Array.isArray(jsx)) {
		// 배열의 각 항목 처리 [<div>hi</div>, <h1>hello</h1> ...]
		return Promise.all(jsx.map(child => rednerJSXToClientJSX(child)));
	}

	if (jsx != null && typeof jsx === "object") {
		if (jsx.$$typeof === Symbol.for("react.element")) {
			if (typeof jsx.type === "string" ){
				// div같은 내장 컴포넌트, 그대로 반환
				return {jsx, props: await renderJSXToClientJSX(jsx.props),
			};
			if (typeof jsx.type === "function") {
				// <Footer /> 같은 작성된 리액트 컴포넌트인 경우,
				// 함수를 호출하고 반환되는 JSX를 반복해서 처리
				const Component = jsx.type;
				const props = jsx.props;
				const returnedJsx = await Component(props);
				return await renderJSXToClientJSX(returnedJsx);
			}
			throw new Error("Not implemented.");
		...
	}
```

이제 해당 엘리먼트 트리를 `renderToString`이나 `renderToPipeableStream`으로 전달하거나 직렬화해 클라로 전송할 수 있다. 다만 직렬화가 남았다.

### 직렬화

리액트 엘리먼트를 문자열로 변환하는 과정을 직렬화라고 한다.

```tsx
const element = <h1>Hello, world</h1>;
const htmlString = ReactDOMServer.renderToString(element); // htmlString은 똑같다.
```

직렬화는 서버가 즉시 표시 가능한 HTML 페이지를 빠르게 클라에 전송할 수 있게한다. 또한 리액트 엘리먼트를 HTML 문자열로 직렬화하면 환경에 구애받지 않는 일관된 초기 렌더링이 가능하다. 마지막으로 직렬화는 클라측의 하이드레이션을 용이하게 한다. (직렬화된 HTML 문자열을 초기 마크업으로 사용)

컴포넌트도 직렬화가 필요하지만 리액트 엘리먼트는 평범한 JS 객체가 아니어서 JSON.stringify만으로는 직렬화가 불가능하다.

<aside>

    💡 왜 불가능?

    리액트가 엘리먼트 식별에 사용하는 $$typeof 속성을 지닌 객체인데, 이 속성값은 심벌이다. 심벌은 직렬화하여 네트워크로 전송할 수 없다.

</aside>

Node.js와 브라우저는 JS 런타임을 포함해 이들에게 어려운 작업은 아니다. JSON.stringify와 JSON.parse로 재귀적으로 리액트 엘리먼트를 JSON 객체로 직렬화, 역직렬화 한다. (`JSON.stringify(object, replacer)`)

### 페이지 탐색

RSC를 지원하는 애플리케이션에서 `<a href=”/blog”>블로그</a>`를 클릭해서 페이지 탐색을 할 수 있다. 전체 페이지 탐색이 발생해 PHP처럼 느린 느낌을 준다. RSC는 소프트 네비게이션을 구현할 수 있으며 이 케이스 경로가 변경되어도 상태가 유지된다.

<aside>

    💡 하드 네비게이션: 페이지 완전 새로고침
    소프트 네비게이션: 필요한 데이터만 불러와 새 페이지 표시

</aside>

다만 이를 위해서 클라 코드를 수정해야한다.

```tsx
import App from "./App";

const root = hydrateRoot(document, <App />);

window.addEventListener("click", (event) => {
	// window에 이벤트 위임
	if (event.target.tagName !== "A") {
		return;
	}

	event.preventDefault();
	navigate(event.target.href);
});

async function navigate(url) {
	const res = await fetch(url, { headers: { "jsx-only": true } });
	const jsxTree = await res.json();
	const element = JSON.parse(jsxTree, ...);

	root.render(element);
}
```

![image.png](https://prod-files-secure.s3.us-west-2.amazonaws.com/907641dc-2bf8-4fd3-8c6a-84e5873017ac/211c00f1-f19c-4a9c-bff3-b035ff8a5c21/image.png)

서버에서는 jsx-only 헤더를 받으면 전체 HTML 문자열을 반환하지 않고 다음 페이지의 JSX 트리 객체만 응답하게끔 해야한다.

이렇게 페이지 네비게이션은 처리했는데, 업데이트는?

## 9.2.2 업데이트

RSC에 장점이 많지만, 서버 컴포넌트와 클라이언트 컴포넌트 두 종류의 컴포넌트를 고려해야 한다.

<aside>

    💡

    ![image.png](attachment:01bb268c-f0dc-4488-807f-61fe59802134:image.png)

    왜 이 컴포넌트는 서버 컴포넌트가 안되나요?

    - 클라이언트 전용 API useState를 사용
    	- useState가 왜?: 서버는 상태 개념이 여러 클라에 걸쳐 공유됨. 보안 위험이 되기 때문에 RSC는 서버에서 useState 사용을 지원하지 않음.
    	- useState에서 반환하는 setState 디스패처 함수는 직렬화가 안됨
    - 함수는 왜 직렬화가 안되나요 - 함수의 클로저와 스코프 전체를 추적해 직렬화 하는게 현실적으로 불가능 - 서버 실행 함수와 클라 실행 함수가 다름. NodeAPI와 WebAPI의 차이처럼

</aside>

### 내부 동작

리액트가 서버 컴포넌트와 클라 컴포넌트를 내부적으로 어떻게 구분하고 처리하는지? 차세대 도구가 필요하긴 하다. 이 차세대 번들러를 사용하면 번들러는 리액트 앱에 대해 서버 그래프와 클라 그래프의 분리된 모듈 그래프를 생성할 수 있다.

서버 그래프는 번들에 포함되지 않고, “use client”로 시작하는 모든 파일은 하나의 클라 번들에 포함되거나 여러 번들로 나뉘어 지연 로딩될 수 있다. 그렇다면 리액트는 언제 어느 클라 컴포넌트를 가져와 실행할지?

서버에서는 클라 컴포넌트를 위한 플레이스홀더를 렌더링한다. 이는 클라 번들러가 생성한 특정 모듈에 대한 레퍼런스다. (ex. `getModuleFromBundleAtPosition([0,4]`)

## 9.2.3 주의할 점

서버 컴포넌트는 서버에서만 실행, 클라 컴포넌트는 클라에서만 실행? → No

뒤가 틀렸다. 클라 컴포넌트는 클라에서만 실행되지 않는다. (”함수가 클라에서만 호출되지 않는다”)

- 서버 컴포넌트는 서버에서 실행, 리액트 엘리먼트를 나타내는 객체를 출력
- 클라 컴포넌트는 서버에서 실행, 리액트 엘리먼트를 나타내는 객체를 출력
- 서버에는 클라 컴포넌트와 서버 컴포넌트의 모든 리액트 엘리먼트를 표현하는 객체 존재
- 이는 문자열로 변환되어 클라로 전송
- 이떄부터 서버 컴포넌트는 클라에서 실행되지 않고, 클라 컴포넌트는 클라에서만 실행됨

# 9.3 서버 컴포넌트 규칙

## 9.3.1 직렬화 가능성이 가장 중요

함수 등 직렬화 불가한 값을 prop으로 사용 못함 (5의 렌더 프롭같은거)

필요하다면 함수를 클라 컴포넌트로 캡슐화

## 9.3.2 side effect가 있는 훅 금지

서버 환경은 클라와 완전히 다르다. 인터렉티브 하지 않고 DOM, window 객체가 없다.

상태 변경, DOM 업데이트, 데이터 페칭 등의 사이드이펙트가 있는 훅은 사용 불가하다.

## 9.3.3 상태는 동일하지 않다

서버 컴포넌트의 상태와 클라 컴포넌트의 상태는 다르다. 서버와 클라는 1:N이므로, 그러므로 useState나 useReducer같은 상태에 관여하는 컴포넌트는 클라 컴포넌트에 떨어진다.

## 9.3.4 클라 컴포넌트는 서버 컴포넌트를 import 할 수 없다

서버 컴포넌트는 서버에서만 실행되는데 이를 클라 컴포넌트에서 불러올 수 없다.

다만 명시적으로 import 하지 않으면 되긴 하는데,

```tsx
'use client';

function ClientComponent({ children }) {
  return (
    <div>
      <h1>내 멋진 서버 컴포넌트를 봐라!</h1>
      {children}
    </div>
  );
}

import { ServerComponent } from './ServerComponent';

async function TheParentOfBothComponents() {
  return (
    <ClientComponent>
      <ServerComponent />
    </ClientComponent>
  );
}
```

위 코드는 문제 없다. 부모 서버 컴포넌트가 서버 컴포넌트를 클라 컴포넌트의 프롭으로 전달한다. import 사용 금지의 이유는 클라 번들에 서버 컴포넌트가 포함되는 것을 방지하는 것이지, 프롭으로 전달되는 컴포넌트 구성에는 주의를 기울이지 않는다.

## 9.3.5 클라 컴포넌트는 나쁘지 않다

나쁘다고 한 적 없음

# 9.4 서버 액션

RSC는 “use server” 지시자로 클라 코드에서 호출할 수 있는 서버 함수를 표시하는데, 이것이 서버 액션이다. 비동기 함수 첫 줄에 “use server”를 추가하면 리액트와 번들러는 함수를 클라에서 호출할 수 있지만 서버에서만 실행되는 함수로 취급한다.

클라에서 서버 액션을 호출할 때, 함수에 전달된 모든 인수가 직렬화되어 네트워크를 통해 서버에 전달된다. 서버 액션이 값을 반환하면 이는 직렬화되어 클라에 반환된다.

JS 번들이 전부 로드되기도 전에 사용 가능하다는 장점이 있다.

<aside>

    💡 의문

    fetch를 서버 액션 쓴다고 할 때, 클라 - FE서버 - 백엔드 서버로 중간에 단계가 하나 더 생긴다.

    서버 투 서버 통신이 빠르다곤 해도 필요한 경우에만 쓰는게 좋겠다.

    백엔드 서버에서 받는 데이터가 너무 방대해서 필터링이 필요할 때 (10MB 데이터를 서버액션 거쳐서 1KB로 줄인다는지?),
    민감한 토큰 값을 다룬다든지

</aside>
