---
aliases:
  - as
  - 타입 단언
  - satisfies
tags:
  - TypeScript
related:
  - "[[00_JS_Ecosystem_HomePage]]"
  - "[[TS_Generics]]"
  - "[[TS_Type_Guards]]"
---
# TS_TypeAssertion — 타입 단언

> [!info] 
> 타입 단언 = TypeScript의 타입 추론을 개발자가 직접 덮어쓰는 것.
>  `as`(타입 강제), `!`(null 아님 단언), `satisfies`(검증 후 유지)가 대표적. 
>  컴파일 타임에만 작동 — 잘못 쓰면 런타임 에러가 생겨도 TypeScript가 잡아주지 못한다.

---

# 타입 단언이란 — 왜 위험한가 ⭐️⭐️⭐️⭐️

```txt
TypeScript는 코드를 분석해서 타입을 추론함
단언 = "TypeScript야, 네 추론 대신 내 말을 믿어"

핵심 위험:
  단언은 컴파일 타임에만 작동
  런타임에 실제 타입이 다르면 → 에러가 그대로 발생
  TypeScript가 미리 잡아주지 못함

  as string을 붙인다고 값이 string이 되는 게 아님
  TypeScript가 string으로 "보는" 것뿐
```

```typescript
// as의 위험 — 런타임 에러를 TypeScript가 못 잡음
const el = document.querySelector('.btn') as HTMLButtonElement;
el.click();  // 실제로 .btn이 없으면 el = null → 런타임 에러
// TypeScript는 null이 아닌 HTMLButtonElement로 믿어서 경고 없음
```

```txt
단언이 필요한 경우가 분명히 있지만
"TypeScript가 틀렸고 내가 맞다"는 확신이 있을 때만 써야 함
```

---

# as — 타입 강제 ⭐️⭐️⭐️⭐️

```typescript
// 기본 사용
const input = document.getElementById('search') as HTMLInputElement;
input.value;  // ✅ HTMLElement에는 value 없지만 HTMLInputElement에는 있음

// API 응답 타입 지정
const data = response.json() as Promise<User>;

// 타입 좁히기
const shape = getShape() as Circle;  // Shape 유니온에서 Circle로
```

```txt
as 를 써도 되는 경우:
  ① DOM — getElementById, querySelector 등이 더 넓은 타입을 반환
     실제 HTML 구조를 알고 있을 때
  ② API 응답 — 서버 타입을 알고 있을 때
  ③ 유니온 타입 — 로직상 특정 타입임이 확실할 때
```

## as unknown as — 이중 단언 ⭐️⭐️⭐️

```typescript
// 연관 없는 두 타입 사이를 강제로 변환
const str = 'hello' as unknown as number;  // ⚠️ 말이 안 되는 변환

// 이게 필요한 경우:
//   A as B 로 변환이 안 될 때 (두 타입이 겹치는 부분이 없을 때)
//   중간에 unknown을 거쳐서 어떤 타입이든 변환 가능

// 실전 — 테스트에서 mock 객체 만들 때
const mockUser = { id: '1', name: 'test' } as unknown as User;
```

```txt
as unknown as 는 더 위험:
  TypeScript가 "정말로 연관 없는 타입"이라서 막는 걸 뚫는 것
  진짜 어쩔 수 없을 때만 (테스트 mock, 라이브러리 타입 문제 등)
```

## as 대신 타입 가드가 더 안전한 경우

```typescript
// ❌ as — 틀려도 런타임에 터짐
const user = data as User;
user.email.toUpperCase();  // data에 email이 없으면 런타임 에러

// ✅ 타입 가드 — 실제로 확인 후 사용
function isUser(val: unknown): val is User {
  return typeof val === 'object' && val !== null && 'email' in val;
}
if (isUser(data)) {
  data.email.toUpperCase();  // 검증 후 사용 → 안전
}
```

---

# ! — 두 가지 단언 ⭐️⭐️⭐️⭐️

## value! — null 아님 단언

```typescript
// TypeScript: getElementById 반환 타입 = HTMLElement | null
const el = document.getElementById('app');

el.style.color = 'red';    // ❌ el이 null일 수 있음
el!.style.color = 'red';   // ✅ ! → "el은 절대 null이 아니야"
```

```typescript
// 자주 보이는 패턴
const user = users.find(u => u.id === id)!;
//                                       ↑ 반드시 있다고 확신

const ref = useRef<HTMLInputElement>(null);
ref.current!.focus();
//          ↑ 마운트 후에는 반드시 있다고 확신
```

