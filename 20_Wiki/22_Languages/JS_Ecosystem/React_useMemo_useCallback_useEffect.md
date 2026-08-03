---
aliases:
  - 의존성배열
  - cancelled
  - cleanup 함수
  - dependendy array
  - useCallback
  - useEffect
  - useMemo
tags:
  - React
  - NextJS
related:
  - "[[00_JS_Ecosystem_HomePage]]"
  - "[[React_Context_Provider]]"
  - "[[NestJS_Throttle]]"
  - "[[React_useId]]"
  - "[[React_AsyncUI]]"
---
# React_useMemo_useCallback_useEffect

> [!info] 
> `useEffect`는 렌더와 별개인 부수효과(fetch·타이머·구독). 
> `useMemo`는 계산 결과 재사용
>`useCallback`은 함수 참조 고정.
>useMemo·useCallback은 같은 메모이제이션 부류, useEffect는 전혀 다른 부류.

---

# useEffect — 렌더 이후에 실행되는 부수효과 ⭐️⭐️⭐️⭐️

```txt
컴포넌트가 "그리는 것"(JSX 반환) 과는 별개로,
렌더가 끝난 뒤에 따로 처리해야 하는 일들을 담는 자리

부수효과 = 서버 데이터 가져오기 · 이벤트 리스너 등록 · 타이머 · document.title 변경
```

```typescript
useEffect(() => {
  document.title = `알림 ${count}개`;
}, [count]);  // count가 바뀔 때만 다시 실행
```

## 의존성 배열 3가지

|형태|실행 시점|
|---|---|
|`useEffect(fn, [])`|마운트 시 딱 1번|
|`useEffect(fn, [a, b])`|마운트 1번 + a 또는 b가 바뀔 때마다|
|`useEffect(fn)`|매 렌더마다 — 의도적으로 쓸 일 거의 없음|

## cleanup — 시작한 일 정리하기

```typescript
useEffect(() => {
  const id = setInterval(() => tick(), 1000);
  return () => clearInterval(id);  // 다음 effect 실행 전 또는 언마운트 시 호출
}, []);
```

```txt
구독 해제 · 타이머 정리 · 이벤트 리스너 제거처럼
"시작했으면 나중에 끝내야 하는 것"이 있으면 return 함수로 정리
안 하면 컴포넌트가 사라져도 계속 살아있는 누수(leak)가 생김
```

---

# useEffect에서 async — 두 문법, 같은 동작 ⭐️⭐️⭐️⭐️

```txt
핵심: .then() 방식과 async/await 방식은 동작이 완전히 같다.
문법만 다른 것 — 둘 중 하나를 고르면 됨.
```

```typescript
// 방법 A — Promise chain (.then)
useEffect(() => {
  let cancelled = false;

  fetchRoom(roomId)
    .then((data) => {
      if (!cancelled) setRoom(data);
    })
    .catch((err) => {
      if (!cancelled) setError(err.message);
    })
    .finally(() => {
      if (!cancelled) setLoading(false);
    });

  return () => { cancelled = true; };
}, [roomId]);


// 방법 B — async/await (내부 함수)
useEffect(() => {
  let cancelled = false;

  async function load() {
    try {
      const data = await fetchRoom(roomId);  // ← 방법 A의 fetchRoom(roomId).then(data => ...) 와 동일
      if (!cancelled) setRoom(data);
    } catch (err) {
      if (!cancelled) setError(err.message);
    } finally {
      if (!cancelled) setLoading(false);
    }
  }

  void load();
  return () => { cancelled = true; };
}, [roomId]);
```

```txt
둘의 차이가 없는 부분:
  cancelled 체크 위치 — 둘 다 응답 도착 후에 if (!cancelled)로 확인
  cleanup 함수 — 둘 다 return () => { cancelled = true }
  실제 네트워크 요청 — 둘 다 fetchRoom(roomId) 를 호출

문법만 다른 것:
  .then(data => ...)       = async/await에서 const data = await ...
  .catch(err => ...)       = try/catch의 catch 블록
  .finally(() => ...)      = try/catch의 finally 블록

useEffect에 직접 async를 못 쓰는 이유:
  async 함수는 항상 Promise를 반환함
  useEffect는 cleanup 함수(또는 undefined)만 반환값으로 인정함
  → 내부에 async function을 따로 선언 후 void로 호출하는 것이 표준 패턴

취향 차이:
  .then() 방식   → 중첩 없이 평탄한 구조
  async/await   → try/catch 구조가 더 익숙한 경우
```

## cancelled 플래그 ⭐️⭐️⭐️⭐️

```typescript
useEffect(() => {
  let cancelled = false;          // ① 이 effect 인스턴스의 유효성 플래그

  fetchRoom(roomId).then((data) => {
    if (!cancelled) setRoom(data); // ③ 아직 유효할 때만 state 갱신
  });

  return () => { cancelled = true; }; // ② 언마운트 or deps 변경 시 무효화
}, [roomId]);
```

