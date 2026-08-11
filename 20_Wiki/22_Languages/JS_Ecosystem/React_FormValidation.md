---
aliases:
  - Zod
  - 폼 검증
  - react-hook-form
  - isSubmitting
  - useForm
tags:
  - React
  - NextJS
related:
  - "[[00_JS_Ecosystem_HomePage]]"
  - "[[NestJS_DTO]]"
  - "[[JS_FormData]]"
---
# React_FormValidation — 폼 검증 (Zod + react-hook-form)

>[!info]
>클라이언트 폼 검증 표준 조합: `Zod 4`(스키마·타입추론) + `react-hook-form`(입력등록·에러·isSubmitting) + `@hookform/resolvers/zod`(연결). 
>Zod 4에서 이메일은 `z.email()` 권장 (`z.string().email()`은 구식). 
>NestJS 쪽은 `class-validator` DTO, Web 쪽은 Zod 스키마로 입구 검증을 양쪽에서 한다. 
>NestJS class-validator → [[NestJS_DTO]]

---

# 왜 이 조합인가 ⭐️⭐️⭐️⭐️

```txt
❌ 수동 검증 객체 (authFormErrors.ts 식):
  프로젝트마다 새로 만들어야 함
  타입·에러 메시지를 직접 유지
  작은 앱엔 괜찮지만 확장하면 비표준

✅ Zod + react-hook-form:
  Zod = 스키마로 검증 규칙 + 타입 자동 추론 (z.infer)
  react-hook-form = 입력 등록·에러 상태·isSubmitting
  @hookform/resolvers/zod = 둘을 연결

NestJS DTO ↔ Web 스키마 대칭:
  NestJS: class-validator @IsEmail() @MinLength(8)
  Web:    Zod z.email() .min(8)
  → "입구 검증"을 양쪽에서 하되 에러 메시지만 맞추면 됨

API 에러 (401·409 등)는 Zod 스키마 밖:
  → setError('root', { message: '...' })
  → 또는 setError('email', { message: '이미 사용 중인 이메일' })
```

---

# Zod — 스키마 검증 + 타입 추론 ⭐️⭐️⭐️⭐️

## 기본 타입

```typescript
import { z } from 'zod';

// 문자열
z.string()
z.string().min(1, '필수 항목입니다.')
z.string().min(8, '8자 이상이어야 합니다.')
z.string().max(20, '20자 이하로 입력해주세요.')
z.string().trim()             // 공백 제거 후 검증
z.string().url('올바른 URL 형식이 아닙니다.')

// ─────────────────────────────────────────────────
// 이메일 — Zod 3 vs Zod 4

// Zod 3 스타일 (아직 동작하지만 구식)
z.string().email('올바른 이메일 형식이 아닙니다.')

// Zod 4 권장 — z.email() (포맷 스키마)
z.email({ error: '올바른 이메일을 입력해주세요.' })

// Zod 4 — 빈 값 · trim 체크까지
z.string()
  .trim()
  .min(1, '이메일을 입력해주세요.')
  .pipe(z.email({ error: '올바른 이메일을 입력해주세요.' }))
// .pipe() = 앞 스키마를 통과한 값을 다음 스키마로 넘김
// trim + 빈 값 체크 → 이메일 형식 체크 순서

// 단순화 (빈 값 에러 메시지 구분 불필요할 때)
z.email({ error: '올바른 이메일을 입력해주세요.' })
// ─────────────────────────────────────────────────

// 숫자
z.number().min(0).max(100)
z.number().int('정수만 입력 가능합니다.')

// 선택형
z.boolean()
z.enum(['user', 'admin'])
z.literal('active')

// 선택적 (optional)
z.string().optional()         // string | undefined
z.string().nullable()         // string | null
z.string().nullish()          // string | null | undefined
```

## z.object — 폼 스키마 정의 ⭐️⭐️⭐️⭐️

```typescript
// z.object() = "이 객체는 이런 모양이어야 한다"는 스키마
const loginSchema = z.object({
  email:    z.email(),          // email 키는 이메일 형식
  password: z.string().min(1),  // password 키는 비어있으면 안 됨
});
```

