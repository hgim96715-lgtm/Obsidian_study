---
aliases:
  - 비동기 UI
  - isLoading
  - isPending
  - isXxxing
  - useTransition
  - 중복 클릭 방지
  - 낙관적 업데이트
  - fire-and-forget
tags:
  - React
relations:
  - "[[00_JS_Ecosystem_HomePage]]"
  - "[[JS_Promise]]"
  - "[[React_useEffect]]"
  - "[[React_useFormStatus]]"
---
# React_AsyncUI — 비동기 UI 패턴

>[!info]
> 비동기 작업(API 요청)은 항상 세 가지 상태를 가짐: 진행 중 · 성공 · 실패.
> 이 파일은 그 상태를 어떻게 관리하고 UI에 반영하는지 패턴별로 정리.
> 흐름: 기본 원칙 → 초기 로드 → 액션 → 중복 방지 → useTransition → 낙관적 업데이트

---

# 비동기 상태 3가지 — 핵심 원칙 ⭐️⭐️⭐️⭐️⭐️

```txt
API 요청은 반드시 세 가지 상태를 가짐:

  ① loading (진행 중) → 스피너, 버튼 비활성화
  ② success (성공)   → 데이터 표시, 다음 단계
  ③ error   (실패)   → 에러 메시지, 재시도 버튼

이 세 가지를 state로 관리:
  const [loading, setLoading] = useState(false);
  const [error,   setError]   = useState('');
  const [data,    setData]    = useState<T[]>([]);
```

## 네이밍 체계

```txt
"진행 중" 상태 이름 — 상황에 따라 다르게 씀:

  isLoading    → 페이지·섹션 최초 데이터 조회 (마운트 시 자동 실행)
  isPending    → 단일 액션 진행 중 (버튼 클릭 → 저장·전송)
  isSubmitting → 폼 제출 진행 중 (react-hook-form 컨벤션)
  isDeleting   → 삭제 진행 중
  isAdding     → 추가 진행 중
  isSaving     → 저장 진행 중
  → 동사+ing 형태로 어떤 액션인지 명시

  is를 붙이지 않는 경우:
    loading, pending, deleting — 팀 컨벤션에 따라 자유롭게
    동작은 완전히 동일 (boolean이면 됨)
```

---

# try / finally 원칙 ⭐️⭐️⭐️⭐️⭐️

```typescript
async function load() {
  setError('');         // ① 이전 에러 초기화
  setLoading(true);     // ② 진행 중 표시 — try 바깥에서

  try {
    const data = await fetchSomething();
    setData(data);
  } catch {
    setError('불러오지 못했어요.');
  } finally {
    setLoading(false);  // ③ 성공·실패 모두 false 복원
  }
}
```

```txt
① setError('') — try 전에 하는 이유:
  이전 요청이 실패 → error 메시지가 남아있음
  재시도 클릭 → 새 요청이 성공해도 이전 에러가 화면에 남음
  → try 전에 '' 초기화 필수

② setLoading(true) — try 바깥에서 하는 이유:
  try 안에 넣으면 await 이전에 렌더링이 안 됨
  → 스피너가 실제로 늦게 표시됨
  → 바깥에서 해야 "요청 시작하자마자 스피너 표시"

③ finally — setLoading(false)를 여기에 하는 이유:
  try와 catch 중 어느 쪽이든 항상 실행됨
  → ❌ catch에만 → 성공 시 loading이 true인 채로 남음
  → ❌ try 끝에만 → 실패 시 finally 안 돌아 loading 안 꺼짐
  → ✅ finally → 성공이든 실패든 항상 loading = false
```

---

# 트리거 카운터 — refreshKey 패턴 ⭐️⭐️⭐️⭐️

```txt
refreshKey는 "몇 개인지"가 아니라 "바뀌었다"는 신호만 필요
→ 값의 의미는 없고, 값이 변하면 useEffect가 재실행된다는 것만 중요
→ 삭제든 추가든 수정이든 항상 +1
```

## 왜 삭제해도 -1이 아니라 +1인가

