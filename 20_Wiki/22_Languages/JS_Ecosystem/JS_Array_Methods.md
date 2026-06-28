---
aliases: [array methods, filter, forEach, map, reduce]
tags:
  - JavaScript
related:
  - "[[00_JS_Ecosystem_HomePage]]"
  - "[[JS_Loops_Conditionals]]"
  - "[[JS_Primitive_Methods]]"
  - "[[NextJS_API_Mapper]]"
---
# JS_Array_Methods — 배열 메서드, 언제 뭘 쓰나

> [!info] 
> 배열 메서드는 ① 원본을 바꾸는가(변형) vs 새로 만드는가(비변형), ② 무엇을 반환하는가(같은 길이의 배열 / 줄어든 배열 / 값 하나 / boolean) 두 기준으로 나누면 헷갈리지 않는다.

---

# 변형(mutating) vs 비변형(non-mutating) ⭐️⭐️⭐️⭐️

|구분|메서드|원본 배열|
|---|---|---|
|변형 — 원본을 직접 바꿈|`push` · `pop` · `shift` · `unshift` · `splice` · `sort` · `reverse`|바뀜|
|비변형 — 새 배열/값을 만들어 반환|`map` · `filter` · `slice` · `concat` · `find` · `reduce`|안 바뀜|

```txt
⚠️ 의외로 자주 걸리는 함정 — sort()와 reverse()는 "새 배열을 반환"하는 것처럼 보이지만
   실제로는 원본 배열을 그 자리에서 직접 바꿔버림(변형) — 반환값은 그냥 바뀐 원본 자체

const sorted = items.sort((a, b) => a - b);
items === sorted  // true — 같은 배열을 가리킴, 새 배열이 아님

원본을 보존하고 싶으면 복사본을 먼저 만들고 정렬:
const sorted = [...items].sort((a, b) => a - b);
```

---

# 순회 계열 — 무엇을 반환하는가로 구분 ⭐️⭐️⭐️⭐️

| 메서드         | 반환하는 것                 | 길이          | 용도                                |
| ----------- | ---------------------- | ----------- | --------------------------------- |
| `forEach`   | `undefined` (반환값 없음)   | —           | 각 요소에 대해 뭔가 "실행"만 함 (부수효과용)       |
| `map`       | 새 배열                   | 원본과 같음      | 각 요소를 변환                          |
| `filter`    | 새 배열                   | 원본과 같거나 줄어듦 | 조건에 맞는 것만 추림                      |
| `find`      | 요소 하나 (또는 `undefined`) | —           | 조건에 맞는 첫 번째 요소 찾기                 |
| `findIndex` | 인덱스 (또는 `-1`)          | —           | 조건에 맞는 첫 번째 위치 찾기                 |
| `some`      | `boolean`              | —           | 조건에 맞는 게 하나라도 있는가                 |
| `every`     | `boolean`              | —           | 전부 조건에 맞는가                        |
| `includes`  | `boolean`              | —           | 정확히 그 값이 배열에 있는가 (`{text}===` 비교) |
| `reduce`    | 값 하나 (타입은 직접 정함)       | —           | 배열 전체를 하나의 값으로 누적/집계              |

```typescript
// 실전에서 본 예시들
reactions.filter((r) => r.type === 'like').length        // filter — 조건에 맞는 것만 추리고 개수
requiredRoles.includes(user?.role)                        // includes — 정확히 그 값이 있는가
requiredRoles.some((role) => user?.roles?.includes(role)) // some — 하나라도 겹치는 role이 있는가
apis.map((api) => mapPost(api, currentUserId))             // map — 각 항목을 변환해서 같은 길이로
```

```txt
map vs forEach 헷갈릴 때 기준 한 줄: "변환한 새 배열이 필요하면 map, 그냥 출력/저장 같은
부수효과만 필요하면 forEach" — forEach의 반환값(undefined)을 쓰려고 하면 그 자체가 잘못 쓰고 있다는 신호
```

