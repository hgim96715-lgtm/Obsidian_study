---
aliases:
  - generics
  - T
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
---
# TS_Generics — `<T>`는 호출할 때 정해지는 타입 변수

> [!info] 
> `<T>`는 함수/타입을 "쓸 때" 채워지는 타입 자리표시자
> `any`처럼 타입 정보를 버리지 않으면서도, 함수 하나로 여러 타입에 똑같이 동작하게 해준다.

---

# 왜 필요한가 — any와 다른 점 ⭐️⭐️⭐️⭐️

```typescript
// any로 쓰면 — 타입 정보가 사라짐
async function fetchAny(path: string): Promise<any> {
  const res = await fetch(path);
  return res.json();
}
const data = await fetchAny('/posts'); // data의 타입: any (자동완성도 없음)

// 제네릭으로 쓰면 — 타입 정보가 유지됨
async function fetchAPI<T>(path: string): Promise<T> {
  const res = await fetch(path);
  return res.json() as Promise<T>;
}
const data = await fetchAPI<ApiPost[]>('/posts'); // data의 타입: ApiPost[] (자동완성 됨)
```

```txt
any:      "타입을 모른다" → 이후 그 값에 어떤 코드를 써도 TS가 검사를 안 함
제네릭<T>: "지금은 모르지만, 호출하는 쪽이 알려줄 것이다"
           → 호출 시점에 T가 정해지고 이후로는 정확히 그 타입으로 추적됨
```

---

# `<T>`가 정해지는 시점 — 호출할 때 ⭐️⭐️⭐️

```typescript
function identity<T>(value: T): T {
  return value;
}

identity<string>('hello');  // T = string으로 직접 지정
identity(42);               // 인자(42)를 보고 TS가 T = number로 자동 추론
```

```txt
<T>를 직접 적을 수도(identity<string>(...)) 있고, 생략해서 TS가 인자로부터 추론하게 둘 수도 있음
fetchAPI<ApiPost[]>('/posts')처럼 인자만 보고는 T를 추론할 거리가 없으면 직접 적어줘야 함
(path는 string이라 T와 무관 → 반환 타입에만 쓰인 T는 호출하는 쪽이 직접 알려줘야 함)
```

---

# 함수 타입 표기법 ⭐️⭐️⭐️⭐️

```txt
TypeScript에서 "함수 그 자체"를 타입으로 표현하는 방법
함수를 인자로 받거나(콜백), props로 내려줄 때 반드시 알아야 하는 문법
```

## 기본 형태

```typescript
// (매개변수: 타입) => 반환타입
() => void                        // 인자 없음, 반환값 없음
() => Promise<void>               // 인자 없음, async 함수
(id: number) => void              // 인자 하나, 반환값 없음
(id: number) => Promise<User>     // 인자 하나, Promise 반환
(a: string, b: number) => boolean // 인자 여러 개
```

```txt
void vs Promise<void>:
  void           동기 함수, 반환값 없음
  Promise<void>  async 함수 또는 Promise를 반환하는 함수, 반환값 없음
  → 이벤트 핸들러(onClick 등)는 void
  → API 호출을 포함하는 함수는 Promise<void>
```

## 실전 사용 — 함수를 인자로 받을 때 ⭐️⭐️⭐️⭐️

```typescript
// fn: () => Promise<unknown>
//   unknown = "fn이 어떤 타입을 resolve하든 상관없음, 어차피 await만 함"
//   Promise<void>로 두면 반환값 있는 함수(Promise<ApiFriendship> 등)를 넣을 때 TS 에러
const runAction = async (fn: () => Promise<unknown>) => {
  setActing(true);
  setError('');
  try {
    await fn();          // 반환값은 사용 안 함 — unknown이라 안전
    await loadRelation();
  } catch (err) {
    setError(err instanceof Error ? err.message : '요청에 실패했어요.');
  } finally {
    setActing(false);
  }
};

// void runAction(...): onClick에서 runAction의 반환 Promise를 버리는 연산자
<button onClick={() => void runAction(() => likePost(id))}>좋아요</button>
<button onClick={() => void runAction(() => createFriendRequest(id))}>친구 요청</button>
```

```txt
⚠️ void가 두 곳에 등장하는데 완전히 다른 얘기:

  onClick에서 void runAction(...)
    → runAction이 반환하는 Promise를 "의도적으로 버리는" JS 연산자
    → floating promise 경고 방지 — 타입이 아니라 연산자

  fn: () => Promise<unknown>
    → unknown = "fn이 어떤 타입을 resolve하든 상관없음"
    → runAction은 fn()의 결과를 쓰지 않으니 가장 넓은 unknown이 맞음
    → Promise<void>로 두면 Promise<ApiFriendship>처럼 반환값 있는 함수를 넣을 때 TS 에러

비교:
  Promise<void>     반환값이 없는 Promise
  Promise<unknown>  반환값이 뭐든 상관없는 Promise — 가장 넓은 타입
  void runAction()  Promise를 버리는 JS 연산자 — 타입이 아님

() => likePost(id)를 넘기는 이유:
  likePost(id)라고 쓰면 그 자리에서 즉시 실행됨
  () => likePost(id)라고 쓰면 "나중에 호출될 함수"가 됨
  → runAction 안의 await fn()이 실행될 때 비로소 likePost(id)가 호출됨

패턴 자체 설명 → [[JS_Promise]] "async 래퍼 패턴" 참고
```

