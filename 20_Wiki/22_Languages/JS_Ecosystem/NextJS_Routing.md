---
aliases:
  - useSearchParams
  - useParams
  - Link
  - usePathname
  - useRouter
  - redirect
  - window.location.replace
  - middleware
tags:
  - React
  - NextJS
related:
  - "[[00_JS_Ecosystem_HomePage]]"
  - "[[JS_URL_Encoding]]"
  - "[[TS_Generics]]"
  - "[[React_Concept]]"
---
# NextJS_Routing — Next.js 라우팅

>[!info]
>Next.js App Router = 폴더 구조가 곧 URL.
> `app/posts/[id]/page.tsx` → `/posts/:id`. 
> `<Link>`로 클라이언트 이동(SPA 방식 — 새로고침 없이 화면만 교체),
>  `useRouter()`로 프로그래밍 이동, `useParams()`로 경로 파라미터 읽기. 
>  SPA 개념 → [[React_Concept]], 쿼리스트링(`?q=...`) → [[JS_URL_Encoding]]

---

# App Router란 ⭐️⭐️⭐️⭐️

```txt
Next.js의 라우팅 방식:
  폴더 구조 = URL 구조
  app/ 아래 폴더를 만들면 그게 그대로 URL 경로가 됨

  app/
  ├── page.tsx          → /
  ├── posts/
  │   └── page.tsx      → /posts
  └── settings/
      └── page.tsx      → /settings

파일이 라우트를 만드는 것이 아니라 폴더가 라우트를 만듦
page.tsx = "이 경로에서 보여줄 컴포넌트"
```

---

# 폴더 구조 → URL 매핑 ⭐️⭐️⭐️⭐️

## 정적 경로

```txt
app/
├── page.tsx                      → /
├── about/
│   └── page.tsx                  → /about
├── posts/
│   └── page.tsx                  → /posts
└── admin/
    ├── page.tsx                  → /admin
    ├── users/
    │   └── page.tsx              → /admin/users
    └── settings/
        └── page.tsx              → /admin/settings
```

## 동적 경로 — [파라미터]

```txt
app/
└── posts/
    ├── page.tsx                  → /posts
    └── [id]/
        └── page.tsx              → /posts/123, /posts/abc-xyz

└── rooms/
    └── [roomId]/
        ├── page.tsx              → /rooms/123
        └── messages/
            └── [msgId]/
                └── page.tsx     → /rooms/123/messages/456
```

```txt
[id] 폴더:
  대괄호로 감싸면 동적 세그먼트
  폴더명(id)이 파라미터 이름 → params.id로 접근
  /posts/abc → params.id = 'abc'

여러 파라미터:
  /rooms/[roomId]/messages/[msgId]
  → params = { roomId: '123', msgId: '456' }
```

## 특수 폴더 패턴

```txt
(group)/            → URL에 포함 안 됨 (라우트 그룹 — 레이아웃 분리용)
_folder/            → URL에 포함 안 됨 (비공개 폴더)
[...slug]/          → catch-all (0개 이상의 세그먼트)
[[...slug]]/        → optional catch-all (없어도 됨)

예시:
  (auth)/login/page.tsx  → /login  (auth는 URL에 없음)
  [...slug]/page.tsx     → /a, /a/b, /a/b/c 전부 매칭
```

---

# 특수 파일들 ⭐️⭐️⭐️⭐️

```txt
각 폴더에 만들 수 있는 파일들:
  page.tsx       → 실제 페이지 UI (이게 있어야 라우트로 접근 가능)
  layout.tsx     → 이 경로와 하위 경로에 공통 적용되는 레이아웃
  loading.tsx    → page.tsx가 로딩 중일 때 보여줄 UI (Suspense 자동)
  error.tsx      → 에러 발생 시 UI (반드시 'use client')
  not-found.tsx  → 404 페이지
  route.ts       → API 엔드포인트 (GET, POST 등 HTTP 메서드 핸들러)
```

```typescript
// layout.tsx — 공통 레이아웃
export default function Layout({ children }: { children: React.ReactNode }) {
  return (
    <div>
      <Sidebar />
      <main>{children}</main>  {/* page.tsx가 여기 들어감 */}
    </div>
  );
}

// loading.tsx — 자동으로 Suspense로 감싸줌
export default function Loading() {
  return <Spinner />;
}

// error.tsx
'use client';
export default function Error({ error, reset }: { error: Error; reset: () => void }) {
  return <button onClick={reset}>다시 시도</button>;
}
```

