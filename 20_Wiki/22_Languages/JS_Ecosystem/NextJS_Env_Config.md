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
  - "[[NestJS_Env_Config]]"
  - "[[NextJS_ApiTypes_Mapper]]"
  - "[[NextJS_UI_Types]]"
  - "[[Deploy_CloudMVP]]"
---
# NextJS_Env_Config — 환경변수

> [!info]
>  Next.js 환경변수는 NestJS와 핵심이 다르다 
>   "서버에서만 보이는가, 브라우저에도 노출되는가"가 1순위 문제. `NEXT_PUBLIC_` 접두사 하나가 그 경계를 결정

---

# NestJS와 뭐가 다른가 ⭐️⭐️⭐️

| 구분                        | NestJS                                                      | Next.js                                               |
| ------------------------- | ----------------------------------------------------------- | ----------------------------------------------------- |
| `.env` 멀티파일 자동 인식         | ❌ (직접 `envFilePath` 지정 필요)                                  | ✅ `.env` < `.env.local` < `.env.production` 등 자동 우선순위 |
| `.env` → `process.env` 로드 | ConfigModule 또는 `--env-file` 플래그 ([[NestJS_Env_Config]] 참고) | 불필요 — Next.js가 항상 자동으로 로드                             |
| 서버/클라이언트 구분               | 없음 (전부 서버)                                                  | 있음 — `NEXT_PUBLIC_` 접두사가 핵심 규칙                        |
| 기본 검증 도구                  | Joi (선택)                                                    | Zod (선택) — Joi도 가능하지만 비표준                             |
| 프로덕션 `.env` 로드            | 보통 그대로 사용                                                   | ❌ 기본적으로 안 읽음 — 호스팅 플랫폼에 직접 등록                         |

---

# 0단계 — .env / .env.example 만들기

```properties
# .env.local — 로컬 개발용 (Git 제외)
DATABASE_URL=postgresql://postgres:password@localhost:5432/mydb
NEXT_PUBLIC_API_URL=http://localhost:3000
```

```properties
# .env.example — 팀 공유용 (Git 포함)
DATABASE_URL=
NEXT_PUBLIC_API_URL=
```

## 파일 우선순위 — Next.js가 자동으로 처리 ⭐️⭐️

|파일|용도|
|---|---|
|`.env`|모든 환경 공통 기본값|
|`.env.local`|로컬 전용, 최우선 — Next.js 기본 `.gitignore`에 포함됨|
|`.env.development` / `.env.production`|`next dev` / `next build` 시 자동 선택|
|`.env.test`|테스트 실행 시|

```txt
⚠️ 프로덕션에서는 .env 파일을 기본적으로 안 읽음
   Vercel 등 호스팅 플랫폼의 환경변수 설정에 직접 등록 — .env.production에 시크릿을 두지 말 것
   ([[Deploy_CloudMVP]] 배포 환경변수 설정 참고)
```

---

# 1단계 — NEXT_PUBLIC_ 접두사 ⭐️⭐️⭐️⭐️

```txt
Next.js 환경변수에서 가장 먼저 이해해야 하는 단 하나의 규칙:
  NEXT_PUBLIC_ 으로 시작 → 브라우저(클라이언트) 코드에서도 접근 가능, 빌드 시 JS 번들에 그대로 박힘
  접두사 없음            → 서버 코드(Server Component, Route Handler 등)에서만 접근 가능
```

|변수|위치|예시|
|---|---|---|
|`NEXT_PUBLIC_API_URL`|서버 + 브라우저|API 서버 주소처럼 클라이언트도 알아야 하는 값|
|`DATABASE_URL`|서버만|DB 비밀번호 등 절대 노출되면 안 되는 값|

```txt
⚠️ 빌드 타임에 "문자 그대로" 치환됨 (static replacement) — 런타임에 바뀌는 게 아님
   값을 바꿨다면 재빌드해야 반영됨 (서버 재시작만으로는 부족할 수 있음)

⚠️ 절대 하면 안 되는 것:
  const { NEXT_PUBLIC_API_URL } = process.env;  // ❌ 구조분해 — 정적 치환 안 됨 → undefined
  process.env[key]                               // ❌ 동적 키 — 마찬가지로 치환 안 됨
  process.env.NEXT_PUBLIC_API_URL                // ✅ 리터럴 형태로만 정상 동작

비밀값에 NEXT_PUBLIC_을 잘못 붙이면 그대로 브라우저에 공개됨 — 변수 추가할 때마다 항상 확인
```

```txt
NEXT_PUBLIC_API_URL처럼 "프론트가 백엔드 주소를 알아야 하는" 패턴은 거의 모든 프로젝트의 공통 변수
→ 실제 활용 (getApiBaseUrl, 절대경로 처리) → [[NextJS_API_Client]] 참고
```

