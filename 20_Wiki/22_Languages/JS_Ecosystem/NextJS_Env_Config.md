---
aliases:
  - NEXT_PUBLIC
  - Next.js 환경변수
  - t3-env, zod env
tags:
  - NextJS
related:
  - "[[00_JS_Ecosystem_HomePage]]"
  - "[[Monorepo_PNPM]]"
  - "[[NextJS_Concept]]"
---
# NextJS_Env_Config — Next.js 환경변수

>[!info]
>`NEXT_PUBLIC_` 접두사가 없으면 서버에서만 접근 가능. 있으면 클라이언트(브라우저)에서도 접근 가능 — 브라우저 소스에 노출됨.
>`.env.local`은 git에 올리지 않고 로컬 개발에만 사용.
>프로덕션에서 `.env` 파일 기본 미로드 → 배포 플랫폼 대시보드에 직접 설정.
>⚠️ `NEXT_PUBLIC_` 변수는 **빌드 시 번들에 정적으로 포함** — Vercel에 등록 후 **반드시 재배포**해야 반영됨.
>검증이 필요하면 Zod + `@t3-oss/env-nextjs`.
>NestJS 서버 환경변수 → [[NestJS_Env_Config]]

---

# NEXT_PUBLIC_ — 핵심 개념 ⭐️⭐️⭐️⭐️

```txt
Next.js 환경변수의 가장 중요한 규칙:

  접두사 없음 (API_URL):
    서버에서만 접근 가능
    빌드 시 번들에 포함되지 않음
    클라이언트에서 접근하면 undefined

  NEXT_PUBLIC_ 접두사 (NEXT_PUBLIC_API_URL):
    서버 + 클라이언트(브라우저) 모두에서 접근 가능
    빌드 시 번들에 정적으로 포함됨
    브라우저 소스에서 값을 볼 수 있음
```

```typescript
// Server Component — 둘 다 접근 가능
export default async function Page() {
  const secret = process.env.DATABASE_URL;        // ✅ 서버에서만
  const apiUrl = process.env.NEXT_PUBLIC_API_URL; // ✅ 서버에서도
}

// Client Component ('use client')
'use client';
export function Component() {
  const secret = process.env.DATABASE_URL;        // ❌ undefined (서버 전용)
  const apiUrl = process.env.NEXT_PUBLIC_API_URL; // ✅ 브라우저에서도
}
```

```txt
⚠️ NEXT_PUBLIC_ 변수는 브라우저 소스에서 노출됨
  → API 키, 비밀번호, 토큰 등 민감한 값에 NEXT_PUBLIC_ 쓰면 안 됨
  → 브라우저에서 봐도 되는 값만 (API URL, 앱 이름 등)

서버에서만 써야 하는 값은 접두사 없이:
  DATABASE_URL, JWT_SECRET → 접두사 없음 (서버 전용)
  NEXT_PUBLIC_API_URL      → 접두사 있음 (클라이언트에서도 API 주소 필요)
```

---

# .env 파일 종류 ⭐️⭐️⭐️⭐️

```txt
Next.js가 인식하는 .env 파일 (우선순위 높은 순):

  .env.local           로컬 개발 전용 — git에 올리지 않음 (가장 우선)
  .env.development     next dev 실행 시 (NODE_ENV=development)
  .env.production      next build/start 시 (NODE_ENV=production)
  .env                 모든 환경 공통

  .env.local이 가장 우선이므로 로컬에서 개별 override 가능
```

```bash
# .env.local (로컬 개발용 — gitignore)
NEXT_PUBLIC_API_URL=http://localhost:3030
NEXT_PUBLIC_APP_NAME=Music Community

# .env.production (배포 환경 — gitignore)
NEXT_PUBLIC_API_URL=https://api.myapp.com

# .env.example (git에 올려도 됨 — 키만 공유)
NEXT_PUBLIC_API_URL=
NEXT_PUBLIC_APP_NAME=
```

```txt
.gitignore에 반드시 추가:
  .env.local
  .env.production
  .env*.local

.env.example은 올려도 됨:
  실제 값 없이 키 목록만
  팀원이 어떤 환경변수가 필요한지 파악하는 용도
```

---

# 사용 방법 ⭐️⭐️⭐️⭐️

## Server Component에서

```typescript
// app/page.tsx — Server Component
export default async function Page() {
  // 빌드 타임에 교체됨
  const apiUrl = process.env.NEXT_PUBLIC_API_URL;

  // 서버 전용 (클라이언트에서 접근 불가)
  const secret = process.env.SESSION_SECRET;

  const data = await fetch(`${apiUrl}/posts`);
  ...
}
```

## Client Component에서

