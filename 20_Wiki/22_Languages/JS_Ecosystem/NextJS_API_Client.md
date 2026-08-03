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
---
# NextJS_API_Client — apiFetch 래퍼

> [!info] 
> apiFetch = 모든 API 호출의 기반. 토큰 추가·에러 파싱·URL 구성을 한 곳에서 처리해서 각 기능 파일이 반복하지 않아도 됨. 
> 개별 fetch 함수들은 apiFetch를 감싸서 특정 엔드포인트를 호출한다 → [[NextJS_API_Mapper]]

---

# 왜 래퍼가 필요한가 ⭐️⭐️⭐️⭐️

```typescript
// ❌ fetch를 직접 쓰면 매 호출마다 반복
const res1 = await fetch(`${process.env.NEXT_PUBLIC_API_URL}/users`, {
  headers: { Authorization: `Bearer ${token}` },
});
if (!res1.ok) throw new Error('실패');
const user = await res1.json();

const res2 = await fetch(`${process.env.NEXT_PUBLIC_API_URL}/rooms`, {
  headers: { Authorization: `Bearer ${token}` },
});
if (!res2.ok) throw new Error('실패');
const rooms = await res2.json();
```

```typescript
// ✅ apiFetch 래퍼 — 반복 제거
const user  = await apiFetch<ApiUser>('/users');
const rooms = await apiFetch<ApiRoom[]>('/rooms');
```

```txt
apiFetch가 대신 해주는 것:
  ① Base URL 앞에 붙이기
  ② Authorization 헤더 추가
  ③ 응답 JSON 파싱
  ④ HTTP 에러 처리 (res.ok 체크 + 에러 응답 파싱)
  ⑤ 401 시 토큰 갱신 후 재시도

이 5가지를 모든 fetch 호출에서 직접 하면:
  코드 중복 + 수정 시 모든 호출을 찾아다녀야 함
```

---

# apiFetch 구현 ⭐️⭐️⭐️⭐️

```typescript
// lib/api-client.ts

export type ApiError = {
  statusCode: number;
  message:    string;
};

export class ApiRequestError extends Error {
  constructor(
    public readonly statusCode: number,
    message: string,
  ) {
    super(message);
    this.name = 'ApiRequestError';
  }
}

export async function apiFetch<T>(
  path: string,
  options: RequestInit = {},
): Promise<T> {
  const baseUrl = process.env.NEXT_PUBLIC_API_URL ?? '';
  const token   = getAccessToken();  // 쿠키 또는 메모리에서 토큰 읽기

  const res = await fetch(`${baseUrl}${path}`, {
    ...options,
    headers: {
      'Content-Type': 'application/json',
      ...(token ? { Authorization: `Bearer ${token}` } : {}),
      ...options.headers,  // 호출하는 쪽에서 헤더를 추가/덮어쓸 수 있음
    },
  });

  // ① 401 → 토큰 갱신 후 재시도
  if (res.status === 401) {
    const refreshed = await tryRefreshToken();
    if (refreshed) return apiFetch<T>(path, options);  // 재시도
    redirectToLogin();
    throw new ApiRequestError(401, '로그인이 필요합니다.');
  }

  // ② 성공
  if (res.ok) {
    if (res.status === 204) return undefined as T;  // No Content
    return res.json() as Promise<T>;
  }

  // ③ HTTP 에러 — 서버 에러 메시지 파싱
  const body = await res.json().catch(() => null);
  const message =
    typeof body?.message === 'string'
      ? body.message
      : `요청에 실패했습니다. (${res.status})`;

  throw new ApiRequestError(res.status, message);
}
```

---

# HTTP 메서드 헬퍼 ⭐️⭐️⭐️

```typescript
// GET — body 없음
export const apiGet = <T>(path: string) =>
  apiFetch<T>(path, { method: 'GET' });

// POST — body 있음
export const apiPost = <T>(path: string, body?: unknown) =>
  apiFetch<T>(path, {
    method: 'POST',
    body:   body !== undefined ? JSON.stringify(body) : undefined,
  });

// PATCH — 부분 업데이트
export const apiPatch = <T>(path: string, body?: unknown) =>
  apiFetch<T>(path, {
    method: 'PATCH',
    body:   body !== undefined ? JSON.stringify(body) : undefined,
  });

// DELETE
export const apiDelete = <T = void>(path: string) =>
  apiFetch<T>(path, { method: 'DELETE' });
```

