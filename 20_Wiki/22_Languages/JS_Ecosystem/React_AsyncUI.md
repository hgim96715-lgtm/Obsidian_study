---
aliases:
  - 비동기 UI
tags:
  - React
related:
  - "[[00_JS_Ecosystem_HomePage]]"
  - "[[JS_Operators]]"
  - "[[JS_Promise]]"
  - "[[TS_Generics]]"
  - "[[React_useEffect]]"
---
# React_AsyncUI — 비동기 UI 패턴

>[!info]
>비동기 UI = 서버 요청 중·성공·실패 각 상태를 화면에 반영하는 패턴. 
>`isLoading`·`error`·`data`를 세트로 관리
>try 전에 `setError('')`·`setIsLoading(true)`로 이전 상태를 초기화하고, `finally`에서 `setIsLoading(false)`로 항상 정리한다.

---

# 비동기 작업의 3가지 상태 ⭐️⭐️⭐️⭐️

```txt
서버에 요청을 보내면 항상 세 가지 상태가 존재:

  ① 요청 중 (loading)
     화면에 스피너 또는 버튼 비활성화

  ② 성공 (success)
     데이터 표시 또는 다음 단계 진행

  ③ 실패 (error)
     에러 메시지 표시, 재시도 버튼

이 세 가지를 state로 관리:
  const [isLoading, setIsLoading] = useState(false);
  const [error,     setError]     = useState('');
  const [data,      setData]      = useState(null);
```

---

# isLoading · error — 상태 세트 ⭐️⭐️⭐️⭐️

```typescript
// 이 세 state는 항상 세트로 움직임
const [isLoading, setIsLoading] = useState(false);  // 요청 중?
const [error,     setError]     = useState('');     // 에러 메시지
const [rows,      setRows]      = useState<Row[]>([]);  // 데이터
```

```txt
isLoading:
  true  → 요청 진행 중 (스피너 표시, 버튼 disabled)
  false → 완료 (성공이든 실패든)

error:
  ''    → 에러 없음
  '...' → 에러 메시지 (화면에 표시)

초기값 선택:
  useState(false) → 버튼 클릭으로 요청 시작 (처음엔 로딩 아님)
  useState(true)  → 컴포넌트 마운트 즉시 로딩 시작 (useEffect fetch)
```

```typescript
// 초기값 true — 마운트하자마자 로딩 시작
const [isLoading, setIsLoading] = useState(true);
//                                              ↑ 처음부터 스피너 표시

useEffect(() => {
  async function load() {
    // isLoading이 이미 true라서 마운트 즉시 스피너 보임
    const data = await fetchData();
    setRows(data);
    setIsLoading(false);
  }
  void load();
}, []);
```

---

# try 전에 초기화하는 이유 ⭐️⭐️⭐️⭐️

```typescript
async function load() {
  setError('');        // ← 왜?
  setIsLoading(true);  // ← 왜?
  try {
    const data = await fetch(...);
    setRows(data);
  } catch {
    setError('불러오지 못했어요.');
  } finally {
    setIsLoading(false);
  }
}
```

```txt
setError('') — 이전 에러를 지우는 이유:
  첫 번째 요청 → 실패 → error = '불러오지 못했어요.'
  사용자가 재시도 버튼 클릭 → load() 다시 실행
  → setError('')가 없으면:
    새 요청이 성공해도 이전 에러 메시지가 화면에 남아있음
  → setError('')로 이전 에러를 지우고 새로 시작

setIsLoading(true) — try 전에 하는 이유:
  try 안에서 하면 await 전에 렌더링이 안 됨
  → 스피너가 실제로 보이는 시점이 늦어짐
  → try 전에 해야 "요청 시작하자마자 스피너 표시"
```

```txt
finally — setIsLoading(false)를 여기에 하는 이유:
  try와 catch 중 어느 쪽이 실행되든 항상 실행됨
  → 성공해도, 실패해도 isLoading을 false로 만들어야 함

  ❌ catch에만 쓰면:
    성공 시 → catch가 실행 안 됨 → isLoading이 true인 채로 남음

  ❌ try 끝에 쓰면:
    catch에서 에러 처리 후에도 isLoading이 false가 안 됨

  ✅ finally:
    성공이든 실패든 항상 isLoading = false
```

---

# 패턴 1 — useEffect 자동 로드 ⭐️⭐️⭐️⭐️

```typescript
// 컴포넌트가 마운트될 때 자동으로 데이터 로드
const [isLoading, setIsLoading] = useState(true);  // 처음부터 로딩 중
const [error,     setError]     = useState('');
const [rows,      setRows]      = useState<Row[]>([]);

useEffect(() => {
  let cancelled = false;

  async function load() {
    setError('');
    setIsLoading(true);
    try {
      const data = await adminFetchJson<ApiAdminNotice[]>('/notices');
      if (!cancelled) setRows(data);
    } catch {
      if (!cancelled) setError('공지 목록을 불러오지 못했어요.');
    } finally {
      if (!cancelled) setIsLoading(false);
    }
  }

  void load();
  return () => { cancelled = true; };
}, []);
```

```txt
if (!cancelled) 가 필요한 이유:
  컴포넌트가 언마운트된 뒤에 응답이 오면
  → 언마운트된 컴포넌트에 setState → React 경고 + 메모리 누수
  → cancelled = true이면 setState를 모두 건너뜀

void load():
  useEffect 콜백은 async 불가 (cleanup 함수를 반환해야 함)
  → 내부에 async 함수를 선언하고 void로 호출
  → void = "이 Promise의 반환값을 무시한다"
```

