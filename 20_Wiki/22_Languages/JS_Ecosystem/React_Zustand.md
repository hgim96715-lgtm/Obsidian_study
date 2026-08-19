---
aliases:
  - 전역 상태 관리
  - Zustand
  - Context
tags:
  - React
  - NextJS
related:
  - "[[00_JS_Ecosystem_HomePage]]"
  - "[[NextJS_AuthState]]"
  - "[[React_AsyncUI]]"
  - "[[JS_WebStorage]]"
  - "[[NextJS_TokenStorage]]"
  - "[[TS_Type_Guards]]"
---
# React_Zustand — 전역 상태 관리

> [!info] 
> Zustand = React 전역 상태 관리 라이브러리. 
> Context보다 설정이 단순하고 Provider 없이 어디서든 접근 가능.
>  `create()`로 스토어 정의, `useXxxStore(selector)`로 컴포넌트에서 구독. 
>  선택적 구독으로 불필요한 리렌더 방지.
>   Context 방식 → [[NextJS_AuthState]]

---

# 왜 Zustand인가 ⭐️⭐️⭐️⭐️

```txt
Context 방식의 문제:
  Provider로 감싸야 함
  useCallback·useMemo 직접 챙겨야 함
  구독하는 컴포넌트 전부 리렌더 (useMemo 없으면)

Zustand:
  Provider 불필요 — 어디서든 import해서 바로 사용
  리렌더 최적화 내장 — selector로 필요한 것만 구독
  설정이 훨씬 단순
  devtools, persist 등 미들웨어 지원
```

---

# 설치 ⭐️⭐️⭐️

```bash
pnpm add zustand
# 또는
pnpm --filter web add zustand
```

---

# 기본 사용법 ⭐️⭐️⭐️⭐️

## 개념 먼저

```txt
Zustand 스토어 = "전역 변수" + "그 변수를 바꾸는 함수들"을 하나로 묶은 것

일반 변수:
  let count = 0;
  function increment() { count++; }
  → 바뀌어도 컴포넌트가 다시 그려지지 않음

Zustand:
  create()로 만든 스토어 = 바뀌면 구독 컴포넌트를 자동으로 리렌더
  useXxxStore(selector)로 구독

흐름:
  1. create()로 스토어 정의 (초기값 + 액션)
  2. useXxxStore(s => s.count) 로 컴포넌트에서 구독
  3. 액션 호출 → set()으로 상태 변경 → 구독 컴포넌트 리렌더
```

## 스토어 만들기

```typescript
// store/counterStore.ts
import { create } from 'zustand';

// ① 타입 정의 — 상태와 액션을 한 타입에
type CounterStore = {
  count:     number;        // 상태 (읽는 것)
  increment: () => void;   // 액션 (바꾸는 것)
  decrement: () => void;
  reset:     () => void;
};

// ② create() — 스토어 생성
export const useCounterStore = create<CounterStore>((set) => ({
  //         ↑ 훅 이름. 컴포넌트에서 이걸 import해서 씀
  //                                  ↑ set = 상태 변경 함수

  // 초기값
  count: 0,

  // 액션 — set()으로 상태 업데이트
  increment: () => set((state) => ({ count: state.count + 1 })),
  //                   ↑ 이전 state를 받아서 새 값 반환
  decrement: () => set((state) => ({ count: state.count - 1 })),
  reset:     () => set({ count: 0 }),
  //               ↑ 이전 값 필요 없으면 그냥 객체로
}));
```

## 컴포넌트에서 사용

```typescript
'use client';
import { useCounterStore } from '@/store/counterStore';

function Counter() {
  // ③ selector — 스토어에서 필요한 것만 꺼내서 구독
  const count     = useCounterStore((s) => s.count);
  //                                 ↑ s = 스토어 전체
  //                                       count만 구독 → count 바뀔 때만 리렌더
  const increment = useCounterStore((s) => s.increment);
  const decrement = useCounterStore((s) => s.decrement);

  return (
    <div>
      <button onClick={decrement}>-</button>
      <span>{count}</span>
      <button onClick={increment}>+</button>
    </div>
  );
}
```

```txt
selector = (s) => s.count:
  스토어 전체가 아닌 count 하나만 구독
  → count가 바뀔 때만 이 컴포넌트 리렌더
  → increment만 구독한 컴포넌트는 count 변경으로 리렌더 안 됨

Context와 다른 점:
  useContext는 Provider 값이 바뀌면 구독 컴포넌트 전부 리렌더
  Zustand는 selector로 지정한 값이 바뀔 때만 리렌더
  → 성능 최적화 내장

Provider가 없는 이유:
  Zustand 스토어는 React 트리 밖에 있는 독립적인 저장소
  어떤 컴포넌트에서든 import해서 바로 사용 가능
```