```txt
cancelled가 막는 것:
  fetch를 멈추는 게 아님 — 요청은 끝까지 날아감
  "이미 무효화된 effect의 응답이 도착했을 때 setState하는 것"을 막음

왜 필요한가:
  roomId = 'A' → fetch 시작
  roomId = 'B' 로 바뀜 → cleanup 실행(cancelled = true) → 새 effect 시작
  'A' 응답이 나중에 도착 → if (!cancelled) → cancelled = true → setState 무시 ✅

  이 체크가 없으면:
  현재 화면은 'B'인데 'A'의 데이터가 덮어쓰는 경쟁 조건(race condition) 발생

비동기 패턴 전체(reqIdRef · 디바운스 · fire-and-forget) → [[React_AsyncUI]]
```

---

# useMemo — 계산 결과 재사용 ⭐️⭐️⭐️

```typescript
const sorted = useMemo(() => {
  return [...items].sort((a, b) => a.price - b.price);
}, [items]);
// items가 안 바뀌면 이전 sorted를 그대로 재사용 — 매 렌더마다 정렬 안 함
```

## 언제 쓰는가

```txt
① 무거운 계산: 데이터가 많고 정렬·필터링 비용이 클 때
② 참조 안정화: 객체나 배열을 만들어서 다른 훅의 deps나 Context value로 넘길 때
   (매 렌더마다 새 객체가 생기면 deps가 매번 바뀐 것으로 취급됨)
```

```typescript
// ② 참조 안정화 — Context value
const value = useMemo(() => ({
  user,
  logout,
  refresh,
}), [user, logout, refresh]);
// useMemo 없이 그냥 객체 리터럴로 만들면
// 매 렌더마다 새 객체 → Context 구독자 전부 불필요 리렌더
```

## 언제 필요 없는가

```txt
a + b 같은 가벼운 계산 — 메모이제이션 비용이 계산 비용보다 클 수 있음
단순 화면 표시용 문자열 조합
자식에게 안 넘기고 그냥 렌더에서만 쓰는 값

useMemo 자체도 비용이 있음 (이전 값 저장 + 매 렌더마다 deps 비교)
"일단 다 감싸기" 대신, 실제로 무거운 계산이거나 참조 안정성이 필요한 곳에만
```

---

# useCallback — 함수 참조 고정 ⭐️⭐️⭐️⭐️

```txt
useMemo(fn, deps)  →  계산 결과 값을 기억
useCallback(fn, deps)  →  함수 자체를 기억 (useMemo의 함수 전용 축약형, 별개 메커니즘 아님)

useCallback(fn, deps) = useMemo(() => fn, deps)  — 완전히 동일한 동작
```

## 핵심 질문 하나 ⭐️⭐️⭐️⭐️

```txt
이 함수의 참조(주소)가 바뀌면 문제가 생기는 곳이 있는가?

→ 예  : useCallback 필요
→ 아니오 : useCallback 불필요 (그냥 함수 선언)
```

```txt
"참조가 바뀌면 문제가 생기는 곳" — 딱 3가지:

  ① React.memo로 감싼 자식에 props로 전달
     memo는 props를 === 비교 → 함수 참조가 바뀌면 리렌더
     → 바뀌지 않아야 memo가 의미 있음

  ② useEffect, useMemo의 deps 배열에 함수를 넣음
     deps 배열의 값이 바뀌면 effect/메모 재실행
     → 함수 참조가 매 렌더 바뀌면 effect가 매 렌더 재실행

  ③ useMemo로 만든 Context value 안에 함수 포함
     함수 참조가 바뀌면 value 객체가 새 참조 → Context 구독자 전부 리렌더
```

## 대부분의 경우 필요 없다

```typescript
// ❌ useCallback 불필요 — onClick만 있고 memo 자식도, deps도 아님
function Form() {
  const handleSubmit = () => { /* ... */ };  // 매 렌더 새 함수여도 문제 없음
  return <button onClick={handleSubmit}>제출</button>;
}

// ❌ useCallback 불필요 — effect 안에서만 정의하고 끝나는 함수
useEffect(() => {
  async function load() { const data = await fetch(); setData(data); }
  void load();  // load는 effect 밖으로 나가지 않음 → 참조 비교 없음
}, [id]);
```

## 필요한 상황 1 — effect deps + 바깥에서도 호출 (공유 load)

```typescript
// 화면 진입 시 자동 로드 + 액션 후 수동 재로드가 같은 함수를 씀
const load = useCallback(async () => {
  const data = await fetchProfile(id);
  setProfile(data);
}, [id]);  // load 본문이 읽는 값만

useEffect(() => {
  void load();
}, [load]);  // load가 deps에 있음

// 같은 load를 자식에게도 전달
<ProfileActions onRefresh={load} />
```

```txt
useCallback 없이 const load = async () => {...} 로 하면:
  매 렌더마다 load가 새 함수
  → useEffect deps [load]가 매 렌더 변경 감지
  → effect가 매 렌더 재실행 → fetch 무한 루프

useCallback([id])으로 하면:
  id가 안 바뀌면 load 참조 유지
  → effect는 id가 바뀔 때만 재실행
```

## 필요한 상황 2 — memo 자식 props

