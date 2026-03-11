# 8. 프레임워크

## 서버사이드렌더링 원리

```js
import { join } from "path";
import express from "express";
import React from "react";

const app = express();

app.get("/", (req, res) => {
  const html = ReactDOMServer.renderToString(<App />);
  res.send(html);
});

app.get("/:route", async (req, res) => {
  const exportedStuff = await import(
    join(process.cwd(), "pages", req.params.route)
  );
  const Page = exportedStuff.default;

  const data = exportedStuff.getData ? await exportedStuff.getData() : null;
  const props = req.query || {};

  res.setHeader("Content-Type", "text/html");
  res.end(createLayout(renderToString(<Page {...props} data={data} />)));
});

app.listen(3000, () => {
  console.log("Server is running on port 3000");
});
```

이런식으로 express, React를 활용해 서버사이드에서 `getData`를 페칭하고 `Page`컴포넌트를 렌더링하는 방식으로 SSR을 구현할 수 있습니다. 하지만, 이런 방식은 라우팅, 데이터 페칭, 코드 스플리팅 등 다양한 기능을 직접 구현해야 하기 때문에 복잡하고 유지보수가 어려울 수 있습니다. 그래서 Next.js와 같은 프레임워크가 등장하여 이러한 기능들을 쉽게 사용할 수 있도록 도와줍니다.

코드보면 알겠지만 exportedStuff (export 된거)에는 getData함수와 default로 export된 Page에 해당되는 컴포넌트가 있습니다.

## 프레임워크종류

### Remix

isBot이라는 라이브러리를 사용하여 봇인지 아닌지를 판단하여 SSR과 CSR을 선택적으로 사용할 수 있도록 지원합니다. 또한, 라우팅과 데이터 페칭을 통합하여 개발자가 더 쉽게 페이지를 구성할 수 있도록 도와줍니다.

RemixServer는 SSR과 CSR을 모두 지원하는 프레임워크로, 서버에서 렌더링된 HTML을 클라이언트로 전송하여 초기 로딩 속도를 개선하고 SEO를 향상시킵니다. 또한, Remix는 라우팅과 데이터 페칭을 통합하여 개발자가 더 쉽게 페이지를 구성할 수 있도록 도와줍니다.

```js
// entry.server.jsx
import { PassThrough } from "node:stream";
import type { AppLoadContext, EntryContext } from '@remix-run/node';
import { createReadableStreamFromReadable } from "@remix-run/node";
import { RemixServer } from "@remix-run/react";
import isbot from "isbot";
import { renderToPipeableStream } from "react-dom/server";

export default function handleRequest(
  request: Request,
  responseStatusCode: number,
  responseHeaders: Headers,
  remixContext: EntryContext
) {
  const callbackName = isbot(request.headers.get("user-agent"))
    ? "onAllReady"
    : "onShellReady";

  return new Promise((resolve, reject) => {
    let didError = false;

    const { pipe } = renderToPipeableStream(
      <RemixServer context={remixContext} url={request.url} abortDelay={5000} />,
      {
        [callbackName]: () => {
          const body = new PassThrough();

          responseHeaders.set("Content-Type", "text/html");

          resolve(
            new Response(createReadableStreamFromReadable(body), {
              headers: responseHeaders,
              status: didError ? 500 : responseStatusCode,
            })
          );

          pipe(body);
        },
        onShellError: (error) => {
          reject(error);
        },
        onError: (error) => {
          didError = true;
          console.error(error);
        },
      }
    );
  });
}
```

RemixServer의 3가지 props 설명은 다음과 같습니다.

- context: Remix의 라우팅과 데이터 페칭에 필요한 정보를 담고 있는 객체입니다. 이 객체는 라우트 매칭, 데이터 로딩, 에러 처리 등에 사용됩니다.
- url: 클라이언트에서 요청된 URL을 나타내는 문자열입니다. 이 URL을 기반으로 Remix는 적절한 라우트를 매칭하고 해당 라우트에 필요한 데이터를 로드합니다.
- abortDelay: 서버 렌더링이 일정 시간 내에 완료되지 않을 경우 렌더링을 중단하는 데 사용되는 시간(밀리초 단위)입니다.

