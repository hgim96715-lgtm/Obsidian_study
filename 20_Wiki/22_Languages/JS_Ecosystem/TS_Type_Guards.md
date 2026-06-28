---
aliases:
  - 타입 가드
  - 타입 좁히기
  - instanceof
  - narrowing
  - type guard
  - type predicate
  - typeof
tags:
  - TypeScript
related:
  - "[[00_JS_Ecosystem_HomePage]]"
  - "[[NestJS_DTO]]"
---
# TS_Type_Guards — 타입 가드 & 타입 좁히기

# 한 줄 요약

```txt
타입 가드 = 넓은 타입을 특정 조건 안에서 좁은 타입으로 좁히는 것
Union 타입 / unknown 타입을 다룰 때, 또는 string 중 "특정 값들" 만 허용하고 싶을 때 필수
```

---

---

# 왜 필요한가 ⭐️

```typescript
function process(value: string | number) {
  value.toUpperCase();   // ❌ 에러
  // number 에는 toUpperCase 없음 — TS 는 string 인지 number 인지 아직 모름
}
```

```txt
string | number 같은 Union 타입이나 unknown 타입은 "어떤 값인지 모르는 상태"
→ 특정 블록 안에서 타입을 확정해줘야 해당 타입의 메서드를 안전하게 사용 가능

이 과정이 "타입 좁히기(Narrowing)"
타입을 좁히는 조건문이 "타입 가드"
```

## 더 흔하게 마주치는 상황 — "이 문자열이 내가 원하는 값들 중 하나인가?" ⭐️

```typescript
type ThemePreference = 'system' | 'light' | 'dark';

// localStorage 는 항상 string | null 을 줌 — 'system'/'light'/'dark' 인지는 보장 안 됨
const stored: string | null = localStorage.getItem('theme');

function applyTheme(theme: ThemePreference) { /* ... */ }

applyTheme(stored);   // ❌ string | null 은 ThemePreference 가 아님 → 에러
```

```txt
실무에서 타입 가드가 자주 필요한 또 다른 이유:
  localStorage / URL 쿼리 / 환경변수 / API 응답 등은
  TypeScript 입장에서 전부 "그냥 string(또는 unknown)" 일 뿐임
  → 그 문자열이 내가 정의한 리터럴 유니온(ThemePreference) 중 하나인지
    런타임에 직접 확인해줘야 안전하게 좁은 타입으로 쓸 수 있음

→ 바로 이 상황에 쓰는 게 이 노트 끝에서 다시 다룰 isThemePreference 패턴
```

---

---

# typeof — 원시 타입 확인 ⭐️

```typescript
function process(value: string | number) {
  if (typeof value === 'string') {
    console.log(value.toUpperCase());   // ✅ string 으로 확정
  } else {
    console.log(value.toFixed(2));      // ✅ number 로 확정
  }
}
```

```txt
typeof 가 반환하는 값:
  'string' / 'number' / 'boolean' / 'undefined' / 'function'
  'object'  → 객체 / 배열 / null ⚠️ (null 도 'object' 로 나옴)

⚠️ typeof null === 'object':
  null 을 걸러내려면 추가 체크 필요
  if (typeof val === 'object' && val !== null) { ... }

언제: string / number / boolean 같은 원시 타입을 구분할 때
```

## typeof 로 "선언조차 안 된 변수" 안전하게 확인하기 — SSR 환경 체크 ⭐️

```typescript
// window, document, localStorage 는 브라우저에만 있는 전역 객체
// Node.js 서버 환경(Next.js Server Component, 빌드 타임 등)에는 존재하지 않음

if (typeof window !== 'undefined') {
  // 이 블록은 브라우저에서 실행될 때만 들어옴
  window.localStorage.setItem('theme', 'dark');
}
```

