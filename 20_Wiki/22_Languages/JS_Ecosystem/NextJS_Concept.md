---
aliases:
  - NextJS
  - Concept
  - Setup
  - SSR
  - AppRouter
tags:
  - NextJS
related:
  - "[[00_JS_Ecosystem_HomePage]]"
  - "[[NextJS_Concept]]"
  - "[[JS_URL_Encoding]]"
  - "[[JS_DOM]]"
  - "[[JS_BrowserAPI]]"
---
# NextJS_Concept — Next.js 핵심 개념

>[!info]
>Next.js = React 위에 라우팅·SSR·이미지 최적화를 얹은 풀스택 프레임워크. 
>SSR로 초기 로딩 속도와 SEO를 개선하고, Server Component로 서버에서 DB 접근까지 가능하다. 
>서버에는 `window`·`document`가 없으므로 브라우저 API는 `useEffect` 안에서만 사용한다.

---

# 왜 Next.js인가 — 순수 React의 한계 ⭐️⭐️⭐️⭐️

```txt
순수 React(Vite, CRA):
  서버가 빈 HTML + JavaScript 파일만 보냄
  브라우저가 JS를 다운로드·실행해서 화면을 그림

문제 1 — 느린 초기 로딩:
  JS 다운로드 → 실행 → 데이터 fetch → 화면 표시
  사용자가 흰 화면을 보는 시간이 김

문제 2 — SEO 불리:
  검색 엔진 봇이 JS 실행 전에 빈 HTML만 봄
  → 콘텐츠를 인식 못해서 검색 순위에 불리

Next.js:
  서버에서 HTML을 완성해서 보내줌 (SSR)
  → 브라우저가 HTML을 받자마자 콘텐츠 표시
  → 검색 엔진도 완성된 HTML을 읽음
```

---

# 렌더링 방식 비교 ⭐️⭐️⭐️⭐️

## CSR — Client-Side Rendering (클라이언트 렌더링)

```txt
흐름: 서버 → 빈 HTML + JS → 브라우저가 JS 실행 → 화면 그림

서버:  <html><body><div id="root"></div><script src="app.js"></script></body></html>
           ↑ 내용 없음

브라우저: app.js 실행 → fetch('/api/data') → 데이터 오면 React가 그림

언제 씀:
  로그인 후 개인화된 대시보드 (SEO 불필요)
  실시간 업데이트가 많은 앱
  순수 React 앱의 기본 방식
```

## SSR — Server-Side Rendering (서버 렌더링)

```txt
흐름: 요청마다 서버에서 HTML 생성 → 완성된 HTML을 브라우저에 전달

서버:  DB 조회 → 데이터를 HTML에 삽입 → 완성된 HTML 전송
       <html><body><h1>홍길동님의 피드</h1><article>...</article></body></html>
           ↑ 내용이 채워진 상태

브라우저: HTML 즉시 표시 → JS 로드 후 React가 이어받음 (hydration)

언제 씀:
  콘텐츠가 자주 바뀌는 페이지 (뉴스, 소셜 피드)
  로그인한 유저마다 다른 데이터
  SEO가 중요한 페이지
```

## SSG — Static Site Generation (정적 생성)

```txt
흐름: 빌드 시 미리 HTML 생성 → 요청 시 파일 바로 제공 (CDN)

빌드 시: 모든 페이지를 미리 렌더링해서 HTML 파일로 저장
요청 시: 저장된 HTML 파일을 그냥 보냄 (DB 조회 없음)

장점: 매우 빠름, 서버 부하 없음
단점: 데이터가 바뀌어도 반영 안 됨 (재빌드 필요)

언제 씀:
  블로그 포스트, 문서, 마케팅 페이지
  데이터가 거의 안 바뀌는 콘텐츠
```

## ISR — Incremental Static Regeneration

```txt
SSG + 주기적 재생성:
  처음엔 SSG처럼 빌드 시 생성
  revalidate 시간이 지나면 백그라운드에서 재생성
  사용자는 항상 캐시된 HTML을 받고, 백그라운드에서 최신화

// Next.js에서 설정
fetch('/api/data', { next: { revalidate: 60 } })  // 60초마다 재검증
```

## 비교표

|방식|HTML 생성 시점|속도|SEO|실시간 데이터|
|---|---|---|---|---|
|CSR|브라우저에서|느림(초기)|❌|✅|
|SSR|요청마다 서버|빠름|✅|✅|
|SSG|빌드 시|매우 빠름|✅|❌|
|ISR|빌드+주기적|빠름|✅|△|