```txt
z.object({ 키: 스키마, ... }):
  객체의 각 필드(키)에 검증 규칙을 붙이는 것
  폼의 input name과 키가 반드시 일치해야 함

  loginSchema의 키 = 'email', 'password'
  register('email'), register('password')  ← 이름이 같아야 연결됨

  키가 다르면:
  z.object({ emailAddress: z.email() })
  register('email')  ← 불일치 → 검증이 적용 안 됨
```


```typescript
// z.infer — 스키마에서 TypeScript 타입 자동 추론
type LoginValues = z.infer<typeof loginSchema>;
// = { email: string; password: string }

// useForm에 타입으로 전달 → errors.email, data.email 타입 안전
const { register } = useForm<LoginValues>({
  resolver: zodResolver(loginSchema),
});
```


```txt
z.infer<typeof schema>:
  스키마 정의 → TypeScript 타입이 자동으로 따라옴
  스키마를 바꾸면 타입도 자동으로 바뀜
  타입을 따로 손으로 쓸 필요 없음

  z.object({ email: z.email() })
    → type = { email: string }

  z.object({ email: z.email().optional() })
    → type = { email?: string }
```

```typescript
// lib/schemas/auth.ts
import { z } from 'zod';

// 공통 필드 — Zod 4 방식
const emailField = z
  .string()
  .trim()
  .min(1, '이메일을 입력해주세요.')
  .pipe(z.email({ error: '올바른 형식의 이메일을 입력해주세요.' }));
//      ↑ Zod 4 권장: z.email() 포맷 스키마
//        .pipe()로 앞 검증을 통과한 값을 이메일 형식 검증으로 넘김

const passwordField = z
  .string()
  .min(1, '비밀번호를 입력해주세요.')
  .min(8, '비밀번호는 8자 이상이어야 합니다.');

// Nest LoginDto와 맞춤
export const loginSchema = z.object({
  email:    emailField,
  password: z.string().min(1, '비밀번호를 입력해주세요.'),
});

// Nest RegisterDto — password MinLength(8)
export const registerSchema = z.object({
  email:    emailField,
  password: passwordField,
  nickname: z.string().trim().min(2, '닉네임은 2자 이상이어야 합니다.'),
});

// z.infer — 스키마에서 TypeScript 타입 자동 추론
export type LoginValues    = z.infer<typeof loginSchema>;
export type RegisterValues = z.infer<typeof registerSchema>;
```


```txt
z.infer<typeof schema>:
  스키마에서 TypeScript 타입을 자동으로 만들어줌
  스키마가 바뀌면 타입도 자동으로 바뀜
  → 따로 타입을 손으로 쓸 필요 없음

.trim():
  입력값 앞뒤 공백 제거 후 검증
  " a@b.com " → "a@b.com" 으로 처리하고 검증

체이닝 순서:
  .min(1) 먼저 → 빈 문자열 체크
  .email() 나중 → 이메일 형식 체크
  → 빈 칸이면 "입력해주세요." 메시지가 먼저 뜸
```

---

# react-hook-form — 폼 상태 관리 ⭐️⭐️⭐️⭐️

```bash
pnpm --filter web add react-hook-form zod @hookform/resolvers
```

```typescript
import { useForm } from 'react-hook-form';
import { zodResolver } from '@hookform/resolvers/zod';

const {
  register,         // 입력 필드 등록
  handleSubmit,     // 제출 핸들러 래퍼
  formState,        // errors, isSubmitting, isDirty 등
  setError,         // 수동으로 에러 설정 (API 에러용)
  reset,            // 폼 초기화
} = useForm<LoginValues>({
  resolver: zodResolver(loginSchema),  // Zod 스키마 연결
  defaultValues: { email: '', password: '' },
});

const { errors, isSubmitting } = formState;
```