```txt
! vs ?.:
  el?.style.color = 'red'  → null이면 조용히 건너뜀 (안전)
  el!.style.color = 'red'  → null이면 런타임 에러 (확신이 있을 때만)

  조금이라도 불안하면 ?. 또는 if 체크
```

## property!: type — 확정 할당 단언 ⭐️⭐️⭐️⭐️

```typescript
// strict 모드에서 클래스 프로퍼티는 반드시 초기화해야 함
class MyClass {
  name: string;      // ❌ 초기화 안 됨 → 에러
  name: string = ''; // ✅ 기본값 초기화
  name!: string;     // ✅ "런타임에 반드시 채워질 거야" 단언
}
```

```typescript
// NestJS DTO에서 자주 보이는 이유
export class CreateReportDto {
  @IsString()
  @MinLength(2)
  reason!: string;   // ← !가 붙는 이유
}
```

```txt
DTO에서 reason!: string인 이유:
  class-validator + ValidationPipe가 런타임에 프로퍼티를 채워줌
  TypeScript는 이걸 모름 → "constructor에서 초기화 안 됐잖아" 에러

  ! 로 해결 — "런타임에 반드시 값이 들어올 거야"

  대안 비교:
    reason?: string    optional이 돼버림 — 의도와 다름
    reason: string = '' 빈 문자열 초기화 — @MinLength(2) 검증 실패 가능
    reason!: string    ✅ DTO에서 가장 일반적인 방법
```

## ! 사용 기준

```txt
✅ 써도 되는 경우:
  DOM id가 항상 존재함을 HTML 구조로 확신할 때
  useRef.current를 이벤트 핸들러(마운트 후)에서 접근할 때
  NestJS DTO 프로퍼티 (class-validator가 런타임에 채워줌)
  find() 결과가 로직상 반드시 있음을 보장할 때

❌ 쓰면 안 되는 경우:
  "아마 있을 것 같다"는 추측
  외부 API 응답 같은 불확실한 데이터
  에러 처리를 피하기 위해 남용
```

---

# satisfies — 검증은 하되 타입은 유지 ⭐️⭐️⭐️

```typescript
type Config = {
  theme: 'light' | 'dark';
  lang:  'ko' | 'en';
};

// as — 강제 변환, 리터럴 타입 잃음
const config = {
  theme: 'light',
  lang:  'ko',
} as Config;
config.theme  // 타입: 'light' | 'dark' (넓어짐)

// satisfies — 검증 + 리터럴 타입 유지
const config = {
  theme: 'light',
  lang:  'ko',
} satisfies Config;
config.theme  // 타입: 'light' (리터럴 그대로)
```

```txt
as vs satisfies 차이:
  as        → TypeScript에 강요 (검증 없음, 틀려도 에러 안 남)
  satisfies → TypeScript에 검증 요청 (틀리면 에러) + 리터럴 타입 유지

  as Config: theme이 'dark'여도 에러 없음
  satisfies Config: theme이 'purple'이면 에러

언제 satisfies:
  객체 리터럴에 타입을 "확인"만 받고 싶을 때
  자동완성을 유지하면서 타입 체크도 받고 싶을 때
  as보다 안전한 대부분의 경우에 satisfies 우선 고려
```

---

# as const — 리터럴 타입 고정

```typescript
const direction = 'left' as const;
// 'left' (리터럴) — string으로 넓혀지지 않음

const config = { theme: 'dark', size: 3 } as const;
// { readonly theme: 'dark'; readonly size: 3 }
// → 값 변경 불가 + 리터럴 타입 유지
```

```txt
as const 자세한 내용 → [[TS_Type_Guards]] as const 섹션
```

---

# 단언 선택 기준

```txt
상황에 따른 선택:

  값이 null/undefined가 아님이 확실  →  ! (non-null assertion)
  타입이 A인데 B로 보고 싶음         →  as B
  객체 리터럴을 타입에 맞는지 확인   →  satisfies
  리터럴 타입 그대로 유지             →  as const
  실제로 타입을 검증하고 좁히기       →  타입 가드 (is)  → [[TS_Type_Guards]]

  as, ! 보다 타입 가드가 더 안전한 경우가 많음
  "단언 없이 해결되면" 단언 쓰지 않는 것이 원칙
```