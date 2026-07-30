---
aliases: [callbackUrl, redirect, usePathname, useRouter]
tags: [React, NextJS]
related:
  - "[[00_JS_Ecosystem_HomePage]]"
  - "[[JS_URL_Encoding]]"
  - "[[NestJS_JwtGuard]]"
  - "[[NextJS_API_Client]]"
  - "[[NextJS_ServerClient]]"
  - "[[NextJS_TokenStorage]]"
---
# NextJS_Routing — 라우팅 & 이동

> [!info]
>  Next.js에서 페이지를 이동하는 방법은 세 가지 — `<Link>`, `router.push()`, `redirect()`. 
>  어디서 실행하는가(서버 vs 클라이언트)에 따라 선택이 달라진다.

---

# 세 가지 이동 수단 ⭐️⭐️⭐️⭐️

| |`<Link>`|`router.push()`|`redirect()`|
|---|---|---|---|
|실행 위치|JSX (렌더링)|클라이언트 이벤트 핸들러|서버 컴포넌트 / 서버 액션|
|언제|메뉴, 카드, 버튼|조건부 이동, 폼 제출 후|접근 제어, 경로 변경|
|HTTP 상태|—|—|307 (기본) / 308 (permanent)|

---

# `<Link>` — 기본 이동 ⭐️⭐️⭐️

```tsx
import Link from 'next/link';

// 정적 경로
<Link href="/about">소개</Link>

// 동적 경로
<Link href={`/users/${userId}`}>프로필</Link>

// 쿼리스트링
<Link href={{ pathname: '/search', query: { q: 'keyword' } }}>검색</Link>

// 새 탭
<Link href="/docs" target="_blank">문서</Link>
```

```txt
Link가 기본인 이유:
  Next.js가 자동으로 prefetch — 링크 위에 hover 하면 미리 로드
  SPA 방식으로 이동 — 전체 페이지 새로고침 없음
  <a> 태그로 렌더링 — 접근성 자동 처리

<a> 대신 Link를 쓰는 이유:
  <a href="..."> 는 전체 페이지 reload → 느림
  Link는 SPA 방식으로 클라이언트 사이드 이동 → 빠름
```

---

# router.push() — 이벤트 후 이동 ⭐️⭐️⭐️⭐️

```typescript
'use client';
import { useRouter } from 'next/navigation';

function LoginForm() {
  const router = useRouter();

  async function handleSubmit() {
    await login(credentials);
    router.push('/dashboard');     // 로그인 성공 → 이동
  }

  function handleCancel() {
    router.back();                 // 이전 페이지로
  }
}
```

|메서드|동작|
|---|---|
|`router.push(href)`|히스토리에 추가하며 이동|
|`router.replace(href)`|현재 히스토리를 교체하며 이동 (뒤로가기 불가)|
|`router.back()`|이전 페이지|
|`router.forward()`|다음 페이지|
|`router.refresh()`|서버 데이터 재요청 (SPA 유지)|

```txt
push vs replace:
  push     히스토리 스택에 추가 → 뒤로가기로 이전 페이지 돌아올 수 있음
  replace  현재 항목을 교체 → 뒤로가기로 못 돌아옴

  로그인 완료 후 이동: replace 권장 (로그인 페이지로 뒤로가기 방지)
  일반 탐색: push

router는 'use client' 컴포넌트에서만 사용 가능
```

---

# redirect() — 서버에서 이동 ⭐️⭐️⭐️⭐️

```typescript
import { redirect } from 'next/navigation';
```

## 언제 쓰는가

```txt
redirect()가 필요한 상황:

  1. 로그인 필요 → 로그인 페이지로
     Server Component에서 session 확인 → 없으면 redirect('/login')

  2. 권한 없음 → 접근 거부
     역할 확인 → 관리자 아니면 redirect('/403')

  3. 폐기된 경로 → 새 경로로
     예전 /archive → 새 /recommendations 로 통합

  4. 조건에 따른 랜딩 분기
     이미 로그인 → /dashboard (로그인 페이지에 접근 시)
     첫 방문 → /onboarding

  5. 데이터 없음 → 404 대신 다른 경로로
     존재하지 않는 roomId → /rooms (목록으로)
```

## 사용 패턴