---

# set · get ⭐️⭐️⭐️⭐️

```typescript
export const useStore = create<Store>((set, get) => ({
  count: 0,
  name:  'Alice',

  // set — 상태 업데이트 (이전 상태를 안 봐도 될 때)
  setName: (name: string) => set({ name }),

  // set(fn) — 이전 상태 기반으로 업데이트
  increment: () => set((state) => ({ count: state.count + 1 })),

  // get — 다른 액션에서 현재 상태 읽기
  logCount: () => {
    const { count } = get();
    console.log(`현재 count: ${count}`);
  },

  // 여러 필드 동시 업데이트
  reset: () => set({ count: 0, name: 'Alice' }),
}));
```

```txt
set(부분객체):
  넘긴 필드만 업데이트 — 나머지는 유지
  set({ count: 5 }) → name은 그대로

set(fn):
  이전 state를 인자로 받아서 새 state 반환
  count + 1 처럼 이전 값 기반 업데이트에 사용

get():
  현재 스토어 전체 상태를 읽음
  액션 안에서 다른 상태 값이 필요할 때
```

---

# 실전 — 인증 스토어 ⭐️⭐️⭐️⭐️

## 기본 버전

```typescript
// store/authStore.ts
import { create }            from 'zustand';
import { removeApiAccessToken } from '@/lib/authToken';
import type { ApiAuthUser }  from '@/lib/apiTypes';

type AuthStore = {
  user:      ApiAuthUser | null;
  isLoading: boolean;
  setUser:   (user: ApiAuthUser | null) => void;
  logout:    () => void;
};

export const useAuthStore = create<AuthStore>((set) => ({
  user:      null,
  isLoading: true,

  setUser: (user) => set({ user, isLoading: false }),

  logout: () => {
    removeApiAccessToken();
    set({ user: null });
  },
}));
```

## localStorage 통합 버전 — token + user 함께 ⭐️⭐️⭐️⭐️

```typescript
// lib/auth-store.ts
'use client';
import { create } from 'zustand';

type AuthUser = {
  id:       string;
  email:    string;
  nickname: string;
  role:     'user' | 'admin';
};

type AuthState = {
  accessToken: string | null;
  user:        AuthUser | null;

  setSession:   (accessToken: string, user: AuthUser) => void;
  // 로그인 성공 시 — 토큰 + 유저를 한 번에 저장
  clearSession: () => void;
  // 로그아웃 시 — 토큰 + 유저를 한 번에 제거
  hydrate:      () => void;
  // 앱 시작 시 — localStorage에서 토큰 복원
};

const STORAGE_KEY = 'app_access_token';

export const useAuthStore = create<AuthState>((set) => ({
  accessToken: null,
  user:        null,

  setSession: (accessToken, user) => {
    localStorage.setItem(STORAGE_KEY, accessToken);
    set({ accessToken, user });
    // localStorage + Zustand 상태를 동시에 업데이트
  },

  clearSession: () => {
    localStorage.removeItem(STORAGE_KEY);
    set({ accessToken: null, user: null });
  },

  hydrate: () => {
    if (typeof window === 'undefined') return;
    // SSR(서버) 환경에서는 localStorage 없음 → 체크 필수
    const accessToken = localStorage.getItem(STORAGE_KEY);
    set({ accessToken });
    // 토큰만 복원 — user는 /auth/me 호출로 따로 복원
  },
}));
```

```txt
setSession(token, user):
  로그인 성공 후 한 번에 저장
  localStorage.setItem → 새로고침 후에도 토큰 유지
  set({ accessToken, user }) → 현재 세션 메모리에도 반영

clearSession():
  로그아웃 시 한 번에 정리
  한 곳에서 처리 → 빼먹는 곳 없음

hydrate():
  앱 시작(layout.tsx 마운트) 시 호출
  localStorage에서 토큰 읽어서 Zustand에 넣음
  → 새로고침 후에도 토큰 살아있음
  user는 없으므로 /auth/me로 따로 조회 필요

typeof window === 'undefined' 체크:
  Next.js Server Component·SSR에서 hydrate 호출되면
  서버에는 window·localStorage가 없어서 에러
  → undefined 체크로 서버에서는 바로 return
```

## hydrate란 — 서버 → 클라이언트 전환 시 store 복원 ⭐️⭐️⭐️⭐️