```tsx
// ❌ 헷갈리는 생각
// "영화를 삭제했으니까 숫자가 줄어야 하지 않나?"
setCalendarRefreshKey((current) => current - 1);

// ✅ 실제 의미
// refreshKey = "캘린더를 다시 그려라" 신호
// 값이 무엇인지는 관계없음 — 이전과 "달라졌는가"가 전부
setCalendarRefreshKey((current) => current + 1);
```

```txt
refreshKey는 총 개수가 아님:
  total = 10 → 삭제 → total = 9  ← 데이터 카운트 (의미 있는 값)
  refreshKey = 3 → 삭제 → refreshKey = 4  ← 트리거 카운터 (값 자체에 의미 없음)

  total은 "몇 개냐"는 질문에 답함
  refreshKey는 "언제 바뀌었냐"는 질문에 답함

  삭제 → +1,  추가 → +1,  수정 → +1
  항상 +1 — 변화가 일어났다는 신호를 보내는 것뿐
```

## useEffect 의존성과 연결

```tsx
const [calendarRefreshKey, setCalendarRefreshKey] = useState(0);

useEffect(() => {
  fetchCalendar();
}, [calendarRefreshKey]);  // ← 값이 바뀔 때마다 fetchCalendar 재실행

// 어떤 액션이든 완료 후 +1 → useEffect 재실행 → 화면 갱신
const handleDelete = async () => {
  await deleteMovie(id);
  setCalendarRefreshKey((k) => k + 1);  // "다시 불러와" 신호
};

const handleAdd = async () => {
  await addMovie(data);
  setCalendarRefreshKey((k) => k + 1);  // 동일하게 +1
};
```

```txt
React useEffect 동작 원리:
  의존성 배열 [calendarRefreshKey]의 값이 이전 렌더와 다르면 → effect 재실행
  0 → 1 : 다름 → 재실행 ✅
  1 → 2 : 다름 → 재실행 ✅
  1 → 0 : 다름 → 재실행 ✅  (-1도 기술적으로 작동함)

  결국 +1이든 -1이든 Date.now()든 작동하지만
  +1이 관례 — "버전이 올라간다"는 직관적 의미
```

## 함수형 업데이트 — (current) => current + 1

```tsx
// ❌ 직접 값 참조 — 클로저 함정
setCalendarRefreshKey(calendarRefreshKey + 1);
// 비동기 핸들러 안에서 calendarRefreshKey가 오래된 값일 수 있음

// ✅ 함수형 업데이트 — 항상 최신 상태 기반
setCalendarRefreshKey((current) => current + 1);
// current = React가 보장하는 현재 최신값
// 비동기 컨텍스트에서도 안전
```

## 네이밍 — "숫자가 아님"을 이름에서 드러내기

```tsx
// ❌ 의미가 헷갈리는 이름
const [calendarCount, setCalendarCount] = useState(0);    // "영화 개수"처럼 보임
const [refreshCount, setRefreshCount]   = useState(0);    // 그나마 낫지만

// ✅ 트리거/신호임을 명확히
const [calendarRefreshKey, setCalendarRefreshKey] = useState(0);
const [movieListKey,       setMovieListKey]        = useState(0);
const [version,            setVersion]             = useState(0);   // 짧게 쓸 때
```

```txt
Key라는 단어의 힌트:
  React의 key prop — 값이 바뀌면 컴포넌트를 완전히 새로 마운트
  refreshKey — 값이 바뀌면 데이터를 다시 불러옴
  둘 다 "값이 달라졌다는 신호"를 보내는 역할
```

## key prop 활용 — 컴포넌트 완전 초기화

```tsx
// refreshKey를 key prop에 넣으면 → 컴포넌트 언마운트 후 새로 마운트
// 내부 state, useEffect 전부 초기화됨

<Calendar key={calendarRefreshKey} year={year} month={month} />

// vs useEffect 의존성 방식 — 컴포넌트는 유지, 데이터만 다시 fetch
useEffect(() => {
  fetchCalendar();
}, [calendarRefreshKey]);
```

```txt
선택 기준:
  key prop                컴포넌트 자체를 리셋 (스크롤 위치, 내부 폼, 에니메이션 포함)
  useEffect 의존성        데이터만 다시 불러옴 (UI 상태는 유지)

  캘린더 전체 다시 그리기 → key prop
  목록만 새로고침         → useEffect 의존성
```