```jsx
// root.jsx
import {
  Links,
  Meta,
  Outlet,
  Scripts,
  ScrollRestoration,
} from "@remix-run/react";

// 1. 공통 스타일시트 연결 (선택)
import "./tailwind.css";

export default function App() {
  return (
    <html lang="ko">
      <head>
        <meta charSet="utf-8" />
        <meta name="viewport" content="width=device-width, initial-scale=1" />
        {/* 2. 각 페이지의 메타 태그 주입 */}
        <Meta />
        {/* 3. 각 페이지의 링크 태그(CSS 등) 주입 */}
        <Links />
      </head>
      <body>
        {/* 4. 자식 라우트들이 렌더링되는 지점 */}
        <Outlet />

        {/* 5. 페이지 이동 시 스크롤 위치 복구 */}
        <ScrollRestoration />
        {/* 6. 클라이언트 사이드 JS 실행 (Hydration) */}
        <Scripts />
      </body>
    </html>
  );
}

// 7. 에러 발생 시 보여줄 화면 (선택)
export function ErrorBoundary() {
  return (
    <html>
      <head>
        <title>오류 발생!</title>
        <Meta />
        <Links />
      </head>
      <body>
        <h1>문제가 발생했습니다.</h1>
        <Scripts />
      </body>
    </html>
  );
}
```

root.jsx에서 Meta, Links, Outlet, ScrollRestoration, Scripts 컴포넌트는 각각 다음과 같은 역할을 합니다.

- Meta: 각 페이지에서 정의한 메타 태그를 HTML head에 주입하는 역할을 합니다. 이를 통해 SEO 최적화와 소셜 미디어 공유 시 적절한 정보가 제공될 수 있습니다.
- Links: 각 페이지에서 정의한 링크 태그(예: CSS 파일)를 HTML head에 주입하는 역할을 합니다. 이를 통해 페이지별로 필요한 스타일시트나 기타 리소스를 로드할 수 있습니다.
- Outlet: 자식 라우트들이 렌더링되는 지점입니다. 라우팅에 따라 해당 위치에 적절한 컴포넌트가 렌더링됩니다.
- ScrollRestoration: 페이지 이동 시 스크롤 위치를 복구하는 역할을 합니다.
- Scripts: 클라이언트 사이드 JavaScript를 실행하는 역할을 합니다. 이를 통해 서버에서 렌더링된 HTML이 클라이언트에서 인터랙티브하게 작동할 수 있도록 합니다.

`routes/` 디렉토리에 파일을 생성하면 해당 파일이 자동으로 라우트로 인식됩니다. 예를 들어, `app/routes/index.jsx` 파일은 루트 경로(`/`)에 매핑되고, `app/routes/about.jsx` 파일은 `/about` 경로에 매핑됩니다.

#### 라우팅

Remix는 파일 기반 라우팅을 지원합니다. `/routes` 디렉토리에 파일을 생성하면 해당 파일이 자동으로 라우트로 인식됩니다. 예를 들어, `/routes/index.jsx` 파일은 루트 경로(`/`)에 매핑되고, `/routes/about.jsx` 파일은 `/about` 경로에 매핑됩니다.

해당하는 라우터파일들이 root.jsx의 Outlet 컴포넌트에 렌더링됩니다.

```jsx
// ./routes/counter.jsx
import { useState } from "react";

export default function Counter() {
  const [count, setCount] = useState(0);
  return (
    <div>
      <h1>Counter: {count}</h1>
      <button onClick={() => setCount(count + 1)}>Increment</button>
    </div>
  );
}
```

#### 데이터페칭

```jsx
import { json } from "@remix-run/node";
import { useLoaderData } from "@remix-run/react";
// 로더
export async function loader() {
  const data = await fetch("https://api.com/get-cheeses");
  return json(await data.json());
}
export default function CheesePage() {
  const cheeses = useLoaderData();
  return (
    <div>
      <h1>치즈 이름</h1>
      <ul>
        {cheeses.map((cheese) => (
          <li key={cheese.id}>{cheese.name}</li>
        ))}
      </ul>
    </div>
  );
}
```

loader함수는 서버에서 실행되며, 데이터를 페칭하여 JSON 형태로 반환합니다. 이 데이터는 클라이언트에서 useLoaderData 훅을 통해 접근할 수 있습니다. 이렇게 하면 서버에서 데이터를 미리 로드하여 페이지를 렌더링할 때 필요한 데이터를 제공할 수 있습니다.

#### 데이터조작(액션)

```jsx
import { json } from "@remix-run/node";
import { useLoaderData, Form } from "@remix-run/react";
// 로더
export async function loader() {
  const data = await fetch("https://api.com/get-cheeses");
  return json(await data.json());
}
// 액션
export async function action({ request }) {
  const formData = await request.formData();
  const newCheese = formData.get("cheese");
  await fetch("https://api.com/add-cheese", {
    method: "POST",
    body: JSON.stringify({ name: newCheese }),
    headers: { "Content-Type": "application/json" },
  });
  return null;
}
export default function CheesePage() {
  const cheeses = useLoaderData();
  return (
    <div>
      <h1>치즈 이름</h1>
      <ul>
        {cheeses.map((cheese) => (
          <li key={cheese.id}>{cheese.name}</li>
        ))}
      </ul>
      <Form method="post">
        <input type="text" name="cheese" placeholder="새 치즈 이름" />
        <button type="submit">추가</button>
      </Form>
    </div>
  );
}
```

