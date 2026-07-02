---
aliases:
  - CORS
  - Security
  - HTTP
tags:
  - NestJS
related:
  - "[[00_NestJS_Ecosystem_HomePage]]"
  - "[[Deploy_CloudMVP]]"
  - "[[JS_Fetch_API]]"
  - "[[Web_XSS_CSRF]]"
  - "[[NextJS_TokenStorage]]"
---
# NestJS_CORS — Cross-Origin 요청 허용

> [!info] 
> CORS = 브라우저가 다른 도메인(origin)으로 요청 보낼 때 서버가 명시적으로 허용해야 하는 보안 정책.
>  Vercel(프론트) + Railway(API)처럼 도메인이 다른 구조에서 로그인·쿠키가 안 되는 이유가 대부분 여기에 있다.

---
# 흐름도

```mermaid-beautiful
flowchart TB
    FE["프론트<br/>origin A · vercel.app"] --> BR["브라우저"]
    BR -->|"cross-origin fetch"| API["Nest API<br/>origin B · railway.app"]
    API --> CORS["main.ts enableCors<br/>origin · credentials"]
    CORS --> HDR["Access-Control-* 응답 헤더"]
    HDR --> BR
    BR -->|origin 매칭| OK["요청 통과"]
    BR -->|미설정 · 불일치| BLOCK["브라우저 차단"]
```

```txt
same-origin = 프로토콜·도메인·포트 모두 같음 — CORS 불필요
cross-origin = 하나라도 다름 — 서버가 origin을 명시 허용해야 함
credentials: true 는 서버·fetch 양쪽 모두 — origin: '*' 와 같이 쓸 수 없음
```

---

# CORS가 필요한 이유 ⭐️⭐️⭐️

```txt
same-origin = 프로토콜 + 도메인 + 포트가 모두 같음
cross-origin = 셋 중 하나라도 다름

  프론트: https://my-app.vercel.app
  API:    https://my-api.railway.app  ← 도메인이 다름 → cross-origin

브라우저는 보안상 cross-origin 요청을 기본적으로 차단함
→ 서버가 "이 출처는 허용한다"고 응답 헤더로 알려줘야 브라우저가 통과시킴

로컬에서도:
  프론트: http://localhost:3000
  API:    http://localhost:3001  ← 포트가 다름 → cross-origin → CORS 설정 필요
```

---

# NestJS enableCors() ⭐️⭐️⭐️⭐️

## main.ts 설정

```typescript
// apps/api/src/main.ts
import { NestFactory } from '@nestjs/core';
import { ConfigService } from '@nestjs/config';

async function bootstrap() {
  const app = await NestFactory.create(AppModule);

  const configService = app.get(ConfigService);
  const frontendUrl   = configService.get<string>('FRONTEND_URL');
  const frontendOrigin = frontendUrl
    ? new URL(frontendUrl).origin   // URL에서 origin(프로토콜+도메인)만 추출
    : undefined;

  app.enableCors({
    origin: frontendOrigin
      ? [
          'http://localhost:3000',    // 로컬 개발 (프론트)
          'http://127.0.0.1:3000',   // 일부 브라우저는 127.0.0.1을 localhost와 다르게 봄
          frontendOrigin,             // 운영 프론트엔드 도메인
        ]
      : undefined,                   // FRONTEND_URL 없으면 CORS 제한 없음 (로컬 전용)
    credentials: true,               // 쿠키/Authorization 헤더 허용
  });

  await app.listen(3000);
}
```

## new URL(frontendUrl).origin — 왜 origin만 추출하는가

```txt
FRONTEND_URL 환경변수에는 경로까지 포함될 수 있음:
  https://my-app.vercel.app/recommendations  ← /recommendations 경로가 붙어있음
  https://my-app.vercel.app                  ← 이게 origin (프로토콜 + 도메인)

CORS의 origin 비교는 경로를 포함하지 않음 → 경로 포함된 URL을 그대로 쓰면 매칭 실패
→ new URL(frontendUrl).origin 으로 경로를 제거한 도메인만 사용

  new URL('https://my-app.vercel.app/path').origin
  // → 'https://my-app.vercel.app'
```

