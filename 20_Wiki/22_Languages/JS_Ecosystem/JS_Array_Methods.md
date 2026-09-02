---
aliases: [Array.from, 배열 메서드, JS 배열]
tags: [JavaScript]
relations:
  - "[[00_JS_Ecosystem_HomePage]]"
  - "[[JS_Object_Methods]]"
  - "[[JS_Map_Set]]"
  - "[[JS_Primitive_Methods]]"
  - "[[React_useState]]"
---
# JS_Array_Methods — 배열 메서드

>[!info]
> 배열 메서드 = 함수를 인자로 받아 각 요소에 실행하는 패턴 하나.
> `filter` · `map` · `some` · `find` 전부 이 구조다.
> Map · Set 자료구조 → [[JS_Map_Set]] 참고.

---

# 배열 메서드의 기본 구조 ⭐️⭐️⭐️⭐️

```typescript
// 모든 배열 메서드는 이 형태
array.메서드((item) => /* item에 대해 실행할 코드 */ );
//             ↑ 각 요소에 호출되는 함수 (콜백)
```

```typescript
const users = [
  { id: 1, name: '홍길동', role: 'admin' },
  { id: 2, name: '김철수', role: 'user'  },
  { id: 3, name: '이영희', role: 'user'  },
];

// 각 메서드는 이 함수를 users[0], users[1], users[2]에 순서대로 실행함
users.filter((user) => user.role === 'admin');
//            ↑ 이 함수가 user = {id:1...}, {id:2...}, {id:3...} 순으로 호출됨
```

```txt
반환값의 의미 — 메서드마다 함수의 반환값을 다르게 쓴다:

  filter   → true 반환: "이 요소 남겨줘" / false 반환: "버려줘"
  map      → 반환값이 새 배열의 그 자리에 들어감
  some     → true 반환: "찾았다, 멈춰" (전체 true) / false: "계속 찾아"
  every    → false 반환: "하나가 틀렸다, 멈춰" (전체 false)
  find     → true 반환: "이게 찾던 것, 이 요소 반환하고 멈춰"
  forEach  → 반환값 무시 (부수효과만)
```

---

# 언제 뭘 쓰는가 ⭐️⭐️⭐️⭐️

```txt
핵심 질문 하나:
  "결과로 boolean이 필요한가?" → some / every / includes
  "요소 하나가 필요한가?"      → find / findLast
  "새 배열이 필요한가?"        → filter / map / flatMap
  "값 하나로 줄이고 싶은가?"   → reduce
```

|메서드|반환|한 줄 설명|
|---|---|---|
|`some`|`boolean`|하나라도 조건 맞는 게 있나?|
|`every`|`boolean`|전부 다 조건에 맞나?|
|`includes`|`boolean`|이 값이 있나? (원시값)|
|`find`|요소 or `undefined`|앞에서 첫 번째 일치 항목|
|`findLast`|요소 or `undefined`|뒤에서 첫 번째 일치 항목|
|`findIndex`|`number` (`-1` 없음)|첫 번째 일치 인덱스|
|`filter`|새 배열|조건 맞는 것만 남기기|
|`map`|새 배열 (같은 길이)|각 요소를 변환|
|`flatMap`|새 배열|map 후 한 단계 펼치기|
|`reduce`|값 하나|배열 → 합계·객체·Map|
|`sort`|원본 배열|비교 함수로 정렬 (원본 변경 주의)|
|`slice`|새 배열|원본 유지하며 일부 추출|
|`forEach`|`undefined`|반복 실행 (반환 없음)|

---

# boolean — some · every · includes ⭐️⭐️⭐️⭐️

```typescript
const nums = [1, 2, 3, 4, 5];

nums.some((n) => n > 3)   // true  — 4, 5 중 하나라도 > 3
nums.every((n) => n > 0)  // true  — 전부 > 0
nums.every((n) => n > 3)  // false — 1,2,3은 아님
```

