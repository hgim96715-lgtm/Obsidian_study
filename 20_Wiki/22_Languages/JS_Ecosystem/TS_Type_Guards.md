---
aliases:
  - type guard
  - type narrowing
  - typeof
  - instanceof
  - in
  - is
tags:
  - TypeScript
related:
  - "[[00_JS_Ecosystem_HomePage]]"
  - "[[NestJS_DTO]]"
  - "[[JS_Array_Methods]]"
  - "[[JS_Operators]]"
  - "[[TS_Utility_Types]]"
  - "[[TS_Unknown_Any]]"
  - "[[TS_TypeAssertion]]"
---
# TS_Type_Guards — 타입 좁히기

> [!info] 
> 타입 가드 = "이 값이 어떤 타입인지 런타임에 확인해서 TS에게 알려주는 것." 
> `unknown` / 유니온 타입처럼 "여러 가능성" 안에서 실제 타입을 특정하는 방법들이다.

---

# 왜 필요한가 ⭐️⭐️⭐️

```typescript
function process(value: string | number) {
  value.toUpperCase(); // ❌ number에는 없음 → TS 에러
}

function process(value: string | number) {
  if (typeof value === 'string') {
    value.toUpperCase(); // ✅ 여기서는 string 확정
  } else {
    value.toFixed(2);   // ✅ 여기서는 number 확정
  }
}
```

```txt
좁히기(Narrowing):
  타입의 범위를 점점 좁혀나가는 것
  if 블록 안에서 TS는 조건을 분석해서 타입이 무엇으로 확정됐는지 추적함
  좁힌 이후에는 그 타입의 메서드/속성을 자동완성과 함께 안전하게 쓸 수 있음
```

---

# typeof — 원시 타입 좁히기 ⭐️⭐️⭐️

```typescript
typeof x === 'string'     // string
typeof x === 'number'     // number
typeof x === 'boolean'    // boolean
typeof x === 'bigint'     // bigint
typeof x === 'symbol'     // symbol
typeof x === 'undefined'  // undefined
typeof x === 'function'   // function
typeof x === 'object'     // object | null ⚠️
```

```txt
⚠️ typeof null === 'object' — 역사적 버그, null이 object로 잘못 분류됨
  null 체크는 별도로 해야 함

  if (typeof x === 'object' && x !== null) { ... }
```

```typescript
function format(value: string | number | boolean): string {
  if (typeof value === 'string')  return value;
  if (typeof value === 'number')  return value.toLocaleString();
  if (typeof value === 'boolean') return value ? '예' : '아니오';
  const _: never = value; // 모든 케이스 처리 확인
  return _;
}
```

---

# instanceof — 클래스 인스턴스 좁히기 ⭐️⭐️⭐️⭐️

```typescript
class NetworkError extends Error {
  constructor(public statusCode: number, message: string) {
    super(message);
    this.name = 'NetworkError';
  }
}

function handleError(err: unknown) {
  if (err instanceof NetworkError) {
    console.log(err.statusCode); // NetworkError 확정 → statusCode 접근 가능
    return;
  }
  if (err instanceof Error) {
    console.log(err.message);    // Error 확정
    return;
  }
  console.log(String(err));      // 나머지: 문자열로 변환
}
```

```txt
instanceof 체크 순서:
  서브클래스(NetworkError)를 먼저, 부모 클래스(Error)를 나중에 체크
  순서가 반대면 NetworkError도 Error이므로 부모 블록에 걸려버림

catch 블록에서 가장 자주 쓰이는 패턴:
  catch(err) → err: unknown (TS 4.0+)
  → instanceof Error로 좁혀야 err.message에 접근 가능
```

## catch 블록 — 가장 흔한 실전 패턴 ⭐️⭐️⭐️⭐️

```typescript
try {
  await fetchData();
} catch (err) {
  // 패턴 1 — 단순 메시지 추출
  const message = err instanceof Error ? err.message : String(err);
  setError(message);

  // 패턴 2 — Prisma 에러 분기
  if (err instanceof Prisma.PrismaClientKnownRequestError) {
    if (err.code === 'P2002') throw new ConflictException('중복입니다.');
  }
  if (err instanceof HttpException) throw err; // NestJS HTTP 예외는 그대로 올려보냄
  throw new InternalServerErrorException('서버 오류');
}
```

---

# in — 객체 속성 존재 여부로 좁히기 ⭐️⭐️⭐️

```typescript
type Cat = { meow: () => void };
type Dog = { bark: () => void };

function makeSound(animal: Cat | Dog) {
  if ('meow' in animal) {
    animal.meow(); // Cat 확정
  } else {
    animal.bark(); // Dog 확정
  }
}
```

```txt
'속성명' in 객체 — 해당 속성이 객체에 있는지 런타임에 확인
  → 타입 간에 고유한 속성으로 구분할 때 유용
  → 인터페이스/타입에만 있고 클래스가 아닌 경우 (instanceof 불가)

unknown 좁히기에서 in 쓸 때 주의:
  typeof x === 'object' && x !== null 체크 먼저 필요
  (null이나 원시값에 in 연산자를 쓰면 런타임 에러)

  if (typeof err === 'object' && err !== null && 'message' in err) {
    console.log((err as { message: unknown }).message);
  }
```

---

# 타입 서술어 (Type Predicate) — `value is Type` ⭐️⭐️⭐️⭐️

