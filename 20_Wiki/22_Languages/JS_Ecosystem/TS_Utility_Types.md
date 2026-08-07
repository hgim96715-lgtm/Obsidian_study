---
aliases:
  - Omit
  - Partial
  - Record
  - 유틸리티 타입
  - Exclude
  - Extract
  - NonNullable
  - ReturnType
  - Pick
  - Readonly
  - Required
tags:
  - TypeScript
related:
  - "[[00_JS_Ecosystem_HomePage]]"
  - "[[TS_Generics]]"
  - "[[OpenAPI_Codegen]]"
  - "[[NestJS_DTO]]"
---
# TS_Utility_Types — 유틸리티 타입

>[!info]
>유틸리티 타입 = TypeScript가 기본 제공하는 타입 변환 헬퍼.
> `Partial`·`Omit`·`Pick`·`Record`처럼 기존 타입을 변환해서 새 타입을 만든다. 
> 타입을 처음부터 다시 쓸 필요 없이 기존 타입에서 파생시킨다. 
> 직접 만드는 제네릭 타입 → [[TS_Generics]]

---

# 유틸리티 타입이란 ⭐️⭐️⭐️⭐️

```typescript
type User = {
  id:       string;
  email:    string;
  nickname: string;
  password: string;
};

// 타입을 처음부터 다시 쓰는 대신:
type PartialUser = {
  id?:       string;
  email?:    string;
  nickname?: string;
  password?: string;
};

// 유틸리티 타입으로 파생:
type PartialUser = Partial<User>;  // 전부 optional로 변환
```

```txt
왜 유틸리티 타입을 쓰는가:
  User 타입이 바뀌면 → Partial<User>도 자동으로 바뀜
  수동으로 만들면 → User 바꿀 때 PartialUser도 따로 수정해야 함
  → 원본 타입 하나만 관리하면 됨
```

---

# Partial\<T\> — 전부 optional ⭐️⭐️⭐️⭐️

```typescript
type User = { id: string; email: string; nickname: string; };

type PartialUser = Partial<User>;
// = { id?: string; email?: string; nickname?: string; }
//    모든 필드가 optional

// PATCH 요청에서 자주 사용 — 일부만 보내도 됨
async function updateUser(id: string, dto: Partial<User>) {
  // dto.email만 보내도 OK
  // dto.nickname만 보내도 OK
}
```

```txt
주로 쓰이는 경우:
  PATCH API — 수정할 필드만 선택적으로 보냄
  초기값 설정 — 전부 없어도 되는 설정 객체
  테스트 목(mock) 데이터 — 일부 필드만 채움
```

## PATCH 요청 — 변경된 필드만 보내기 ⭐️⭐️⭐️⭐️

```typescript
// 변경된 필드만 골라서 PATCH body 만들기
function buildPatchBody<T extends object>(
  original: T,
  next:     T,
): Partial<T> {
  const patch: Partial<T> = {};
  for (const key of Object.keys(next) as (keyof T)[]) {
    if (next[key] !== original[key]) {
      patch[key] = next[key];
    }
  }
  return patch;  // 바뀐 필드만 담긴 객체
}

const original = { title: '원래 제목', content: '원래 내용', status: 'draft' };
const next     = { title: '새 제목',   content: '원래 내용', status: 'draft' };

buildPatchBody(original, next)
// { title: '새 제목' }  ← title만 바뀌었으므로
```

```typescript
// 실전 — 폼에서 변경된 것만 서버에 보내기
const [form, setForm]       = useState(originalPost);
const [saved, setSaved]     = useState(originalPost);

async function handleSave() {
  const patch = buildPatchBody(saved, form);   // 변경분만 추출
  if (Object.keys(patch).length === 0) return; // 변경 없으면 skip

  await updatePost(post.id, patch);            // PATCH /posts/:id
  setSaved(form);
}
```

```txt
NestJS DTO 쪽의 Partial 처리:
  UpdatePostDto extends PartialType(CreatePostDto)
  → NestJS에서 class-validator와 함께 쓰는 방법 → [[NestJS_DTO]]
```

---

# Required\<T\> — 전부 필수

```typescript
type Config = { timeout?: number; retries?: number; baseUrl?: string; };

type RequiredConfig = Required<Config>;
// = { timeout: number; retries: number; baseUrl: string; }
//    ? 가 전부 제거됨

// Partial 반대 — 기본값 처리 후 타입을 보장할 때
function initConfig(config: Config): RequiredConfig {
  return {
    timeout:  config.timeout  ?? 3000,
    retries:  config.retries  ?? 3,
    baseUrl:  config.baseUrl  ?? 'http://localhost:3000',
  };
}
```