```txt
some  → OR 논리 (하나라도 맞으면 true) — 찾는 순간 멈춤
every → AND 논리 (모두 맞아야 true) — 틀린 순간 멈춤

빈 배열:
  [].some(...)  → false (하나도 없으니 false)
  [].every(...) → true  (반례가 없으니 true)

성능:
  some  → 조건 만족하는 순간 나머지 순회 멈춤
  find  → 찾는 순간 멈춤
  filter → 무조건 전체 순회

  "있냐 없냐"만 알면 됨 → some (filter보다 빠름)
```

```typescript
// 실전 패턴 — boolean을 setState에 바로 연결
const hasUnreadRoom = rooms.some(
  (room) => Boolean(room.unread) && !isRoomMuted(userId, room.id),
);
setRoomsUnread(hasUnreadRoom);
```

---

# 찾기 — find · findLast · findIndex ⭐️⭐️⭐️

```typescript
const users = [
  { id: 1, name: '홍길동' },
  { id: 2, name: '김철수' },
  { id: 3, name: '홍길순' },
];

users.find((u) => u.id === 2)     // { id: 2, name: '김철수' }
users.find((u) => u.id === 99)    // undefined — 없으면 undefined

users.findLast((u) => u.name.startsWith('홍'))  // { id: 3, name: '홍길순' } — 뒤에서 첫 번째
users.findIndex((u) => u.id === 2)              // 1 (인덱스)
users.findIndex((u) => u.id === 99)             // -1 (없음)
```

```txt
find vs filter:
  find   → 첫 번째 하나만 (없으면 undefined)
  filter → 조건 맞는 전부 (없으면 빈 배열)

  "특정 항목 꺼내기" → find
  "조건 맞는 항목들 목록" → filter

findLast — ES2023, Node 20+:
  배열 뒤에서부터 찾음 → 시간순 정렬된 배열에서 "가장 최근 것"
  지원 안 되는 환경: [...arr].reverse().find(...)
```

```typescript
// 실전 — 채팅에서 마지막 추천 메시지
const lastRec = messages.findLast(
  (m) => m.type === 'recommendation' && m.recommendation,
)?.recommendation;
// ?. — findLast가 undefined 반환하면 접근 안 함
```

---

# 새 배열 — filter · map · flatMap

## filter ⭐️⭐️⭐️⭐️

```typescript
const nums   = [1, 2, 3, 4, 5];
const evens  = nums.filter((n) => n % 2 === 0);   // [2, 4]

const active = users.filter((u) => u.isActive);
const admins = users.filter((u) => u.role === 'admin');

// null / undefined 제거
const values  = [1, null, 2, undefined, 3];
const cleaned = values.filter((v): v is number => v != null);  // [1, 2, 3]
// 타입 서술어 → [[TS_Type_Guards]]
```

```txt
filter 조건 방향:
  남길 것  → 긍정 조건  (u.isActive)
  제거할 것 → 부정 조건  (m.id !== deletedId)

  "이 항목 빼고 나머지" = 삭제 패턴 → filter(x => x.id !== 삭제할id)
```

## map ⭐️⭐️⭐️⭐️

```typescript
const nums    = [1, 2, 3];
const doubled = nums.map((n) => n * 2);          // [2, 4, 6]
const names   = users.map((u) => u.name);        // ['홍길동', '김철수']
const dtos    = users.map((u) => toUserDto(u));  // 형태 변환
```

```txt
map = 길이는 같고, 각 요소의 형태만 바꿈
filter = 길이가 줄어들 수 있음, 형태는 그대로

map vs forEach:
  map     → 변환된 새 배열 반환 (결과가 필요할 때)
  forEach → 반환값 없음 (로깅, DOM 조작 같은 부수효과만)
```

## flatMap ⭐️⭐️

