---
aliases:
  - BFF
  - Cross-Origin Cookie
  - GET /me
  - SSR CSR 인증
  - Zustand
tags:
  - NextJS
related:
  - "[[00_JS_Ecosystem_HomePage]]"
  - "[[Auth_Concept]]"
  - "[[NextJS_TokenStorage]]"
  - "[[React_Context_Provider]]"
  - "[[React_Zustand]]"
---
# NextJS_AuthState — 로그인 유저 상태 관리

>[!info]
>로그인한 유저 정보(`{ id, email, role }`)를 앱 전체에서 공유하는 방법. 
>Context + useState로 `AuthProvider`를 만들거나 Zustand를 쓴다. 
>`useCallback`으로 함수 참조를 안정화하고 `useMemo`로 value 객체를 안정화해야 무한루프·불필요한 리렌더를 막는다.
> Access Token 저장 → [[NextJS_TokenStorage]], `useCallback`·`useMemo` 개념 → [[React_useMemo_useCallback]]

---

# 왜 유저 상태 관리가 필요한가 ⭐️⭐️⭐️⭐️

```txt
로그인 후 사용하는 곳:
  Header → "홍길동님 환영합니다"
  게시글 → 내 글이면 수정/삭제 버튼
  라우트 → 비로그인이면 /login 리다이렉트

문제:
  props로 내리면 depth가 깊어질수록 불편 (prop drilling)
  → 전역 상태(Global State)로 관리
```

---

# Context value에 무엇을 넣는가 ⭐️⭐️⭐️⭐️

|필드|타입|역할|
|---|---|---|
|`user`|`ApiAuthUser \| null`|로그인한 유저. null = 비로그인|
|`isLoading`|`boolean`|첫 복구 전 깜빡이는 UI 방지|
|`setUser`|`(user) => void`|login/register 직후 응답의 user를 저장. 매번 fetchMe 불필요|
|`clearSession`|`() => void`|logout: 토큰 삭제 + user = null을 한 곳에|
|`refreshUser`|`() => Promise<void>`|토큰 보고 fetchMe로 user 재동기화. 마운트·수동 갱신|

```txt
setUser만으로 부족한가:
  로그인 직후 → 응답에 user 있음 → setUser 충분 (fetchMe 절약)
  새로고침 후 → 메모리에 user 없음 → refreshUser 필요

clearSession을 매 페이지에서 직접 쓰지 않는 이유:
  removeToken + setUser(null)을 따로 호출하면 빼먹는 곳 생김
  → 한 함수로 묶어서 일관성 보장
```

---

# 파일 쌓는 순서 ⭐️⭐️⭐️⭐️

```txt
1. 'use client'         Context·hooks는 클라이언트
2. createContext        상자 타입 정의 (처음엔 null)
3. useState(user)       실제 유저 데이터
4. useState(isLoading)  첫 fetchMe 끝나기 전 true
5. useCallback 메서드   clearSession / refreshUser (안정화 필요)
6. useEffect            마운트 시 refreshUser (복구 시도)
7. useMemo(value)       Provider에 넣을 객체 안정화
8. export useAuth()     Provider 밖이면 throw
9. layout에서           <AuthProvider>{children}</AuthProvider>
```

---

# AuthProvider 구현 ⭐️⭐️⭐️⭐️

```typescript
// contexts/AuthContext.tsx
'use client';

import {
  createContext, useCallback, useContext, useEffect,
  useMemo, useState, type ReactNode,
} from 'react';
import { fetchMe, clearAuthStorage } from '@/lib/api';
import { getApiAccessToken }         from '@/lib/authToken';
import type { ApiAuthUser }          from '@/lib/apiTypes';

type AuthContextValue = {
  user:         ApiAuthUser | null;
  isLoading:    boolean;
  setUser:      (user: ApiAuthUser) => void;
  clearSession: () => void;
  refreshUser:  () => Promise<void>;
};

const AuthContext = createContext<AuthContextValue | null>(null);

export function AuthProvider({ children }: { children: ReactNode }) {
  const [user,      setUser]      = useState<ApiAuthUser | null>(null);
  const [isLoading, setIsLoading] = useState(true);

  const clearSession = useCallback(() => {
    clearAuthStorage();  // 토큰 삭제
    setUser(null);
  }, []);
  // deps [] = 이 함수는 절대 바뀌지 않음

  const refreshUser = useCallback(async () => {
    if (!getApiAccessToken()) {
      setUser(null);   // 토큰 없으면 비로그인
      return;
    }
    try {
      const me = await fetchMe();
      setUser(me);
    } catch {
      clearSession();  // 토큰 만료·위조 → 세션 정리
    }
  }, [clearSession]);
  // deps [clearSession] = clearSession이 바뀌면 refreshUser도 새로 만듦

  useEffect(() => {
    let cancelled = false;
    async function init() {
      await refreshUser();
      if (!cancelled) setIsLoading(false);
    }
    init();
    return () => { cancelled = true; };
  }, [refreshUser]);
  // refreshUser를 deps에 넣어도 useCallback 덕분에 무한루프 없음

  const value = useMemo(
    () => ({ user, isLoading, setUser, clearSession, refreshUser }),
    [user, isLoading, setUser, clearSession, refreshUser],
  );

  return (
    <AuthContext.Provider value={value}>
      {children}
    </AuthContext.Provider>
  );
}

export function useAuth(): AuthContextValue {
  const ctx = useContext(AuthContext);
  if (!ctx) throw new Error('useAuth는 AuthProvider 안에서만 사용할 수 있습니다.');
  return ctx;
}
```

