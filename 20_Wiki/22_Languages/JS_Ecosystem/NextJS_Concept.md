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
  - "[[NextJS_API_Client]]"
  - "[[NextJS_API_Mapper]]"
  - "[[NextJS_Types]]"
  - "[[NestJS_CORS]]"
  - "[[NextJS_Concept]]"
  - "[[JS_Fetch_API]]"
---
# NextJS_Concept — Next.js 핵심 개념

>[!info]
>Next.js = React 위에 라우팅·SSR·이미지 최적화를 얹은 풀스택 프레임워크. SSR로 초기 로딩 속도와 SEO를 개선하고, Server Component로 서버에서 DB 접근까지 가능하다. 서버에는 `window`·`document`가 없으므로 브라우저 API는 `useEffect` 안에서만 사용한다.

---
# NestJS에서 Next.js로 — 머릿속 지도 ⭐️⭐️⭐️⭐️

```txt
NestJS를 배우고 Next.js를 보면 헷갈리는 이유:
  둘은 방향이 반대

  NestJS: 요청이 서버로 들어옴
           클라이언트 → 서버 (요청을 받는 쪽)

  Next.js: 브라우저가 fetch로 밖으로 나감
           브라우저 → 서버 (요청을 보내는 쪽)
```

|NestJS (`apps/api`)|Next.js (`apps/web`)|
|---|---|
|Module·Controller·Service|페이지·컴포넌트 + `lib` 유틸|
|`ConfigService` + Joi + `EnvKeys`|`NEXT_PUBLIC_*` + `.env.local`|
|Guard가 Bearer 토큰 검증|클라이언트가 헤더에 Bearer를 **붙여서 보냄**|
|요청이 서버로 **들어옴**|브라우저가 `fetch`로 **밖으로 나감**|
|`UsersModule` 새로 만들어서 추가|`lib/api.ts`에 함수 하나 추가|

```txt
갑자기 lib/api.ts를 쓰는 이유:
  page.tsx 안에 fetch를 매번 복붙하지 않으려고
  URL · Content-Type · Authorization 헤더 붙이는 규칙을 한 곳에
  NestJS로 치면 Service에 prisma 호출 모으는 것과 비슷한
  "얇은 클라이언트 레이어"

NestJS는 "UsersModule을 새로 만드는" 방식으로 기능을 추가
Next.js Web은 "화면 + HTTP 클라이언트"라서
  → 공통 호출 코드를 lib/에 모으는 것으로 충분

NestJS Guard가 Bearer를 검증한다면
Next.js는 그 Bearer를 Authorization 헤더에 담아 보내는 쪽
```

## 하고 싶은 것 — Nest 한 줄 ↔ Web 한 줄

|하고 싶은 것|NestJS (`apps/api`)|Next.js (`apps/web`)|
|---|---|---|
|설정 읽기|`configService.get(EnvKeys.PORT)`|`process.env.NEXT_PUBLIC_API_URL`|
|로그인|`AuthService.login()`|`lib/api.ts`의 `login()` → `fetch('/auth/login')`|
|"나" 조회|`@UserId()` + `UsersService.findMe()`|`Authorization: Bearer` + `GET /auth/me`|
|인증 강제|`JwtAuthGuard` (Guard가 차단)|토큰 없으면 요청 안 보내거나 UI에서 로그인 유도|
|역할 제한|`@Roles('admin')` + `RolesGuard`|토큰의 role을 클라이언트에서 확인해서 UI 분기|
|에러 처리|`throw new NotFoundException()`|`fetch` 응답의 `res.ok` 체크 후 throw|

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
apps/web/
├── app/                   라우트 (Next.js App Router)
│   ├── layout.tsx         루트 레이아웃
│   ├── page.tsx           → /
│   ├── posts/
│   │   └── [id]/
│   │       └── page.tsx   → /posts/123
│   └── api/
│       └── route.ts       API Route
│
├── src/ (또는 app과 같은 레벨)
│   ├── components/        재사용 UI 컴포넌트
│   ├── hooks/             커스텀 훅 (useXxx)
│   ├── lib/               API 클라이언트·유틸리티 함수
│   ├── types/             TypeScript 타입 정의
│   ├── store/             전역 상태 (Zustand 등)
│   ├── config/            env 등 설정
│   └── styles/            글로벌 CSS
```

## 각 폴더의 역할 ⭐️⭐️⭐️⭐️

```txt
app/
  Next.js 라우팅 전용 폴더
  page.tsx, layout.tsx, loading.tsx, error.tsx 등 특수 파일만
  실제 비즈니스 로직은 여기 넣지 않음

components/
  여러 페이지에서 재사용하는 UI 컴포넌트
  Button, Modal, PostCard, UserAvatar 등

hooks/
  커스텀 훅 (use로 시작)
  useAuth, usePosts, useInfiniteScroll 등
  데이터 페칭·상태 로직을 컴포넌트에서 분리

lib/
  "라이브러리 수준의 유틸리티" — 범용 함수들
  API 클라이언트 함수 (login, register, fetchPosts 등)
  유틸 함수 (cn, formatDate, formatNumber 등)
  → 컴포넌트도 훅도 아닌 순수 함수

types/
  TypeScript 타입 정의
  api.d.ts (openapi-typescript 생성), apiTypes.ts (커스텀 타입)

store/ (선택)
  전역 상태 관리 (Zustand, Jotai 등)
  로그인 유저 정보, 알림 등
```

## lib 폴더 — 무엇을 넣는가 ⭐️⭐️⭐️⭐️

```typescript
// lib/api.ts — NestJS API 클라이언트
// 서버에 요청을 보내는 함수들을 여기 모아둠

async function postAuth(
  path: 'login' | 'register',
  body: Record<string, string>,
): Promise<ApiAuthResponse> {
  const data = await fetchApi<ApiAuthResponse>(`/auth/${path}`, {
    method:  'POST',
    headers: { 'Content-Type': 'application/json' },
    body:    JSON.stringify(body),
  });
  setApiAccessToken(data.accessToken);
  return {
    accessToken: data.accessToken,
    user: { ...data.user, image: data.user.image ?? null },
  };
}

export function login(email: string, password: string) {
  return postAuth('login', { email: email.trim(), password });
}

export function register(email: string, password: string, nickname: string) {
  return postAuth('register', { email: email.trim(), password, nickname: nickname.trim() });
}

export async function fetchMe(): Promise<ApiAuthUser> {
  return authFetchApi<ApiAuthUser>('/auth/me');
}
```

```typescript
// lib/utils.ts — 범용 유틸리티
import { clsx } from 'clsx';
import { twMerge } from 'tailwind-merge';

// Tailwind 클래스 조합 유틸
export function cn(...inputs: ClassValue[]) {
  return twMerge(clsx(inputs));
}

// 날짜 포맷
export function formatDate(date: string | Date): string {
  return new Intl.DateTimeFormat('ko-KR').format(new Date(date));
}
```

```txt
lib에 넣는 기준:
  ✅ 서버 API 호출 함수 (login, fetchPosts 등)
  ✅ 범용 유틸리티 (cn, formatDate, formatNumber)
  ✅ 외부 서비스 클라이언트 설정

  ❌ React 훅 (useState, useEffect 포함) → hooks/
  ❌ JSX 반환 함수 → components/
  ❌ 전역 상태 → store/

API 클라이언트 상세 패턴 → [[NextJS_API_Client]]
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