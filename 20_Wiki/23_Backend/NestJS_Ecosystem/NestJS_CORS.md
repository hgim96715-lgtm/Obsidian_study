---
aliases:
  - CORS
  - HTTP
  - Security
tags:
  - NestJS
related:
  - "[[00_NestJS_Ecosystem_HomePage]]"
  - "[[NestJS_WebSocket]]"
  - "[[Web_Cookie]]"
  - "[[NestJS_Concept]]"
---
# NestJS_CORS — CORS 설정

>[!info]
>CORS(Cross-Origin Resource Sharing) = 브라우저의 기본 보안 정책(Same-Origin Policy)이 차단하는 cross-origin 요청을, 서버가 명시적으로 허용하는 메커니즘.
> 프론트(Vercel) + API(Railway)처럼 도메인이 다른 구조에서 로그인·API 호출이 안 되는 이유가 대부분 여기에 있다.

---

# CORS란 — 왜 존재하는가 ⭐️⭐️⭐️⭐️

```txt
Cross-Origin Resource Sharing
  Cross        = 다른 / 교차
  Origin       = 출처 (프로토콜 + 도메인 + 포트)
  Resource     = 리소스 (API 응답, 이미지 등)
  Sharing      = 공유

= "다른 출처의 리소스를 공유하는 것을 허용하는 메커니즘"
```

## 브라우저의 기본 정책 — Same-Origin Policy

```txt
브라우저는 기본적으로 다른 출처로의 요청을 차단함 (Same-Origin Policy)

왜 차단하는가 — 보안 위협 예시:
  내가 bank.com 에 로그인되어 있는 상태
  악성 사이트 evil.com 을 방문
  evil.com의 JavaScript가 몰래 bank.com/transfer 로 요청을 보냄
  → 내 쿠키가 자동으로 첨부됨 → 인증된 요청처럼 처리됨 → 계좌 이체 성공

  Same-Origin Policy가 없으면 이런 공격(CSRF)이 가능해짐
  → 브라우저는 기본적으로 다른 출처로의 요청을 막음

CORS = 이 차단을 선택적으로 풀어주는 메커니즘
  서버가 "나는 이 출처에서 오는 요청을 허용한다"고 헤더로 알려줌
  브라우저가 그 헤더를 보고 통과시킴
```

---

# Origin이란 ⭐️⭐️⭐️⭐️

```txt
Origin = 프로토콜 + 도메인 + 포트 세 가지의 조합

  https://my-app.vercel.app
  ↑ 프로토콜   ↑ 도메인              (포트 없음 = 443 기본)

  http://localhost:3000
  ↑ 프로토콜   ↑ 도메인    ↑ 포트

셋 중 하나라도 다르면 cross-origin:
```

|비교|결과|이유|
|---|---|---|
|`https://a.com` vs `https://a.com`|same-origin|동일|
|`https://a.com` vs `http://a.com`|cross-origin|프로토콜 다름|
|`https://a.com` vs `https://b.com`|cross-origin|도메인 다름|
|`http://localhost:3000` vs `http://localhost:3001`|cross-origin|포트 다름|
|`http://localhost:3000` vs `http://127.0.0.1:3000`|cross-origin|도메인 다름|

```txt
⚠️ localhost와 127.0.0.1은 같은 IP지만 브라우저는 다른 origin으로 취급
  → 둘 다 허용 목록에 넣어야 로컬 개발 시 CORS 에러 없음
```

---

# 브라우저에서 실제로 일어나는 일 ⭐️⭐️⭐️⭐️

```txt
CORS 오류의 핵심을 많이 오해하는 부분:
  "서버가 요청을 차단한다" → ❌ 틀림
  "브라우저가 응답을 차단한다" → ✅ 맞음

서버는 요청을 받고 처리하고 응답을 보냄
브라우저가 응답을 받고 CORS 헤더를 확인한 뒤
→ 허용된 origin이면: JavaScript에 응답 전달 ✅
→ 허용 안 된 origin이면: JavaScript에 응답 차단 ❌ (콘솔에 CORS 에러 표시)
```

```txt
흐름:
  브라우저 ──요청 + Origin: https://front.com──▶ 서버 (요청 처리 완료)
  브라우저 ◀──응답 + Access-Control-Allow-Origin: https://front.com── 서버

  브라우저가 Access-Control-Allow-Origin 확인:
    origin이 목록에 있음 → JavaScript에 응답 전달
    origin이 목록에 없음 → JavaScript에 "CORS 에러" → 응답 데이터 접근 불가
```