```txt
Next.js는 서버에서 먼저 렌더링한다.
서버에는 localStorage가 없다 → store 초기값은 null로 시작

서버 렌더:    accessToken = null  (localStorage 없음)
브라우저 마운트: useEffect → hydrate() → accessToken = "eyJhb..."

hydrate() = localStorage에서 토큰을 꺼내 Zustand store에 넣는 동작
  "서버에서 빈 상태로 시작한 store를 실제 값으로 채운다"는 의미
  = Next.js의 Hydration(서버 HTML + 클라이언트 JS 결합)에서 온 단어
```

## hydrated — "판단 가능한가" 플래그 ⭐️⭐️⭐️⭐️

```txt
문제:
  브라우저가 처음 뜨는 순간 accessToken은 항상 null
  hydrate가 아직 안 된 것뿐인데
  Gate가 "null = 비로그인"으로 판단 → /login으로 튕김

hydrated 플래그로 해결:
  hydrated: false → "아직 localStorage 못 읽었음, 기다려"
  hydrated: true  → "localStorage 읽기 완료, 이제 판단해도 됨"

  마운트 직후   hydrated: false, accessToken: null → 아직 모름, 대기
  hydrate 완료  hydrated: true,  accessToken: null → 진짜 비로그인 → /login
  hydrate 완료  hydrated: true,  accessToken: "eyJ..." → 로그인 됨 → 통과
```

```typescript
// AuthBootstrap — 앱 시작 시 hydrate 호출
'use client';
import { useEffect } from 'react';
import { useAuthStore } from '@/lib/auth-store';

export function AuthBootstrap() {
  const hydrate = useAuthStore(s => s.hydrate);

  useEffect(() => {
    hydrate();  // 마운트 직후 localStorage → store
  }, [hydrate]);

  return null;  // 화면에 아무것도 안 그림
}

// layout.tsx
<AuthBootstrap />    ← 전역 최상단에 두면 hydrate 보장
{children}
```

```txt
hydrate()가 useEffect 안에 있는 이유:
  useEffect는 브라우저에서만 실행됨
  서버 렌더 시 실행 안 됨 → localStorage 에러 없음
  typeof window === 'undefined' 체크와 같은 효과

→ [[NextJS_AuthState]] Gate에서 hydrated 사용 패턴
  서버에는 window·localStorage가 없어서 에러
  → undefined 체크로 서버에서는 바로 return
```

```typescript
// app/layout.tsx — 앱 시작 시 토큰 복원
'use client';
import { useEffect } from 'react';
import { useAuthStore } from '@/lib/auth-store';

export function AuthInitializer() {
  const hydrate = useAuthStore((s) => s.hydrate);

  useEffect(() => {
    hydrate();  // localStorage → Zustand
  }, [hydrate]);

  return null;
}

// 로그인 후 저장
const { setSession } = useAuthStore.getState();
setSession(data.accessToken, data.user);

// 로그아웃
const { clearSession } = useAuthStore.getState();
clearSession();

// 컴포넌트에서 읽기
const user        = useAuthStore((s) => s.user);
const accessToken = useAuthStore((s) => s.accessToken);
```

## AuthBootstrap — 앱 시작 시 세션 복구 ⭐️⭐️⭐️⭐️

```txt
왜 AuthBootstrap을 따로 분리하는가:
  app/layout.tsx = Server Component (기본값)
  → useEffect·localStorage·Zustand 훅 전부 사용 불가

  해결:
  'use client' 래퍼 컴포넌트(AuthBootstrap)를 만들어서
  클라이언트 전용 로직(hydrate, /auth/me)을 여기에 격리
  layout.tsx는 서버 컴포넌트 유지하면서 AuthBootstrap만 감싸기

문제:
  앱이 시작되면 Zustand 스토어는 초기값(null)으로 시작
  localStorage에 토큰이 있어도 모름 → 비로그인 상태로 출발

해결 — 2단계 복구:
  1단계: hydrate() → localStorage에서 accessToken 꺼내 Zustand에 넣기
  2단계: accessToken이 생기면 → /auth/me 호출 → user 정보 복원
```

