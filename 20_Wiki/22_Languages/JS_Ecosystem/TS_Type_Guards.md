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
---
# TS_Type_Guards — 타입 가드 & 타입 좁히기

> [!info] 
> 타입 좁히기(narrowing)는 "여러 타입이 될 수 있는 값"을 특정 블록 안에서 더 구체적인 타입으로 확정하는 것이다. TypeScript는 특정 조건문 패턴을 보고 그 블록 안에서 자동으로 타입을 좁혀준다 — 이 조건문 패턴을 타입 가드라고 부른다.

---

# 왜 타입 좁히기가 필요한가 ⭐️⭐️⭐️⭐️

```typescript
function process(input: string | number) {
  input.toUpperCase(); // ❌ — input이 number일 수도 있어서 TS가 에러를 냄
}
```

```txt
input이 string | number 타입이면, TS 입장에서는 "둘 중 뭔지 모름"
string에만 있는 .toUpperCase()를 바로 호출하면 "number에는 이 메서드 없다"고 에러를 냄

→ "지금 이 값이 어떤 타입인지"를 런타임에 확인하고, 그 확인한 블록 안에서만 해당 타입의 메서드를 쓰게 함
  이게 타입 좁히기고, 그 조건식을 타입 가드라고 부름
```

---

# typeof — 원시 타입 판별 ⭐️⭐️⭐️⭐️

```typescript
function process(input: string | number) {
  if (typeof input === 'string') {
    input.toUpperCase(); // 이 블록 안에서 input은 string
  } else {
    input.toFixed(2);    // 이 블록 안에서 input은 number
  }
}
```

```txt
typeof가 반환하는 문자열:
  'string' / 'number' / 'boolean' / 'bigint' / 'symbol' / 'undefined' / 'function' / 'object'

⚠️ 배열과 null은 typeof로 구분 안 됨 — 둘 다 'object'를 반환함
  typeof []    → 'object'
  typeof null  → 'object'
  → 배열은 Array.isArray(), null은 === null 체크로 따로 처리해야 함
```

---

# Array.isArray — 배열 판별 ⭐️⭐️⭐️⭐️

```typescript
function process(input: string | string[]) {
  if (Array.isArray(input)) {
    input.join(', '); // string[]
  } else {
    input.toUpperCase(); // string
  }
}
```

```txt
typeof []는 'object'라서 typeof로 배열을 잡을 수 없음
Array.isArray()가 배열 전용 타입 가드 — 자세한 동작은 [[JS_Array_Methods]] 참고
```

---

# instanceof — 클래스 인스턴스 판별 ⭐️⭐️⭐️

```typescript
function handleError(error: unknown) {
  if (error instanceof Error) {
    console.log(error.message); // Error 클래스의 속성 접근 가능
  }
}

class ApiError extends Error {
  constructor(public statusCode: number, message: string) {
    super(message);
  }
}

function handle(error: Error | ApiError) {
  if (error instanceof ApiError) {
    error.statusCode; // ApiError에만 있는 속성
  }
}
```

```txt
instanceof는 클래스(생성자 함수)에만 씀 — 일반 interface나 type alias는 런타임에 사라지므로 instanceof 불가
  (interface는 TS 컴파일 후 JS에 남지 않음 → 런타임에 "이게 X 인터페이스인가" 체크 자체가 불가능)
  → interface/type을 판별하려면 아래 in 연산자나 사용자 정의 타입 가드를 써야 함
```

---

# null / undefined 체크 ⭐️⭐️⭐️⭐️

```typescript
function greet(name: string | null | undefined) {
  if (name == null) {            // null과 undefined 동시에 (== 활용)
    return '이름 없음';
  }
  return name.toUpperCase();    // 이 시점에서 name은 string
}

// 더 명시적으로 구분하고 싶을 때
if (name === null) { /* null 처리 */ }
if (name === undefined) { /* undefined 처리 */ }
if (name != null) { /* null/undefined 아닌 경우 */ }
```

```txt
== null (느슨한 비교)은 null과 undefined 둘 다를 한 번에 걸러주는 유일한 상황 —
평소엔 ===을 써야 하지만 이 경우만은 == null이 관용적인 패턴
(== vs ===의 일반론은 [[JS_Operators]] 참고)
```

---

# in 연산자 — 객체에 특정 키가 있는지 ⭐️⭐️⭐️⭐️

```typescript
type Cat = { meow: () => void };
type Dog = { bark: () => void };

function makeSound(animal: Cat | Dog) {
  if ('meow' in animal) {
    animal.meow(); // Cat
  } else {
    animal.bark(); // Dog
  }
}
```

```txt
interface/type alias처럼 런타임에 없어지는 타입들을 구분할 때 주로 씀
"이 객체에 이 키가 있으면 이 타입이다"라는 방식으로 구조적으로 판별
(in 연산자 자체의 동작은 [[JS_Operators]] 참고)

API 응답 타입이 여러 모양일 때 특히 유용:
```

