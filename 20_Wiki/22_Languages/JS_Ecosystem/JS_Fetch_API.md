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
  - "[[JS_Promise]]"
  - "[[TS_TypeAssertion]]"
  - "[[NestJS_CORS]]"
  - "[[NextJS_TokenStorage]]"
---
# JS_Fetch_API — fetch & HTTP 요청

> [!info] 
> `fetch`는 URL로 HTTP 요청을 보내는 JavaScript 내장 함수.
>  실전에서는 매번 직접 쓰지 않고, 공통 옵션·에러 처리를 감싼 래퍼 함수 하나로 만들어 앱 전체에서 재사용
>   이 노트 순서: fetch 자체 이해 → 왜/어떻게 래퍼로 감싸는지

---

# fetch 등장 배경

|시대|방식|
|---|---|
|예전 브라우저|`XMLHttpRequest`(XHR) — 콜백 기반, 코드 복잡|
|fetch 등장|Promise 기반으로 재설계, 브라우저 내장|
|Node.js|18+ 부터 전역 내장 (설치 불필요) / 17 이하는 `node-fetch` 설치 필요|

---

# 기본 사용법 ⭐️⭐️

```javascript
const res  = await fetch(url, options);  // Response 객체 반환 (body 아님)
const data = await res.json();           // body를 JS 객체로 파싱
```

```txt
fetch가 반환하는 건 "응답 자체"일 뿐
body를 꺼내려면 res.json() / res.text() 를 따로 호출해야 함

두 줄이 필요한 이유:
  1. fetch()    → 헤더 수신 완료 (body는 아직 스트리밍 중)
  2. res.json() → body를 모두 받아서 파싱

body는 스트림 — res.json()은 한 번만 호출 가능 ⚠️
```

---

# RequestInit — 요청 옵션 ⭐️⭐️⭐️

```javascript
fetch(url, {
  method:      'POST',
  headers:     { 'Content-Type': 'application/json', Authorization: 'Bearer token' },
  body:        JSON.stringify({ title: '제목' }),
  cache:       'no-store',
  credentials: 'include',
});
```

|옵션|주요 값|설명|
|---|---|---|
|`method`|`'GET'`(기본) / `'POST'` / `'PATCH'` / `'PUT'` / `'DELETE'`|HTTP 메서드|
|`headers`|`{ 'Content-Type': ..., Authorization: ... }`|요청 헤더|
|`body`|문자열 또는 `FormData`만 허용|요청 본문 (POST/PATCH)|
|`cache`|`'no-store'` (매번 새 요청) / `'force-cache'` (캐시 우선)|캐시 정책|
|`credentials`|`'same-origin'`(기본) / `'include'` / `'omit'`|쿠키 전송 여부|

## body — JSON.stringify가 왜 필요한가

```txt
body는 문자열/FormData만 허용 — 객체를 그대로 넣으면 "[object Object]"로 전송됨 ⚠️

  body: { title: '...' }                          ❌
  body: JSON.stringify({ title: '...' })          ✅ + Content-Type: application/json 세트로

Content-Type 없이 stringify만 하면 서버가 body 형식을 모름
stringify 없이 Content-Type만 선언하면 body가 문자열이 아님
→ 둘 다 같이 써야 함

GET / DELETE는 body 없음 → JSON.stringify 불필요
```

## credentials: 'include' — 언제 필요한가

|인증 방식|credentials|
|---|---|
|JWT (Authorization 헤더)|불필요 — 헤더에 직접 담음|
|쿠키 기반, same-origin|기본값으로 충분|
|쿠키 기반, cross-origin|`'include'` 필수|

```txt
cross-origin = 프론트(localhost:3001)와 백엔드(localhost:3000)처럼 포트/도메인이 다른 경우
  → 기본값(same-origin)이면 브라우저가 쿠키를 안 보냄

⚠️ credentials: 'include' 사용 시 서버 CORS 설정도 같이 필요:
  credentials: true + origin을 구체적 주소로 (와일드카드 * 불가)
  → [[NestJS_CORS]] 참고
```

---

# Response 객체 ⭐️⭐️

```javascript
res.status      // 200 / 404 / 500 등
res.ok          // 200~299면 true, 나머지 false
res.statusText  // 'OK' / 'Not Found' 등

await res.json()  // JSON → JS 객체
await res.text()  // 텍스트 / HTML / XML
await res.blob()  // 파일 / 이미지
```

## res.json()은 항상 any ⭐️⭐️

```typescript
const data = await res.json();               // TS 입장에서 any
const data = (await res.json()) as Movie[];  // "이 모양이라고 치자"는 단언
```

```txt
res.json()의 타입 정의 자체가 Promise<any> — 응답 모양을 TS가 알 길이 없음

⚠️ as는 검증이 아니라 약속일 뿐
  실제 응답이 그 모양이 아니어도 에러를 안 냄
  진짜 검증이 필요하면 zod 같은 스키마 검증 라이브러리 사용

as 단언 자체의 위험성 → [[TS_TypeAssertion]] 참고
```

