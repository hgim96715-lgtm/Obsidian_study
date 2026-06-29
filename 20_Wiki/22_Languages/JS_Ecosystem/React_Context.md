---
aliases: [Context, createContext, Provider, useContext]
tags:
  - React
  - NextJS
related:
  - "[[00_JS_Ecosystem_HomePage]]"
  - "[[JS_Operators]]"
  - "[[NextJS_AuthCache]]"
  - "[[React_useMemo_useCallback_useEffect]]"
---
# React_Context — 전역 상태를 트리 전체에 공급하기

> [!info] 
> createContext로 "채널"을 만들고 useContext로 그 채널을 구독하는 게 Context API의 전부다. useCallback/useMemo/useEffect는 Context 전용 훅이 아니라 일반 React 훅인데, Context Provider를 만들 때 생기는 특정 문제(불필요한 리렌더, 마운트 시 초기화)를 풀기 위해 거의 항상 같이 등장한다.

---

# Hook이란 무엇인가 — 간단히 ⭐️

```txt
"use"로 시작하는 함수들 — 함수형 컴포넌트 안에서 React의 내부 기능
(상태 저장, 생명주기, 컨텍스트 구독 등)에 접근하게 해주는 통로

  useState     컴포넌트 하나의 지역 상태
  useEffect    렌더링 이후에 실행되는 부수효과(side effect)
  useContext   Context 채널을 구독
  useCallback  함수를 재사용(메모이제이션)
  useMemo      계산 결과를 재사용(메모이제이션)

⚠️ 규칙: 컴포넌트(또는 다른 Hook) 함수의 최상위에서만 호출 — if/for 안에서 호출 금지
   (호출 순서로 내부 상태를 추적하는 구조라서, 조건에 따라 호출 여부가 바뀌면 추적이 깨짐)
```

# ContextAPI 흐름도

| 노드                | 의미                                     |
| ----------------- | -------------------------------------- |
| **props 위**       | App이 user를 Page·FeedList에 **계단식**으로 넘김 |
| **createContext** | 채널(그릇) 정의                              |
| **AuthProvider**  | 트리를 **감싸** value 공급 — 막는 게 아님          |
| **Page**          | 중간 컴포넌트 — **user props 안 넘김**          |
| **FeedList**      | 깊은 자손                                  |
| **useAuth**       | `useContext`로 **여기서** user 꺼냄          |

```mermaid-beautiful
%%{init: {'theme':'base', 'themeVariables': {'fontSize':'16px', 'primaryTextColor':'#0f172a', 'lineColor':'#2563eb', 'primaryColor':'#ffffff', 'primaryBorderColor':'#2563eb', 'clusterBkg':'#eff6ff', 'clusterBorder':'#2563eb', 'titleColor':'#0f172a', 'edgeLabelBackground':'#ffffff'}}}%%
flowchart TB
    subgraph props["props — user 계단 전달"]
        direction LR
        A[App] -->|user| B[Page] -->|user| C[FeedList]
    end

    props ==>|Context| ctx

    subgraph ctx["Context — Provider가 감싸 공급"]
        direction TB
        D[createContext]
        subgraph provider[AuthProvider]
            direction TB
            E[Page]
            F[FeedList]
            G[useAuth]
            E --> F --> G
        end
        D --> provider
    end

    classDef node fill:#ffffff,stroke:#2563eb,color:#0f172a,stroke-width:2px
    class A,B,C,D,E,F,G node
```

---

# createContext + useContext — Context API의 진짜 짝 ⭐️⭐️⭐️

```txt
props로 값을 일일이 내려보내지 않고("prop drilling"), 트리 어디서든 바로 꺼내 쓰게 하는 방법
```

```tsx
// 1. 채널 만들기
import { createContext, useContext } from 'react';

const ThemeContext = createContext<'light' | 'dark' | null>(null);

// 2. 트리 상위에서 값을 공급(Provider)
function App() {
  return (
    <ThemeContext.Provider value="dark">
      <Page /> {/* Page 안의 어떤 깊이의 자손도 이 값을 바로 꺼낼 수 있음 */}
    </ThemeContext.Provider>
  );
}

// 3. 트리 하위에서 값을 구독(useContext)
function Page() {
  const theme = useContext(ThemeContext); // 'dark' — props로 안 받아도 바로 접근
  return <div>{theme}</div>;
}
```

|함수|역할|
|---|---|
|`createContext(기본값)`|"채널"을 만듦 — 이 채널에 누가 값을 흘려보낼지(Provider)와는 별개|
|`<XxxContext.Provider value={...}>`|이 컴포넌트 아래 트리 전체에 value를 공급|
|`useContext(XxxContext)`|가장 가까운 상위 Provider가 공급한 value를 구독|

