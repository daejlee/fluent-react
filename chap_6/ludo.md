# Week 5. 서버 사이드 리액트

## 6.1 클라이언트 사이드 렌더링 한계

**클라이언트 사이드 렌더링(CSR)** 은 2013년 리액트 오픈 소스 공개 이래 UI를 작성하는 주요 방법이었지만, 세 가지 주요 한계가 드러나기 시작했다.

### 6.1.1 검색 엔진 최적화 (SEO)

CSR 앱에서 서버는 거의 비어 있는 HTML 파일만 전송하고, 콘텐츠는 자바스크립트가 실행된 후 동적으로 렌더링된다.

- 일부 검색 엔진 크롤러는 자바스크립트를 실행하지 않아 콘텐츠를 색인하지 못할 수 있다.
- 구글·빙은 어느 정도 자바스크립트를 실행하지만, 크롤러 구현은 대부분 비공개라 불확실성이 남는다.
- SSR을 사용하면 크롤러가 완전히 렌더링된 HTML을 바로 읽을 수 있어 SEO가 개선된다.

### 6.1.2 성능

CSR은 콘텐츠를 표시하기 전에 자바스크립트를 다운로드 → 파싱 → 실행해야 하는 **네트워크 폭포(network waterfall)** 문제가 있다.

```
CSR 흐름:
HTML 로딩(빈 페이지) → 자바스크립트 로딩 → 초기 UI 렌더링 → 데이터 페치 → UI 업데이트

HTML 로딩(빈 페이지)
    └─▶ 자바스크립트 로딩
            └─▶ 초기 UI 렌더링
                    └─▶ 데이터 가져오기(페치)
                                └─▶ UI 업데이트

---

SSR 흐름:
HTML 로딩(전체 UI + 서버에서 가져온 데이터 포함) → 완료
```

- 리액트 18 기준 리액트 + 리액트 DOM 번들 크기는 약 **136 KB** — 느린 네트워크·저전력 기기에서 로딩이 지연된다.
- **인터랙티브 가능 시간(Time to Interactive)** 이 길어지면 이탈률이 높아지고 검색 순위에 부정적 영향을 미친다.

### 6.1.3 보안

CSR은 서버와 클라이언트 간에 공유하는 공통 비밀이 없어 **CSRF(Cross-Site Request Forgery)** 공격에 취약할 수 있다.

---

## 6.2 서버 렌더링의 부상

### 6.2.1 서버 렌더링의 장점

| 장점 | 설명 |
|------|------|
| **FMP(First Meaningful Paint)** | 서버가 완전히 렌더링된 HTML을 전송해 사용자가 즉시 콘텐츠를 볼 수 있다 |
| **접근성 향상** | 느린 네트워크·저전력 기기 사용자도 콘텐츠를 즉시 수신한다 |
| **SEO 개선** | 검색 엔진 크롤러가 완전히 렌더링된 HTML을 직접 읽을 수 있다 |
| **보안 향상** | CSRF 방지 토큰을 서버에서 생성·전송해 보안 계약을 수립할 수 있다 |

단, 서버에서 렌더링된 HTML은 **정적**이며 이벤트 리스너가 없어 사용자 상호 작용이 불가능하다. 동적 기능을 활성화하려면 **하이드레이션**이 필요하다.

---

## 6.3 하이드레이션

**하이드레이션(Hydration)** 은 서버에서 생성된 정적 HTML에 이벤트 리스너와 자바스크립트 기능을 추가하는 프로세스다.

### 하이드레이션 단계

1. **클라이언트 번들 로딩** — 브라우저가 정적 HTML을 렌더링하는 동안 자바스크립트 번들을 다운로드·파싱한다.
2. **이벤트 리스너 추가** — `react-dom`의 `hydrateRoot` 함수를 사용해 DOM에 이벤트 리스너를 연결한다.

```tsx
// 클라이언트 사이드 하이드레이션
import React from "react";
import { hydrateRoot } from "react-dom/client";
import App from "./App";

hydrateRoot(document, <App />);
```

> **중요**: 서버에서 생성된 DOM 구조와 리액트 컴포넌트의 JSX 구조가 반드시 일치해야 한다. 불일치하면 이벤트 리스너를 올바르게 연결할 수 없어 예기치 않은 동작이 발생한다.

### 6.3.1 하이드레이션에 대한 비판