```txt
왜 window !== undefined 가 아니라 typeof window !== 'undefined' 인가:

  window !== undefined
    → window 라는 변수를 직접 참조함
    → 서버 환경엔 window 가 "선언조차" 안 되어 있어서
      ReferenceError: window is not defined 가 그 자리에서 바로 발생함 ❌

  typeof window !== 'undefined'
    → typeof 는 변수가 선언 안 돼 있어도 에러 없이 문자열 'undefined' 를 반환함
    → 존재 자체가 불확실한 변수도 안전하게 확인 가능 ✅

엄밀히는 "타입 좁히기"가 아니라
"지금 이 코드가 어떤 실행 환경에 있는지" 확인하는 용도로 typeof 를 빌려 쓰는 것
```

```bash
자주 같이 쓰는 것들:
  typeof window !== 'undefined'        브라우저 환경인지
  typeof document !== 'undefined'      DOM 접근 가능한지
  typeof localStorage !== 'undefined'  localStorage 사용 가능한지

Next.js 에서 특히 필요한 이유:
  Server Component / 빌드 타임 렌더링은 Node.js 환경 → window 자체가 없음
  → window 를 쓰는 코드는 'use client' 컴포넌트 안에서만 두거나
    이 가드로 감싸서 브라우저에서만 실행되게 해야 함
  # → 관련: [[Next_Data_Fetching]] 의 Server vs Client Component
```

---

---

# instanceof — 클래스 인스턴스 확인 ⭐️

```txt
instanceof 는 "이 값이 특정 클래스로 만들어진 객체인가?" 를 확인
A instanceof B → A 가 B 클래스의 인스턴스면 true

원리: new 로 객체를 만들면 그 객체의 프로토타입 체인에 클래스가 연결되고
      instanceof 는 이 체인을 타고 올라가며 확인함
```

```typescript
class Dog { bark() { console.log('왈!'); } }
class Cat { meow() { console.log('야옹!'); } }

function makeSound(animal: Dog | Cat) {
  if (animal instanceof Dog) {
    animal.bark();   // Dog 로 확정
  } else {
    animal.meow();   // Cat 으로 확정
  }
}
```

## instanceof 로 에러 타입 구분 ⭐️

```typescript
// catch(e) 에서 e 의 타입은 unknown → 바로 e.message 접근 불가
try {
  await someOperation();
} catch (e) {
  if (e instanceof Error) {
    console.log(e.message);   // ✅ Error 로 확정
    console.log(e.stack);
    console.log(e.name);
  }
}
```

```typescript
// JWT 에러 종류 구분
import { TokenExpiredError, JsonWebTokenError } from 'jsonwebtoken';

try {
  const payload = jwt.verify(token, secret);
} catch (e) {
  if (e instanceof TokenExpiredError) {
    throw new UnauthorizedException('토큰이 만료되었습니다.');
  }
  if (e instanceof JsonWebTokenError) {
    throw new UnauthorizedException('유효하지 않은 토큰입니다.');
  }
  throw new UnauthorizedException('알 수 없는 에러');
}
```

```txt
instanceof 는 상속 체인도 확인:
  TokenExpiredError 인스턴스
  → instanceof TokenExpiredError  true
  → instanceof JsonWebTokenError  true  (부모 클래스)
  → instanceof Error              true  (최상위)
  → instanceof TypeError          false (다른 계열)
```

---

---

# in — 속성 존재 여부 확인

```txt
'속성명' in 객체 → 그 속성이 있으면 true
class 가 아닌 interface / type 으로 정의된 Union 에서 고유 속성으로 구분할 때 사용
```

```typescript
interface Bird { fly(): void; feathers: number; }
interface Fish { swim(): void; fins: number; }

function move(animal: Bird | Fish) {
  if ('fly' in animal) {
    animal.fly();              // Bird 로 확정
    console.log(animal.feathers);
  } else {
    animal.swim();             // Fish 로 확정
    console.log(animal.fins);
  }
}
```

```txt
언제: class 없이 interface 로만 정의된 Union 타입을, 서로 다른 고유 속성으로 구분할 때
```

---

---

# 값 비교로 좁히기 — 문자열 리터럴 Union ⭐️

```txt
string | number 처럼 "타입 종류" 가 다른 게 아니라
'system' | 'light' | 'dark' 처럼 "같은 string 인데 그중 특정 값들" 만 허용하고 싶을 때는
typeof / instanceof 가 아니라 === 로 직접 값을 비교해서 좁힘
```

