---
aliases:
  - for in
  - for loop
  - for of
  - if else
  - switch
  - while
tags:
  - JavaScript
related:
  - "[[00_JS_Ecosystem_HomePage]]"
  - "[[JS_Promise]]"
  - "[[JS_Object_Methods]]"
---
# JS_Loops_Conditionals — 반복문 · 조건문

>[!info]
>반복문 선택 기준: 배열/이터러블 순회 → `for...of`, 인덱스 필요 → `for (i)`, 객체 키 → `for...in`, 
>종료 조건 모를 때 → `while`. `for...of`는 `break`·`continue`·`async/await`가 자연스럽게 작동한다.

---

# 반복문 선택 기준 ⭐️⭐️⭐️⭐️

```txt
배열·Set·Map·이터러블 순회     → for...of  (가장 많이 씀)
순회 중 인덱스가 필요          → for (let i = 0; ...)
객체의 키를 순회               → for...in  또는 Object.keys()
순회 중 async/await 필요      → for...of  (forEach는 안 됨)
종료 조건을 미리 모를 때       → while
최소 1번은 반드시 실행해야     → do...while
배열에 콜백으로 처리           → forEach  (break 필요 없을 때)
```

---

# for...of — 이터러블 순회 ⭐️⭐️⭐️⭐️

```typescript
const items = ['a', 'b', 'c'];

// 기본
for (const item of items) {
  console.log(item);  // 'a', 'b', 'c'
}

// 인덱스도 필요하면 entries()
for (const [index, item] of items.entries()) {
  console.log(index, item);  // 0 'a', 1 'b', 2 'c'
}

// Set 순회
const roomIds = new Set<string>();
for (const id of roomIds) { ... }

// Map 순회
const map = new Map([['a', 1], ['b', 2]]);
for (const [key, value] of map) {
  console.log(key, value);  // 'a' 1, 'b' 2
}
```

## for...of vs forEach ⭐️⭐️⭐️⭐️

```typescript
// forEach — 배열 전용, break 불가
items.forEach(item => {
  if (item === 'b') return;  // ← 이건 forEach 콜백 리턴 (break 아님)
  console.log(item);
});

// for...of — break, continue, return 전부 작동
for (const item of items) {
  if (item === 'b') break;   // ✅ 진짜 루프 중단
  if (item === 'a') continue; // ✅ 이번 순회 건너뜀
  console.log(item);
}
```

```txt
forEach vs for...of 선택:
  forEach → 단순 순회, break 필요 없음, 콜백 스타일이 자연스러울 때
  for...of → break·continue 필요, async/await 필요, 일반 코드 흐름이 자연스러울 때

async/await에서 forEach는 안 됨:
  items.forEach(async (item) => {
    await doSomething(item);  // ← 각 콜백이 독립적 Promise, 순서 보장 안 됨
  });

  for (const item of items) {
    await doSomething(item);  // ✅ 순서대로 대기
  }
```

## 중첩 for...of ⭐️⭐️⭐️⭐️

```typescript
// targets의 각 유저가 가진 roomMembers를 순회해서 roomId 수집
for (const user of targets) {
  for (const member of user.roomMembers) {
    roomIds.add(member.roomId);
  }
}

// 읽는 법:
// targets 배열의 각 user에 대해
//   그 user의 roomMembers 배열의 각 member에 대해
//     member.roomId를 roomIds Set에 추가
```

```txt
중첩 루프 + Set 조합:
  roomIds.add()는 중복을 자동으로 무시
  여러 유저가 같은 방에 있어도 roomId가 한 번만 들어감

중첩 루프에서 외부 루프 break:
  레이블(label) 사용:

  outer: for (const user of users) {
    for (const item of user.items) {
      if (item.id === targetId) {
        foundUser = user;
        break outer;  // 외부 루프까지 종료
      }
    }
  }
```

---

# for (let i) — 인덱스 기반 ⭐️⭐️⭐️

```typescript
// 인덱스 접근이 필요할 때
for (let i = 0; i < items.length; i++) {
  console.log(i, items[i]);
}

// 역방향 순회
for (let i = items.length - 1; i >= 0; i--) {
  console.log(items[i]);
}

// 2칸씩 이동
for (let i = 0; i < items.length; i += 2) {
  console.log(items[i]);
}
```

```txt
for...of 대신 for (let i)를 쓰는 경우:
  인덱스 값 자체가 필요 (items[i-1]과 비교 등)
  역방향 순회
  특정 간격으로 이동
  두 배열을 같은 인덱스로 동시에 순회
```

---

# for...in — 객체 키 순회 ⭐️⭐️

```typescript
const obj = { a: 1, b: 2, c: 3 };

for (const key in obj) {
  console.log(key, obj[key]);  // 'a' 1, 'b' 2, 'c' 3
}
```