하이드레이션은 서버 렌더링 후 클라이언트 번들을 내려받아 **사실상 다시 렌더링**해야 하기 때문에 느리다는 비판이 있다. 대안으로 **재개 가능성(Resumability)** 이 제시된다.

| 구분 | 하이드레이션 | 재개 가능성 |
|------|-------------|------------|
| **동작 방식** | 서버 렌더링 후 클라이언트에서 이벤트 리스너 재연결 (사실상 리렌더링) | 인터랙티브 동작을 직렬화해 클라이언트로 전송, 서버 중단 지점에서 재개 |
| **하이드레이션 단계** | 필요 | 불필요 |
| **복잡도** | 상대적으로 낮음 | 높음 |
| **인터랙티브 시간(TTI)** | 상대적으로 느림 | 더 빠름 |

재개 가능성의 장점은 명확하지만, 구현 복잡성 대비 이점이 더 큰지에 대해 리액트 커뮤니티에서 논란이 계속되고 있다.

---

## 6.4 서버 렌더링 작성

실제 프로젝트에서는 **Next.js** 또는 **Remix** 같은 프레임워크를 사용하는 것이 권장된다. 아래는 교육 목적으로 수동으로 추가하는 방법이다.

### 6.4.1 Express로 수동 SSR 추가

```js
// server.js
const express = require("express");
const path = require("path");
const React = require("react");
// 서버 사이드 렌더링을 위해 ReactDOMServer 가져오기
const ReactDOMServer = require("react-dom/server");
const App = require("./src/App");

const app = express();

app.use(express.static(path.join(__dirname, "build")));

app.get("*", (req, res) => {
  // App 컴포넌트를 렌더링해 HTML 문자열을 생성합니다.
  const html = ReactDOMServer.renderToString(<App />);

  // 렌더링된 App 컴포넌트가 포함된 HTML 응답을 전송합니다.
  res.send(`
    <!DOCTYPE html>
    <html>
      <head><title>예시 리액트 애플리케이션</title></head>
      <body>
        <!-- 렌더링된 App 컴포넌트를 여기에 삽입 -->
        <div id="root">${html}</div>
        <!-- 메인 자바스크립트 번들을 연결 -->
        <script src="/static/js/main.js"></script>
      </body>
    </html>
  `);
});

app.listen(3000, () => {
  console.log("Server listening on port 3000");
});
```

서버를 실행한 후 페이지 소스를 확인하면, 빈 HTML 대신 **완전히 렌더링된 마크업**이 표시된다.

### 6.4.2 하이드레이션

서버에서 렌더링된 HTML이 클라이언트에 전달되고, `<script>` 태그를 통해 번들이 로드될 때 하이드레이션이 발생한다.

```tsx
import React from "react";
import { hydrateRoot } from "react-dom/client";
import App from "./App";

hydrateRoot(document, <App />);
```

---

## 6.5 리액트의 서버 렌더링 API

### 6.5.1 renderToString

`renderToString`은 리액트 컴포넌트를 **동기적**으로 HTML 문자열로 변환하는 가장 기본적인 SSR API다.

```tsx
import React from "react";
import { renderToString } from "react-dom/server";

// 서버에서 렌더링할 간단한 리액트 컴포넌트
function App() {
  return (
    <div>
      <h1>안녕하세요!</h1>
      <p>리액트 앱 예시입니다.</p>
    </div>
  );
}

// App 컴포넌트를 인수로 전달하면 완전히 렌더링된 HTML 문자열을 반환한다
const html = renderToString(<App />);
// → '<div><h1>안녕하세요!</h1><p>리액트 앱 예시입니다.</p></div>'

console.log(html);
```

**변환 흐름:**

```
JSX → React.createElement → React 엘리먼트 → renderToString → HTML 문자열
```

**동작 방식:**

`renderToString`은 리액트 엘리먼트 트리를 순회하며 각 엘리먼트를 실제 DOM에 해당하는 **문자열 표현**으로 변환한 뒤, 전체 결과를 하나의 문자열로 반환한다.

JSX는 내부적으로 `React.createElement` 호출로 컴파일되며, 이 호출은 타입·props·자식으로 구성된 리액트 엘리먼트 객체를 반환한다:

`renderToString`은 이 엘리먼트 객체들의 트리를 재귀적으로 탐색하며 HTML 문자열로 직렬화한다. 중첩 구조의 변환 예시는 다음과 같다:

