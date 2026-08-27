---
aliases:
  - Client Component
  - Server Component
  - use client
  - use server
  - async/await 사이에서 narrowing이 풀린다
  - never
tags:
  - TypeScript
related:
  - "[[00_JS_Ecosystem_HomePage]]"
  - "[[TS_Generics]]"
  - "[[TS_TypeAssertion]]"
  - "[[NestJS_Controller]]"
  - "[[JS_Operators]]"
---
# TS_Type_Guards — 타입 좁히기

>[!info]
>TypeScript는 코드 흐름을 분석해서 변수의 타입을 좁힌다(narrow). 
>`if (!x) return` 뒤에서 TypeScript가 자동으로 x를 non-null로 인식하는 것이 제어 흐름 좁히기.
> `typeof`, `instanceof`, `in`, `is` 데코레이터도 같은 원리.
> `!` 단언·`as`·`satisfies` → [[TS_TypeAssertion]]

---

# 제어 흐름 타입 좁히기 — early return 패턴 ⭐️⭐️⭐️⭐️

```txt
TypeScript는 if문, return, throw 등을 보고
"이 줄 이후에는 어떤 타입이 가능한가"를 자동으로 추론함
이걸 제어 흐름 분석(Control Flow Analysis)이라고 함
```

## 핵심 — `if (!x) return` 뒤에서 x는 non-null

```typescript
// focusMessage: Message | null

async function confirmDelete() {
  if (!focusMessage) return;  // null이면 여기서 함수 종료
  //                    ↑ TypeScript: "이 아래에선 focusMessage가 null일 수 없음"

  focusMessage.body   // ✅ non-null로 확정 — 오류 없음
  focusMessage.id     // ✅
}
```

```txt
왜 return 뒤에서 null이 아닌지:
  if (!focusMessage) return
  = "focusMessage가 null이거나 undefined이면 함수를 종료한다"

  이 줄을 통과했다는 것 = focusMessage가 null이 아님이 보장됨
  TypeScript가 이 논리를 자동으로 추론 → 이후 코드에서 타입 좁혀짐
```

## async/await 사이에서 narrowing이 풀린다 ⭐️⭐️⭐️⭐️

```typescript
// accessToken: string | null

async function fetchUser() {
  if (!accessToken) return;
  // 여기서는 string으로 좁혀짐

  // ❌ await 뒤에서는 다시 string | null
  const me = await apiFetch('/auth/me', { token: accessToken });
  //                                              ↑ TS 에러 또는 경고
  setSession(accessToken, me);  // accessToken이 또 null일 수 있다고 봄
}
```

```typescript
// ✅ await 전에 const로 고정
async function fetchUser() {
  if (!accessToken) return;

  const token = accessToken;  // string으로 확정한 값을 변수에 캡처
  //    ↑ const이므로 이후 절대 바뀌지 않음 → TS가 항상 string으로 인식

  const me = await apiFetch('/auth/me', { token });   // ✅
  setSession(token, me);                               // ✅
}
```

```txt
왜 await 뒤에서 narrowing이 풀리는가:
  accessToken은 클로저 변수 (외부에서 바뀔 수 있음)
  await 지점에서 실행이 잠시 멈추고 다른 코드가 실행될 수 있음
  → 그 사이에 accessToken이 null로 바뀔 수도 있다고 TypeScript가 판단
  → narrowing 해제

  const token = accessToken:
  const = 재할당 불가 → 이후 절대 바뀌지 않음
  → TypeScript가 항상 string으로 추론
  → await 몇 번을 해도 token은 string 유지

실전 패턴:
  if (!가변값) return;
  const 고정값 = 가변값;  // ← await 전에 캡처
  await 비동기작업(고정값);
```

## 왜 다른 변수 체크는 도움이 안 되는가 ⭐️⭐️⭐️⭐️

```typescript
// focusRoomId: string | null
// focusMessage: Message | null — focusRoomId와 별개의 변수

// ❌ focusRoomId를 체크해도 focusMessage는 여전히 null일 수 있음
if (!focusRoomId) return;
focusMessage.body  // ❌ TypeScript: focusMessage는 여전히 null | Message

// ✅ focusMessage 자체를 체크해야 함
if (!focusMessage) return;
focusMessage.body  // ✅ non-null 확정
```

