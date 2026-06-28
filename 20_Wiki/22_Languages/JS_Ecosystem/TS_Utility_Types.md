---
aliases:
  - Record
  - Partial
  - Pick
  - Omit
  - ReturnType
  - utility types
tags:
  - TypeScript
related:
  - "[[00_JS_Ecosystem_HomePage]]"
  - "[[NestJS_DTO]]"
  - "[[TS_Generics]]"
  - "[[NestJS_Controller]]"
  - "[[React_useRef]]"
---
# TS_Utility_Types — Record, Partial, Pick, Omit 등

> [!info] 
> TS가 기본 제공하는 "타입을 변형해서 새 타입을 만드는" 도구들이다. 객체 모양을 처음부터 다시 적지 않고, 기존 타입에서 일부를 빼거나(Omit) 고르거나(Pick) 선택적으로 만들거나(Partial) 하는 식으로 재사용한다.

---

# Record<K, V> — 키-값 객체 타입 ⭐️⭐️⭐️⭐️

```typescript
const messages: Record<string, string> = {
  isEmail: '올바른 이메일을 입력해주세요.',
  isNotEmpty: '필수 항목입니다.',
};
```

```txt
Record<K, V> = "키는 K 타입, 값은 V 타입"인 객체
  { [key: string]: string } 처럼 직접 인덱스 시그니처를 적는 것과 거의 같은 의미지만,
  Record가 더 짧고 "이건 키-값 매핑 객체다"라는 의도를 한눈에 드러냄
```

---

# `Partial<T>` — 모든 속성을 옵셔널로 ⭐️⭐️⭐️⭐️

```typescript
interface User { name: string; age: number; }

type PartialUser = Partial<User>;
// { name?: string; age?: number; } 와 동일 — 둘 다 있어도, 하나만 있어도, 아예 없어도 됨
```

## 중첩된 형태 — Record<string, Partial<Record<string, string>>> 분해 ⭐️⭐️⭐️⭐️

```typescript
const FIELD_MESSAGES: Record<string, Partial<Record<string, string>>> = {
  email: { isEmail: '올바른 이메일을 입력해주세요.' },
  password: { minLength: '비밀번호는 8자 이상이어야 합니다.' },
};
```

```txt
안에서 바깥으로 풀면:
  Record<string, string>                    → "제약 이름 → 메시지" 객체 (예: { isEmail: '...' })
  Partial<Record<string, string>>            → 그 객체의 모든 키가 옵셔널 — 모든 제약이 다 있을 필요는 없음
  Record<string, Partial<Record<string,...>>> → "필드 이름 → (제약별 메시지, 일부만 있을 수 있음)" 객체

전체 의미: "필드 이름으로 들어가면, 그 필드가 가질 수 있는 제약들의 메시지 묶음이 나오는데,
           그 메시지들이 전부 다 있는 건 아닐 수도 있다"
```

### Partial이 왜 필요한가 — 없으면 생기는 문제 ⭐️⭐️⭐️⭐️

```typescript
// Partial 없이 Record<string, Record<string, string>> 라면
FIELD_MESSAGES['email']['isEmail'];  // TS가 "이건 항상 string"이라고 믿어버림

// Partial이 있으면
FIELD_MESSAGES['email']?.['isEmail']; // TS가 "string | undefined"로 정확히 봄 → ?. 가 타입상으로도 강제됨
```

```txt
Record<string, V>는 "이 객체의 어떤 키로 접근해도 항상 V 타입의 값이 있다"고 TS에게 약속하는 것
실제로는 email/password 같은 일부 키만 있고 나머지 무수히 많은 문자열 키는 없는데,
Record<string, V>만 쓰면 TS는 그 차이를 모름 — Partial을 씌워야 "없을 수도 있다"는 게 타입에 반영됨

→ FIELD_MESSAGES[property]?.[constraint] 처럼 ?.(옵셔널 체이닝)를 쓰는 게 의미 있어지는 것도
  바로 이 Partial 덕분 — Partial이 없었다면 TS 입장에서 그 ?.는 "불필요한 안전장치"로 보였을 것
  (?. 자체의 동작은 [[JS_OptionalChaining]] 참고)
```

---

# Pick<T, K> / Omit<T, K> — 일부만 고르거나 빼기 ⭐️⭐️⭐️⭐️

```typescript
interface User { id: number; email: string; password: string; }

type PublicUser = Omit<User, 'password'>;        // { id: number; email: string; }
type LoginInput = Pick<User, 'email' | 'password'>; // { email: string; password: string; }
```

|유틸리티|역할|
|---|---|
|`Pick<T, K>`|T에서 K로 지정한 속성들만 골라서 새 타입을 만듦|
|`Omit<T, K>`|T에서 K로 지정한 속성들을 제외한 나머지로 새 타입을 만듦|