```typescript
type ThemePreference = 'system' | 'light' | 'dark';

function applyTheme(theme: ThemePreference) { /* ... */ }

const stored: string | null = localStorage.getItem('theme');

// 인라인으로 직접 비교 — 별도 가드 함수 없이도 TS 가 알아서 좁혀줌
if (stored === 'system' || stored === 'light' || stored === 'dark') {
  applyTheme(stored);   // ✅ 이 블록 안에서 stored 는 ThemePreference 로 자동 좁혀짐
}
```

```txt
=== 로 문자열 리터럴과 비교하면, TS 는 그 비교식 자체를 보고
"이 블록에서는 stored 가 'system' 이거나 'light' 이거나 'dark' 다" 라고 추론함
→ 인라인으로 쓸 때는 이걸로 충분, 가드 함수가 따로 필요 없음

그런데 이 비교 로직을 여러 함수에서 반복해서 써야 한다면?
→ 함수로 추출하고 싶어짐 → 바로 아래 "사용자 정의 타입 가드" 로 이어짐
   (여기서부터 is 가 등장하는 이유가 나옴)
```

---

---

# 사용자 정의 타입 가드 — is ⭐️⭐️

```txt
반복되는 타입 좁히기 로직을 함수로 추출해서 재사용
반환 타입: value is 타입
  → 이 함수가 true 를 반환하면 TypeScript 가 해당 타입으로 좁혀줌
```

## unknown 을 구체적 타입으로 — 가장 흔한 형태

```typescript
function isString(value: unknown): value is string {
  return typeof value === 'string';
}

function isError(value: unknown): value is Error {
  return value instanceof Error;
}

function isUser(value: unknown): value is { id: number; email: string } {
  return (
    typeof value === 'object' &&
    value !== null &&
    'id' in value &&
    'email' in value
  );
}
```

## string 을 더 좁은 리터럴 Union 으로 — isThemePreference 패턴 ⭐️

```typescript
type ThemePreference = 'system' | 'light' | 'dark';

const isThemePreference = (value: string): value is ThemePreference => {
  return value === 'system' || value === 'light' || value === 'dark';
};
```

```txt
isString(value: unknown)          전혀 모르는 값(unknown) → 구체적 타입으로 좁힘
isThemePreference(value: string)  이미 string 인 건 아는데 → 그중 특정 리터럴들인지 확인

입력 타입이 unknown 이냐 string 이냐 차이일 뿐, "is" 의 역할은 동일함
```

## ⭐️ 왜 함수로 추출하면 is 가 필요해지는가 (인라인과의 차이)

```typescript
// ❌ is 없이 그냥 boolean 만 반환하면
const isThemePreference = (value: string): boolean => {
  return value === 'system' || value === 'light' || value === 'dark';
};

const stored = localStorage.getItem('theme');
if (stored && isThemePreference(stored)) {
  applyTheme(stored);   // ❌ 여전히 string 타입 — 에러! (ThemePreference 가 아님)
}
```

```typescript
// ✅ value is ThemePreference 로 반환하면
const isThemePreference = (value: string): value is ThemePreference => {
  return value === 'system' || value === 'light' || value === 'dark';
};

if (stored && isThemePreference(stored)) {
  applyTheme(stored);   // ✅ ThemePreference 로 확정됨
}
```

```txt
바로 위 "값 비교로 좁히기" 의 인라인 if 문은 is 없이도 잘 좁혀졌는데,
함수로 감싸는 순간 왜 is 가 필요해지는가:

  인라인 비교 (if (stored === 'system' || ...))
    → TS 가 "그 비교식 자체"를 직접 보고 있어서 알아서 좁혀줌

  함수로 감싸면
    → TS 는 함수를 호출하는 쪽에서는 "함수의 반환 타입"만 보고 판단함
    → 그냥 boolean 이면 "true/false 를 반환하는 함수" 라는 정보만 있을 뿐
      "true 일 때 입력값이 어떤 타입인지" 는 전혀 알려주지 않음
    → value is ThemePreference 라고 명시해야
      "이 함수가 true 를 반환하면, 입력값은 ThemePreference 다" 라는 약속을
      함수 경계를 넘어 호출하는 쪽까지 전달할 수 있음

한 줄로: is 는 "함수 경계를 넘어서도 좁히기 정보가 유지되게 해주는 표시" 임
```

