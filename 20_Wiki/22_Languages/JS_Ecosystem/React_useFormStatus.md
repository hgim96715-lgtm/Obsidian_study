---
aliases: [React_useActionState, useActionState, useFormStatus]
tags: [React]
related:
  - "[[00_JS_Ecosystem_HomePage]]"
  - "[[NextJS_Server_Actions]]"
  - "[[React_ControlledInput]]"
---
# React_useFormStatus — useFormStatus

>[!info]
>`useFormStatus` = 부모 `<form>`의 제출 상태를 읽는 React 19 훅. 
>Server Actions와 함께 쓴다.
> `const { pending } = useFormStatus()` — `pending`이 true이면 제출 진행 중.
>  **같은 컴포넌트의 form이 아닌, 자식 컴포넌트에서만 동작한다.** 
>  Server Actions 개념 → [[NextJS_Server_Actions]]

---

# useFormStatus란 ⭐️⭐️⭐️⭐️

```txt
Server Actions와 함께 쓰는 훅
<form action={serverAction}> 이 제출 중인지 상태를 읽음

용도:
  Submit 버튼을 제출 중일 때 비활성화
  "저장 중..." 같은 로딩 텍스트 표시
  입력 필드를 제출 중에 비활성화
```

```typescript
import { useFormStatus } from 'react-dom';  // react-dom에서 import

function SubmitButton() {
  const { pending } = useFormStatus();

  return (
    <button type="submit" disabled={pending}>
      {pending ? '저장 중...' : '저장'}
    </button>
  );
}
```

---

# 핵심 제약 — 자식 컴포넌트에서만 ⭐️⭐️⭐️⭐️

```typescript
// ❌ 작동 안 함 — form과 같은 컴포넌트 안
function LoginForm() {
  const { pending } = useFormStatus();  // 항상 { pending: false }

  return (
    <form action={loginAction}>
      <button disabled={pending}>로그인</button>
    </form>
  );
}

// ✅ 작동 — 자식 컴포넌트로 분리
function SubmitButton() {
  const { pending } = useFormStatus();  // 부모 form 상태를 읽음

  return (
    <button type="submit" disabled={pending}>
      {pending ? '로그인 중...' : '로그인'}
    </button>
  );
}

function LoginForm() {
  return (
    <form action={loginAction}>
      <input name="email" />
      <input name="password" type="password" />
      <SubmitButton />  {/* 자식으로 분리 */}
    </form>
  );
}
```

```txt
왜 자식 컴포넌트에서만 작동하는가:
  useFormStatus는 "가장 가까운 부모 form"의 상태를 구독
  같은 컴포넌트에 form이 있으면 자기 form을 참조할 수 없음
  → 반드시 form 안의 자식 컴포넌트에서 호출해야 함

  이 규칙을 자주 헷갈려서 pending이 항상 false로 나옴
```

---

# useFormStatus가 반환하는 값

```typescript
const {
  pending,  // boolean — 제출 진행 중이면 true
  data,     // FormData | null — 제출 중인 폼 데이터
  method,   // string | null — "get" | "post"
  action,   // 현재 실행 중인 action 함수
} = useFormStatus();
```

```txt
실전에서는 pending만 주로 씀
data, method, action은 고급 케이스에서 사용
```

---

# 실전 패턴 ⭐️⭐️⭐️⭐️

## 공통 SubmitButton 컴포넌트

```typescript
// components/SubmitButton.tsx — 재사용 가능한 버튼
'use client';
import { useFormStatus } from 'react-dom';

type Props = {
  label:        string;
  loadingLabel?: string;
};

export function SubmitButton({ label, loadingLabel }: Props) {
  const { pending } = useFormStatus();

  return (
    <button
      type="submit"
      disabled={pending}
      className={pending ? 'opacity-50 cursor-not-allowed' : ''}
    >
      {pending ? (loadingLabel ?? '처리 중...') : label}
    </button>
  );
}

// 사용
<form action={saveAction}>
  <SubmitButton label="저장" loadingLabel="저장 중..." />
</form>
```

## 입력 필드도 비활성화

```typescript
// 제출 중 입력 필드도 막기
function FormField({ name }: { name: string }) {
  const { pending } = useFormStatus();

  return (
    <input
      name={name}
      disabled={pending}
      className={pending ? 'opacity-50' : ''}
    />
  );
}
```

---

# useFormStatus vs isSubmitting ⭐️⭐️⭐️

```txt
useFormStatus (react-dom):
  Server Actions 전용
  <form action={serverAction}>와 함께
  자식 컴포넌트에서만 사용 가능
  별도 상태 관리 불필요 (React가 자동으로 처리)

react-hook-form의 isSubmitting:
  클라이언트 폼 전용 (Client Component)
  Zod 스키마 검증 + API fetch와 함께
  같은 컴포넌트 안에서 사용 가능
  → [[React_FormValidation]]

언제 어떤 것을:
  Server Actions 쓰는 폼 → useFormStatus
  Zod + react-hook-form 쓰는 폼 → isSubmitting
```

---

# Server Actions와 함께 쓰는 전체 예시

```typescript
// app/settings/page.tsx
import { updateProfile } from '@/app/actions/profile';
import { SubmitButton } from '@/components/SubmitButton';

export default function SettingsPage() {
  return (
    <form action={updateProfile}>
      <input name="nickname" defaultValue="홍길동" />
      <textarea name="bio" defaultValue="안녕하세요" />
      <SubmitButton label="프로필 저장" loadingLabel="저장 중..." />
    </form>
  );
}

// app/actions/profile.ts
'use server';

export async function updateProfile(formData: FormData) {
  const nickname = formData.get('nickname') as string;
  const bio      = formData.get('bio') as string;

  await prisma.user.update({ ... });
  revalidatePath('/settings');
}
```

```txt
Server Actions 흐름:
  <form action={updateProfile}> 제출
  → React가 pending = true 로 설정
  → updateProfile(formData) 서버에서 실행
  → 완료 → pending = false

  useFormStatus().pending 이 이 상태를 읽음
  → SubmitButton이 자동으로 disabled/loading 처리
```