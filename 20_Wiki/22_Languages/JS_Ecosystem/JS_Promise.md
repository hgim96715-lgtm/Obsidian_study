---
aliases:
  - Promise
  - async/await
  - Promise.al
  - then
tags:
  - JavaScript
related:
  - "[[00_JS_Ecosystem_HomePage]]"
  - "[[JS_Array_Methods]]"
  - "[[JS_Loops_Conditionals]]"
  - "[[JS_Operators]]"
---

# JS_Promise — 비동기 처리의 기본 단위

> [!info] 
> Promise는 "지금은 모르지만 나중에 결과가 나올 값"을 표현하는 객체다. 
> pending(대기) → fulfilled(성공) 또는 rejected(실패) 중 하나로 끝나는 상태 머신이고, async/await는 이 Promise를 다루는 문법을 더 읽기 쉽게 만들어준 것일 뿐이다.

---

# Promise란 — 세 가지 상태 ⭐️⭐️⭐️⭐️

```txt
pending    아직 결과가 안 나온 상태 (진행 중)
fulfilled  성공적으로 끝남 — 결과값을 가짐
rejected   실패로 끝남 — 에러를 가짐

한 번 fulfilled나 rejected가 되면 그 뒤로 다시 안 바뀜(한 번만 결정되는 상태)
```

```javascript
const promise = fetch('/api/data'); // 이 시점엔 아직 pending — 응답이 안 왔으니까
```

---

# .then() / .catch() / .finally() — Promise를 다루는 기본 방법 ⭐️⭐️⭐️⭐️

```javascript
fetch('/api/data')
  .then((res) => res.json())
  .then((data) => console.log(data))
  .catch((err) => console.error(err))
  .finally(() => console.log('끝'));
```

|메서드|언제 실행되나|
|---|---|
|`.then(fn)`|Promise가 fulfilled(성공)됐을 때, 그 결과값을 fn에 넘겨서 실행|
|`.catch(fn)`|Promise가 rejected(실패)됐을 때, 그 에러를 fn에 넘겨서 실행|
|`.finally(fn)`|성공/실패와 무관하게 항상 마지막에 실행|

```txt
.then()을 여러 번 이어붙일 수 있는 이유: .then() 자체도 새로운 Promise를 반환하기 때문 —
그래서 .then().then().catch() 처럼 체이닝이 가능함(배열의 메서드 체이닝과 비슷한 발상)
```

---

# async/await — Promise를 다루는 더 읽기 쉬운 문법 ⭐️⭐️⭐️⭐️

```javascript
async function getData() {
  try {
    const res = await fetch('/api/data');
    const data = await res.json();
    return data;
  } catch (err) {
    console.error(err);
  } finally {
    console.log('끝');
  }
}
```

```txt
async 함수는 항상 Promise를 반환함:
  return data;처럼 평범한 값을 반환해도, 그 함수를 호출하면 결과는 항상 Promise로 한 번 감싸져서 나옴

await는 "그 Promise가 fulfilled/rejected될 때까지 기다렸다가, 결과값을 꺼내오는" 것:
  .then() 체인을 안 쓰고도, 마치 동기 코드처럼 위에서 아래로 순서대로 읽을 수 있게 해줌

try/catch가 .catch()의 역할을 대신함:
  await한 Promise가 reject되면 그 자리에서 에러가 throw된 것처럼 동작 → catch 블록이 잡음
```

```txt
.then() 체이닝과 async/await는 결국 같은 걸 하는 두 가지 문법일 뿐임 —
지금은 async/await가 가독성 면에서 훨씬 더 흔하게 쓰임(특히 NestJS/Next.js 코드 전반)
```

---

# Promise.all — 여러 Promise를 동시에 실행하고 다 끝나면 한꺼번에 받기 ⭐️⭐️⭐️⭐️

```typescript
const [total, hidden, today] = await Promise.all([
  this.prisma.recommendation.count(),
  this.prisma.recommendation.count({ where: { hidden: true } }),
  this.prisma.recommendation.count({ where: { createdAt: { gte: startOfToday } } }),
]);
```

```txt
순서대로 하나씩 await하면(직렬):
  const total = await ...count();
  const hidden = await ...count({...});
  const today = await ...count({...});
  → 각 쿼리가 끝나야 다음 쿼리가 시작됨 — 총 걸리는 시간 = 세 쿼리 시간의 합

Promise.all로 한꺼번에 보내면(병렬):
  세 쿼리를 동시에 날리고, "전부 다" 끝날 때까지만 기다림
  → 총 걸리는 시간 = 그중 가장 오래 걸리는 쿼리 하나의 시간(셋을 다 더한 게 아님)

→ 서로 의존 관계가 없는 독립적인 작업들일 때만 이렇게 안전하게 동시 실행할 수 있음
  (today 쿼리가 total의 결과를 필요로 한다면 동시에 실행하면 안 됨 — 그땐 직렬이어야 함)
```

