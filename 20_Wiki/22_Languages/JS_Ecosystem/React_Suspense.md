---
aliases:
  - Suspense
  - fallback
tags:
  - React
  - NextJS
related:
  - "[[00_JS_Ecosystem_HomePage]]"
  - "[[React_Lazy]]"
---
# React_Suspense — Suspense · 로딩 경계

>[!info]
>Suspense = 자식 컴포넌트가 "아직 준비 안 됐어"라고 신호를 보내면 그 동안 `fallback` UI를 대신 보여주는 로딩 경계. 
>준비 완료 신호가 오면 fallback이 사라지고 자식 컴포넌트가 나타난다. 
>Next.js의 async Server Component, `React.lazy()` 지연 로딩이 이 신호를 보낸다.

---

# Suspense가 해결하는 문제 ⭐️⭐️⭐️⭐️

```tsx
// ❌ Suspense 없이 — 데이터 올 때까지 페이지 전체가 멈춤
export default async function Page() {
  const posts = await fetchPosts();  // 3초 걸림
  return <PostList posts={posts} />;
  // 3초 동안 사용자는 아무것도 못 봄 (흰 화면)
}
```

```tsx
// ✅ Suspense 사용 — 빠른 부분 먼저 보여주고, 느린 부분은 준비되면 추가
export default function Page() {
  return (
    <main>
      <Header />          {/* 즉시 보임 — 기다릴 필요 없음 */}
      <Sidebar />         {/* 즉시 보임 */}

      <Suspense fallback={<PostSkeleton />}>
        <PostList />      {/* 데이터 오는 동안 PostSkeleton 보임 */}
      </Suspense>         {/* 데이터 오면 PostList로 교체 */}
    </main>
  );
}
```

```txt
Suspense 없이 SSR:
  서버에서 모든 데이터가 준비될 때까지 기다렸다가 HTML 한 번에 전송
  → 사용자는 흰 화면을 보다가 한꺼번에 화면이 나타남

Suspense 사용 (스트리밍):
  준비된 부분(Header, Sidebar)을 먼저 전송
  데이터가 필요한 부분(PostList)은 준비되면 그때 추가
  → 사용자가 빈 화면 보는 시간 최소화
```

---

# Suspense의 동작 원리 ⭐️⭐️⭐️⭐️

```txt
Suspense는 "특별한 신호"를 감지해서 작동

자식 컴포넌트가 렌더링 중에 Promise를 던지면(throw):
  React가 이것을 감지 → "아직 준비 안 됐구나"
  → 가장 가까운 Suspense의 fallback을 보여줌
  → Promise가 resolve되면 → 자식을 다시 렌더링 → fallback을 자식으로 교체

개발자가 Promise를 직접 throw하지 않아도 됨:
  Next.js async Server Component: 내부적으로 처리
  React.lazy(): 내부적으로 처리
  → 이 두 가지를 쓰면 자동으로 Suspense와 연동됨
```

---

# fallback — 로딩 중 보여줄 것 ⭐️⭐️⭐️⭐️

```tsx
// fallback에 올 수 있는 것들
<Suspense fallback={<LoadingSpinner />}>   {/* 컴포넌트 */}
<Suspense fallback={<p>로딩 중...</p>}>   {/* JSX */}
<Suspense fallback={null}>                {/* 아무것도 안 보임 */}
```

## fallback={null}을 쓰는 이유

```tsx
<Suspense fallback={null}>
  <FeedList />
</Suspense>
```

```txt
null을 쓰는 경우:
  ① 로딩 시간이 매우 짧을 때
     0.1초 만에 로딩되는데 스피너가 반짝이다 사라지면 → 오히려 UX가 나빠짐
     null이면 → 아무것도 없다가 → 바로 컨텐츠

  ② 부모가 이미 로딩 상태를 처리할 때
     페이지 전체 로딩이 loading.tsx에서 처리되고 있다면
     내부 Suspense는 null로 조용히 처리

  ③ 없어도 레이아웃이 무너지지 않는 부분
     있으면 좋지만 없어도 괜찮은 컴포넌트

fallback을 쓰는 경우:
  데이터 로딩이 눈에 띄게 걸릴 때 (1초 이상)
  콘텐츠 없는 빈 공간이 레이아웃을 깨뜨릴 때
  사용자에게 "로딩 중"임을 명확히 알려야 할 때
```

---

# Suspense를 발동시키는 것들 ⭐️⭐️⭐️⭐️

## ① Next.js async Server Component

```tsx
// PostList.tsx — async 함수
async function PostList() {
  const posts = await db.post.findMany();  // 이 await 동안 Suspense 발동
  return <ul>{posts.map(p => <li key={p.id}>{p.title}</li>)}</ul>;
}

// Page.tsx
export default function Page() {
  return (
    <Suspense fallback={<PostSkeleton />}>
      <PostList />   {/* async 함수 → 데이터 올 때까지 Suspense 발동 */}
    </Suspense>
  );
}
```