---

# 패턴 A — 초기 데이터 로드 (useEffect) ⭐️⭐️⭐️⭐️

```txt
컴포넌트 마운트 시 자동으로 데이터를 불러오는 패턴
초기값: isLoading = true (마운트하자마자 스피너 표시)
```

```tsx
const [isLoading, setIsLoading] = useState(true);  // ← 처음부터 스피너
const [error,     setError]     = useState('');
const [rows,      setRows]      = useState<Row[]>([]);

useEffect(() => {
  let cancelled = false;  // 언마운트 시 setState 방지용 플래그

  async function load() {
    setError('');
    setIsLoading(true);
    try {
      const data = await fetchRows();
      if (!cancelled) setRows(data);
    } catch {
      if (!cancelled) setError('불러오지 못했어요.');
    } finally {
      if (!cancelled) setIsLoading(false);
    }
  }

  void load();
  return () => { cancelled = true; };  // 언마운트 시 cancelled = true
}, []);
```

```txt
if (!cancelled) 이유:
  컴포넌트가 언마운트된 뒤 응답이 도착하면
  → 사라진 컴포넌트에 setState → 로직 버그
  → cancelled = true이면 setState 건너뜀

void load():
  useEffect 콜백은 async 불가 (cleanup 함수를 반환해야 하므로)
  → 내부에 async 함수 선언 후 void로 호출
  → void = "이 Promise 결과를 의도적으로 무시"
```

```tsx
// 화면 반영
function NoticeList() {
  // ...상태 선언, useEffect...

  if (isLoading) return <Spinner />;
  if (error)     return <ErrorMessage message={error} />;
  return <Table rows={rows} />;
}
```

---

# 패턴 B — 버튼 클릭 액션 (isPending) ⭐️⭐️⭐️⭐️

```txt
버튼 클릭 → 서버 요청 → 완료 후 결과 처리
초기값: isPending = false (버튼 클릭 전엔 진행 중 아님)
```

```tsx
const [isPending, setIsPending] = useState(false);
const [error,     setError]     = useState('');

async function handleSubmit() {
  setError('');
  setIsPending(true);
  try {
    await createPost(formData);
    router.push('/posts');   // 성공 시 이동
  } catch (err) {
    setError(err instanceof Error ? err.message : '저장하지 못했어요.');
  } finally {
    setIsPending(false);
  }
}

// JSX
<button onClick={handleSubmit} disabled={isPending}>
  {isPending ? '저장 중...' : '저장'}
</button>
{error && <p className="text-red-500 text-sm">{error}</p>}
```

---

# 패턴 C — 중복 클릭 방지 ⭐️⭐️⭐️⭐️⭐️

## 왜 isLoading 하나로 부족한가

```txt
컴포넌트에 독립된 액션이 여러 개일 때:
  isLoading  → 전체 페이지 초기 로딩
  isDeleting → 삭제 버튼 진행 중
  isAdding   → 추가 버튼 진행 중

isLoading 하나만 쓰면:
  삭제 중 → isLoading=true → 관계없는 버튼들도 disabled
  어떤 액션이 진행 중인지 구분 불가
  → 액션마다 전용 boolean state 분리
```

## 기본 패턴 — if 가드 + finally 복원

```tsx
const [isAdding, setIsAdding] = useState(false);

async function handleAdd(movie: GachaMovie) {
  // ① JS 가드: 이미 진행 중이면 즉시 리턴
  if (!addDate || !accessToken || isAdding) return;

  setIsAdding(true);    // ② 진입 즉시 true

  try {
    await addWatchedMovieRequest(accessToken, movie.id, addDate);
    setRefreshKey((k) => k + 1);
    setAddDate(null);
  } catch {
    setAddDate(null);
    setAddError('관람 영화 추가에 실패했어요.');
  } finally {
    setIsAdding(false); // ③ 성공·실패 모두 복원
  }
}

// ① JS 가드 + ② UI 가드(disabled) 둘 다 필요
<button
  onClick={() => handleAdd(movie)}
  disabled={isAdding}
  className={isAdding ? 'opacity-50 cursor-not-allowed' : ''}
>
  {isAdding ? '추가 중...' : '추가'}
</button>
```

