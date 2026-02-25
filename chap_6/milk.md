# 서버 사이드 리액트

## 6.1 CSR 한계

### SEO

일반적인 CSR에서는 html이 비어있기 때문에 검색 엔진이 페이지의 내용을 인덱싱할 수 없습니다. SSR에서는 서버에서 완전한 HTML을 제공하므로 검색 엔진이 페이지의 내용을 쉽게 인덱싱할 수 있습니다.

### 성능

CSR에서는 초기 로드 시 JavaScript 파일을 다운로드하고 실행해야 하므로 페이지가 완전히 렌더링되기까지 시간이 걸릴 수 있습니다. SSR에서는 서버에서 완전한 HTML을 제공하므로 초기 로드가 빠르며, JavaScript가 로드되고 실행되는 동안에도 사용자가 콘텐츠를 볼 수 있습니다.

> 그렇다고 SSR이 항상 CSR보다 빠르다는 것은 아닙니다. SSR은 서버에서 렌더링하는 데 시간이 걸릴 수 있으며, 서버의 부하가 증가할 수 있습니다. 또한, CSR에서는 클라이언트 측에서 렌더링이 이루어지므로, 서버의 부하를 줄일 수 있습니다.

### 보안

클라이언트 사이드에서는 CSRF에 취약할 수 있습니다.
버튼을 눌러 출금하는 로직이 있다고 합시다.

```tsx
const Account = () => {
  const [balance, setBalance] = useState(1000);
  const handleWithdraw = async (amount: number) => {
    fetch("/withdraw", { method: "POST", body: JSON.stringify({ amount }) });
  };

  return <button onClick={() => handleWithdraw(100)}>출금</button>;
};
```

이 경우, 악의적인 웹사이트가 사용자의 브라우저에서 `/withdraw` 엔드포인트로 POST 요청을 보내어 사용자의 계좌에서 돈을 출금할 수 있습니다. SSR에서는 서버에서 렌더링되므로, CSRF 공격에 대한 보호를 구현하기가 더 쉽습니다. 예를 들어, 서버에서 CSRF 토큰을 생성하여 클라이언트에 전달하고, 클라이언트는 이 토큰을 요청 헤더에 포함하여 서버로 보내도록 할 수 있습니다. 즉 신뢰할 수 있는 클라이언트만에서 요청을 받아 처리할 수 있도록 하는것입니다.

> 추가적인 보안으로 찾은거?
>
> - Get 요청과 같은 요청은 서버사이드에서 미리 그려 네크워크 탭에서 노출시키지 않도록 해 보안성을 높일 수 있습니다.
> - Proxy 화하여 API 요청을 서버사이드에서 처리하도록 하여, 클라이언트에서는 API 엔드포인트를 알 수 없도록 할 수 있습니다.

## 6.2 SSR 부상

장점

- 최초 의미있는 페인트가 빠르다.
- 웹 어플리케이션 접근성 개선이 된다.
- SEO 도움이 된다.
- 보안 향상 된다.

> 책에서 소개되지 않은 단점
>
> - 서버를 두게 되어 비용이 든다.
> - 간혹 CSR보다 렌더링이 느린 상황이 발생할 수도 있다.

## 6.3 하이드레이션

사용자와 상호작용하지않는 정적인 정보들을 미리 HTML로 그리는걸 SSR이라고 한다면, 하이드레이션은 SSR로 그려진 HTML에 React의 이벤트 핸들러를 연결하는 과정입니다. 하이드레이션을 통해 서버에서 렌더링된 HTML이 클라이언트에서 React 컴포넌트로 변환되어 사용자와 상호작용할 수 있게 됩니다.

### 재개가능성

하지만 이런 하이드레이션은 사용자가 화면에 버튼이 있지만 눌러도 응답이 없는 현상이 있을 수 있습니다.

이런 문제를 해결하기 위해 Qwik같은 프레임워크에서 재개가능성을 도입했습니다.

1. 서버가 렌더링을 하다가, 특정 버튼에 클릭 이벤트가 있다는 정보를 HTML 태그 안에 속성(Attribute)으로 적어둡니다. (예: on:click="chunk_1.js#handleBtn")
2. 브라우저는 자바스크립트를 미리 실행하지 않습니다.
3. 사용자가 버튼을 실제로 클릭하는 순간, 딱 필요한 그 코드 조각(Chunk)만 가져와서 서버가 멈췄던 지점부터 실행합니다.

### Selective Hydration(React 18)

기존: 전체 페이지의 자바스크립트가 로드될 때까지 기다려야 했습니다.

개선: Suspense로 감싼 부분은 준비가 덜 됐더라도 나머지 부분을 먼저 하이드레이션합니다. 유저가 특정 부분을 클릭하면 그 부분을 우선적으로 처리하는 '지능적인 순서 정하기' 방식입니다.