```typescript
const sentences = ['hello world', 'foo bar'];
sentences.flatMap((s) => s.split(' '));  // ['hello', 'world', 'foo', 'bar']
// map 하면 [['hello','world'], ['foo','bar']] — 한 단계 더 펼쳐줌

const nested = [[1, 2], [3, 4], [5]];
nested.flat();   // [1, 2, 3, 4, 5]
nested.flat(2);  // 2단계 중첩까지 펼치기
```

---

# 줄이기 — reduce ⭐️⭐️⭐️

```typescript
const nums = [1, 2, 3, 4, 5];

// 합계
const sum = nums.reduce((acc, n) => acc + n, 0);  // 15
//                       ↑   ↑                ↑
//                    누적값 현재요소       초기값

// 배열 → 객체 (그룹핑)
const byRole = users.reduce<Record<string, User[]>>((acc, user) => {
  (acc[user.role] ??= []).push(user);
  return acc;
}, {});
// { admin: [...], user: [...] }
```

```txt
reduce 읽는 법:
  (acc, item) => 새로운 acc

  acc  = 지금까지 누적된 결과 (초기값에서 시작)
  item = 현재 요소
  반환값 = 다음 acc

  초기값이 0이면 합계, {}이면 객체, []이면 배열, new Map이면 Map

언제 reduce 대신 다른 메서드:
  새 배열 만들기 → map 또는 flatMap이 더 명확
  조건 맞는 것만 → filter가 더 명확
  그룹핑이나 집계 → reduce가 적합

Map 그루핑 → [[JS_Map_Set]] — Map<string, T[]> 그루핑 패턴
```

## 배열 → Record — 1:1 키-값 매핑 ⭐️⭐️⭐️⭐️

```typescript
// 각 item을 특정 key(wallSlot)로 인덱싱
const posters = response.items.reduce<Record<number, GachaMovie>>(
  (current, item) => {
    current[item.wallSlot] = item.movie;
    return current;
  },
  {},
);
// 결과: { 1: GachaMovie, 2: GachaMovie, 3: GachaMovie }
// posters[1] → O(1) 즉시 조회
```

```typescript
// 동일한 결과, 세 가지 방법

// 1. reduce — 변환 로직이 복잡하거나 조건 분기 있을 때
const posters = items.reduce<Record<number, GachaMovie>>((acc, item) => {
  acc[item.wallSlot] = item.movie;
  return acc;
}, {});

// 2. Object.fromEntries — 단순 매핑이면 가장 선언적
const posters = Object.fromEntries(
  items.map(item => [item.wallSlot, item.movie]),
) as Record<number, GachaMovie>;

// 3. for 루프 — 디버깅 중이거나 break가 필요할 때
const posters: Record<number, GachaMovie> = {};
for (const item of items) {
  posters[item.wallSlot] = item.movie;
}
```

```txt
그룹핑과의 차이:
  그룹핑 (1:N)  acc[key] ??= []; acc[key].push(item)  — key 하나에 여러 값
  매핑  (1:1)   acc[key] = value                       — key 하나에 값 하나

Record vs Map 선택:
  JSON 직렬화 필요 / 키가 string·number → Record
  키가 객체·Symbol / 삽입 순서 보장 필요 → Map  → [[JS_Map_Set]]
```

---

# 정렬 — sort ⭐️⭐️⭐️⭐️

```typescript
arr.sort((a, b) => 비교값);
```

|반환값|의미|
|---|---|
|음수|a가 b보다 앞에|
|양수|b가 a보다 앞에|
|`0`|순서 유지|

```txt
외우는 법:
  a - b → 오름차순 (작은 것이 앞)
  b - a → 내림차순 (큰 것이 앞)

⚠️ sort는 원본 배열을 직접 바꿈
  React state에서는 [...prev].sort(...)로 복사 후 정렬
```

```typescript
// 숫자
nums.sort((a, b) => a - b);  // 오름차순
nums.sort((a, b) => b - a);  // 내림차순

// 날짜 (최신순)
messages.sort((a, b) =>
  new Date(b.createdAt).getTime() - new Date(a.createdAt).getTime()
);

// 문자열 (한국어 포함)
names.sort((a, b) => a.localeCompare(b, 'ko'));  // 오름차순 (가나다순)
```