```txt
TypeScript는 각 변수를 독립적으로 추적함
focusRoomId와 focusMessage는 서로 다른 변수
focusRoomId가 있다고 해서 focusMessage도 있다는 보장이 없음
→ TypeScript는 이 연관관계를 알 수 없음
→ 사용하는 변수를 직접 체크해야 함

실수가 생기는 이유:
  "roomId가 있으면 message도 있겠지"라는 개발자의 가정이 있지만
  TypeScript는 그 가정을 코드로 증명할 수 없음
  → focusMessage를 직접 체크하는 것만이 TypeScript에게 증명이 됨
```

## 여러 조건 동시 체크 ⭐️⭐️⭐️⭐️

```typescript
async function confirmDeleteMessage() {
  // 여러 조건을 한 번에 체크
  if (!focusMessage || deletingMessage || focusMessage.deletedAt) return;
  //    ↑ null 체크      ↑ 진행 중 체크    ↑ 이미 삭제됨 체크

  // 여기서 TypeScript가 아는 것:
  // focusMessage → Message (null 아님)
  // deletingMessage → false
  // focusMessage.deletedAt → null/undefined/falsy

  setDeletingMessage(true);
  await deleteMessage(focusMessage.roomId, focusMessage.id);  // ✅ 안전
}
```

```txt
|| (or) 조건과 타입 좁히기:
  if (!a || !b || !c) return
  = a, b, c 중 하나라도 falsy면 return
  = 이 줄을 통과하면 a, b, c 모두 truthy(non-null)

  TypeScript는 각 변수를 개별적으로 좁힘:
    !focusMessage 조건 → focusMessage: Message (null 제거)
    !deletingMessage 조건 → deletingMessage: false
```

## 타입 좁히기 vs !  단언

```typescript
// 방법 1 — early return (권장)
if (!focusMessage) return;
focusMessage.body  // ✅ 안전 — null이면 애초에 실행 안 됨

// 방법 2 — !  단언 (위험)
focusMessage!.body  // TypeScript는 OK지만 실제로 null이면 런타임 에러
//           ↑ "나는 null이 아님을 안다"고 개발자가 주장

// 방법 3 — ?. 옵셔널 체이닝 (null이면 조용히 건너뜀)
focusMessage?.body  // null이면 undefined 반환 (에러 없음)
                    // "있으면 접근, 없으면 무시"
```

```txt
세 가지 비교:
  early return   → null이면 함수 자체를 종료 (가장 명확한 의도)
  !  단언         → 개발자가 null 아님을 주장 (틀리면 런타임 에러)
  ?.             → null이어도 조용히 undefined (에러 없이 진행)

언제 뭘:
  null이면 아무것도 안 해야 할 때 → early return 또는 ?.
  null이 절대 불가능하다고 확신 → ! 단언 (확신이 틀리면 위험)
  null이어도 그냥 넘어가도 될 때 → ?.
```

---

# any vs unknown — 타입 가드가 필요한 이유 ⭐️⭐️⭐️⭐️

```typescript
let a: any     = '문자열';
let u: unknown = '문자열';

a.toUpperCase();  // ✅ any — TypeScript가 체크 안 함 (런타임 에러 가능)
u.toUpperCase();  // ❌ unknown — 타입 확인 전 사용 불가
```

```txt
any:
  타입 검사를 완전히 끔
  무엇이든 할 수 있지만 TypeScript의 보호도 없음
  → 런타임에 터질 때까지 에러가 안 보임
  → 쓰면 안 됨 (불가피할 때만 최소한으로)

unknown:
  "어떤 타입인지 모름"을 안전하게 표현
  사용하기 전에 반드시 타입을 확인해야 함
  → 확인하는 행위 = type guard

unknown이 필요한 상황:
  catch (err) — err은 무조건 unknown (어떤 에러가 올지 모름)
  JSON.parse()  — 결과가 무엇인지 모름
  외부 API 응답 — 서버가 뭘 보낼지 모름
```

## any가 필요한 경우 — 최소한으로 ⭐️⭐️