```txt
Pick과 Omit은 정확히 반대 방향에서 같은 일을 함 — "남길 것"을 지정하느냐, "뺄 것"을 지정하느냐
필드가 적으면 Pick, 필드가 많고 빼는 게 적으면 Omit이 더 짧아짐
```

---

# Omit + 교차(&) — 서드파티 타입의 필드를 재정의하는 패턴 ⭐️⭐️⭐️⭐️

```typescript
// 이미 user 필드가 다른 타입으로 존재하는 third-party 타입이 있다고 가정
type RequestWithUser = Omit<Request, 'user'> & { user: JwtPayload };
```

```txt
왜 Omit이 먼저 필요한가:
  그냥 Request & { user: JwtPayload } 라고만 하면, 원래 Request에 user 필드가
  전혀 없을 때는 문제없이 "추가"가 되지만, 원래도 user 필드가 있었다면(타입이 다르면)
  교차 타입 안에서 같은 키에 대해 서로 호환 안 되는 두 타입이 충돌하게 됨

  → 먼저 Omit으로 기존 user 필드를 제거한 뒤, 새 모양의 user를 교차(&)로 추가하면
    "필드를 추가하는 것"이 아니라 "필드를 다른 타입으로 교체하는 것"이 안전하게 표현됨

[[NestJS_Controller]]에서 본 Request & { user?: JwtPayload }는 원래 Request에 user 필드가
전혀 없는 경우라 Omit 없이 그냥 교차만 한 것 — 둘 다 같은 발상이고, 원래 필드가 있었는지 여부로 Omit 필요성이 갈림
```

---

# `ReturnType<T>` — 함수의 반환 타입 추출 ⭐️⭐️⭐️

```typescript
const timerRef = useRef<ReturnType<typeof setTimeout> | null>(null);
```

```txt
typeof setTimeout         → setTimeout이라는 "함수 자체"의 타입을 얻음
ReturnType<typeof setTimeout> → 그 함수를 호출했을 때 "반환되는 값"의 타입만 뽑아냄

setTimeout이 환경(브라우저/Node)에 따라 반환 타입이 다를 수 있어서(number vs NodeJS.Timeout 등),
그 타입을 직접 알아내 적기보다 ReturnType으로 "지금 이 환경에서 setTimeout이 실제로 반환하는 타입"을
자동으로 가져오는 것 — useRef로 타이머 id를 저장하는 패턴 자체는 [[React_useRef]] 참고
```

---

# `Required<T>` /` Readonly<T>` — 짧게 ⭐️

```typescript
interface Config { url?: string; }

type FullConfig = Required<Config>;   // { url: string; } — Partial의 반대, 전부 필수로
type FrozenUser = Readonly<User>;     // 모든 필드가 readonly — 재할당 불가
```

---

# ⚠️ NestJS의 PartialType/OmitType/PickType과 혼동 주의 ⭐️⭐️⭐️⭐️

```txt
이름이 거의 같아서 헷갈리기 쉬운 다른 것:
  TS 내장          Partial<T> / Omit<T,K> / Pick<T,K>     — 순수 타입 변형, 런타임에 아무 일도 안 함
  @nestjs/mapped-types  PartialType() / OmitType() / PickType() — 실제 클래스를 만들어서 반환하는 함수

NestJS 버전이 따로 필요한 이유:
  DTO는 클래스이고, class-validator 데코레이터(@IsString() 등)가 그 필드에 붙어있음
  TS의 Partial<CreateDto> 는 "타입"만 옵셔널로 바꿀 뿐, 데코레이터(런타임 검증 로직)까지는 안 따라옴
  → NestJS의 PartialType(CreateDto)는 필드를 옵셔널로 바꾸면서 데코레이터도 그대로 복사해서
    "진짜 동작하는 DTO 클래스"를 새로 만들어줌 — 그래서 UpdateDto에는 항상 이 NestJS 버전을 씀
    (DTO에 적용하는 실전 예시는 [[NestJS_DTO]]의 "Mapped Types" 참고)

→ 일반 interface/type에는 TS 내장 버전, class-validator가 붙은 DTO 클래스에는 NestJS 버전
```

---

# 한눈에

|유틸리티|역할|
|---|---|
|`Record<K, V>`|키-값 객체 타입|
|`Partial<T>`|모든 속성을 옵셔널로|
|`Required<T>`|모든 속성을 필수로 (Partial의 반대)|
|`Readonly<T>`|모든 속성을 재할당 불가로|
|`Pick<T, K>`|지정한 속성만 골라서 새 타입|
|`Omit<T, K>`|지정한 속성만 제외하고 새 타입|
|`Omit<T,K> & {...}`|서드파티 타입의 필드를 다른 타입으로 안전하게 교체|
|`ReturnType<typeof fn>`|함수가 반환하는 값의 타입 추출|
|NestJS `PartialType()` 등|TS 내장과 이름은 같지만 실제 클래스+데코레이터까지 만드는 런타임 함수|