## localeCompare — 문자열 비교 ⭐️⭐️⭐️⭐️

```typescript
'apple'.localeCompare('banana')     // -1  (apple이 banana보다 앞)
'banana'.localeCompare('apple')     // 1   (banana가 apple보다 뒤)
'apple'.localeCompare('apple')      // 0   (같음)
```

```txt
localeCompare란:
  두 문자열을 비교해서 -1 · 0 · 1을 반환
  sort의 비교 함수가 기대하는 음수/0/양수를 그대로 반환
  → sort((a, b) => a.localeCompare(b))로 바로 사용 가능

왜 a - b 대신 localeCompare를 쓰는가:
  a - b는 숫자에서만 작동
  문자열을 < > 로 비교하면 유니코드 코드 포인트 순서 → 한글·특수문자 이상하게 정렬됨
  localeCompare는 언어 규칙에 맞게 비교 (한국어는 'ko')
```

## ISO 날짜 문자열 정렬 ⭐️⭐️⭐️⭐️

```typescript
// "YYYY-MM-DD" 형식은 문자열 사전순 = 날짜 순서가 일치
messages.sort((a, b) => b.createdAt.localeCompare(a.createdAt));
//                       ↑ b가 앞 → 최신순 (내림차순)
messages.sort((a, b) => a.createdAt.localeCompare(b.createdAt));
//                       ↑ a가 앞 → 오래된 순 (오름차순)
```

```txt
b.localeCompare(a) 읽는 법:
  b가 a보다 크면(최신이면) 양수 → b가 앞 → 최신순

  new Date().getTime() 방식과의 차이:
    getTime() 방식  → Date 객체를 두 번 생성 → 약간 느림
    localeCompare   → 문자열 직접 비교 → 더 빠름

  단, ISO 형식이 아닌 날짜 문자열 ("2024년 1월 15일" 등)은 불가
  → 반드시 "YYYY-MM-DD" 또는 "YYYY-MM-DDTHH:mm:ssZ" 형식이어야 함
```

## 다중 조건 정렬 ⭐️⭐️⭐️⭐️

```typescript
// 방장을 항상 앞에, 나머지는 입장 시각 순
members.sort((a, b) => {
  if (a.role === 'owner' && b.role !== 'owner') return -1;  // a 앞
  if (b.role === 'owner' && a.role !== 'owner') return 1;   // b 앞
  return a.joinedAt - b.joinedAt;  // 둘 다 같은 역할 → 시각 순
});
```

```txt
다중 조건 정렬 읽는 법:
  return -1 → a, b 순서 그대로 (a 앞)
  return 1  → b, a로 바꿈 (b 앞)
  마지막 return → 1차 조건이 같을 때 2차 기준으로 결정
```

---

# 일부 추출 — slice ⭐️⭐️⭐️⭐️

```typescript
const arr = ['a', 'b', 'c', 'd', 'e'];
//           [0]  [1]  [2]  [3]  [4]

arr.slice(1)       // ['b', 'c', 'd', 'e']  — 1번 인덱스부터 끝까지
arr.slice(1, 3)    // ['b', 'c']             — 1번부터 3번 앞까지 (3 미포함)
arr.slice(0, 3)    // ['a', 'b', 'c']        — 처음 3개
arr.slice(-2)      // ['d', 'e']             — 뒤에서 2개
arr.slice(0, -1)   // ['a', 'b', 'c', 'd']  — 마지막 제외
```

```txt
slice(start, end):
  start → 시작 인덱스 (포함)
  end   → 끝 인덱스 (미포함) — 생략하면 끝까지
  음수  → 뒤에서부터 (-1 = 마지막)

  원본 배열을 바꾸지 않음 → 새 배열 반환 (안전)

splice와 헷갈리지 말 것:
  slice  → 원본 변경 없음, 새 배열 반환
  splice → 원본을 직접 변경 (React state에서 쓰면 안 됨)
```

