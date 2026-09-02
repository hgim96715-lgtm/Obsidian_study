---
aliases: [로딩 스피너, 스켈레톤, fallback, LoaderCircle, loading.tsx, Suspense]
tags: [React, NextJS]
relations:
  - "[[00_JS_Ecosystem_HomePage]]"
  - "[[React_AsyncUI]]"
  - "[[React_Lazy]]"
  - "[[React_LucideIcons]]"
---
# React_Suspense — Suspense · 로딩 경계

>[!info]
> Suspense = 자식 컴포넌트가 "아직 준비 안 됐어" 신호를 던지면 그 동안 fallback UI를 보여주는 경계.
> **useEffect 안에서 fetch하는 로딩에는 Suspense가 동작하지 않음** — 그건 [[React_AsyncUI]] 패턴으로 처리.

---

# ⚠️ Suspense ≠ useEffect 로딩 — 가장 중요한 오해 ⭐️⭐️⭐️⭐️⭐️

```tsx
// ❌ 이렇게 해도 Suspense는 발동하지 않음
function MovieSearch() {
  const [movies, setMovies] = useState([]);
  const [isLoading, setIsLoading] = useState(false);

  useEffect(() => {
    setIsLoading(true);
    searchMovies(query).then((data) => {
      setMovies(data);
      setIsLoading(false);
    });
  }, [query]);

  return <MovieList movies={movies} />;
}

// Suspense로 감싸도 fallback이 전혀 안 보임
<Suspense fallback={<Spinner />}>
  <MovieSearch />   {/* useEffect fetch → Suspense 발동 안 됨 */}
</Suspense>
```

```txt
이유:
  Suspense는 컴포넌트가 렌더링 중에 Promise를 throw해야 발동
  useEffect는 렌더링 완료 후에 실행 → throw가 일어나지 않음
  → Suspense는 useEffect fetch를 인식하지 못함

useEffect 기반 로딩 상태 관리 → [[React_AsyncUI]]:
  const [isLoading, setIsLoading] = useState(false);
  → 직접 isLoading state로 조건부 렌더링
```

## useEffect 로딩 — 실제 사용 패턴

```tsx
import { LoaderCircle, X } from 'lucide-react';

function MovieSearch({ query }: { query: string }) {
  const [movies, setMovies]   = useState<Movie[]>([]);
  const [isLoading, setIsLoading] = useState(false);
  const [error, setError]     = useState('');

  useEffect(() => {
    if (!query) return;
    let cancelled = false;

    async function search() {
      setError('');
      setIsLoading(true);
      try {
        const data = await searchMovies(query);
        if (!cancelled) setMovies(data);
      } catch {
        if (!cancelled) setError('검색에 실패했어요.');
      } finally {
        if (!cancelled) setIsLoading(false);
      }
    }
    void search();
    return () => { cancelled = true; };
  }, [query]);

  // isLoading으로 직접 조건부 렌더링 — Suspense 아님
  if (isLoading) return (
    <div className="flex justify-center py-8">
      <LoaderCircle className="animate-spin text-lobby-gold" size={24} />
    </div>
  );
  if (error)     return <p className="text-red-500 text-sm">{error}</p>;
  if (!movies.length) return <p className="text-gray-400">검색 결과가 없어요.</p>;
  return <MovieList movies={movies} />;
}
```

---

# Suspense가 실제로 발동하는 것들 ⭐️⭐️⭐️⭐️⭐️

```txt
Suspense는 렌더링 중 Promise throw를 감지해서 작동

발동하는 상황:
  ① Next.js async Server Component   → await 중에 자동 throw
  ② React.lazy()                     → JS 파일 다운로드 중 자동 throw
  ③ useSearchParams()                → 정적 렌더링 시 자동 throw

발동하지 않는 상황:
  ④ useEffect 안의 fetch             → 렌더링 후에 실행 → throw 없음
  ⑤ 일반 이벤트 핸들러 비동기        → 마찬가지
```

---