```typescript
// 반환 타입을 "value is Type" 형태로 쓰면
// 이 함수가 true를 반환한 블록에서 TS가 자동으로 타입을 좁혀줌
function isError(value: unknown): value is Error {
  return value instanceof Error;
}

function isString(value: unknown): value is string {
  return typeof value === 'string';
}

// 사용
const err: unknown = getError();
if (isError(err)) {
  err.message; // ✅ 여기서는 Error로 확정됨
}
```

```txt
Type Predicate가 필요한 경우:
  같은 좁히기 로직을 여러 곳에서 재사용하고 싶을 때
  복잡한 조건(여러 필드 동시 체크)을 함수로 추출하고 싶을 때
  Array.filter에서 타입을 좁히고 싶을 때 (아래 참고)
```

## Array.filter + 타입 서술어 ⭐️⭐️⭐️

```typescript
const items: (string | null)[] = ['a', null, 'b', null, 'c'];

// ❌ filter(Boolean)은 null을 걸러내지만 타입은 여전히 (string | null)[]
const filtered = items.filter(Boolean);

// ✅ 타입 서술어로 null을 걸러내면서 타입도 string[]로 확정
function isNotNull<T>(value: T | null | undefined): value is T {
  return value !== null && value !== undefined;
}
const filtered = items.filter(isNotNull); // string[]
```

---

# 판별 유니온 (Discriminated Union) ⭐️⭐️⭐️⭐️

```typescript
// 공통 tag 필드로 타입을 구분
type SuccessResponse = { status: 'success'; data: User };
type ErrorResponse   = { status: 'error';   message: string };
type Response = SuccessResponse | ErrorResponse;

function handleResponse(res: Response) {
  switch (res.status) {
    case 'success':
      res.data.name; // SuccessResponse 확정
      break;
    case 'error':
      res.message;   // ErrorResponse 확정
      break;
  }
}
```

```txt
판별 유니온의 조건:
  ① 공통 필드(tag)가 있어야 함 (status, type, kind 등)
  ② 그 필드의 값이 각 타입마다 리터럴 타입으로 달라야 함
  → TS가 switch/if로 tag를 확인하면 나머지 필드를 자동으로 좁혀줌

API 응답, Redux 액션, 이벤트 타입 등에서 자주 쓰임
```

## 판별 유니온 + never 소진 체크

```typescript
function handleResponse(res: Response) {
  switch (res.status) {
    case 'success': return res.data;
    case 'error':   throw new Error(res.message);
    default:
      const _: never = res; // 모든 케이스 처리 → 여기 도달 불가
      throw new Error('처리되지 않은 응답');
  }
}
```

---

# Assertion Function — 에러를 throw해서 좁히기 ⭐️⭐️

```typescript
// assert 함수: 조건이 false면 throw, true면 통과 → 이후 타입이 좁혀짐
function assertString(value: unknown): asserts value is string {
  if (typeof value !== 'string') {
    throw new TypeError(`Expected string, got ${typeof value}`);
  }
}

function process(value: unknown) {
  assertString(value);
  value.toUpperCase(); // ✅ assertString 통과 후에는 string 확정
}
```

```txt
asserts value is Type:
  이 함수가 정상적으로 반환(throw 없이)되면
  호출한 이후부터 value가 Type이라고 TS가 보장

용도:
  유효성 검사 함수를 별도로 분리하면서 타입 좁히기도 같이 하고 싶을 때
  테스트 코드의 assert 유틸
```

---

# 좁히기가 안 되는 경우 — as 단언과의 차이 ⭐️⭐️⭐️

```typescript
// ❌ as는 실제로 검사하지 않음 — "내가 보장하겠다"는 약속만
const user = data as User; // data가 실제로 User 모양이 아니어도 TS는 통과

// ✅ 타입 가드는 런타임에 실제로 확인
if (isUser(data)) {
  // 여기서 data가 진짜 User 모양임을 런타임에 확인함
}
```

```txt
타입 가드 vs as 단언:
  타입 가드  런타임에 실제로 확인 → 안전
  as         컴파일 타임만 속임 → 런타임 에러 가능

  as는 "내가 확신한다"고 책임지는 것
  → 확신이 없으면 타입 가드로 실제로 확인해야 함

  → [[TS_TypeAssertion]] 참고
```

---

# 한눈에

|방법|문법|주요 용도|
|---|---|---|
|`typeof`|`typeof x === 'string'`|string/number/boolean 등 원시 타입|
|`instanceof`|`x instanceof Error`|클래스 인스턴스, catch 블록|
|`in`|`'prop' in obj`|속성 존재 여부로 구분 (인터페이스 타입)|
|타입 서술어|`fn(x): x is Type`|재사용 가능한 좁히기 함수, Array.filter|
|판별 유니온|`switch(x.status)`|status/type 공통 필드로 구분|
|Assertion|`asserts x is Type`|조건 불만족 시 throw하는 검사 함수|

```txt
가장 자주 쓰는 순서:
  catch 블록     → instanceof Error
  원시 타입 구분  → typeof
  유니온 타입    → 판별 유니온 (status/type 필드)
  null 제거      → isNotNull 타입 서술어 + Array.filter
  복잡한 객체    → in + typeof 조합

타입 가드 없이 좁힐 때 → as (안전 보장 없음) → [[TS_TypeAssertion]]
any/unknown/void/never 개념 → [[TS_Unknown_Any]]
```