```txt
중복 클릭이 문제가 되는 이유:
  요청이 두 번 나가면 → 서버에 같은 데이터 중복 생성
  두 번째 요청이 먼저 완료되면 → state가 첫 번째 결과로 덮어써짐 (race condition)

JS 가드만 있으면: disabled 없어서 사용자 혼란
UI 가드만 있으면: JS로 직접 함수 호출 시 뚫림
→ 두 레이어 모두 적용
```

## 아이템별 로딩 — deletingId 패턴

```tsx
// 목록의 각 아이템마다 삭제 버튼이 있을 때
const [deletingId, setDeletingId] = useState<number | null>(null);

async function handleDelete(id: number) {
  if (deletingId !== null) return;
  setDeletingId(id);
  try {
    await deleteMovie(id);
    setMovies((prev) => prev.filter((m) => m.id !== id));
  } catch {
    setError('삭제에 실패했어요.');
  } finally {
    setDeletingId(null);
  }
}

{movies.map((movie) => (
  <li key={movie.id}>
    {movie.title}
    <button
      onClick={() => handleDelete(movie.id)}
      disabled={deletingId === movie.id}   // 이 아이템만 disabled
    >
      {deletingId === movie.id ? '삭제 중...' : '삭제'}
    </button>
  </li>
))}
```

## isPending prop — 자식 컴포넌트 UI 전체 잠금

```txt
문제: 요청 중 닫기 버튼을 누르면?
  → 컴포넌트 언마운트 → finally의 setState가 사라진 컴포넌트에 적용 → 버그

해결: 부모가 isPending을 자식 prop으로 전달 → 닫기 버튼까지 잠금
```

```tsx
// 부모
function CalendarPage() {
  const [isAdding, setIsAdding] = useState(false);

  async function handleSelect(movie: GachaMovie) {
    if (isAdding) return;
    setIsAdding(true);
    try {
      await addWatchedMovieRequest(accessToken, movie.id, addDate);
      setAddDate(null);
    } catch {
      setAddError('추가 실패');
    } finally {
      setIsAdding(false);
    }
  }

  return (
    <PosterPicker
      onSelect={handleSelect}
      onClose={() => setAddDate(null)}
      isPending={isAdding}           // ← 자식에 전달
    />
  );
}

// 자식
interface PosterPickerProps {
  onSelect: (movie: GachaMovie) => void;
  onClose: () => void;
  isPending?: boolean;   // optional — undefined === false
}

function PosterPicker({ onSelect, onClose, isPending }: PosterPickerProps) {
  return (
    <div>
      <button
        onClick={onClose}
        disabled={isPending}        // 닫기도 잠금 ← 언마운트 방지
        aria-label="포스터 검색 닫기"
      />
      {movies.map((movie) => (
        <button
          key={movie.id}
          onClick={() => onSelect(movie)}
          disabled={isPending}      // 선택도 잠금 ← 중복 방지
        >
          {movie.title}
        </button>
      ))}
      {isPending && <p>추가 중...</p>}
    </div>
  );
}
```

```txt
잠가야 하는 버튼: 액션 버튼 · 닫기 버튼 · 취소 버튼
잠그지 않아도 되는 것: 스크롤, 텍스트 선택, 읽기 전용 UI
```

---

# 패턴 D — useTransition (React 18+) ⭐️⭐️⭐️⭐️

```txt
useTransition = React가 pending 상태를 자동 추적
  [isPending, startTransition] = useTransition()

  → setIsAdding(true) / finally { setIsAdding(false) } 코드 불필요
  → React가 startTransition 내부 비동기 완료 시 자동으로 isPending=false
```

## ⚠️ await 이후 setState는 startTransition으로 반드시 재감싸야 함

```tsx
const [isAdding, startTransition] = useTransition();

function handleAdd(movie: GachaMovie) {
  if (!addDate || !accessToken || isAdding) return;

  startTransition(async () => {           // ① 외부 transition
    try {
      await addWatchedMovieRequest(       // ② await — 여기서 컨텍스트 끊김
        accessToken, movie.id, addDate,
      );
      startTransition(() => {             // ③ await 이후 재감싸기 ⭐️
        setRefreshKey((k) => k + 1);
        setAddDate(null);
      });
    } catch {
      startTransition(() => {             // ③ catch도 동일
        setAddDate(null);
        setAddError('관람 영화 추가에 실패했어요.');
      });
    }
  });
}
```

