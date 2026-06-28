---
aliases: [generics, T]
tags:
  - TypeScript
related:
  - "[[00_JS_Ecosystem_HomePage]]"
  - "[[NextJS_API_Client]]"
  - "[[React_useRef]]"
  - "[[TS_TypeAssertion]]"
---

# TS_Generics — `<T>`는 호출할 때 정해지는 타입 변수

> [!info] 
>  `<T>`는 함수/타입을 "쓸 때" 채워지는 타입 자리표시자다. 
>  `any`처럼 타입 정보를 버리지 않으면서도, 함수 하나로 여러 타입에 똑같이 동작하게 해준다.

---

# 왜 필요한가 — any와 다른 점 ⭐️⭐️⭐️⭐️

```typescript
// any로 쓰면 — 타입 정보가 사라짐
async function fetchAny(path: string): Promise<any> {
  const res = await fetch(path);
  return res.json();
}
const data = await fetchAny('/posts'); // data의 타입: any (뭐든 다 됨, 자동완성도 없음)

// 제네릭으로 쓰면 — 타입 정보가 유지됨
async function fetchAPI<T>(path: string): Promise<T> {
  const res = await fetch(path);
  return res.json() as Promise<T>;
}
const data = await fetchAPI<ApiPost[]>('/posts'); // data의 타입: ApiPost[] (자동완성 됨)
```

```txt
any: "타입을 모른다" → 이후 그 값에 어떤 코드를 써도 TS가 검사를 안 함 (자유롭지만 위험)
제네릭<T>: "지금은 모르지만, 호출하는 쪽이 알려줄 것이다" → 호출 시점에 T가 정해지고
           그 이후로는 정확히 그 타입으로 추적됨 (자유로움 + 타입 안정성 둘 다)
```

---

# `<T>`가 정해지는 시점 — 호출할 때 ⭐️⭐️⭐️

```typescript
function identity<T>(value: T): T {
  return value;
}

identity<string>('hello');  // T = string으로 직접 지정
identity(42);                // T를 안 적어도, 인자(42)를 보고 TS가 T = number로 자동 추론
```

```txt
<T>를 직접 적을 수도(identity<string>(...)) 있고, 생략해서 TS가 인자로부터 추론하게 둘 수도 있음
(fetchAPI<ApiPost[]>('/posts')처럼 인자만 보고는 T를 추론할 거리가 없으면 직접 적어줘야 함 —
 path는 string이라 T와 무관하기 때문에, 반환 타입에만 쓰인 T는 호출하는 쪽이 직접 알려줘야 함)
```

---

# 실전 예시 — `fetchAPI<T>` ⭐️⭐️⭐️

```typescript
async function fetchAPI<T>(path: string, init?: RequestInit): Promise<T> {
  const res = await fetch(`${getApiBaseUrl()}${path}`, init);
  await throwIfNotOk(res, path);
  return res.json() as Promise<T>;
}

const posts = await fetchAPI<ApiPost[]>('/posts');      // T = ApiPost[]
const user  = await fetchAPI<ApiAuthResponse>('/me');    // T = ApiAuthResponse
```

```txt
같은 함수 하나(fetchAPI)가 어떤 엔드포인트를 호출하든 재사용됨 —
함수 내부 로직(fetch→ok 확인→json)은 항상 같고, "결과를 무슨 타입으로 볼지"만 호출할 때마다 다름
→ 이게 제네릭이 풀어주는 문제: "로직은 같은데 다루는 타입만 매번 다른" 함수를 중복 없이 작성
```

---

# 실전 예시 — `useRef<T>` ⭐️⭐️⭐️

```typescript
const rootRef = useRef<HTMLDivElement>(null);   // T = HTMLDivElement → .current: HTMLDivElement | null
const countRef = useRef<number>(0);             // T = number → .current: number
```

```txt
useRef도 내부 동작(값을 박스에 담아 리렌더 없이 유지)은 항상 같고,
"그 박스 안에 뭐가 들어가는지"만 호출할 때 T로 결정됨 — fetchAPI<T>와 같은 발상
자세한 useRef 자체 설명은 [[React_useRef]] 참고
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
이 제약이 없으면 item.id 자체를 쓸 수 없음(T가 정말 아무 타입이나 될 수 있어서 id가 있다는 보장이 없음)
```

---

# 한눈에

|개념|핵심|
|---|---|
|`<T>`|함수/타입을 호출·사용하는 시점에 정해지는 타입 변수|
|`any`와의 차이|any는 타입 정보를 버림, 제네릭은 타입 정보를 끝까지 추적함|
|추론|인자로부터 T를 알 수 있으면 생략 가능, 알 수 없으면(반환 타입에만 쓰일 때 등) 직접 명시|
|쓰는 이유|"로직은 같은데 다루는 타입만 다른" 함수/클래스를 중복 없이 작성|
|`T extends 조건`|T가 최소한 갖춰야 하는 모양을 제약|