---

# 왜 useCallback인가 ⭐️⭐️⭐️⭐️

```txt
useCallback 없이 그냥 함수를 정의하면:
  AuthProvider가 렌더링될 때마다
  clearSession, refreshUser가 새 함수 객체로 생성됨

  useEffect([refreshUser])의 deps가 매 렌더마다 바뀜
  → effect가 매 렌더마다 다시 실행
  → fetchMe 루프 발생!

useCallback(() => { ... }, [deps]):
  deps가 같으면 이전 렌더의 함수 참조를 그대로 유지
  → useEffect deps가 바뀌지 않음 → 루프 없음

setUser는 왜 useCallback이 필요 없는가:
  useState의 setter는 React가 이미 stable하게 만들어줌
  매 렌더마다 같은 참조 → 감쌀 필요 없음
  Context value type에는 (user) => void로만 노출
```

---

# 왜 useMemo로 value를 감싸는가 ⭐️⭐️⭐️⭐️

```txt
useMemo 없이 그냥 객체로 넣으면:
  { user, isLoading, setUser, clearSession, refreshUser }
  → AuthProvider가 렌더링될 때마다 새 객체 생성
  → useContext(AuthContext)를 구독하는 모든 컴포넌트가 리렌더
  → user가 안 바뀌어도 Header, NavBar 전부 다시 그려짐

useMemo(valueFactory, [user, isLoading, ...]):
  deps가 같으면 이전 객체 재사용
  → user나 isLoading이 실제로 바뀔 때만 구독 컴포넌트 리렌더
```

---

# layout.tsx에 등록 + 사용 ⭐️⭐️⭐️

```typescript
// app/layout.tsx
import { AuthProvider } from '@/contexts/AuthContext';

export default function RootLayout({ children }: { children: React.ReactNode }) {
  return (
    <html>
      <body>
        <AuthProvider>
          {children}
        </AuthProvider>
      </body>
    </html>
  );
}

// 어디서든
'use client';
import { useAuth } from '@/contexts/AuthContext';

function Header() {
  const { user, clearSession } = useAuth();
  return user
    ? <button onClick={clearSession}>로그아웃</button>
    : <a href="/login">로그인</a>;
}
```

---

# 역할 분리

```txt
authToken.ts / fetchApi.ts  HTTP · 토큰 저장 → [[NextJS_TokenStorage]]
AuthProvider                "지금 로그인한 사람" UI 상태 (이 파일)
login() 성공 후             setApiAccessToken()  ← auth.ts 내부
                         + setUser()            ← AuthProvider
```

---

# Zustand 방식 (대안)

```typescript
// store/authStore.ts
import { create } from 'zustand';

export const useAuthStore = create<{
  user:    ApiAuthUser | null;
  setUser: (user: ApiAuthUser | null) => void;
  logout:  () => void;
}>((set) => ({
  user:    null,
  setUser: (user) => set({ user }),
  logout:  ()     => set({ user: null }),
}));
```

```txt
setSession·clearSession·hydrate (localStorage 통합 버전) →
  [[React_Zustand]] 실전 — 인증 스토어 섹션

Context vs Zustand:
  Context  → React 내장, Provider 트리 구조가 명확
             useCallback·useMemo 직접 챙겨야 함

  Zustand  → 설정 단순, Provider 없이 어디서든 접근
             성능 최적화가 더 쉬움 (선택적 구독)
             localStorage 통합도 자연스러움

  인증 상태처럼 단순하면 Context로 충분
  앱 전역 상태가 많아지면 Zustand 권장
```