---

# Readonly\<T\> — 읽기 전용

```typescript
type Point = { x: number; y: number; };

type ReadonlyPoint = Readonly<Point>;
// = { readonly x: number; readonly y: number; }

const p: ReadonlyPoint = { x: 1, y: 2 };
p.x = 10;  // ❌ 수정 불가

// 실전 — as const와 비슷한 효과
function processConfig(config: Readonly<Config>) {
  // config를 수정하면 안 된다는 것을 타입으로 표현
}
```

---

# Pick\<T, K\> — 일부 필드만 선택 ⭐️⭐️⭐️⭐️

```typescript
type User = {
  id:       string;
  email:    string;
  nickname: string;
  password: string;  // 민감 정보
  createdAt: string;
};

// 비밀번호 제외하고 공개 정보만
type PublicUser = Pick<User, 'id' | 'nickname' | 'createdAt'>;
// = { id: string; nickname: string; createdAt: string; }

// 목록 조회용 — 필요한 필드만
type UserListItem = Pick<User, 'id' | 'nickname'>;
```

```txt
Pick vs Omit:
  Pick<T, '가져올 키'>  → 원하는 것만 선택 (화이트리스트)
  Omit<T, '제거할 키'>  → 특정 것만 제거 (블랙리스트)

  제거할 것이 적으면 Omit이 편함 (1~2개 제거)
  가져올 것이 적으면 Pick이 편함 (1~2개 선택)
```

---

# Omit\<T, K\> — 일부 필드 제거 ⭐️⭐️⭐️⭐️

```typescript
type User = {
  id:       string;
  email:    string;
  nickname: string;
  password: string;
  createdAt: string;
};

// 비밀번호만 제거
type UserWithoutPassword = Omit<User, 'password'>;
// = { id: string; email: string; nickname: string; createdAt: string; }

// 여러 필드 제거
type UserProfile = Omit<User, 'password' | 'email'>;
```

```typescript
// 실전 — 자동 생성 타입에서 일부 필드 타입 교체
// api.d.ts에서 생성된 타입의 일부를 원하는 타입으로 교체할 때
type ApiComment = Omit<
  Schemas['CommentResponseDto'],
  'author' | 'parentId' | 'deletedAt'
> & {
  parentId:  string | null;  // 생성 타입이 안 맞을 때 교체
  deletedAt: string | null;
  author:    ApiAuthor;
};
```

---

# Record\<K, V\> — 키-값 객체 타입 ⭐️⭐️⭐️⭐️

```typescript
// Record<키 타입, 값 타입>
type ScoreBoard = Record<string, number>;
// = { [key: string]: number }
// → 어떤 문자열 키든 number 값

const scores: ScoreBoard = {
  '홍길동': 100,
  '김철수': 95,
};

// 유니온 타입을 키로 — 모든 케이스를 강제
type Status = 'active' | 'inactive' | 'deleted';
type StatusLabel = Record<Status, string>;
// = { active: string; inactive: string; deleted: string; }
// → 셋 중 하나라도 빠지면 에러

const labels: StatusLabel = {
  active:   '활성',
  inactive: '비활성',
  deleted:  '삭제됨',  // 이게 없으면 TypeScript 에러
};
```

```typescript
// 실전 — SMTP 제공업체별 설정
type MailProvider = 'gmail' | 'icloud';

const PROVIDER_DEFAULTS: Record<MailProvider, { host: string; port: number }> = {
  gmail:  { host: 'smtp.gmail.com',   port: 587 },
  icloud: { host: 'smtp.mail.me.com', port: 587 },
};
// MailProvider에 새 값이 추가되면 여기도 추가해야 함 → 실수 방지
```

---

# Exclude\<T, U\> · Extract\<T, U\> — 유니온 필터 ⭐️⭐️⭐️

```typescript
type Status = 'active' | 'inactive' | 'deleted' | null | undefined;

// Exclude — 제거
type NonNullStatus = Exclude<Status, null | undefined>;
// = 'active' | 'inactive' | 'deleted'

// Extract — 추출
type StringStatus = Extract<Status, string>;
// = 'active' | 'inactive' | 'deleted'  (string인 것만)

// 실전
type StringOrNumber = string | number | boolean;
type OnlyString = Extract<StringOrNumber, string>;  // string
type WithoutBoolean = Exclude<StringOrNumber, boolean>;  // string | number
```

