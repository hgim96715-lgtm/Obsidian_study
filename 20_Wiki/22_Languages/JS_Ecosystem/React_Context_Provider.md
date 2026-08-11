---
aliases:
  - Context
  - createContext
  - Provider
  - useContext
tags:
  - React
  - NextJS
related:
  - "[[00_JS_Ecosystem_HomePage]]"
  - "[[NextJS_AuthState]]"
---
# React_Context_Provider — Context API

>[!info]
>Context = "이 트리 안 어디서든 꺼내 쓸 수 있는 공유 상자". 
>props를 중간 컴포넌트들을 거쳐 전달하지 않아도 된다. 
>로그인 유저, 테마, 친구 목록처럼 **여러 컴포넌트가 공통으로 필요한 데이터**에 사용.
> Context 안의 함수는 `useCallback`, value 객체는 `useMemo`로 안정화해야 불필요한 리렌더·useEffect 루프를 막는다. 
> 실전 AuthProvider → [[NextJS_AuthState]]

---

# Context가 해결하는 문제 ⭐️⭐️⭐️⭐️

```txt
Prop Drilling:
  UserAvatar에서 user가 필요한데
  Page → Layout → Sidebar → UserAvatar 순서로 전달해야 함

  Page(user) → Layout(user) → Sidebar(user) → UserAvatar ← 여기서만 실제 사용
                ↑ Layout은 user 안 씀  ↑ Sidebar도 안 씀

  중간 컴포넌트들이 "전달만을 위해" props를 받아야 함
  컴포넌트 구조가 바뀌면 중간 컴포넌트 props도 전부 수정해야 함
```

```txt
Context 사용 후:
  Page → Layout → Sidebar → UserAvatar
                              ↑
                         useAuth()로 바로 꺼냄 (중간 컴포넌트 무관)
```

---

# 세 가지 역할 ⭐️⭐️⭐️⭐️

```txt
createContext  →  "이런 데이터를 공유할 거야" 선언
Provider       →  "이 컴포넌트 아래에서 데이터 제공"
useContext     →  "Provider가 제공한 데이터 꺼내 쓰기"
```

세 가지가 항상 함께 만들어지고, 항상 이 순서로 동작합니다.

---

# 만드는 방법 — 커스텀 훅 패턴 ⭐️⭐️⭐️⭐️

Context 관련 코드는 파일 하나에 세 가지를 모아둡니다.

```tsx
// auth-context.tsx — createContext + Provider + 커스텀 훅 세트

type AuthContextValue = {
  user:   User | null;
  logout: () => void;
};

// ① createContext — null을 초기값으로, 타입은 명시
const AuthContext = createContext<AuthContextValue | null>(null);

// ② Provider 컴포넌트 — state를 들고 있고, value로 내려줌
export function AuthProvider({ children }: { children: ReactNode }) {
  const [user, setUser] = useState<User | null>(null);

  // ⚠️ useCallback 필요 — 아래 설명 참고
  const logout = useCallback(() => setUser(null), []);

  const value = useMemo(
    () => ({ user, logout }),
    [user, logout],
  );

  return (
    <AuthContext.Provider value={value}>
      {children}
    </AuthContext.Provider>
  );
}

// ③ 커스텀 훅 — Provider 바깥 호출 시 즉시 에러
export function useAuth() {
  const ctx = useContext(AuthContext);
  if (!ctx) throw new Error('AuthProvider 안에서만 사용할 수 있습니다.');
  return ctx;
}
```