---

# 경로 파라미터 읽기 ⭐️⭐️⭐️⭐️

## Server Component — params prop

```typescript
// app/posts/[id]/page.tsx
export default async function PostPage({
  params,
}: {
  params: { id: string };
}) {
  const { id } = params;
  const post = await prisma.post.findUnique({ where: { id } });
  return <PostDetail post={post} />;
}

// 여러 파라미터
export default async function MessagePage({
  params,
}: {
  params: { roomId: string; msgId: string };
}) {
  const { roomId, msgId } = params;
  ...
}
```

## Client Component — useParams() ⭐️⭐️⭐️⭐️

```typescript
'use client';
import { useParams } from 'next/navigation';

export function EditButton() {
  const { id } = useParams<{ id: string }>();
  //              ↑ 제네릭으로 타입 명시 필수

  return <button onClick={() => router.push(`/posts/${id}/edit`)}>수정</button>;
}
```

```txt
왜 useParams<{ id: string }>() 제네릭이 필요한가:
  제네릭 없이 useParams()만 쓰면:
  반환 타입: { [key: string]: string | string[] }
  → id가 string | string[] 이라서 타입 에러 발생 가능

  <{ id: string }>으로 명시하면:
  → id: string 으로 좁혀짐 → 타입 안전

  왜 string | string[]인가:
  [...slug] catch-all 경로는 string[]을 반환
  [id] 일반 경로는 string이지만 타입 상 둘 다 가능해서
  → 제네릭으로 "이건 string이야"라고 명시
```

---

# 이동 — Link vs useRouter ⭐️⭐️⭐️⭐️

## Link — 선언형 이동 (주로 사용)

```tsx
import Link from 'next/link';

// 기본
<Link href="/posts">게시글 목록</Link>

// 동적 경로
<Link href={`/posts/${post.id}`}>{post.title}</Link>

// 객체 형태
<Link href={{ pathname: '/posts', query: { page: 2 } }}>
  다음 페이지
</Link>
// → /posts?page=2

// 스타일 적용
<Link href="/about" className="text-blue-500">
  소개
</Link>
```

```txt
<Link>:
  HTML <a> 태그로 렌더링됨
  클릭 시 페이지 새로고침 없이 클라이언트 이동 (SPA 방식)
  뷰포트에 보이면 자동으로 prefetch (미리 로드)
  → 가장 기본적인 이동 방식, 사용자가 클릭해서 이동할 때
```

## useRouter — 프로그래밍 이동 ⭐️⭐️⭐️⭐️

```typescript
'use client';
import { useRouter } from 'next/navigation';
//                          ↑ 반드시 'next/navigation' (next/router 아님)

export function LoginForm() {
  const router = useRouter();

  async function handleSubmit() {
    await login(email, password);
    router.push('/dashboard');   // 로그인 성공 후 이동
  }

  return ...;
}
```

```typescript
// router 메서드
router.push('/posts')        // 이동 (뒤로가기 가능)
router.replace('/posts')     // 이동 (현재 히스토리 교체 — 뒤로가기 불가)
router.back()                // 뒤로가기
router.forward()             // 앞으로가기
router.refresh()             // 현재 페이지 새로고침 (Server Component 재실행)
router.prefetch('/posts')    // 미리 로드
```

```txt
Link vs useRouter:
  <Link>     → 사용자가 클릭해서 이동 (UI에서 이동)
  useRouter  → 코드에서 이동 (로그인 성공, 폼 제출 후 등)

  useRouter는 'use client' 필요 — Server Component에서 사용 불가
  Server Component에서 리다이렉트가 필요하면 redirect() 함수 사용
```

## push vs replace — 히스토리 차이 ⭐️⭐️⭐️⭐️

```txt
브라우저 히스토리 스택:
  페이지를 이동할 때마다 히스토리에 기록이 쌓임
  뒤로가기 = 스택에서 이전 항목으로 이동

  push:   스택에 새 항목 추가 → 뒤로가기 가능
  replace: 현재 항목을 교체  → 뒤로가기하면 그 전 페이지로 감

  A → B (push)   → 뒤로가기: A
  A → B (replace)→ 뒤로가기: A  (B는 히스토리에 없음)
```

