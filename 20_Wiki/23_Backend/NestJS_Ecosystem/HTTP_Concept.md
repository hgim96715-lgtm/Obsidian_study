---
aliases:
  - HTTP
  - Request
  - Headers
  - Authorization
  - Content-Type
  - Cookie
  - Response
  - HTTPS
  - 멱등성(Idempotency)
tags:
  - NodeJS
  - NestJS
related:
  - "[[00_NestJS_Ecosystem_HomePage]]"
  - "[[NestJS_JwtGuard]]"
  - "[[NestJS_Controller]]"
  - "[[NestJS_Service_Provider]]"
---
# HTTP_Concept — HTTP 요청 · 응답 · 헤더

>[!info]
>HTTP = 클라이언트와 서버가 데이터를 주고받는 규약.
> 요청(Request)과 응답(Response)이 한 쌍. 헤더(Header)는 "본문 외에 전달하는 메타 정보" — 인증 토큰·콘텐츠 타입·쿠키 등.
>  REST CRUD 메서드 매핑 → [[NestJS_Controller]], 헤더 접근 → `@Req()` 또는 `@Headers()`, 인증 헤더 → [[NestJS_JwtGuard]]

---

# HTTP란 ⭐️⭐️⭐️⭐️

```txt
HTTP(HyperText Transfer Protocol):
  클라이언트(브라우저·앱)와 서버가 데이터를 주고받는 규약(약속)

  클라이언트 → 서버: 요청(Request)
  서버 → 클라이언트: 응답(Response)
  항상 이 쌍으로 이루어짐

비유:
  HTTP = 편지 규격
  요청 = 고객이 식당에 주문서 넣기
  응답 = 식당이 음식 가져다주기
  헤더 = 주문서의 부가 정보 (알레르기, 포장 여부 등)
```

---

# 요청(Request) 구조 ⭐️⭐️⭐️⭐️

```txt
요청 = Method + URL + Headers + Body

GET /posts?page=1&limit=20 HTTP/1.1
Host: api.example.com
Authorization: Bearer eyJhbGci...
Content-Type: application/json

{ "title": "새 게시글" }
↑ Body (GET은 보통 없음)
```

```txt
각 부분의 역할:

Method:  무엇을 하려는가 (GET=조회, POST=생성, PATCH=수정, DELETE=삭제)
URL:     어떤 리소스를 대상으로 하는가 (/posts, /users/123)
Headers: 요청에 대한 부가 정보 (누가 보내는지, 어떤 형식인지)
Body:    전달할 데이터 (POST, PATCH, PUT에서 사용)
```

---

# HTTP 메서드 ⭐️⭐️⭐️⭐️

|메서드|의미|Body|성공 상태코드|
|---|---|---|---|
|`GET`|조회|없음|200 OK|
|`POST`|생성|있음|201 Created|
|`PATCH`|일부 수정|있음|200 OK|
|`PUT`|전체 교체|있음|200 OK|
|`DELETE`|삭제|없음|204 No Content|

```txt
GET vs POST:
  GET  → URL에 데이터 포함 (?q=검색어) — 북마크 가능, 캐시 가능
  POST → Body에 데이터 — URL에 안 보임, 길이 제한 없음

PATCH vs PUT:
  PATCH → 변경할 필드만 보냄 (부분 수정)
  PUT   → 전체 데이터를 보냄 (전체 교체) — 거의 안 씀
```

## REST API CRUD — HTTP ↔ 경로 ↔ 서비스 메서드 매핑 ⭐️⭐️⭐️⭐️

```txt
REST API는 HTTP 메서드 + 경로로 "무엇을 어떻게 할지"를 표현하는 관례

리소스 예시: users (/users)
```

|HTTP|경로|의미|서비스 메서드|
|---|---|---|---|
|`POST`|`/users`|새 사용자 생성|`create`|
|`GET`|`/users`|전체 목록 조회|`findAll`|
|`GET`|`/users/:id`|특정 사용자 조회|`findOne`|
|`PATCH`|`/users/:id`|특정 사용자 수정|`update`|
|`DELETE`|`/users/:id`|특정 사용자 삭제|`remove`|