---

# 2단계 — process.env 사용

```typescript
// 서버 코드 (Server Component, Route Handler, Server Action)
const dbUrl  = process.env.DATABASE_URL;          // 서버 전용 변수도 읽을 수 있음
const apiUrl = process.env.NEXT_PUBLIC_API_URL;   // 공개 변수도 당연히 읽을 수 있음
```

```typescript
// 클라이언트 코드 ('use client')
const apiUrl = process.env.NEXT_PUBLIC_API_URL;   // ✅ 됨 (접두사 있음)
const dbUrl  = process.env.DATABASE_URL;           // ❌ undefined (서버 전용이라 브라우저엔 없음)
```

```txt
이 단계만으로도 대부분의 프로젝트는 충분히 동작함 — 검증(3단계)은 선택사항
```

---

# 3단계 — 검증 추가 (선택) ⭐️⭐️

```txt
배포했는데 환경변수가 빠져서 빌드/런타임에 늦게 발견되는 걸 막고 싶을 때 추가하는 단계
Next.js/TS 진영은 Zod를 씀 (Joi는 NestJS 관례, 기술적으로는 가능하지만 비표준)
```

```bash
pnpm add zod
```

```typescript
// lib/env.ts — 가장 단순한 형태
import { z } from 'zod';

const schema = z.object({
  DATABASE_URL:        z.string().url(),
  NEXT_PUBLIC_API_URL: z.string().url(),
});

export const env = schema.parse(process.env);  // 누락/형식 오류 시 즉시 throw
```

## 더 안전하게 — server/client 분리 (@t3-oss/env-nextjs) ⭐️⭐️⭐️

```txt
단순 버전의 문제:
  서버 전용 스키마와 공개 스키마가 한 객체에 섞여 있어서
  "이 변수가 서버 전용인지 공개인지"를 코드만 보고 구분하기 어려움

@t3-oss/env-nextjs는 server/client를 분리해서:
  클라이언트 코드에서 서버 변수에 접근하면 타입 에러 + 런타임 에러로 바로 잡아줌
```

```bash
pnpm add @t3-oss/env-nextjs zod
```

```typescript
// lib/env.ts
import { createEnv } from '@t3-oss/env-nextjs';
import { z }         from 'zod';

export const env = createEnv({
  server: {
    DATABASE_URL: z.string().url(),
  },
  client: {
    NEXT_PUBLIC_API_URL: z.string().url(),  // NEXT_PUBLIC_ 없으면 타입 에러로 바로 알려줌
  },
  experimental__runtimeEnv: process.env,
});
```

|항목|역할|
|---|---|
|`server: {...}`|서버에서만 쓸 변수 — 클라이언트에서 import하면 에러|
|`client: {...}`|`NEXT_PUBLIC_` 접두사 필수, 빠뜨리면 타입 에러로 미리 발견|
|`z.coerce.number()` / `z.coerce.boolean()`|`.env` 값은 항상 string이므로 다른 타입이 필요하면 `coerce` 사용|

```txt
이후로는 process.env 직접 참조 대신 env 객체에서 꺼내 씀
  env.DATABASE_URL / env.NEXT_PUBLIC_API_URL
```

---

# 어느 단계까지 필요한가

|상황|권장|
|---|---|
|작은 프로젝트, 변수 몇 개뿐|2단계 (process.env)|
|배포 전 누락을 미리 잡고 싶음|3단계 (Zod 단순 버전)|
|서버/클라이언트 변수가 많고 혼동이 잦음|3단계 + @t3-oss/env-nextjs|

---

# 한눈에

```txt
0단계: .env / .env.local / .env.example
  Next.js가 우선순위까지 자동 처리 (NestJS와 다른 점)
  프로덕션은 .env 파일을 안 읽음 → 호스팅 플랫폼에 직접 등록

1단계: NEXT_PUBLIC_ 접두사 — 가장 중요한 규칙
  있음 → 빌드타임에 번들로 박힘 (브라우저에서도 접근 가능)
  없음 → 서버 코드에서만
  구조분해 / 동적 키 금지 — 리터럴 형태로만 정상 동작
  비밀값에 NEXT_PUBLIC_ 붙이면 브라우저에 공개됨 ⚠️

2단계: process.env — 대부분의 프로젝트는 이걸로 충분

3단계: 검증 (선택)
  Zod 단순 버전 → schema.parse(process.env)
  @t3-oss/env-nextjs → server/client 분리, 잘못된 접근을 타입/런타임에서 잡아줌

NEXT_PUBLIC_API_URL 실제 활용 → [[NextJS_API_Client]]
```