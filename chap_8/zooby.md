# 8.1 프레임워크가 필요한 이유

React는 UI 라이브러리일 뿐이다. 실제 애플리케이션을 만들려면 서버 렌더링, 라우팅, 데이터 페칭 같은 문제를 직접 해결해야 한다. 이 과정에서 마주치는 복잡성이 프레임워크가 필요한 이유다.

## 서버 렌더링

클라이언트 사이드 렌더링(CSR)만으로는 한계가 있다.

```
[CSR의 문제]
브라우저 요청 → 빈 HTML 반환 → JS 다운로드 → JS 실행 → API 호출 → 렌더링
                    ↑
              사용자는 빈 화면을 봄 (First Contentful Paint 지연)
              검색 엔진은 빈 HTML을 봄 (SEO 문제)
```

서버에서 HTML을 미리 렌더링하면 이 문제를 해결할 수 있다. Express로 직접 구현해보자.

```js
// server.js
import express from "express";
import React from "react";
import { renderToString } from "react-dom/server";
import { ItemList, ItemDetail } from "./components";

const app = express();

// 목록 페이지
app.get("/items", async (req, res) => {
  const items = await fetchItems(); // DB에서 아이템 목록 조회

  const html = renderToString(<ItemList items={items} />);

  res.send(`
    <!DOCTYPE html>
    <html>
      <body>
        <div id="root">${html}</div>
        <script>
          window.__INITIAL_DATA__ = ${JSON.stringify(items)};
        </script>
        <script src="/bundle.js"></script>
      </body>
    </html>
  `);
});

// 상세 페이지
app.get("/items/:id", async (req, res) => {
  const item = await fetchItem(req.params.id); // DB에서 단일 아이템 조회

  const html = renderToString(<ItemDetail item={item} />);

  res.send(`
    <!DOCTYPE html>
    <html>
      <body>
        <div id="root">${html}</div>
        <script>
          window.__INITIAL_DATA__ = ${JSON.stringify(item)};
        </script>
        <script src="/bundle.js"></script>
      </body>
    </html>
  `);
});

app.listen(3000);
```

```
[SSR 흐름]
브라우저 요청 → 서버에서 데이터 조회 → HTML 렌더링 → 완성된 HTML 반환
                                                          ↓
                                              사용자는 즉시 콘텐츠를 봄
                                              검색 엔진도 콘텐츠를 읽음
```

하지만 이것만으로는 부족하다. 클라이언트에서 React가 다시 작동하려면 **하이드레이션**이 필요하다.

```js
// client.js
import { hydrateRoot } from "react-dom/client";
import { ItemList, ItemDetail } from "./components";

// 서버에서 주입한 초기 데이터 사용
const initialData = window.__INITIAL_DATA__;

// 현재 URL에 따라 적절한 컴포넌트 하이드레이션
if (window.location.pathname === "/items") {
  hydrateRoot(
    document.getElementById("root"),
    <ItemList items={initialData} />,
  );
} else if (window.location.pathname.startsWith("/items/")) {
  hydrateRoot(
    document.getElementById("root"),
    <ItemDetail item={initialData} />,
  );
}
```

벌써 문제가 보인다. URL에 따라 어떤 컴포넌트를 렌더링할지 서버와 클라이언트 양쪽에서 판단해야 한다.

## 라우팅

URL과 컴포넌트를 매핑하는 로직이 서버와 클라이언트에 중복된다.

```js
// server.js - 서버 라우팅
app.get("/items", ...) // ItemList 렌더링
app.get("/items/:id", ...) // ItemDetail 렌더링

// client.js - 클라이언트 라우팅
if (pathname === "/items") // ItemList 하이드레이션
if (pathname.startsWith("/items/")) // ItemDetail 하이드레이션
```

이 중복을 해결하려면 **동형(Isomorphic) 라우터**가 필요하다.

### 동형 라우터 (Isomorphic Router)

동형 라우터는 서버와 클라이언트에서 동일한 라우팅 로직을 공유하는 라우터다. "동형"은 같은 코드가 서버와 브라우저 양쪽에서 실행될 수 있다는 의미다.