```typescript
// 사용 예
const user  = await apiGet<ApiUser>('/users/me');
const post  = await apiPost<ApiPost>('/posts', { title, content });
const updated = await apiPatch<ApiPost>(`/posts/${id}`, { title });
await apiDelete(`/posts/${id}`);
```

---

# Content-Type 분기 — FormData ⭐️⭐️⭐️

```typescript
// 파일 업로드처럼 multipart/form-data를 써야 할 때
// Content-Type을 자동으로 설정하려면 header에서 제거

export async function apiFetch<T>(
  path:    string,
  options: RequestInit = {},
): Promise<T> {
  const isFormData = options.body instanceof FormData;

  const res = await fetch(`${baseUrl}${path}`, {
    ...options,
    headers: {
      // FormData면 Content-Type 생략 → 브라우저가 boundary 포함해서 자동 설정
      ...(isFormData ? {} : { 'Content-Type': 'application/json' }),
      ...(token ? { Authorization: `Bearer ${token}` } : {}),
      ...options.headers,
    },
  });
  // ...
}

// 사용
const formData = new FormData();
formData.append('image', file);
const result = await apiFetch<ApiUser>('/users/me/image', {
  method: 'PATCH',
  body:   formData,   // FormData 직접 전달
});
```

---

# 에러 처리 패턴 ⭐️⭐️⭐️⭐️

```typescript
// 호출하는 쪽에서 에러 처리
try {
  const user = await apiGet<ApiUser>('/users/me');
  setUser(user);
} catch (err) {
  if (err instanceof ApiRequestError) {
    if (err.statusCode === 404) setError('사용자를 찾을 수 없습니다.');
    else if (err.statusCode === 403) setError('접근 권한이 없습니다.');
    else setError(err.message);  // 서버가 보낸 메시지
  } else {
    setError('네트워크 오류가 발생했습니다.');
  }
}
```

```txt
ApiRequestError vs 일반 Error:
  ApiRequestError → HTTP 응답이 왔는데 에러인 경우 (4xx, 5xx)
                    statusCode로 분기 가능
  일반 Error      → 네트워크 자체가 안 됨 (fetch 실패, CORS 등)

err.message:
  서버가 응답 body에 담아 보낸 메시지
  NestJS에서 throw new NotFoundException('메시지') 하면 이게 message에 담김
```

---

# 토큰 관리 ⭐️⭐️⭐️⭐️

```typescript
// 토큰 저장 위치에 따라 다르게 구현
// 방법 1 — 메모리 (XSS에 안전, 새로고침 시 사라짐)
let accessToken: string | null = null;

export const getAccessToken   = () => accessToken;
export const setAccessToken   = (token: string) => { accessToken = token; };
export const clearAccessToken = () => { accessToken = null; };

// 방법 2 — 쿠키 (HttpOnly면 XSS에 안전, 새로고침에도 유지)
// → 쿠키는 서버가 Set-Cookie로 직접 설정
// → 클라이언트는 withCredentials: true 또는 credentials: 'include'만 추가
```

```typescript
// 토큰 갱신
async function tryRefreshToken(): Promise<boolean> {
  try {
    const res = await fetch(`${baseUrl}/auth/refresh`, {
      method:      'POST',
      credentials: 'include',  // 갱신 토큰 쿠키 포함
    });
    if (!res.ok) return false;

    const { accessToken } = await res.json();
    setAccessToken(accessToken);  // 새 토큰 저장
    return true;
  } catch {
    return false;
  }
}
```

```txt
토큰 저장 전략 상세 → [[NextJS_TokenStorage]]
갱신 토큰 흐름, iOS ITP 문제 → [[NestJS_CORS]]
```

---

# 환경변수 설정

```bash
# .env.local
NEXT_PUBLIC_API_URL=http://localhost:3000
```

```txt
NEXT_PUBLIC_ 붙이는 이유:
  Next.js에서 클라이언트 사이드 코드에서 process.env를 쓰려면
  빌드 시 번들에 포함되어야 함
  NEXT_PUBLIC_ prefix가 있으면 브라우저에 노출
  없으면 서버 사이드에서만 접근 가능 (API 시크릿 키 등)
```