### PPR(React 19 + Next.js 15)

수정 포인트: "서버 vs 클라이언트"의 이분법보다는 **"정적(Static) vs 동적(Dynamic)"**의 결합으로 이해해야 합니다.

핵심: 페이지의 변하지 않는 부분(네비게이션 등)은 빌드 시점에 미리 HTML로 만들어(Static Shell) 즉시 보여주고, 데이터가 필요한 부분(장바구니, 추천 목록 등)만 서버에서 실시간으로 스트리밍하여 채워넣는 방식입니다.

## 6.4 스킵

## 6.5 React의 SSR API

### renderToString

서버에서 React 컴포넌트를 HTML 문자열로 변환하는 함수입니다. 이 함수는 React 요소를 받아서 완전한 HTML 문자열을 반환합니다. 이 HTML 문자열은 클라이언트로 전송되어 브라우저에서 렌더링됩니다.

```tsx
import { renderToString } from "react-dom/server";
import App from "./App";
const html = renderToString(<App />);
```

동기식으로 작동하기 때문에, 렌더링이 완료될 때까지 서버가 블로킹됩니다. 따라서, 렌더링이 오래 걸리는 컴포넌트가 있다면, 전체 페이지의 응답 시간이 길어질 수 있습니다.

### renderToPipeableStream

서버에서 React 컴포넌트를 스트림으로 렌더링하는 함수입니다. 이 함수는 React 요소를 받아서 스트림을 반환합니다. 이 스트림은 클라이언트로 전송되어 브라우저에서 렌더링됩니다.

```tsximport { renderToPipeableStream } from "react-dom/server";
import App from "./App";
const stream = renderToPipeableStream(<App />);
```

renderToString처럼 유사하게, 선언적으로 ReactNode를 받아서 HTML을 생성하지만, 스트림으로 반환하기 때문에 렌더링이 완료되기 전에 클라이언트로 데이터를 전송할 수 있습니다. 그리고 HTML 문자열이 아닌 Node.js 스트림을 반환하기 합니다.

#### 1. 기본 구조

`renderToPipeableStream`은 크게 **3가지 단계**로 동작합니다:

```
1. 요청 받기 → 2. 스트림 렌더링 → 3. 클라이언트 전송
```

#### 2. 전체 플로우

```tsx
import { renderToPipeableStream } from "react-dom/server";

const { pipe, abort } = renderToPipeableStream(<App />, {
  bootstrapScripts: ["/client.js"],
  onShellReady() {
    // Shell(기본 구조)이 준비되면 호출
    response.statusCode = 200;
    response.setHeader("Content-Type", "text/html");
    pipe(response);
  },
  onAllReady() {
    // 모든 컨텐츠(Suspense 포함)가 준비되면 호출
  },
  onShellError(error) {
    // Shell 렌더링 중 에러 발생시
    response.statusCode = 500;
    response.send("<!doctype html><p>에러 발생</p>");
  },
  onError(error) {
    // 렌더링 중 에러 로깅
    console.error(error);
  },
});
```

#### 3. 내부 동작 순서

**Step 1: Shell 렌더링 (초기 구조)**

```
- React 컴포넌트 트리를 탐색 시작
- Suspense 경계를 만나면 fallback을 먼저 렌더링
- 즉시 렌더링 가능한 부분(Shell)을 먼저 HTML로 변환
- onShellReady() 콜백 호출
```

**Step 2: 스트리밍 시작**

```
- Shell HTML이 클라이언트로 즉시 전송됨
- 브라우저는 받은 부분부터 화면에 표시 시작
- 사용자는 로딩 중에도 일부 UI를 볼 수 있음
```

**Step 3: Suspense 해결 (비동기 처리)**

```
- 데이터 페칭이나 lazy 컴포넌트 로드 진행
- 각 Suspense가 resolve되면 해당 부분 HTML 생성
- <script> 태그로 감싸서 클라이언트에 전송
- 클라이언트는 받은 스크립트를 실행해 placeholder를 실제 컨텐츠로 교체
```

**Step 4: 완료**

```
- 모든 Suspense가 해결되면 onAllReady() 호출
- 스트림 종료
```

#### 4. 실제 예제로 이해하기

```tsx
function App() {
  return (
    <html>
      <head>
        <title>My App</title>
      </head>
      <body>
        <header>헤더</header>
        <Suspense fallback={<div>로딩중...</div>}>
          <AsyncContent />
        </Suspense>
        <footer>푸터</footer>
      </body>
    </html>
  );
}
```

**전송되는 HTML 순서:**