---

---

# catch(e) unknown 처리 패턴 ⭐️

```typescript
// TypeScript 4.0+ 에서 catch 의 e 타입 = unknown → 바로 e.message 접근 불가

// 패턴 1: instanceof Error (가장 안전, 권장)
try {
  await riskyOperation();
} catch (e) {
  if (e instanceof Error) {
    console.log(e.message);
    throw e;
  }
  throw new Error('알 수 없는 에러');
}

// 패턴 2: e.name 으로 에러 종류 구분
catch (e) {
  if (e instanceof Error) {
    switch (e.name) {
      case 'TokenExpiredError':   throw new UnauthorizedException('토큰 만료');
      case 'JsonWebTokenError':   throw new UnauthorizedException('잘못된 토큰');
      default:                   throw new UnauthorizedException('인증 에러');
    }
  }
}

// 패턴 3: as Error (빠르지만 안전하지 않음 — TS 에게 거짓말하는 것)
catch (e) {
  const error = e as Error;
  console.log(error.message);   // 런타임 에러 가능성 있음
}
```

```txt
패턴 1 권장: Error 가 아닌 값도 throw 될 수 있음 (throw 'string' 등)
            instanceof Error 로 실제로 확인하고 쓰는 게 안전함
```

---

---

# 타입 좁히기 흐름 ⭐️

```typescript
function process(value: string | number | null | undefined) {
  // 1단계: null / undefined 한번에 제거
  if (value == null) return;
  // 이후 value: string | number

  // 2단계: 원시 타입 구분
  if (typeof value === 'string') {
    return value.toUpperCase();
  }
  return value.toFixed(2);
}
```

```bash
value == null 패턴:
  == (느슨한 비교) 사용 — null 과 undefined 를 동시에 걸러냄
  value == null  → true if value is null or undefined
  value === null → true if value is null only
  (자세한 ==/=== 차이는 # [[JS_Control_Flow]] 참고)
```

---

---

# 좁히기 도구 선택 기준

```txt
원시 타입 (string / number / boolean) 구분
  → typeof

클래스로 만들어진 객체 / 에러 종류 구분
  → instanceof ⭐️

interface / type 으로 정의된 객체 구분 (속성으로 구분)
  → in

string 중 "특정 리터럴 값들" 만 허용하고 싶음 (한 곳에서만 비교)
  → === 로 직접 비교 (인라인이면 충분)

같은 검증 로직을 여러 함수/파일에서 재사용 / API 응답 검증 / unknown 처리
  → 사용자 정의 타입 가드 (is)
```

---

---

# 한눈에

| 방법                  | 언제                   | 예시                              |
| ------------------- | -------------------- | ------------------------------- |
| `typeof`            | 원시 타입                | `typeof x === 'string'`         |
| `typeof` (변수 존재 체크) | 선언 안 됐을 수도 있는 전역 변수  | `typeof window !== 'undefined'` |
| `instanceof`        | 클래스·에러 인스턴스 ⭐️       | `e instanceof Error`            |
| `in`                | 속성 존재 여부             | `'fly' in animal`               |
| `{text}===` 직접 비교   | 문자열 리터럴 Union, 1회성   | `stored === 'light'`            |
| `is` (타입 서술)        | 위 로직을 재사용 가능한 함수로    | `value is ThemePreference`      |
| `{text}== null`     | null + undefined 한번에 | `value == null`                 |

```txt
is 가 필요한 순간을 기억하는 방법:
  타입 좁히기 로직을 "함수로 뽑아내고 싶다" → 그 순간부터 is 필요
  그냥 if 문 안에서 한 번만 비교한다 → is 없이도 TS 가 알아서 좁혀줌
```