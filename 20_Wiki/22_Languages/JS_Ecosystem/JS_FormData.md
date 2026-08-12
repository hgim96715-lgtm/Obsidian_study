---
aliases:
  - FormData
  - multipart/form-data
  - append vs set
  - React 19
tags:
  - JavaScript
related:
  - "[[00_JS_Ecosystem_HomePage]]"
  - "[[NextJS_Server_Actions]]"
  - "[[React_FormValidation]]"
  - "[[React_useFormStatus]]"
---
# JS_FormData — FormData

>[!info]
>`FormData` = 폼 데이터(텍스트·파일)를 key-value 쌍으로 담는 브라우저 내장 객체. 
>`fetch` 파일 업로드에서 사용하고, Next.js Server Actions가 `FormData`를 인자로 받는다.
> `Content-Type: multipart/form-data`가 자동으로 설정됨. 
> Server Actions → [[NextJS_Server_Actions]]

---

# FormData란 ⭐️⭐️⭐️⭐️

```txt
FormData = 폼 데이터를 담는 객체
  텍스트와 파일을 함께 담을 수 있음
  fetch의 body로 전달 → 파일 업로드
  <form>의 제출 데이터를 나타냄 → Server Actions

보통 JSON으로 보내는 것과 다른 점:
  JSON → 텍스트 데이터만 (파일 불가)
  FormData → 텍스트 + 파일 동시에 가능
```

---

# 만드는 방법 ⭐️⭐️⭐️⭐️

## 직접 만들기

```typescript
const formData = new FormData();

// 텍스트 추가
formData.append('title', '게시글 제목');
formData.append('content', '내용입니다.');

// 파일 추가 (File 객체)
formData.append('image', file);
formData.append('image', file, 'custom-name.jpg');  // 파일 이름 지정

// 같은 key로 여러 값
formData.append('tags', 'react');
formData.append('tags', 'nextjs');
```

## form 요소에서 자동으로

```typescript
const form = document.querySelector('form');
const formData = new FormData(form);
// form의 모든 input[name]이 자동으로 담김

// React에서
function handleSubmit(e: React.FormEvent<HTMLFormElement>) {
  e.preventDefault();
  const formData = new FormData(e.currentTarget);
}
```

---

# FormData API — 읽기·쓰기 ⭐️⭐️⭐️

```typescript
const fd = new FormData();
fd.append('email', 'hong@example.com');
fd.append('role', 'admin');
fd.append('role', 'editor');
fd.append('file', imageFile);

// 읽기
fd.get('email')            // 'hong@example.com' (첫 번째 값)
fd.get('role')             // 'admin' (첫 번째만)
fd.getAll('role')          // ['admin', 'editor'] (전부)
fd.has('email')            // true
fd.has('phone')            // false

// 수정
fd.set('email', 'new@example.com')   // 기존 값 교체 (append는 추가)
fd.delete('role')                    // 삭제

// 순회
for (const [key, value] of fd.entries()) {
  console.log(key, value);
}
// 또는
Object.fromEntries(fd.entries())    // { email: '...', file: File }
```

```txt
append vs set:
  append → 같은 key로 값 추가 (기존 값 유지)
  set    → 기존 값을 교체 (하나만 남음)

get vs getAll:
  get     → 첫 번째 값만 반환 (없으면 null)
  getAll  → 모든 값을 배열로 반환
  → 체크박스, 다중 파일처럼 같은 key가 여러 개면 getAll
```

---

# fetch로 파일 업로드 ⭐️⭐️⭐️⭐️

```typescript
// 파일 선택 input
const fileInput = document.querySelector<HTMLInputElement>('input[type=file]');
const file = fileInput?.files?.[0];

if (!file) return;

const formData = new FormData();
formData.append('image', file);
formData.append('title', '이미지 제목');

const res = await fetch('/api/upload', {
  method: 'POST',
  body:   formData,
  // ❌ Content-Type 헤더를 직접 설정하면 안 됨
  // → 브라우저가 boundary 포함해서 자동 설정
});
```

```txt
⚠️ Content-Type 헤더를 직접 설정하면 안 되는 이유:
  multipart/form-data는 boundary 값이 포함돼야 함
  예: Content-Type: multipart/form-data; boundary=----WebKitFormBoundary...

  직접 'multipart/form-data'만 쓰면 boundary가 없어서 서버 파싱 실패
  → headers에 Content-Type을 아예 쓰지 않으면 브라우저가 자동으로 올바르게 설정
```

## React에서 파일 업로드