```js
// routes.js - 라우트 정의 (서버/클라이언트 공유)
export const routes = [
  {
    path: "/items",
    component: ItemList,
    fetchData: () => fetchItems(),
  },
  {
    path: "/items/:id",
    component: ItemDetail,
    fetchData: (params) => fetchItem(params.id),
  },
];
```

```js
// 라우트 매칭 함수 (서버/클라이언트 공유)
function matchRoute(pathname, routes) {
  for (const route of routes) {
    const match = matchPath(pathname, route.path);
    if (match) {
      return { route, params: match.params };
    }
  }
  return null;
}
```

서버에서 사용:

```js
// server.js
import { routes, matchRoute } from "./routes";

app.get("*", async (req, res) => {
  const { route, params } = matchRoute(req.path, routes);

  // 라우트 정의에서 데이터 페칭 함수 실행
  const data = await route.fetchData(params);
  const Component = route.component;

  const html = renderToString(<Component {...data} />);

  res.send(`
    <!DOCTYPE html>
    <html>
      <body>
        <div id="root">${html}</div>
        <script>
          window.__INITIAL_DATA__ = ${JSON.stringify(data)};
          window.__INITIAL_PATH__ = "${req.path}";
        </script>
        <script src="/bundle.js"></script>
      </body>
    </html>
  `);
});
```

클라이언트에서 사용:

```js
// client.js
import { routes, matchRoute } from "./routes";

const { route, params } = matchRoute(window.__INITIAL_PATH__, routes);
const Component = route.component;

hydrateRoot(
  document.getElementById("root"),
  <Component {...window.__INITIAL_DATA__} />,
);

// 클라이언트 네비게이션
window.addEventListener("popstate", async () => {
  const { route, params } = matchRoute(window.location.pathname, routes);
  const data = await route.fetchData(params);

  // 리렌더링...
});
```

```
[동형 라우터의 핵심]

                routes.js (공유)
                     │
         ┌───────────┴───────────┐
         ▼                       ▼
    server.js                client.js
    - matchRoute()           - matchRoute()
    - renderToString()       - hydrateRoot()
    - fetchData()            - fetchData() (네비게이션 시)
```

동일한 라우트 정의를 양쪽에서 사용하므로 URL ↔ 컴포넌트 매핑이 항상 일치한다.

## 데이터 페칭

서버 렌더링에서 데이터 페칭은 특히 까다롭다.

```
[문제: 데이터가 필요한 시점]

서버: 렌더링 전에 모든 데이터가 준비되어야 함
클라이언트: 컴포넌트가 마운트된 후 데이터를 가져와도 됨
```

컴포넌트 내부에서 데이터를 가져오는 일반적인 패턴은 SSR에서 작동하지 않는다:

```jsx
// ❌ SSR에서 작동하지 않음
function ItemList() {
  const [items, setItems] = useState([]);

  useEffect(() => {
    // useEffect는 서버에서 실행되지 않음!
    fetchItems().then(setItems);
  }, []);

  return (
    <ul>
      {items.map((item) => (
        <li key={item.id}>{item.name}</li>
      ))}
    </ul>
  );
}
```

그래서 데이터 페칭 로직을 컴포넌트 외부로 빼야 한다:

```js
// 라우트 레벨에서 데이터 요구사항 정의
const routes = [
  {
    path: "/items",
    component: ItemList,
    fetchData: () => fetchItems(), // 컴포넌트 외부에서 정의
  },
];

// 서버에서: 렌더링 전에 fetchData 호출
// 클라이언트에서: 네비게이션 시 fetchData 호출
```

하지만 중첩된 라우트가 있다면?

```
/users/:userId/posts/:postId

UserLayout (userId 필요)
  └─ PostDetail (postId 필요, userId도 필요할 수 있음)
```

```js
// 중첩 라우트의 데이터 페칭
const routes = [
  {
    path: "/users/:userId",
    component: UserLayout,
    fetchData: (params) => fetchUser(params.userId),
    children: [
      {
        path: "posts/:postId",
        component: PostDetail,
        fetchData: (params) => fetchPost(params.postId),
      },
    ],
  },
];

// 서버에서 모든 중첩 라우트의 데이터를 병렬로 가져오기
async function fetchAllData(matchedRoutes, params) {
  const dataPromises = matchedRoutes.map((route) => route.fetchData(params));
  return Promise.all(dataPromises);
}
```

