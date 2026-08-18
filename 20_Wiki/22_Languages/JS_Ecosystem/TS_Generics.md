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
  - "[[NestJS_WebSocket]]"
  - "[[JS_Promise]]"
  - "[[React_AsyncUI]]"
  - "[[JS_Promise]]"
  - "[[React_Types]]"
---
# TS_Generics — 제네릭

>[!info]
>제네릭 = 타입을 나중에 결정할 수 있게 하는 것. 
>`<T>`를 선언하면 호출 시 타입이 결정된다.
> 대부분 자동 추론되지만 `useState<Set<string>>(() => new Set())`처럼 빈 객체·null이 초기값이면 명시해야 한다.
>  `Set<string>` · `Map<string, User>` · `Promise<User>` 같은 중첩 제네릭도 같은 원리.

---

# 제네릭이란 — 타입을 변수처럼 ⭐️⭐️⭐️⭐️

```txt
일반 변수:   값을 나중에 결정
  const x = 5;     // 숫자 5를 x에 저장
  const x = 'hi';  // 문자열 'hi'를 x에 저장

타입 변수:   타입을 나중에 결정
  function fn<T>  // T에 어떤 타입이 들어갈지는 호출 시 결정됨
  fn('hello')  → T = string
  fn(42)       → T = number
  fn({ id: 1}) → T = { id: number }
```

## any와의 차이 ⭐️⭐️⭐️⭐️

```typescript
// ❌ any — 타입 정보가 사라짐
function wrap(value: any): any { return { value }; }
const result = wrap('hello');
// result 타입: any → result.value 가 string인지 모름

// ✅ 제네릭 — 타입 정보 유지
function wrap<T>(value: T): { value: T } { return { value }; }
const result = wrap('hello');
// result 타입: { value: string } → result.value 가 string임을 앎
```

```txt
any는 "어떤 타입이든 받겠다, 대신 타입 정보는 포기"
제네릭은 "어떤 타입이든 받겠다, 그리고 그 타입 정보를 유지"

제네릭 없으면 → any로 처리 → 타입 자동완성 없음, 오타도 잡아줌 없음
제네릭 있으면 → T = string으로 결정 → string의 메서드 자동완성, 타입 검사 유지
```

---

# 기본 문법 ⭐️⭐️⭐️⭐️

```typescript
// 함수 선언 — <T>를 파라미터 목록 앞에 선언
function first<T>(arr: T[]): T | undefined {
  return arr[0];
}

// 화살표 함수
const first = <T>(arr: T[]): T | undefined => arr[0];
```

## 타입 추론 — 대부분 자동으로 ⭐️⭐️⭐️⭐️

```typescript
first([1, 2, 3])   // TypeScript가 T = number로 자동 추론
first(['a', 'b'])  // T = string으로 자동 추론
first([])          // T = never (빈 배열 — 타입 알 수 없음)
```

```txt
T를 직접 쓰는 경우는 드뭄:
  first<string>([])     // 빈 배열처럼 추론이 안 될 때만 명시

대부분의 제네릭 함수는:
  호출할 때 인자를 보고 TypeScript가 T를 자동으로 결정
  → 매번 first<string>(...) 처럼 명시하지 않아도 됨
```

## 타입 파라미터 이름 관례

```txt
T   일반 타입 (Type)
U   두 번째 타입
K   키 타입 (Key)
V   값 타입 (Value)

의미를 명확히 하고 싶으면 긴 이름도 OK:
  <TResult>, <TInput>, <TKey>
```

---

# 제약 조건 — extends ⭐️⭐️⭐️⭐️

```typescript
// T는 어떤 타입이든 — length가 없을 수 있음
function logLength<T>(arg: T) {
  arg.length;  // ❌ T에 length가 있다는 보장 없음
}

// T는 반드시 length 속성을 가져야 함
function logLength<T extends { length: number }>(arg: T) {
  arg.length;  // ✅ extends로 보장됨
}

logLength('hello');  // ✅ string은 length 있음
logLength([1, 2]);   // ✅ 배열은 length 있음
logLength(42);       // ❌ number에는 length 없음
```