```txt
createContext와 useContext만 있으면 Context API의 핵심은 끝임
이 둘이 진짜 "짝"인 이유: 하나는 채널을 만들고(정의), 하나는 그 채널을 구독함(소비) — 1:1 대응
```

---

# Provider 컴포넌트의 기본 틀 ⭐️⭐️⭐️⭐️

```tsx
export function XxxProvider({ children }: { children: React.ReactNode }) {
  // ... 이 Context만의 상태/로직 ...
  const value = useMemo(() => ({ /* ... */ }), [ /* ... */ ]);

  return <XxxContext.Provider value={value}>{children}</XxxContext.Provider>;
}
```

```txt
어떤 Context든(인증, 테마, 저장된 항목 등 도메인이 뭐든) Provider 컴포넌트는 이 모양을 그대로 따름:
  매개변수는 항상 { children }: { children: React.ReactNode } 하나뿐 —
  이 Provider로 감싸지는 모든 자손을 그대로 받아서, 마지막에 그 children을 다시 그려줌

  안에서 어떤 상태(useState)를 들고 있고, 어떤 함수(useCallback)를 만들고,
  그것들을 어떻게 묶어서 value(useMemo)로 흘려보내는지는 도메인마다 다 다름 —
  바뀌는 건 "내용"뿐, "틀" 자체는 항상 같음

→ 아래 AuthProvider도, 그 다음에 나오는 SavedItemsProvider도 전부 이 틀 위에 지어진 것
```

---

# 왜 useMemo / useCallback / useEffect가 Provider에서 같이 보이는가 ⭐️⭐️⭐️

```txt
이 셋은 Context 전용 훅이 아님 — Context와 무관한 코드에서도 항상 등장하는 범용 훅임
다만 "값을 흘려보내는 Provider"를 직접 만들 때 생기는 문제들을 풀기 위해 자연스럽게 같이 쓰이게 됨

각 훅 자체가 무엇을 하는지/언제 쓰는지(Context와 무관한 일반론)는 [[React_useMemo_useCallback_useEffect]] 참고
여기서는 "Context Provider를 만들 때 왜 하필 이 셋이 같이 필요해지는가"에만 집중
```

## useMemo — value 객체가 매 렌더마다 "새 것"이 되는 문제 ⭐️⭐️⭐️

```tsx
// value를 그냥 객체 리터럴로 넘기면
return <AuthContext.Provider value={{ user, isLoading }}>{children}</AuthContext.Provider>;
```

```txt
Provider가 리렌더될 때마다 { user, isLoading } 는 매번 "새로 만들어진 객체"임
(user/isLoading 값 자체가 안 바뀌어도, { } 리터럴은 항상 새로운 참조)
→ useContext로 이 값을 구독하는 모든 자손 컴포넌트가, 의미 있는 변화가 없어도 전부 리렌더됨

useMemo(() => ({ user, isLoading }), [user, isLoading]) 로 감싸면:
  user/isLoading 이 실제로 바뀔 때만 새 객체를 만들고, 그 외엔 이전 객체를 그대로 재사용
  → Context를 구독하는 자손들의 불필요한 리렌더를 막음
```

## useCallback — 함수도 매 렌더마다 "새 것"이 되는 문제 ⭐️⭐️

```txt
value 안에 함수(예: clearSession, refreshUser)를 넣어야 한다면, 그 함수도 똑같은 문제를 가짐
일반 함수 선언은 렌더마다 새로 만들어지므로, useMemo의 의존성 배열에 넣어도 매번 "바뀐 것"으로 보임

useCallback(() => { ... }, [의존성])으로 그 함수 자체를 감싸두면
의존성이 안 바뀌는 한 같은 함수 참조를 재사용 → useMemo가 제대로 "안 바뀜"을 인식할 수 있게 됨

→ useMemo와 useCallback은 이 패턴에서 거의 항상 같이 다니는 이유가 이것:
  useMemo가 value 객체를 안정시키려면, 그 안에 들어가는 함수들도 먼저 안정되어 있어야 함
```

## useEffect — Provider가 "마운트될 때 한 번" 해야 하는 일 ⭐️⭐️

```txt
Context 자체와는 무관 — "컴포넌트가 화면에 나타난 뒤 한 번 실행해야 하는 부수효과"를 위한 일반 훅
Provider 안에서는 보통 "이 앱이 시작될 때 한 번 인증 상태를 확인" 같은 초기화 로직에 씀
```

---