action함수는 post, delete, patch와 같은 HTTP 메서드로 요청이 들어왔을 때 실행됩니다. 우리가 흔히 아는 react-query에서 mutation에 해당되는 개념이고, next.js에서는 서버액션과 동일합니다.

### Next.js

Next.js 13+의 App Router는 파일 기반 라우팅과 서버 컴포넌트를 지원합니다. 서버 액션을 사용하여 폼 제출이나 데이터 변경을 처리할 수 있습니다.

```jsx
// app/cheeses/page.jsx
import { sql } from "@vercel/postgres";
import { revalidatePath } from "next/cache";

// 서버 액션
async function addCheese(formData) {
  "use server";
  const name = formData.get("cheese");
  await sql`INSERT INTO cheeses (name) VALUES (${name})`;
  revalidatePath("/cheeses"); // 캐시 재검증
}

export default async function CheesePage() {
  // 서버 컴포넌트에서 직접 데이터 페칭
  const { rows } = await sql`SELECT * FROM cheeses`;

  return (
    <div>
      <h1>치즈 목록</h1>
      <ul>
        {rows.map((cheese) => (
          <li key={cheese.id}>{cheese.name}</li>
        ))}
      </ul>

      {/* 폼은 클라이언트 컴포넌트로 분리 가능 */}
      <form action={addCheese}>
        <input type="text" name="cheese" placeholder="새 치즈 이름" required />
        <button type="submit">추가</button>
      </form>
    </div>
  );
}
```

~~와 프론트에서 SQL를 직접쏜다 ㅋㅋ~~

App Router의 주요 특징:

- **서버 컴포넌트**: 기본적으로 모든 컴포넌트가 서버에서 실행되어 번들 크기 감소
- **서버 액션**: `"use server"` 지시자로 서버 함수를 정의하여 데이터 변경
- **캐시 전략**: `revalidatePath()`, `revalidateTag()`로 ISR 구현
- **레이아웃**: `layout.jsx`로 공통 UI 구성

### SvelteKit

SvelteKit은 서버와 클라이언트에서 동일한 코드를 사용할 수 있는 프레임워크입니다.

```svelte
<!-- src/routes/cheeses/+page.svelte -->
<script>
  export let data;
  let newCheese = "";

  async function addCheese() {
    const res = await fetch("/api/cheeses", {
      method: "POST",
      body: JSON.stringify({ name: newCheese }),
    });
    if (res.ok) {
      data.cheeses = [...data.cheeses, await res.json()];
      newCheese = "";
    }
  }
</script>

<h1>치즈 목록</h1>
<ul>
  {#each data.cheeses as cheese (cheese.id)}
    <li>{cheese.name}</li>
  {/each}
</ul>

<form on:submit|preventDefault={addCheese}>
  <input bind:value={newCheese} placeholder="새 치즈 이름" />
  <button type="submit">추가</button>
</form>
```

```javascript
// src/routes/cheeses/+page.server.js
export async function load() {
  const res = await fetch("https://api.com/cheeses");
  const cheeses = await res.json();
  return { cheeses };
}
```

### Astro

Astro는 기본적으로 정적 사이트 생성(SSG)에 최적화되어 있으며, 필요한 부분만 동적으로 처리합니다.

```astro
---
// src/pages/cheeses.astro
const cheeses = await fetch("https://api.com/cheeses").then(r => r.json());
---

<h1>치즈 목록</h1>
<ul>
  {cheeses.map((cheese) => (
    <li key={cheese.id}>{cheese.name}</li>
  ))}
</ul>

{/* CheesesForm은 클라이언트 컴포넌트 */}
<CheesesForm client:load />

<script>
  import CheesesForm from "../components/CheesesForm.jsx";
export { CheesesForm };
</script>
```

Astro의 특징:

- **기본 SSG**: 빌드 시점에 HTML 생성으로 최고의 성능
- **Partial Hydration**: 필요한 부분만 클라이언트 JS 로드
- **멀티 프레임워크**: React, Vue, Svelte 등과 함께 사용 가능

### Nuxt

Nuxt는 Vue.js를 기반으로 하는 풀스택 프레임워크입니다.