```txt
Next.js App Router에서:
  async 함수 컴포넌트 = 서버에서 실행되는 컴포넌트
  await 중일 때 Suspense fallback 표시
  await 완료 → 렌더링 결과를 클라이언트로 스트리밍

중요:
  Suspense가 없으면 → page.tsx 전체가 await 완료될 때까지 전송 안 됨
  Suspense가 있으면 → 준비된 부분 먼저 전송, PostList는 준비되면 추가 전송
```

## ② React.lazy() — 코드 지연 로딩

```tsx
import { lazy, Suspense } from 'react';

// 이 파일은 처음 렌더링될 때까지 다운로드하지 않음
const HeavyEditor = lazy(() => import('./HeavyEditor'));

function App() {
  const [showEditor, setShowEditor] = useState(false);

  return (
    <>
      <button onClick={() => setShowEditor(true)}>에디터 열기</button>

      {showEditor && (
        <Suspense fallback={<div>에디터 로딩 중...</div>}>
          <HeavyEditor />  {/* 처음 보여질 때 JS 파일 다운로드 → 다운로드 중 Suspense 발동 */}
        </Suspense>
      )}
    </>
  );
}
```

```txt
React.lazy()가 필요한 이유:
  모든 컴포넌트를 처음부터 다 로드하면 초기 번들이 커짐
  → 첫 화면 로딩이 느려짐

  lazy()를 쓰면:
  처음 화면에 필요 없는 컴포넌트(에디터, 차트, 모달 등)를
  실제 필요할 때 다운로드 → 초기 로딩 빠름
```

---

# Suspense 위치 — 어디에 감싸는가 ⭐️⭐️⭐️⭐️

```tsx
// ❌ 너무 넓게 감싸면 — 하나 때문에 전체가 fallback
<Suspense fallback={<PageSkeleton />}>
  <Header />     {/* 빠름 — 기다릴 필요 없는데 가려짐 */}
  <Sidebar />    {/* 빠름 — 기다릴 필요 없는데 가려짐 */}
  <PostList />   {/* 느림 */}
</Suspense>

// ✅ 느린 컴포넌트만 감싸기
<>
  <Header />     {/* 즉시 보임 */}
  <Sidebar />    {/* 즉시 보임 */}
  <Suspense fallback={<PostSkeleton />}>
    <PostList />   {/* 얘만 로딩 중 표시 */}
  </Suspense>
</>
```

## 중첩 Suspense — 독립적인 로딩

```tsx
export default function DashboardPage() {
  return (
    <div className="grid grid-cols-2 gap-4">
      {/* 사이드바와 메인이 독립적으로 로딩 */}
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
중첩 Suspense의 장점:
  Sidebar가 느려도 → MainContent는 준비되면 먼저 보임
  MainContent가 느려도 → Sidebar는 먼저 보임
  서로 독립적으로 로딩

  중첩 없이 하나의 Suspense:
  둘 중 하나라도 준비 안 되면 → 둘 다 fallback 표시
```

---

# Next.js — loading.tsx ⭐️⭐️⭐️

```txt
Next.js에서 page.tsx 옆에 loading.tsx를 만들면
page.tsx 전체가 자동으로 Suspense에 감싸짐
별도로 <Suspense>를 작성하지 않아도 됨
```

```tsx
// app/feed/loading.tsx — page.tsx의 Suspense fallback
export default function Loading() {
  return (
    <div className="flex justify-center py-20">
      <Spinner />
    </div>
  );
}

// app/feed/page.tsx — loading.tsx가 자동으로 감쌈
export default async function FeedPage() {
  const posts = await fetchPosts();  // 로딩 중 → loading.tsx 보임
  return <PostList posts={posts} />;
}
```

```txt
loading.tsx vs 직접 <Suspense>:
  loading.tsx   → 페이지 전체 로딩 UI (간편)
  <Suspense>    → 페이지 안 특정 영역만 로딩 UI (세밀한 제어)

  둘을 함께 쓰기:
  loading.tsx로 페이지 전체 기본 처리
  + 내부에서 <Suspense>로 세분화

error.tsx:
  로딩 중 에러 발생 시 보여줄 UI
  loading.tsx(로딩) + error.tsx(에러) 세트로 관리
```

---

# 실전 패턴

## 스켈레톤 UI

```tsx
// 데이터 모양을 흐릿하게 미리 보여주는 UI
function PostSkeleton() {
  return (
    <div className="space-y-4">
      {Array.from({ length: 3 }).map((_, i) => (
        <div key={i} className="h-24 animate-pulse rounded-lg bg-gray-200" />
      ))}
    </div>
  );
}

<Suspense fallback={<PostSkeleton />}>
  <PostList />
</Suspense>
```

## 여러 컴포넌트 독립 로딩

```tsx
export default function RecommendationPage() {
  return (
    <main className="...">
      {/* FeedList만 Suspense로 감쌈 */}
      {/* fallback={null} — 빠르게 로딩돼서 스피너 불필요 */}
      <Suspense fallback={null}>
        <FeedList />
      </Suspense>
    </main>
  );
}
```

```txt
이 패턴을 쓰는 이유:
  main의 Header, 레이아웃 등은 즉시 렌더링
  FeedList만 데이터를 기다림
  fallback={null} — 빠른 로딩이라 스피너가 반짝이는 것보다 없는 게 더 자연스러움
```