# AuthProvider 구현 예시 — 한 줄씩 ⭐️⭐️⭐️

```tsx
const [user, setUser] = useState<ApiAuthUser | null>(null);
const [isLoading, setIsLoading] = useState(true);
```

```txt
user/isLoading — 이 Provider가 들고 있는 "진짜" 상태. 나머지(value, clearSession 등)는
다 이 둘을 다루기 위한 도구일 뿐
```

```tsx
const clearSession = useCallback(() => {
  clearAuthStorage();
  setUser(null);
}, []);
```

```txt
의존성 배열이 빈 배열([])인 이유: 이 함수 안에서 바깥의 "바뀌는 값"을 참조하는 게 없음
(clearAuthStorage는 외부 함수고, setUser는 React가 항상 같은 참조를 보장해주는 특수한 함수라서
 의존성에 안 넣어도 안전함) → 한 번 만들어진 뒤로는 계속 같은 함수 참조를 재사용함
```

```tsx
const refreshUser = useCallback(async () => {
  if (!getApiAccessToken()) {
    setUser(null);
    return;
  }
  try {
    const me = await fetchMe();
    setUser(me);
  } catch {
    clearSession();
  }
}, [clearSession]);
```

```txt
의존성 배열에 clearSession이 들어간 이유: 이 함수 안에서 clearSession을 호출하고 있어서
(clearSession이 바뀌면 refreshUser도 새로 만들어져야 항상 최신 clearSession을 참조함)
→ 다행히 clearSession은 위에서 빈 배열로 고정돼있어서, 사실상 refreshUser도 한 번만 만들어짐
```

```tsx
useEffect(() => {
  let cancelled = false;
  async function init() {
    await refreshUser();
    if (!cancelled) setIsLoading(false);
  }
  init();
  return () => {
    cancelled = true;
  };
}, [refreshUser]);
```

```txt
Provider가 처음 마운트될 때 init()을 한 번 실행 → refreshUser가 끝나면 isLoading을 false로

return () => { cancelled = true; } 가 정확히 이 useEffect의 클린업 함수임
(컴포넌트가 사라지거나, 의존성[refreshUser]이 바뀌어서 이 effect가 다시 실행되기 직전에 호출됨)

cancelled 플래그가 있는 이유: init()의 await refreshUser()가 끝나기 전에 컴포넌트가
사라지면(예: 페이지 이동), 이미 사라진 컴포넌트의 setIsLoading을 호출하지 않도록 막는 안전장치
→ 이 "cancelled 플래그" 패턴 자체는 Context와 무관한 범용 useEffect 패턴이라
  [[React_useMemo_useCallback_useEffect]]의 useEffect 섹션에 일반화해서 정리해둠
```

```tsx
const value = useMemo(
  () => ({ user, isLoading, setUser, clearSession, refreshUser }),
  [user, isLoading, clearSession, refreshUser],
);
```

```txt
지금까지 만든 모든 조각(상태값 + 안정된 함수들)을 하나의 value 객체로 묶고,
그 객체 자체도 useMemo로 안정시킴 — 이 한 줄이 위에서 설명한 모든 useCallback의 "목적지"
```

```tsx
export function useAuth(): AuthContextValue {
  const ctx = useContext(AuthContext);
  if (!ctx) {
    throw new Error('useAuth는 AuthProvider 안에서만 사용할 수 있습니다.');
  }
  return ctx;
}
```

```txt
useContext(AuthContext)만 하면 Provider 밖에서 쓰일 경우 ctx가 null일 수 있음(createContext의 기본값)
→ null이면 바로 에러를 던져서, "Provider로 안 감쌌다"는 실수를 그 자리에서 바로 알게 해줌
   (조용히 undefined로 진행돼서 한참 뒤에 엉뚱한 곳에서 에러가 나는 것보다 훨씬 디버깅하기 쉬움)
```

---

# 실전 — useAuth로 보호된 페이지 만들기 ⭐️⭐️⭐️⭐️

```tsx
const { user, isLoading } = useAuth();
const router = useRouter();

useEffect(() => {
  if (!isLoading && !user) {
    router.replace('/login?next=/users/me');
  }
}, [isLoading, user, router]);
```