---

# 환경변수로 분리 ⭐️⭐️⭐️

```typescript
// ❌ 하드코딩 — Git에 도메인 노출 + 환경마다 코드 수정 필요
app.enableCors({
  origin: ['https://my-app.vercel.app'],
  credentials: true,
});

// ✅ 환경변수로 분리
app.enableCors({
  origin: [
    'http://localhost:3000',
    'http://127.0.0.1:3000',
    process.env.FRONTEND_URL,
  ].filter(Boolean),   // FRONTEND_URL 없을 때 undefined 제거 → 배열에 undefined 들어가면 에러
  credentials: true,
});
```

```txt
filter(Boolean)이 필요한 이유:
  process.env.FRONTEND_URL이 없으면 undefined
  origin 배열에 undefined가 들어가면 예기치 않은 동작 발생
  → filter(Boolean)으로 falsy 값(undefined, null, '') 전부 제거

127.0.0.1을 따로 추가하는 이유:
  브라우저에 따라 localhost와 127.0.0.1을 다른 origin으로 취급하는 경우가 있음
  curl이나 일부 도구는 127.0.0.1로 접근 → 둘 다 허용하는 게 안전
```

---

# credentials: true — 양쪽 모두 설정 ⭐️⭐️⭐️⭐️

```txt
쿠키나 Authorization 헤더를 cross-origin 요청에서 주고받으려면
서버(NestJS)와 클라이언트(fetch) 양쪽 모두 설정이 필요함
한쪽만 하면 동작 안 함
```

|위치|설정|
|---|---|
|서버 (NestJS)|`app.enableCors({ credentials: true })`|
|클라이언트 (fetch)|`fetch(url, { credentials: 'include' })`|

```typescript
// 클라이언트 — credentials: 'include' 없으면 쿠키가 안 붙어서 전송됨
const res = await fetch(`${API_URL}/auth/me`, {
  credentials: 'include',
});
```

```txt
⚠️ credentials: true 와 origin: '*' 는 같이 쓸 수 없음
  → 와일드카드 허용 + credentials 허용은 보안상 브라우저가 차단
  → credentials를 쓰려면 origin을 구체적인 주소로 명시해야 함

fetch에서 credentials: 'include'가 언제 필요한지 → [[JS_Fetch_API]] 참고
JWT(Bearer 헤더) 방식이면 credentials 설정 불필요 — 헤더에 직접 담기 때문
```

---

# 트러블슈팅 — 모바일(iOS Safari) 로그인·인증 안 됨 ⭐️⭐️⭐️⭐️

## 증상

|항목|내용|
|---|---|
|증상|로그인 API 응답은 200(성공)인데, 이후 인증이 필요한 요청(`/auth/me`, 찜하기 등)이 전부 401|
|재현 환경|iOS Safari, 일부 모바일 브라우저|
|PC 브라우저|전혀 문제없음|

## 원인

```txt
NEXT_PUBLIC_API_URL = 'https://my-api.railway.app' (절대 URL) 로 설정했을 때:

  프론트 도메인: https://my-app.vercel.app
  API 도메인:    https://my-api.railway.app  ← 다른 도메인

  PC 브라우저: cross-origin 쿠키지만 허용 (CORS 설정 있으므로)
  iOS Safari:  서드파티 쿠키(cross-site)로 인식 → ITP(Intelligent Tracking Prevention)가 차단

  → 로그인 응답의 Set-Cookie는 railway.app 도메인에 세팅됨
  → 이후 fetch 요청에 쿠키가 안 붙음 → 서버는 인증 정보 없음 → 401
```

## 해결 — API URL을 상대 경로로 변경

