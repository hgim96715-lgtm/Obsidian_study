---
aliases: [유틸리티 타입, Awaited, Exclude, Extract, NonNullable, Omit, Partial, Pick, Readonly, Record, Required, ReturnType]
tags: [TypeScript]
related:
  - "[[00_JS_Ecosystem_HomePage]]"
  - "[[NestJS_DTO]]"
  - "[[OpenAPI_Codegen]]"
  - "[[React_FormValidation]]"
  - "[[TS_Generics]]"
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
const [form, setForm]   = useState(originalPost);
const [saved, setSaved] = useState(originalPost);

async function handleSave() {
  const patch = buildPatchBody(saved, form);
  if (Object.keys(patch).length === 0) return; // 변경 없으면 skip
  await updatePost(post.id, patch);
  setSaved(form);
}
```

```txt
NestJS DTO 쪽의 Partial 처리:
  UpdatePostDto extends PartialType(CreatePostDto)
  → NestJS에서 class-validator와 함께 쓰는 방법 → [[NestJS_DTO]]
```

---

# 교차 타입 & 커스텀 유틸리티 패턴 ⭐️⭐️⭐️⭐️

## Partial\<T\> & { 필드 } — 교차 타입으로 특정 필드 강제 ⭐️⭐️⭐️⭐️

```typescript
// ❗ Partial<T>만 쓰면 id도 optional
type UpdateUser = Partial<User>;
// { id?: string; email?: string; nickname?: string }

// ✅ id는 required, 나머지는 optional
type UpdateUserDto = Partial<User> & { id: string };
// { id: string; email?: string; nickname?: string }

// 새 필드 추가
type UpdateUserWithMeta = Partial<User> & {
  id:        string;   // User에 있는 필드를 required로 강제
  updatedBy: string;   // 새 필드 추가
};
```

```typescript
// ⚠️ 동명 필드 타입 충돌 주의
type A = Partial<User> & { nickname: string };
// Partial<User>.nickname?: string  +  & { nickname: string }
// → nickname: string (required로 교체됨)

// 의도가 명확하게: nickname만 required로 올리기
type B = Partial<Omit<User, 'nickname'>> & { nickname: string };
// id?: string; email?: string; nickname: string ← required
```

```txt
Partial<T> & { 필드: 타입 } 읽는 법:
  Partial<User>    → User의 모든 필드를 optional로
  & { id: string } → id는 반드시 string (required)

  두 타입의 교집합(intersection), 같은 키가 있으면 & 오른쪽이 우선됨
  → id?: string (Partial) + id: string (& 쪽) = id: string (required)

PATCH API DTO 패턴:
  UpdateUserDto = Partial<User> & { id: string }
  → 서버에서 id는 반드시 받고, 나머지는 변경된 것만 받음
```

---

## PartialBy\<T, K\> — 특정 필드만 optional ⭐️⭐️⭐️

```typescript
// K 필드만 optional, 나머지는 required 유지
type PartialBy<T, K extends keyof T> =
  Omit<T, K> & Partial<Pick<T, K>>;

type User = { id: string; email: string; nickname: string; role: string };

type CreateUserDto = PartialBy<User, 'id' | 'role'>;
// = { email: string; nickname: string; id?: string; role?: string }
//     ↑ required 유지               ↑ optional로
```

```txt
언제 쓰는가:
  POST — 대부분 필드는 필수, id·timestamps 등 일부만 선택
  "특정 필드만 빼고 다 required"가 필요할 때

  vs Omit<T, K>:
    Omit → 해당 필드 자체를 제거 (없어짐)
    PartialBy → 해당 필드는 남기되 optional로

  vs RequiredBy (아래):
    PartialBy  → 특정 K를 optional로, 나머지는 required
    RequiredBy → 특정 K를 required로, 나머지는 optional
```

---

## RequiredBy\<T, K\> — 특정 필드만 required ⭐️⭐️⭐️

```typescript
// K 필드만 required, 나머지는 optional 유지
type RequiredBy<T, K extends keyof T> =
  Partial<T> & Required<Pick<T, K>>;