```txt
useForm<LoginValues>:
  <LoginValues> 제네릭으로 타입 안전
  resolver: zodResolver(loginSchema) → 제출 시 Zod로 검증

isSubmitting:
  handleSubmit의 async 함수가 실행 중이면 true
  → 버튼 disabled 처리에 사용 (자동으로 관리됨)
```

---

# 전체 로그인 폼 예시 ⭐️⭐️⭐️⭐️

```typescript
// app/(auth)/login/page.tsx
'use client';

import { useForm } from 'react-hook-form';
import { zodResolver } from '@hookform/resolvers/zod';
import { loginSchema, type LoginValues } from '@/lib/schemas/auth';
import { login } from '@/lib/api';
import { useAuth } from '@/contexts/AuthContext';
import { useRouter } from 'next/navigation';

export default function LoginPage() {
  const { setUser } = useAuth();
  const router      = useRouter();

  const {
    register,
    handleSubmit,
    setError,
    formState: { errors, isSubmitting },
  } = useForm<LoginValues>({
    resolver:      zodResolver(loginSchema),
    defaultValues: { email: '', password: '' },
  });

  const onSubmit = async (data: LoginValues) => {
    // data는 Zod 검증 통과한 값 (타입 보장)
    try {
      const result = await login(data.email, data.password);
      setUser(result.user);
      router.push('/');
    } catch (err) {
      // API 에러 → 폼에 표시
      const message = err instanceof Error ? err.message : '로그인에 실패했습니다.';
      setError('root', { message });
      // 특정 필드에 표시하려면:
      // setError('email', { message: '이미 사용 중인 이메일입니다.' });
    }
  };

  return (
    <form onSubmit={handleSubmit(onSubmit)}>
      <div>
        <input
          {...register('email')}
          type="email"
          placeholder="이메일"
        />
        {errors.email && <p>{errors.email.message}</p>}
      </div>

      <div>
        <input
          {...register('password')}
          type="password"
          placeholder="비밀번호"
        />
        {errors.password && <p>{errors.password.message}</p>}
      </div>

      {/* API 에러 */}
      {errors.root && <p>{errors.root.message}</p>}

      <button type="submit" disabled={isSubmitting}>
        {isSubmitting ? '로그인 중...' : '로그인'}
      </button>
    </form>
  );
}
```

```txt
handleSubmit(onSubmit):
  Zod 검증 먼저 실행
  통과하면 onSubmit(data) 호출 — data는 검증된 값
  실패하면 onSubmit 호출 안 함 → errors에 자동 채워짐

{...register('email')}:
  name, onChange, onBlur, ref를 input에 한 번에 전달
  react-hook-form이 이 값을 추적

setError('root', { message }):
  'root'는 특정 필드가 아닌 폼 전체의 에러
  API 에러처럼 스키마 밖의 에러를 표시할 때
  errors.root.message로 접근
```

---
# react-hook-form + Zod — 내부 흐름 ⭐️⭐️⭐️⭐️

```txt
① useForm({ resolver: zodResolver(loginSchema) })
   RHF가 제출 직전에 loginSchema로 검증을 위임

② {...register('email')}
   input 값을 RHF 상태에 연결
   name이 스키마 키(email)와 같아야 함

③ handleSubmit(onSubmit) 실행 시:
   검증 실패 → onSubmit 호출 안 함, errors.* 에 Zod 메시지 채움
   검증 성공 → onSubmit({ email, password }) 호출
              → 이 values는 Zod를 통과한 값 (빈 이메일로 login() 안 감)

④ errors.email?.message
   Zod가 준 에러 메시지 → UI에 표시

⑤ isSubmitting
   onSubmit의 async 함수가 끝날 때까지 true → 버튼 disabled 처리
```
---
# Zod vs setError — 핵심 구분 ⭐️⭐️⭐️⭐️

```txt
Zod / zodResolver = 클라이언트 게이트 (서버에 보내기 전)
  "이메일 형식이 맞는가", "비밀번호가 비어있지 않은가"
  → 실패 시 서버에 요청조차 안 함

catch + setError = 서버가 거절했을 때 UI에 붙이기
  "이메일 또는 비밀번호가 일치하지 않습니다" (401)
  CORS·네트워크 오류 메시지
  → 스키마가 모르는 실패 → 수동으로 RHF 에러 칸에 넣음
```