```typescript
type SuccessResponse = { data: unknown; status: 'ok' };
type ErrorResponse = { error: string; status: 'error' };

function handleResponse(res: SuccessResponse | ErrorResponse) {
  if ('data' in res) {
    res.data; // SuccessResponse
  } else {
    res.error; // ErrorResponse
  }
}
```

---

# 사용자 정의 타입 가드 — is 키워드 ⭐️⭐️⭐️⭐️

```typescript
function isString(value: unknown): value is string {
  return typeof value === 'string';
}
```

```txt
반환 타입을 boolean 대신 "value is string"으로 선언하면:
  TS가 "이 함수가 true를 반환한 블록 안에서는 value가 string이다"라고 인식해줌
  → 재사용 가능한 타입 가드 함수를 직접 만드는 방법
```

```typescript
// 실전 — API 응답이 특정 모양인지 검증
type ApiUser = { id: number; email: string };

function isApiUser(value: unknown): value is ApiUser {
  return (
    typeof value === 'object' &&
    value !== null &&
    'id' in value &&
    'email' in value &&
    typeof (value as ApiUser).id === 'number' &&
    typeof (value as ApiUser).email === 'string'
  );
}

const data: unknown = await fetch('/api/user').then(r => r.json());
if (isApiUser(data)) {
  data.id;    // ApiUser 타입으로 안전하게 접근
  data.email;
}
```

```txt
is 키워드가 없이 그냥 boolean을 반환하면:
  if (isString(value)) { ... } 블록 안에서도 value의 타입은 여전히 unknown 그대로
  → is 키워드가 "이 함수의 반환이 true일 때 이 매개변수는 이 타입이다"라는 약속을 TS에 알려주는 것

⚠️ 이 약속이 런타임에서도 맞아야 함 — TS는 그 함수 내부 구현을 믿고 따라가는 것이지,
   직접 검증하지는 않음 → 구현이 잘못되면 컴파일은 통과해도 런타임에 에러가 날 수 있음
```

---

# unknown — any보다 안전한 "모르는 타입" ⭐️⭐️⭐️⭐️

```typescript
function process(value: unknown) {
  value.toUpperCase();     // ❌ — unknown에는 아무 메서드도 바로 못 씀

  if (typeof value === 'string') {
    value.toUpperCase();   // ✅ — 타입 가드 통과 후에는 사용 가능
  }
}
```

```txt
any vs unknown:
  any   → 모든 타입 체크를 끔 — 어디서든 마음대로 접근 가능, TS가 아무것도 안 잡아줌
  unknown → "일단 모르는 타입"이지만 타입 체크를 켜놓음
            바로 사용 불가, 반드시 타입 가드로 확인 후에만 사용 가능

→ "타입을 모르는 값"을 다룰 때는 any 대신 unknown을 쓰고, 쓰기 전에 타입 가드로 확인하는 게 안전함
  (API 응답, try/catch의 error, JSON.parse 결과 등이 대표적인 unknown 상황)
```

```typescript
// try/catch의 error는 unknown 타입
try {
  await api.call();
} catch (error) {
  if (error instanceof Error) {
    console.error(error.message); // Error로 좁혀진 후에야 .message 사용 가능
  }
}
```

---

# 타입 가드 선택 기준 ⭐️⭐️⭐️⭐️

|판별 대상|방법|
|---|---|
|원시 타입(string/number/boolean)|`typeof value === 'string'`|
|null / undefined|`value === null` / `value == null`(둘 다)|
|배열|`Array.isArray(value)`|
|클래스 인스턴스|`value instanceof ClassName`|
|interface/type — 특정 키 유무로|`'key' in value`|
|재사용 가능한 복잡한 검증|사용자 정의 타입 가드 (`value is Type`)|

---

# 한눈에

```txt
타입 좁히기: 여러 타입이 될 수 있는 값을, 조건 블록 안에서 더 구체적인 타입으로 확정하는 것
타입 가드: 그 조건식 패턴 — TypeScript가 "이 조건을 통과하면 이 타입이다"라고 인식하는 특정 패턴들

typeof      원시 타입 — ⚠️ 배열/null은 둘 다 'object'라 구분 못 함
Array.isArray  배열 전용 — typeof의 한계를 메움
instanceof  클래스 인스턴스 — interface/type에는 못 씀(런타임에 존재 안 함)
in          객체의 키 존재 여부 — interface/type을 구분할 때 주로 씀
value is T  사용자 정의 타입 가드 — 재사용 가능, 구현 정확성은 개발자 책임
== null     null과 undefined 동시 체크 — 이 경우만 ==가 관용적

unknown vs any: unknown은 타입 가드 필수, any는 체크 없이 통과 — 외부 데이터는 unknown이 더 안전
```