```typescript
// ❌ 전체를 any로
function process(data: any) { ... }

// ✅ 필요한 부분만 any로 우회
const iframe = document.createElement('iframe') as any;
iframe.src = '...';  // TypeScript가 모르는 속성에 접근할 때 최소 범위로

// ✅ unknown으로 받고 내부에서 좁히기
function process(data: unknown) {
  if (typeof data === 'string') data.toUpperCase(); // ✅
}
```

---

# typeof — 원시 타입 확인 ⭐️⭐️⭐️⭐️

```typescript
function format(value: string | number | boolean) {
  if (typeof value === 'string')  return value.toUpperCase();
  if (typeof value === 'number')  return value.toFixed(2);
  if (typeof value === 'boolean') return value ? '예' : '아니오';
}
```

|`typeof` 결과|타입|
|---|---|
|`'string'`|string|
|`'number'`|number|
|`'boolean'`|boolean|
|`'undefined'`|undefined|
|`'function'`|function|
|`'object'`|object (⚠️ null도 'object')|
|`'symbol'`|symbol|
|`'bigint'`|bigint|

```typescript
// null 체크 — typeof 'object' 함정 피하기
if (typeof value === 'object' && value !== null) {
  // 여기서야 value가 진짜 object
}

// undefined 확인
if (typeof value !== 'undefined') { ... }
// 또는 더 간결하게
if (value !== undefined) { ... }
```

---

# instanceof — 클래스 인스턴스 확인 ⭐️⭐️⭐️⭐️

```typescript
// 가장 자주 쓰이는 곳 — 에러 처리
try {
  await someAsyncOperation();
} catch (err) {
  if (err instanceof Error) {
    console.log(err.message);  // ✅ Error 클래스의 message 속성 사용 가능
    console.log(err.stack);    // ✅
  }
}
```

```txt
catch (err)에서 err 타입이 unknown인 이유:
  throw 'string 에러'    — 문자열도 throw 가능
  throw 42               — 숫자도 throw 가능
  throw new Error('...')  — Error 인스턴스도 가능
  → 어떤 타입이든 올 수 있으니 unknown

  err instanceof Error 체크 없이 err.message 쓰면:
  → TypeScript 에러 (unknown에 .message 없음)
  → instanceof로 좁힌 뒤에만 사용 가능
```

```typescript
// 커스텀 에러 클래스 구분
class NetworkError extends Error { statusCode: number; }
class ValidationError extends Error { field: string; }

if (err instanceof NetworkError)   handleNetwork(err.statusCode);
if (err instanceof ValidationError) handleValidation(err.field);
```

## 에러 메시지 추출 유틸 패턴

```typescript
// catch 블록에서 반복되는 패턴
function getErrorMessage(err: unknown): string {
  if (err instanceof Error) return err.message;
  if (typeof err === 'string') return err;
  return '알 수 없는 오류가 발생했습니다.';
}

// 사용
catch (err) {
  setError(getErrorMessage(err));
}
```

---

# in — 속성 존재로 유니온 구분 ⭐️⭐️⭐️

```typescript
type Circle    = { kind: 'circle';    radius: number };
type Rectangle = { kind: 'rectangle'; width: number; height: number };

function getArea(shape: Circle | Rectangle) {
  if ('radius' in shape) {
    return Math.PI * shape.radius ** 2;  // shape: Circle
  }
  return shape.width * shape.height;    // shape: Rectangle
}
```

```txt
in 연산자:
  'propertyName' in object
  → 그 속성이 있으면 true, 없으면 false

유니온 타입에서 각 타입만 가진 속성을 확인해서 분기
```

## discriminated union — kind 필드로 구분 ⭐️⭐️⭐️⭐️

```typescript
type Shape =
  | { kind: 'circle';    radius: number }
  | { kind: 'rectangle'; width: number; height: number }
  | { kind: 'triangle';  base: number; height: number };

function getArea(shape: Shape): number {
  switch (shape.kind) {
    case 'circle':    return Math.PI * shape.radius ** 2;
    case 'rectangle': return shape.width * shape.height;
    case 'triangle':  return (shape.base * shape.height) / 2;
  }
}
```

```txt
discriminated union = 공통 필드(kind)의 리터럴 값으로 타입 구분
각 case 안에서 TypeScript가 해당 타입으로 자동 좁힘
switch문과 조합하면 in보다 더 명확하게 분기 가능
```

