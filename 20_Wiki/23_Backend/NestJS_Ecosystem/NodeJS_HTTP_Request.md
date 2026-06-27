---
aliases:
  - HTTP Headers
  - Authorization Header
  - request.headers
tags:
  - NodeJS
related:
  - "[[00_NestJS_Ecosystem_HomePage]]"
  - "[[JS_Destructuring]]"
  - "[[NestJS_Env_Config]]"
  - "[[NestJS_JwtGuard]]"
---
# NodeJS_HTTP_Request — HTTP 헤더와 요청 객체

> [!info] 
> HTTP 요청은 본문(body)과 별개로 헤더(header)라는 key-value 메타정보를 같이 들고 다닌다.
> 
>  `Authorization` 헤더는 그중 인증 정보를 싣는 자리이고, `"Bearer xxx"`처럼 "방식 + 값"을 공백 하나로 구분해서 담는 게 표준 형식이다. 
>  NestJS의 `context.switchToHttp().getRequest()`는 이 헤더를 포함한 Express(또는 Fastify)의 Request 객체 그 자체를 돌려주는 것뿐이다.

---

# HTTP 헤더란 무엇인가 ⭐️⭐️

```txt
HTTP 요청/응답은 항상 "헤더 + 본문(body)" 두 부분으로 옴
  본문    실제로 전달하려는 데이터 (JSON body, 업로드 파일 등)
  헤더    그 데이터를 어떻게 다뤄야 하는지에 대한 메타정보 — key: value 형태가 줄줄이 옴

예시 (실제 요청에 찍히는 raw 헤더):
  Authorization: Bearer eyJhbGciOiJIUzI1NiIs...
  Content-Type: application/json
  Cookie: sessionId=abc123
```

|자주 보는 헤더|뜻|
|---|---|
|`Authorization`|이 요청을 보낸 사람이 누구인지 증명하는 정보 (토큰, 인증 정보 등)|
|`Content-Type`|본문이 어떤 형식인지 (`application/json`, `multipart/form-data` 등)|
|`Cookie`|브라우저가 들고 있는 쿠키 값들|
|`Accept`|클라이언트가 어떤 응답 형식을 받고 싶은지|
|`User-Agent`|요청을 보낸 클라이언트(브라우저/앱)가 뭔지|

---

# Authorization 헤더 — 인증 정보를 싣는 자리 ⭐️⭐️⭐️

```txt
형식은 항상 같음:  Authorization: <Scheme> <Credentials>
                              ↑ 방식      ↑ 실제 인증 값
공백 하나로 "방식"과 "값"을 구분해서 적는 게 표준(RFC 7235/6750)임
```

|Scheme|값으로 들어가는 것|예시|
|---|---|---|
|`Bearer`|토큰 (보통 JWT)|`Bearer eyJhbGciOiJIUzI1NiIs...`|
|`Basic`|`username:password`를 base64로 인코딩한 값|`Basic dXNlcjpwYXNz`|
|`Digest`|비밀번호를 직접 안 보내고 해시로 증명하는 방식 (요즘은 잘 안 씀)|`Digest username="...", response="..."`|

```txt
"Bearer"라는 단어 자체의 의미: "이걸 들고 있는 사람(bearer)에게 권한을 준다"는 뜻
→ 누가 이 토큰을 들고 오든, 그 토큰이 유효하면 그 사람으로 인정한다는 OAuth 2.0 표준 방식
```

---

# 코드에서 토큰만 뽑아내기 — split + 배열 디스트럭처링 ⭐️⭐️⭐️

```typescript
const [type, token] = request.headers.authorization?.split(' ') ?? [];
```