```js
// 중첩된 React.createElement 호출로 구성된 엘리먼트 트리
React.createElement(
  "section",
  { id: "list" },
  React.createElement("h1", {}, "내가 만든 목록!"),
  React.createElement(
    "p",
    {},
    "굉장하지 않나요? 어마어마한 걸 담고 있거든요!"
  ),
  React.createElement(
    "ul",
    {},
    // amazingThings 배열을 map으로 순회하여 각 아이템을 li 엘리먼트로 변환
    amazingThings.map((t) => React.createElement("li", { key: t.id }, t.label))
  )
);
```

위 엘리먼트 트리는 `renderToString`을 통해 아래 HTML 문자열로 변환된다:

```html
<section id="list">
  <h1>내가 만든 목록!</h1>
  <p>굉장하지 않나요? 어마어마한 걸 담고 있거든요!</p>
  <ul>
    <li>아이템 1</li>
    <li>아이템 2</li>
    <li>아이템 3</li>
  </ul>
</section>
```

> **핵심**: 리액트 엘리먼트는 선언적 추상화이므로 엘리먼트 트리는 어떤 형태의 트리로도 변환될 수 있다. `renderToString`은 이 원리를 활용해 리액트 엘리먼트 트리를 HTML 엘리먼트 트리의 문자열 표현으로 변환한다.

**renderToString이 동기식인 이유와 한계:**

`renderToString`은 **흐름을 가로막는 동기식 API**다. 실행이 시작되면 완료될 때까지 중단하거나 일시 정지할 수 없다. 컴포넌트 트리가 깊을수록 처리 시간이 길어지며, 이 시간 동안 서버의 이벤트 루프는 완전히 차단된다.

```
renderToString 실행 중
│
├── 이벤트 루프 차단 (신규 요청 처리 불가)
├── 컴포넌트 트리 전체를 동기적으로 순회·직렬화
└── 완전한 HTML 문자열 반환 후 이벤트 루프 재개
```

적절한 캐싱 전략 없이 다수의 클라이언트가 동시에 접속하면, 모든 요청마다 `renderToString`이 이벤트 루프를 차단해 서버가 빠르게 과부하 상태에 이를 수 있다.

**단점:**

| 단점 | 설명 |
|------|------|
| **성능** | 동기식이므로 이벤트 루프를 차단한다. 대규모 앱·다수 동시 클라이언트 환경에서 서버 과부하 가능 |
| **메모리** | 완전히 렌더링된 HTML 문자열 전체를 메모리에 유지하므로 대규모 앱에서 메모리 사용량이 급증할 수 있다 |
| **스트리밍 미지원** | HTML 전체가 생성될 때까지 전송할 수 없어 첫 번째 바이트 시간(TTFB)이 느려진다 |
| **비동기 미지원** | 비동기 데이터 페치를 기다릴 수 없어 네트워크 폭포 문제가 발생할 수 있다 |

### 6.5.2 renderToPipeableStream

`renderToPipeableStream`은 리액트 18에 도입된 **비동기 스트리밍 SSR API**로, Node.js 스트림을 반환한다. `Suspense`를 완벽 지원하며, HTML 청크를 준비되는 즉시 클라이언트로 전송해 TTFB(첫 번째 바이트 시간)를 크게 단축한다.

**기본 사용법:**

```js
// server.js (renderToPipeableStream 사용)
const { pipe } = ReactDOMServer.renderToPipeableStream(<App />, {
  // onShellReady: Suspense 경계 바깥의 HTML(셸)이 준비되면 호출되는 콜백
  onShellReady: () => {
    // 응답 Content-Type을 HTML로 설정한다
    res.setHeader("Content-Type", "text/html");
    // 리액트 읽기 가능 스트림을 Express 응답(쓰기 가능 스트림)에 파이프 연결한다
    pipe(res);
  },
});
```

**내부 동작 방식:**

`renderToPipeableStream`은 다음 단계로 동작한다:

1. **요청 생성**: 리액트 엘리먼트와 옵션 객체를 받아 `createRequestImpl`로 요청 객체를 생성한다. 요청 객체에는 리액트 엘리먼트, 리소스, 응답 상태, 포맷 컨텍스트가 캡슐화된다.