```typescript
'use client';
import { useState } from 'react';

export function ImageUpload() {
  const [preview, setPreview] = useState<string>('');

  async function handleUpload(e: React.ChangeEvent<HTMLInputElement>) {
    const file = e.target.files?.[0];
    if (!file) return;

    // 미리보기
    setPreview(URL.createObjectURL(file));

    // 업로드
    const formData = new FormData();
    formData.append('image', file);

    const res = await fetch('/api/images', {
      method: 'POST',
      body:   formData,
    });
    const { url } = await res.json();
  }

  return (
    <div>
      <input type="file" accept="image/*" onChange={handleUpload} />
      {preview && <img src={preview} alt="preview" />}
    </div>
  );
}
```

---
# React 19 — form action에 클라이언트 함수 ⭐️⭐️⭐️⭐️

```txt
React 19부터 <form action={fn}>에 클라이언트 async 함수를 넘길 수 있음
제출 시 React가 fn(formData) 자동 호출

Server Actions ('use server') 와 같은 문법이지만
'use client' 컴포넌트 안에서 선언한 일반 함수
→ 브라우저에서 실행, fetch 호출 가능
→ [[React_FormValidation]] React 19 form action 패턴 섹션
```

```typescript
'use client';
import { useState } from 'react';

export default function LoginPage() {
  const [error, setError] = useState<string | null>(null);

  async function login(formData: FormData) {
    // formData = form 안의 name 속성 있는 input 전부 담김
    const email    = formData.get('email');
    const password = formData.get('password');

    // formData.get() = string | File | null
    // typeof 체크로 string임을 확정
    if (typeof email !== 'string' || typeof password !== 'string') {
      setError('입력을 확인해 주세요');
      return;
    }

    try {
      await loginRequest(email, password);
    } catch (err) {
      setError(err instanceof Error ? err.message : '실패했습니다.');
    }
  }

  return (
    <form action={login}>
      <input name="email"    type="email"    required />
      <input name="password" type="password" required />
      {error && <p>{error}</p>}
      <button type="submit">로그인</button>
    </form>
  );
}
```

```txt
onSubmit과의 차이:
  기존: <form onSubmit={e => { e.preventDefault(); ... }}>
  React 19: <form action={fn}> — e.preventDefault() 불필요
            React가 제출을 가로채서 fn(formData) 자동 호출

formData.get() 타입:
  string → 텍스트 input
  File   → input[type="file"]
  null   → name 없거나 체크박스 미체크

typeof email !== 'string' 체크:
  File이나 null이 넘어올 수 있어서 narrowing 필요
  통과하면 email, password는 string으로 확정
```

---

# Server Actions에서 FormData 받기 ⭐️⭐️⭐️⭐️

```typescript
// app/actions/posts.ts
'use server';

export async function createPost(formData: FormData) {
  // 텍스트 값
  const title   = formData.get('title') as string;
  const content = formData.get('content') as string;

  // 파일
  const image = formData.get('image') as File | null;
  if (image && image.size > 0) {
    // 파일 처리
  }

  // 체크박스 (체크됨 = 'on', 안 됨 = null)
  const isPublic = formData.get('isPublic') === 'on';

  // 다중 선택 (같은 name)
  const tags = formData.getAll('tags') as string[];

  await prisma.post.create({ data: { title, content, isPublic } });
}
```

```typescript
// 폼에서 사용
<form action={createPost}>
  <input name="title" />
  <textarea name="content" />
  <input name="image" type="file" />
  <input name="isPublic" type="checkbox" />
  <input name="tags" type="checkbox" value="react" />
  <input name="tags" type="checkbox" value="nextjs" />
  <button type="submit">저장</button>
</form>
```

```txt
Server Actions와 FormData:
  <form action={serverAction}> 제출 → serverAction(formData) 자동 호출
  name 속성이 있는 모든 input이 FormData에 담김
  formData.get('name') → 해당 input의 값

  타입 변환이 필요:
  formData.get() → File | string | null (항상 이 세 가지)
  숫자가 필요하면 → Number(formData.get('age'))
  불리언 → formData.get('check') === 'on'
```

---

# JSON 방식과 비교

```typescript
// JSON — 텍스트 데이터만, 파일 불가
fetch('/api/posts', {
  method:  'POST',
  headers: { 'Content-Type': 'application/json' },
  body:    JSON.stringify({ title, content }),
});

// FormData — 텍스트 + 파일 동시
const fd = new FormData();
fd.append('title', title);
fd.append('file', file);
fetch('/api/posts', { method: 'POST', body: fd });
```

```txt
언제 FormData:
  파일 업로드가 포함될 때
  <form> 기반 제출 (Server Actions)
  multipart/form-data가 필요할 때

언제 JSON:
  파일 없는 순수 데이터 전송
  NestJS API에 데이터 보낼 때 (대부분)
  타입 안전성이 중요할 때
```