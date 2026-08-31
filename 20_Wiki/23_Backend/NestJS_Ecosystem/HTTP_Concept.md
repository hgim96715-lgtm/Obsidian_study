---
aliases:
  - HTTP
  - Request
  - Response
  - Headers
  - Authorization
  - Content-Type
  - Cookie
  - CORS
  - HTTPS
  - 상태코드
  - 멱등성
  - Cache-Control
  - ETag
tags:
  - NodeJS
  - NestJS
  - Network
related:
  - "[[00_NestJS_Ecosystem_HomePage]]"
  - "[[NestJS_JwtGuard]]"
  - "[[NestJS_Controller]]"
  - "[[NestJS_Prisma]]"
  - "[[NestJS_AsyncJob]]"
---
# HTTP_Concept — HTTP 완전 정리

>[!info]
> HTTP = 클라이언트 ↔ 서버 통신 규약. 요청(Request) + 응답(Response)이 한 쌍.
> 헤더 = 메타 정보(인증·형식·캐시), 바디 = 실제 데이터.
> REST CRUD → [[NestJS_Controller]] / 인증 헤더 → [[NestJS_JwtGuard]]

---

# HTTP란 ⭐️⭐️⭐️⭐️⭐️

```txt
HTTP (HyperText Transfer Protocol)
  = 웹에서 클라이언트와 서버가 데이터를 주고받는 규약

특징:
  Stateless (무상태):
    각 요청은 독립적 — 서버는 이전 요청을 기억하지 않음
    → 인증 정보를 매 요청마다 헤더에 담아야 하는 이유

  Request-Response 모델:
    반드시 요청이 있어야 응답이 있음
    한 요청에 하나의 응답 (HTTP/1.1 기준)

  텍스트 기반 → 바이너리:
    HTTP/1.x = 사람이 읽을 수 있는 텍스트 형식
    HTTP/2   = 바이너리 프레임으로 전송 (더 빠름)

계층:
  OSI 7계층 — 7계층(응용 계층)에 위치
  TCP/IP 위에서 동작 (HTTP/3는 UDP 위 QUIC)
```

---

# URL 구조 ⭐️⭐️⭐️⭐️

```
https://api.example.com:8080/users/123?page=1&limit=10#top
└─┬──┘  └──────┬───────┘└─┬┘└───┬───┘└──────┬───────┘└─┬┘
scheme        host       port  path       query string  fragment
(프로토콜)   (도메인)   (포트) (경로)    (쿼리 파라미터) (앵커)
```

```txt
scheme:    http / https / ws / wss
host:      도메인 또는 IP 주소
port:      생략 시 기본값 → http=80, https=443
path:      리소스 위치 (/users/123)
query:     ?key=value&key2=value2 형태, 검색·필터·페이지에 사용
fragment:  # 이후, 서버로 전송 안 됨 — 브라우저 내부 스크롤 앵커

경로 파라미터 vs 쿼리 파라미터:
  /users/:id       → 특정 리소스 식별 (필수값)
  /users?page=1    → 필터·정렬·페이지 (선택값)
```

---

# HTTP 버전 ⭐️⭐️⭐️⭐️

| 버전 | 연도 | 핵심 특징 |
|------|------|-----------|
| HTTP/1.0 | 1996 | 요청마다 TCP 연결 새로 맺음 → 느림 |
| HTTP/1.1 | 1997 | **Keep-Alive** 연결 재사용, 파이프라이닝 |
| HTTP/2   | 2015 | **멀티플렉싱**, 헤더 압축(HPACK), 서버 푸시 |
| HTTP/3   | 2022 | **QUIC(UDP 기반)**, 0-RTT 연결, 패킷 손실 독립 |

