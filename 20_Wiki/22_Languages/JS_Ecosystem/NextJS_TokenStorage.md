---
aliases:
  - Access Token Storage
  - authToken
  - localStorage
tags:
  - NextJS
related:
  - "[[00_JS_Ecosystem_HomePage]]"
  - "[[Auth_Concept]]"
  - "[[JS_WebStorage]]"
---
# NextJS_TokenStorage — 토큰 저장 전략

>[!info]
>Access Token을 브라우저 어디에 저장하는가의 문제. 
>메모리(가장 안전, 새로고침 소멸), localStorage(새로고침 유지, XSS 취약), httpOnly 쿠키(JS 접근 불가, 가장 안전). 
>실전에서는 메모리 또는 localStorage 중 선택하고 `authToken.ts`로 캡슐화. 
>JWT 개념 → [[Auth_Concept]]

---

# 왜 저장 전략이 필요한가 ⭐️⭐️⭐️⭐️

```txt
로그인하면 Access Token을 받음
이후 API 요청마다 Authorization: Bearer {token} 헤더에 담아야 함
→ 토큰을 어딘가에 저장해야 함

문제:
  1. 어디에 저장하면 안전한가? (보안)
  2. 새로고침해도 유지되는가? (UX)
  3. 여러 컴포넌트에서 쉽게 꺼낼 수 있는가? (편의)
```

---

# 저장 방법 비교 ⭐️⭐️⭐️⭐️

|저장소|새로고침|XSS|CSRF|설명|
|---|---|---|---|---|
|메모리 (변수)|❌ 소멸|✅ 안전|✅ 안전|JS 변수에 저장|
|localStorage|✅ 유지|❌ 취약|✅ 안전|브라우저 로컬 저장|
|sessionStorage|❌ 탭 닫으면 소멸|❌ 취약|✅ 안전|탭 단위 저장|
|httpOnly 쿠키|✅ 유지|✅ 안전|❌ 취약|서버가 Set-Cookie로 설정|

```txt
XSS (Cross-Site Scripting):
  악성 스크립트가 페이지에 삽입돼서 localStorage 등을 읽어가는 공격
  localStorage는 JS로 접근 가능 → XSS에 취약
  메모리 변수·httpOnly 쿠키는 JS로 접근 불가 → XSS에 안전

CSRF (Cross-Site Request Forgery):
  다른 사이트에서 사용자 쿠키를 이용해 요청을 보내는 공격
  쿠키는 자동으로 전송되므로 CSRF 위험
  Authorization 헤더는 자동 전송 안 됨 → CSRF 안전

Access Token 권장:
  짧은 만료시간(15분) → 탈취돼도 빨리 만료
  → 메모리 또는 localStorage 둘 다 실용적
  → httpOnly 쿠키는 JS에서 읽을 수 없어서 Bearer 헤더에 못 넣음

Refresh Token 권장:
  만료시간이 김 → 탈취 시 위험
  → httpOnly 쿠키 (JS 접근 불가 = 안전)
```

---

# authToken.ts — 캡슐화 패턴 ⭐️⭐️⭐️⭐️

```txt
authToken.ts를 만드는 이유:
  저장 방법(메모리 vs localStorage)을 사용처에서 몰라도 됨
  나중에 저장 방법을 바꿔도 authToken.ts만 수정하면 됨
  get / set / remove 세 함수로 통일된 인터페이스 제공
```

## 메모리 방식 (Access Token 권장)

```typescript
// lib/authToken.ts — 메모리 저장
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

```txt
장점:
  JS 변수라 XSS로 탈취 불가 (다른 스크립트가 접근 못함)
  구현이 단순

단점:
  새로고침하면 사라짐
  → 앱 초기화 시 /auth/me 로 서버에서 유저 정보 다시 받아와야 함
  (서버가 Refresh Token 쿠키로 새 Access Token 발급)
```

## localStorage 방식

```typescript
// lib/authToken.ts — localStorage 저장
const STORAGE_KEY = 'mc_access_token';

export function getApiAccessToken(): string | null {
  if (typeof window === 'undefined') return null;
  // typeof window === 'undefined' → Server Component에서 실행 중
  // Server에는 localStorage가 없으므로 null 반환

  return localStorage.getItem(STORAGE_KEY);
}

export function setApiAccessToken(token: string): void {
  localStorage.setItem(STORAGE_KEY, token);
}

export function removeApiAccessToken(): void {
  localStorage.removeItem(STORAGE_KEY);
}
```

```txt
장점:
  새로고침해도 토큰 유지
  탭을 닫았다가 다시 열어도 유지 (sessionStorage는 탭 닫으면 소멸)

단점:
  XSS 공격으로 토큰 탈취 가능

typeof window === 'undefined' 체크가 필요한 이유:
  Next.js는 Server Component를 서버에서 실행
  서버에는 window, localStorage가 없음
  → undefined 체크 없으면 ReferenceError
  → null 반환해서 "로그인 안 된 상태"로 처리
```

---

# 새로고침 문제 — 메모리 방식의 해결 ⭐️⭐️⭐️⭐️

```txt
메모리 저장의 문제:
  새로고침 → _accessToken = null로 초기화
  → authFetchApi가 Bearer 없이 요청 → 401

해결 흐름:
  앱 시작(layout.tsx) → fetchMe() 호출
  → 서버가 httpOnly 쿠키의 Refresh Token으로 Access Token 재발급
  → setApiAccessToken(newToken)
  → 이후 요청 정상
```

```typescript
// app/layout.tsx — 앱 시작 시 토큰 복구
'use client';
import { useEffect } from 'react';
import { fetchMe }   from '@/lib/api';
import { useAuthStore } from '@/store/authStore';

export function AuthInitializer() {
  const setUser = useAuthStore(s => s.setUser);

  useEffect(() => {
    fetchMe()
      .then(user => setUser(user))
      .catch(() => {})  // 비로그인이면 그냥 무시
  }, []);

  return null;
}
```

---

# 어떤 방식을 선택할까 ⭐️⭐️⭐️

```txt
보안이 최우선:
  메모리 방식 + Refresh Token httpOnly 쿠키
  XSS로 Access Token 탈취 불가
  새로고침 시 /auth/me로 복구

구현 단순함 우선:
  localStorage 방식
  새로고침 복구 로직 불필요
  XSS 대비는 다른 방법으로 (CSP 헤더 등)

이 프로젝트:
  authToken.ts가 캡슐화되어 있어서
  나중에 메모리 ↔ localStorage 교체가 쉬움
  → 먼저 localStorage로 개발하고 나중에 메모리로 전환도 가능
```

---

# localStorage API 기본 ⭐️⭐️

```typescript
// 저장
localStorage.setItem('key', 'value');      // 문자열만 저장 가능
localStorage.setItem('user', JSON.stringify({ id: 1 }));  // 객체는 직렬화

// 읽기
localStorage.getItem('key');               // 없으면 null
JSON.parse(localStorage.getItem('user') ?? '{}');

// 삭제
localStorage.removeItem('key');

// 전체 삭제
localStorage.clear();
```

```txt
localStorage 특징:
  같은 도메인(origin)에서만 접근 가능
  문자열만 저장 → 객체는 JSON.stringify/parse 필요
  용량 제한: 약 5MB
  동기 API → 대용량 데이터에 부적합 (IndexedDB 사용)
  탭 간 공유됨 → 같은 도메인의 모든 탭에서 접근 가능
  → [[JS_WebStorage]] localStorage·sessionStorage·IndexedDB 상세
```