type UpdateUserDto = RequiredBy<User, 'id'>;
// = { id: string; email?: string; nickname?: string }
//     ↑ required          ↑ optional
```

```txt
Partial<T> & { id: string } 와 같은 효과
차이: RequiredBy는 제네릭으로 재사용 가능
```

---

## Override\<T, U\> — 필드 타입 교체 ⭐️⭐️⭐️⭐️

```typescript
// 기존 필드의 타입을 완전히 덮어씀
type Override<T, U extends Partial<Record<keyof T, unknown>>> =
  Omit<T, keyof U> & U;

type User = { id: string; email: string; role: string; createdAt: Date; updatedAt: Date };

// 활용 1: Date → string (Prisma → API 응답 타입)
type UserResponse = Override<User, {
  createdAt: string;
  updatedAt: string;
}>;
// = { id: string; email: string; role: string; createdAt: string; updatedAt: string }

// 활용 2: string → 구체적 union으로 타입 강화
type StrictUser = Override<User, {
  role: 'admin' | 'editor' | 'viewer';
}>;

// 활용 3: nullable 제거
type RequiredUser = Override<User, {
  avatarUrl: string;  // string | null → string
}>;
```

```typescript
// 여러 모델에 재사용
type PostResponse    = Override<Post,    { createdAt: string; publishedAt: string | null }>;
type CommentResponse = Override<Comment, { createdAt: string; deletedAt:  string | null }>;
```

```txt
수동 Omit + & 방식과 비교:
  수동:     Omit<User, 'createdAt' | 'updatedAt'> & { createdAt: string; updatedAt: string }
  Override: Override<User, { createdAt: string; updatedAt: string }>
  → 의미가 명확하고 재사용 가능
```

---

## DeepPartial\<T\> — 중첩 객체까지 optional

```typescript
type DeepPartial<T> = T extends object
  ? { [P in keyof T]?: DeepPartial<T[P]> }
  : T;

type Config = {
  db:    { host: string; port: number };
  cache: { ttl: number; maxSize: number };
};

type PartialConfig = DeepPartial<Config>;
// = {
//     db?:    { host?: string; port?: number };
//     cache?: { ttl?: number; maxSize?: number };
//   }
```

```txt
Partial<T> vs DeepPartial<T>:
  Partial → 최상위 필드만 optional (중첩 객체 내부는 required 유지)
  DeepPartial → 재귀적으로 모든 중첩 필드까지 optional

  Partial<Config>.db → { host: string; port: number } (내부는 required)
  DeepPartial<Config>.db → { host?: string; port?: number } (내부도 optional)
```

---

## SerializeDate\<T\> — Prisma Date → string ⭐️⭐️⭐️⭐️

```typescript
// Prisma DateTime 필드를 재귀적으로 string으로 변환
type SerializeDate<T> = {
  [K in keyof T]: T[K] extends Date
    ? string
    : T[K] extends Date | null
    ? string | null
    : T[K] extends object
    ? SerializeDate<T[K]>
    : T[K];
};

import { User, Post } from '@prisma/client';

type UserResponse = SerializeDate<User>;
// createdAt: Date → string ✅
// updatedAt: Date → string ✅
// 나머지 필드 타입 유지 ✅

// 중첩 관계도 처리
type UserWithPostsResponse = SerializeDate<User & { posts: Post[] }>;
```

```txt
왜 필요한가:
  Prisma는 DateTime 필드를 JS Date 객체로 반환
  HTTP 응답 시 JSON.stringify() 자동 호출 → Date.toJSON() → ISO 8601 string
  런타임은 string인데 타입은 Date → 타입-런타임 불일치

  → SerializeDate<T>로 API 응답 타입을 명확히 선언
  → 상세 원인 및 해결 패턴 → [[NestJS_Prisma]] "Date 타입 직렬화" 섹션
```

---

# Required\<T\> — 전부 필수

```typescript
type Config = { timeout?: number; retries?: number; baseUrl?: string; };