```txt
HTTP/1.1 Keep-Alive:
  한 번 맺은 TCP 연결을 여러 요청에 재사용
  기본 활성화 (Connection: keep-alive)
  HOL(Head-of-Line) Blocking — 앞 요청이 막히면 뒤도 대기

HTTP/2 멀티플렉싱:
  하나의 TCP 연결로 여러 요청/응답을 동시에 → HOL Blocking 해결
  헤더 압축 (반복 헤더 오버헤드 감소)
  서버 푸시 (클라이언트 요청 없이 리소스 미리 전송)

HTTP/3 (QUIC):
  UDP 기반 → TCP 3-way handshake 없음 → 연결 빠름
  연결 손실 시 다른 스트림에 영향 없음
  모바일 (IP 변경 환경)에 특히 강함
```

---

# 요청(Request) 구조 ⭐️⭐️⭐️⭐️⭐️

```http
POST /posts HTTP/1.1
Host: api.example.com
Authorization: Bearer eyJhbGci...
Content-Type: application/json
Content-Length: 42

{"title": "새 게시글", "content": "내용"}
```

```txt
구성요소:
  1. 시작줄:  METHOD  PATH  HTTP-VERSION
  2. 헤더:    key: value 형태, 빈 줄로 바디와 구분
  3. 바디:    실제 전송 데이터 (POST/PATCH/PUT에서 사용)
              GET/DELETE는 바디 없음 (관례 + 일부 서버 미지원)
```

---

# HTTP 메서드 ⭐️⭐️⭐️⭐️⭐️

| 메서드 | 의미 | Body | 멱등성 | 안전성 | 성공 코드 |
|--------|------|:----:|:------:|:------:|:---------:|
| `GET`    | 조회       | ✗ | ✅ | ✅ | 200 |
| `POST`   | 생성       | ✅ | ❌ | ❌ | 201 |
| `PUT`    | 전체 교체  | ✅ | ✅ | ❌ | 200 |
| `PATCH`  | 부분 수정  | ✅ | △  | ❌ | 200 |
| `DELETE` | 삭제       | ✗ | ✅ | ❌ | 204 |
| `HEAD`   | 헤더만 조회| ✗ | ✅ | ✅ | 200 |
| `OPTIONS`| 허용 메서드 확인| ✗ | ✅ | ✅ | 200 |

```txt
안전성(Safe): 서버 상태를 변경하지 않음 → GET, HEAD, OPTIONS만 해당
멱등성(Idempotent): 같은 요청 N번 = 1번과 결과 동일

PUT vs PATCH:
  PUT   → 리소스 전체 교체 (보내지 않은 필드 = null/기본값)
  PATCH → 변경할 필드만 전송 (보내지 않은 필드 = 그대로 유지)
  → 실무에서는 PATCH가 안전해서 거의 PATCH 사용

PATCH 멱등성이 △인 이유:
  PATCH /count { "increment": 1 } → 호출할 때마다 다른 결과 → 비멱등
  PATCH /user  { "name": "kim" }  → 몇 번 해도 name=kim → 멱등
  → 바디 내용에 따라 다름, 일반적으로 비멱등 취급
```

## 멱등성 — 실무 중요성 ⭐️⭐️⭐️⭐️

```txt
멱등성이 중요한 이유:
  네트워크 오류 시 클라이언트가 자동으로 요청을 재시도함
  → GET, DELETE: 재시도해도 결과 동일 → 안전
  → POST: 재시도 시 중복 생성 위험 (주문, 결제, 회원가입)

POST 중복 방지 방법:
  1. Idempotency-Key 헤더:
     클라이언트가 UUID를 헤더에 포함
     서버가 동일 키 요청이 오면 캐시된 응답 반환

  2. DB Upsert:
     "없으면 생성, 있으면 무시"
     → [[NestJS_Prisma]] upsert 패턴

  3. unique constraint + P2002 처리:
     중복 INSERT 에러를 던지지 않고 성공으로 처리
```

---

# REST API CRUD 설계 ⭐️⭐️⭐️⭐️