---
# children 감싸는 래퍼 3종 비교 ⭐️⭐️⭐️⭐️

```txt
Provider, Gate, Bootstrap — 겉모양이 같지만 역할이 다름

  공통점: <Wrapper>{children}</Wrapper> 형태
  차이점: 내부에서 뭘 하는가

  Provider  → createContext로 값을 아래로 내려줌
              children은 항상 렌더됨
              context 값을 소비하는 하위 컴포넌트를 위한 것

  Gate      → 조건 확인 후 children 렌더 여부 결정
              조건 실패 시 redirect 또는 null
              children 자체가 렌더될 자격을 따짐

  Bootstrap → 초기화만 수행하고 children을 그대로 반환
              화면에 아무것도 더하지 않음
              ex) 앱 시작 시 localStorage → Zustand 복원
```

## Gate 패턴 — 조건부 children 렌더 ⭐️⭐️⭐️⭐️

```typescript
// AdminGate — admin이 아니면 redirect, 맞으면 children 렌더
'use client';
import { useEffect, useState } from 'react';
import { useRouter }           from 'next/navigation';
import { useAuthStore }        from '@/lib/auth-store';

export function AdminGate({ children }: { children: React.ReactNode }) {
  const router      = useRouter();
  const accessToken = useAuthStore(s => s.accessToken);
  const user        = useAuthStore(s => s.user);
  const [ready, setReady] = useState(false);

  // SSR 후 클라이언트 마운트 확인 (hydrate 완료 후 체크)
  useEffect(() => { setReady(true); }, []);

  useEffect(() => {
    if (!ready) return;
    if (!accessToken) {
      router.replace('/login?next=/admin');  // 미로그인 → 로그인 페이지
      return;
    }
    if (!user) return;                       // user 로딩 중 → 대기
    if (user.role !== 'admin') router.replace('/');  // 일반 유저 → 홈
  }, [ready, accessToken, user, router]);

  // 확인 중 → 로딩 표시
  if (!ready || !accessToken || !user) return <p>확인 중…</p>;

  // admin 아니면 아무것도 안 그림 (redirect 중)
  if (user.role !== 'admin') return null;

  return children;  // admin만 도달
}
```

```typescript
// 사용 — layout.tsx에서 전체 감싸기
export default function AdminLayout({ children }: { children: React.ReactNode }) {
  return (
    <AdminGate>
      <div className="admin-shell">
        <aside>...</aside>
        {children}
      </div>
    </AdminGate>
  );
}
```

```txt
Gate 내부 흐름:
  ready = false (SSR/마운트 전) → "확인 중…"
  accessToken 없음              → /login으로 redirect
  user 없음                     → "확인 중…" (auth/me 응답 대기)
  user.role !== 'admin'         → / 로 redirect → null 반환
  user.role === 'admin'         → children 렌더 ✅

ready 가드가 필요한 이유:
  Zustand는 SSR 시 초기값(null)으로 시작
  마운트 전에 체크하면 항상 "미로그인"으로 판단 → 잘못된 redirect
  useEffect(() => setReady(true)) → 클라이언트 마운트 후에만 체크
  → [[React_Zustand]] hydrate 패턴 참고

AuthBootstrap과의 차이:
  AuthBootstrap → localStorage 복원 + /auth/me 호출 (초기화)
  AdminGate     → 그 결과를 보고 접근 허용/거부 (게이트)
  둘을 함께 쓰면: Bootstrap이 user를 채우면 Gate가 판단
```

## 범용 Gate 패턴

```typescript
// 조건만 바꾸면 어디서든 재사용
function AuthGate({ children }: { children: React.ReactNode }) {
  // 로그인한 유저만
  if (!user) { router.replace('/login'); return null; }
  return children;
}

function AdminGate({ children }: { children: React.ReactNode }) {
  // 어드민만
  if (user?.role !== 'admin') { router.replace('/'); return null; }
  return children;
}

function GuestGate({ children }: { children: React.ReactNode }) {
  // 비로그인 유저만 (로그인 페이지에 이미 로그인한 유저가 접근하면 홈으로)
  if (user) { router.replace('/'); return null; }
  return children;
}
```

```txt
Gate 이름 패턴:
  [대상]Gate — 그 대상만 통과
  AdminGate, AuthGate, GuestGate, PremiumGate ...

Context Provider가 아닌 이유:
  Provider는 createContext + value를 내려줌
  Gate는 값을 내려주지 않고 렌더 여부만 결정
  형태가 비슷해도 역할이 전혀 다름
```