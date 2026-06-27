---
aliases:
  - Next.js 개념
  - Next.js 란
tags:
  - NextJS
related:
  - "[[00_JS_Ecosystem_HomePage]]"
  - "[[NextJS_Routing]]"
  - "[[NextJS_Data_Fetching]]"
  - "[[NextJS_Env_Config]]"
  - "[[Monorepo_PNPM]]"
---
# NextJS_Concept — Next.js 란

# 한 줄 요약

```
Next.js = React 기반 풀스택 프레임워크 (Vercel 제작, 오픈소스)
SSR / SSG / 라우팅 / API Route 내장
```

---

---

# React 만으로 부족한 것 ⭐️

|항목|React 단독|Next.js 가 해결|
|---|---|---|
|라우팅|직접 설치 필요 (react-router-dom)|폴더 구조 = 자동 라우팅|
|SEO|CSR 기본 → 검색엔진 불리|SSR / SSG 기본 지원|
|API 서버|별도 구성 필요|같은 프로젝트 안에 Route Handler 내장|
|이미지/폰트 최적화|직접 구현|내장|

---

---

# App Router vs Pages Router ⭐️

|구분|Pages Router (Next.js 12 이하)|App Router (Next.js 13+, 현재 권장)|
|---|---|---|
|예시|`pages/index.tsx` → `/`|`app/page.tsx` → `/`|
|동적 라우트|`pages/movie/[id].tsx`|`app/movie/[id]/page.tsx`|
|기본 컴포넌트|전부 Client|Server Component 기본|
|공통 레이아웃|`_app.tsx` 하나|`layout.tsx` 폴더별로 중첩 가능|

---

---

# 폴더 기반 라우팅 — 개념만 짧게

```
폴더 구조 = URL 구조
app/page.tsx → /, app/about/page.tsx → /about, app/movie/[id]/page.tsx → /movie/1 ...

특수 파일: page.tsx(페이지) / layout.tsx(공통 레이아웃) / loading.tsx / error.tsx / route.ts(API)
```

> `[id]` 동적 라우팅, `(그룹)` 라우트 그룹, `useParams`/`usePathname` 등 자세한 건 [[NextJS_Routing]] 참고

---

---

# Server Component vs Client Component ⭐️

|구분|Server Component (기본)|Client Component|
|---|---|---|
|실행 위치|서버 — HTML 생성 후 전달|브라우저|
|선언|없음 (기본값)|파일 맨 위 `'use client'`|
|DB 직접 접근|가능|불가|
|`useState`/`useEffect`/이벤트 핸들러|불가|가능|
|용도|데이터 페칭 / 정적 UI|인터랙티브 UI|

```tsx
// Server Component (기본 — 선언 없음)
async function MovieList() {
  const movies = await fetch('https://api.example.com/movies');
  return <ul>...</ul>;
}

// Client Component
'use client';
import { useState } from 'react';

function Counter() {
  const [count, setCount] = useState(0);
  return <button onClick={() => setCount(count + 1)}>{count}</button>;
}
```

> 어느 쪽을 써야 하는지 판단 기준, 컴포넌트 분리 패턴은 [[NextJS_Data_Fetching]] 참고

---

---

# 렌더링 방식 — 개념만 짧게

|방식|시점|예시 옵션|
|---|---|---|
|SSR|요청마다 서버에서 새로 생성|`cache: 'no-store'`|
|SSG|빌드 시점에 미리 생성 (빠름)|`cache: 'force-cache'`|
|ISR|SSG + 주기적 재생성|`next: { revalidate: 60 }`|

> 캐싱 옵션별 기본값, Cache Components 등 자세한 건 [[NextJS_Data_Fetching]] / [[NextJS_Caching_Frontend]] 참고

---

---

# 프로젝트 생성

```bash
npx create-next-app@latest my-app          # 대화형 설치

# 옵션 직접 지정 (질문 없이 한 번에)
pnpm create next-app@latest web --typescript --eslint --app --no-src-dir --import-alias "@/*"
```