```txt
REST = HTTP 메서드 + 경로로 "무엇을 어떻게" 표현하는 관례
리소스 = 명사(복수) + HTTP 메서드 조합
```

| HTTP | 경로 | 의미 | NestJS 서비스 |
|------|------|------|---------------|
| `POST`   | `/users`       | 생성      | `create()`  |
| `GET`    | `/users`       | 목록 조회 | `findAll()` |
| `GET`    | `/users/:id`   | 단건 조회 | `findOne()` |
| `PATCH`  | `/users/:id`   | 부분 수정 | `update()`  |
| `DELETE` | `/users/:id`   | 삭제      | `remove()`  |

```txt
중첩 리소스 예시:
  GET  /users/:userId/posts       → 특정 유저의 게시글 목록
  GET  /users/:userId/posts/:id   → 특정 유저의 특정 게시글
  POST /users/:userId/posts       → 특정 유저 게시글 생성

→ [[NestJS_Controller]] 데코레이터로 매핑
```

---

# 상태 코드 ⭐️⭐️⭐️⭐️⭐️

## 2xx — 성공

| 코드 | 의미 | 사용 상황 |
|------|------|-----------|
| `200 OK` | 성공 | GET, PATCH, PUT 성공 |
| `201 Created` | 리소스 생성됨 | POST 성공 (Location 헤더에 새 URL 포함 권장) |
| `202 Accepted` | 수락됨, 처리 중 | 비동기 작업 시작 → [[NestJS_AsyncJob]] |
| `204 No Content` | 성공, 바디 없음 | DELETE 성공 |

## 3xx — 리다이렉트

| 코드 | 의미 | 메서드 유지 | 캐시 |
|------|------|:-----------:|:----:|
| `301 Moved Permanently` | 영구 이동 | ❌ (GET으로 변경) | ✅ |
| `302 Found` | 임시 이동 | ❌ (GET으로 변경) | ❌ |
| `307 Temporary Redirect` | 임시 이동 | ✅ | ❌ |
| `308 Permanent Redirect` | 영구 이동 | ✅ | ✅ |

```txt
301/302 vs 307/308:
  301/302 → 브라우저가 메서드를 GET으로 바꿀 수 있음
  307/308 → 원래 메서드 그대로 유지 (POST → POST 리다이렉트)
  → POST 폼 제출 후 리다이렉트 시 307 사용
```

## 4xx — 클라이언트 오류

| 코드 | 의미 | 사용 상황 |
|------|------|-----------|
| `400 Bad Request` | 잘못된 요청 | 유효성 검사 실패, 파라미터 누락 |
| `401 Unauthorized` | 인증 없음 | 토큰 없음 또는 만료 |
| `403 Forbidden` | 권한 없음 | 인증됐지만 접근 권한 없음 |
| `404 Not Found` | 리소스 없음 | 해당 ID 데이터 없음 |
| `409 Conflict` | 충돌 | 중복 데이터 (이메일 중복 등) |
| `422 Unprocessable` | 처리 불가 | 형식은 맞지만 의미상 오류 |
| `429 Too Many Requests` | 요청 과다 | Rate Limiting |

```txt
401 vs 403 구분 (자주 헷갈림):
  401 Unauthorized → "누구세요?" (인증 자체가 없음)
  403 Forbidden    → "알지만 안 됩니다" (인증됐지만 권한 없음)

  예시:
    로그인 안 한 상태에서 API 호출 → 401
    로그인 했지만 남의 게시글 삭제 시도 → 403
```

## 5xx — 서버 오류

| 코드                          | 의미         | 사용 상황             |
| --------------------------- | ---------- | ----------------- |
| `500 Internal Server Error` | 서버 내부 오류   | 예상치 못한 예외, 코드 버그  |
| `502 Bad Gateway`           | 게이트웨이 오류   | 프록시가 업스트림 응답 못 받음 |
| `503 Service Unavailable`   | 서비스 불가     | 서버 과부하, 점검 중      |
| `504 Gateway Timeout`       | 게이트웨이 타임아웃 | 업스트림 응답 시간 초과     |