```txt
⚠️ for...in 주의사항:
  프로토타입 체인의 키까지 순회할 수 있음
  → 보통 Object.keys() / Object.entries()를 더 많이 씀

// 권장 방법
for (const key of Object.keys(obj)) { ... }
for (const [key, value] of Object.entries(obj)) { ... }

for...in이 배열에 쓰이면:
  인덱스를 문자열로 순회 → 의도치 않은 동작 가능
  배열에는 for...of 사용
```

---

# while — 종료 조건 모를 때 ⭐️⭐️⭐️⭐️

```typescript
// 기본 while
let count = 0;
while (count < 10) {
  count++;
}

// 종료 조건이 없어 보일 때 — 탈출 조건 반드시 필요
while (true) {
  const data = getNext();
  if (!data) break;   // 탈출 조건
  process(data);
}
```

## 트리/그래프 탐색 — 깊이 제한 + 순환 감지 ⭐️⭐️⭐️⭐️

```typescript
// 카테고리의 depth(깊이)를 계산 — 부모를 따라 올라가며 카운트
function getCategoryDepth(id: string, list: Category[]): number {
  const byId = new Map(list.map((c) => [c.id, c]));
  //           ↑ O(1) 탐색을 위해 Map으로 변환

  let depth   = 0;
  let current = byId.get(id);
  const seen  = new Set<string>();   // 방문한 노드 기록

  while (current?.parentId && depth < 32) {
    //          ↑ 부모가 있을 때만    ↑ 최대 32단계 제한

    if (seen.has(current.id)) break;  // 순환 참조 감지 → 즉시 탈출
    seen.add(current.id);             // 현재 노드를 방문 기록에 추가

    depth  += 1;
    current = byId.get(current.parentId);  // 부모로 이동
  }

  return depth;
}
```

```txt
이 while 패턴이 하는 것:

  current?.parentId:
    부모가 있으면 계속 올라감
    부모가 없으면 (루트 카테고리) 루프 종료

  depth < 32 — 최대 깊이 제한:
    순환 참조(A→B→A→B→...)가 있으면 무한루프 발생 가능
    32단계 이상이면 비정상 데이터로 판단하고 중단

  seen Set — 순환 참조 감지:
    이미 방문한 노드를 다시 만나면 순환이므로 break
    depth 제한보다 더 빠르게 순환을 잡아냄

  new Map(list.map((c) => [c.id, c])):
    배열 → Map 변환 → byId.get(id)로 O(1) 탐색
    배열에서 .find(c => c.id === id) 하면 O(n) → n이 크면 느림
```

---

# break · continue ⭐️⭐️⭐️

```typescript
const numbers = [1, 2, 3, 4, 5, 6, 7, 8, 9, 10];

// break — 조건 만족 시 루프 완전 종료
for (const n of numbers) {
  if (n === 5) break;
  console.log(n);   // 1, 2, 3, 4
}

// continue — 이번 순회 건너뛰고 다음으로
for (const n of numbers) {
  if (n % 2 === 0) continue;  // 짝수 건너뜀
  console.log(n);   // 1, 3, 5, 7, 9
}
```

---

# 조건문 패턴 ⭐️⭐️⭐️⭐️

## early return — 중첩 줄이기

```typescript
// ❌ 중첩이 깊어지는 패턴
function process(user: User | null) {
  if (user) {
    if (user.isActive) {
      if (user.role === 'admin') {
        doSomething(user);
      }
    }
  }
}

// ✅ early return — 각 조건을 순서대로 처리
function process(user: User | null) {
  if (!user)              return;  // null이면 종료
  if (!user.isActive)     return;  // 비활성이면 종료
  if (user.role !== 'admin') return;  // 어드민 아니면 종료

  doSomething(user);  // 여기까지 오면 모든 조건 통과
}
```

## switch — 여러 케이스 분기

```typescript
switch (action) {
  case 'accept':
    return handleAccept();
  case 'decline':
    return handleDecline();
  case 'block':
    return handleBlock();
  default:
    throw new Error(`알 수 없는 action: ${action}`);
}
```

```txt
if-else vs switch:
  if-else  → 조건이 범위나 복잡한 표현식일 때
  switch   → 하나의 값을 여러 케이스와 비교할 때 (더 읽기 쉬움)

switch에서 default:
  예상치 못한 값이 들어왔을 때 에러를 던지는 게 안전
  조용히 무시하면 버그가 숨어있을 수 있음
```

## 삼항 연산자 — 짧은 조건 분기

```typescript
// if-else 대신 — 값을 결정할 때
const label = isPending ? '처리 중' : '완료';
const className = isActive ? 'bg-green-500' : 'bg-gray-300';

// 중첩 삼항은 피함 (읽기 어려움)
// ❌ const status = a ? 'A' : b ? 'B' : 'C';
// ✅ if-else 또는 switch 사용
```