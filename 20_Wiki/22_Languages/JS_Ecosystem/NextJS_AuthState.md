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

> [!info] 
> 로그인한 유저 정보를 앱 전체에서 공유하는 방법. 
> Context + useState로 `AuthProvider`를 만들거나 Zustand를 쓴다. 
> `hydrate` = 앱 시작 시 localStorage에서 토큰을 꺼내 store에 넣는 동작.
>  `hydrated` 플래그가 false인 동안은 Guard가 판단을 보류해야 로그인한 사람이 /login으로 튕기지 않는다. 
>  Access Token 저장 → [[NextJS_TokenStorage]], hydrate 함수·store 구현 → [[React_Zustand]]

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
|`setUser`|`(user) => void`|login/register 직후 응답의 user를 저장|
|`clearSession`|`() => void`|logout: 토큰 삭제 + user = null을 한 곳에|
|`refreshUser`|`() => Promise<void>`|토큰 보고 fetchMe로 user 재동기화|

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
    clearAuthStorage();
    setUser(null);
  }, []);

  const refreshUser = useCallback(async () => {
    if (!getApiAccessToken()) { setUser(null); return; }
    try {
      const me = await fetchMe();
      setUser(me);
    } catch {
      clearSession();
    }
  }, [clearSession]);

  useEffect(() => {
    let cancelled = false;
    async function init() {
      await refreshUser();
      if (!cancelled) setIsLoading(false);
    }
    init();
    return () => { cancelled = true; };
  }, [refreshUser]);

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

```txt
useCallback이 필요한 이유:
  함수를 그냥 선언하면 매 렌더마다 새 참조
  useEffect([refreshUser]) deps가 매번 바뀜 → fetchMe 무한 루프
  useCallback으로 deps가 같으면 이전 참조 유지 → 루프 없음

useMemo(value)가 필요한 이유:
  { user, isLoading, ... } 객체를 그냥 넣으면 매 렌더마다 새 객체
  → useContext를 쓰는 모든 컴포넌트가 리렌더
  useMemo로 deps가 실제로 바뀔 때만 새 객체 생성

setUser는 useState setter라 React가 이미 stable → useCallback 불필요
```

```typescript
// app/layout.tsx
import { AuthProvider } from '@/contexts/AuthContext';

export default function RootLayout({ children }: { children: React.ReactNode }) {
  return (
    <html>
      <body>
        <AuthProvider>{children}</AuthProvider>
      </body>
    </html>
  );
}

// 어디서든
const { user, clearSession } = useAuth();
```

---

# Zustand 방식 (대안)

```typescript
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
Context vs Zustand:
  Context  → React 내장, Provider 트리 구조가 명확
             useCallback·useMemo 직접 챙겨야 함

  Zustand  → 설정 단순, Provider 없이 어디서든 접근
             성능 최적화가 쉬움 (선택적 구독)
             localStorage 통합도 자연스러움

  setSession·clearSession·hydrate (localStorage 통합) →
  [[React_Zustand]] 실전 — 인증 스토어 섹션
```

---

# hydrate · hydrated — Guard의 판단 기준 ⭐️⭐️⭐️⭐️

```txt
Next.js는 서버에서 먼저 렌더링한다.
서버에는 localStorage가 없다 → store 초기값은 null

서버 렌더:     accessToken = null  (localStorage 없음)
브라우저 마운트: useEffect → hydrate() → accessToken = "eyJhb..."

hydrate() = localStorage에서 토큰을 꺼내 store에 넣는 동작
hydrated  = hydrate()가 완료됐는지 알려주는 플래그
```

```txt
hydrated 없이 Guard를 짜면:

  if (accessToken === null) router.replace('/login')

  브라우저 첫 렌더 시 accessToken은 항상 null
  hydrate가 아직 안 된 것뿐인데
  로그인한 사람도 /login으로 튕겨버림

hydrated 있으면:

  마운트   hydrated: false, accessToken: null  → 아직 모름, 기다려
  완료     hydrated: true,  accessToken: null  → 진짜 비로그인 → /login
  완료     hydrated: true,  accessToken: "eyJ" → 로그인 됨 → 통과

  hydrated: false인 동안은 Guard가 아무것도 안 하고 기다림
  true가 된 뒤에야 null인지 값이 있는지를 판단
```

---

# children 감싸는 래퍼 3종 ⭐️⭐️⭐️⭐️

```txt
Provider, Gate, Bootstrap — 겉모양이 같지만 역할이 다름

  Provider  → createContext로 값을 아래로 내려줌. children 항상 렌더
  Gate      → 조건 확인 후 children 렌더 여부 결정. 실패 시 redirect/null
  Bootstrap → 초기화만 하고 children 그대로 반환. 화면에 아무것도 안 더함
```

## Gate 패턴 ⭐️⭐️⭐️⭐️

```typescript
// AdminGate — hydrated store 방식 (권장)
'use client';
import { useEffect }    from 'react';
import { useRouter }    from 'next/navigation';
import { useAuthStore } from '@/lib/auth-store';

export function AdminGate({ children }: { children: React.ReactNode }) {
  const router      = useRouter();
  const accessToken = useAuthStore(s => s.accessToken);
  const user        = useAuthStore(s => s.user);
  const hydrated    = useAuthStore(s => s.hydrated);

  useEffect(() => {
    if (!hydrated) return;                     // hydrate 완료 전 → 대기
    if (!accessToken) {
      router.replace('/login?next=/admin');
      return;
    }
    if (!user) return;                         // user 로딩 중 → 대기
    if (user.role !== 'admin') router.replace('/');
  }, [hydrated, accessToken, user, router]);

  if (!hydrated || !accessToken || !user) return <p>확인 중…</p>;
  if (user.role !== 'admin') return null;

  return children;
}
```

```typescript
// 또는 ready useState 방식 — store에 hydrated가 없을 때
export function AdminGate({ children }: { children: React.ReactNode }) {
  const router      = useRouter();
  const accessToken = useAuthStore(s => s.accessToken);
  const user        = useAuthStore(s => s.user);
  const [ready, setReady] = useState(false);

  useEffect(() => { setReady(true); }, []);

  useEffect(() => {
    if (!ready) return;
    if (!accessToken) { router.replace('/login?next=/admin'); return; }
    if (!user) return;
    if (user.role !== 'admin') router.replace('/');
  }, [ready, accessToken, user, router]);

  if (!ready || !accessToken || !user) return <p>확인 중…</p>;
  if (user.role !== 'admin') return null;
  return children;
}
```

```txt
hydrated vs ready:
  hydrated (store) → AuthBootstrap.hydrate() 완료 시 true. "localStorage 읽기 완료"를 정확히 추적
  ready (useState) → 마운트 됐음만 표시. store에 hydrated 없을 때 간단하게 사용

사용:
  AdminGate, AuthGate, GuestGate, PremiumGate — [대상]Gate 패턴

→ hydrate 함수, hydrated 상태 구현 → [[React_Zustand]] localStorage 통합 섹션
```

```typescript
// layout.tsx에서
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