```txt
502 Bad Gateway 실전:
  Railway Edge Proxy 같은 중간 프록시가
  백엔드 서버 응답을 일정 시간 내에 못 받으면 502 반환
  → 오래 걸리는 작업은 202 Accepted + 폴링 패턴으로 해결
  → [[NestJS_AsyncJob]]
```

---

# 헤더(Headers) 완전 정리 ⭐️⭐️⭐️⭐️

```txt
헤더 = 요청/응답의 메타 정보
  key: value 형태
  대소문자 구분 없음 (소문자 관행)
  여러 개 가능
```

## 주요 요청 헤더

| 헤더 | 의미 | 예시 |
|------|------|------|
| `Authorization` | 인증 토큰 | `Bearer eyJ...` |
| `Content-Type` | 바디 형식 | `application/json` |
| `Accept` | 원하는 응답 형식 | `application/json` |
| `Cookie` | 쿠키 전달 | `refreshToken=abc` |
| `Origin` | 요청 출처 (CORS) | `https://myapp.com` |
| `User-Agent` | 클라이언트 환경 | `Mozilla/5.0...` |
| `Referer` | 이전 페이지 URL | `https://google.com` |
| `Cache-Control` | 캐시 제어 | `no-cache` |
| `If-None-Match` | ETag 조건 요청 | `"abc123"` |

## 주요 응답 헤더

| 헤더 | 의미 | 예시 |
|------|------|------|
| `Content-Type` | 바디 형식 | `application/json` |
| `Set-Cookie` | 쿠키 설정 | `token=abc; HttpOnly` |
| `Location` | 리다이렉트 URL | `/users/123` |
| `Cache-Control` | 캐시 정책 | `max-age=3600` |
| `ETag` | 리소스 버전 식별자 | `"abc123"` |
| `Access-Control-Allow-Origin` | CORS 허용 출처 | `*` |

## Content-Type 종류

| Content-Type | 용도 |
|---|---|
| `application/json` | JSON 데이터 (API 기본) |
| `multipart/form-data` | 파일 업로드 (텍스트 + 파일 동시) |
| `application/x-www-form-urlencoded` | HTML form 기본값 |
| `text/html` | HTML 문서 |
| `application/octet-stream` | 바이너리 파일 다운로드 |

## Authorization 헤더

```txt
형식: Authorization: <scheme> <credentials>

Bearer 토큰 (JWT):
  Authorization: Bearer eyJhbGciOiJIUzI1NiIsInR5cCI6IkpXVCJ9...
  "Bearer " + 토큰 (띄어쓰기 포함 필수)

Basic 인증:
  Authorization: Basic dXNlcjpwYXNz
  = base64(username:password) → 보안 약함, HTTPS 필수

API Key (커스텀 헤더):
  X-API-Key: my-secret-key
  X-Cron-Secret: my-cron-secret
```

```typescript
// fetch에서 Authorization 헤더 추가
fetch('/api/posts', {
  headers: {
    Authorization: `Bearer ${accessToken}`,
    'Content-Type': 'application/json',
  },
});
```

---

# 쿠키(Cookie) 완전 정리 ⭐️⭐️⭐️⭐️

```txt
쿠키 = 서버가 브라우저에 저장 지시하는 작은 데이터
  서버   → Set-Cookie 헤더로 설정
  브라우저 → 이후 요청마다 Cookie 헤더로 자동 전송
  용도:  세션 관리, Refresh Token, 사용자 설정
```

## Set-Cookie 속성

```http
Set-Cookie: refreshToken=eyJ...; HttpOnly; Secure; SameSite=Strict; Path=/; Max-Age=604800
```

