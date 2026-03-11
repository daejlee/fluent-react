# Week 7. 프레임워크

## 8.1 프레임워크가 필요한 이유

애플리케이션이 확장되면 라우팅, 데이터 페치, 서버 렌더링 같은 반복적인 문제를 계속 다뤄야 한다.

**프레임워크가 제공하는 것:**
- 미리 정의된 구조와 해결책
- 모범 사례 강제
- 반복 코드 제거

### 8.1.1 서버 렌더링

**서버 렌더링 구현:**

프레임워크는 일반적으로 서버 렌더링을 제공한다.

```tsx
// Express 서버로 SSR 구현
app.get("/", (req, res) => {
  res.end(createLayout(renderToString(<List />)));
});
```

프로덕션에서는 `renderToPipeableStream` 같은 비동기 API를 사용하는 것이 좋다.

### 8.1.2 라우팅

**파일 시스템 기반 라우팅:**

경로를 추가할 때마다 `req.get` 호출을 작성하는 방식은 확장성이 떨어진다. **파일 시스템 기반 라우팅**을 도입하면 `./pages` 디렉터리의 모든 파일이 자동으로 경로가 된다.

```
index.js
pages/
├── list.js    → /list
└── detail.js  → /detail
dist/
└── clientBundle.js
```

```tsx
// 서버에서 동적으로 페이지 컴포넌트 로드
app.get("/:route", async (req, res) => {
  const Page = (await import(`./pages/${req.params.route}`)).default;
  res.end(createLayout(renderToString(<Page {...req.query} />)));
});
```

페이지 컴포넌트는 **디폴트 export** 필수. 예측 가능한 방식으로 컴포넌트를 가져오기 위해 `.default`를 사용.

### 8.1.3 데이터 페치

**네트워크 폭포 문제**

클라이언트에서 데이터를 가져오면 JS 다운로드 → 컴포넌트 렌더링 → useEffect 실행 순서로 진행 -> 시간 지연

**해결 방법:**

페이지 렌더링 전에 서버에서 데이터를 미리 가져와 초기 props로 전달한다. 컴포넌트는 `initialThings` prop을 받아 즉시 렌더링하고, 이 값이 없을 때만 클라이언트에서 fetch.

```tsx
// 서버에서 데이터 미리 가져오기
export const getData = async () => {
  return {
    props: {
      initialThings: await fetch("...").then(r => r.json())
    }
  };
};
```

서버는 페이지 컴포넌트에서 `getData` 함수를 import하여 호출하고, 반환된 props를 컴포넌트에 전달한다.

## 8.2 프레임워크 사용 시 장점

**구조와 일관성**
- 코드베이스 구조 강제 → 신규 개발자 온보딩 용이

**모범 사례**
- 서버에서 데이터 미리 가져오기 → 성능 향상

**추상화**
- 라우팅, 데이터 페치 등 일반 작업 처리
- 예: Next.js의 `useRouter` 훅

**성능 최적화**
- 코드 분할, SSR, SSG 기본 제공
- Next.js는 링크 호버 시 자동 프리페치

**커뮤니티와 생태계**
- 풍부한 플러그인, 라이브러리
- 빠른 문제 해결

## 8.3 프레임워크 사용 시 트레이드오프

**학습 곡선**
- 고유한 개념, API, 규칙 학습 필요

**유연성과 규칙**
- 프레임워크 모델에 맞지 않는 요구사항 구현 시 제약

**의존과 서약**
- 프레임워크 운명에 애플리케이션 운명이 결부됨
- 유지보수 중단 시 마이그레이션 비용

**추상화 오버헤드**
- 내부 동작 이해 어려움 → 디버깅/성능 튜닝 복잡
- 예: Next.js `"use server"` 지시어의 내부 동작

## 8.4 인기 있는 리액트 프레임워크

### 8.4.1 Remix

**Remix란?**

Remix는 리액트와 웹 플랫폼의 기능을 활용하는 강력한 최신 웹 프레임워크다. 웹 표준을 적극적으로 활용하며, 서버 구성을 직접 수정할 수 있는 투명성을 제공한다.