---

# reduce — 가장 헷갈리는 메서드 ⭐️⭐️⭐️⭐️

```typescript
const total = [1, 2, 3].reduce((acc, cur) => acc + cur, 0);
//                               ↑누적값  ↑현재값      ↑초기값
// acc: 0→1→3→6 순서로 누적되며 마지막엔 6
```

|인자|의미|
|---|---|
|`acc` (accumulator)|지금까지 누적된 결과|
|`cur` (current)|지금 처리 중인 배열 요소|
|초기값 (두 번째 인자)|`acc`의 첫 시작값|

```txt
⚠️ 초기값을 안 주면: 배열의 첫 번째 요소가 초기값으로 쓰이고, 콜백은 두 번째 요소부터 실행됨
   배열이 비어있는데 초기값도 없으면 → TypeError 발생
   → 안전하게 쓰려면 초기값은 항상 명시하는 게 좋음

reduce는 "배열을 하나의 값으로 뭉친다"는 점에서 map/filter보다 일반적임 —
사실 map과 filter도 reduce로 구현 가능하지만, 의도가 명확한 map/filter가 있다면 그걸 우선 사용
```

---

# 메서드 체이닝 ⭐️⭐️

```typescript
const likedTitles = posts
  .filter((p) => p.likedByMe)   // 좋아요 누른 것만
  .map((p) => p.title);          // 제목만 추출
```

```txt
filter는 배열을 반환하니, 그 결과에 바로 .map을 또 호출할 수 있음
→ 비변형 메서드들은 항상 새 배열을 반환하기 때문에 이렇게 줄줄이 이어붙일 수 있음(체이닝)
```

---

# forEach/map에서 async를 쓸 때 — 흔한 함정 ⭐️⭐️⭐️⭐️

```typescript
// ❌ forEach는 콜백이 async여도 기다려주지 않음
items.forEach(async (item) => {
  await saveItem(item); // forEach 자체는 이 await를 기다리지 않고 바로 다음 반복으로 넘어감
});
console.log('다 끝남?'); // 실제로는 저장이 끝나기 전에 이 줄이 먼저 실행됨

// ✅ 순서대로 하나씩 기다려야 한다면 for...of 사용
for (const item of items) {
  await saveItem(item); // 진짜로 끝날 때까지 기다린 후 다음 반복
}

// ✅ 순서 상관없이 전부 동시에 실행해도 된다면 Promise.all + map
await Promise.all(items.map((item) => saveItem(item)));
```

```txt
forEach는 원래 "콜백의 반환값을 신경 안 쓰는" 메서드라서, async 함수가 반환하는 Promise도
그냥 무시해버림 — await가 있어도 forEach 입장에서는 못 본 척하고 다음 요소로 넘어가는 것
순서대로 기다릴지(for...of), 동시에 다 실행할지(Promise.all)는 [[JS_Loops_Conditionals]] 참고
```

---

# 한눈에

| 상황                            | 메서드                               |
| ----------------------------- | --------------------------------- |
| 각 요소를 변환한 새 배열이 필요            | `map`                             |
| 조건에 맞는 것만 걸러내고 싶음             | `filter`                          |
| 그냥 각 요소에 대해 실행만 하면 됨(반환값 불필요) | `forEach`                         |
| 조건에 맞는 첫 번째 요소/위치를 찾고 싶음      | `find` / `findIndex`              |
| 조건에 맞는 게 있는지/전부 맞는지만 알고 싶음    | `some` / `every`                  |
| 정확히 그 값이 배열에 있는지만 확인          | `includes`                        |
| 배열 전체를 하나의 값으로 집계             | `reduce`                          |
| 원본을 그대로 두고 정렬하고 싶음            | `[...arr].sort(...)` (복사 먼저)      |
| 배열 안에서 async 작업을 순서대로 기다려야 함  | `forEach` 대신 `for...of` + `await` |