```typescript
const handleDelete = useCallback((id: number) => {
  setItems(prev => prev.filter(item => item.id !== id));
}, []);  // setItems는 안정적 — deps 불필요

// CommentItem이 React.memo로 감싸져 있을 때
{items.map(item => (
  <CommentItem key={item.id} onDelete={handleDelete} />
))}
```

```txt
React.memo(CommentItem)은 props가 같으면 리렌더를 건너뜀
useCallback 없이 handleDelete를 매 렌더 새로 만들면
onDelete가 매번 새 참조 → memo가 항상 "props 바뀜"으로 판단 → memo 무력화
```

## 필요한 상황 3 — Context value 안의 함수

```typescript
const logout = useCallback(() => {
  setUser(null);
  router.replace('/login');
}, [router]);

const value = useMemo(() => ({ user, logout }), [user, logout]);
// logout이 안정적이어야 value 참조가 안정적
// value가 안정적이어야 Context 구독자들이 불필요하게 리렌더되지 않음
```

## 함수형 업데이트로 deps 줄이기

```typescript
// ❌ state를 직접 읽으면 deps에 포함해야 함
const addItem = useCallback((item: Item) => {
  setItems([...items, item]);  // items 읽음 → deps에 items 필요 → items 바뀔 때마다 새 함수
}, [items]);

// ✅ 함수형 업데이트 — state를 deps에서 뺄 수 있음
const addItem = useCallback((item: Item) => {
  setItems(prev => [...prev, item]);  // React가 최신 prev를 직접 줌 → items deps 불필요
}, []);  // 빈 배열 → 함수 참조 완전 고정
```

## useCallback 필요 여부 체크

```txt
이 함수가...                                    useCallback
─────────────────────────────────────────────────────────
onClick만 (memo 자식 아님, deps 아님)             ❌ 불필요
effect 안에서만 정의하고 호출                      ❌ 불필요
effect deps에 들어가고 바깥에서도 호출 (공유 load) ✅ 필요
React.memo 자식의 props onX=fn                  ✅ 필요
Context value useMemo 안에 포함                  ✅ 필요
```

---

# Rules of Hooks — 훅 규칙 ⭐️⭐️⭐️⭐️

```txt
① 최상위 레벨에서만 — if / for / map 안에서 호출 금지
② React 함수 컴포넌트 또는 커스텀 훅 안에서만 — 일반 함수에서 호출 금지
```

## map 안 훅 호출 — 절대 금지

```typescript
// ❌ 훅 규칙 위반
{members.map((m) => {
  const isFriend = useIsFriend(m.userId);  // map 안에서 훅 호출
  return <MemberItem key={m.id} isFriend={isFriend} />;
})}
```

```txt
왜 안 되는가:
  React는 훅을 "항상 같은 순서, 같은 개수"로 호출된다고 가정하고
  순서 기반으로 각 훅을 내부 슬롯에 연결함

  members.length = 3 → useIsFriend 3번
  members.length = 5 → useIsFriend 5번

  렌더마다 훅 개수가 달라짐 → React가 state 추적 불가
  → "Rendered more/fewer hooks than expected" 에러
```

## 해결 — Set으로 최상위에서 한 번만

```typescript
// 커스텀 훅으로 의도 명시
function useFriendIdSet(): ReadonlySet<string> {
  const { ids } = useFriendIds();  // Context에서 한 번만
  return ids;
}

function MemberList({ members }: { members: Member[] }) {
  const friendIds = useFriendIdSet();  // ← 훅은 최상위에서 한 번

  return (
    <ul>
      {members.map((m) => {
        const isFriend = friendIds.has(m.userId);  // ← Set.has()는 훅 아님, 개수 달라져도 OK
        return <MemberItem key={m.id} isFriend={isFriend} />;
      })}
    </ul>
  );
}
```

```txt
패턴:
  훅 → 최상위에서 1번 → 결과(Set, 배열, 값)를 변수에 저장
  map 안 → 그 변수를 이용한 일반 연산 (훅 아님)
```

---

# 의존성 배열 공통 규칙 ⭐️⭐️⭐️

## 객체 대신 원시값으로 좁히기

```typescript
// ❌ 객체 전체 — user 참조가 바뀔 때마다 실행 (닉네임만 바뀌어도)
useEffect(() => { ... }, [user]);

// ✅ 필요한 원시값만 — 실제로 의미 있는 변화에만 반응
useEffect(() => { ... }, [user?.id]);
```

```txt
객체는 내용이 같아도 참조(메모리 주소)가 다르면 deps가 바뀐 것으로 취급
"이 effect가 반응해야 하는 게 뭔가?" 를 생각하고 그 값만 deps에
```

## 흔한 실수

| 실수                                  | 문제                                           |
| ----------------------------------- | -------------------------------------------- |
| deps에 써야 하는 값을 빠뜨림                  | 그 값이 바뀌어도 effect가 옛 값을 계속 참조 (stale closure) |
| 렌더 중에 계산 가능한 값을 useEffect+state로 만듦 | 불필요한 리렌더 추가 발생 — 그냥 렌더 중에 바로 계산              |
| deps에 객체 전체를 넣음                     | 내용 변화가 없어도 참조 바뀌면 재실행                        |