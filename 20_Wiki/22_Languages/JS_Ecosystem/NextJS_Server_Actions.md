---
aliases:
  - Server Actions
  - use server
  - bind
tags:
  - NextJS
related:
  - "[[00_JS_Ecosystem_HomePage]]"
  - "[[React_useFormStatus]]"
  - "[[JS_FormData]]"
  - "[[NextJS_API_Client]]"
---
# NextJS_Server_Actions — Server Actions

>[!info]
>Server Actions = `'use server'`로 선언한 서버에서 실행되는 async 함수. 
>API Route 없이 클라이언트에서 서버 코드를 직접 호출할 수 있다.
> `<form action={serverAction}>`으로 폼 제출하거나 컴포넌트에서 직접 호출.
>  `useFormStatus` → [[React_useFormStatus]], fetch 기반 API 클라이언트 → [[NextJS_API_Client]]

---

# Server Actions란 — 왜 필요한가 ⭐️⭐️⭐️⭐️

```txt
기존 방식 (API Route):
  클라이언트 → POST /api/profile → route.ts → DB 업데이트
  route.ts 파일을 따로 만들어야 함

Server Actions:
  클라이언트 → updateProfile() 직접 호출 → DB 업데이트
  별도 API Route 파일 불필요
  함수 자체가 서버에서 실행됨

'use server'를 붙이면:
  이 함수는 서버에서만 실행됨
  클라이언트 번들에 포함되지 않음
  클라이언트에서 호출하면 → 네트워크 요청이 자동으로 생성됨
```

---

# 'use server' 선언 방법 ⭐️⭐️⭐️⭐️

```typescript
// 방법 1 — 파일 상단에 선언 (파일 전체가 Server Actions)
'use server';

export async function createPost(formData: FormData) { ... }
export async function updatePost(id: string, data: object) { ... }
// 이 파일의 모든 export 함수가 Server Action

// 방법 2 — 함수 안에 선언 (인라인 Server Action)
export default function Page() {
  async function handleSubmit(formData: FormData) {
    'use server';  // 이 함수만 Server Action
    // 서버 코드
  }

  return <form action={handleSubmit}>...</form>;
}
```

```txt
방법 1 (파일) 권장:
  재사용 가능 — 여러 컴포넌트에서 import해서 사용
  테스트하기 쉬움
  app/actions/ 폴더에 모아두는 것이 관례

방법 2 (인라인):
  해당 컴포넌트에서만 쓰는 작은 액션
  단, Server Component 안에서만 사용 가능
```

---

# 두 가지 사용 방법 ⭐️⭐️⭐️⭐️

## 방법 1 — form action

```typescript
// app/actions/profile.ts
'use server';
import { revalidatePath } from 'next/cache';

export async function updateProfile(formData: FormData) {
  const nickname = formData.get('nickname') as string;
  const bio      = formData.get('bio') as string;

  await prisma.user.update({
    where: { id: getCurrentUserId() },
    data:  { nickname, bio },
  });

  revalidatePath('/settings');  // 캐시 무효화 → 새 데이터 표시
}

// app/settings/page.tsx
import { updateProfile } from '@/app/actions/profile';
import { SubmitButton } from '@/components/SubmitButton';

export default function SettingsPage() {
  return (
    <form action={updateProfile}>
      <input name="nickname" />
      <textarea name="bio" />
      <SubmitButton label="저장" />
    </form>
  );
}
```

```txt
form action 방식:
  <form action={함수}> — 제출 시 Server Action 호출
  formData로 입력값을 받음 (formData.get('name'))
  JavaScript 없이도 동작 (Progressive Enhancement)
  useFormStatus로 pending 상태 읽기 가능
  → [[React_useFormStatus]]
```

## 방법 2 — 직접 호출

```typescript
// 버튼 클릭, 드롭다운 변경 등 form이 아닌 경우
'use client';
import { deletePost } from '@/app/actions/posts';

function DeleteButton({ postId }: { postId: string }) {
  return (
    <button onClick={() => deletePost(postId)}>
      삭제
    </button>
  );
}
```

```typescript
// app/actions/posts.ts
'use server';

export async function deletePost(postId: string) {
  await prisma.post.delete({ where: { id: postId } });
  revalidatePath('/posts');
}
```

```txt
직접 호출 방식:
  onClick, onChange 등 이벤트에서 호출
  일반 async 함수처럼 사용
  인자를 자유롭게 전달 가능 (formData 아닌 직접 값)
  await로 결과값 받기 가능
```

---

# formData 다루기 ⭐️⭐️⭐️