```typescript
// 자주 쓰는 패턴
const replies = thread.slice(1);               // 첫 번째 요소 제외 (나머지만)
const preview = thread.slice(0, PREVIEW_COUNT); // 처음 N개만
const recent  = messages.slice(-5);            // 최근 5개
const copy    = arr.slice();                   // 전체 복사 (원본 변경 방지)
```

---

# 반복 생성 — Array.from ⭐️⭐️⭐️⭐️

```txt
Array.from(배열처럼 생긴 것, 변환함수?)

배열이 아닌데 배열처럼 순회하고 싶을 때:
  문자열, Set, Map, NodeList, arguments 등
  또는 { length: n } 으로 원하는 길이의 배열을 만들 때
```

```typescript
// 기본 사용
Array.from('hello')                        // ['h', 'e', 'l', 'l', 'o']
Array.from(new Set([1, 2, 2, 3]))          // [1, 2, 3]
Array.from([1, 2, 3], (x) => x * 2)       // [2, 4, 6]  — map처럼 변환
```

## { length: n } — 원하는 길이로 배열 생성 ⭐️⭐️⭐️⭐️

```typescript
Array.from({ length: 3 })                  // [undefined, undefined, undefined]

Array.from({ length: 3 }, (_, i) => i)    // [0, 1, 2]
//                          ↑   ↑
//                        값(무시) 인덱스

Array.from({ length: 5 }, (_, i) => i + 1)  // [1, 2, 3, 4, 5]
Array.from({ length: 5 }, () => 0)           // [0, 0, 0, 0, 0]  고정값
Array.from({ length: 5 }, () => [])          // 독립된 빈 배열 5개 ← 중요
Array.from({ length: 5 }, () => null)        // [null, null, null, null, null]
```

```txt
_ 가 뭔가:
  Array.from의 변환 함수는 (값, 인덱스) 두 인자를 받음
  { length: n }으로 만들면 값이 undefined라 쓸모없음
  → _  로 "이 인자는 안 쓴다"는 관례, i 로 인덱스만 사용

왜 new Array(n).fill([]) 대신 Array.from:
  new Array(3).fill([]) → [ref, ref, ref]  같은 배열 참조 공유!
  arr[0].push(1) → arr[1]도 바뀜

  Array.from({ length: 3 }, () => []) → 독립된 배열 3개
  초기값이 원시값(0, '', null)이면 fill()도 안전
  초기값이 객체·배열이면 Array.from 사용
```

```typescript
// 실전 — 시간대별 통계 버킷 초기화
const series = Array.from({ length: 24 }, () => 0);
// [0, 0, ..., 0]  길이 24

for (const time of times) {
  const slot = Math.floor((time.getTime() - start.getTime()) / slotMs);
  if (slot >= 0 && slot < 24) series[slot]++;
}

// React — 별점·슬롯·반복 UI
Array.from({ length: full }, (_, i) => <Star key={`f${i}`} fill="currentColor" />)
Array.from({ length: 12 }, (_, i) => <div key={i} className="grid-cell" />)

// 페이지 번호 버튼
Array.from({ length: totalPages }, (_, i) => (
  <button key={i} onClick={() => goTo(i + 1)}>{i + 1}</button>
))
```

## 역순 숫자 배열 — year - index 패턴 ⭐️⭐️⭐️⭐️

```typescript
// 현재 연도부터 2000년까지 내림차순 배열
const year = new Date().getFullYear();  // 예: 2026

Array.from({ length: year - 1999 }, (_, index) => year - index)
// length = 2026 - 1999 = 27
// index: 0, 1, 2, ..., 26
// 결과:  [2026, 2025, 2024, ..., 2000]
```