## 제네릭과 함수 타입 조합 ⭐️⭐️⭐️

```typescript
// 반환값이 있는 버전 — <T>로 반환 타입을 유연하게
const runActionWithResult = async <T>(fn: () => Promise<T>): Promise<T | null> => {
  try {
    return await fn();
  } catch {
    return null;
  }
};

// 사용
const post = await runActionWithResult<Post>(() => fetchPost(id));  // T = Post
const list = await runActionWithResult<Post[]>(() => fetchPosts()); // T = Post[]
```

## React Props에서 함수 타입 ⭐️⭐️⭐️

```typescript
type ButtonProps = {
  label:    string;
  onClick:  () => void;                   // 동기 핸들러
  onSubmit: (value: string) => void;      // 인자 있는 핸들러
  onLoad?:  () => Promise<void>;          // optional async 핸들러
};

// 컴포넌트에서 사용
function Button({ label, onClick }: ButtonProps) {
  return <button onClick={onClick}>{label}</button>;
}
```

## 함수 타입 별칭 — 반복될 때 ⭐️

```typescript
// 매번 (fn: () => Promise<void>) 타이핑이 번거로울 때
type AsyncAction = () => Promise<void>;
type AsyncCallback<T> = () => Promise<T>;

const runAction = async (fn: AsyncAction) => { ... };
const runActionWithResult = async <T>(fn: AsyncCallback<T>): Promise<T | null> => { ... };
```

---

# 실전 예시 — `fetchAPI<T>` ⭐️⭐️⭐️

```typescript
async function fetchAPI<T>(path: string, init?: RequestInit): Promise<T> {
  const res = await fetch(`${getApiBaseUrl()}${path}`, init);
  await throwIfNotOk(res, path);
  return res.json() as Promise<T>;
}

const posts = await fetchAPI<ApiPost[]>('/posts');     // T = ApiPost[]
const user  = await fetchAPI<ApiAuthResponse>('/me');  // T = ApiAuthResponse
```

```txt
같은 함수 하나(fetchAPI)가 어떤 엔드포인트를 호출하든 재사용됨
"로직은 같은데 다루는 타입만 매번 다른" 함수를 중복 없이 작성 → 이게 제네릭이 풀어주는 문제
```

---

# 실전 예시 — `useRef<T>` ⭐️⭐️⭐️

```typescript
const rootRef  = useRef<HTMLDivElement>(null); // T = HTMLDivElement
const countRef = useRef<number>(0);            // T = number
```

```txt
useRef도 내부 동작(값을 박스에 담아 리렌더 없이 유지)은 항상 같고,
"그 박스 안에 뭐가 들어가는지"만 호출할 때 T로 결정됨 — fetchAPI<T>와 같은 발상
→ [[React_useRef]] 참고
```

---

# 제네릭에 제약 걸기 — extends ⭐️

```typescript
function getId<T extends { id: string }>(item: T): string {
  return item.id;
}
```

```txt
T extends { id: string } — "T가 뭐든 상관없지만, 최소한 id: string 필드는 있어야 한다"는 제약
이 제약이 없으면 item.id를 쓸 수 없음 (T가 아무 타입이나 될 수 있어서)
```

---

# `readonly T[]` — 읽기 전용 배열 파라미터 ⭐️⭐️⭐️⭐️

```typescript
// readonly T[] — "이 함수는 배열을 읽기만 하고, 절대 바꾸지 않는다"는 선언
function pickOne<T>(items: readonly T[]): T {
  return items[Math.floor(Math.random() * items.length)];
}
```

```typescript
// 호출 — const 배열도, mutable 배열도 모두 넣을 수 있음
const WISHES = ['행복하세요.', '좋은 하루 되세요.'] as const;
pickOne(WISHES);          // ✅ readonly (as const) 배열

const arr = ['a', 'b'];
pickOne(arr);             // ✅ 일반 배열도 가능 (readonly로 넣으면 더 넓게 받음)
```