## Preflight — 허락부터 받고 보내는 요청

```txt
단순한 GET, POST(text)는 바로 요청을 보냄
하지만 복잡한 요청(Authorization 헤더, application/json body 등)은
브라우저가 먼저 "이 요청을 보내도 되나요?" 확인 요청을 먼저 보냄 → Preflight
```

```txt
Preflight 흐름:
  1. 브라우저 → OPTIONS 요청 (Preflight)
               Access-Control-Request-Method: POST
               Access-Control-Request-Headers: Authorization, Content-Type

  2. 서버 → 허용 응답
             Access-Control-Allow-Origin: https://front.com
             Access-Control-Allow-Methods: GET, POST, PATCH, DELETE
             Access-Control-Allow-Headers: Content-Type, Authorization

  3. 허용 확인됨 → 브라우저가 실제 요청 전송
  4. 서버가 실제 요청 처리 및 응답

Preflight가 실패하면:
  실제 요청 자체를 안 보냄 → 네트워크 탭에 OPTIONS 요청만 보이고 실제 요청 없음
  콘솔에 CORS 에러

maxAge 옵션:
  Preflight 결과를 캐싱하는 시간 (초)
  86400 = 24시간 — 같은 요청에 대해 24시간 동안 Preflight 생략
```

---

# NestJS enableCors() 설정 ⭐️⭐️⭐️⭐️

```typescript
// apps/api/src/main.ts
async function bootstrap() {
  const app = await NestFactory.create(AppModule);

  const configService  = app.get(ConfigService);
  const frontendUrl    = configService.get<string>(EnvKeys.FRONTEND_URL);
  const frontendOrigin = frontendUrl
    ? new URL(frontendUrl).origin  // 경로 제거 → origin만
    : undefined;

  app.enableCors({
    origin: frontendOrigin
      ? ['http://localhost:3031', 'http://127.0.0.1:3031', frontendOrigin]
      : undefined,    // frontendOrigin이 없으면 모든 출처 허용
    credentials: true,
  });

  await app.listen(configService.get<number>(EnvKeys.PORT) ?? 3030);
}
```

## new URL(frontendUrl).origin — 경로 제거

```typescript
new URL('https://my-app.vercel.app/some/path').origin
// → 'https://my-app.vercel.app'

new URL('https://my-app.vercel.app').origin
// → 'https://my-app.vercel.app'
```

```txt
왜 origin만 추출하는가:
  CORS origin 비교는 경로(path)를 포함하지 않음
  환경변수 FRONTEND_URL에 경로가 포함되어 있으면 origin 비교가 실패함
  → new URL(url).origin으로 경로를 제거한 도메인만 사용

삼항 연산자로 undefined 처리:
  frontendOrigin이 있으면 → 로컬 + 운영 origin 배열로 허용
  frontendOrigin이 없으면 → undefined → 모든 출처 허용 (로컬 개발용)
  값이 하나뿐이라 filter(Boolean)보다 삼항이 더 명확
```

---

# credentials — 쿠키 포함 요청 ⭐️⭐️⭐️⭐️

```txt
쿠키나 Authorization 헤더를 cross-origin 요청에서 주고받으려면
서버와 클라이언트 양쪽 모두 설정 필요 — 한쪽만 하면 동작 안 함
```

|위치|설정|
|---|---|
|서버 (NestJS)|`app.enableCors({ credentials: true })`|
|클라이언트 (fetch)|`fetch(url, { credentials: 'include' })`|
|클라이언트 (socket.io)|`io(url, { withCredentials: true })`|

```txt
⚠️ credentials: true 와 origin: '*' 는 같이 쓸 수 없음
  와일드카드 + credentials 허용은 보안상 브라우저가 차단
  credentials를 쓰려면 origin을 구체적인 주소로 반드시 명시

JWT(Bearer 헤더) 방식이면:
  쿠키가 아니라 Authorization 헤더에 토큰을 담음
  credentials: 'include' 는 쿠키 전송을 위한 것
  → 헤더 방식이면 credentials 설정 불필요할 수 있음
  (allowedHeaders에 'Authorization' 은 여전히 필요)
```

---

# WebSocket CORS 설정 ⭐️⭐️⭐️

```typescript
@WebSocketGateway({
  namespace: '/chat',
  cors: {
    origin: [
      'http://localhost:3031',    // 로컬 프론트
      'http://127.0.0.1:3031',   // 로컬 127.0.0.1
      process.env.FRONTEND_URL
        ? new URL(process.env.FRONTEND_URL).origin
        : undefined,
    ].filter(Boolean),
    credentials: true,
  },
})
```