```txt
분해:
  { length: year - 1999 }
    → 배열 길이 = "현재 연도 - 1999"
    → 2026이면 length=27 (2000~2026 = 27개)

  (_, index) => year - index
    _ = 값 (undefined, 무시)
    index = 0, 1, 2, ... (오름차순)
    year - index = 2026-0=2026, 2026-1=2025, 2026-2=2024 ...
    → 최신 연도가 앞에 오는 내림차순

왜 i + 2000이 아닌 year - index인가:
  i + 2000 → [2000, 2001, ..., 2026]  오름차순 (가장 오래된 것부터)
  year - index → [2026, 2025, ..., 2000]  내림차순 (최신 것부터)
  select 드롭다운에서는 "올해"가 기본값이고 위에 있어야 UX에 좋음
```

```tsx
// 실전 — 연도 select 드롭다운
const year = new Date().getFullYear();

<select
  value={selectedYear}
  onChange={(e) => setSelectedYear(Number(e.target.value))}
  aria-label="통계 연도 선택"
>
  {Array.from({ length: year - 1999 }, (_, index) => year - index).map(
    (optionYear) => (
      <option key={optionYear} value={optionYear}>
        {optionYear}년
      </option>
    ),
  )}
</select>
```

```txt
포인트:
  value={optionYear}       숫자로 저장 (Number 타입 유지)
  onChange에서 Number(e.target.value)  select value는 항상 string → 숫자로 변환
  key={optionYear}         고유한 숫자 → key로 적합
  기본 선택: selectedYear 초기값을 year로 설정하면 "올해"가 선택됨

범용 변형:
  2000 ~ 올해    { length: year - 1999 }, (_, i) => year - i   → 내림차순
  올해 ~ 5년 후  { length: 6 },           (_, i) => year + i   → 오름차순
  1900 ~ 올해    { length: year - 1899 }, (_, i) => year - i   → 생년월일 드롭다운
```

```typescript
// 월 배열 (1~12)
Array.from({ length: 12 }, (_, i) => i + 1)
// [1, 2, 3, ..., 12]

// 일 배열 (1~31)
Array.from({ length: 31 }, (_, i) => i + 1)
// [1, 2, 3, ..., 31]
```


---

# 반복 실행 — forEach ⚠️

```typescript
// ❌ forEach + async — await를 기다리지 않음
users.forEach(async (user) => {
  await sendEmail(user.email);  // 완료를 기다리지 않고 다음으로 넘어감
});

// ✅ for...of — 순서대로 기다림
for (const user of users) {
  await sendEmail(user.email);
}

// ✅ Promise.all — 동시에 실행, 전부 완료될 때까지 기다림
await Promise.all(users.map((user) => sendEmail(user.email)));
```

```txt
forEach가 async를 기다리지 않는 이유:
  forEach는 콜백의 반환값을 무시하도록 설계됨
  async 함수는 Promise를 반환하는데 forEach가 그 Promise를 기다리지 않음
  → 루프가 끝나도 이메일이 다 발송되지 않은 상태

순서가 중요 → for...of + await
순서 무관, 동시에 빠르게 → Promise.all + map
```

---

# 조건 함수 (predicate) 패턴 ⭐️⭐️⭐️⭐️

```typescript
// filter / some / find / every에 넘기는 함수 = 조건 함수 (predicate)
// 이 함수가 true를 반환하면 "이 요소는 조건에 맞음"

function matchesQuery(room: Room, raw: string): boolean {
  const q = raw.trim().toLowerCase();

  if (!q) return true;
  //       ↑ 검색어 없음 → 모든 방이 조건에 맞음 (filter에서 전부 통과)

  if (room.name.toLowerCase().includes(q)) return true;
  //                                         ↑ 이름에 있으면 확정 → 조기 반환

  return room.tags.some((t) => t.toLowerCase().includes(q));
  //     ↑ 이름엔 없었음 → 태그에서 확인 → true 또는 false 둘 다 가능
}

// 사용
rooms.filter((room) => matchesQuery(room, searchQuery));
```