```typescript
// components/ApiClient.ts
'use client';

// NEXT_PUBLIC_ 만 사용 가능
const API_URL = process.env.NEXT_PUBLIC_API_URL;

export async function fetchPosts() {
  const res = await fetch(`${API_URL}/posts`, {
    headers: { Authorization: `Bearer ${token}` },
  });
  return res.json();
}
```

## next.config.ts에서 env 추가

```typescript
// next.config.ts
const nextConfig = {
  env: {
    APP_VERSION: '1.0.0',   // process.env.APP_VERSION 으로 접근 가능
  },
};
```

---

# 타입 안전하게 사용하기 ⭐️⭐️⭐️

```typescript
// src/config/env.ts — 환경변수를 한 곳에서 관리
const getEnv = (key: string): string => {
  const value = process.env[key];
  if (!value) throw new Error(`Missing env: ${key}`);
  return value;
};

export const env = {
  apiUrl:  process.env.NEXT_PUBLIC_API_URL ?? 'http://localhost:3030',
  appName: process.env.NEXT_PUBLIC_APP_NAME ?? 'App',
} as const;
```

```typescript
// 사용
import { env } from '@/config/env';

fetch(`${env.apiUrl}/posts`);
// process.env.NEXT_PUBLIC_API_URL을 직접 쓰는 것보다 타입 안전
```

```txt
process.env.NEXT_PUBLIC_X를 직접 쓰면:
  타입이 string | undefined
  undefined 체크를 매번 해야 함

env 객체로 한 곳에서 관리하면:
  기본값 설정 가능
  타입이 string으로 좁혀짐
  사용처에서 undefined 체크 불필요
```

---

# 배포 환경 설정 — Vercel ⭐️⭐️⭐️⭐️

## Vercel 등록 절차

```txt
프로젝트 → Settings → Environment Variables
  → Name:        NEXT_PUBLIC_API_URL
  → Value:       https://api-production-xxxx.up.railway.app
  → Environment: Production ✅ (Preview / Development는 선택)
  → Save
```

```txt
⚠️ Railway API URL 값 주의:
  Railway 도메인 전체를 그대로 입력
  https://api-production-xxxx.up.railway.app   ← /v1 붙이지 않음 ⭐️

  /v1은 코드에서 경로로 붙이는 것 — 베이스 URL에 포함 금지

  올바른 사용:
    env: NEXT_PUBLIC_API_URL=https://api-production-xxxx.up.railway.app
    코드: fetch(`${process.env.NEXT_PUBLIC_API_URL}/v1/users`)
       → https://api-production-xxxx.up.railway.app/v1/users  ✅

  잘못된 사용:
    env: NEXT_PUBLIC_API_URL=https://api-production-xxxx.up.railway.app/v1
    코드: fetch(`${process.env.NEXT_PUBLIC_API_URL}/v1/users`)
       → https://api-production-xxxx.up.railway.app/v1/v1/users  ❌ /v1 중복
```

---

## ⚠️ 변수 변경 후 반드시 재배포 ⭐️⭐️⭐️⭐️

```txt
NEXT_PUBLIC_ 변수는 빌드 시 브라우저 번들에 정적으로 포함됨
  → Next.js가 빌드 시 process.env.NEXT_PUBLIC_X를 실제 값으로 치환해서 번들에 박아버림
  → 런타임에 동적으로 읽는 구조가 아님

결과:
  Vercel에 환경변수를 새로 등록하거나 값을 바꿔도
  기존 빌드 결과물(번들)에는 반영 안 됨
  → 재배포(새 빌드)를 해야 새 값이 번들에 들어감

Vercel Redeploy 방법:
  방법 A: main 브랜치에 커밋 push → 자동 배포
  방법 B: Vercel 대시보드 → Deployments 탭 → 최근 배포 → ⋯ → Redeploy

증상:
  Vercel Settings에 등록 완료했는데 여전히 undefined
  → 재배포 안 한 것 → Redeploy 실행
```

---

## 환경별 값 관리

```txt
로컬 개발:
  .env.local
    NEXT_PUBLIC_API_URL=http://localhost:3030

Vercel Production:
  Settings → Environment Variables → Production 환경
    NEXT_PUBLIC_API_URL=https://api-production-xxxx.up.railway.app

Vercel Preview (PR 미리보기 배포):
  별도 스테이징 API URL 지정 가능
  없으면 Production과 동일한 값으로 설정

Railway (NestJS API 서버):
  프로젝트 → Variables 탭
  → DATABASE_URL, JWT_SECRET 등 NestJS 서버 전용 환경변수
  → NEXT_PUBLIC_ 변수는 여기 넣지 않음 (Vercel 쪽 변수)
```

---

# NestJS와 뭐가 다른가 ⭐️⭐️⭐️

