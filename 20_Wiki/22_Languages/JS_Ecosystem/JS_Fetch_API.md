---
aliases:
  - fetchAPI
  - HTTP
  - Fetch
  - Network
tags:
  - JavaScript
related:
  - "[[00_JS_Ecosystem_HomePage]]"
  - "[[NextJS_API_Client]]"
  - "[[NextJS_API_Mapper]]"
  - "[[NextJS_Types]]"
  - "[[NestJS_CORS]]"
---
# JS_Fetch_API — fetch

> [!info]
>  fetch = 브라우저 내장 HTTP 요청 API. 
>  `await fetch(url)`이 반환하는 것은 데이터가 아닌 Response 객체 — `await res.json()`으로 한 번 더 파싱해야 함
>  HTTP 4xx·5xx는 reject하지 않으므로 `res.ok`를 직접 확인해야 한다. 
>  실제 프로젝트에서는 apiFetch 래퍼를 만들어서 씀 → [[NextJS_API_Client]]

---

# fetch란 ⭐️⭐️⭐️⭐️

```txt
브라우저에 내장된 HTTP 요청 API
  별도 설치 없이 window.fetch() 로 사용 가능
  Node.js 18+ 에서도 기본 지원

용도:
  서버에 데이터를 요청하거나 (GET)
  데이터를 서버에 보내거나 (POST/PATCH/DELETE)
  파일을 업로드하거나 (FormData)
  외부 API를 호출할 때
```

---

# 두 단계로 이루어진 응답 ⭐️⭐️⭐️⭐️

```txt
fetch를 처음 쓸 때 가장 많이 헷갈리는 부분:
await fetch(url) 의 결과가 데이터가 아니라 "Response 객체"
```

```typescript
// ① 첫 번째 await — HTTP 헤더가 도착하면 완료
const res = await fetch('/api/user');
// res = Response 객체 (아직 데이터가 아님)
// res.status = 200, res.ok = true 같은 정보만 있음

// ② 두 번째 await — body(실제 데이터)를 파싱
const user = await res.json();
// 이제야 실제 데이터를 얻음
```

```txt
왜 두 단계인가:
  HTTP 응답은 헤더와 body가 따로 도착함
  헤더가 먼저 도착 → status 코드, Content-Type 등을 먼저 확인 가능
  body는 그 다음 스트리밍으로 도착 → 파싱까지 기다려야 함

  → 헤더만 보고 "파일이 너무 크면 body 파싱 안 하기" 같은 최적화 가능
  → 일반적으로는 그냥 두 줄로 이어서 씀
```

---

# ⚠️ fetch는 HTTP 에러에 throw 안 함 ⭐️⭐️⭐️⭐️

```txt
fetch를 쓸 때 가장 흔한 실수:
  "await fetch를 try/catch로 감쌌으니까 에러 처리 됐겠지" → ❌
```

```typescript
// ❌ 잘못된 에러 처리
try {
  const res  = await fetch('/api/user');
  const user = await res.json();
  // 404나 500이어도 여기까지 실행됨!
} catch (err) {
  // 네트워크 자체가 끊겼을 때만 여기로
}
```

```typescript
// ✅ 올바른 에러 처리 — res.ok 체크 필수
const res = await fetch('/api/user');

if (!res.ok) {
  // 4xx, 5xx → res.ok = false
  throw new Error(`HTTP 에러: ${res.status}`);
}

const user = await res.json();  // 여기서야 안전하게 파싱
```

```txt
fetch의 동작 방식:
  네트워크 자체 실패 (인터넷 끊김, CORS 에러) → Promise reject → catch로
  HTTP 응답 옴 (200, 404, 500 전부) → Promise resolve → catch 안 들어옴

res.ok:
  status 200~299 → true
  status 400, 404, 500 등 → false

  if (!res.ok) throw 패턴을 항상 써야 에러 응답을 잡을 수 있음
```

---

# Response 객체 ⭐️⭐️⭐️

```typescript
const res = await fetch('/api/user');

res.status      // HTTP 상태 코드: 200, 404, 500 ...
res.statusText  // 상태 메시지: "OK", "Not Found" ...
res.ok          // status 200~299 이면 true
res.headers     // 응답 헤더

// body 파싱 메서드 (한 번만 쓸 수 있음)
await res.json()        // JSON → JavaScript 객체
await res.text()        // 일반 텍스트
await res.blob()        // 이미지·파일 같은 바이너리
await res.arrayBuffer() // 바이너리를 ArrayBuffer로
```

