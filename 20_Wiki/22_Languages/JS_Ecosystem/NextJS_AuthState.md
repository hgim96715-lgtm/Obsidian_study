---
aliases:
  - BFF
  - Cross-Origin Cookie
  - GET /me
  - SSR CSR 인증
tags:
  - NextJS
related:
  - "[[00_JS_Ecosystem_HomePage]]"
  - "[[Auth_Concept]]"
  - "[[NextJS_TokenStorage]]"
---
# NextJS_AuthState — 로그인 유저 상태 관리

>[!info]
>로그인한 유저 정보(`{ id, email, role }`)를 앱 전체에서 공유하는 방법. 
>Access Token 저장 → [[NextJS_TokenStorage]], 컴포넌트 간 상태 공유는 Zustand·Context 중 선택. 
>앱 시작 시 `/auth/me`로 유저 정보를 복구한다.

---

# 왜 유저 상태 관리가 필요한가 ⭐️⭐️⭐️⭐️

```txt
로그인 후 사용하는 곳:
  Header → "홍길동님 환영합니다"
  포스트 → 내 글이면 수정/삭제 버튼 표시
  라우트 → 비로그인이면 /login으로 리다이렉트

문제:
  로그인 유저 정보를 필요한 모든 컴포넌트에서 접근해야 함
  props로 넘기면 depth가 깊어질수록 불편 (prop drilling)
  → 전역 상태(Global State)로 관리
```

---

# 유저 상태에 담는 것 ⭐️⭐️⭐️⭐️

```typescript
// 로그인 유저 정보 타입
type AuthUser = {
  id:       string;
  email:    string;
  nickname: string;
  role:     'user' | 'admin';
  image:    string | null;
};

// 인증 상태 전체
type AuthState = {
  user:         AuthUser | null;  // null = 비로그인
  isLoading:    boolean;          // 초기 /auth/me 호출 중
};
```

```txt
user가 null인 경우:
  ① 아직 /auth/me 호출 전 (isLoading: true)
  ② /auth/me 실패 = 비로그인 (isLoading: false)

user가 있는 경우:
  로그인 완료 (isLoading: false)
```

---

# Zustand로 관리 ⭐️⭐️⭐️⭐️

```bash
pnpm --filter web add zustand
```

```typescript
// store/authStore.ts
import { create } from 'zustand';

type AuthUser = { id: string; email: string; nickname: string; role: string; };

type AuthStore = {
  user:      AuthUser | null;
  isLoading: boolean;
  setUser:   (user: AuthUser | null) => void;
  setLoading:(loading: boolean) => void;
  logout:    () => void;
};

export const useAuthStore = create<AuthStore>((set) => ({
  user:      null,
  isLoading: true,  // 앱 시작 시 /auth/me 호출 전까지 로딩

  setUser:    (user)    => set({ user, isLoading: false }),
  setLoading: (loading) => set({ isLoading: loading }),

  logout: () => {
    set({ user: null });
    removeApiAccessToken();  // 토큰도 제거
  },
}));
```

```typescript
// 컴포넌트에서 사용
'use client';
import { useAuthStore } from '@/store/authStore';

function Header() {
  const user = useAuthStore(s => s.user);

  if (!user) return <a href="/login">로그인</a>;
  return <span>{user.nickname}님</span>;
}
```

---

# 앱 시작 시 유저 복구 ⭐️⭐️⭐️⭐️

```txt
새로고침하면:
  Access Token (메모리 방식) → 소멸
  Zustand 상태 → 소멸 (메모리이므로)
  → user = null, isLoading = true 로 초기화

복구 방법:
  앱이 시작될 때 /auth/me 호출
  → 서버가 Refresh Token(쿠키)으로 새 Access Token 발급
  → 유저 정보를 Zustand에 저장
```

```typescript
// components/AuthInitializer.tsx
'use client';
import { useEffect }    from 'react';
import { fetchMe }      from '@/lib/api';
import { useAuthStore } from '@/store/authStore';

export function AuthInitializer() {
  const { setUser, setLoading } = useAuthStore();

  useEffect(() => {
    fetchMe()
      .then(user => setUser(user))
      .catch(() => setLoading(false));  // 비로그인이면 로딩만 끔
  }, []);

  return null;  // UI 없음, 사이드이펙트만
}
```

```typescript
// app/layout.tsx — 앱 전체에 AuthInitializer 마운트
import { AuthInitializer } from '@/components/AuthInitializer';

export default function RootLayout({ children }: { children: React.ReactNode }) {
  return (
    <html>
      <body>
        <AuthInitializer />  {/* 앱 시작 시 유저 복구 */}
        {children}
      </body>
    </html>
  );
}
```

---

# 로그인·로그아웃 흐름 ⭐️⭐️⭐️⭐️

```typescript
// 로그인 버튼 클릭
async function handleLogin(email: string, password: string) {
  const data = await login(email, password);
  // login() 내부에서 setApiAccessToken(data.accessToken) 실행
  useAuthStore.getState().setUser(data.user);  // Zustand에 유저 저장
  router.push('/');
}

// 로그아웃 버튼 클릭
function handleLogout() {
  useAuthStore.getState().logout();  // user = null + 토큰 제거
  router.push('/login');
}
```

---

# 보호 라우트 패턴 ⭐️⭐️⭐️

```typescript
// hooks/useRequireAuth.ts
'use client';
import { useEffect }    from 'react';
import { useRouter }    from 'next/navigation';
import { useAuthStore } from '@/store/authStore';

export function useRequireAuth() {
  const { user, isLoading } = useAuthStore();
  const router = useRouter();

  useEffect(() => {
    if (!isLoading && !user) {
      router.replace('/login');  // 비로그인 → 로그인 페이지
    }
  }, [user, isLoading, router]);

  return { user, isLoading };
}

// 사용
function ProtectedPage() {
  const { user, isLoading } = useRequireAuth();

  if (isLoading) return <Spinner />;
  if (!user)     return null;  // redirect 중

  return <div>{user.nickname}의 페이지</div>;
}
```

---

# TokenStorage vs AuthState — 역할 구분

```txt
NextJS_TokenStorage:
  Access Token 자체를 어디에 저장하는가
  메모리 변수 or localStorage
  authToken.ts가 담당

NextJS_AuthState (이 파일):
  로그인한 유저 정보(id, email, role)를
  React 상태로 앱 전체에서 접근하는 방법
  Zustand store가 담당

둘은 함께 작동:
  로그인 성공
  → setApiAccessToken(token)  [TokenStorage]
  → setUser(user)             [AuthState]

  로그아웃
  → removeApiAccessToken()    [TokenStorage]
  → logout() → user = null    [AuthState]
```