```typescript
router.push('/posts')     // 히스토리에 추가 — 뒤로가기 가능
router.replace('/posts')  // 현재 항목 교체 — 뒤로가기하면 전전 페이지로
```

```txt
router.replace를 쓰는 경우:
  리다이렉트 — "이 페이지는 사실 저 페이지야"
  → 뒤로가기 눌러서 다시 리다이렉트되는 루프 방지

  ex) /login → 로그인 성공 → /dashboard (replace)
  → 뒤로가기 눌러도 /login으로 안 돌아감
  → push면: /dashboard → 뒤로가기 → /login → 또 리다이렉트 → /dashboard (루프)
```

## window.location.replace — 풀 리로드 + 히스토리 교체 ⭐️⭐️⭐️⭐️

```typescript
// 내 프로필 페이지인데 /users/me/album 으로 정규화
if (me?.id === id) {
  window.location.replace('/users/me/album');
}
```

```txt
window.location.replace(url):
  브라우저를 통째로 새 URL로 이동 (풀 페이지 리로드)
  현재 히스토리 항목을 교체 (뒤로가기 불가)

router.replace('/url'):
  SPA 방식으로 이동 (페이지 리로드 없음)
  현재 히스토리 항목을 교체

왜 위 코드에서 router.replace가 아닌 window.location.replace를 쓰는가:
  /users/123/album → "이 사람이 나인가?" 체크
  → 나이면 /users/me/album 으로 이동 (정규화)
  
  router.replace를 써도 동작은 같지만
  window.location.replace는 풀 리로드 → 새 URL로 완전히 재시작
  → React 상태 초기화, 서버에서 /users/me로 다시 렌더링
  
  me.id === id 체크를 서버에서 하면 redirect()로 처리
  클라이언트에서만 알 수 있는 상황(로그인 state)이라 window.location 사용
```

```txt
세 가지 비교:

  router.push('/url')             → SPA 이동, 히스토리 추가
  router.replace('/url')          → SPA 이동, 히스토리 교체
  window.location.replace('/url') → 풀 리로드, 히스토리 교체

언제 어떤 것을:
  일반 이동          → router.push
  리다이렉트         → router.replace  (루프 방지)
  완전한 재시작 필요  → window.location.replace
  서버에서 리다이렉트 → redirect()  (Server Component)
```

## redirect() — Server Component에서 이동

```typescript
import { redirect } from 'next/navigation';

// Server Component에서 조건부 이동
export default async function ProtectedPage() {
  const session = await getSession();
  if (!session) redirect('/login');  // 로그인 안 됐으면 이동

  return <ProtectedContent />;
}
```

---

# usePathname — 현재 경로 ⭐️⭐️⭐️

```typescript
'use client';
import { usePathname } from 'next/navigation';

export function NavLink({ href, children }) {
  const pathname = usePathname();
  const isActive = pathname === href;
  //                ↑ 현재 URL 경로 (쿼리스트링 제외)
  //                ex) /posts (? 이후 없음)

  return (
    <Link
      href={href}
      className={isActive ? 'font-bold text-blue-500' : ''}
    >
      {children}
    </Link>
  );
}
```

---

# useParams vs useSearchParams — 언제 뭘 ⭐️⭐️⭐️⭐️

```typescript
// URL: /posts/123?sort=latest&page=2

// 경로 파라미터 (/posts/[id]) → useParams
const { id } = useParams<{ id: string }>();
// id = '123'

// 쿼리스트링 (?sort=latest) → useSearchParams
const searchParams = useSearchParams();
const sort = searchParams.get('sort');  // 'latest'
const page = searchParams.get('page'); // '2'
```