```txt
HTTP CORS와 WebSocket CORS는 별도 설정
  main.ts의 enableCors()  → HTTP REST API
  @WebSocketGateway cors  → WebSocket 연결

둘 다 같은 패턴 (origin 목록 + credentials: true)
로컬에서 localhost와 127.0.0.1 둘 다 넣는 것도 동일하게 적용
```

---

# 주요 enableCors 옵션

|옵션|설명|기본값|
|---|---|---|
|`origin`|허용할 출처 — 문자열 / 배열 / 정규식 / `true`(전체)|`false`|
|`credentials`|`true` = 쿠키·Authorization 헤더 허용|`false`|
|`methods`|허용할 HTTP 메서드|`GET,HEAD,PUT,PATCH,POST,DELETE`|
|`allowedHeaders`|허용할 요청 헤더|`Content-Type, Authorization`|
|`exposedHeaders`|브라우저가 읽을 수 있게 노출할 응답 헤더|없음|
|`maxAge`|Preflight 결과 캐시 시간(초)|없음|

---

# 트러블슈팅

## CORS 에러인지 확인하는 방법

```txt
브라우저 콘솔:
  "Access to fetch at '...' from origin '...' has been blocked by CORS policy"
  → CORS 에러 확정

브라우저 네트워크 탭:
  OPTIONS 요청이 있고 응답 상태가 200이 아님 → Preflight 실패
  OPTIONS 요청 자체가 없고 실제 요청에 CORS 에러 → Simple request CORS 실패
  요청은 성공(200)인데 JavaScript에서 응답 못 받음 → origin 설정 누락
```

## 자주 하는 실수

|증상|원인|해결|
|---|---|---|
|로컬에서만 CORS 에러|`origin` 목록에 로컬 주소 없음|`localhost:포트`, `127.0.0.1:포트` 둘 다 추가|
|운영에서만 CORS 에러|`FRONTEND_URL` 환경변수 미설정|Railway/배포 환경에 `FRONTEND_URL` 추가|
|쿠키가 안 전달됨|`credentials` 설정 누락|서버 `credentials: true` + 클라이언트 `credentials: 'include'`|
|`origin: '*'`인데 쿠키 안 됨|와일드카드 + credentials 불가|origin을 구체적 주소로 변경|
|모바일(iOS Safari)에서만 401|iOS ITP가 cross-origin 쿠키 차단|→ 아래 섹션 참고|

## 모바일(iOS Safari) 로그인 안 됨 — ITP 문제 ⭐️⭐️⭐️⭐️

```txt
증상: 로그인 200 성공인데 이후 /auth/me 등이 전부 401
      PC 브라우저는 정상, 아이폰만 안 됨

원인:
  프론트: https://my-app.vercel.app
  API:    https://my-api.railway.app  ← 다른 도메인

  API에서 쿠키를 Set-Cookie로 보냄 → 쿠키가 my-api.railway.app 도메인에 귀속
  다음 요청에서 my-api.railway.app 으로 쿠키를 보내야 함 → cross-site 쿠키
  iOS Safari의 ITP(Intelligent Tracking Prevention): cross-site 쿠키를 차단

해결: Vercel rewrites로 API를 프론트 도메인 하위로 프록시
```

```json
// vercel.json
{
  "rewrites": [
    { "source": "/api/nest/:path*", "destination": "https://my-api.railway.app/:path*" }
  ]
}
```

```typescript
// 클라이언트에서 절대 URL 대신 상대 경로 사용
// ❌ process.env.NEXT_PUBLIC_API_URL = 'https://my-api.railway.app'
// ✅ process.env.NEXT_PUBLIC_API_URL = '/api/nest'

fetch('/api/nest/auth/login', { credentials: 'include' })
// → Vercel이 https://my-api.railway.app/auth/login 으로 프록시
// → 쿠키가 my-app.vercel.app 도메인에 귀속 → same-site → iOS 차단 없음
```

```txt
프록시가 해결하는 원리:
  브라우저 입장에서 요청은 /api/nest → 같은 도메인(same-site)
  Vercel 서버가 실제 API 서버로 중계
  Set-Cookie가 프론트 도메인(vercel.app)에 귀속
  → iOS ITP가 same-site 쿠키로 인식 → 차단 안 함

ITP 상세 → [[Web_Cookie]] "iOS Safari와 ITP" 섹션
```