여기까지만 해도 상당히 복잡하다. 그리고 아직 고려하지 않은 것들이 많다:

- 에러 처리 (404, 500)
- 리다이렉트
- 코드 스플리팅
- 캐싱
- 스트리밍 SSR
- 정적 생성 (SSG)

이 모든 것을 직접 구현하면 결국 프레임워크를 만들게 된다.

# 8.2 프레임워크의 장점과 트레이드오프

## 장점

### 구조와 일관성

프레임워크는 프로젝트 구조에 대한 명확한 규칙을 제공한다.

```
[프레임워크 없이]
팀원 A: "컴포넌트는 src/components에 둘까요?"
팀원 B: "저는 features 폴더별로 나누는 게 좋은데..."
팀원 C: "API 호출은 어디서 하죠? 컴포넌트 안에서? 별도 파일?"
→ 매번 논의와 합의가 필요

[프레임워크 사용]
app/
├── routes/          → 라우트는 여기
├── components/      → 공유 컴포넌트는 여기
└── models/          → 데이터 모델은 여기
→ 프레임워크가 정한 규칙을 따르면 됨
```

새 팀원이 합류해도 구조가 익숙하다. 다른 회사의 Next.js 프로젝트도 비슷한 구조를 가진다.

### 모범 사례

프레임워크는 수년간의 커뮤니티 경험이 녹아든 모범 사례를 기본으로 제공한다.

- **라우팅**: 코드 스플리팅이 적용된 파일 기반 라우팅
- **데이터 페칭**: 워터폴을 피하는 병렬 로딩 패턴
- **캐싱**: 적절한 캐시 전략과 재검증 로직
- **에러 처리**: 라우트별 에러 바운더리

### 추상화

복잡한 구현 세부사항을 숨기고 간단한 API를 제공한다.  
서버 렌더링, 하이드레이션, 스트리밍 같은 복잡한 과정이 추상화 뒤에 숨겨져 있다.

### 성능 최적화

프레임워크는 성능 최적화를 자동으로 처리한다.

```
[자동 적용되는 최적화들]

번들링:
├── 라우트별 코드 스플리팅
├── 트리 쉐이킹
└── 청크 최적화

로딩:
├── 프리페칭 (링크 호버 시)
├── 스트리밍 SSR
└── 선택적 하이드레이션

에셋:
├── 이미지 자동 최적화 (리사이징, WebP 변환)
├── 폰트 서브셋팅
└── CSS 최적화
```

이런 최적화를 직접 구현하려면 상당한 시간과 전문 지식이 필요하다.

### 커뮤니티와 생태계

널리 사용되는 프레임워크는 풍부한 생태계를 가진다.

- **문서와 튜토리얼**: 공식 문서, 강의, 블로그 글
- **서드파티 라이브러리**: 프레임워크에 최적화된 패키지들
- **호스팅 플랫폼**: Vercel, Netlify 등의 통합 지원
- **채용 시장**: 해당 프레임워크 경험자를 찾기 쉬움

```
[Next.js 생태계 예시]

인증: NextAuth.js
CMS: Contentful, Sanity 공식 통합
배포: Vercel 원클릭 배포
분석: Vercel Analytics
```

## 트레이드오프

### 학습 곡선

React만 알아서는 부족하다. 프레임워크 고유의 개념을 추가로 학습해야 한다.

```
[학습해야 할 것들]

Next.js:
├── App Router vs Pages Router
├── 서버 컴포넌트 vs 클라이언트 컴포넌트
├── 렌더링 전략 (SSG, SSR, ISR)
├── Server Actions
├── 미들웨어
└── 캐싱 전략

Remix:
├── loader / action 패턴
├── 중첩 라우트
├── Form vs fetcher
├── defer / Await
└── 에러 바운더리
```

프레임워크가 업데이트될 때마다 새로운 개념이 추가되기도 한다.

### 유연성과 규칙