```txt
useParams:
  경로 자체가 리소스 식별자
  /posts/123   → "123번 게시글"
  /users/hong  → "hong 사용자"
  → [id] 폴더로 정의된 동적 세그먼트

useSearchParams:
  경로 뒤의 필터·옵션 값
  /posts?sort=latest&page=2 → "게시글을 최신순 2페이지로"
  /search?q=홍길동           → "'홍길동' 검색"
  → URL_Encoding·URLSearchParams → [[JS_URL_Encoding]]

⚠️ useSearchParams()는 반드시 <Suspense> 안에 있는 컴포넌트에서만 사용
  정적 렌더링 시 Suspense 없으면 빌드 에러 발생
  → page.tsx에서 해당 컴포넌트를 <Suspense>로 감싸기
  → 원리·패턴 [[React_Suspense#③ useSearchParams()]] · 에러 기록 [[Next_Troubleshooting]]
```

---

# 훅 사용 가능 위치 정리

| 훅 / 함수              | Server | Client | 설명                          |
| ------------------- | ------ | ------ | --------------------------- |
| `params` prop       | ✅      | ❌      | Server Component에서 경로 파라미터  |
| `searchParams` prop | ✅      | ❌      | Server Component에서 쿼리스트링    |
| `useParams()`       | ❌      | ✅      | Client Component에서 경로 파라미터  |
| `useSearchParams()` | ❌      | ✅      | Client Component에서 쿼리스트링    |
| `useRouter()`       | ❌      | ✅      | Client Component에서 프로그래밍 이동 |
| `usePathname()`     | ❌      | ✅      | Client Component에서 현재 경로    |
| `redirect()`        | ✅      | ❌      | Server Component에서 리다이렉트    |
| `<Link>`            | ✅      | ✅      | 어디서나 사용 가능                  |

---

# Middleware 리다이렉트 — 쿼리 파라미터 우회 플래그 ⭐️⭐️⭐️⭐️

```txt
패턴:
  특정 경로(ex. /)에 접근 시 미들웨어가 자동으로 다른 경로로 리다이렉트
  단, 특정 쿼리 파라미터가 있으면 리다이렉트를 건너뜀

예시:
  /              → 관리자 감지 → /admin 자동 이동
  /?lobby=1      → lobby=1 플래그 감지 → 리다이렉트 없이 / 그대로 표시
```

```typescript
// middleware.ts
import { NextResponse } from 'next/server';
import type { NextRequest } from 'next/server';

export function middleware(request: NextRequest) {
  const { pathname, searchParams } = request.nextUrl;

  // / 접근 시 관리자이면 /admin으로 자동 이동
  // /?lobby=1 이면 우회 — 로비를 그대로 표시
  if (pathname === '/' && !searchParams.get('lobby')) {
    return NextResponse.redirect(new URL('/admin', request.url));
  }
}
```

```typescript
// 관리자 화면에서 로비로 이동하는 링크
// ❌ 잘못된 사용 — / 접근 시 다시 /admin으로 리다이렉트됨
<Link href="/">로비로</Link>

// ✅ 올바른 사용 — lobby=1 플래그로 미들웨어 우회
<Link href="/?lobby=1">로비로</Link>
```

```txt
왜 이 패턴이 필요한가:
  미들웨어는 경로(pathname)만으로 리다이렉트를 판단
  → /admin에 있는 관리자가 /로 가면 → 또 /admin으로 튕겨나감
  → 관리자가 실제 로비(/)를 보려면 우회 수단이 필요

쿼리 파라미터 우회 플래그:
  ?lobby=1 처럼 "리다이렉트 건너뜀" 신호를 쿼리 파라미터로 전달
  미들웨어가 해당 파라미터를 보고 → redirect 없이 통과
  로비 페이지 자체에서는 이 파라미터를 신경 쓸 필요 없음 (단순 무시)

주의:
  이 플래그는 URL에 남아있음 → /?lobby=1이 로비 URL이 됨
  보기 싫으면 로비 page.tsx에서 router.replace('/')로 파라미터 제거 가능
  (하지만 replace 시 미들웨어 재실행 → 다시 /admin으로 튕길 수 있으므로 주의)
```

```txt
범용 패턴:
  미들웨어 리다이렉트를 특정 상황에서만 건너뛰어야 할 때
  쿼리 파라미터를 "우회 플래그"로 사용하는 것이 관례

  다른 예:
    /  → 로그인 사용자를 /dashboard로 → /?noRedirect=1로 우회
    /  → 지역에 따라 /ko 또는 /en으로 → /?locale=skip으로 우회
```