| 속성 | 의미 |
|------|------|
| `HttpOnly` | JS에서 `document.cookie` 접근 불가 → XSS 방어 |
| `Secure` | HTTPS에서만 전송 |
| `SameSite=Strict` | 동일 사이트 요청에만 전송 → CSRF 방어 |
| `SameSite=Lax` | 기본값. 링크 클릭 등 안전한 크로스사이트는 허용 |
| `SameSite=None` | 크로스사이트 허용 → `Secure` 필수 |
| `Domain` | 쿠키 적용 도메인 (`.example.com` = 서브도메인 포함) |
| `Path` | 쿠키 적용 경로 (`/` = 전체 경로) |
| `Max-Age` | 유효기간 (초 단위, 604800 = 7일) |
| `Expires` | 만료 날짜 (GMT 형식) |

```txt
HttpOnly가 중요한 이유:
  XSS 공격 = 악성 JS가 document.cookie를 읽어 토큰 탈취
  HttpOnly 설정 시 → JS 접근 자체가 불가능 → 탈취 불가

SameSite가 중요한 이유:
  CSRF 공격 = 다른 사이트에서 사용자 대신 요청 전송
  SameSite=Strict → 다른 사이트 요청에 쿠키 미포함 → CSRF 방어
```

## 쿠키 vs 웹 스토리지

| | Cookie | localStorage | sessionStorage |
|--|--------|:------------:|:--------------:|
| 서버 자동 전송 | ✅ | ❌ | ❌ |
| JS 접근 | HttpOnly 시 ❌ | ✅ | ✅ |
| 유효기간 | Max-Age 설정 | 영구 | 탭 닫으면 삭제 |
| 용량 | ~4KB | ~5MB | ~5MB |
| 주 용도 | Refresh Token | Access Token (임시) | 임시 상태 |

---

# CORS ⭐️⭐️⭐️⭐️

```txt
CORS (Cross-Origin Resource Sharing)
= 다른 Origin의 리소스 접근을 제어하는 브라우저 보안 정책

Origin = scheme + host + port
  https://app.com    ↔  https://api.com    → 다른 Origin (host 다름)
  http://app.com     ↔  https://app.com    → 다른 Origin (scheme 다름)
  https://app.com    ↔  https://app.com:3000 → 다른 Origin (port 다름)

주의: CORS는 브라우저 정책 — curl, Postman은 CORS 제한 없음
```

## Preflight 요청 흐름

```mermaid
sequenceDiagram
    participant B as Browser
    participant S as API Server
    B->>S: OPTIONS /api/posts (Preflight)
    Note over B,S: Origin, Access-Control-Request-Method 헤더 포함
    S->>B: 200 OK
    Note over B,S: Access-Control-Allow-Origin, Allow-Methods 응답
    B->>S: POST /api/posts (실제 요청)
    S->>B: 201 Created
```

```txt
Preflight 발생 조건:
  - PUT, DELETE, PATCH 메서드
  - Content-Type: application/json (커스텀 헤더)
  - Authorization 헤더 포함

단순 요청 (Preflight 없음):
  - GET, POST, HEAD
  - Content-Type: text/plain, multipart/form-data, application/x-www-form-urlencoded
```

## CORS 관련 헤더

| 헤더 | 방향 | 의미 |
|------|:----:|------|
| `Origin` | 요청 | 요청 출처 |
| `Access-Control-Request-Method` | 요청(Preflight) | 실제로 쓸 메서드 |
| `Access-Control-Allow-Origin` | 응답 | 허용할 출처 (`*` 또는 특정 URL) |
| `Access-Control-Allow-Methods` | 응답 | 허용할 메서드 |
| `Access-Control-Allow-Headers` | 응답 | 허용할 헤더 |
| `Access-Control-Allow-Credentials` | 응답 | 쿠키 포함 요청 허용 여부 |
| `Access-Control-Max-Age` | 응답 | Preflight 캐시 시간(초) |