```typescript
const onSubmit = async (data: LoginValues) => {
  // ↑ 여기까지 왔으면 Zod 검증 이미 통과
  //   data.email, data.password 는 보장된 값

  try {
    const result = await login(data.email, data.password);
    // ← Nest까지 요청이 감
    setUser(result.user);
    router.push('/');

  } catch (err) {
    // ← Nest가 거절한 경우 (401, 409 등) 또는 네트워크 오류
    const message = err instanceof Error ? err.message : '오류가 발생했습니다.';

    // 특정 필드에 표시 (Nest 에러 메시지로 분기)
    if (message.includes('이메일')) {
      setError('email', { message });
      //        ↑ errors.email 에 표시 (필드 바로 아래)
    } else {
      setError('root', { message });
      //        ↑ errors.root 에 표시 (폼 전체 알림)
    }
  }
};
```

txt

```txt
setError('email', { message })  → errors.email.message (필드 아래)
setError('root',  { message })  → errors.root.message  (폼 공통 알림)

message.includes('...') 분기:
  Nest가 string 메시지를 그대로 돌려줄 때 쓰는 얇은 매핑
  Nest가 에러 코드(JSON)를 주면 코드로 분기하는 게 더 낫지만
  지금처럼 message 문자열이면 이 정도로 충분

한 줄 요약:
  Zod    = 클라이언트 게이트 (형식·필수 검증)
  setError in catch = 서버가 거절했을 때 UI에 에러 붙이기
```


----
# API 에러 vs 스키마 에러 ⭐️⭐️⭐️⭐️

```txt
1단계 — Zod (클라이언트):
  빈 이메일 → errors.email = '이메일을 입력해주세요.'
  형식 틀림 → errors.email = '올바른 형식의 이메일을 입력해주세요.'
  → handleSubmit이 onSubmit을 호출하지 않음
  → 서버에 요청 안 감

2단계 — catch setError (서버):
  Zod를 통과해서 Nest까지 갔지만 거절된 경우
  401 → '이메일 또는 비밀번호가 일치하지 않습니다'
  409 → '이미 사용 중인 이메일입니다'
  → setError로 수동으로 errors.*에 넣음
  → 화면에 표시
```

```tsx
// UI에 표시하는 법
{errors.email && <p className="text-red-500">{errors.email.message}</p>}
{/* ↑ Zod 에러도, setError('email') 에러도 여기로 다 나옴 */}

{errors.root  && <p className="text-red-500">{errors.root.message}</p>}
{/* ↑ setError('root') — 폼 공통 에러 */}
```

---

# 자주 쓰는 패턴

```typescript
// 비밀번호 확인 (.refine으로 교차 검증)
const registerSchema = z.object({
  password:        z.string().min(8),
  passwordConfirm: z.string(),
}).refine(
  (data) => data.password === data.passwordConfirm,
  { message: '비밀번호가 일치하지 않습니다.', path: ['passwordConfirm'] }
);

// 선택적 필드
const profileSchema = z.object({
  nickname: z.string().min(2),
  bio:      z.string().max(200).optional(),  // 없어도 됨
});

// watch — 특정 필드 실시간 감시
const password = watch('password');

// reset — 성공 후 폼 초기화
reset({ email: '', password: '' });
```

---

# NestJS DTO ↔ Zod 스키마 대응

| NestJS DTO           | Zod 스키마              |
| -------------------- | -------------------- |
| `@IsEmail()`         | `z.string().email()` |
| `@IsString()`        | `z.string()`         |
| `@MinLength(8)`      | `z.string().min(8)`  |
| `@MaxLength(20)`     | `z.string().max(20)` |
| `@IsOptional()`      | `.optional()`        |
| `@IsEnum(['a','b'])` | `z.enum(['a','b'])`  |