type RequiredConfig = Required<Config>;
// = { timeout: number; retries: number; baseUrl: string; }

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

const p: Readonly<Point> = { x: 1, y: 2 };
p.x = 10;  // ❌ 수정 불가

function processConfig(config: Readonly<Config>) {
  // config를 수정하면 안 된다는 것을 타입으로 표현
}
```

---

# Pick\<T, K\> — 일부 필드만 선택 ⭐️⭐️⭐️⭐️

```typescript
type User = {
  id:        string;
  email:     string;
  nickname:  string;
  password:  string;  // 민감 정보
  createdAt: string;
};

type PublicUser    = Pick<User, 'id' | 'nickname' | 'createdAt'>;
type UserListItem  = Pick<User, 'id' | 'nickname'>;
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
type UserWithoutPassword = Omit<User, 'password'>;
type UserProfile         = Omit<User, 'password' | 'email'>;
```

```typescript
// 실전 — 자동 생성 타입에서 일부 필드 타입 교체
type ApiComment = Omit<
  Schemas['CommentResponseDto'],
  'author' | 'parentId' | 'deletedAt'
> & {
  parentId:  string | null;
  deletedAt: string | null;
  author:    ApiAuthor;
};
```

---

# Record\<K, V\> — 키-값 객체 타입 ⭐️⭐️⭐️⭐️

```typescript
// 유니온 타입을 키로 — 모든 케이스를 강제
type Status = 'active' | 'inactive' | 'deleted';
type StatusLabel = Record<Status, string>;
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

## Partial\<Record\<K, V\>\> — 동적 키 추가·제거 ⭐️⭐️⭐️⭐️

```typescript
// Record<number, Movie> — 모든 키가 반드시 존재해야 함
type Posters = Record<number, Movie>;
const p: Posters = {};
delete p[1];  // ❌ TS 에러: delete 대상은 optional이어야 함

// Partial<Record<number, Movie>> — 키가 있을 수도 없을 수도 있음
type Posters = Partial<Record<number, Movie>>;
// = { [key: number]?: Movie }

const p: Posters = {};
p[1] = movie;    // ✅ 추가
delete p[1];     // ✅ 제거 (optional이라 허용)
p[1]?.title;     // ✅ undefined 가능성 있으므로 ?. 사용
```

```txt
언제 Partial<Record> vs Record:
  키가 고정·전부 존재 보장  → Record<K, V>          (누락 시 TS 에러로 실수 방지)
  키가 동적·일부만 존재     → Partial<Record<K, V>> (React state 슬롯 맵 등)

React state 슬롯 패턴:
  선택된 포스터 { [wallSlot]: Movie } — 슬롯이 비어있으면 키가 없어야 함
  → Partial<Record<number, Movie>> 적합
  → delete next[wallSlot] 가능 → [[JS_Operators]] delete 섹션
```


---

# Exclude\<T, U\> · Extract\<T, U\> — 유니온 필터 ⭐️⭐️⭐️

```typescript
type Status = 'active' | 'inactive' | 'deleted' | null | undefined;

type NonNullStatus = Exclude<Status, null | undefined>;
// = 'active' | 'inactive' | 'deleted'

type StringStatus = Extract<Status, string>;
// = 'active' | 'inactive' | 'deleted'  (string인 것만)
```

---

# NonNullable\<T\> — null · undefined 제거 ⭐️⭐️⭐️

```typescript
type MaybeString = string | null | undefined;
type DefinitelyString = NonNullable<MaybeString>;
// = string

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
// 함수가 바뀌면 ReturnType도 자동으로 바뀜
```

```typescript
// 실전 — 라이브러리 함수의 반환 타입
type PostData = Awaited<ReturnType<typeof fetchPost>>;
```

---

# Awaited\<T\> — Promise 안의 타입 ⭐️⭐️⭐️⭐️