```typescript
// NestJS CORS 설정
app.enableCors({
  origin: ['https://myapp.com', 'http://localhost:3000'],
  methods: ['GET', 'POST', 'PATCH', 'DELETE', 'OPTIONS'],
  allowedHeaders: ['Authorization', 'Content-Type'],
  credentials: true, // 쿠키 포함 요청 허용
});

// credentials: true이면 origin을 '*'으로 설정 불가
// → 명시적 도메인 목록 필요
```

---

# 캐시(Cache) ⭐️⭐️⭐️⭐️

## Cache-Control 값

| 값 | 의미 |
|----|------|
| `no-store` | 캐시 금지 (민감한 데이터) |
| `no-cache` | 캐시하지만 매번 서버에 유효성 확인 |
| `private` | 브라우저에만 캐시 (CDN 불가) |
| `public` | 브라우저 + CDN 모두 캐시 가능 |
| `max-age=3600` | 3600초(1시간) 동안 캐시 유효 |
| `s-maxage=86400` | CDN 캐시 유효 시간 (max-age 무시) |
| `must-revalidate` | 만료 후 반드시 서버 확인 |
| `immutable` | 절대 변하지 않음 (JS/CSS 빌드 해시 파일) |

```txt
일반적인 캐시 전략:
  HTML 페이지:     Cache-Control: no-cache (항상 최신 확인)
  API 응답:        Cache-Control: no-store (민감) or max-age=60
  정적 파일(JS/CSS): Cache-Control: public, max-age=31536000, immutable
```

## ETag — 조건부 요청

```txt
ETag = 리소스 버전 식별자 (해시값)
  서버: 응답에 ETag 헤더 포함
  클라이언트: 다음 요청에 If-None-Match 헤더로 ETag 전송
  → 변경 없으면 304 Not Modified (Body 없음) → 대역폭 절약
```

```http
# 1. 첫 번째 요청
GET /posts/1

# 2. 첫 번째 응답
200 OK
ETag: "abc123"
Cache-Control: max-age=60
{"id": 1, "title": "게시글"}

# 3. max-age 만료 후 재요청
GET /posts/1
If-None-Match: "abc123"

# 4a. 변경 없을 때
304 Not Modified   ← Body 없음, 캐시 그대로 사용

# 4b. 변경됐을 때
200 OK
ETag: "xyz789"
{"id": 1, "title": "수정된 게시글"}
```

---

# HTTPS / TLS ⭐️⭐️⭐️⭐️

```txt
HTTP  = 평문 전송 → 패킷 캡처 시 내용 그대로 노출
HTTPS = HTTP + TLS (Transport Layer Security) 암호화

TLS가 보장하는 것:
  기밀성: 제3자가 내용을 볼 수 없음
  무결성: 전송 중 데이터 변조 감지
  인증:   실제 해당 서버와 통신 중임을 증명 (인증서)
```

## TLS 핸드셰이크 (요약)

```mermaid
sequenceDiagram
    participant C as Client
    participant S as Server
    C->>S: ClientHello (지원하는 암호화 방식 목록)
    S->>C: ServerHello (선택한 방식 + 인증서)
    C->>S: 인증서 검증 + 세션 키 교환
    S->>C: 핸드셰이크 완료
    C->>S: 암호화된 HTTP 요청
    S->>C: 암호화된 HTTP 응답
```

```txt
실무:
  Vercel, Railway 등 → HTTPS 기본 제공 (Let's Encrypt 인증서 자동 갱신)
  운영 환경에서 HTTP 절대 사용 금지
  Authorization: Bearer 토큰 → HTTPS 없으면 중간자 탈취 가능
```

---

# 요청-응답 전체 흐름 ⭐️⭐️⭐️⭐️