# ① Next.js async Server Component ⭐️⭐️⭐️⭐️

```tsx
// PostList.tsx — async 컴포넌트 (서버에서 실행)
async function PostList() {
  const posts = await db.post.findMany();  // 이 await 동안 Suspense 발동
  return <ul>{posts.map(p => <li key={p.id}>{p.title}</li>)}</ul>;
}

// page.tsx
export default function Page() {
  return (
    <main>
      <Header />         {/* 즉시 렌더링 */}
      <Suspense fallback={<PostSkeleton />}>
        <PostList />     {/* 데이터 오는 동안 PostSkeleton 표시 */}
      </Suspense>
    </main>
  );
}
```

```txt
Suspense 없이:
  page.tsx 전체가 PostList 완료를 기다린 후 HTML 한 번에 전송
  → 사용자는 흰 화면을 보다 한꺼번에 나타남

Suspense 있을 때 (스트리밍):
  Header 먼저 전송 → PostSkeleton → PostList 데이터 오면 교체
  → 사용자가 빈 화면 보는 시간 최소화
```

---

# ② React.lazy() — 코드 지연 로딩 ⭐️⭐️⭐️⭐️

```tsx
import { lazy, Suspense } from 'react';

// 처음 렌더링 전까지 JS 파일 다운로드 안 함
const HeavyEditor = lazy(() => import('./HeavyEditor'));

function App() {
  const [show, setShow] = useState(false);
  return (
    <>
      <button onClick={() => setShow(true)}>에디터 열기</button>
      {show && (
        <Suspense fallback={<div>에디터 로딩 중...</div>}>
          <HeavyEditor />   {/* 첫 렌더링 시 JS 다운로드 → 그 동안 fallback */}
        </Suspense>
      )}
    </>
  );
}
```

```txt
lazy() 쓰는 이유:
  무거운 컴포넌트(에디터, 차트, 모달)를 초기 번들에서 제외
  → 실제 필요할 때 다운로드 → 첫 화면 로딩 빠름
```

---

# ③ useSearchParams() — Next.js 정적 렌더링 ⭐️⭐️⭐️⭐️

```tsx
// ❌ 빌드 에러
// "useSearchParams() should be wrapped in a suspense boundary"
export default function LoginPage() {
  return <LoginForm />;   // LoginForm 내부에서 useSearchParams() 호출
}

// ✅ 해결: useSearchParams() 쓰는 컴포넌트를 Suspense로 감싸기
export default function LoginPage() {
  return (
    <Suspense fallback={<p>불러오는 중…</p>}>
      <LoginForm />
    </Suspense>
  );
}
```

```txt
이유:
  정적 렌더링 시 빌드 타임에 URL을 알 수 없음
  → useSearchParams()가 "아직 준비 안 됨" throw
  → 감싸줄 Suspense 없으면 빌드 에러

패턴:
  useSearchParams()를 쓰는 컴포넌트를 별도 파일로 분리
  page.tsx에서 그 컴포넌트를 <Suspense>로 감싸기
```

---

# fallback — 로딩 중 보여줄 UI ⭐️⭐️⭐️⭐️

## lucide-react 스피너 패턴

```tsx
import { LoaderCircle } from 'lucide-react';

// 기본 스피너
<Suspense fallback={
  <div className="flex justify-center py-8">
    <LoaderCircle className="animate-spin text-lobby-gold" size={24} />
  </div>
}>
  <MovieList />
</Suspense>
```

```tsx
// 재사용 가능한 스피너 컴포넌트
function Spinner({ size = 24, className = '' }: { size?: number; className?: string }) {
  return (
    <div className="flex justify-center py-8">
      <LoaderCircle
        className={`animate-spin text-lobby-gold ${className}`}
        size={size}
      />
    </div>
  );
}

<Suspense fallback={<Spinner />}>
  <PostList />
</Suspense>
```

## fallback 값 선택