```txt
body 파싱은 한 번만:
  res.json()을 한 번 호출하면 body 스트림이 소비됨
  다시 res.text()를 호출하면 빈 값 반환
  → 한 가지만 선택해서 한 번만 파싱

Content-Type 확인:
  res.headers.get('content-type')
  'application/json' → res.json()
  'text/html' → res.text()
  'image/...' → res.blob()
```

---

# 요청 옵션 ⭐️⭐️⭐️⭐️

```typescript
const res = await fetch(url, {
  method:  'POST',           // GET(기본) / POST / PATCH / PUT / DELETE
  headers: {
    'Content-Type': 'application/json',  // body 형식 알림
    'Authorization': `Bearer ${token}`,  // 인증 토큰
  },
  body: JSON.stringify({ name: '홍길동' }),  // 전송할 데이터
});
```

```txt
method 기본값은 GET — body 없는 요청은 method 생략 가능

headers:
  Content-Type: 'application/json'
    → "body가 JSON 형식이야"를 서버에 알림
    → 이 헤더가 없으면 서버가 body를 파싱하지 못할 수 있음

  Authorization: `Bearer ${token}`
    → JWT 인증 토큰 첨부

body:
  반드시 문자열이나 FormData 형태여야 함
  객체를 그냥 넣으면 "[object Object]" 문자열이 됨
  → JSON.stringify()로 반드시 변환
```

## credentials — 쿠키 포함 ⭐️⭐️⭐️

```typescript
const res = await fetch('/api/user', {
  credentials: 'include',   // cross-origin 요청에 쿠키 포함
});
```

```txt
credentials 옵션:
  'omit'        쿠키 안 보냄 (기본값)
  'same-origin' 같은 도메인일 때만 쿠키 포함
  'include'     cross-origin도 쿠키 포함

서버에서 Set-Cookie로 인증 토큰을 관리할 때 필요
서버 CORS 설정에 credentials: true 도 같이 필요 → [[NestJS_CORS]]
```

---

# 자주 쓰는 패턴

## GET 요청

```typescript
const res  = await fetch('/api/posts');
if (!res.ok) throw new Error(`${res.status}`);
const posts = await res.json();
```

## POST — JSON 전송

```typescript
const res = await fetch('/api/posts', {
  method:  'POST',
  headers: { 'Content-Type': 'application/json' },
  body:    JSON.stringify({ title: '제목', content: '내용' }),
});
if (!res.ok) throw new Error(`${res.status}`);
const post = await res.json();
```

## FormData — 파일 업로드

```typescript
const formData = new FormData();
formData.append('image', file);           // File 객체
formData.append('title', '이미지 제목');  // 텍스트도 같이

const res = await fetch('/api/upload', {
  method: 'POST',
  body:   formData,
  // Content-Type 헤더 직접 설정 금지
  // → 브라우저가 boundary 포함해서 자동 설정
});
```

```txt
FormData를 쓸 때 Content-Type을 직접 설정하면 안 되는 이유:
  multipart/form-data의 Content-Type은 boundary 값을 포함해야 함
  예: Content-Type: multipart/form-data; boundary=----abc123

  직접 'multipart/form-data'만 쓰면 boundary 없어서 서버가 파싱 실패
  → Content-Type 헤더를 아예 안 넣으면 브라우저가 올바르게 자동 설정
```

## DELETE — body 없음

```typescript
const res = await fetch(`/api/posts/${postId}`, {
  method: 'DELETE',
  // body 없음 → Content-Type 헤더도 불필요
});
if (!res.ok) throw new Error(`${res.status}`);
// 204 No Content면 res.json() 호출 불필요
```

---

# fetch vs apiFetch ⭐️⭐️⭐️

```txt
fetch를 직접 쓰면 매 요청마다 반복해야 하는 것들:
  ① Base URL 앞에 붙이기
  ② Authorization 헤더 추가
  ③ res.ok 체크
  ④ 에러 응답 body 파싱
  ⑤ JSON.stringify

→ apiFetch 래퍼를 만들어서 이 반복을 없앰
  프로젝트에서는 fetch를 직접 쓰지 않고 apiFetch만 씀
  apiFetch 구현 → [[NextJS_API_Client]]
```

```typescript
// 직접 fetch
const res = await fetch(`${process.env.NEXT_PUBLIC_API_URL}/users/me`, {
  headers: { Authorization: `Bearer ${token}` },
});
if (!res.ok) throw new Error(`${res.status}`);
const user = await res.json();

// apiFetch 래퍼 사용
const user = await apiFetch<ApiUser>('/users/me');
```