---

# 사용자 정의 타입 가드 — is ⭐️⭐️⭐️⭐️

```typescript
// 반환 타입에 'x is Type' 형태로 선언
function isUser(value: unknown): value is User {
  return (
    typeof value === 'object' &&
    value !== null &&
    'id' in value &&
    'name' in value
  );
}

// 사용
if (isUser(data)) {
  console.log(data.name);  // ✅ TypeScript가 data를 User로 확정
}
```

```txt
일반 boolean 반환 함수 vs is 타입 가드:
  function check(x: unknown): boolean { return typeof x === 'string'; }
  if (check(data)) { data.toUpperCase(); }  // ❌ data 타입이 여전히 unknown

  function isString(x: unknown): x is string { return typeof x === 'string'; }
  if (isString(data)) { data.toUpperCase(); }  // ✅ data 타입이 string으로 좁혀짐

언제 쓰는가:
  같은 객체 검사를 여러 곳에서 반복할 때 함수로 추출
  API 응답 검증처럼 복잡한 조건을 한 곳에서 관리
```

```typescript
// filter와 조합 — 타입 안전하게 null 제거
const values = [1, null, 2, undefined, 3];

// ❌ boolean만 반환 — 결과 타입이 여전히 (number | null | undefined)[]
const cleaned = values.filter((v) => v != null);

// ✅ is 타입 가드 — 결과 타입이 number[]
const cleaned = values.filter((v): v is number => v != null);
```

---

# never — 절대 일어나지 않는 타입 ⭐️⭐️⭐️⭐️

```txt
never = "이 코드는 절대 실행되지 않는다"

유니온 타입을 switch/if로 전부 처리하면
마지막 분기에서 TypeScript가 타입을 never로 추론
→ 새 타입이 추가됐는데 분기를 빠뜨리면 컴파일 에러
```

```typescript
type Shape = Circle | Rectangle | Triangle;

function getArea(shape: Shape): number {
  switch (shape.kind) {
    case 'circle':    return Math.PI * shape.radius ** 2;
    case 'rectangle': return shape.width * shape.height;
    case 'triangle':  return (shape.base * shape.height) / 2;
    default:
      const _exhaustiveCheck: never = shape;  // ← 모든 케이스 처리됐으면 never
      throw new Error(`처리되지 않은 shape: ${JSON.stringify(shape)}`);
  }
}
```

```txt
새 타입 추가 시 효과:
  type Shape = Circle | Rectangle | Triangle | Pentagon  ← Pentagon 추가

  switch에서 Pentagon case가 없으면:
  const _exhaustiveCheck: never = shape;
  → shape 타입이 Pentagon (never가 아님) → TS 컴파일 에러
  → "분기 빠뜨렸어"를 런타임 전에 잡아줌

never가 등장하는 다른 상황:
  반환하지 않는 함수 (throw만 하거나 무한루프)
  function fail(msg: string): never { throw new Error(msg); }

  불가능한 타입 조합
  type Impossible = string & number  // → never

  조건부 타입에서 "이 경우는 없다"를 표현
  T extends never ? never : ActualType
  → [[TS_Generics]] 조건부 타입 + never 섹션
```

---

# as const — 리터럴 타입 고정 ⭐️⭐️⭐️

```typescript
// 타입 widening (넓혀짐):
const direction = 'left';        // 타입: string
const directions = ['left', 'right']; // 타입: string[]

// as const — 리터럴 타입 유지:
const direction = 'left' as const;         // 타입: 'left' (리터럴)
const directions = ['left', 'right'] as const; // 타입: readonly ['left', 'right']
```

```typescript
// 실전 — Prisma select 객체
const userSelect = {
  id: true,
  email: true,
  name: true,
} as const;
// as const 없으면: { id: boolean; email: boolean; name: boolean }
// as const 있으면: { readonly id: true; readonly email: true; ... }
// → Prisma가 반환 타입을 정확히 추론 가능
```