```txt
fallback={<Spinner />}   → 로딩 시간이 눈에 띌 때 (1초 이상)
fallback={<Skeleton />}  → 콘텐츠 형태를 미리 보여줄 때 (권장)
fallback={null}          → 로딩이 빠르거나 없어도 레이아웃 안 깨질 때
```

## 스켈레톤 UI

```tsx
// 콘텐츠 형태를 흐릿하게 미리 보여주는 패턴
function PostSkeleton() {
  return (
    <div className="space-y-4">
      {Array.from({ length: 3 }).map((_, i) => (
        <div key={i} className="h-24 animate-pulse rounded-lg bg-gray-700" />
      ))}
    </div>
  );
}

<Suspense fallback={<PostSkeleton />}>
  <PostList />
</Suspense>
```

---

# Suspense 위치 — 어디에 감싸는가 ⭐️⭐️⭐️⭐️

```tsx
// ❌ 너무 넓게 — 하나 때문에 전체가 fallback
<Suspense fallback={<PageSkeleton />}>
  <Header />    {/* 빠름 — 불필요하게 가려짐 */}
  <Sidebar />   {/* 빠름 — 불필요하게 가려짐 */}
  <PostList />  {/* 느림 */}
</Suspense>

// ✅ 느린 것만 감싸기
<>
  <Header />    {/* 즉시 보임 */}
  <Sidebar />   {/* 즉시 보임 */}
  <Suspense fallback={<PostSkeleton />}>
    <PostList />
  </Suspense>
</>
```

```tsx
// 중첩 Suspense — 각 영역이 독립적으로 로딩
export default function DashboardPage() {
  return (
    <div className="grid grid-cols-2 gap-4">
      <Suspense fallback={<SidebarSkeleton />}>
        <Sidebar />
      </Suspense>
      <Suspense fallback={<MainSkeleton />}>
        <MainContent />
      </Suspense>
    </div>
  );
}
```

```txt
중첩 Suspense:
  Sidebar가 느려도 → MainContent는 준비되면 먼저 보임
  서로 독립적으로 로딩

  하나의 Suspense로 감싸면:
  둘 중 하나라도 준비 안 되면 → 둘 다 fallback 표시
```

---

# Next.js — loading.tsx ⭐️⭐️⭐️

```txt
page.tsx 옆에 loading.tsx를 만들면
page.tsx 전체가 자동으로 Suspense에 감싸짐
```

```tsx
// app/feed/loading.tsx
import { LoaderCircle } from 'lucide-react';

export default function Loading() {
  return (
    <div className="flex justify-center py-20">
      <LoaderCircle className="animate-spin text-lobby-gold" size={32} />
    </div>
  );
}

// app/feed/page.tsx — loading.tsx가 자동으로 Suspense 감쌈
export default async function FeedPage() {
  const posts = await fetchPosts();   // 로딩 중 → loading.tsx 보임
  return <PostList posts={posts} />;
}
```

```txt
loading.tsx vs 직접 <Suspense>:
  loading.tsx   → 페이지 전체 로딩 UI (간편)
  <Suspense>    → 페이지 안 특정 영역만 (세밀한 제어)
  → 둘을 함께: loading.tsx로 기본 처리 + 내부에 <Suspense>로 세분화

error.tsx — 로딩 중 에러 발생 시 UI
  loading.tsx + error.tsx 세트로 관리
```

---

# 정리 — 로딩 처리 방법 선택 ⭐️⭐️⭐️⭐️⭐️

```txt
상황                                    → 사용 방법
────────────────────────────────────────────────────────────────
useEffect로 fetch (검색·조회 등)        → isLoading state + LoaderCircle
                                          [[React_AsyncUI]] 패턴
버튼 클릭 → 서버 요청                   → isPending state + disabled
                                          [[React_AsyncUI]] 패턴
Next.js async Server Component          → <Suspense> + fallback
무거운 컴포넌트 지연 로딩               → React.lazy() + <Suspense>
useSearchParams() 사용                  → <Suspense>로 감싸기 (빌드 에러 방지)
Next.js 페이지 전체 로딩 UI             → loading.tsx
```
