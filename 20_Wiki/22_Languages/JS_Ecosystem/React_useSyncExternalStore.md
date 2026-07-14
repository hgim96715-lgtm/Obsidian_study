---
aliases:
  - React
  - 외부 스토어 구독
tags:
  - React
related:
  - "[[00_JS_Ecosystem_HomePage]]"
  - "[[React_useMemo_useCallback_useEffect]]"
  - "[[React_Suspense]]"
  - "[[NextJS_Concept]]"
  - "[[JS_WebStorage]]"
---
# React_useSyncExternalStore — 외부 스토어 구독

> [!info] 
> React 바깥에 있는 값(localStorage, window 이벤트, 외부 라이브러리 등)을 React 렌더링과 안전하게 동기화하는 훅.
>  `useEffect + useState`보다 concurrent mode에서 안전하다.

---

# 왜 필요한가 ⭐️⭐️⭐️⭐️

```txt
문제: localStorage, window 이벤트 같은 값은 React 바깥에 있음
  → 값이 바뀌어도 React가 자동으로 인식 못 함
  → 컴포넌트가 바뀐 값을 반영하려면 직접 구독 로직이 필요

기존 방식 — useEffect + useState:
  useEffect(() => {
    const handler = () => setState(getValue());
    window.addEventListener('storage', handler);
    return () => window.removeEventListener('storage', handler);
  }, []);

문제점:
  React 18 concurrent mode에서 "tearing" 발생 가능
  → 같은 렌더링 안에서 다른 시점의 값을 읽어 UI가 불일치하는 현상
  SSR(서버사이드 렌더링)에서 hydration 불일치 위험

useSyncExternalStore:
  React가 공식으로 제공하는 외부 스토어 구독 방법
  concurrent mode에서 tearing 방지 보장
  getServerSnapshot으로 SSR/hydration 불일치 방지
```

---

# 기본 형태 ⭐️⭐️⭐️⭐️

```typescript
import { useSyncExternalStore } from 'react';

const value = useSyncExternalStore(
  subscribe,           // 외부 스토어 구독 함수
  getSnapshot,         // 현재 값을 반환하는 함수 (브라우저)
  getServerSnapshot,   // 서버에서 반환할 값 (선택, SSR용)
);
```

|파라미터|타입|역할|
|---|---|---|
|`subscribe`|`(callback) => () => void`|외부 값 변경 시 callback을 호출하도록 구독 등록. 반환값은 구독 해제 함수|
|`getSnapshot`|`() => T`|현재 스토어의 값을 반환. 렌더링마다 호출됨|
|`getServerSnapshot`|`() => T`|서버 렌더링 시 사용할 값. 없으면 SSR에서 에러 가능|

```txt
subscribe의 역할:
  외부 스토어가 바뀌면 React에게 "다시 렌더링해"를 알려줌
  callback = React가 넘겨준 리렌더 트리거
  반환값(언마운트 시 호출)으로 이벤트 리스너를 제거

getSnapshot 주의:
  렌더링마다 호출되므로 순수 함수여야 함
  매번 새 객체/배열을 반환하면 무한 리렌더 발생
  → 원시값(string, number, boolean) 또는 참조가 변하지 않는 값을 반환
```

---

# localStorage 구독 패턴 ⭐️⭐️⭐️⭐️

```typescript
// hooks/useLocalStorage.ts
import { useSyncExternalStore } from 'react';

function subscribe(callback: () => void) {
  // storage 이벤트 = 다른 탭에서 localStorage가 바뀔 때 발생
  window.addEventListener('storage', callback);
  return () => window.removeEventListener('storage', callback);
}

export function useLocalStorage(key: string): string | null {
  return useSyncExternalStore(
    subscribe,
    () => localStorage.getItem(key),  // 브라우저: 실제 값 읽기
    () => null,                        // 서버: null 반환 (localStorage 없음)
  );
}
```

```typescript
// 사용
function AuthStatus() {
  const token = useLocalStorage('accessToken');

  if (!token) return <LoginButton />;
  return <UserMenu />;
}
```

```txt
왜 getServerSnapshot이 () => null인가:
  서버(Node.js)에는 localStorage가 없음
  SSR 시 null을 반환해서 "로그인 안 된 상태"로 렌더링
  브라우저에서 hydration 되면 실제 값으로 교체됨

storage 이벤트의 한계:
  같은 탭에서 localStorage.setItem()을 해도 storage 이벤트가 발생 안 함
  → 같은 탭에서 값을 바꾸고 즉시 반영하려면 커스텀 이벤트가 필요 (아래 참고)
```

---

# 같은 탭에서도 반응하는 패턴 ⭐️⭐️⭐️

```typescript
// localStorage를 쓰면서 같은 탭에도 알려주는 유틸
function setLocalStorage(key: string, value: string | null) {
  if (value === null) {
    localStorage.removeItem(key);
  } else {
    localStorage.setItem(key, value);
  }
  // storage 이벤트는 다른 탭에만 발생 → 커스텀 이벤트로 같은 탭에도 알림
  window.dispatchEvent(new StorageEvent('storage', { key }));
}

function subscribe(callback: () => void) {
  window.addEventListener('storage', callback);
  return () => window.removeEventListener('storage', callback);
}

export function useLocalStorage(key: string): string | null {
  return useSyncExternalStore(
    subscribe,
    () => localStorage.getItem(key),
    () => null,
  );
}
```