프레임워크의 "올바른 방법"을 따라야 한다. 규칙을 벗어나면 문제가 생긴다.  
프레임워크가 정한 경계를 이해하고 그 안에서 작업해야 한다.

### 의존과 서약

프레임워크에 의존하면 그 프레임워크의 방향을 따라가야 한다.

```
[의존성 문제]

1. 버전 업그레이드 강제
   Next.js 12 → 13 (App Router)
   → 완전히 다른 패러다임, 대규모 마이그레이션 필요

2. 지원 중단 위험
   Gatsby의 인기 하락
   → 생태계 축소, 플러그인 관리 안 됨

3. React 버전 종속
   React 19 사용하고 싶어도
   → 프레임워크가 지원할 때까지 대기
```

프레임워크 선택은 장기적인 서약이다. 나중에 바꾸려면 비용이 크다.

### 추상화 오버헤드

추상화는 양날의 검이다.

```
[추상화가 문제가 될 때]

1. 디버깅 어려움
   에러: "Hydration mismatch"
   → 프레임워크 내부에서 발생, 원인 파악 어려움

2. 성능 병목
   "왜 이렇게 느리지?"
   → 프레임워크 내부 동작을 몰라서 최적화 못 함

3. 제한된 커스터마이징
   "이 동작만 바꾸고 싶은데..."
   → 프레임워크가 노출하지 않은 부분은 수정 불가
```

```
[추상화 레이어]

내 코드
   ↓
프레임워크 (Next.js, Remix)
   ↓
React
   ↓
브라우저 API

문제가 어느 레이어에서 발생했는지 파악해야 함
→ 모든 레이어를 이해해야 효과적인 디버깅 가능
```

결국 프레임워크 내부를 들여다봐야 할 때가 온다. 추상화가 복잡성을 없애는 게 아니라 숨길 뿐이라는 것을 기억해야 한다.

# 8.3 리믹스 (Remix)

리믹스는 웹 표준에 충실한 풀스택 프레임워크다. "웹의 기본으로 돌아가자"는 철학을 가진다.

## 핵심 철학: 웹 표준

리믹스는 브라우저가 이미 잘 하는 것을 다시 구현하지 않는다.

```jsx
// 폼 제출 - 브라우저 기본 동작 활용
function NewPost() {
  return (
    <Form method="post">
      <input name="title" />
      <textarea name="content" />
      <button type="submit">작성</button>
    </Form>
  );
}

// JS가 로드되기 전에도 폼이 동작함 (Progressive Enhancement)
// JS가 로드되면 클라이언트 사이드로 향상됨
```

## loader와 action

리믹스의 데이터 흐름은 명확하다: `loader`로 읽고, `action`으로 쓴다.

```jsx
// app/routes/items.$id.tsx

// GET 요청 - 데이터 로딩
export async function loader({ params }) {
  const item = await db.item.findUnique({
    where: { id: params.id },
  });

  if (!item) {
    throw new Response("Not Found", { status: 404 });
  }

  return json({ item });
}

// POST/PUT/DELETE 요청 - 데이터 변경
export async function action({ request, params }) {
  const formData = await request.formData();

  if (request.method === "DELETE") {
    await db.item.delete({ where: { id: params.id } });
    return redirect("/items");
  }

  await db.item.update({
    where: { id: params.id },
    data: { title: formData.get("title") },
  });

  return json({ success: true });
}

// 컴포넌트
export default function ItemDetail() {
  const { item } = useLoaderData();

  return (
    <div>
      <h1>{item.title}</h1>
      <Form method="delete">
        <button>삭제</button>
      </Form>
    </div>
  );
}
```

```
[리믹스 데이터 흐름]

GET /items/123
    ↓
loader 실행 → 데이터 반환 → 컴포넌트 렌더링

POST /items/123
    ↓
action 실행 → 데이터 변경 → loader 재실행 → UI 자동 갱신
```

## 중첩 라우트와 병렬 데이터 로딩

리믹스의 강점은 중첩 라우트에서 드러난다.

```
app/routes/
├── _layout.tsx           → 전체 레이아웃
├── dashboard.tsx         → /dashboard 레이아웃
├── dashboard._index.tsx  → /dashboard
└── dashboard.stats.tsx   → /dashboard/stats
```