```txt
이 effect가 하는 일: "확인이 끝났는데 로그인된 사용자가 없다면" 로그인 페이지로 보냄
(!isLoading/!user 같은 부정 표현을 정확히 읽는 법은 [[JS_Operators]]의 "! (논리 NOT)" 참고)

⚠️ isLoading을 먼저 확인해야 하는 이유:
  isLoading이 true인 동안(아직 /me 응답을 기다리는 중)에는 user가 당연히 아직 null임
  이 시점에 곧바로 "user가 없으니 비로그인"이라고 판단하면, 실제로는 로그인된 사용자인데도
  응답이 도착하기 전이라는 이유만으로 잘못 로그인 페이지로 튕겨버리는 버그가 생김
  (왜 새로고침 시 이 "확인 시간"이 필요한지는 [[NextJS_AuthCache]]의 "/me 패턴" 참고)

  → isLoading이 끝나길(false) 먼저 기다린 뒤에야 user 유무로 진짜 판단을 내림

의존성 배열에 router가 들어가는 이유:
  effect 안에서 router.replace를 호출하므로 — useRouter()가 보통 안정적인 참조를 주긴 하지만,
  effect 안에서 쓰는 외부 값은 의존성 배열에 명시하는 게 정석(린트 규칙이 강제하기도 함)
```

---

# 실전 예시 2 — 다른 Context를 쓰는 Provider (Context 합성) ⭐️⭐️⭐️⭐️

```txt
Provider 안에서 다른 Context의 훅을 그대로 가져다 쓸 수 있음 — 이걸 "Context 합성"이라고 부름
아래 SavedItemsProvider는 useAuth()(AuthProvider가 공급하는 값)를 가져다가
"로그인한 사용자가 바뀌면 저장 목록을 다시 불러온다"는 로직을 만듦

→ 이 패턴이 동작하려면 컴포넌트 트리에서 SavedItemsProvider가 AuthProvider보다
  "안쪽"(자손)에 있어야 함 — 더 안쪽 Provider가 더 바깥 Provider의 값을 구독하는 구조
```

```tsx
'use client';
import { useAuth } from '@/components/auth/AuthProvider';
import { fetchSavedItems } from '@/lib/api';
import {
  createContext,
  useCallback,
  useContext,
  useEffect,
  useMemo,
  useState,
} from 'react';

type SavedItemsContextValue = {
  isSaved: (postId: string) => boolean;
  markSaved: (postId: string) => void;
};

const SavedItemsContext = createContext<SavedItemsContextValue | null>(null);

export function SavedItemsProvider({ children }: { children: React.ReactNode }) {
  const { user } = useAuth(); // 다른 Context를 그대로 구독
  const [savedIds, setSavedIds] = useState<Set<string>>(new Set());

  useEffect(() => {
    if (!user) {
      setSavedIds(new Set());
      return;
    }
    let cancelled = false;
    fetchSavedItems()
      .then((items) => {
        if (!cancelled) setSavedIds(new Set(items.map((i) => i.postId)));
      })
      .catch(() => {
        if (!cancelled) setSavedIds(new Set());
      });
    return () => {
      cancelled = true;
    };
  }, [user?.id]); // user 객체 전체가 아니라 id만 — 자세한 이유는 아래 참고

  const markSaved = useCallback((postId: string) => {
    setSavedIds((prev) => new Set(prev).add(postId)); // 함수형 업데이트 — 자세한 이유는 아래 참고
  }, []);

  const isSaved = useCallback(
    (postId: string) => savedIds.has(postId),
    [savedIds],
  );

  const value = useMemo(() => ({ isSaved, markSaved }), [isSaved, markSaved]);

  return <SavedItemsContext.Provider value={value}>{children}</SavedItemsContext.Provider>;
}
```

```txt
이 예시에서 AuthProvider와 달라 보이는 부분들 — 전부 의미 있는 선택임:

  [user?.id]를 의존성으로 둔 이유
    user 객체 전체를 의존성에 두면, 닉네임 등 다른 필드만 바뀌어도 이 effect가 다시 돎
    "로그인한 사용자 자체가 바뀌었는가"만 보고 싶으므로 id(원시값)만 뽑아서 의존성에 둠
    (이 패턴 자체의 일반론은 [[React_useMemo_useCallback_useEffect]]의
     "의존성을 객체 대신 원시값으로 좁히기" 참고)

  markSaved가 setSavedIds(prev => ...)를 쓰는 이유
    savedIds를 함수 본문에서 직접 읽지 않고, "이전 값을 받아서 다음 값을 반환하는 함수"를 넘김
    → savedIds를 useCallback 의존성에 안 넣어도 항상 최신 상태를 정확히 반영함(빈 배열로 충분)
    (이 패턴의 일반론은 [[React_useMemo_useCallback_useEffect]]의
     "함수형 업데이트로 useCallback 의존성 줄이기" 참고)

  isLoading이 따로 없는 이유
    AuthProvider는 "확인 중인지"를 알아야 보호된 페이지에서 리다이렉트 여부를 정확히 판단할 수 있어서
    isLoading이 꼭 필요했음. SavedItemsProvider는 그런 게이팅이 필요 없음 —
    아직 안 불러왔으면 그냥 빈 Set(전부 isSaved=false)으로 보여줘도 자연스러운 기본 상태이기 때문
    → 모든 Context가 isLoading을 가져야 하는 건 아님, "그 값이 없으면 잘못된 판단을 하게 되는가"로 판단
```