```typescript
// components/auth/AuthBootstrap.tsx
'use client';
import { useEffect }                          from 'react';
import { apiFetch }                           from '@/lib/api';
import { useAuthStore, type AuthUser }        from '@/lib/auth-store';

export function AuthBootstrap({ children }: { children: React.ReactNode }) {
  const hydrate      = useAuthStore((s) => s.hydrate);
  const accessToken  = useAuthStore((s) => s.accessToken);
  const setSession   = useAuthStore((s) => s.setSession);
  const clearSession = useAuthStore((s) => s.clearSession);

  // 1단계 — 마운트 1회: localStorage → Zustand로 토큰 복원
  useEffect(() => {
    hydrate();
    // accessToken이 생기면 아래 useEffect가 감지해서 /auth/me 실행
  }, [hydrate]);

  // 2단계 — accessToken이 생길 때: /auth/me로 user 정보 복원
  useEffect(() => {
    if (!accessToken) return;  // 토큰 없으면 실행 안 함

    const token = accessToken;
    let cancelled = false;     // 언마운트·재실행 시 이전 fetch 무시

    async function fetchUser() {
      try {
        const me = await apiFetch<AuthUser>('/auth/me', { token });
        if (cancelled) return;
        setSession(token, me);   // 토큰 + user 함께 저장
      } catch {
        if (!cancelled) clearSession();  // 토큰 만료·위조 → 세션 정리
      }
    }

    fetchUser();
    return () => { cancelled = true; };  // 클린업
  }, [accessToken, setSession, clearSession]);

  return children;  // UI 없이 children을 그대로 렌더링
}
```

```typescript
// app/layout.tsx — 전체 앱을 감싸기
import { AuthBootstrap } from '@/components/auth/AuthBootstrap';

export default function RootLayout({ children }: { children: React.ReactNode }) {
  return (
    <html>
      <body>
        <AuthBootstrap>
          {children}
        </AuthBootstrap>
      </body>
    </html>
  );
}
```

```txt
useEffect가 두 개인 이유:
  1번: deps = [hydrate] → 마운트 1회만 실행 (localStorage → Zustand)
  2번: deps = [accessToken, ...] → accessToken이 생길 때 실행 (/auth/me)
  분리하면 각각 언제 실행되는지가 명확해짐

cancelled 플래그:
  컴포넌트 언마운트 or accessToken이 바뀌면
  진행 중인 fetch 결과를 무시 → 언마운트 후 setState 방지
  → [[React_AsyncUI]] useEffect fetch 섹션

children을 받는 이유:
  null 반환 대신 children을 감싸는 래퍼
  <AuthBootstrap>으로 전체를 감싸면
  "이 안에서 auth가 준비된다"는 의도가 명확
```


```typescript
// 훅 없이 스토어 직접 접근 (이벤트 핸들러, 비동기 함수 등)
useAuthStore.getState()           // 현재 상태 읽기
useAuthStore.getState().logout()  // 액션 실행
useAuthStore.setState({ user: null })  // 상태 직접 변경
```

---

# 컴포넌트 밖에서 접근 ⭐️⭐️⭐️

```typescript
// 훅 없이 스토어 직접 접근 (이벤트 핸들러, 비동기 함수 등)
useAuthStore.getState()           // 현재 상태 읽기
useAuthStore.getState().logout()  // 액션 실행
useAuthStore.setState({ user: null })  // 상태 직접 변경
```

---

# persist — localStorage 연동 ⭐️⭐️⭐️

```typescript
import { create }   from 'zustand';
import { persist }  from 'zustand/middleware';

type ThemeStore = {
  theme:    'light' | 'dark';
  setTheme: (theme: 'light' | 'dark') => void;
};

export const useThemeStore = create<ThemeStore>()(
  persist(
    (set) => ({
      theme:    'light',
      setTheme: (theme) => set({ theme }),
    }),
    {
      name: 'theme-storage',   // localStorage key 이름
      // partialize: (state) => ({ theme: state.theme })  // 일부만 저장
    },
  ),
);
```

```txt
persist 미들웨어:
  스토어 상태를 localStorage에 자동 저장
  새로고침해도 상태 유지
  앱 시작 시 localStorage에서 자동 복원

name: 'theme-storage':
  localStorage.getItem('theme-storage') 키로 저장됨

partialize:
  스토어 전체 중 저장할 필드만 선택
  비밀번호 같은 민감한 정보 제외할 때
```

---

# Context vs Zustand 선택 ⭐️⭐️⭐️

```txt
Context:
  ✅ React 내장 — 추가 라이브러리 없음
  ✅ 트리 구조와 함께 흐름 파악이 쉬움
  ❌ useCallback·useMemo 직접 챙겨야 함
  ❌ Provider 설정 필요
  적합: 인증 상태처럼 단순한 경우

Zustand:
  ✅ 설정이 단순 — create 하나로 끝
  ✅ Provider 없이 어디서든 접근
  ✅ 선택적 구독으로 리렌더 자동 최적화
  ✅ 컴포넌트 밖에서도 getState()로 접근
  ✅ persist·devtools 미들웨어 지원
  적합: 앱 전역 상태가 여러 개일 때, 성능이 중요할 때

보통:
  auth 상태 하나 → Context로 충분
  auth + 장바구니 + 알림 + 필터 → Zustand 권장
```