```txt
배열 구조분해로 결과를 받을 수 있는 이유:
  Promise.all이 반환하는 배열의 순서는 항상 "입력한 배열의 순서"와 같음
  (먼저 끝난 게 먼저 오는 게 아니라, 넣은 순서 그대로 결과가 채워짐)
  → const [total, hidden, today] = await Promise.all([...]) 처럼 순서대로 이름을 붙일 수 있는 것
  (배열 구조분해 자체는 [[JS_Operators]] 참고)
```

## ⚠️ 하나라도 실패하면 전체가 실패 ⭐️⭐️⭐️⭐️

```txt
배열 안의 Promise 중 하나라도 reject되면, Promise.all 전체가 그 즉시 reject됨
(다른 두 개가 이미 성공했어도 그 결과는 그냥 버려짐 — 받을 방법이 없음)

→ "일부는 성공, 일부는 실패해도 각각의 결과를 다 보고 싶다"면 Promise.all이 아니라
  Promise.allSettled를 써야 함(바로 아래 참고)
```

---

# Promise.allSettled / race / any — 변형들 ⭐️⭐️⭐️

|메서드|동작|
|---|---|
|`Promise.all`|전부 성공해야 성공 — 하나라도 실패하면 즉시 전체 실패|
|`Promise.allSettled`|성공/실패 여부와 무관하게 전부 기다린 뒤, 각각의 결과(성공/실패)를 모아서 반환|
|`Promise.race`|가장 먼저 끝난 것(성공이든 실패든) 하나만 반환 — 나머지는 그냥 무시됨|
|`Promise.any`|가장 먼저 "성공"한 것 하나만 반환 — 전부 실패해야만 전체 실패|

```javascript
const results = await Promise.allSettled([fetchA(), fetchB(), fetchC()]);
// [{ status: 'fulfilled', value: ... }, { status: 'rejected', reason: ... }, ...]
// 실패한 것도 결과 배열 안에 그대로 들어있음 — 어떤 게 성공/실패했는지 각자 확인 가능
```

```txt
실무에서 가장 자주 쓰는 건 Promise.all(독립적인 작업 동시 실행)이고,
"일부 실패는 허용하고 싶다"는 요구가 있을 때만 allSettled로 바꾸는 정도면 충분함
race/any는 비교적 특수한 상황(타임아웃 구현, 여러 소스 중 가장 빠른 응답 쓰기 등)에 씀
```

---

# 직렬(for...of+await) vs 병렬(Promise.all) — 언제 뭘 쓰나 ⭐️⭐️⭐️⭐️

```txt
판단 기준은 딱 하나 — "다음 작업이 이전 작업의 결과를 필요로 하는가?"

필요함(의존 관계 있음)  → 직렬: for...of + await로 하나씩 순서대로
필요 없음(서로 독립적) → 병렬: Promise.all로 동시에 — 보통 훨씬 빠름

(forEach + async가 왜 await를 안 기다려주는지, for...of/Promise.all 중 어느 쪽을 써야 하는지의
 실전 코드 비교는 [[JS_Array_Methods]]의 "forEach/map에서 async를 쓸 때" 참고)
```

---

# 한눈에

| 개념                                    | 핵심                                                         |
| ------------------------------------- | ---------------------------------------------------------- |
| Promise                               | "나중에 결과가 나올 값" — pending → fulfilled 또는 rejected           |
| `.then()` / `.catch()` / `.finally()` | 성공 시 / 실패 시 / 항상 실행되는 콜백                                   |
| `async`/`await`                       | Promise를 동기 코드처럼 읽기 쉽게 만든 문법(같은 것의 다른 표현)                  |
| `Promise.all([...])`                  | 독립적인 작업들을 동시에 실행, 전부 끝나면 결과를 순서대로 배열로                      |
| `Promise.all`의 함정                     | 하나라도 실패하면 전체가 즉시 실패 — 다른 성공 결과도 버려짐                        |
| `Promise.allSettled`                  | 성공/실패 여부와 무관하게 전부의 결과를 모아서 반환                              |
| 직렬 vs 병렬                              | 다음 작업이 이전 결과에 의존하면 직렬(`for...of`), 독립적이면 병렬(`Promise.all`) |