```txt
extends 읽는 법:
  <T extends { length: number }>
  = "T는 어떤 타입이든 되는데, 최소한 { length: number }는 가져야 함"
  = T의 범위를 제한하는 조건
```

## keyof 조합 — 객체의 키를 타입으로 ⭐️⭐️⭐️

```typescript
// K extends keyof T = K는 T의 키 중 하나여야 함
function getProperty<T, K extends keyof T>(obj: T, key: K): T[K] {
  return obj[key];
}

const user = { id: 1, name: 'Tom', age: 30 };
getProperty(user, 'name');  // ✅ 반환 타입: string
getProperty(user, 'id');    // ✅ 반환 타입: number
getProperty(user, 'email'); // ❌ 'email'은 user의 키가 아님
```

```typescript
// keyof 단독 사용 — 타입의 키 유니온
type User = { id: string; name: string; age: number };
type UserKey = keyof User;  // 'id' | 'name' | 'age'

// 타입 안전한 설정 배열
const FIELDS: { key: keyof User; label: string }[] = [
  { key: 'name', label: '이름' },
  { key: 'age',  label: '나이' },
  // key에 오타 넣으면 컴파일 에러
];
```

---

# React에서 제네릭 ⭐️⭐️⭐️⭐️

```typescript
// useState — 초기값으로 추론 가능할 때는 생략 가능
const [count, setCount] = useState(0);          // T = number 자동 추론
const [name, setName]   = useState('');          // T = string 자동 추론

// 초기값이 null이거나 복잡한 경우 — 명시 필요
const [user, setUser]   = useState<User | null>(null);   // null만 보고는 User를 모름
const [items, setItems] = useState<Post[]>([]);           // 빈 배열만 보고는 Post를 모름

// Set · Map — 초기값이 비어있으면 반드시 명시
const [ids, setIds] = useState<Set<string>>(new Set());
//                             ↑ 없으면 Set<unknown>으로 추론됨
const [map, setMap] = useState<Map<string, User>>(new Map());

// 지연 초기화 + 타입 명시
const [expandedRootIds, setExpandedRootIds] = useState<Set<string>>(
  () => new Set(),
);
```

```txt
useState<Set<string>> 읽는 법:
  useState → React 훅 (제네릭 함수)
  <Set<string>> → "이 state의 타입은 Set<string>이야"
  () => new Set() → 초기값 (지연 초기화)

  Set<string> = string 값들의 집합
  new Set()만 전달하면 TypeScript가 Set<unknown>으로 추론
  → <Set<string>>을 명시해야 setIds에서 타입 안전하게 쓸 수 있음

중첩 제네릭:
  Set<string>       → string을 담는 Set
  Map<string, User> → string 키, User 값의 Map
  Array<Post>       → Post를 담는 배열 (= Post[])
  Promise<User>     → resolve되면 User를 반환하는 Promise
```

```typescript
// Promise — 응답 타입 명시
const user = await fetch('/api/user').then(r => r.json() as Promise<User>);

// 이미 내장된 제네릭 타입들
Array<string>      // = string[]
Map<string, User>  // 키: string, 값: User
Set<number>
Promise<User>      // resolve되면 User
```

---

# 함수 타입 — (param: T) => void ⭐️⭐️⭐️⭐️

```typescript
// 함수를 타입으로 표현
type Handler  = () => void;                     // 파라미터 없음, 반환 없음
type OnChange = (value: string) => void;        // string 받음, 반환 없음
type Fetcher  = (id: string) => Promise<User>;  // 비동기 반환

// React Props에서 콜백
type ButtonProps = {
  label:    string;
  onClick:  () => void;                         // 필수 콜백
  onChange?: (value: string) => void;           // 선택 콜백
};
```

```txt
void vs Promise<T>:
  () => void            반환값을 신경 안 씀 (이벤트 핸들러)
  () => Promise<User>   비동기 함수 — await 하면 User
  () => Promise<void>   비동기지만 반환값 없음
  () => Promise<unknown> 어떤 타입이든 받을 때 (래퍼 패턴) → [[JS_Promise]]
```

---

# 함수를 인자로 받기 — 고차 함수 ⭐️⭐️⭐️⭐️