2. **작업 시작**: `startWork` 함수로 **비동기 렌더링 프로세스**를 시작한다. `renderToString`과 달리 이벤트 루프를 차단하지 않으며, 필요에 따라 중단하고 재개할 수 있다.

3. **Suspense 처리**: 컴포넌트가 `Suspense` 경계로 감싸여 있고 데이터 페치 같은 비동기 작업을 시작하면, 해당 컴포넌트의 렌더링이 **일시 중단**된다. 중단된 동안은 폴백(fallback)이 먼저 렌더링되고, 작업이 완료되면 컴포넌트가 **재개**되어 실제 콘텐츠로 교체된다.

4. **스트림 반환**: `pipe`와 `abort` 메서드를 포함한 객체를 반환한다.
   - `pipe(destination)`: 렌더링 출력을 쓰기 가능 스트림(응답 객체)으로 연결한다.
   - `abort(reason)`: 대기 중인 모든 I/O 작업을 취소하고 나머지를 클라이언트 렌더링 모드로 전환한다.

5. **스트림 이벤트 처리**: 목적지 스트림의 `drain`, `error`, `close` 이벤트를 감지한다. `drain` 시 데이터 전송을 재개하고, `error`/`close` 시 렌더링을 중단한다.

```
renderToPipeableStream 실행 흐름
│
├── 1. 요청 객체 생성 (createRequestImpl)
├── 2. 비동기 렌더링 시작 (startWork) — 이벤트 루프 비차단
│   ├── Suspense 미포함 컴포넌트 → 즉시 HTML 청크 생성
│   └── Suspense 포함 컴포넌트 → 폴백 먼저 렌더링
│       └── 데이터 준비 완료 → 실제 콘텐츠로 스트리밍 교체
├── 3. onShellReady 콜백 호출 (셸 준비 완료)
│   └── pipe(res) → HTML 청크를 클라이언트에 점진적 전송
└── 4. Suspense 데이터 도착 시 나머지 콘텐츠 스트리밍
```

**Suspense + renderToPipeableStream 전체 예시:**

```tsx
const dogResource = createResource(
  fetch("https://dog.ceo/api/breeds/list/all")
    .then((r) => r.json())
    .then((r) => Object.keys(r.message))
);

function DogBreeds() {
  return (
    <ul>
      <Suspense fallback="Loading...">
        {dogResource.read().map((breed) => (
          <li key={breed}>{breed}</li>
        ))}
      </Suspense>
    </ul>
  );
}

export default DogBreeds;
```

```tsx
import React, { Suspense } from "react";

const ListOfBreeds = React.lazy(() => import("./DogBreeds"));

function App() {
  return (
    <div>
      <h1>견종</h1>
      {/* 데이터 또는 코드 로딩 중에는 폴백 컴포넌트를 표시한다 */}
      <Suspense fallback={<div>견종 목록을 불러오는 중...</div>}>
        <ListOfBreeds />
      </Suspense>
    </div>
  );
}

export default App;
```

```js
// server.js
import express from "express";
import React from "react";
import { renderToPipeableStream } from "react-dom/server";
import App from "./App.jsx";

const app = express();
app.use(express.static("build"));

app.get("/", async (req, res) => {
  // HTML 문서의 시작 부분을 문자열로 정의한다
  const htmlStart = `
    <!DOCTYPE html>
    <html lang="ko">
      <head>
        <meta charset="UTF-8" />
        <meta name="viewport" content="width=device-width, initial-scale=1.0" />
        <title>React Suspense + renderToPipeableStream</title>
      </head>
      <body>
        <div id="root">`;

  // HTML 시작 부분을 먼저 클라이언트에 전송한다 (브라우저가 즉시 파싱 시작 가능)
  res.write(htmlStart);

  // renderToPipeableStream으로 App 컴포넌트를 비동기 스트림으로 렌더링한다
  const { pipe } = renderToPipeableStream(<App />, {
    // onShellReady: Suspense 경계 바깥의 HTML(셸)이 준비되면 호출된다
    onShellReady: () => {
      // 준비된 HTML 스트림을 Express 응답 스트림에 파이프 연결한다
      pipe(res);
    },
  });
});

app.listen(3000, () => {
  console.log("Server is listening on port 3000");
});
```

**실행 시 클라이언트가 수신하는 HTML:**

페이지에 처음 접속하면 서버는 Suspense 폴백이 포함된 HTML **셸**을 즉시 전송하고, 데이터가 준비되면 실제 콘텐츠를 추가로 스트리밍한다:

```html
<!DOCTYPE html>
<html lang="ko">
<head>...</head>
<body>
  <div id="root">
    <div>
      <h1>사용자 프로필</h1>
      <!-- Suspense 경계의 시작을 표시하는 주석 마커 -->
      <!--$?--><template id="B:0"></template>
      <!-- 데이터 로딩 중 표시되는 폴백 UI -->
      <div>사용자 프로필 로딩 중</div>
      <!-- Suspense 경계의 끝 마커 -->
      <!--/$-->
    </div>

    <!-- 데이터가 준비되면 이 hidden div의 내용으로 폴백을 교체한다 -->
    <div hidden id="S:0">
      <ul>
        <!--$-->
        <li>아펜핀셔</li>
        <li>아프리칸</li>
        <li>에어데일</li>
        <!-- ... -->
        <!--/$-->
      </ul>
    </div>

    <!-- $RC 함수: Suspense 폴백을 실제 콘텐츠로 교체하는 인라인 스크립트 -->
    <script>
      function $RC(a, b) { /* ... */ }
      // "B:0" 마커 위치의 폴백을 "S:0" 콘텐츠로 교체한다
      $RC("B:0", "S:0");
    </script>
  </div>
</body>
</html>
```

**내부 동작 (`$RC` 함수):**

서버는 HTML에 주석 마커(`<!--$?-->`)와 `<template>` 엘리먼트를 사용해 Suspense 경계 위치를 표시한다. 데이터가 준비되면 실제 콘텐츠를 담은 `hidden` div와 함께 인라인 `<script>`가 스트리밍되고, `$RC` 함수가 폴백을 실제 콘텐츠로 교체한다:

```js
function reactComponentCleanup(reactMarkerId, siblingId) {
  // reactMarkerId: Suspense 경계를 표시하는 template 엘리먼트의 ID (예: "B:0")
  // siblingId: 서버에서 렌더링된 실제 콘텐츠를 담은 hidden div의 ID (예: "S:0")

  let reactMarker = document.getElementById(reactMarkerId);
  let sibling = document.getElementById(siblingId);

  // 1. 실제 콘텐츠를 담은 hidden div를 DOM에서 제거한다
  sibling.parentNode.removeChild(sibling);

  if (reactMarker) {
    reactMarker = reactMarker.previousSibling;
    let parentNode = reactMarker.parentNode;
    let nextSibling = reactMarker.nextSibling;
    let nestedLevel = 0;

    // 2. do...while 루프로 폴백 영역의 노드들을 순회하며 DOM에서 제거한다
    //    주석 노드의 데이터를 확인해 Suspense 경계의 시작/끝을 파악한다:
    //    - "/$": 경계의 끝 → nestedLevel이 0이면 루프 종료
    //    - "$", "$?", "$!": 중첩 경계의 시작 → nestedLevel 증가
    do {
      if (nextSibling && 8 === nextSibling.nodeType) { // 주석 노드(nodeType 8)인지 확인
        let nodeData = nextSibling.data;
        if ("/$" === nodeData) {
          if (0 === nestedLevel) break; // Suspense 경계 끝에 도달, 루프 종료
          else nestedLevel--;           // 중첩 경계 종료, 레벨 감소
        } else if ("$" !== nodeData && "$?" !== nodeData && "$!" !== nodeData) {
          nestedLevel++; // 새로운 중첩 Suspense 경계 진입, 레벨 증가
        }
      }
      let nextNode = nextSibling.nextSibling;
      parentNode.removeChild(nextSibling); // 폴백 노드를 DOM에서 제거
      nextSibling = nextNode;
    } while (nextSibling);

    // 3. 실제 콘텐츠(sibling의 자식들)를 폴백이 있던 위치에 삽입한다
    while (sibling.firstChild) {
      parentNode.insertBefore(sibling.firstChild, nextSibling);
    }

    // 4. 마커 데이터를 "$"로 업데이트해 렌더링 완료를 표시한다
    reactMarker.data = "$";
    // _reactRetry가 있으면 호출하여 리액트 내부 재시도 로직을 실행한다
    reactMarker._reactRetry && reactMarker._reactRetry();
  }
}

// "B:0" 위치의 폴백을 "S:0"의 실제 콘텐츠로 교체한다
reactComponentCleanup("B:0", "S:0");
```

