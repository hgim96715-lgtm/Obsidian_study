---
aliases:
  - api.ts
  - fetch 래퍼
  - fetchAPIVoid
  - fetchPosts
tags:
  - NextJS
related:
  - "[[00_JS_Ecosystem_HomePage]]"
  - "[[NextJS_API_Mapper]]"
  - "[[NextJS_Types]]"
  - "[[NextJS_Concept]]"
  - "[[JS_Fetch_API]]"
  - "[[NextJS_ServerClient]]"
---
# NextJS_API_Client — API 클라이언트 레이어

>[!info]
>Next.js에서 NestJS API를 호출할 때 `fetch`를 직접 쓰지 않고 래퍼로 감싼다. **패턴 A** — `fetchApi`(공개)·`authFetchApi`(Bearer 자동) 두 함수로 역할 분리. **패턴 B** — `apiFetch(path, { token? })` 하나로 통합. 둘 다 도메인 파일(`auth.ts`, `users.ts`)에 엔드포인트 함수를 모으고 화면은 함수만 호출한다.

---

# 왜 API 클라이언트 레이어가 필요한가 ⭐️⭐️⭐️⭐️

```txt
NestJS (API 서버):
  GET /posts, POST /auth/login, PATCH /users/:id ...
  요청을 받는 쪽

Next.js (클라이언트):
  브라우저가 fetch로 NestJS에 요청을 보내는 쪽
  → 이 fetch 호출을 어디에, 어떻게 쓸 것인가?

❌ 나쁜 방식 — page.tsx에 직접:
  const res = await fetch('http://localhost:3030/auth/login', {
    method: 'POST',
    headers: { 'Content-Type': 'application/json', Authorization: `Bearer ${token}` },
    body: JSON.stringify({ email, password }),
  });
  → 모든 페이지마다 URL·헤더·에러처리 반복
  → URL 바뀌면 전부 수정해야 함

✅ 좋은 방식 — 레이어 분리:
  화면 → 도메인 함수(auth.ts) → fetchApi/authFetch → NestJS
  화면은 login(email, password) 만 호출
  URL·헤더·에러처리는 fetchApi가 담당
```

---

# fetch — Web이 API를 부르는 방법 ⭐️⭐️⭐️⭐️

```typescript
// fetch 기본 구조
const res = await fetch(url, {
  method:  'POST',   // 없으면 GET
  headers: {
    'Content-Type': 'application/json',
    Authorization:  `Bearer ${token}`,  // 필요할 때만
  },
  body: JSON.stringify({ email, password }),  // GET에는 없음
});
```

```typescript
// RequestInit — fetch 두 번째 인자의 TypeScript 타입
fetch(url: string, options?: RequestInit)
//                           ↑ method·body·headers·signal 등을 담는 타입

// options?: RequestInit 을 파라미터로 받으면
// ...options 로 fetch에 그대로 투과 가능
```

|조각|의미|
|---|---|
|`fetch(...)`|Promise 반환 — 네트워크 끝날 때까지 기다림|
|`await`|Promise 결과를 받을 때까지 멈춤|
|`res.ok`|상태코드 200~299이면 true. 401·500은 false|
|`await res.json()`|응답 body를 객체로 파싱 — 이것도 Promise|
|`await res.text()`|JSON이 아닐 때 / 에러 메시지 문자열|

```txt
fetch가 throw하는 경우:
  네트워크 끊김, CORS 차단 → catch로 잡힘
  401·403·500 → throw 안 함 → if (!res.ok) 체크 필수

await를 두 번 쓰는 이유:
  1) await fetch  → Response 헤더·상태까지 도착
  2) await res.json() → body 읽기·파싱 (별도 비동기)

Bearer 형식:
  Authorization: `Bearer ${accessToken}`
  → JwtAuthGuard의 extractBearerToken과 짝
```

---

# 두 가지 구현 패턴 ⭐️⭐️⭐️⭐️

```txt
패턴 A — fetchApi + authFetchApi 분리:
  공개 API → fetchApi()      (Bearer 없음)
  인증 API → authFetchApi()  (Bearer 자동 첨부)
  명확한 역할 분리, Bearer 필요 여부를 함수 이름으로 구분

패턴 B — apiFetch + token? 통합:
  모든 API → apiFetch(path, { token? })
  token 있으면 Bearer 추가, 없으면 생략
  함수 하나로 통일, 호출 시 token을 직접 제어
```

---

# 패턴 A — fetchApi + authFetchApi 분리

## lib/api 폴더 구조

```txt
apps/web/lib/
  authToken.ts          토큰 get/set (메모리)
  api/
    fetchApi.ts         base URL · !ok throw · JSON (Bearer 없음)
    authFetch.ts        Authorization Bearer + 같은 에러 처리
    auth.ts             login · register · fetchMe
    users.ts            유저 관련 엔드포인트
    index.ts            export * from './auth' …
```

## authToken.ts — 토큰 메모리 관리

```typescript
// lib/authToken.ts
let _accessToken: string | null = null;

export function getApiAccessToken(): string | null {
  return _accessToken;
}

export function setApiAccessToken(token: string): void {
  _accessToken = token;
}

export function removeApiAccessToken(): void {
  _accessToken = null;
}
```

## fetchApi.ts — 공개 API용

```typescript
// lib/api/fetchApi.ts
const API_URL = process.env.NEXT_PUBLIC_API_URL ?? 'http://localhost:3030';

export async function fetchApi<T>(
  path: string,
  options?: RequestInit,   // fetch 옵션 그대로 투과
): Promise<T> {
  const res = await fetch(`${API_URL}${path}`, {
    ...options,
    headers: {
      'Content-Type': 'application/json',
      ...options?.headers,
    },
  });

  if (!res.ok) {
    const message = await res.text();
    throw new Error(message || `HTTP ${res.status}`);
  }

  return res.json() as Promise<T>;
}
```