```txt
왜 createContext 초기값을 null로 하는가:
  Provider 없이 useContext를 호출하면 초기값이 반환됨
  의미 있는 초기값을 주면 에러 없이 조용히 잘못된 값을 씀
  null로 두면 → if (!ctx) throw 로 즉시 잡을 수 있음

왜 커스텀 훅으로 감싸는가:
  useContext(AuthContext)를 직접 쓰면 null 체크를 매번 해야 함
  커스텀 훅 하나에서 null 체크를 끝내면 사용처에서는 그냥 useAuth() 로 끝남

⚠️ Context 안의 함수에는 useCallback이 필요한 이유:
  const logout = () => setUser(null);  ← useCallback 없음
  → AuthProvider가 렌더링될 때마다 logout이 새 함수로 생성
  → useMemo([user, logout])의 logout deps가 매번 바뀜
  → value 객체가 매번 새로 생성 → 구독 컴포넌트 전부 리렌더

  useEffect([logout])을 쓰는 자식이 있다면 더 심각:
  → logout이 새 함수 → effect 재실행 → 루프 발생 가능

  useCallback(() => setUser(null), []):
  deps []  = 절대 바뀌지 않는 함수 참조 보장
  setUser  = useState setter라 이미 stable → deps 불필요
```

---

# 쓰는 방법 ⭐️⭐️⭐️⭐️

Provider는 공유할 범위의 최상위에 감쌉니다.

```tsx
// layout.tsx — 앱 전체에서 로그인 정보 필요
export default function RootLayout({ children }: { children: ReactNode }) {
  return (
    <AuthProvider>  {/* ← 이 안의 모든 컴포넌트가 useAuth() 사용 가능 */}
      {children}
    </AuthProvider>
  );
}
```

```tsx
// 어디서든 꺼내 쓰기
function ProfileButton() {
  const { user, logout } = useAuth();  // 트리 어디서든 동작

  if (!user) return null;
  return <button onClick={logout}>{user.nickname} 로그아웃</button>;
}
```

```txt
Provider 위치 선택:
  앱 전체에서 필요 → layout.tsx 또는 루트 컴포넌트
  특정 페이지에서만 필요 → 그 페이지 컴포넌트에 감쌈
  여러 Provider가 필요하면 → 아래 "여러 Provider 중첩" 참고
```

---

# 여러 Provider 중첩 ⭐️⭐️⭐️⭐️

한 페이지에서 여러 Context가 필요할 때 Provider를 겹겹이 감쌉니다.

```tsx
export default function FriendsPage() {
  return (
    <FriendIdsProvider>        {/* 바깥: 친구 목록 */}
      <AvatarActionProvider>   {/* 안쪽: 아바타 클릭 메뉴 */}
        <FriendsPageInner />   {/* 실제 화면 */}
      </AvatarActionProvider>
    </FriendIdsProvider>
  );
}
```

```txt
FriendsPage에서 직접 useFriendIds()를 쓰면 안 되는 이유:
  FriendsPage가 렌더링될 때 아직 Provider가 없음
  Provider를 먼저 렌더링해야 그 안의 자손이 Context를 쓸 수 있음
  → Provider 설정을 담당하는 컴포넌트(FriendsPage)와
    Context를 실제로 쓰는 컴포넌트(FriendsPageInner)를 분리

순서 규칙:
  안쪽 Provider가 바깥 Context를 useContext로 읽을 수 있음
  AvatarActionProvider가 FriendIdsContext를 내부에서 쓴다면
  → FriendIdsProvider가 반드시 바깥에 있어야 함
```

Provider가 많아지면 합성 컴포넌트로 정리합니다.

```tsx
function AppProviders({ children }: { children: ReactNode }) {
  return (
    <AuthProvider>
      <FriendIdsProvider>
        <AvatarActionProvider>
          {children}
        </AvatarActionProvider>
      </FriendIdsProvider>
    </AuthProvider>
  );
}
```

---

# 실전 예 — 친구 ID Set ⭐️⭐️⭐️⭐️

"이 유저가 내 친구인가?"를 여러 컴포넌트에서 확인해야 할 때의 패턴입니다.

```tsx
// friend-ids-context.tsx
const FriendIdsContext = createContext<ReadonlySet<string> | null>(null);

export function FriendIdsProvider({ children }: { children: ReactNode }) {
  const [friendIds, setFriendIds] = useState<ReadonlySet<string>>(new Set());

  useEffect(() => {
    fetchFriendIds().then(ids => setFriendIds(new Set(ids)));
  }, []);

  return (
    <FriendIdsContext.Provider value={friendIds}>
      {children}
    </FriendIdsContext.Provider>
  );
}

export function useFriendIds() {
  const ctx = useContext(FriendIdsContext);
  if (!ctx) throw new Error('FriendIdsProvider 안에서만 사용 가능합니다.');
  return ctx;
}
```