---

# ⚠️ fetch는 4xx/5xx에서 throw하지 않음 ⭐️⭐️⭐️⭐️

```txt
fetch의 가장 헷갈리는 특성 — axios와의 핵심 차이

fetch가 reject(throw)하는 경우:   DNS 실패, 네트워크 끊김 등 "연결 자체"가 실패할 때만
fetch가 reject 안 하는 경우:      404 / 500 등 서버가 응답을 보낸 경우 → res.ok = false일 뿐

→ res.ok 체크를 안 하면 404 응답을 정상처럼 처리하는 버그 발생
```

```javascript
// ❌ res.ok 체크 없음 — 404여도 그냥 진행됨
const res  = await fetch('/api/item/999');
const data = await res.json();   // 404 에러 body가 그냥 파싱됨

// ✅ 항상 체크
const res = await fetch('/api/item/999');
if (!res.ok) throw new Error(`HTTP ${res.status}: ${res.statusText}`);
const data = await res.json();
```

```txt
axios를 쓰던 사람이 fetch로 옮길 때 가장 자주 생기는 버그
axios는 4xx/5xx에서 자동으로 throw → fetch는 직접 체크해야 함
```

---

# HTTP 메서드별 패턴 ⭐️⭐️

```javascript
// GET
const res = await fetch('/api/items');

// POST
const res = await fetch('/api/items', {
  method:  'POST',
  headers: { 'Content-Type': 'application/json' },
  body:    JSON.stringify({ title: '새 항목', content: '내용' }),
});

// PATCH
const res = await fetch(`/api/items/${id}`, {
  method:  'PATCH',
  headers: { 'Content-Type': 'application/json' },
  body:    JSON.stringify({ title: '수정된 제목' }),
});

// DELETE
const res = await fetch(`/api/items/${id}`, { method: 'DELETE' });
```

---

# 왜 래퍼 함수로 감싸는가 ⭐️⭐️⭐️

```txt
매번 반복되는 것:
  baseURL 조합 / credentials / Content-Type 헤더 / res.ok 체크 / res.json() 파싱

직접 다 쓰면:
  코드 중복 + 한 곳이라도 빠뜨리면 버그 (특히 res.ok)

래퍼로 모으면:
  호출하는 쪽은 endpoint와 옵션만 넘기면 됨
  baseURL이나 인증 방식이 바뀌어도 래퍼 한 곳만 수정
```

---

# `fetchAPI<T>` 래퍼 — 분해 ⭐️⭐️⭐️⭐️

```typescript
async function fetchAPI<T>(endpoint: string, options?: RequestInit): Promise<T> {
  let res: Response;
  try {
    res = await fetch(`${getApiBaseUrl()}${endpoint}`, {
      ...options,
      credentials: 'include',
      cache:       'no-store',
      headers:     { 'Content-Type': 'application/json', ...options?.headers },
    });
  } catch {
    throw new Error('네트워크 오류가 발생했습니다.');
  }

  if (!res.ok) {
    const body = await res.json().catch(() => ({}));
    throw new Error(body.message ?? `HTTP ${res.status}`);
  }
  return res.json();
}
```

## `<T> 제네릭` — 호출하는 쪽이 응답 타입을 지정

```typescript
const items = await fetchAPI<Item[]>('/items');     // items가 Item[] 타입
const item  = await fetchAPI<Item>('/items/1');     // 단건이면 Item
```

```txt
T는 함수가 "호출될 때" 결정되는 타입 변수
함수 하나로 어떤 응답이든 타입 안전하게 재사용할 수 있음
res.json()의 Promise<any>를 T로 좁히는 역할 — 본질적으로 as와 같은 약속 (검증 아님)
```

## try/catch vs res.ok — 책임 분리

```txt
try/catch:  "연결 자체" 실패 — DNS 오류, 네트워크 끊김 (fetch가 reject하는 경우)
res.ok:     "응답 내용" 실패 — 서버가 4xx/5xx 응답 (fetch가 resolve하는 경우)

let res: Response를 try 바깥에 선언하는 이유:
  try 안에서는 연결만 확인
  try가 끝난 뒤 res.ok로 응답 내용을 확인
  → 두 가지 실패를 명확히 분리해서 처리
```

## ⚠️ headers merge 순서 — 가장 찾기 어려운 버그 ⭐️⭐️⭐️

```typescript
// ❌ 이렇게 쓰면 Content-Type이 사라짐
fetch(url, {
  headers: { 'Content-Type': 'application/json', ...options?.headers },
  ...options,   // ← options.headers가 있으면 위 headers를 통째로 덮어씀
});
```

