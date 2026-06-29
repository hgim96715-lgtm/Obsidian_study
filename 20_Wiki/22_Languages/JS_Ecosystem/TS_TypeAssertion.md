---
aliases: [as, as const, type assertion]
tags:
  - TypeScript
related:
  - "[[00_JS_Ecosystem_HomePage]]"
  - "[[NextJS_API_Client]]"
  - "[[TS_Generics]]"
---

# TS_TypeAssertion — as 키워드

> [!info] 
>  `as`는 "이 값의 타입을 내가 더 잘 안다"고 컴파일러에게 직접 알려주는 것이다. 
>  검증이 아니라 단언이라서, 실제 런타임 값이 단언한 타입과 다르면 그 자리에서는 에러가 안 나고 나중에 엉뚱한 곳에서 터진다.

---

# 왜 필요한가 ⭐️⭐️⭐️

```typescript
const data = await res.json();        // 타입: any (res.json()은 항상 any를 반환)
const posts = data as ApiPost[];       // "이건 ApiPost[] 모양일 거야" 라고 직접 알려줌
```

```txt
TS가 타입을 추론할 수 없는 경계(JSON 파싱 결과, DOM API 반환값 등)에서
"이건 실제로 이런 모양일 것"이라는 사실을 컴파일러에게 전달하는 용도

이 경계를 넘는 순간부터는 TS가 더 이상 검증을 안 해줌 — 개발자가 책임지고 정확히 단언해야 함
```

---

# 검증이 아니라 단언 — 가장 중요한 차이 ⭐️⭐️⭐️⭐️

```typescript
const data = (await res.json()) as ApiPost[];
// 실제 응답이 ApiPost[] 모양이 아니어도(예: 서버가 에러 객체를 돌려줬어도)
// 이 줄에서는 아무 에러도 안 남 — TS는 "그렇다고 치자"고 그냥 믿어버림

return mapPosts(data, userId);
// 한참 뒤, data.reactions를 읽으려는 순간에야 실제로 undefined라서 런타임 에러
```

```txt
as는 "타입 체크를 통과시키는 것"이 아니라 "타입 체크를 건너뛰게 하는 것"임
→ 잘못 단언하면 컴파일은 깨끗하게 통과하지만, 런타임에 전혀 다른 곳에서 에러가 터짐
  (단언한 그 줄이 아니라, 그 값을 실제로 "잘못된 모양으로" 사용하는 더 나중 줄에서 터지는 게 특징)
```

|구분|하는 일|런타임 검증|
|---|---|---|
|`: Type` (타입 선언)|이 값이 정말 그 타입과 호환되는지 컴파일 타임에 검사|TS가 대신 확인해줌|
|`as Type` (타입 단언)|"이 값을 그 타입으로 취급해라"고 강제로 지시|전혀 안 함 — 개발자 책임|

```txt
실전 예시 — CSS 커스텀 속성을 style prop에 넣을 때(as CSSProperties)도 같은 이유로 필요함
자세한 내용은 [[React_CSSProperties]] 참고

실전 예시 — Prisma의 Json 필드에 DTO 값을 넣을 때(as Prisma.InputJsonValue)도 자주 필요함
옵셔널 속성이 있는 객체 타입이 왜 구조적으로 막히는지는 [[NestJS_Prisma]]의 "Json 필드" 참고
```

---

# 안전하게 쓰는 법 — 단언이 정당화되는 경계 ⭐️⭐️⭐️

```txt
as를 써도 되는 경우: "이 값의 실제 모양을 내가 다른 방법으로 이미 확신할 수 있을 때"
  - 백엔드 응답 스펙을 직접 통제하고 있고, Swagger로 실제 JSON을 확인했을 때 (apiTypes로 받기)
  - DOM API가 특정 상황에서 항상 특정 타입을 준다고 명세상 보장될 때 (e.target as Node)

as를 쓰면 위험한 경우: "내가 확신이 없는데 타입 에러를 그냥 없애려고" 쓰는 경우
  → 이럴 땐 as 대신 실제로 값을 검사하는 type guard 함수나 zod 같은 검증 라이브러리를 쓰는 게 맞음
    (타입을 우기는 게 아니라 진짜로 확인하는 것 — as와는 다른 해결책)
```

---

# as const — 다른 용도의 as ⭐️⭐️

```typescript
const role = 'admin';          // 타입: string (넓혀짐)
const role = 'admin' as const; // 타입: 'admin' (리터럴 그대로 고정)

const ANONYMOUS_AUTHOR = { id: '', nickname: '익명' } as const;
// 이 객체의 각 필드가 string이 아니라 정확히 그 리터럴 값으로 고정됨
```

```txt
as const는 "이 값을 다른 타입으로 우기는" 일반 as와 성격이 다름 —
"이 값을 있는 그대로, 더 넓혀지지 않게 고정해라"는 뜻 (타입을 좁히는 용도)
일반 단언(as ApiPost[])과 같은 키워드를 쓸 뿐, 목적은 반대 방향임
```

---

# 한눈에

| 표현               | 의미                                                       | 위험도                    |
| ---------------- | -------------------------------------------------------- | ---------------------- |
| `value as Type`  | "이건 Type이다"라고 강제로 단언 — 런타임 검증 없음                         | 틀리면 더 나중에 엉뚱한 곳에서 에러   |
| `value as const` | 이 리터럴 값을 넓히지 말고 그대로 고정                                   | 단언이라기보단 타입 좁히기, 비교적 안전 |
| 타입 단언이 필요한 흔한 경계 | `res.json()`(항상 any), DOM 이벤트(`e.target`은 `EventTarget`) |                        |
| 진짜 검증이 필요하면      | `as`가 아니라 type guard 함수 또는 zod 같은 런타임 검증 라이브러리           |                        |