```typescript
// count 파라미터 자체가 "함수 타입"
const periodCounts = async (
  count: (gte?: Date) => Promise<number>
) => {
  const [week, month, total] = await Promise.all([
    count(startOfWeek),
    count(startOfMonth),
    count(),
  ]);
  return { week, month, total };
};

// 사용 — 어떤 모델이든 주입 가능
const postStats = await periodCounts(
  (gte) => prisma.post.count({
    where: { createdAt: gte ? { gte } : undefined }
  })
);
const commentStats = await periodCounts(
  (gte) => prisma.comment.count({
    where: { createdAt: gte ? { gte } : undefined }
  })
);
```

```txt
왜 이렇게 쓰는가:
  "기간별(week/month/total) 집계 로직"은 모든 모델에서 동일
  어떤 모델을 셀지만 다름 → count 함수를 주입받아 재사용

  periodCounts는 날짜 계산 + Promise.all만 담당
  실제 DB 쿼리는 주입된 count 함수가 담당
```

---

# Partial\<T\> + 스프레드 패치 패턴 ⭐️⭐️⭐️

```typescript
// 전체 중 일부 필드만 업데이트
const patch = (partial: Partial<ApiCustomization>) => {
  onChange({ ...current, ...partial });
};

patch({ theme: 'dark' });              // theme만 바꿈
patch({ layout: 'grid', size: 'sm' }); // 두 필드 바꿈
```

```txt
Partial<T>:
  T의 모든 필드를 optional로 만듦 → "일부 필드만 보내면 됨"
  PATCH 업데이트 파라미터 타입으로 자주 씀

{ ...current, ...partial }:
  partial의 필드가 기존값을 덮어씀
  나머지 필드는 그대로 유지

Partial 포함 유틸리티 타입 전체 → [[TS_Utility_Types]]
```

---

# readonly T[] — 읽기 전용 배열 ⭐️⭐️⭐️

```typescript
function process(items: readonly string[]) {
  items.push('x');  // ❌ readonly라 push 불가
  return items[0];  // ✅ 읽기만 가능
}

process(['a', 'b'])           // ✅ string[]도 통과
process(['a', 'b'] as const)  // ✅ readonly string[]도 통과
```

```txt
T[] vs readonly T[]:
  T[]          함수 안에서 수정 가능
  readonly T[] 수정 불가 — 더 넓은 타입을 받을 수 있음

  as const로 만든 배열도 받고 싶을 때 readonly T[]로 선언하면 유연함
```


```txt
& = "A이고 동시에 B이기도 한" 타입 → 두 타입의 속성을 전부 가짐
| = "A이거나 B인" 타입 → 둘 중 하나면 됨
```

```typescript
type A = { name: string };
type B = { age: number };

type AB   = A & B;  // { name: string; age: number } — 둘 다 있어야 함
type AorB = A | B;  // name이 있거나 age가 있거나

const ok: AB   = { name: '홍길동', age: 30 };  // ✅
const fail: AB = { name: '홍길동' };             // ❌ age 없음
```

## 수정 못 하는 타입 확장 — 실전 패턴 ⭐️⭐️⭐️⭐️

```typescript
// socket.io의 Socket 타입은 직접 수정 불가
// data 필드가 기본적으로 any → 구체적인 타입을 추가하고 싶을 때

type AuthedSocket = Socket & { data: { userId?: string } };
//   ↑ 새 이름      ↑ 기존 타입  ↑ 추가할 타입

client.id           // ✅ Socket 기존 속성
client.emit(...)    // ✅ Socket 기존 메서드
client.data.userId  // ✅ 내가 추가한 타입 (string | undefined)
```

```txt
읽는 법:
  Socket & { data: { userId?: string } }
  = "Socket이 가진 모든 것을 가지면서, data.userId?: string 도 가진 타입"

& vs interface extends:
  // 방법 1 — & (즉석 선언, 인라인 가능)
  type AuthedSocket = Socket & { data: { userId?: string } };

  // 방법 2 — interface extends (정식 선언)
  interface AuthedSocket extends Socket { data: { userId?: string }; }

  제3자 라이브러리 타입 빠르게 확장 → &
  재사용·상속이 필요한 정식 선언 → extends
```

