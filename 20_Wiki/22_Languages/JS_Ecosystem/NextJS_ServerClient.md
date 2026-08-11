---
aliases:
  - Client Component
  - Server Component
  - use client
  - use server
tags:
  - NextJS
  - React
related:
  - "[[00_JS_Ecosystem_HomePage]]"
  - "[[React_useId]]"
  - "[[NextJS_API_Client]]"
---
# NextJS_ServerClient — Server · Client Component

>[!info]
>Server Component = 서버에서만 실행 (Next.js 기본값).
> Client Component = `'use client'` 선언, 브라우저에서도 실행. 
> **판단 기준 하나: 이벤트 핸들러나 React 훅이 필요한가? 필요하면 Client, 아니면 Server.** 
> Hydration = 서버 HTML을 클라이언트 JS로 "살려내는" 과정.

---

# 핵심 구분 ⭐️⭐️⭐️⭐️

```txt
Next.js의 모든 컴포넌트는 기본이 Server Component
'use client'를 파일 맨 위에 쓰면 Client Component

판단 기준 — 딱 하나만 기억하면 됨:
  이벤트 핸들러(onClick, onChange)나 React 훅(useState, useEffect)이 필요한가?
  → YES: 'use client'
  → NO:  그냥 Server Component
```

|구분|Server Component|Client Component|
|---|---|---|
|선언|없음 (기본)|`'use client'` 파일 맨 위|
|실행 위치|서버에서만|서버(초기) + 브라우저|
|async/await|✅ 바로 사용 가능|❌ (useEffect로 우회)|
|useState, useEffect|❌|✅|
|onClick 등 이벤트|❌|✅|
|브라우저 API (window, localStorage)|❌|✅|
|DB·서버 환경변수 접근|✅|❌|
|번들 크기|0 (클라이언트 번들에 미포함)|번들에 포함됨|

---

# Server Component ⭐️⭐️⭐️⭐️

```typescript
// app/posts/page.tsx — 기본이 Server Component
// 'use client' 없으면 서버에서만 실행

export default async function PostsPage() {
  // ✅ async/await 바로 사용
  const posts = await fetch(`${process.env.API_URL}/posts`).then(r => r.json());
  //                         ↑ 서버 전용 환경변수 (NEXT_PUBLIC_ 없음)

  return (
    <ul>
      {posts.map((post) => (
        <li key={post.id}>{post.title}</li>
      ))}
    </ul>
  );
}
```

```txt
Server Component가 좋은 경우:
  데이터 페칭 (API 호출, DB 조회)
  민감한 환경변수 사용 (API 키 등)
  큰 라이브러리 사용 (번들 크기 절약)
  SEO가 중요한 콘텐츠

Server Component에서 못 하는 것:
  useState, useEffect, useCallback 등 React 훅
  onClick, onChange, onSubmit 등 이벤트 핸들러
  window, document, localStorage 등 브라우저 API
  → 이런 게 필요하면 'use client' 추가
```

---

# Client Component ⭐️⭐️⭐️⭐️

```typescript
// components/LikeButton.tsx
'use client';  // ← 파일 맨 위에 선언

import { useState } from 'react';

export function LikeButton({ postId }: { postId: string }) {
  const [liked, setLiked] = useState(false);  // ✅ 훅 사용 가능

  return (
    <button
      onClick={() => setLiked(!liked)}   // ✅ 이벤트 핸들러 사용 가능
    >
      {liked ? '❤️' : '🤍'}
    </button>
  );
}
```

```txt
Client Component가 필요한 경우:
  useState, useEffect 등 React 훅
  onClick, onChange 등 이벤트 핸들러
  window, localStorage, navigator 등 브라우저 API
  useRouter, useParams, useSearchParams (Next.js 훅)

Client Component도 처음 렌더링은 서버에서:
  서버에서 HTML을 미리 생성 (SSR)
  브라우저에서 JS를 받아서 "살려냄" (Hydration)
  → 초기 화면이 빠르고 SEO도 됨
```

---

# Hydration — 서버 HTML을 살려내기 ⭐️⭐️⭐️

```txt
Hydration 과정:
  ① 서버가 HTML을 생성해서 브라우저로 보냄
  ② 브라우저가 HTML을 즉시 표시 (빠른 초기 로딩)
  ③ JS 번들이 로드됨
  ④ React가 서버 HTML과 JS를 연결 (Hydration)
  ⑤ 이제 onClick 등 이벤트가 작동함

  ②와 ⑤ 사이에 클릭해도 반응 없음 — Hydration 완료 전
```