---

# Provider 없을 때 — throw vs 안전한 기본값 ⭐️⭐️⭐️⭐️

```txt
useAuth()는 Provider 밖에서 쓰이면 throw 했는데, 아래 useSavedItems()는 다르게 동작함
둘 다 "맞는" 선택임 — 그 Context가 얼마나 필수적인지에 따라 전략이 달라짐
```

```tsx
const noop = () => {}; // "아무 일도 안 하는 함수" — no-operation의 줄임말, 흔히 쓰는 관용적 이름

export function useSavedItems() {
  const ctx = useContext(SavedItemsContext);
  if (!ctx) {
    return {
      isSaved: () => false, // Provider가 없으면 "아무것도 저장 안 됨"으로 안전하게 취급
      markSaved: noop,       // 저장 시도해도 조용히 무시
    };
  }
  return ctx;
}
```

|전략|언제 쓰나|예시|
|---|---|---|
|Provider 없으면 throw|이 Context 없이는 정상 동작이 불가능 — 누락 자체가 버그|`useAuth()` — 인증 정보 없이는 의미 있는 동작을 할 수 없음|
|Provider 없으면 안전한 기본값 반환|이 기능이 없어도 나머지 화면은 정상 동작해야 함 — 선택적 기능|`useSavedItems()` — 저장 기능이 없어도 피드는 그냥 보여지면 됨|

```txt
판단 기준: "이 Context 없이 쓰이면, 그게 진짜 버그인가, 아니면 그 기능만 빠진 정상적인 상황인가"
  버그라면(예: 인증 없이 사용자 정보를 써야 하는 컴포넌트)        → throw로 그 자리에서 바로 드러나게
  정상적인 상황이라면(예: 미리보기 화면이라 저장 기능 자체가 없음) → 안전한 기본값으로 조용히 우회

throw가 항상 "더 안전한" 선택은 아님 — 정말 선택적인 기능까지 throw로 막아두면,
그 Provider 없이 컴포넌트를 재사용하고 싶을 때마다 매번 가짜 Provider로 감싸야 하는 불편함이 생김
```

---

# 이 코드, 범용적으로 쓸 수 있나 ⭐️

```txt
패턴 자체(Provider + useAuth + 마운트 시 refreshUser + 401 시 clearSession)는 완전히 범용적임
다른 프로젝트에 그대로 가져가도 되는 구조

⚠️ 다만 clearAuthStorage라는 이름이 [[NextJS_TokenStorage]]에서 만든
   clearApiAccessToken과 이름이 다름 — 의도적으로 리네임한 거라면 그 노트도 맞춰서 바꿀 것,
   아니라면 실제 코드에 clearAuthStorage가 진짜 존재하는지 확인 필요
```

---

# 한눈에

| 키워드                                     | 역할                                                        |
| --------------------------------------- | --------------------------------------------------------- |
| `createContext`                         | 값을 흘려보낼 "채널" 정의                                           |
| `<Context.Provider value={...}>`        | 그 채널에 실제 값을 공급                                            |
| `useContext(Context)`                   | 가장 가까운 Provider의 값을 구독                                    |
| `{ children }: { children: ReactNode }` | 모든 Provider 컴포넌트가 공유하는 매개변수 틀                             |
| `useMemo`                               | Provider의 value 객체를 매 렌더마다 새로 안 만들고 재사용 — 자손 불필요 리렌더 방지   |
| `useCallback`                           | value에 들어갈 함수들을 안정된 참조로 유지 — useMemo가 제대로 동작하기 위한 전제조건    |
| `useEffect`                             | Provider 마운트 시 한 번 실행해야 하는 초기화 로직(여기선 refreshUser)        |
| Context 합성                              | Provider 안에서 다른 Context의 훅을 가져다 쓰는 것 — 트리 순서(바깥/안쪽)가 중요   |
| Provider 밖에서 `useContext` 호출            | 기본값(보통 `null`)이 나옴 — throw(필수 기능) 또는 안전한 기본값(선택적 기능)으로 방어 |
| `noop`                                  | 아무 일도 안 하는 함수 — Provider 없을 때의 안전한 기본 동작으로 흔히 씀           |