**기본 구조**
- `app/entry.client.tsx` - 클라이언트 진입점
- `app/entry.server.tsx` - 서버 진입점  
- `app/root.tsx` - 공유 레이아웃

**서버 렌더링**

Remix는 `entry.server.tsx`에서 서버 렌더링을 자동 처리한다. 이 파일은 HTTP 응답을 생성하며, 특히 **봇과 브라우저 요청을 다르게 관리**한다.

```tsx
export default function handleRequest(request, ...) {
  return isbot(request.headers.get("user-agent"))
    ? handleBotRequest(...)
    : handleBrowserRequest(...);
}
```

**차이점:**
- **봇 요청**: SEO를 위해 렌더링된 HTML 콘텐츠 전체를 즉시 제공 (`onAllReady`)
- **브라우저 요청**: HTML + 하이드레이션용 자바스크립트 번들 제공 (`onShellReady`)

`renderToPipeableStream`으로 스트리밍 렌더링을 수행하며, 5초 타임아웃을 설정하여 긴 렌더링을 방지.

**라우팅**

Remix에서 각 경로는 `routes` 디렉터리의 파일로 표시된다. 파일 이름이 경로가 되며, 컴포넌트는 **디폴트 export**로 내보내야 한다.

```tsx
// app/routes/cheese.tsx
export default function CheesePage() {
  return <h1>치즈 이름</h1>;
}
```

**데이터 페치**

Remix는 **loader 함수**로 데이터를 가져온다. 비동기 `loader` 함수를 export하면, 반환된 값을 `useLoaderData` 훅으로 컴포넌트에서 사용할 수 있다.

```tsx
export async function loader() {
  const data = await fetch("...");
  return json(await data.json());
}

export default function CheesePage() {
  const cheeses = useLoaderData();
  return <ul>{cheeses.map(cheese => <li key={cheese.id}>{cheese.name}</li>)}</ul>;
}
```

서버가 페이지를 렌더링하기 전에 `loader`를 먼저 실행하여 데이터를 가져오므로, 네트워크 폭포 문제가 해결된다.

**데이터 조작**

Remix는 **웹 플랫폼의 기본 기능**을 적극 활용한다.

```tsx
export async function action({ request }) {
  const formData = await request.formData();
  await fetch("...", {
    method: "POST",
    body: JSON.stringify({ name: formData.get("cheese") })
  });
  return redirect("/cheese");
}

export default function CheesePage() {
  return (
    <form action="/cheese" method="post">
      <input type="text" name="cheese" />
      <button type="submit">추가</button>
    </form>
  );
}
```

- **action 함수**: 폼이 제출(POST)되면 자동 호출되어 서버에서 데이터 처리
- **브라우저가 폼 상태 관리**: `useState`, `onChange` 불필요
- **프로그레시브 인핸스먼트**: 자바스크립트 없이도 동작 (접근성 향상)
- **리디렉션**: `redirect("/cheese")`로 GET 요청 후 `loader` 재실행하여 업데이트된 데이터 표시

### 8.4.2 Next.js

**Next.js란?**

Vercel에서 개발한 인기 있는 리액트 프레임워크로, SSR과 정적 사이트 생성을 간편하게 할 수 있다. **설정보다 규칙(Convention over Configuration)** 원칙을 따라 반복 코드를 최소화한다.

Next.js 13부터 **앱 라우터(App Router)**가 도입되었으며, Remix와 달리 **서버 구성을 숨기고** 개발자가 기능 구현에 집중할 수 있도록 복잡성을 추상화한다.

**기본 구조**

```
app/
├── page.tsx      - 페이지
├── layout.tsx    - 레이아웃
├── error.tsx     - 에러 처리
└── loading.tsx   - 로딩 UI
```

**서버 렌더링**

Next.js는 **서버 우선(Server-First)** 접근 방식을 사용. 모든 페이지와 컴포넌트는 기본적으로 **서버 컴포넌트**이며, 빌드 시 최대한 정적 콘텐츠로 렌더링한다.