```
생성되는 구조:
  app/layout.tsx   루트 레이아웃
  app/page.tsx     홈 페이지 (/)
  public/          정적 파일
  next.config.js   Next.js 설정
```

> 모노레포(apps/web 구조)로 시작한다면 → [[Monorepo_PNPM]] 참고

## 프로젝트가 커지면 추가하는 폴더 — lib / types / components ⭐️⭐️⭐️

```
create-next-app 이 만들어주는 건 app/ 뿐 — 컴포넌트/로직/타입을 어디 둘지는 관례로 정함
(Next.js 가 강제하는 규칙은 아니라 프로젝트마다 폴더명이 조금씩 다를 수 있음)
```

```
my-app/
├── app/              라우팅 (page.tsx, layout.tsx, route.ts ...)
├── components/       재사용 UI 컴포넌트
├── lib/              컴포넌트가 아닌 로직 — API 호출, 유틸 함수, DB 클라이언트 등
├── types/            TS 타입 정의 (API 응답 타입, 공통 타입)
└── public/
```

|폴더|넣는 것|예시|
|---|---|---|
|`lib/`|"컴포넌트가 아닌" 모든 로직 — API 래퍼, 인증 헬퍼, DB 클라이언트, 포맷 함수|`lib/api.ts`, `lib/prisma.ts` ([[NextJS_API_Integration]] 참고)|
|`types/`|프로젝트 전역에서 쓰는 타입 — API 응답 타입, 공통 인터페이스|`types/movie.ts`|
|`components/`|재사용 가능한 UI 조각|`components/Button.tsx`|

```
프로젝트 시작할 때 보통 같이 비워두는 자리:
  types/index.ts(또는 types.ts) — 도메인이 아직 안 잡혔어도, 공통 타입 둘 자리부터 마련
  lib/api-base.ts          — getApiBaseUrl 처럼 "어디 둘지 애매한" 로직의 기본 자리

⚠️ lib/ 와 utils/ 를 혼용하는 프로젝트도 있음 — 정해진 규칙은 없고 관례적인 이름일 뿐
  중요한 건 "컴포넌트(화면)" 와 "로직(데이터/연동)" 을 분리해서 찾기 헷갈리지 않게 하는 것
```

---

---

# Turbopack ⭐️

```
Turbopack = Next.js 에 내장된 Rust 기반 번들러 (Webpack 대체)
개발 서버 시작 속도 / HMR(핫 리로드) 속도가 훨씬 빠름
```

|버전|상태|
|---|---|
|13~14|실험적, 별도 활성화 필요|
|15|`next dev` 에서 안정화 (옵션으로 사용)|
|16+ (현재 최신 메이저)|`next dev` 와 `next build` 모두 기본값 — 별도 플래그 불필요|

```bash
pnpm dev     # Next.js 16+ 부터는 이것만으로 Turbopack 사용 (별도 플래그 불필요)
pnpm build   # 프로덕션 빌드도 16+ 부터 Turbopack 기본
```

```
⚠️ 커스텀 Webpack 설정이 있는 프로젝트는 16+ 에서 빌드가 실패할 수 있음
   → --webpack 플래그로 기존 Webpack 유지, 또는 설정을 Turbopack 호환으로 이전
   모노레포처럼 web/ 같은 하위 폴더에서 실행할 땐 next.config.ts 의 turbopack.root 로
   파일 찾는 루트 경로를 지정해야 할 수 있음
```

---

---

# NestJS 와 함께 쓸 때 ⭐️⭐️

```
Next.js  → 프론트엔드 (UI / 라우팅 / SEO)
NestJS   → 백엔드 API (DB / 인증 / 비즈니스 로직)

흐름: Next.js 컴포넌트 → fetch(NestJS API 주소) → 응답 받아 렌더링
```

## 왜 포트를 분리해야 하나