```vue
<!-- app.vue -->
<template>
  <div>
    <h1>치즈 목록</h1>
    <ul>
      <li v-for="cheese in cheeses" :key="cheese.id">{{ cheese.name }}</li>
    </ul>
    <form @submit.prevent="addCheese">
      <input v-model="newCheese" placeholder="새 치즈 이름" />
      <button type="submit">추가</button>
    </form>
  </div>
</template>

<script setup>
import { ref } from "vue";

const cheeses = ref([]);
const newCheese = ref("");

onMounted(async () => {
  cheeses.value = await $fetch("/api/cheeses");
});

const addCheese = async () => {
  const res = await $fetch("/api/cheeses", {
    method: "POST",
    body: { name: newCheese.value },
  });
  cheeses.value.push(res);
  newCheese.value = "";
};
</script>
```

### Qwik

Qwik은 가장 빠른 초기 로딩 시간을 목표로 하며, 사용자 상호작용이 필요한 부분만 동적으로 로드합니다.

```tsx
// src/routes/cheeses/index.tsx
import { component$, useSignal } from "@builder.io/qwik";

export default component$(function Cheeses() {
  const cheeses = useSignal<any[]>([]);
  const newCheese = useSignal("");

  return (
    <div>
      <h1>치즈 목록</h1>
      <ul>
        {cheeses.value.map((cheese) => (
          <li key={cheese.id}>{cheese.name}</li>
        ))}
      </ul>
      <form
        onSubmit$={async () => {
          const res = await fetch("/api/cheeses", {
            method: "POST",
            body: JSON.stringify({ name: newCheese.value }),
          });
          cheeses.value.push(await res.json());
          newCheese.value = "";
        }}
      >
        <input
          value={newCheese.value}
          onInput$={(e) => (newCheese.value = e.target.value)}
          placeholder="새 치즈 이름"
        />
        <button type="submit">추가</button>
      </form>
    </div>
  );
});
```

## 프레임워크 비교

### 기능 비교

| 프레임워크               | 라우팅    | 데이터 페칭               | 서버 액션                      | SSR/CSR/SSG           | 번들 크기 최적화                       |
| ------------------------ | --------- | ------------------------- | ------------------------------ | --------------------- | -------------------------------------- |
| **Next.js (App Router)** | 파일 기반 | Server Components / fetch | Server Actions                 | SSR / CSR / ISR / SSG | Server Components로 클라이언트 JS 감소 |
| **Remix**                | 파일 기반 | loader / action           | action 함수                    | SSR + Progressive CSR | 브라우저 JS 최소화 구조                |
| **SvelteKit**            | 파일 기반 | load 함수                 | form actions                   | SSR / CSR / SSG       | Svelte 컴파일러 최적화                 |
| **Astro**                | 파일 기반 | build-time / server       | 별도 Server Actions 없음       | SSG / SSR             | Partial Hydration (Islands)            |
| **Nuxt**                 | 파일 기반 | useAsyncData / useFetch   | server routes / server actions | SSR / CSR / SSG       | 자동 코드 스플리팅 + Nitro             |
| **Qwik**                 | 파일 기반 | Resource API              | server functions               | SSR / CSR             | Resumability (JS lazy execution)       |

### 성능 비교

| 프레임워크    | 초기 로딩  | Time to Interactive (TTI) | Core Web Vitals | SEO 최적화 |
| ------------- | ---------- | ------------------------- | --------------- | ---------- |
| **Astro**     | ⭐⭐⭐⭐⭐ | ⭐⭐⭐⭐⭐                | 최고            | 우수       |
| **Qwik**      | ⭐⭐⭐⭐⭐ | ⭐⭐⭐⭐⭐                | 최고            | 우수       |
| **SvelteKit** | ⭐⭐⭐⭐   | ⭐⭐⭐⭐                  | 우수            | 우수       |
| **Next.js**   | ⭐⭐⭐⭐   | ⭐⭐⭐⭐                  | 우수            | 우수       |
| **Nuxt**      | ⭐⭐⭐⭐   | ⭐⭐⭐⭐                  | 우수            | 우수       |
| **Remix**     | ⭐⭐⭐⭐   | ⭐⭐⭐⭐                  | 우수            | 우수       |

### 주요 성능 특징

**Astro**

- Islands Architecture: 필요한 컴포넌트만 hydration
- 나머지는 순수 HTML로 제공되어 최고의 성능

**Qwik**

- resumability 기반: 서버 상태를 그대로 전달
- 이벤트 발생 시에만 필요한 JS 로드

**Next.js**

- React Server Components로 서버에서만 실행
- streaming SSR로 점진적 렌더링

**SvelteKit**

- 컴파일러 기반 최적화
- 최소 런타임 JS로 번들 크기 감소

**Nuxt**

- Nitro 서버 엔진으로 통합된 백엔드
- 자동 코드 스플리팅

**Remix**

- 네이티브 웹 플랫폼 중심 설계
- loader/action 기반 명확한 데이터 흐름