```html
<!-- 1. Shell (즉시 전송) -->
<!DOCTYPE html>
<html>
  <head>
    <title>My App</title>
  </head>
  <body>
    <header>헤더</header>
    <div>로딩중...</div>
    <!-- Suspense fallback -->
    <footer>푸터</footer>

    <!-- 2. 비동기 컨텐츠 (나중에 전송) -->
    <script>
      // React가 자동으로 생성하는 교체 스크립트
      $RC = function (b, c, e) {
        // fallback을 실제 컨텐츠로 교체하는 함수
      };
      $RC("B:0", "<div>실제 컨텐츠</div>");
    </script>
  </body>
</html>
```

#### 5. 핵심 개념 정리

**Segment (세그먼트)**

- 스트림으로 전송되는 HTML 조각
- Shell, Suspense 컨텐츠 각각이 별도 세그먼트

**Boundary (경계)**

- Suspense 컴포넌트가 만드는 경계
- 각 경계는 독립적으로 resolve됨
- 하나가 느려도 다른 부분에 영향 없음

**Progressive Enhancement (점진적 향상)**

```
1. HTML만 있을 때: 기본 컨텐츠 표시 가능
2. HTML + 데이터: Suspense 컨텐츠 교체
3. HTML + 데이터 + JS: 하이드레이션으로 인터랙티브
```

#### 6. renderToString과의 차이

| renderToString                     | renderToPipeableStream          |
| ---------------------------------- | ------------------------------- |
| 동기식 - 모든 렌더링 완료까지 대기 | 비동기식 - 준비된 부분부터 전송 |
| 문자열 반환                        | 스트림 반환                     |
| Suspense 지원 안함                 | Suspense 완벽 지원              |
| TTFB 느림                          | TTFB 빠름                       |
| 간단한 구현                        | 복잡하지만 성능 좋음            |

#### 7. 성능상 이점

```
renderToString: [====서버 렌더링====][전송][클라이언트 표시]
                (5초)              (1초)  (즉시)
                총 6초 후 사용자가 컨텐츠 확인

renderToPipeableStream: [Shell][  비동기  ][완료]
                        (0.5초) (4.5초)
                        [전송→표시][전송→표시][전송→표시]
                        총 0.5초 후 사용자가 일부 컨텐츠 확인
```

#### 8. 주요 내부 메커니즘

1. **Task 스케줄링**: React는 내부적으로 렌더링 작업을 작은 단위(task)로 나눔
2. **Chunk 전송**: 각 task가 완료되면 해당 HTML chunk를 즉시 전송
3. **ID 매핑**: 각 Suspense boundary에 고유 ID 부여하여 나중에 교체할 위치 추적
4. **Script Injection**: 클라이언트에서 실행될 교체 로직을 inline script로 삽입
5. **Error Handling**: 각 boundary별로 독립적인 에러 처리

#### 9. 실무 활용 팁

**언제 사용하면 좋은가?**

- 데이터 페칭이 많은 페이지
- 일부 컨텐츠가 느리게 로드되는 경우
- LCP(Largest Contentful Paint) 개선이 필요할 때
- 사용자 경험을 점진적으로 향상시키고 싶을 때

**주의사항**

- 에러 처리를 반드시 구현해야 함
- SEO가 중요한 컨텐츠는 Shell에 포함시켜야 함
- Suspense boundary를 너무 많이 만들면 오히려 복잡해짐
- 스트리밍을 지원하지 않는 인프라도 있음 (일부 CDN)

### renderToReadableStream

`renderToReadableStream`은 `renderToPipeableStream`과 유사하지만, Node.js 스트림 대신 웹 표준의 ReadableStream을 반환하는 함수입니다. 이 함수는 React 요소를 받아서 ReadableStream을 반환합니다. 이 스트림은 클라이언트로 전송되어 브라우저에서 렌더링됩니다.

edge 런타임(fs, path같은게 없고 fetch같은 브라우저 API만 있는 가벼운.. v8엔진 코어만 있는 런타임)에서 사용하기 위해 설계되었습니다. Node.js의 스트림 API 대신 웹 표준인 ReadableStream을 사용하여, 서버리스 환경에서도 효율적으로 스트리밍 렌더링을 지원합니다.

내부적으로 renderToPipeableStream과 유사한 방식으로 작동하지만 아래처럼 다릅니다.

```jsx
// renderToPipeableStream 예시
const { pipe } = renderToPipeableStream(<App />);
// Node.js 특유의 res 객체에 직접 파이프를 꽂음
pipe(res);

// renderToReadableStream 예시
const stream = await renderToReadableStream(<App />);
// 웹 표준 Response 객체를 생성해서 반환함
return new Response(stream, {
  headers: { "Content-Type": "text/html" },
});
```

next.js에서는 `export const runtime = 'edge'`로 설정하면 자동으로 renderToReadableStream이 사용됩니다.

## 6.6 그 이후 이야기

캐싱, 에러바운더리, 로딩바운더리(Suspense), 스트리밍 등 모든 관리를 개발자가 도맡기 어려우니 next.js, remix가 해주는거 갖다 쓰라는 이야기를 주저리주러리.