```typescript
export async function createPost(formData: FormData) {
  // 단일 값
  const title   = formData.get('title') as string;
  const content = formData.get('content') as string;

  // 파일
  const image = formData.get('image') as File;

  // 체크박스 (없으면 null)
  const isPublic = formData.get('isPublic') === 'on';

  // 여러 값 (같은 name)
  const tags = formData.getAll('tags') as string[];

  // Object.fromEntries로 한 번에
  const data = Object.fromEntries(formData.entries());
}
```

---

# revalidatePath · revalidateTag ⭐️⭐️⭐️⭐️

```typescript
import { revalidatePath, revalidateTag } from 'next/cache';

export async function updatePost(formData: FormData) {
  // DB 업데이트 ...

  revalidatePath('/posts');           // /posts 페이지 캐시 무효화
  revalidatePath('/posts/[id]', 'page');  // 동적 경로 타입 지정
  revalidatePath('/', 'layout');      // 레이아웃 포함 전체 무효화

  revalidateTag('posts');             // 'posts' 태그가 붙은 fetch 캐시 무효화
}
```

```txt
왜 revalidatePath가 필요한가:
  Next.js는 Server Component의 데이터를 캐싱함
  DB를 업데이트해도 캐시가 남아있으면 이전 데이터가 보임
  → revalidatePath로 해당 경로 캐시를 무효화
  → 다음 요청 시 서버에서 새 데이터를 가져옴

revalidatePath vs revalidateTag:
  revalidatePath('/posts') → 그 경로 전체 무효화
  revalidateTag('posts')   → 특정 태그가 붙은 fetch만 무효화 (더 세밀)
```

---

# redirect ⭐️⭐️⭐️

```typescript
import { redirect } from 'next/navigation';

export async function login(formData: FormData) {
  // 로그인 처리...
  const success = await authenticate(email, password);

  if (!success) {
    return { error: '이메일 또는 비밀번호가 틀렸습니다.' };
  }

  redirect('/dashboard');  // 성공 시 이동
  // redirect는 throw로 구현 → try/catch 안에 넣으면 안 됨
}
```

```txt
⚠️ redirect는 try/catch 밖에 두어야 함:
  redirect() 내부적으로 특수한 에러를 throw함
  try/catch로 잡히면 리다이렉트가 작동 안 함

  // ❌
  try {
    redirect('/dashboard');  // catch에 잡혀서 리다이렉트 안 됨
  } catch(e) { ... }

  // ✅
  if (success) redirect('/dashboard');  // try/catch 밖에서 호출
```

---

# 에러 처리 — 결과값 반환 패턴 ⭐️⭐️⭐️⭐️

```typescript
// Server Action에서 에러를 throw하면 클라이언트가 처리 어려움
// → 에러를 반환값으로 처리하는 패턴 권장

export async function createPost(formData: FormData) {
  const title = formData.get('title') as string;

  if (!title) {
    return { success: false, error: '제목을 입력해주세요.' };
  }

  try {
    await prisma.post.create({ data: { title } });
    revalidatePath('/posts');
    return { success: true };
  } catch {
    return { success: false, error: '저장에 실패했습니다.' };
  }
}
```

```typescript
// 클라이언트에서 결과값 받기 (직접 호출 방식)
'use client';
import { useState } from 'react';
import { createPost } from '@/app/actions/posts';

function CreatePostForm() {
  const [error, setError] = useState('');

  async function handleSubmit(formData: FormData) {
    const result = await createPost(formData);
    if (!result.success) {
      setError(result.error ?? '오류가 발생했습니다.');
    }
  }

  return (
    <form action={handleSubmit}>
      <input name="title" />
      {error && <p>{error}</p>}
      <button type="submit">저장</button>
    </form>
  );
}
```

---

# Server Action vs fetch API Route ⭐️⭐️⭐️

|구분|Server Action|fetch + API Route|
|---|---|---|
|파일|`app/actions/xxx.ts`|`app/api/xxx/route.ts`|
|호출|함수 직접 호출 / form action|`fetch('/api/xxx')`|
|인증|세션·쿠키 직접 접근|Authorization 헤더 필요|
|적합한 경우|Next.js 내부 폼·뮤테이션|외부 클라이언트(모바일 등)도 쓸 때|
|타입 안전|✅ 함수 타입 그대로|❌ fetch는 타입 단언 필요|

```txt
언제 Server Action:
  Next.js 웹 앱에서만 쓰는 폼 처리
  DB 업데이트 후 revalidate + redirect가 필요할 때

언제 fetch + API Route:
  모바일 앱, 다른 서비스도 같은 API를 써야 할 때
  NestJS 백엔드가 있을 때 (이 프로젝트)
  → 이 프로젝트는 NestJS가 API 서버 역할 → fetch 방식 사용
```

---

# 파일 구조 관례

```txt
app/
  actions/           Server Actions 파일 모음
    auth.ts          login · register · logout
    posts.ts         createPost · updatePost · deletePost
    profile.ts       updateProfile
  api/               API Route (필요한 경우만)
```