```txt
멱등성(Idempotency):
  같은 요청을 여러 번 보내도 결과가 같음

  GET /users/1   → 몇 번 해도 같은 사용자 정보 → 멱등
  DELETE /users/1 → 이미 삭제됐어도 "없음" 상태 동일 → 멱등
  POST /orders   → 매번 새 주문이 생김 → 멱등 아님

  실무에서 왜 중요한가:
  네트워크 오류로 클라이언트가 요청을 재시도할 때
  GET·DELETE는 재시도해도 안전
  POST는 재시도하면 중복 데이터 생성 위험
  → 결제·주문처럼 중요한 POST는 Idempotency-Key 헤더로 중복 방지
```

```txt
:id — 경로 파라미터:
  /users/abc-123  → id = 'abc-123'
  /users/456      → id = '456'

NestJS에서는 @Get(), @Post() 등 데코레이터로 매핑
→ [[NestJS_Controller]]
```

## curl — HTTP 요청 직접 보내기 ⭐️⭐️⭐️

```txt
curl = 터미널에서 HTTP 요청을 직접 보내는 도구
API가 제대로 동작하는지 테스트할 때 사용
```

```bash
# POST — 생성
curl -s -X POST http://localhost:3030/users \
  -H 'Content-Type: application/json' \
  -d '{"email":"a@b.com"}'

# GET — 전체 목록
curl -s http://localhost:3030/users

# GET — 단건 조회
curl -s http://localhost:3030/users/<uuid>

# PATCH — 수정
curl -s -X PATCH http://localhost:3030/users/<uuid> \
  -H 'Content-Type: application/json' \
  -d '{"email":"b@b.com"}'

# DELETE — 삭제
curl -s -X DELETE http://localhost:3030/users/<uuid>
```

```txt
curl 옵션:
  -s              → silent (진행 표시 숨김)
  -X POST         → HTTP 메서드 지정 (기본값은 GET)
  -H '헤더: 값'   → 헤더 추가 (Content-Type 등)
  -d '{"key":"value"}' → 요청 Body (JSON 문자열)

GET은 -X 없이 URL만 써도 됨:
  curl -s http://localhost:3030/users   ← GET 요청
```

---

# 헤더(Headers) — 핵심 ⭐️⭐️⭐️⭐️

```txt
헤더 = 요청/응답의 부가 정보
  실제 데이터(Body)와 별개로 전달하는 메타 정보
  "key: value" 형태
  여러 개 가능
```

## Authorization — 인증 토큰

```txt
Authorization: Bearer eyJhbGciOiJIUzI1NiIsInR5cCI6IkpXVCJ9...

JWT 토큰을 서버에 전달하는 헤더
"Bearer " + 토큰 (띄어쓰기 포함)

서버가 이 헤더를 보고:
  토큰 유효성 검증 → 누가 요청했는지 파악
  → [[NestJS_JwtGuard]] 에서 처리
```

```typescript
// Next.js에서 요청 시 Authorization 헤더 추가
fetch('/api/posts', {
  headers: {
    'Authorization': `Bearer ${accessToken}`,
    'Content-Type':  'application/json',
  },
});
```

## Content-Type — 데이터 형식

```txt
Content-Type: application/json
Content-Type: multipart/form-data; boundary=----FormBoundary
Content-Type: application/x-www-form-urlencoded

"Body에 담긴 데이터가 어떤 형식인지"를 서버에 알림

application/json:
  { "title": "게시글", "content": "내용" }
  → JSON 객체를 Body로 보낼 때 (가장 많이 씀)

multipart/form-data:
  파일 업로드 시 사용
  텍스트와 파일을 함께 보낼 수 있음
  → NestJS에서 @UploadedFile() 로 받음

application/x-www-form-urlencoded:
  HTML form 기본 형식
  title=게시글&content=내용 형태
  → 일반 API에서는 거의 안 씀
```

## Cookie

```txt
Cookie: sessionId=abc123; refreshToken=eyJ...

브라우저가 서버로 쿠키를 자동으로 전송
httpOnly 쿠키 = JS에서 접근 불가, 서버만 읽을 수 있음
→ Refresh Token 저장에 주로 사용

서버가 Set-Cookie 헤더로 쿠키를 설정:
  Set-Cookie: refreshToken=eyJ...; HttpOnly; Secure; SameSite=Strict
```