## authFetch.ts — 인증 API용 (Bearer 자동 첨부)

```typescript
// lib/api/authFetch.ts
import { getApiAccessToken } from '../authToken';
import { fetchApi }          from './fetchApi';

export async function authFetchApi<T>(
  path: string,
  options?: RequestInit,
): Promise<T> {
  const token = getApiAccessToken();

  return fetchApi<T>(path, {
    ...options,
    headers: {
      ...options?.headers,
      ...(token ? { Authorization: `Bearer ${token}` } : {}),
    },
  });
}
```

## 도메인 파일 — auth.ts

```typescript
// lib/api/auth.ts
import { fetchApi }          from './fetchApi';
import { authFetchApi }      from './authFetch';
import { setApiAccessToken } from '../authToken';
import type { ApiAuthResponse, ApiAuthUser } from './apiTypes';

async function postAuth(
  path: 'login' | 'register',
  body: Record<string, string>,
): Promise<ApiAuthResponse> {
  const data = await fetchApi<ApiAuthResponse>(`/auth/${path}`, {
    method: 'POST',
    body:   JSON.stringify(body),
  });
  setApiAccessToken(data.accessToken);
  return data;
}

export function login(email: string, password: string) {
  return postAuth('login', { email: email.trim(), password });
}

export function register(email: string, password: string, nickname: string) {
  return postAuth('register', { email: email.trim(), password, nickname: nickname.trim() });
}

export async function fetchMe(): Promise<ApiAuthUser> {
  return authFetchApi<ApiAuthUser>('/auth/me');
}
```

## index.ts — 진입점

```typescript
// lib/api/index.ts
export * from './auth';
export * from './users';

// 사용
import { login, fetchMe } from '@/lib/api';
```

---

# 패턴 B — apiFetch + token? 통합

```typescript
// lib/api/apiFetch.ts
const API_URL = process.env.NEXT_PUBLIC_API_URL ?? 'http://localhost:3030';

export async function apiFetch<T>(
  path: string,
  options: RequestInit & { token?: string | null } = {},
  //       ↑ RequestInit의 모든 필드 + token 필드 교차 추가
  //                        ? = 없어도 됨
  //                               | null = null도 허용 (토큰 없음 명시)
  //       = {} → 아무 옵션 안 넘겨도 됨: apiFetch('/health')
): Promise<T> {
  const { token, headers, ...rest } = options;
  //      ↑ token은 fetch 표준이 아니라서 따로 꺼냄
  //                           ↑ 나머지(method, body, signal)는 rest로

  const res = await fetch(`${API_URL}${path}`, {
    ...rest,
    headers: {
      'Content-Type': 'application/json',
      ...(token ? { Authorization: `Bearer ${token}` } : {}),
      //  token 있으면 Bearer 헤더 추가, 없으면 생략
      ...headers,
    },
  });

  if (!res.ok) {
    const body = await res.json().catch(() => null);
    const message = Array.isArray(body?.message)
      ? body.message.join(', ')
      : body?.message;
    throw new Error(message ?? `HTTP ${res.status}`);
  }

  return res.json() as Promise<T>;
}
```

```typescript
// 사용 예
apiFetch('/health')                          // 토큰 없이
apiFetch('/posts', { token: accessToken })   // 토큰 포함 → Bearer 자동
apiFetch('/posts', {
  method: 'POST',
  body: JSON.stringify(data),
  token: accessToken,
})
```

---

# 전체 흐름 요약

```txt
page.tsx (화면)
  ↓ login(email, password) 호출
auth.ts (도메인)
  ↓ fetchApi('/auth/login', ...) 호출       [패턴 A]
  ↓ apiFetch('/auth/login', ...)  호출      [패턴 B]
fetchApi / apiFetch (인프라)
  ↓ fetch('http://localhost:3030/...') 실행
NestJS
  ↓ 응답
fetchApi / apiFetch
  ↓ res.ok 체크 → res.json() 파싱 → 반환
auth.ts
  ↓ setApiAccessToken(data.accessToken)
page.tsx
```

---

# 헷갈리기 쉬운 구분 ⭐️⭐️⭐️⭐️

## fetchApi.ts 파일 vs fetchApi() 함수

```txt
파일: 공통 HTTP 유틸이 모인 곳 (fetchApi 함수, getApiBaseUrl 등)
함수: Bearer 없는 완성 호출 — login 등 공개 API에서 사용

authFetch.ts:
  fetchApi()를 호출하지 않고
  getApiBaseUrl + !ok 처리만 재사용 (Bearer를 먼저 붙여야 해서)
```

## authFetch.ts vs auth.ts

| |`authFetch.ts`|`auth.ts`|
|---|---|---|
|층|인프라 (도구)|도메인 (메뉴)|
|질문|어떻게 보내나?|무엇을 보내나?|
|내용|path + Bearer + `!ok` + json|`login` · `fetchMe` · body · setToken|

## 누가 누굴 호출하는가

```txt
page    → auth.ts 만 호출 (path 문자열·헤더 직접 쓰지 않음)
auth.ts → fetchApi 또는 authFetchApi 선택
  login()   → fetchApi      (공개, Bearer 없음)
  fetchMe() → authFetchApi  (보호, Bearer 있음)
```
 
```mermaid-beautiful
flowchart LR
  P["page"] --> A["auth.ts\nlogin / fetchMe"]
  A -->|공개| F["fetchApi()"]
  A -->|보호| AF["authFetchApi()"]
  AF --> T["authToken"]
  F --> N["NestJS"]
  AF --> N
```