```typescript
// 여러 타입 합치기
type AdminRequest = Request & {
  user:      JwtPayload;
  adminMeta: AdminInfo;
};
```

```txt
실제 사용 위치 → [[NestJS_WebSocket]]
```

----
# never — 절대 존재할 수 없는 타입 ⭐️⭐️⭐️⭐️

```txt
never = "이 타입의 값은 절대 존재할 수 없다"

  void  = 반환값이 없음 (undefined 반환)
  never = 이 코드에 절대 도달하지 않음 (함수가 끝나지 않음)

언제 나타나는가:
  빈 배열의 요소 타입   → never
  도달 불가 코드        → never
  조건부 타입에서 제외   → T extends never ? 걸러냄
  throw만 하는 함수     → never 반환
```

```typescript
// 빈 배열 → never
first([])    // T = never (요소가 없으니 타입을 알 수 없음)

// 항상 throw → 절대 반환 안 함 → never
function fail(msg: string): never {
  throw new Error(msg);
}

// 도달 불가 코드
function check(x: string | number) {
  if (typeof x === 'string') return x.toUpperCase();
  if (typeof x === 'number') return x.toFixed();
  // 여기 x의 타입은 never — string도 number도 아닌 경우가 없음
}
```

---

# 조건부 타입 + never — 타입 가드 ⭐️⭐️⭐️⭐️

```typescript
// T extends U ? X : Y
// "T가 U에 할당 가능하면 X, 아니면 Y"

type IsString<T> = T extends string ? 'yes' : 'no';
type A = IsString<string>  // 'yes'
type B = IsString<number>  // 'no'
```

```typescript
// extends never — "이 타입이 never인지 확인"
type IsNever<T> = T extends never ? true : false;

// 실전 — 배열 요소 타입이 never면 (빈 배열/없는 필드) → 다른 타입 반환
type SafeElement<T> =
  T extends never
    ? never                          // never이면 never 그대로
    : { snapshot: T; closed: boolean }; // 값이 있으면 래핑
```

## 인덱스드 액세스 + [number] + 조건부 타입 조합 ⭐️⭐️⭐️⭐️

```typescript
// 실전 예
apiFetch<
  CafeHallResponse['tables'][number] extends never
    ? never
    : { snapshot: CafeTableSnapshot; cafeJustClosed: boolean }
>

// 분해:
// ① CafeHallResponse['tables']
//    → CafeHallResponse 타입에서 tables 필드의 타입 꺼내기
//    → 예: CafeTable[]

// ② CafeHallResponse['tables'][number]
//    → 배열 타입에서 요소 타입 꺼내기
//    → CafeTable[number] = CafeTable (배열의 한 요소)
//    → 배열이 never[]이면 never

// ③ extends never ? never : { snapshot: ...; cafeJustClosed: boolean }
//    → tables가 비어있거나(never) → never 반환 (API 타입 없음)
//    → tables에 실제 타입 있으면 → 실제 응답 타입 사용
```

```typescript
// [number] — 배열 요소 타입 꺼내기
type Items = string[];
type Item = Items[number];  // string
//                 ↑ "배열의 인덱스 타입(number)으로 접근" = 요소 타입

type Users = { id: string; name: string }[];
type User = Users[number];  // { id: string; name: string }

// 0, 1, 2... 특정 인덱스로도 가능
type Tuple = [string, number, boolean];
type First  = Tuple[0];  // string
type Second = Tuple[1];  // number
```

```txt
T[number] 읽는 법:
  "이 배열 타입 T에서 number 인덱스로 접근했을 때의 타입"
  = 배열의 요소 타입

T['fieldName'] vs T[number]:
  T['fieldName'] → 객체 필드 타입 꺼내기
  T[number]      → 배열 요소 타입 꺼내기

조건부 타입이 필요한 이유:
  API가 items: [] 처럼 빈 배열을 반환할 수 있음
  [number]로 꺼내면 never → 응답 타입도 never
  extends never 체크 → never이면 안전하게 never 반환
  → 호출하는 쪽에서 never 타입을 인지하고 처리 가능

→ [[TS_Type_Guards]] never 섹션
→ [[TS_Utility_Types]] Awaited + ReturnType 조합
```