```typescript
type StringPromise = Promise<string>;
type Resolved = Awaited<StringPromise>;
// = string  (Promise가 벗겨짐)

// 중첩 Promise도 처리
type Resolved2 = Awaited<Promise<Promise<number>>>;
// = number

// 실전 — async 함수의 반환 타입
type Post = Awaited<ReturnType<typeof fetchPost>>;
// = Post 모델 타입 (Promise 제거 + ReturnType 추출)
```

## Awaited + ReturnType + 인덱스드 액세스 조합 ⭐️⭐️⭐️⭐️

```typescript
// TmdbService의 getMovie 메서드가 반환하는 타입을 꺼내는 패턴
movie: Awaited<ReturnType<TmdbService['getMovie']>> | null;
//                        ↑ 인덱스드 액세스
//             ↑ 메서드 반환 타입 추출
//     ↑ Promise 벗기기
```

```txt
분해:

TmdbService['getMovie']:
  클래스·객체에서 특정 메서드의 타입을 꺼내는 인덱스드 액세스
  typeof TmdbService.prototype.getMovie 와 같은 의미

ReturnType<TmdbService['getMovie']>:
  getMovie 함수 타입에서 반환 타입을 추출
  getMovie()가 Promise<TmdbMovie>를 반환하면 → Promise<TmdbMovie>

Awaited<ReturnType<TmdbService['getMovie']>>:
  Promise를 벗겨서 안의 타입만 꺼냄 → TmdbMovie

전체:
  "TmdbService.getMovie()가 resolve한 값 또는 null"
  → getMovie의 반환 타입이 바뀌면 자동으로 따라옴
```

```typescript
// typeof vs 인덱스드 액세스
type A = Awaited<ReturnType<typeof tmdbService.getMovie>>;
//               ↑ 인스턴스가 있을 때

type B = Awaited<ReturnType<TmdbService['getMovie']>>;
//               ↑ 클래스 타입만 있을 때 (인스턴스 없이)

// 실전 — Service 메서드 반환 타입을 state·prop에 재사용
type MovieState = {
  movie: Awaited<ReturnType<TmdbService['getMovie']>> | null;
  isLoading: boolean;
};
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

// ⑤ PartialBy — 특정 필드만 optional
type CreatePostDto = PartialBy<Post, 'id' | 'createdAt'>;

// ⑥ Override — 필드 타입 통째로 교체
type ApiResponse = Override<PrismaModel, { createdAt: string; updatedAt: string }>;
```

---

# 한눈에 보기

| 유틸리티 타입                 | 하는 일                             |
| ----------------------- | -------------------------------- |
| `Partial<T>`            | 모든 필드 optional (? 추가)            |
| `Required<T>`           | 모든 필드 필수 (? 제거)                  |
| `Readonly<T>`           | 모든 필드 읽기 전용                      |
| `Pick<T, K>`            | K에 해당하는 필드만 선택                   |
| `Omit<T, K>`            | K에 해당하는 필드 제거                    |
| `Record<K, V>`          | K를 키, V를 값으로 하는 객체 타입            |
| `Exclude<T, U>`         | 유니온에서 U에 해당하는 것 제거               |
| `Extract<T, U>`         | 유니온에서 U에 해당하는 것만 추출              |
| `NonNullable<T>`        | null · undefined 제거              |
| `ReturnType<T>`         | 함수의 반환 타입 추출                     |
| `Awaited<T>`            | Promise 안의 타입 추출                 |
| `Parameters<T>`         | 함수의 파라미터 타입 추출                   |
| **커스텀 유틸리티 타입**         |                                  |
| `Partial<T> & { K: V }` | 특정 필드만 required, 나머지는 optional   |
| `PartialBy<T, K>`       | K 필드만 optional, 나머지는 required 유지 |
| `RequiredBy<T, K>`      | K 필드만 required, 나머지는 optional 유지 |
| `Override<T, U>`        | U에 해당하는 필드 타입 교체 (Omit + &)      |
| `DeepPartial<T>`        | 재귀적으로 모든 중첩 필드까지 optional        |
| `SerializeDate<T>`      | Date 타입 필드를 재귀적으로 string으로 변환    |