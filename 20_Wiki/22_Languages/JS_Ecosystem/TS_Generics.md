---
aliases:
  - generics
  - T
  - "children: ReactNode"
  - ReactNode
  - satisfies
tags:
  - TypeScript
related:
  - "[[00_JS_Ecosystem_HomePage]]"
  - "[[JS_Promise]]"
  - "[[NextJS_API_Client]]"
  - "[[React_useRef]]"
  - "[[TS_TypeAssertion]]"
  - "[[React_Context]]"
  - "[[JS_Primitive_Methods]]"
  - "[[TS_ImportType]]"
---
# TS_Generics — 제네릭

> [!info] 
> 제네릭 = "타입을 변수처럼 나중에 결정하는 것". 
> `Array<string>`, `Promise<User>`, `useState<number>` 처럼 `<>` 안에 타입을 넣는 패턴이다.

---

# 왜 필요한가 ⭐️⭐️⭐️⭐️

```typescript
// ❌ any — 타입 정보가 사라짐
function identity(arg: any): any {
  return arg;
}
const result = identity('hello');
// result 타입: any — string인지 number인지 모름

// ✅ 제네릭 — 타입 정보 유지
function identity<T>(arg: T): T {
  return arg;
}
const result = identity('hello');
// result 타입: string — 입력 타입이 그대로 출력으로 나옴
```

```txt
제네릭의 핵심:
  "어떤 타입이 들어오든 그 타입 그대로 돌려준다"를 표현
  T는 타입 변수 — 함수 호출 시 실제 타입으로 채워짐
  identity('hello')  → T = string
  identity(42)       → T = number
  identity({})       → T = {}
```

---

# 기본 문법 ⭐️⭐️⭐️⭐️

```typescript
// 함수
function first<T>(arr: T[]): T | undefined {
  return arr[0];
}
first([1, 2, 3])   // T = number, 반환: number | undefined
first(['a', 'b'])  // T = string, 반환: string | undefined

// 화살표 함수
const wrap = <T>(value: T): { value: T } => ({ value });

// 타입 명시 (추론이 안 될 때)
first<string>([])
```

```txt
T, U, K, V — 관례적인 타입 파라미터 이름:
  T        일반 타입 (Type)
  U        두 번째 타입
  K        키 타입 (Key)
  V        값 타입 (Value)
  TResult  명확한 의미를 표현할 때 긴 이름도 OK
```

---

# 제약 조건 — extends ⭐️⭐️⭐️⭐️

```typescript
// T는 어떤 타입이든 — length가 없을 수 있음
function logLength<T>(arg: T) {
  console.log(arg.length);  // ❌ T에 length가 있다는 보장 없음
}

// T는 { length: number }를 가져야 함
function logLength<T extends { length: number }>(arg: T) {
  console.log(arg.length);  // ✅
}

logLength('hello');  // string은 length 있음 ✅
logLength([1, 2]);   // array는 length 있음 ✅
logLength(42);       // ❌ number에는 length 없음
```

```typescript
// 실전 — 배열 요소를 반환
function getProperty<T, K extends keyof T>(obj: T, key: K): T[K] {
  return obj[key];
}

const user = { id: 1, name: 'Tom' };
getProperty(user, 'name');  // ✅ 'name'은 user의 키
getProperty(user, 'age');   // ❌ 'age'는 user에 없음
```

---

# 기본값 — 제네릭 기본 타입 ⭐️⭐️

```typescript
// T의 기본값을 string으로
function createPair<T = string>(first: T, second: T): [T, T] {
  return [first, second];
}

createPair('a', 'b')    // T = string (추론)
createPair(1, 2)        // T = number (추론)
createPair()            // T = string (기본값 사용)
```

---

# React에서 제네릭 ⭐️⭐️⭐️⭐️