|구분|NestJS|Next.js|
|---|---|---|
|`.env` 멀티파일 자동 인식|❌ `envFilePath` 직접 지정 필요|✅ `.env` → `.env.local` → `.env.production` 자동 우선순위|
|`.env` → `process.env` 로드|ConfigModule 또는 `--env-file` → [[NestJS_Env_Config]]|불필요 — Next.js가 항상 자동 로드|
|서버/클라이언트 구분|없음 (전부 서버)|있음 — `NEXT_PUBLIC_` 접두사가 핵심 규칙|
|검증 도구|Joi (NestJS 관례)|Zod (Next.js·TS 진영 표준)|
|프로덕션 `.env` 로드|보통 그대로 사용|❌ 기본적으로 안 읽음 — 호스팅 플랫폼에 직접 등록|

```txt
프로덕션에서 .env 파일 안 읽는 이유:
  Next.js는 프로덕션에서 .env.local, .env.production 파일을
  기본적으로 자동 로드하지 않음
  → Vercel·Railway 같은 플랫폼의 환경변수 대시보드에서 직접 설정
  → 배포 환경에서 .env 파일을 서버에 올리는 방식은 권장하지 않음
```

---

# 타입 안전 검증 — Zod + @t3-oss/env-nextjs ⭐️⭐️⭐️

```txt
왜 @t3-oss/env-nextjs가 필요한가:
  배포했는데 환경변수가 빠져서 빌드/런타임에 늦게 발견되는 문제
  → 빌드 시점에 env를 검증해서 즉시 발견

NestJS에서 Joi를 쓰듯
Next.js·TS 진영은 Zod를 씀
  Joi도 기술적으로 가능하지만 Next.js에서는 비표준
  Zod = TypeScript 네이티브, 타입 자동 추론
```

## 설치

```bash
pnpm --filter web add @t3-oss/env-nextjs zod
```

## env.ts 작성

```typescript
// src/env.ts
import { createEnv } from '@t3-oss/env-nextjs';
import { z } from 'zod';

export const env = createEnv({
  // 서버에서만 사용하는 환경변수 (NEXT_PUBLIC_ 없음)
  server: {
    NODE_ENV: z.enum(['development', 'production', 'test']).default('development'),
  },

  // 클라이언트에서도 사용하는 환경변수 (NEXT_PUBLIC_ 접두사 필수)
  client: {
    NEXT_PUBLIC_API_URL:  z.string().url(),
    NEXT_PUBLIC_APP_NAME: z.string().min(1).default('App'),
  },

  // 실제 process.env에서 읽어올 값 명시
  runtimeEnv: {
    NODE_ENV:             process.env.NODE_ENV,
    NEXT_PUBLIC_API_URL:  process.env.NEXT_PUBLIC_API_URL,
    NEXT_PUBLIC_APP_NAME: process.env.NEXT_PUBLIC_APP_NAME,
  },
});
```

## next.config.ts에 연결

```typescript
// next.config.ts
import './src/env'; // 빌드 시 env 검증 실행

const nextConfig = { ... };
export default nextConfig;
```

## 사용

```typescript
import { env } from '@/env';

// 서버 컴포넌트
const nodeEnv = env.NODE_ENV;           // string, undefined 아님

// 클라이언트 컴포넌트
const apiUrl = env.NEXT_PUBLIC_API_URL;  // string, undefined 아님

// ❌ server 변수를 클라이언트에서 쓰면 빌드 에러
// ❌ client 변수를 server에서만 선언하면 빌드 에러
```

```txt
@t3-oss/env-nextjs의 장점:
  server / client를 명시적으로 분리
  Zod로 타입 검증 → 빌드 시 누락된 환경변수 즉시 발견
  반환 타입이 string | undefined 아닌 string (검증 통과 후)
  server 변수를 클라이언트 번들에 실수로 포함하면 빌드 에러

언제 추가하면 되는가:
  처음엔 process.env 직접 사용으로 시작
  배포 환경에서 env 누락 문제가 생기거나
  server/client 분리가 명확히 필요해지면 그때 추가
```

| 에러                                    | 원인                           | 해결                                      |
| ------------------------------------- | ---------------------------- | --------------------------------------- |
| 클라이언트에서 env가 `undefined`              | `NEXT_PUBLIC_` 접두사 없음        | `NEXT_PUBLIC_` 붙이기                      |
| 서버에서만 쓰는 값이 번들에 포함됨                  | `NEXT_PUBLIC_` 붙인 민감 정보      | 접두사 제거 (서버에서만 접근)                       |
| Vercel에 등록했는데 배포 후에도 `undefined`      | 등록 후 재배포 안 함                 | Vercel Deployments → Redeploy ⭐️        |
| API 요청 경로가 `/v1/v1/users` 로 중복됨       | env에 `/v1` 포함시킴              | env에는 도메인만, `/v1`은 코드에서 붙임 ⭐️          |
| `.env.local` 변경 후 로컬에서 반영 안 됨        | dev 서버 재시작 필요               | `pnpm dev` 재실행                          |