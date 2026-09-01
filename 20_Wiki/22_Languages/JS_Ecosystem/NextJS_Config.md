---
aliases: [next.config.ts, nextConfig, remotePatterns, transpilePackages]
tags: [NextJS, config]
relations:
  - "[[00_JS_Ecosystem_HomePage]]"
  - "[[Monorepo_PNPM]]"
  - "[[NextJS_Concept]]"
  - "[[NextJS_Env_Config]]"
---

# NextJS_Config — next.config.ts 설정

> [!info]
> 프로젝트 루트에 위치하는 Next.js 빌드·런타임 설정 파일
> 웹팩, 이미지, 환경변수, 리다이렉트, 헤더 등 Next.js 전반을 제어
> 변경 후 `next dev` 재시작 필요

---

## 파일 위치 및 기본 구조

```typescript
// next.config.ts (프로젝트 루트)
import type { NextConfig } from "next";

const nextConfig: NextConfig = {
  // 설정들...
};

export default nextConfig;
```

```txt
next.config.js  — CommonJS (module.exports = {...})
next.config.ts  — TypeScript (export default {...}) — Next.js 15+부터 공식 지원
NextConfig 타입 임포트 → 자동완성 + 타입 검사 활용 가능
```

---

## `transpilePackages` — ESM 전용 패키지 처리

```typescript
transpilePackages: ['@nivo/bar', '@nivo/core', '@nivo/heatmap', '@nivo/line', '@nivo/pie'],
```

```txt
문제 배경:
  npm 패키지 중 일부는 ESM(ES Modules)으로만 배포됨
  Next.js는 기본적으로 node_modules를 트랜스파일하지 않음
  → ESM 전용 패키지를 그냥 import하면 에러 발생

  SyntaxError: Cannot use import statement in a module
  SyntaxError: Unexpected token 'export'

transpilePackages:
  명시한 패키지를 Next.js 웹팩이 직접 트랜스파일(SWC로 변환)하도록 지시
  → ESM → CommonJS 호환 형태로 변환 → 에러 없이 import 가능

대표 사례:
  @nivo/* (차트 라이브러리) — ESM 전용이라 반드시 추가 필요
```

| 에러 메시지 | 원인 | 해결 |
|---|---|---|
| `Cannot use import statement` | ESM only 패키지 | `transpilePackages`에 추가 |
| `Unexpected token 'export'` | 동일 | 동일 |

---

## `images.remotePatterns` — 외부 이미지 허용

```typescript
images: {
  remotePatterns: [
    {
      protocol: 'https',
      hostname: 'image.tmdb.org',
      pathname: '/t/p/**',
    },
  ],
},
```

```txt
배경:
  Next.js <Image> 컴포넌트는 이미지를 자동 최적화 (WebP 변환, 리사이즈, lazy load)
  보안상 허용된 외부 도메인에서만 이미지를 가져올 수 있음
  → 허용하지 않은 외부 URL → Error: Invalid src

remotePatterns 필드:
  protocol  — 'https' | 'http'
  hostname  — 허용할 도메인 (image.tmdb.org)
  pathname  — 허용할 경로 패턴 ('/**' = 모든 경로, '/t/p/**' = /t/p/ 하위만)
  port      — 포트 지정 (생략 시 기본 포트)

TMDB 이미지 URL 구조:
  https://image.tmdb.org/t/p/w500/abc123.jpg
                         ↑ /t/p/** 로 커버
```

> [!warning]
> `<img>` 태그(일반 HTML)는 이 설정 무관 — Next.js `<Image>` 컴포넌트에만 적용

---

## 자주 쓰는 옵션 모음

```typescript
const nextConfig: NextConfig = {
  // ① ESM 전용 패키지 트랜스파일
  transpilePackages: ['@nivo/bar', '@nivo/core'],

  // ② 외부 이미지 도메인 허용
  images: {
    remotePatterns: [
      { protocol: 'https', hostname: 'image.tmdb.org', pathname: '/t/p/**' },
      { protocol: 'https', hostname: 'avatars.githubusercontent.com', pathname: '/**' },
    ],
  },

  // ③ 환경변수 클라이언트 노출 (빌드타임에 값 고정)
  env: {
    API_BASE_URL: process.env.API_BASE_URL,
  },

  // ④ 리다이렉트
  async redirects() {
    return [
      { source: '/old', destination: '/new', permanent: true },  // 301
      { source: '/temp', destination: '/new', permanent: false }, // 302
    ];
  },

  // ⑤ 커스텀 응답 헤더
  async headers() {
    return [
      {
        source: '/(.*)',
        headers: [
          { key: 'X-Frame-Options', value: 'DENY' },
          { key: 'X-Content-Type-Options', value: 'nosniff' },
        ],
      },
    ];
  },

  // ⑥ 실험적 기능
  experimental: {
    serverActions: { allowedOrigins: ['localhost:3000'] },
  },
};
```

---

## 환경변수와의 차이 → [[NextJS_Env_Config]]

| 설정 | 위치 | 용도 |
|---|---|---|
| `env: {}` in next.config.ts | next.config.ts | 빌드타임에 값 고정, 코드에서 `process.env.KEY` |
| `NEXT_PUBLIC_` 접두사 | .env.local | 브라우저에 노출되는 런타임 환경변수 |
| `@t3-oss/env-nextjs` | 별도 파일 | Zod 스키마로 환경변수 타입 검증 |

```txt
next.config.ts의 env:{}
  → 빌드 시 값이 번들에 인라인됨
  → 런타임에 변경 불가 (재빌드 필요)
  → NEXT_PUBLIC_ 환경변수와 달리 .env 파일 없이도 프로그래밍 방식으로 주입 가능
```

---

> [!note] 핵심
> `transpilePackages` — ESM only 패키지를 Next.js가 직접 트랜스파일 → import 에러 해결
> `images.remotePatterns` — `<Image>` 컴포넌트가 허용할 외부 도메인·경로 목록
> 변경 후 반드시 개발 서버 재시작