```txt
왜 await 이후 재감싸야 하는가:
  await = 비동기 경계 → 이후 코드는 새로운 마이크로태스크에서 실행
  → React transition 컨텍스트가 이어지지 않음
  → await 이후 setState가 transition 밖에서 실행 → 불필요한 리렌더링, Suspense 오류

  startTransition(async () => {
    await something();    ← 컨텍스트 끊김
    setState(...);        ← ❌ transition 밖
  });

  → 해결: await 이후 setState마다 startTransition으로 다시 감쌈
```

## useState boolean vs useTransition 비교

```txt
                    useState boolean    useTransition
──────────────────────────────────────────────────────────
pending 관리        수동 true/false     자동 (React 추적)
finally 필요        ✅                  ❌ (자동 복원)
await 이후 주의     없음                startTransition 재감싸기 필수 ⭐️
에러 처리           try/catch 직접      try/catch 직접 (동일)
React 버전          모든 버전           React 18+
동시성 모드 통합     ❌                 ✅ (Suspense 연동)

선택 기준:
  단순 액션, 에러 처리 중요    → useState boolean
  React 19 + Suspense 조합   → useTransition
  Server Action              → useFormStatus() → [[React_useFormStatus]]
```

---

# 패턴 E — 낙관적 업데이트 ⭐️⭐️⭐️

```txt
일반(Pessimistic): 클릭 → 요청 → 응답 → 화면 업데이트 (사용자가 기다림)
낙관적(Optimistic): 클릭 → 즉시 화면 업데이트 → 요청 → 실패 시 롤백

  → 좋아요·팔로우·읽음 처리처럼 실패 가능성 낮은 액션에 적합
```

```tsx
async function handleLike(postId: string) {
  const prev = liked;

  // ① 즉시 UI 업데이트 (낙관적)
  setLiked(!liked);
  setLikeCount((c) => liked ? c - 1 : c + 1);

  try {
    await toggleLike(postId);
    // ② 성공 → 그대로 유지
  } catch {
    // ③ 실패 → 롤백
    setLiked(prev);
    setLikeCount((c) => liked ? c + 1 : c - 1);
    setError('잠시 후 다시 시도해주세요.');
  }
}
```

---

# 패턴 F — fire-and-forget ⭐️⭐️⭐️

```tsx
// 결과를 기다리지 않고 그냥 실행
void markAsRead(notificationId);
```

```txt
void를 쓰는 이유:
  await 없이 그냥 markAsRead(id)만 쓰면 → ESLint "unhandled promise" 경고
  void를 붙이면 → "의도적으로 무시함"을 명시

사용 상황:
  실패해도 사용자 경험에 영향 없는 부수 작업
  읽음 처리, 로그 기록, 통계 업데이트

void vs await 판단:
  await → 결과가 중요하거나 실패 시 사용자에게 알려야 할 때
  void  → 실패해도 상관없는 부수 작업
```

---

# 전체 골격 — 참고

```tsx
// ── 패턴 A: 초기 로드 ─────────────────────────────────────
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
}, []);

// ── 패턴 B/C: 액션 핸들러 ────────────────────────────────
const [isPending, setIsPending] = useState(false);

async function handleAction() {
  if (isPending) return;           // 중복 방지 가드
  setError('');
  setIsPending(true);
  try {
    await doSomething();
  } catch (err) {
    setError(err instanceof Error ? err.message : '실패했어요.');
  } finally {
    setIsPending(false);
  }
}

// ── 패턴 D: useTransition ─────────────────────────────────
const [isPending, startTransition] = useTransition();

function handleActionTransition() {
  if (isPending) return;
  startTransition(async () => {
    try {
      await doSomething();
      startTransition(() => { setData(...); });   // await 이후 재감싸기
    } catch {
      startTransition(() => { setError('실패'); });
    }
  });
}
```