---

# NonNullable\<T\> — null · undefined 제거 ⭐️⭐️⭐️

```typescript
type MaybeString = string | null | undefined;

type DefinitelyString = NonNullable<MaybeString>;
// = string

// 실전 — early return 후 타입 보장
function process(value: string | null) {
  if (!value) return;
  // 여기서 value: string (TypeScript가 narrowing)
  // NonNullable<typeof value> = string
}

// 제네릭 함수에서
function compact<T>(arr: (T | null | undefined)[]): T[] {
  return arr.filter((item): item is NonNullable<T> => item != null);
}
```

---

# ReturnType\<T\> — 함수 반환 타입 추출 ⭐️⭐️⭐️⭐️

```typescript
function fetchUser() {
  return { id: '1', nickname: '홍길동', email: 'hong@example.com' };
}

type UserData = ReturnType<typeof fetchUser>;
// = { id: string; nickname: string; email: string; }

// 함수 타입을 별도로 정의하지 않아도 됨
// 함수가 바뀌면 ReturnType도 자동으로 바뀜
```

```typescript
// 실전 — 라이브러리 함수의 반환 타입
type QueryResult = ReturnType<typeof prisma.post.findMany>;
// Awaited<ReturnType<...>>와 조합 (비동기)
type PostData = Awaited<ReturnType<typeof fetchPost>>;
```

---

# Awaited\<T\> — Promise 안의 타입 ⭐️⭐️⭐️⭐️

```typescript
type StringPromise = Promise<string>;
type Resolved = Awaited<StringPromise>;
// = string  (Promise가 벗겨짐)

// 중첩 Promise도 처리
type NestedPromise = Promise<Promise<number>>;
type Resolved2 = Awaited<NestedPromise>;
// = number

// 실전 — async 함수의 반환 타입
async function fetchPost(id: string) {
  return await prisma.post.findUnique({ where: { id } });
}

type Post = Awaited<ReturnType<typeof fetchPost>>;
// = Post 모델 타입 (Promise 제거 + ReturnType 추출)
```

---

# Parameters\<T\> — 함수 파라미터 타입 추출 ⭐️⭐️

```typescript
function login(email: string, password: string, rememberMe?: boolean) { ... }

type LoginParams = Parameters<typeof login>;
// = [email: string, password: string, rememberMe?: boolean]

type FirstParam = Parameters<typeof login>[0];
// = string (첫 번째 파라미터)
```

---

# 자주 쓰는 조합 패턴

```typescript
// ① Omit + Partial — 일부 제거 후 나머지를 optional
type UpdatePostDto = Partial<Omit<Post, 'id' | 'createdAt'>>;
// id와 createdAt을 제거한 후, 나머지를 전부 optional로

// ② ReturnType + Awaited — async 함수 반환 타입
type Result = Awaited<ReturnType<typeof asyncFunction>>;

// ③ Record + 유니온 — 완전한 케이스 보장
type StatusConfig = Record<'active' | 'inactive' | 'deleted', {
  label: string;
  color: string;
}>;

// ④ Omit + 확장 — 일부 필드 타입 교체
type CustomType = Omit<GeneratedType, '교체할필드'> & {
  교체할필드: 원하는타입;
};
```

---

# 한눈에 보기

| 유틸리티 타입          | 하는 일                  |
| ---------------- | --------------------- |
| `Partial<T>`     | 모든 필드 optional (? 추가) |
| `Required<T>`    | 모든 필드 필수 (? 제거)       |
| `Readonly<T>`    | 모든 필드 읽기 전용           |
| `Pick<T, K>`     | K에 해당하는 필드만 선택        |
| `Omit<T, K>`     | K에 해당하는 필드 제거         |
| `Record<K, V>`   | K를 키, V를 값으로 하는 객체 타입 |
| `Exclude<T, U>`  | 유니온에서 U에 해당하는 것 제거    |
| `Extract<T, U>`  | 유니온에서 U에 해당하는 것만 추출   |
| `NonNullable<T>` | null · undefined 제거   |
| `ReturnType<T>`  | 함수의 반환 타입 추출          |
| `Awaited<T>`     | Promise 안의 타입 추출      |
| `Parameters<T>`  | 함수의 파라미터 타입 추출        |