```txt
한 줄씩 분해:

  1. request.headers.authorization
     → "Bearer eyJhbGc..." 형태의 문자열 (헤더가 없으면 undefined)

  2. ?.split(' ')
     → 옵셔널 체이닝(?.) : authorization이 undefined일 수도 있어서,
       undefined인 경우 에러 없이 그냥 undefined를 반환하게 함
     → split(' ') : 공백 하나로 문자열을 쪼개서 ['Bearer', 'eyJhbGc...'] 배열을 만듦

  3. ?? []
     → 위 단계가 undefined였다면(헤더 자체가 없었다면) 빈 배열로 대체
       (그래야 다음 단계의 배열 디스트럭처링이 에러 없이 동작함)

  4. const [type, token] = [...]
     → 배열의 0번째를 type, 1번째를 token이라는 이름으로 꺼냄
     → 헤더가 없었다면 [] 였으니 type/token 둘 다 undefined가 됨 (에러 아님)
```

```txt
배열 디스트럭처링 자체의 일반적인 동작 원리(왜 순서대로 이름이 붙는지 등)는
[[JS_Destructuring]] 참고 — 여기서는 "헤더 파싱에 어떻게 쓰이는지"에만 집중
```

---

# Node/Express의 Request 객체 — 그 안에 뭐가 들어있나 ⭐️⭐️

```txt
Express(혹은 NestJS가 기본으로 쓰는 Express)의 요청 객체 하나에는
헤더 말고도 요청에 관한 거의 모든 정보가 같이 들어있음
```

|속성|들어있는 것|
|---|---|
|`request.headers`|위에서 본 헤더들 (모두 소문자 key로 들어옴)|
|`request.body`|요청 본문 (JSON 등) — 별도 파서/미들웨어가 채워줌|
|`request.params`|라우트 경로의 동적 구간 (`/users/:id` → `{ id: '3' }`)|
|`request.query`|URL의 `?key=value` 쿼리스트링|
|`request.cookies`|쿠키 (cookie-parser 같은 미들웨어가 있어야 채워짐)|
|`request.method`|`'GET'`, `'POST'` 등|

---

# NestJS에서는 — ExecutionContext.switchToHttp() ⭐️⭐️⭐️

```txt
NestJS는 HTTP뿐 아니라 WebSocket, RPC(마이크로서비스), GraphQL 등
여러 "전송 방식(transport)"을 같은 Guard/Interceptor로 처리할 수 있게 만들어졌음
→ 그래서 ExecutionContext는 "지금 어떤 전송 방식으로 들어온 요청인지"를 추상화해두고,
  switchToHttp() / switchToWs() / switchToRpc() 로 원하는 형태로 "전환"해서 꺼내 쓰게 함
```

```typescript
context.switchToHttp().getRequest()   // 위에서 설명한 Express Request 객체 그 자체
context.switchToHttp().getResponse()  // Express Response 객체
```

```txt
즉 switchToHttp().getRequest()는 새로운 무언가를 만들어주는 게 아니라,
지금 들어온 요청이 HTTP라는 걸 전제로 "원래 Express가 주는 그 request 객체"를
그대로 돌려주는 것뿐임 — request.headers.authorization 같은 접근이 그래서 그대로 통함
```

---

# 한눈에

| 키워드                                 | 한 줄 정리                                          |
| ----------------------------------- | ----------------------------------------------- |
| HTTP 헤더                             | 본문과 별개로 같이 오는 key-value 메타정보                    |
| `Authorization: Scheme Credentials` | 공백으로 "방식"과 "값"을 구분하는 표준 형식                      |
| `Bearer`                            | 토큰을 들고 온 사람에게 그대로 권한을 주는 방식                     |
| `?.`                                | 앞 값이 undefined/null이면 에러 없이 undefined로 통과       |
| `?? []`                             | undefined였다면 빈 배열로 대체 — 디스트럭처링이 안전하게 동작하도록      |
| `request.headers/body/params/query` | Express Request 객체가 들고 있는 주요 정보들                |
| `switchToHttp().getRequest()`       | 새로 만드는 게 아니라, 원래 Express Request 객체를 그대로 돌려주는 것 |