```jsx
// dashboard.tsx (부모 레이아웃)
export async function loader() {
  return json({ user: await getUser() });
}

export default function DashboardLayout() {
  const { user } = useLoaderData();
  return (
    <div>
      <Sidebar user={user} />
      <Outlet /> {/* 자식 라우트가 여기에 렌더링 */}
    </div>
  );
}

// dashboard.stats.tsx (자식)
export async function loader() {
  return json({ stats: await getStats() });
}

export default function Stats() {
  const { stats } = useLoaderData();
  return <StatsChart data={stats} />;
}
```

```
[/dashboard/stats 요청 시]

┌─────────────────────────────────────┐
│         병렬로 데이터 로딩            │
│  getUser() ──┬── getStats()         │
│              ↓                       │
│     두 loader가 동시에 실행됨         │
└─────────────────────────────────────┘
                ↓
┌─────────────────────────────────────┐
│     DashboardLayout                  │
│  ┌──────────────────────────────┐   │
│  │  Sidebar (user 데이터)        │   │
│  ├──────────────────────────────┤   │
│  │  Stats (stats 데이터)         │   │
│  └──────────────────────────────┘   │
└─────────────────────────────────────┘
```

Waterfall이 없다. 부모와 자식의 데이터가 병렬로 로딩된다.

## 에러 바운더리

각 라우트 레벨에서 에러를 처리할 수 있다.

```jsx
// dashboard.stats.tsx
export function ErrorBoundary() {
  const error = useRouteError();

  return (
    <div className="error">
      <h2>통계를 불러올 수 없습니다</h2>
      <p>{error.message}</p>
    </div>
  );
}
```

```
[stats 라우트에서 에러 발생 시]

┌─────────────────────────────────────┐
│     DashboardLayout (정상 동작)      │
│  ┌──────────────────────────────┐   │
│  │  Sidebar (정상 표시)          │   │
│  ├──────────────────────────────┤   │
│  │  ErrorBoundary (에러 UI)      │   │  ← 이 부분만 에러 처리
│  └──────────────────────────────┘   │
└─────────────────────────────────────┘

전체 페이지가 깨지지 않고, 해당 영역만 에러 UI로 대체됨
```

## 리믹스의 특징 요약

| 특징                    | 설명                                            |
| ----------------------- | ----------------------------------------------- |
| 웹 표준 중심            | Request/Response, FormData, Web Fetch API 활용  |
| Progressive Enhancement | JS 없이도 기본 동작, JS로 향상                  |
| 중첩 라우트             | 레이아웃 단위로 데이터/에러/로딩 관리           |
| 병렬 데이터 로딩        | Waterfall 없는 데이터 페칭                      |
| 낙관적 UI               | `useNavigation`, `useFetcher`로 즉각적인 피드백 |

# 8.4 넥스트 (Next.js)

넥스트는 가장 널리 사용되는 React 프레임워크다. React 팀과 긴밀하게 협력하며, React의 새 기능을 가장 먼저 도입한다.

## App Router: 서버 컴포넌트 중심

Next.js 13부터 도입된 App Router는 React Server Components를 기본으로 사용한다.

```jsx
// app/items/page.tsx
// 기본적으로 서버 컴포넌트 - async 함수 가능

async function ItemsPage() {
  // 서버에서 직접 DB 접근 (API 라우트 불필요)
  const items = await prisma.item.findMany();

  return (
    <ul>
      {items.map((item) => (
        <li key={item.id}>{item.title}</li>
      ))}
    </ul>
  );
}

export default ItemsPage;
```

```
[서버 컴포넌트의 이점]

1. 번들 크기 감소
   - 서버에서만 실행되는 코드는 클라이언트로 전송되지 않음
   - DB 클라이언트, 마크다운 파서 등 무거운 라이브러리 사용 가능

2. 직접 데이터 접근
   - API 라우트를 거치지 않고 DB 직접 쿼리
   - 서버 간 통신 레이턴시 제거

3. 보안
   - API 키, DB 연결 정보가 클라이언트에 노출되지 않음
```