---

# Server Component vs Client Component ⭐️⭐️⭐️⭐️

```txt
Next.js App Router의 핵심 개념:
  Server Component = 서버에서만 실행 (기본값)
  Client Component = 브라우저에서 실행 ('use client' 명시)
```

## Server Component (기본)

```tsx
// app/posts/page.tsx — 'use client' 없으면 Server Component

// 서버에서만 실행되므로:
//   DB 직접 접근 가능
//   API 키를 클라이언트에 노출 안 해도 됨
//   useState, useEffect 사용 불가 (서버 개념 없음)
//   이벤트 핸들러 불가 (onClick 등)

async function PostsPage() {
  const posts = await prisma.post.findMany();  // DB 직접 접근
  return (
    <ul>
      {posts.map(post => <li key={post.id}>{post.title}</li>)}
    </ul>
  );
}
```

## Client Component

```tsx
'use client';  // ← 이 한 줄로 Client Component로 전환

// 브라우저에서 실행되므로:
//   useState, useEffect 사용 가능
//   이벤트 핸들러(onClick 등) 가능
//   DB 직접 접근 불가 (API를 통해 접근해야 함)
//   window, document 접근 가능

import { useState } from 'react';

function LikeButton({ postId }: { postId: string }) {
  const [liked, setLiked] = useState(false);  // ✅ Client에서 가능
  return (
    <button onClick={() => setLiked(!liked)}>  {/* ✅ 이벤트 핸들러 */}
      {liked ? '❤️' : '🤍'}
    </button>
  );
}
```

## 언제 어느 것을 쓰는가 ⭐️⭐️⭐️⭐️

```txt
기본 전략:
  Server Component로 시작 → 인터랙션이 필요하면 Client로 전환

Server Component가 좋은 경우:
  데이터 fetch (DB 조회, API 호출)
  인터랙션 없는 표시용 컴포넌트
  레이아웃, 네비게이션 (정적인 것)
  민감한 정보 (API 키, DB 연결 등)

Client Component가 필요한 경우:
  useState, useReducer, useRef 사용
  useEffect, useCallback, useMemo 사용
  onClick, onChange 등 이벤트 핸들러
  window, document 등 브라우저 API
  실시간 업데이트 (WebSocket)
```

```tsx
// 혼합 패턴 — Server가 데이터 fetch, Client에게 전달
// app/posts/[id]/page.tsx (Server Component)
async function PostPage({ params }: { params: { id: string } }) {
  const post = await prisma.post.findUnique({ where: { id: params.id } });
  return (
    <div>
      <h1>{post.title}</h1>
      <LikeButton postId={post.id} />  {/* Client Component 내포 */}
    </div>
  );
}
```

```txt
Server Component가 Client Component를 포함할 수 있음
Client Component가 Server Component를 포함할 수 없음
  → 'use client' 아래로는 전부 Client로 취급
```

---

# App Router 폴더 구조 ⭐️⭐️⭐️⭐️

```txt
app/
├── layout.tsx           루트 레이아웃 — 모든 페이지 공통 (html, body 태그)
├── page.tsx             루트 페이지 → /
├── loading.tsx          로딩 UI (Suspense 자동 적용)
├── error.tsx            에러 UI
├── not-found.tsx        404 페이지
│
├── posts/
│   ├── page.tsx         → /posts
│   └── [id]/
│       └── page.tsx     → /posts/123
│
└── api/
    └── users/
        └── route.ts     API Route → GET/POST /api/users
```

## 각 파일 역할

```typescript
// page.tsx — 페이지 컴포넌트 (라우트의 UI)
export default function Page() { ... }

// layout.tsx — 레이아웃 (하위 page들에 공유)
export default function Layout({ children }: { children: React.ReactNode }) {
  return <div><Header />{children}</div>;
}

// loading.tsx — page.tsx가 로딩 중일 때 표시
export default function Loading() {
  return <div>로딩 중...</div>;
}

// error.tsx — 에러 발생 시 (반드시 'use client')
'use client';
export default function Error({ error, reset }) {
  return <button onClick={reset}>다시 시도</button>;
}
```

---

# 데이터 페칭 방식 ⭐️⭐️⭐️⭐️

```typescript
// Server Component에서 — async/await 직접 사용
async function PostsPage() {
  // 방법 1: API 호출
  const posts = await fetch('https://api.example.com/posts').then(r => r.json());

  // 방법 2: DB 직접 (Prisma)
  const posts = await prisma.post.findMany();

  return <PostList posts={posts} />;
}
```