```txt
핵심 아이디어:
  프론트에서 /api/nest 처럼 상대 경로로 요청
  → Vercel이 같은 vercel.app 도메인에서 Railway로 프록시
  → 브라우저 입장에서는 same-origin 요청
  → 쿠키가 vercel.app 도메인에 귀속됨 → iOS Safari도 차단 안 함
```

```txt
Vercel 환경변수 변경:
  ❌ NEXT_PUBLIC_API_URL = https://my-api.railway.app   (절대 URL)
  ✅ NEXT_PUBLIC_API_URL = /api/nest                     (상대 경로)

Vercel rewrites 설정 (vercel.json):
  {
    "rewrites": [
      { "source": "/api/nest/:path*", "destination": "https://my-api.railway.app/:path*" }
    ]
  }

서버사이드(Next.js Server Component)에서 직접 호출할 때는:
  브라우저 제약 없음 → API_INTERNAL_URL = https://my-api.railway.app 로 직접 호출
  NEXT_PUBLIC_이 없는 환경변수 → 서버에서만 접근 가능 (브라우저 노출 안 됨)
```

## 확인 방법

```txt
수정 후 확인:
  모바일 Safari에서 로그인 → 개발자 도구 or 쿠키 확인
  쿠키가 vercel.app 도메인(프론트 도메인)에 있으면 성공
  railway.app 도메인에 있으면 여전히 문제

PC에서는 이 버그가 안 나타나는 이유:
  PC 브라우저들은 SameSite=None; Secure 설정된 cross-origin 쿠키를 허용하는 경우가 많음
  iOS Safari는 설정과 무관하게 third-party 쿠키를 강하게 차단 (ITP)
```

---

# 주요 enableCors 옵션

|옵션|설명|
|---|---|
|`origin`|허용할 출처 — 문자열 / 문자열 배열 / 정규식 / `true`(전체)|
|`credentials`|`true` = 쿠키/Authorization 헤더 허용 (origin이 구체적 주소일 때만)|
|`methods`|허용할 HTTP 메서드 — 기본값: `GET,HEAD,PUT,PATCH,POST,DELETE`|
|`allowedHeaders`|허용할 요청 헤더 — 기본값: `Content-Type, Authorization`|
|`exposedHeaders`|브라우저가 읽을 수 있게 노출할 응답 헤더|
|`maxAge`|preflight 결과 캐시 시간(초) — 기본값: 없음|

```typescript
// 더 구체적인 설정이 필요할 때
app.enableCors({
  origin: ['http://localhost:3000', frontendOrigin].filter(Boolean),
  credentials: true,
  methods: ['GET', 'POST', 'PATCH', 'DELETE'],
  allowedHeaders: ['Content-Type', 'Authorization'],
  maxAge: 86400,   // preflight 24시간 캐싱
});
```

---

# 한눈에

```txt
CORS가 필요한 상황:
  프론트와 API의 도메인/포트가 다를 때 → cross-origin → enableCors() 필수
  로컬: localhost:3000(프론트) + localhost:3001(API) → 포트 달라서 cross-origin

enableCors() 설정:
  origin: [로컬주소, 127.0.0.1주소, frontendOrigin].filter(Boolean)
  credentials: true (쿠키 허용 — origin이 *이면 사용 불가)
  FRONTEND_URL에서 new URL(url).origin으로 경로 제거 후 사용

양쪽 모두 설정:
  서버: credentials: true
  클라이언트: fetch의 credentials: 'include' ([[JS_Fetch_API]] 참고)

환경변수 분리:
  origin 하드코딩 금지 → FRONTEND_URL 환경변수 사용
  filter(Boolean)으로 undefined 제거

모바일(iOS Safari) 401 버그:
  원인: API 절대 URL → 쿠키가 API 도메인에 귀속 → iOS ITP가 서드파티로 인식해 차단
  해결: NEXT_PUBLIC_API_URL을 /api/nest 상대 경로로 + Vercel rewrites로 프록시
  → 쿠키가 프론트 도메인에 귀속 → same-site → iOS 차단 안 함
```