```mermaid
sequenceDiagram
    participant Browser
    participant DNS
    participant Server

    Browser->>DNS: api.example.com IP 주소?
    DNS->>Browser: 203.0.113.1
    Browser->>Server: TCP 3-way Handshake (SYN → SYN-ACK → ACK)
    Browser->>Server: TLS Handshake
    Browser->>Server: HTTP 요청 (암호화)
    Server->>Browser: HTTP 응답 (암호화)
```

```txt
TCP 3-way Handshake:
  1. SYN:     클라이언트 → 서버 "연결 요청"
  2. SYN-ACK: 서버 → 클라이언트 "수락 + 나도 연결 요청"
  3. ACK:     클라이언트 → 서버 "확인"
  → 연결 수립 완료 후 TLS → HTTP 데이터 전송

HTTP/2, HTTP/3는 핸드셰이크 RTT를 줄이는 방향으로 발전
```

## 예시 — 로그인 + 인증 흐름

```http
# 1. 로그인 요청
POST /auth/login HTTP/1.1
Content-Type: application/json

{"email": "user@example.com", "password": "1234"}

---

# 2. 로그인 응답
HTTP/1.1 200 OK
Content-Type: application/json
Set-Cookie: refreshToken=eyJ...; HttpOnly; Secure; SameSite=Strict; Path=/; Max-Age=604800

{"accessToken": "eyJhbGci..."}

---

# 3. 인증 필요한 API 요청
GET /posts HTTP/1.1
Authorization: Bearer eyJhbGci...

---

# 4. API 응답
HTTP/1.1 200 OK
Content-Type: application/json

[{"id": "1", "title": "게시글"}]
```

---

# NestJS — 헤더 접근 ⭐️⭐️⭐️

```typescript
// 특정 헤더만 추출
@Get()
findAll(@Headers('authorization') auth: string) {
  // "Bearer eyJ..."
}

// 전체 요청 객체 (헤더 + 쿠키)
@Get()
findAll(@Req() req: Request) {
  req.headers.authorization;       // "Bearer eyJ..."
  req.headers['content-type'];     // "application/json"
  req.cookies.refreshToken;        // 쿠키 (cookieParser 미들웨어 필요)
}

// 커스텀 헤더 (예: cron 인증)
@Post('cron')
cronJob(@Headers('x-cron-secret') secret: string) {
  if (secret !== process.env.CRON_SECRET) throw new UnauthorizedException();
  // ...
}
```

```txt
실전 패턴:
  Authorization  → JwtAuthGuard 자동 처리 → @UserId()로 userId 추출
  Content-Type   → NestJS + ValidationPipe 자동 처리
  Cookie         → req.cookies (cookie-parser 미들웨어 필요)
  커스텀 헤더    → @Headers('x-custom-key') 직접 읽기
```

---

# curl — HTTP 직접 테스트 ⭐️⭐️⭐️

```bash
# GET
curl -s http://localhost:3000/users

# POST (JSON 바디)
curl -s -X POST http://localhost:3000/users \
  -H 'Content-Type: application/json' \
  -d '{"email":"a@b.com"}'

# PATCH (인증 포함)
curl -s -X PATCH http://localhost:3000/users/123 \
  -H 'Content-Type: application/json' \
  -H 'Authorization: Bearer eyJ...' \
  -d '{"name":"kim"}'

# DELETE
curl -s -X DELETE http://localhost:3000/users/123

# 응답 헤더 확인
curl -i http://localhost:3000/users   # -i: 헤더 포함 출력
curl -v http://localhost:3000/users   # -v: 전체 요청/응답 출력 (디버깅)
```

| 옵션 | 의미 |
|------|------|
| `-s` | silent (진행 표시 숨김) |
| `-X METHOD` | HTTP 메서드 지정 (기본: GET) |
| `-H 'key: val'` | 헤더 추가 |
| `-d 'body'` | 요청 바디 |
| `-i` | 응답 헤더 출력 |
| `-v` | 전체 요청/응답 상세 출력 |
| `-o file` | 응답을 파일로 저장 |