클라이언트에서 상태나 이벤트가 필요한 경우에만 `'use client'` 지시어로 **클라이언트 컴포넌트**임을 명시.

```tsx
// 클라이언트 컴포넌트만 명시적 표시
'use client';
export default function Interactive() {
  const [state, setState] = useState(0);
  return <button onClick={() => setState(state + 1)}>{state}</button>;
}
```

**라우팅**

Next.js의 앱 라우터에서는 **디렉터리가 경로**가 되고, `page.tsx` 파일이 실제 렌더링되는 페이지.

```
app/
└── cheese/
    └── page.tsx  → /cheese
```

```tsx
export default function CheesePage() {
  return <h1>치즈 목록</h1>;
}
```

**데이터 페치**

Next.js는 서버 컴포넌트에서 **비동기 함수**로 직접 `await`를 사용하여 데이터를 가져올 수 있다.

```tsx
export default async function CheesePage() {
  const cheeses = await fetch("...").then(r => r.json());
  return <ul>{cheeses.map(cheese => <li key={cheese.id}>{cheese.name}</li>)}</ul>;
}
```

**컴포넌트 단위 데이터 페치:**

페이지 레벨뿐 아니라 개별 컴포넌트에서도 데이터를 가져올 수 있다.

```tsx
export async function CheeseList() {
  const cheeses = await fetch("...").then(r => r.json());
  return <ul>{cheeses.map(cheese => <li key={cheese.id}>{cheese.name}</li>)}</ul>;
}
```

이를 통해 데이터 페치 로직을 컴포넌트 단위로 세분화할 수 있다.

**데이터 조작**

Next.js는 **서버 액션(Server Actions)**으로 데이터를 조작한다. `"use server"` 지시어를 사용하여 함수가 서버에서만 실행되도록 한다.

```tsx
export default function CheesePage() {
  async function addCheese(formData) {
    "use server"; // 서버에서만 실행
    await fetch("...", {
      method: "POST",
      body: JSON.stringify({ name: formData.get("cheese") })
    });
    revalidatePath("/cheese"); // 캐시 갱신
    return redirect("/cheese");
  }

  return (
    <form action={addCheese} method="post">
      <input type="text" name="cheese" />
      <button type="submit">추가</button>
    </form>
  );
}
```

- **"use server"**: 함수가 서버에서만 실행되도록 보장 (클라이언트 번들에 포함 안 됨)
- **revalidatePath**: 특정 경로의 캐시를 무효화하여 최신 데이터 표시
- **별도 모듈 분리 가능**: 서버 액션을 별도 파일로 작성하여 재사용 가능

내부적으로 Next.js가 자동으로 API 엔드포인트를 생성한다.

## 8.5 Remix vs Next.js 비교

### 차이

| 항목 | Remix | Next.js |
|------|-------|---------|
| **접근 방식** | 웹 표준 우선 | 서버 우선 + 정적 우선 |
| **투명성** | 서버 구성 노출 (수정 가능) | 복잡성 숨김 (추상화 우선) |
| **폼 처리** | 표준 HTML 폼 | 서버 액션 |

### 공통점

- 파일 시스템 기반 라우팅
- 서버 렌더링 기본 제공
- 공유 레이아웃 개념
- 데이터 페치 함수 (loader vs async 컴포넌트)

### 차이점

**Remix**
- `loader`/`action` 함수로 데이터 처리 분리
- JS 없이도 동작 (프로그레시브 인핸스먼트)
- 서버 구성 수정 가능

**Next.js**
- 컴포넌트 단위 데이터 페치
- 서버/클라이언트 컴포넌트 명확한 구분
- 정적 생성 + 서버 렌더링 하이브리드

## 8.6 프레임워크 선택 가이드

**Remix를 선택해야 할 때:**
- 웹 표준 중시
- 프로그레시브 인핸스먼트 필요
- 서버 동작 커스터마이징 필요

**Next.js를 선택해야 할 때:**
- 정적 생성 + SSR 혼용
- 컴포넌트 단위 세밀한 제어
- Vercel 배포 활용