```txt
as const가 필요한 이유:
  TypeScript는 기본적으로 타입을 넓게 추론 (widening)
    'left' → string
    true   → boolean
  as const를 붙이면 리터럴 그대로 유지
    'left' → 'left'
    true   → true

Prisma select에서 as const:
  Prisma select의 각 값이 boolean이 아닌 "정확히 true"여야 타입 추론이 작동
  → [[NestJS_Prisma]] select/include 상수로 빼기 섹션 참고
```

---

# 실전 패턴

## JSON.parse — unknown으로 안전하게 처리 ⭐️⭐️⭐️

```typescript
// ❌ any — 검증 없이 바로 사용
const data = JSON.parse(response) as User;  // 실제로 User인지 모름

// ✅ unknown → 검증 → 좁히기
function parseUser(json: string): User | null {
  try {
    const data: unknown = JSON.parse(json);
    if (isUser(data)) return data;  // isUser는 사용자 정의 타입 가드
    return null;
  } catch {
    return null;
  }
}
```

## API 에러 응답 처리 ⭐️⭐️⭐️

```typescript
// fetch 에러 응답에서 메시지 추출
const body = await res.json() as unknown;

const message =
  typeof body === 'object' &&
  body !== null &&
  'message' in body &&
  typeof (body as { message: unknown }).message === 'string'
    ? (body as { message: string }).message
    : `요청 실패: ${res.status}`;
```

---

# void — 반환값 없음 ⭐️⭐️

```typescript
// 함수가 아무것도 반환하지 않을 때
function logMessage(msg: string): void {
  console.log(msg);
}

// 콜백 타입에서 — "반환값을 신경 안 쓴다"
type OnClick = () => void;
const handler: OnClick = () => '뭔가 반환해도 됨';  // ✅ void는 반환값 무시

// undefined와의 차이:
function a(): void      { }  // ✅ 암묵적 undefined 반환
function b(): undefined { }  // ❌ 명시적으로 undefined를 반환해야 함
```

```txt
void vs undefined:
  void    = "반환값을 신경 안 쓴다" — 콜백 타입에 주로 사용
  undefined = "정확히 undefined를 반환한다"

onClick: () => void  →  어떤 값을 반환해도 TypeScript가 무시
onClick: () => undefined → undefined만 반환해야 함 (더 엄격)
```

---
# string? vs string | undefined — 파라미터 위치 규칙 ⭐️⭐️⭐️⭐️

```typescript
// ❌ TS 에러 — optional(?) 뒤에 required가 올 수 없음
function find(
  machineId?: string,  // optional
  page: number,        // required — optional 뒤에 오면 에러
) {}
// Error: A required parameter cannot follow an optional parameter.
```

```typescript
// ✅ 해결 1 — ? 대신 | undefined 로 명시
function find(
  machineId: string | undefined,  // 타입은 undefined 포함, 위치는 "필수"
  page: number,
) {}
find(undefined, 1);  // 첫 인자를 undefined로 명시해서 전달

// ✅ 해결 2 — optional을 뒤로 이동
function find(
  page: number,
  machineId?: string,  // 필수 파라미터 뒤로
) {}
```

```txt
string? 와 string | undefined 의 차이:

  string? (선택적 파라미터):
    TypeScript 파라미터 문법 — 호출 시 인자를 아예 생략 가능
    뒤에 required 파라미터가 오면 TS 에러
    find(1)  → machineId 생략 가능

  string | undefined (유니온 타입):
    타입에 undefined가 포함됐을 뿐, 파라미터 위치는 "필수"
    뒤에 required 파라미터가 와도 에러 없음
    find(undefined, 1)  → 첫 인자를 명시적으로 넘겨야 함

  둘 다 undefined를 허용하지만 호출 방식이 다름:
    machineId?: string        → find(1)          (생략 가능)
    machineId: string | undefined → find(undefined, 1)  (생략 불가)
```

```typescript
// NestJS @Query 에서 자주 만나는 케이스
// ❌
@Get()
findAll(
  @Query('machineId') machineId?: string,
  @Query('page', new DefaultValuePipe(1), ParseIntPipe) page: number,
) {}

// ✅
@Get()
findAll(
  @Query('machineId') machineId: string | undefined,
  @Query('page', new DefaultValuePipe(1), ParseIntPipe) page: number,
) {}
// → [[NestJS_Controller]] @Query optional 파라미터 순서 에러 섹션
```