```txt
early return 읽는 법:
  답이 확실한 순간 바로 반환 → 이후 검사를 안 해도 됨
  else 중첩 없이 위에서 아래로 평탄하게 읽힘
  "여기까지 왔으면" = 위의 return들에 걸리지 않은 상황
```

---

# React state에서 배열 다루기 ⭐️⭐️⭐️⭐️

## 불변성 — setState 안에서 원본 변경 금지

```txt
이 메서드들은 원본을 직접 바꿈 → React가 변경을 감지 못함 → 리렌더 없음:
  push / pop / shift / unshift / splice / sort / reverse / fill

setState 안에서는 항상 새 배열을 반환해야 함:
  추가    [...prev, newItem]
  삭제    prev.filter(...)
  수정    prev.map(...)
  정렬    [...prev].sort(...)  ← 복사 후 정렬
```

## 기본 패턴 4가지

```typescript
// 추가
setState((prev) => [...prev, newItem]);

// 삭제
setState((prev) => prev.filter((x) => x.id !== targetId));

// 수정
setState((prev) =>
  prev.map((x) => x.id === targetId ? { ...x, ...changes } : x)
);

// 토글 (있으면 삭제, 없으면 추가)
setState((prev) =>
  prev.includes(item)
    ? prev.filter((x) => x !== item)
    : [...prev, item]
);
```

## return prev — 아무것도 안 바꿀 때 ⭐️⭐️⭐️⭐️

```typescript
setState((prev) => {
  if (prev.includes(item))  return prev;  // 이미 있으면 그대로
  if (prev.length >= MAX)   return prev;  // 꽉 찼으면 그대로
  return [...prev, item];                  // 여기까지 오면 추가
});
```

```txt
return prev의 의미:
  React는 setState(fn)에서 반환된 값과 이전 값을 Object.is()로 비교
  return prev         → 같은 참조 → "변경 없음" → 리렌더 없음
  return [...prev, item] → 새 배열 → 다른 참조 → 리렌더 발생

  return [...prev]  ← 내용이 같아도 새 배열 → 리렌더 O
  return prev       ← 참조가 같으면 리렌더 X
```

## 중복 방지 — some 활용 ⭐️⭐️⭐️⭐️

```typescript
// 소켓 echo + REST 응답이 같은 메시지를 두 번 보낼 수 있는 상황
const appendMessage = useCallback((message: ApiRoomMessage) => {
  setMessages((prev) => {
    if (prev.some((m) => m.id === message.id)) return prev;  // 이미 있으면 그대로
    return [...prev, message];
  });
}, []);
```

```txt
some을 쓰는 이유:
  "이미 있나?"라는 boolean 질문 → some이 정확한 선택
  있으면 prev를 그대로 반환 (새 배열 불필요)
  find를 쓰면 항목이 반환되는데 여기선 항목이 필요하지 않음
```

## 중첩 배열 수정 — map + filter ⭐️⭐️⭐️⭐️

```typescript
// 메시지 목록에서 특정 메시지의 reactions만 수정
setMessages((prev) =>
  prev.map((msg) => {
    if (msg.id !== targetId) return msg;  // 관계없으면 참조 그대로 유지

    // reactions에서 기존 반응 제거 후 새 반응 추가
    const without = msg.reactions.filter((r) => r.userId !== userId);
    return { ...msg, reactions: [...without, newReaction] };
  }),
);
```

```txt
return msg (참조 유지):
  관계없는 메시지는 같은 객체 참조를 그대로 반환
  → React가 "이 요소는 변경 없음"으로 판단 → 불필요한 리렌더 없음

return { ...msg, reactions: [...without, newReaction] }:
  새 객체 + 새 배열 → React가 변경 감지 → 이 메시지만 리렌더

filter 먼저, 추가 나중:
  기존 반응을 먼저 지우고 새 반응을 추가하는 순서를 지키면
  이모지 교체 시 같은 userId의 반응이 두 개 생기는 것 방지
```