```typescript
// 사용 — setLocalStorage로 저장하면 같은 탭에도 반응
function OAuthCallbackContent() {
  const searchParams = useSearchParams();
  const router = useRouter();

  useEffect(() => {
    const token = searchParams.get('accessToken');
    if (token) {
      setLocalStorage('accessToken', token);  // ← 커스텀 setter
      router.replace(searchParams.get('next') ?? '/');
    }
  }, [searchParams, router]);
}

function Header() {
  const token = useLocalStorage('accessToken');  // token 저장되면 자동 반응
  return token ? <UserMenu /> : <LoginButton />;
}
```

---

# window 이벤트 구독 패턴 ⭐️⭐️⭐️

```typescript
// 온라인/오프라인 상태
function subscribe(callback: () => void) {
  window.addEventListener('online', callback);
  window.addEventListener('offline', callback);
  return () => {
    window.removeEventListener('online', callback);
    window.removeEventListener('offline', callback);
  };
}

function useOnlineStatus(): boolean {
  return useSyncExternalStore(
    subscribe,
    () => navigator.onLine,  // 브라우저
    () => true,              // 서버 (항상 온라인으로 가정)
  );
}

// 사용
function NetworkBanner() {
  const isOnline = useOnlineStatus();
  if (isOnline) return null;
  return <div className="banner">오프라인 상태입니다</div>;
}
```

```typescript
// 화면 너비 구독
function subscribe(callback: () => void) {
  window.addEventListener('resize', callback);
  return () => window.removeEventListener('resize', callback);
}

function useWindowWidth(): number {
  return useSyncExternalStore(
    subscribe,
    () => window.innerWidth,
    () => 0,  // 서버: 기본값
  );
}
```

---

# SSR / Next.js 주의사항 ⭐️⭐️⭐️

```typescript
// getServerSnapshot 없으면 Next.js App Router에서 경고 또는 에러
useSyncExternalStore(
  subscribe,
  () => localStorage.getItem('token'),
  // getServerSnapshot 생략 → 서버에서 실행 시 에러
);

// ✅ 항상 getServerSnapshot 제공
useSyncExternalStore(
  subscribe,
  () => localStorage.getItem('token'),
  () => null,  // 서버: localStorage 없으므로 null
);
```

```txt
Next.js App Router에서 useSyncExternalStore 사용 조건:
  'use client' 선언 필수 (Client Component에서만 동작)
  getServerSnapshot 반드시 제공 (SSR 지원)

hydration 불일치 방지:
  서버: getServerSnapshot() → null (또는 기본값)
  클라이언트 첫 렌더: getServerSnapshot() → null (서버와 일치하게)
  hydration 완료 후: getSnapshot() → 실제 localStorage 값

  → 서버와 클라이언트 첫 렌더 결과가 같아야 hydration 에러가 없음
```

---

# useEffect + useState vs useSyncExternalStore ⭐️⭐️⭐️

```typescript
// ❌ 기존 방식 — concurrent mode에서 tearing 가능
function useLocalStorageOld(key: string) {
  const [value, setValue] = useState<string | null>(() =>
    typeof window !== 'undefined' ? localStorage.getItem(key) : null
  );

  useEffect(() => {
    const handler = () => setValue(localStorage.getItem(key));
    window.addEventListener('storage', handler);
    return () => window.removeEventListener('storage', handler);
  }, [key]);

  return value;
}

// ✅ 새 방식 — concurrent mode 안전
function useLocalStorageNew(key: string) {
  return useSyncExternalStore(
    (cb) => {
      window.addEventListener('storage', cb);
      return () => window.removeEventListener('storage', cb);
    },
    () => localStorage.getItem(key),
    () => null,
  );
}
```

| |`useEffect + useState`|`useSyncExternalStore`|
|---|---|---|
|concurrent mode tearing|발생 가능|방지 보장|
|SSR hydration|직접 처리 필요|getServerSnapshot으로 내장 지원|
|코드 길이|길어짐|간결|
|React 공식 권장|이전 방식|React 18+ 권장|

---

# 한눈에

```txt
useSyncExternalStore(subscribe, getSnapshot, getServerSnapshot?)

subscribe(callback) → 구독 해제 함수:
  외부 값이 바뀔 때 callback() 호출
  반환값으로 이벤트 리스너 제거

getSnapshot() → 현재 값:
  매 렌더링마다 호출됨 (순수 함수여야 함)
  매번 새 객체 반환 금지 → 무한 리렌더

getServerSnapshot() → 서버에서 반환할 값:
  SSR/Next.js에서 필수
  localStorage 없는 환경 → null/기본값

주요 사용 사례:
  localStorage 읽기 (accessToken 등)
  window.addEventListener (online, resize, scroll)
  외부 상태 라이브러리 연동 (Redux, Zustand 내부에서 사용)

storage 이벤트:
  다른 탭에서 변경 시에만 발생
  같은 탭 반응 필요 → StorageEvent를 수동으로 dispatch

Next.js에서:
  'use client' 필수
  getServerSnapshot 반드시 제공 (hydration 불일치 방지)
```