```txt
객체 스프레드 규칙: 같은 키가 여러 번 나오면 "나중에 오는 것"이 이김

options = { headers: { Authorization: 'Bearer xyz' } } 일 때:
  ① headers: { Content-Type: ..., Authorization: ... } 로 잘 합침
  ② ...options 가 펼쳐지며 options.headers 원본으로 교체됨
  ③ 최종 headers = { Authorization: 'Bearer xyz' } ← Content-Type 사라짐 ⚠️

headers를 안 넘길 때는 문제 없다가, 넘기는 순간에만 조용히 사라지는 버그라 발견이 어려움
```

```typescript
// ✅ ...options를 먼저 펼치고, headers는 맨 마지막에
fetch(url, {
  ...options,
  credentials: 'include',
  cache:       'no-store',
  headers:     { 'Content-Type': 'application/json', ...options?.headers },  // 항상 마지막
});
```

```txt
원칙: "또 합쳐야 하는" 중첩 속성(headers)은 객체 리터럴에서 항상 마지막에 둘 것
```

## 에러 메시지 추출

```typescript
const body = await res.json().catch(() => ({}));
throw new Error(body.message ?? `HTTP ${res.status}`);
```

```txt
res.json().catch(() => ({})):
  서버 응답이 JSON이 아닐 수도 있음 (HTML 에러 페이지 등)
  파싱 실패해도 빈 객체로 대체 → throw가 안 터짐

body.message ?? `HTTP ${res.status}`:
  서버가 { message: '이미 존재하는 이메일입니다.' } 같은 구조로 응답하면 그 메시지 사용
  없으면 상태 코드로 폴백
```

---

# 사용 예

```typescript
// GET
const items = await fetchAPI<Item[]>('/items');
const item  = await fetchAPI<Item>(`/items/${id}`);

// POST
const created = await fetchAPI<Item>('/items', {
  method: 'POST',
  body:   JSON.stringify({ title, content }),
});

// 커스텀 헤더 추가 (Content-Type은 래퍼가 자동으로 넣음)
const data = await fetchAPI<User>('/users/me', {
  headers: { 'X-Custom-Header': 'value' },
});
```

---

# 개별 함수 스타일 — 또 다른 패턴 ⭐️⭐️

```typescript
// 엔드포인트마다 함수를 하나씩 직접 작성하는 스타일
export async function fetchItemList(): Promise<Item[]> {
  const res = await fetch(`${API_URL}/items`, { cache: 'no-store' });
  if (!res.ok) throw new Error(`GET /items 실패: ${res.status}`);
  return (await res.json()) as Item[];
}
```

| |공통 래퍼 `fetchAPI<T>`|엔드포인트별 함수|
|---|---|---|
|공통 설정 (credentials/cache/headers)|한 곳에 모음|함수마다 직접 작성|
|새 엔드포인트 추가|호출 한 줄|함수 통째로 작성|
|엔드포인트별 특이 케이스|옵션으로 끼워 넣어야 함|함수 안에서 자유롭게|

```txt
선택 기준:
  엔드포인트가 많고 옵션이 거의 똑같음 → 공통 래퍼
  캐싱 전략이나 에러 처리가 엔드포인트마다 다름 → 개별 함수
  실제로는 공통 래퍼 + 특이 케이스만 개별 함수로 섞어 쓰는 경우가 많음
```

---

# fetch vs axios

| |fetch|axios|
|---|---|---|
|설치|불필요 (Node 18+, 브라우저 내장)|필요|
|4xx/5xx|`res.ok` 직접 체크 필수 ⚠️|자동 throw|
|JSON 파싱|`await res.json()` 직접 호출|자동|
|TS 제네릭|직접 구현|내장 지원|
|인터셉터|없음|있음|
|요청 취소|`AbortController`|`cancelToken` / `AbortController`|

```txt
간단한 API 호출 → fetch + 직접 만든 래퍼로 충분
인터셉터, 공통 에러 처리가 복잡해지면 → axios 고려
NestJS에서 외부 API 호출: Node 18+ 기준 fetch로 충분 (HttpModule 없이)
```

---

# 한눈에

```txt
fetch(url, options) → Response → res.json()

옵션:
  method: GET(기본) / POST / PATCH / DELETE
  headers: Content-Type + Authorization
  body: JSON.stringify(객체) — POST/PATCH에만, GET/DELETE는 불필요
  cache: 'no-store' (매번 새로) / 'force-cache' (캐시 우선)
  credentials: 'include' — cross-origin 쿠키 필요 시 (+ 서버 CORS 설정 필요)

⚠️ 4xx/5xx는 throw 안 함 → res.ok 체크 필수 (axios와의 핵심 차이)
⚠️ res.json()은 Promise<any> → as 또는 제네릭 <T>로 타입 약속 (검증 아님)
⚠️ headers처럼 "또 합쳐야 하는" 속성은 스프레드 중 항상 마지막에

래퍼 fetchAPI<T>:
  ...options 먼저, headers는 마지막 (Content-Type 사라짐 방지)
  try/catch → 연결 실패 / res.ok → 응답 실패 분리
  body.message ?? `HTTP ${res.status}` 에러 메시지 추출
```