> **핵심**: 이 교체 과정 전체가 서버가 스트리밍한 HTML 안의 인라인 스크립트로 처리된다. 따라서 **클라이언트 사이드 리액트나 하이드레이션 없이도** Suspense 폴백이 실제 콘텐츠로 교체될 수 있다. `renderToPipeableStream`의 가장 강력한 장점이다.

**renderToPipeableStream의 주요 기능:**

| 기능 | 설명 |
|------|------|
| **스트리밍** | 전체 렌더링 전에 HTML 청크를 클라이언트에 전송해 TTFB 단축 |
| **비동기 렌더링** | 이벤트 루프를 차단하지 않아 다수의 동시 클라이언트 처리 가능 |
| **유연성** | Node.js 스트림과 쉽게 통합, 렌더링 파이프라인을 세밀하게 제어 |
| **Suspense 지원** | 비동기 데이터 페치·지연 로딩을 서버 렌더링 중 처리 |
| **점진적 향상** | 데이터가 준비되는 순서대로 콘텐츠를 스트리밍해 사용자 경험 개선 |

### 6.5.3 renderToReadableStream

`renderToReadableStream`은 `renderToPipeableStream`과 유사하지만, Node.js 스트림 대신 **브라우저 네이티브 스트림(WHATWG Streams API)** 을 반환한다.

| 구분 | Node.js 스트림 | 브라우저 스트림 |
|------|---------------|----------------|
| **환경** | 서버 (Node.js) | 브라우저 (클라이언트) |
| **표준** | Node.js 자체 API | WHATWG Streams API |
| **방식** | 이벤트 기반 | 프로미스 기반 |
| **리액트 API** | `renderToPipeableStream` | `renderToReadableStream` |

### 6.5.4 언제 무엇을 사용해야 하나요?

`renderToString`이 적합하지 않은 이유:

- **네트워크 I/O는 비동기** — 동기식인 `renderToString`은 비동기 요청 완료를 기다리지 못해 네트워크 폭포가 발생한다.
- **다수 클라이언트 처리 시 블로킹** — 렌더링 중 새 요청이 들어오면 기존 렌더링이 완료될 때까지 대기해야 한다.

```
환경에 따른 API 선택:
- 서버(Node.js) 환경 → renderToPipeableStream
- 브라우저/엣지 환경 → renderToReadableStream
- 단순한 경우·호환성 우선 → renderToString
```

> **주의**: 현재 많은 서드파티 라이브러리(데이터 페치, CSS 라이브러리 등)가 스트리밍 API와 호환되지 않는다. 실무에서는 프레임워크를 사용하는 것이 현실적이다.

---

## 6.6 직접 구현하지 마세요

서버 렌더링을 직접 구현하는 것은 복잡하고 다양한 문제가 따른다. **Next.js**, **Remix** 같은 프레임워크 사용을 권장하는 이유는 다음과 같다.

**에지 케이스 및 보안 처리:**

직접 캐싱을 구현할 때 발생할 수 있는 보안 취약점 예시:

```js
// 잘못된 캐싱 — 모든 사용자에게 같은 캐시를 반환하는 보안 문제
let cachedUserData = null;

app.get("/user/:userId", (req, res) => {
  if (cachedUserData) {
    return res.json(cachedUserData); // userId 검증 없이 캐시 반환 — 심각한 보안 문제!
  }
  const userData = fetchUserData(req.params.userId);
  cachedUserData = userData;
  res.json(userData);
});
```

`/user/1` 다음에 `/user/2` 요청이 오면, 두 번째 사용자가 첫 번째 사용자의 데이터를 수신하게 된다.

**프레임워크를 사용해야 하는 이유:**

| 이유 | 설명 |
|------|------|
| **에지 케이스 처리** | 비동기 데이터 페치, 코드 분할, 수명 주기 관리 등 복잡한 케이스가 내장 처리됨 |
| **보안** | 안전한 데이터 격리·캐싱 전략이 기본으로 적용됨 |
| **성능 최적화** | 자동 코드 분할, 경로 기반 번들링, 캐싱 등이 기본 제공됨 |
| **개발자 경험** | 인프라보다 기능 구현에 집중할 수 있어 생산성이 향상됨 |
| **모범 사례** | 커뮤니티 검증된 코딩 규칙과 패턴을 기본으로 적용할 수 있음 |