```typescript
// 사용처 — 트리 어디서든 O(1) 탐색
function UserCard({ userId }: { userId: string }) {
  const friendIds = useFriendIds();
  const isFriend  = friendIds.has(userId);  // Set.has() = O(1)
  return <div>{isFriend ? '친구' : '친구 추가'}</div>;
}
```

```txt
왜 배열이 아닌 Set인가:
  배열 .includes(id) = O(n) — 100명이면 최대 100번 비교
  Set .has(id)       = O(1) — 몇 명이든 즉시 찾음

  UserCard가 목록에 100개 있고 각자 친구인지 확인한다면:
  배열 → 100 × 100 = 10,000번 비교
  Set  → 100 × 1   = 100번
```

---

# 언제 Context를 쓰고 언제 쓰지 않는가 ⭐️⭐️⭐️⭐️

```txt
✅ Context가 맞는 경우:
  여러 컴포넌트가 같은 데이터를 필요로 할 때
  중간 컴포넌트들이 전달만을 위해 props를 받아야 할 때
  예: 로그인 유저, 테마, 언어, 친구 목록, 알림 상태

❌ Context가 과한 경우:
  1~2개 컴포넌트만 쓰는 데이터 → 그냥 props 전달이 더 단순
  input value처럼 빠르게 자주 바뀌는 값 → local state
  서버 데이터 + 캐싱·갱신 필요 → React Query / SWR

판단 기준:
  props로 전달하면 중간 컴포넌트 몇 개를 거치는가?
  1~2개 → props, 3개 이상 + 여러 곳에서 필요 → Context
```

---

# 값이 바뀌면 리렌더 — 주의사항 ⭐️⭐️⭐️

```txt
Context value가 바뀌면 useContext를 쓰는 모든 컴포넌트가 리렌더됨

value={{ user, logout }}  ← 매 렌더마다 새 객체 생성 → 참조가 바뀜
→ AuthProvider가 리렌더될 때마다 useAuth() 쓰는 컴포넌트 전부 리렌더
```

```tsx
// useMemo로 value 참조 안정화
export function AuthProvider({ children }: { children: ReactNode }) {
  const [user, setUser] = useState<User | null>(null);
  const logout = useCallback(() => setUser(null), []);

  const value = useMemo(
    () => ({ user, logout }),
    [user, logout],   // user가 바뀔 때만 새 객체 생성
  );

  return <AuthContext.Provider value={value}>{children}</AuthContext.Provider>;
}
```

```txt
useMemo 없이도 동작은 하지만:
  user가 바뀔 때 → 정상 리렌더 (원하는 동작)
  user 안 바뀌는데 부모가 리렌더 → value 새 객체 → 불필요한 리렌더

  규모가 커지거나 useAuth() 쓰는 컴포넌트가 많아지면 useMemo 추가
```

---

# 자주 만나는 에러

|증상|원인|해결|
|---|---|---|
|`useAuth()` 가 에러를 던짐|Provider 바깥에서 호출|해당 컴포넌트가 Provider 안에 있는지 확인|
|값 변경이 화면에 반영 안 됨|Provider value가 state 기반이 아님|value를 useState로 관리하고 있는지 확인|
|페이지 이동해도 Context 초기화 안 됨|Provider가 layout에 있어서 유지됨|의도적이면 OK, 아니면 Provider를 page로 이동|
|Context 안의 함수가 useEffect 루프를 만듦|함수에 useCallback 없음|함수를 `useCallback`으로 감싸기|

---

# 실전 — 인증 AuthProvider

```txt
AuthProvider 전체 구현 (refreshUser, clearSession, isLoading 포함):
  → [[NextJS_AuthState]]
```