```
NestJS 기본 포트도 3000, Next.js 기본 포트도 3000 → 같이 띄우면 충돌
해결: 둘 중 하나(보통 Next.js)를 다른 포트로 띄움 — 포트 번호 자체는 임의, 안 겹치면 됨
```

## 포트 설정 — .env 가 아니라 --port 또는 package.json ⭐️⭐️⭐️

```
⚠️ Next.js 의 PORT 는 .env/.env.local 에 적어도 적용되지 않음
   Next.js 의 HTTP 서버가 바인딩할 포트를 결정하는 시점이 .env 파일 로딩보다 먼저이기 때문
   (Next.js 공식 문서: "PORT cannot be set in .env as booting up the HTTP server happens
    before any other code is initialized")
→ 포트는 항상 CLI 플래그 또는 셸 환경변수로 지정해야 함
```

```json
// package.json
{
  "scripts": {
    "dev":   "next dev --port 3001",
    "build": "next build",
    "start": "next start --port 3001"
  }
}
```

|방법|예시|
|---|---|
|package.json 스크립트 (팀 공유, 권장) ⭐️|`"dev": "next dev --port 3001"`|
|CLI 플래그 직접|`next dev -p 3001`|
|셸 환경변수 (1회성)|`PORT=3001 next dev`|

```
⚠️ 가장 자주 까먹는 지점 — --port 는 dev 뿐 아니라 start 에도 필요함
  dev 에만 주면, 빌드 후 next start 로 실행할 때는 다시 기본값(3000)으로 돌아감
  build/lint 에는 포트가 필요 없음 (서버를 띄우는 명령이 아니라서)

우선순위: -p/--port 플래그 > PORT 환경변수(셸) > 기본값 3000
  (.env/.env.local 의 PORT 는 이 우선순위에 전혀 끼지 못함 — 위 이유 참고)
```

## 모노레포에서 루트로 한 번에 실행하기

```json
// 루트 package.json
{
  "scripts": {
    "dev:api": "pnpm --filter api start:dev",
    "dev:web": "pnpm --filter web dev"
  }
}
```

```
루트 package.json 의 name/private/scripts 작성 규칙, --filter 명령, 여러 개 동시 실행하는 법은
전부 [[Monorepo_PNPM]] 의 "루트 package.json — 무엇을 적는가" 참고 — 여기서는 예시만
```

## 환경변수로 API URL 관리

```bash
# web/.env.local
NEXT_PUBLIC_API_URL=http://localhost:3000
```

```
NEXT_PUBLIC_ 접두사, .env.local vs .env 우선순위, 서버/클라이언트 구분 등
환경변수 관련 디테일은 전부 [[NextJS_Env_Config]] 로 정리해뒀음 — 여기서는 NestJS 연동 맥락만
이 URL 을 실제로 fetch 에 활용하는 방법은 [[NextJS_API_Integration]] 참고
```

---

---

# 한눈에

```
포트가 헷갈릴 때 기억할 한 줄:
  .env 가 아니라 --port 플래그(또는 package.json 스크립트), dev 뿐 아니라 start 에도 똑같이

개념 vs 적용 — 이 노트는 "Next.js 가 뭔지" 와 "NestJS 와 같이 쓸 때 포트" 까지만
```

| 상황                                          | 어디를 보면 되는가                 |
| ------------------------------------------- | -------------------------- |
| App Router 폴더 규칙 / 동적 라우팅 / useParams       | [[NextJS_Routing]]         |
| fetch 캐싱 / Server·Client 컴포넌트 선택 / Suspense | [[NextJS_Data_Fetching]]   |
| 환경변수 / NEXT_PUBLIC_ / .env.local            | [[NextJS_Env_Config]]      |
| fetchAPI 래퍼 / getApiBaseUrl / 도메인 API 함수    | [[NextJS_API_Integration]] |
| 모노레포 설정 / pnpm workspace                    | [[Monorepo_PNPM]]          |
| NestJS PORT 환경변수                            | [[NestJS_Concept]]         |