## Hydration 에러

```txt
에러: Hydration failed because the server rendered HTML
      didn't match the client

원인:
  서버에서 렌더링한 결과와
  클라이언트에서 렌더링한 결과가 다를 때

자주 발생하는 경우:
  window 또는 Date를 컴포넌트 최상위에서 사용
  → 서버에서는 undefined, 클라이언트에서는 값이 있어서 불일치

해결:
  useEffect 안에서 브라우저 API 사용
  또는 동적 import + ssr: false 사용
```

```typescript
// ❌ Hydration 에러 — 서버에 window 없음
function Component() {
  const width = window.innerWidth;  // 서버에서 에러
  return <div>{width}</div>;
}

// ✅ useEffect 안에서
function Component() {
  const [width, setWidth] = useState(0);
  useEffect(() => {
    setWidth(window.innerWidth);   // 브라우저에서만 실행
  }, []);
  return <div>{width}</div>;
}
```

---

# 혼용 패턴 ⭐️⭐️⭐️⭐️

## Server Component 안에 Client Component

```typescript
// app/posts/[id]/page.tsx — Server Component
import { LikeButton } from '@/components/LikeButton';  // Client Component

export default async function PostPage({ params }: { params: { id: string } }) {
  const post = await fetchPost(params.id);  // ✅ 서버에서 데이터 페칭

  return (
    <div>
      <h1>{post.title}</h1>
      <p>{post.content}</p>
      <LikeButton postId={post.id} />  {/* ✅ Client Component 포함 가능 */}
    </div>
  );
}
```

## Client Component에 Server Component를 children으로

```typescript
// ❌ Client Component 안에서 Server Component를 직접 import 안 됨
'use client';
import { ServerComp } from './ServerComp';  // 이러면 ServerComp도 Client가 됨

// ✅ children prop으로 Server Component를 전달
// ClientWrapper.tsx
'use client';
export function ClientWrapper({ children }) {
  const [open, setOpen] = useState(false);
  return <div onClick={() => setOpen(!open)}>{children}</div>;
}

// page.tsx (Server Component)
import { ClientWrapper } from './ClientWrapper';
import { ServerComp }   from './ServerComp';

export default function Page() {
  return (
    <ClientWrapper>
      <ServerComp />  {/* ✅ children으로 전달하면 Server 유지 */}
    </ClientWrapper>
  );
}
```

```txt
핵심 규칙:
  Client Component가 Server Component를 import하면
  → import된 것도 Client Component가 됨

  하지만 children prop으로 전달하면
  → 서버에서 렌더링된 결과를 Client가 받아서 표시
  → Server Component 특성 유지
```

---

# window is not defined 에러 ⭐️⭐️⭐️

```txt
원인: Server Component에서 브라우저 API 사용
     또는 Client Component가 서버에서 초기 렌더링될 때

해결책 3가지:
```

```typescript
// 방법 1 — 'use client' 추가
'use client';
import { useEffect, useState } from 'react';
function Component() {
  const [val, setVal] = useState('');
  useEffect(() => {
    setVal(localStorage.getItem('key') ?? '');
  }, []);
  return <div>{val}</div>;
}

// 방법 2 — 동적 import + ssr: false (브라우저에서만 로드)
import dynamic from 'next/dynamic';
const BrowserOnly = dynamic(() => import('./BrowserOnly'), { ssr: false });

// 방법 3 — typeof window 체크
const isClient = typeof window !== 'undefined';
const width = isClient ? window.innerWidth : 0;
```

---

# 언제 뭘 쓰는가 — 판단 흐름

```txt
컴포넌트를 만들 때:

  1. 데이터를 서버에서 가져와야 하나?
     → async/await가 필요하면 Server Component

  2. 사용자 상호작용이 있나?
     → onClick, onChange 등 이벤트가 있으면 Client Component

  3. React 훅이 필요한가?
     → useState, useEffect 등이 있으면 Client Component

  4. 브라우저 API가 필요한가?
     → window, localStorage 등이 있으면 Client Component

  나머지는 모두 Server Component (기본값)

권장 패턴:
  데이터 페칭은 부모(Server)에서
  → 데이터를 props로 자식(Client)에 전달
  → Client는 상호작용만 담당
```