```typescript
// 1. Server Component — 렌더링 전에 이동
export default async function ArchivePage() {
  redirect('/recommendations');   // 이 컴포넌트는 렌더링 안 됨
}

// 2. 로그인 확인 후 이동
export default async function DashboardPage() {
  const session = await getSession();
  if (!session) redirect('/login');

  return <Dashboard />;
}

// 3. 데이터 없으면 이동
export default async function RoomPage({ params }: { params: { id: string } }) {
  const room = await fetchRoom(params.id);
  if (!room) redirect('/rooms');  // 존재하지 않는 방 → 목록으로

  return <RoomView room={room} />;
}

// 4. Server Action — 폼 제출 후 이동
async function createPost(formData: FormData) {
  'use server';
  const post = await savePost(formData);
  redirect(`/posts/${post.id}`);  // 생성된 게시글로
}
```

```txt
redirect()의 동작:
  NEXT_REDIRECT 에러를 throw해서 렌더링을 중단하고 이동
  → redirect() 이후 코드는 실행 안 됨 (return 불필요)
  → try-catch 안에서 쓰면 catch가 잡아버림 → try 밖에서 사용

  // ❌ try 안에서 redirect
  try {
    const data = await fetch(...);
    redirect('/next');   // catch가 잡아서 이동 안 됨
  } catch { ... }

  // ✅ try 밖에서 redirect
  const data = await fetch(...).catch(() => null);
  if (!data) redirect('/error');

HTTP 상태 코드:
  redirect()           → 307 Temporary Redirect
  permanentRedirect()  → 308 Permanent Redirect (SEO: 구 URL → 신 URL 영구 이동)
```

## redirect vs router.push 선택

```txt
redirect():
  서버 컴포넌트, Server Action에서만 가능
  렌더링 전에 이동 → 보안 체크, 접근 제어에 적합
  클라이언트 코드가 아예 실행 안 됨

router.push():
  'use client' 컴포넌트에서만 가능
  이벤트(클릭, 폼 제출) 후 이동
  이동 전에 추가 처리 가능 (상태 정리, 알림 등)

  서버에서 접근 제어 → redirect()
  클라이언트 이벤트 후 이동 → router.push()
```

---

# notFound() — 404 처리 ⭐️⭐️⭐️

```typescript
import { notFound } from 'next/navigation';

export default async function PostPage({ params }: { params: { id: string } }) {
  const post = await fetchPost(params.id);
  if (!post) notFound();   // → app/not-found.tsx 렌더링

  return <PostView post={post} />;
}
```

```txt
redirect vs notFound:
  redirect('/목록')   → "이 경로는 저쪽으로 가세요"
  notFound()         → "이 경로는 존재하지 않습니다" (404)

  존재하지 않는 콘텐츠: notFound() (SEO에서 404가 정확)
  이동시키고 싶을 때:   redirect()
```

---

# usePathname / useSearchParams ⭐️⭐️⭐️

```typescript
'use client';
import { usePathname, useSearchParams } from 'next/navigation';

function NavItem({ href }: { href: string }) {
  const pathname = usePathname();
  const isActive = pathname === href || pathname.startsWith(`${href}/`);

  return (
    <Link href={href} className={isActive ? 'font-bold' : ''}>
      ...
    </Link>
  );
}

function SearchPage() {
  const searchParams = useSearchParams();
  const query = searchParams.get('q') ?? '';
}
```

---

# 동적 경로 — [id], [...slug] ⭐️⭐️⭐️

```typescript
// app/rooms/[id]/page.tsx
export default function RoomPage({
  params,
  searchParams,
}: {
  params:       { id: string };
  searchParams: { tab?: string };
}) {
  const { id } = params;
  const tab = searchParams.tab ?? 'chat';
}

// app/docs/[...slug]/page.tsx — 여러 세그먼트
// /docs/a/b/c → params.slug = ['a', 'b', 'c']
export default function DocsPage({ params }: { params: { slug: string[] } }) {}

// app/docs/[[...slug]]/page.tsx — 선택적
// /docs 도, /docs/a/b 도 매칭
```

---

# 한눈에

```txt
<Link href="...">     JSX에서 정적/동적 이동 — 기본 이동 수단
router.push(href)     클라이언트 이벤트 후 이동 ('use client' 필요)
router.replace(href)  이동 + 뒤로가기 차단
redirect(href)        서버에서 이동 — 렌더링 전 접근 제어/경로 변경
notFound()            404 처리

redirect() 주요 용도:
  미인증 → /login
  권한 없음 → /403
  폐기 경로 → 새 경로
  데이터 없음 → 목록

redirect() 주의:
  try-catch 밖에서 사용
  이후 코드 실행 안 됨 (return 불필요)
  서버 컴포넌트 / Server Action 전용
```