## 클라이언트 컴포넌트

상호작용이 필요하면 `"use client"` 지시어를 사용한다.

```jsx
"use client";

import { useState } from "react";

function Counter() {
  const [count, setCount] = useState(0);

  return <button onClick={() => setCount(count + 1)}>클릭: {count}</button>;
}
```

```
[서버/클라이언트 컴포넌트 구성]

                서버
┌─────────────────────────────────────┐
│  Page (서버 컴포넌트)                 │
│  ├── Header (서버)                   │
│  ├── ItemList (서버)                 │
│  │     └── data fetching 가능        │
│  └── Counter (클라이언트)            │
│        └── "use client" 경계         │
└─────────────────────────────────────┘
                 ↓ HTML + RSC Payload
               브라우저
┌─────────────────────────────────────┐
│  Counter만 하이드레이션됨             │
│  (나머지는 정적 HTML)                 │
└─────────────────────────────────────┘
```

## 렌더링 전략

Next.js는 다양한 렌더링 전략을 지원한다.

### 정적 생성 (Static Generation)

```jsx
// 빌드 시점에 HTML 생성
async function BlogPost({ params }) {
  const post = await getPost(params.slug);
  return <Article content={post.content} />;
}

// 어떤 경로를 미리 생성할지 지정
export async function generateStaticParams() {
  const posts = await getAllPosts();
  return posts.map((post) => ({ slug: post.slug }));
}
```

### 동적 렌더링

```jsx
// 매 요청마다 서버에서 렌더링
import { cookies } from "next/headers";

async function Dashboard() {
  const session = cookies().get("session");
  const user = await getUser(session);

  return <DashboardContent user={user} />;
}
```

### 증분 정적 재생성 (ISR)

```jsx
// 정적 생성 + 주기적 재생성
async function ProductPage({ params }) {
  const product = await getProduct(params.id);
  return <ProductDetail product={product} />;
}

// 60초마다 백그라운드에서 재생성
export const revalidate = 60;
```

```
[렌더링 전략 비교]

정적 생성 (SSG):
빌드 시 → HTML 생성 → CDN 캐시 → 모든 요청에 동일한 HTML
장점: 가장 빠름
단점: 실시간 데이터 반영 불가

동적 렌더링 (SSR):
요청 시 → 데이터 조회 → HTML 생성 → 응답
장점: 항상 최신 데이터
단점: 서버 부하, 느린 TTFB

증분 정적 재생성 (ISR):
빌드 시 → HTML 생성 → 캐시 → (revalidate 시간 후) → 백그라운드 재생성
장점: 정적의 성능 + 주기적 갱신
단점: 실시간은 아님
```

## Server Actions

서버에서 실행되는 함수를 클라이언트에서 직접 호출할 수 있다.

```jsx
// app/actions.ts
"use server";

export async function createItem(formData: FormData) {
  const title = formData.get("title");

  await prisma.item.create({
    data: { title },
  });

  revalidatePath("/items");
}

// app/items/new/page.tsx
import { createItem } from "../actions";

function NewItemPage() {
  return (
    <form action={createItem}>
      <input name="title" />
      <button type="submit">생성</button>
    </form>
  );
}
```

```
[Server Action 흐름]

클라이언트에서 폼 제출
    ↓
Next.js가 자동으로 서버에 POST 요청
    ↓
"use server" 함수 실행 (서버에서)
    ↓
revalidatePath로 캐시 무효화
    ↓
UI 자동 갱신
```

## 넥스트의 특징 요약

| 특징               | 설명                                     |
| ------------------ | ---------------------------------------- |
| 서버 컴포넌트 기본 | 클라이언트 번들 최소화, 직접 데이터 접근 |
| 다양한 렌더링      | SSG, SSR, ISR을 페이지별로 선택          |
| Server Actions     | API 라우트 없이 서버 함수 직접 호출      |
| 이미지/폰트 최적화 | 자동 최적화 컴포넌트 제공                |
| 미들웨어           | 엣지에서 요청 전처리                     |
| Vercel 통합        | 배포 플랫폼과 긴밀한 최적화              |