```typescript
// useState — 타입 명시
const [count, setCount]   = useState<number>(0);
const [user, setUser]     = useState<User | null>(null);
const [items, setItems]   = useState<string[]>([]);

// useRef — DOM 타입
const inputRef = useRef<HTMLInputElement>(null);
const divRef   = useRef<HTMLDivElement>(null);

// Promise — 응답 타입
const user = await fetch('/api/user').then(r => r.json() as Promise<User>);

// 이미 담긴 제네릭
Array<string>      // string[]과 동일
Map<string, User>  // 키: string, 값: User
Set<number>
```

---

# 함수 타입 — (param: T) => void ⭐️⭐️⭐️⭐️

```typescript
// 함수를 타입으로 표현
type Handler  = () => void;                            // 파라미터 없음
type OnChange = (value: string) => void;               // 파라미터 있음
type Fetcher  = (id: string) => Promise<User>;         // 비동기 반환

// React Props에서 콜백
type Props = {
  value:    string;
  onChange: (value: string) => void;  // 필수 콜백
  onSave?:  () => void;               // 선택 콜백
};
```

```txt
void vs Promise<T>:
  () => void           반환값을 신경 안 씀 (이벤트 핸들러)
  () => Promise<User>  비동기 함수 — await 하면 User
  () => Promise<void>  비동기지만 반환값 없음
```

---

# 함수를 인자로 받기 — 고차 함수 ⭐️⭐️⭐️⭐️

```typescript
// count 파라미터 자체가 함수 타입
const periodCounts = async (
  count: (gte?: Date) => Promise<number>
) => {
  const [week, month, total] = await Promise.all([
    count(startOfWeek),   // 이번 주 이후 count
    count(startOfMonth),  // 이번 달 이후 count
    count(),              // 전체 count
  ]);
  return { week, month, total };
};

// 사용 — 어떤 모델이든 주입 가능
const postStats = await periodCounts(
  (gte) => prisma.post.count({ where: { createdAt: gte ? { gte } : undefined } })
);
const commentStats = await periodCounts(
  (gte) => prisma.comment.count({ where: { createdAt: gte ? { gte } : undefined } })
);
```

```txt
왜 이렇게 쓰는가:
  "기간별(week/month/total) 집계 로직"은 모든 모델에서 동일
  어떤 모델을 셀지만 다름 → 쿼리 함수를 주입받아 재사용

  periodCounts는 날짜 계산 + Promise.all 병렬 실행만 담당
  실제 쿼리는 주입된 count 함수가 담당
```

---

# ReactNode — 렌더링 가능한 모든 것 ⭐️⭐️⭐️⭐️

```typescript
import { type ReactNode } from 'react';

// children prop에 가장 자주 씀
type Props = {
  children: ReactNode;   // JSX, string, number, null, undefined 전부 허용
  label?:   ReactNode;
};
```

```typescript
// ReactNode vs JSX.Element
const a: JSX.Element = <div />;      // ✅
const b: JSX.Element = 'hello';      // ❌ 문자열 안 됨
const c: JSX.Element = null;         // ❌ null 안 됨

const d: ReactNode = <div />;        // ✅
const e: ReactNode = 'hello';        // ✅
const f: ReactNode = null;           // ✅
```

```txt
언제 뭘 쓰는가:
  children?: ReactNode          가장 넓음 — 뭐든 받을 때
  icon?: React.ReactElement     JSX 엘리먼트만 (문자열, null 제외)
  renderHeader?: () => JSX.Element  반드시 JSX를 반환하는 함수
```

---

# keyof — 타입의 키 유니온 ⭐️⭐️⭐️⭐️

```typescript
type User = { id: string; name: string; age: number };

type UserKey = keyof User;
// → 'id' | 'name' | 'age'

// 실전 — 타입 안전한 설정 배열
const DISPLAY_CHIPS: { key: keyof LyricDecorDisplay; label: string }[] = [
  { key: 'lyrics', label: '가사' },
  { key: 'mood',   label: '감정' },
  // key에 오타 넣으면 컴파일 에러
];
```

---