## 자주 쓰는 요청 헤더

|헤더|의미|예시|
|---|---|---|
|`Authorization`|인증 토큰|`Bearer eyJ...`|
|`Content-Type`|Body 데이터 형식|`application/json`|
|`Cookie`|쿠키 전달|`refreshToken=abc`|
|`Accept`|받고 싶은 응답 형식|`application/json`|
|`User-Agent`|클라이언트 환경 정보|`Mozilla/5.0...`|
|`Origin`|요청 출처 (CORS)|`https://myapp.com`|

---

# 응답(Response) 구조 ⭐️⭐️⭐️⭐️

```txt
응답 = 상태코드 + Headers + Body

HTTP/1.1 200 OK
Content-Type: application/json
Set-Cookie: refreshToken=eyJ...; HttpOnly

{ "id": "123", "title": "게시글 제목" }
↑ Body
```

## 자주 쓰는 응답 헤더

|헤더|의미|
|---|---|
|`Content-Type`|Body 데이터 형식|
|`Set-Cookie`|브라우저에 쿠키 저장 지시|
|`Location`|리다이렉트 주소 (301, 302 응답)|
|`Access-Control-Allow-Origin`|CORS 허용 출처|
|`Cache-Control`|캐시 정책|

---

# NestJS에서 헤더 접근 ⭐️⭐️⭐️

```typescript
// 방법 1 — @Headers() 데코레이터로 특정 헤더
@Get()
findAll(@Headers('authorization') auth: string) {
  console.log(auth);  // "Bearer eyJ..."
}

// 방법 2 — @Req()로 Express Request 전체 접근
@Get()
findAll(@Req() req: Request) {
  console.log(req.headers.authorization);  // "Bearer eyJ..."
  console.log(req.headers['content-type']); // "application/json"
  console.log(req.cookies.refreshToken);    // 쿠키
}

// 방법 3 — Guard에서 헤더 접근 (JwtAuthGuard)
// Authorization 헤더는 JwtAuthGuard가 자동으로 처리
// → 컨트롤러에서 @UserId()로 userId만 꺼내면 됨
```

```txt
실전에서:
  Authorization 헤더 → JwtAuthGuard가 처리, @UserId()로 사용 → [[NestJS_JwtGuard]]
  Content-Type 헤더  → NestJS + ValidationPipe가 자동 처리
  Cookie 헤더        → @Req().cookies 또는 @Cookies() 데코레이터
  직접 헤더를 건드릴 일은 많지 않음 — NestJS 레이어가 처리
```

---

# HTTPS — 암호화된 HTTP ⭐️⭐️⭐️

```txt
HTTP:  평문 전송 → 중간에서 패킷을 가로채면 내용이 보임
HTTPS: 암호화 전송 → 가로채도 암호화된 데이터만 보임

Authorization: Bearer 토큰을 HTTP로 보내면:
  → 중간자가 토큰 탈취 가능 → 보안 위험

HTTPS에서:
  → 암호화돼서 전송 → 토큰 안전

운영 환경에서는 반드시 HTTPS
  Vercel, Railway 등 배포 플랫폼은 기본으로 HTTPS 제공
```

---

# 요청-응답 전체 흐름 예시

```txt
로그인 요청:

클라이언트 →
  POST /auth/login HTTP/1.1
  Content-Type: application/json

  { "email": "user@example.com", "password": "1234" }

서버 →
  HTTP/1.1 200 OK
  Content-Type: application/json
  Set-Cookie: refreshToken=eyJ...; HttpOnly; Secure

  { "accessToken": "eyJhbGci..." }

---

인증 필요한 API 요청:

클라이언트 →
  GET /posts HTTP/1.1
  Authorization: Bearer eyJhbGci...

서버 →
  Authorization 헤더에서 토큰 추출
  → 서명 검증 → userId 확인
  HTTP/1.1 200 OK
  Content-Type: application/json

  [{ "id": "1", "title": "게시글" }, ...]
```