## 화면에 반영

```tsx
function NoticeList() {
  const [isLoading, setIsLoading] = useState(true);
  const [error,     setError]     = useState('');
  const [rows,      setRows]      = useState<Row[]>([]);

  // ... useEffect load ...

  if (isLoading) return <Spinner />;
  if (error)     return <ErrorMessage message={error} />;
  return <Table rows={rows} />;
}
```

---

# 패턴 2 — 이벤트 핸들러 비동기 (버튼 클릭) ⭐️⭐️⭐️⭐️

```typescript
// 버튼 클릭 → 서버 요청
const [isPending, setIsPending] = useState(false);  // 처음엔 false
const [error,     setError]     = useState('');

async function handleSubmit() {
  setError('');
  setIsPending(true);
  try {
    await createPost(formData);
    router.push('/posts');      // 성공 시 이동
  } catch (err) {
    setError(err instanceof Error ? err.message : '저장하지 못했어요.');
  } finally {
    setIsPending(false);
  }
}
```

```tsx
// 버튼에 반영
<button
  onClick={handleSubmit}
  disabled={isPending}          // 요청 중 클릭 방지
>
  {isPending ? '저장 중...' : '저장'}
</button>

{error && <p className="text-red-500">{error}</p>}
```

```txt
isPending vs isLoading:
  두 이름 모두 "진행 중"을 의미하지만 관례 차이:
  isLoading → 데이터를 불러오는 중 (페이지 초기 로드, 조회)
  isPending → 작업이 처리되는 중 (저장, 삭제, 전송 등 액션)
  → 어느 쪽을 써도 동작은 같음
```

---

# 패턴 3 — 낙관적 업데이트 (Optimistic UI) ⭐️⭐️⭐️

```txt
일반(Pessimistic):
  버튼 클릭 → 서버 요청 → 성공 응답 → 화면 업데이트
  → 사용자는 완료까지 기다려야 함 (느린 느낌)

낙관적(Optimistic):
  버튼 클릭 → 즉시 화면 업데이트 → 서버 요청 → 실패 시 롤백
  → 사용자에게 즉각적인 반응 (빠른 느낌)
  → 좋아요, 팔로우, 읽음 처리처럼 실패 가능성 낮은 것에 적합
```

```typescript
async function handleLike(postId: string) {
  // ① 즉시 화면 업데이트 (낙관적)
  const prev = liked;
  setLiked(!liked);
  setLikeCount(c => liked ? c - 1 : c + 1);

  try {
    await toggleLike(postId);
    // ② 성공 → 그대로 유지
  } catch {
    // ③ 실패 → 원래대로 롤백
    setLiked(prev);
    setLikeCount(c => liked ? c + 1 : c - 1);
    setError('잠시 후 다시 시도해주세요.');
  }
}
```

---

# 패턴 4 — fire-and-forget ⭐️⭐️⭐️

```typescript
// 결과를 기다리지 않고 그냥 실행
void markAsRead(notificationId);
//  ↑ void = "이 Promise의 결과를 무시한다"

// 실패해도 에러를 잡지 않음 → 중요하지 않은 부수 작업
// 읽음 처리, 로그 기록, 통계 업데이트 등
```

```txt
void vs await 판단:
  await → 결과가 중요하거나 실패 시 사용자에게 알려야 할 때
  void  → 실패해도 사용자 경험에 영향 없는 부수 작업

  void를 명시하는 이유:
  그냥 markAsRead(id)만 쓰면 → ESLint "unhandled promise" 경고
  void를 붙이면 → "의도적으로 무시함"을 명시
```

---

# 패턴 5 — 여러 state 동시 업데이트 ⭐️⭐️⭐️

```typescript
// 성공 시 여러 state를 한꺼번에 업데이트
async function handleConfirm() {
  try {
    const result = await deleteMessage(messageId);
    // 성공 — 여러 state 동시에 변경
    setMessage({ ...message, deletedAt: new Date().toISOString() });
    setConfirmOpen(false);
    setSelectedId(null);
  } catch (err) {
    setError(err instanceof Error ? err.message : '삭제하지 못했어요.');
  }
}
```

```txt
React 18+에서 여러 setState를 연속 호출해도
하나의 렌더링으로 자동 배치(batching)됨
→ setMessage, setConfirmOpen, setSelectedId를 각각 해도
  렌더링은 한 번만 발생
```

---

# 전체 골격 — 참고용

```typescript
// useEffect 자동 로드
const [isLoading, setIsLoading] = useState(true);
const [error,     setError]     = useState('');
const [data,      setData]      = useState<T[]>([]);

useEffect(() => {
  let cancelled = false;
  async function load() {
    setError('');
    setIsLoading(true);
    try {
      const result = await fetchSomething();
      if (!cancelled) setData(result);
    } catch {
      if (!cancelled) setError('불러오지 못했어요.');
    } finally {
      if (!cancelled) setIsLoading(false);
    }
  }
  void load();
  return () => { cancelled = true; };
}, [의존성]);

// 이벤트 핸들러
const [isPending, setIsPending] = useState(false);

async function handleAction() {
  setError('');
  setIsPending(true);
  try {
    await doSomething();
    // 성공 처리
  } catch (err) {
    setError(err instanceof Error ? err.message : '실패했어요.');
  } finally {
    setIsPending(false);
  }
}
```