# `Partial<T>` + 스프레드 패치 패턴 ⭐️⭐️⭐️⭐️

```typescript
// 전체 중 일부 필드만 업데이트
const patch = (partial: Partial<ApiCustomization>) => {
  onChange({ ...normalizeValue(value), ...partial });
};

// 특정 필드만 바꾸기
patch({ display: { ...d, [key]: on } });
```

```txt
Partial<T>:
  T의 모든 필드를 optional로 → "일부 필드만 보내면 됨"
  PATCH 업데이트 파라미터 타입으로 자주 씀

{ ...기존값, ...partial }:
  partial의 필드가 기존값을 덮어씀
  나머지 필드는 그대로 유지
```

---

# readonly T[] — 읽기 전용 배열 파라미터 ⭐️⭐️⭐️

```typescript
// T[]     → 함수 안에서 수정 가능
// readonly T[] → 수정 불가 (더 넓은 타입 허용)
function process(items: readonly string[]) {
  items.push('x');  // ❌ readonly라 push 불가
  return items[0];  // ✅ 읽기만 가능
}

process(['a', 'b'])           // ✅ string[]
process(['a', 'b'] as const)  // ✅ readonly string[]

// as const 배열도 받을 수 있게 readonly로 선언하는 것이 더 유연
```

---

# size variant prop 패턴 ⭐️⭐️⭐️

```typescript
// string으로 받으면 어떤 값이든 통과
function Button({ size }: { size: string }) { }

// 리터럴 유니온으로 제한
function Button({
  size = 'md',
}: {
  size?: 'sm' | 'md' | 'lg';
}) {
  const cls = size === 'sm' ? 'px-2 py-1' : size === 'lg' ? 'px-6 py-3' : 'px-4 py-2';
  // ...
}

// Record로 매핑
const SIZE_CLASS: Record<'sm' | 'md' | 'lg', string> = {
  sm: 'px-2 py-1',
  md: 'px-4 py-2',
  lg: 'px-6 py-3',
};
```

---

# 스프레드 → 타입 느슨해짐 → 콜백 추론 실패 ⭐️⭐️⭐️

```typescript
// ❌ 스프레드가 섞이면 콜백 타입 추론 끊어짐
const options = {
  ...defaults,
  events: {
    onError: (e) => { console.log(e.data); }  // e 타입: any
  },
};

// ✅ 파라미터 직접 명시
const options = {
  ...defaults,
  events: {
    onError: (e: { data: number }) => { console.log(e.data); }
  },
};

// ✅ 또는 satisfies로 연결
const options = {
  ...defaults,
  events: { onError: (e) => { ... } },
} satisfies PlayerOptions;
```

```txt
이유:
  { ...something, key: value } → TS가 "이 객체에 뭐가 더 붙을지 모름"
  → 넓은 타입으로 추론 → 콜백 파라미터 타입 끊어짐
  → strict 모드에서 "Parameter 'e' implicitly has an 'any' type" 에러

satisfies:
  검증(타입 맞는지 체크) + 리터럴 타입 유지
  as는 강요, satisfies는 검증
```

---

# 한눈에

```txt
기본:
  <T>           타입 변수 선언
  <T>(arg: T)   호출 시 T가 실제 타입으로 채워짐
  추론 → 대부분 명시 불필요, 안 될 때만 fn<string>(...)

extends:
  <T extends string>        T는 string만
  <T extends { id: string }> T는 id 필드가 있어야
  <K extends keyof T>       K는 T의 키 중 하나

React:
  useState<User | null>(null)
  useRef<HTMLInputElement>(null)

함수 타입:
  (param: T) => void    파라미터 → 반환 없음
  () => Promise<User>   비동기 반환

고차 함수:
  fn: (arg: T) => U     함수를 인자로 받음

ReactNode:
  children: ReactNode   JSX, string, null 전부 허용

keyof:
  keyof T → T의 키 유니온

Partial 패치:
  { ...base, ...partial }  일부 필드만 덮어쓰기
```