```typescript
// Client Component에서 — useEffect 또는 SWR/React Query
'use client';

function PostList() {
  const [posts, setPosts] = useState([]);

  useEffect(() => {
    fetch('/api/posts').then(r => r.json()).then(setPosts);
  }, []);

  return ...;
}
```

```txt
Server Component에서 데이터 fetch vs Client Component:
  Server: async/await 직접 — 서버에서 렌더링 시 데이터 포함
  Client: useEffect + fetch — 브라우저에서 추가 요청

  가능하면 Server Component에서 fetch 권장:
  → 클라이언트에 데이터 이미 포함돼서 보임 → 로딩 깜빡임 없음
  → API Key 등 민감 정보 서버에서만 처리
```

---

# Hydration ⭐️⭐️⭐️

```txt
SSR로 완성된 HTML을 받은 뒤
React가 그 HTML을 "이어받아서" 이벤트 핸들러 등을 연결하는 과정

흐름:
  1. 서버 → 완성된 HTML 전송
  2. 브라우저 → HTML 즉시 표시 (콘텐츠 보임)
  3. React JS 로드 완료
  4. Hydration — React가 HTML에 이벤트 핸들러 연결
  5. 인터랙션 가능 상태

Hydration 오류:
  서버에서 렌더링한 HTML과 클라이언트에서 React가 그린 결과가 다를 때
  → "Hydration failed" 에러
  예: Math.random(), new Date() 같이 실행마다 다른 값
  예: window 등 서버에 없는 브라우저 API를 컴포넌트 최상단에서 사용
```

---

# 서버 vs 브라우저 환경 — window가 없다 ⭐️⭐️⭐️⭐️

```txt
Next.js는 서버에서도 컴포넌트를 실행함 (SSR, Server Component)
서버 = Node.js 환경 → 브라우저 API가 없음

서버에 없는 것:
  window           → 브라우저의 전역 객체
  document         → DOM 조작
  localStorage     → 브라우저 저장소
  navigator        → 브라우저 정보
  location         → 현재 URL
```

```tsx
// ❌ Server Component에서 window 사용 — 빌드/실행 에러
export default async function Page() {
  const width = window.innerWidth;  // ReferenceError: window is not defined
  return <div>{width}</div>;
}

// ❌ Client Component 최상단에서 사용 — Hydration 에러
'use client';
const id = localStorage.getItem('userId');  // 서버 렌더링 시 없음

export default function Component() {
  return <div>{id}</div>;
}
```

## 해결 방법 ⭐️⭐️⭐️⭐️

```tsx
// ✅ 방법 1 — useEffect 안에서 (클라이언트에서만 실행)
'use client';
import { useEffect, useState } from 'react';

export default function Component() {
  const [width, setWidth] = useState(0);

  useEffect(() => {
    setWidth(window.innerWidth);  // 클라이언트에서만 실행 → 안전
  }, []);

  return <div>{width}</div>;
}
```

```tsx
// ✅ 방법 2 — typeof window 체크
const isClient = typeof window !== 'undefined';
const width    = isClient ? window.innerWidth : 0;
```

```tsx
// ✅ 방법 3 — mounted 패턴 (Portal, 모달에서 자주 씀)
'use client';

export function Modal({ open }: { open: boolean }) {
  const [mounted, setMounted] = useState(false);
  useEffect(() => { setMounted(true); }, []);

  if (!mounted) return null;  // 서버 렌더링 시 아무것도 안 보냄

  return createPortal(<div>...</div>, document.body);  // 클라이언트에서만
}
```

```txt
상황별 선택:
  렌더링 중 값이 필요 → useState + useEffect (초기값 0/null로 시작)
  조건 분기 → typeof window !== 'undefined'
  컴포넌트 자체를 서버에서 안 보내려면 → mounted 패턴
  SSR 자체를 끄려면 → dynamic import + ssr: false

  → window 관련 API 상세 → [[JS_BrowserAPI]]
```

---

# 각 개념의 상세 노트

```txt
라우팅 · 동적 경로 · Link   → [[NextJS_Routing]]
서버/클라이언트 컴포넌트    → [[NextJS_ServerClient]]
API Route (route.ts)        → [[NextJS_API_Client]]
타입 · 매퍼                  → [[NextJS_Types]]
메타데이터 · SEO             → [[NextJS_Metadata]]
WebSocket 클라이언트         → [[NextJS_WebSocket]]
```