```txt
readonly T[] vs T[]:

  파라미터 타입이 T[] 이면:
    일반 배열만 받을 수 있음
    readonly 배열('as const'로 만든 것)을 넣으면 TS 에러

  파라미터 타입이 readonly T[] 이면:
    일반 배열 + readonly 배열 모두 받을 수 있음 (더 넓게 받음)
    함수 안에서 .push() .pop() .splice() 같은 변경 메서드 호출 불가
    → "이 함수는 배열을 바꾸지 않는다"는 약속을 타입으로 강제

  → 배열을 읽기만 하는 함수라면 readonly T[]를 쓰는 것이 더 정확하고 유연함
```

## as const 배열과 조합 ⭐️⭐️⭐️

```typescript
// as const — 배열을 리터럴 타입의 readonly 튜플로 만듦
const MORNING_WISHES = [
  '오늘도 활기차게 시작해요.',
  '좋은 하루 되세요.',
  '오늘 하루도 잘 부탁드려요.',
] as const;

// MORNING_WISHES 타입:
// readonly ['오늘도 활기차게 시작해요.', '좋은 하루 되세요.', '오늘 하루도 잘 부탁드려요.']

// pickOne 반환 타입: 세 문자열 리터럴 유니온
// → '오늘도 활기차게 시작해요.' | '좋은 하루 되세요.' | '오늘 하루도 잘 부탁드려요.'
const wish = pickOne(MORNING_WISHES);
```

```txt
as const의 효과:
  배열이 readonly 튜플로 변환됨 → 요소 수정, 추가, 삭제 불가
  요소 타입이 string에서 각 문자열 리터럴 타입으로 좁혀짐

pickOne<T>에서 T 추론:
  items: readonly T[]에 MORNING_WISHES를 넣으면
  T = '오늘도...' | '좋은 하루...' | '오늘 하루...' 로 추론됨
  반환 타입도 자동으로 그 유니온 타입

실전에서 자주 쓰는 패턴:
  const ITEMS = ['a', 'b', 'c'] as const;  // 상수 배열 정의
  type Item = typeof ITEMS[number];          // 'a' | 'b' | 'c' 타입 추출
  function pickOne<T>(items: readonly T[]): T { ... }  // 꺼내기
```

---

# ReadonlySet / ReadonlyMap / ReadonlyArray ⭐️⭐️⭐️

```typescript
// ReadonlySet — 읽기 전용 Set
const ids: ReadonlySet<string> = new Set(['a', 'b']);
ids.has('a');    // ✅ 읽기는 가능
ids.add('c');    // ❌ TS 에러 — 수정 메서드 없음
ids.delete('a'); // ❌ TS 에러

// ReadonlyMap
const map: ReadonlyMap<string, number> = new Map();
map.get('key');  // ✅
map.set('k', 1); // ❌

// readonly 배열
const arr: ReadonlyArray<string> = ['a', 'b'];
// 또는 축약형
const arr: readonly string[] = ['a', 'b'];
arr[0];          // ✅
arr.push('c');   // ❌
```

```txt
언제 쓰는가:
  Context value에서 "이 Set은 Provider 안에서만 바꾸고, 읽는 쪽은 읽기만 해라"
  → type FriendIdsContextValue = { ids: ReadonlySet<string>; reload: () => void }

  상태를 외부에 노출할 때 수정 권한을 제한
  "이 값을 받은 쪽은 건드리지 마라"는 의도를 타입으로 표현

  ⚠️ 타입 수준의 제약 — 런타임에 실제로 막히는 건 아님
     setIds(원래Set)처럼 Provider 안에서는 여전히 수정 가능
```

---

# 한눈에

| 개념                                          | 핵심                                                        |
| ------------------------------------------- | --------------------------------------------------------- |
| `<T>`                                       | 함수/타입을 호출·사용하는 시점에 정해지는 타입 변수                             |
| `any`와의 차이                                  | any는 타입 정보를 버림, 제네릭은 타입 정보를 끝까지 추적함                       |
| 추론                                          | 인자로부터 T를 알 수 있으면 생략 가능, 알 수 없으면 직접 명시                     |
| 쓰는 이유                                       | "로직은 같은데 다루는 타입만 다른" 함수를 중복 없이 작성                         |
| `T extends 조건`                              | T가 최소한 갖춰야 하는 모양을 제약                                      |
| `() => void`                                | 동기 함수 타입, 반환값 없음                                          |
| `() => Promise<void>`                       | async 함수 타입, 반환값 없음                                       |
| `() => Promise<unknown>`                    | 반환값이 뭐든 상관없는 async 함수 — 결과를 쓰지 않는 콜백에 사용                  |
| `(fn: () => Promise<unknown>) => {}`        | 함수를 인자로 받는 고차 함수 — async 래퍼 패턴                            |
| `void runAction()`                          | Promise를 버리는 JS 연산자 — 타입이 아님, onClick floating promise 방지 |
| `type AsyncAction = () => Promise<unknown>` | 반복되는 함수 타입을 별칭으로 추출                                       |