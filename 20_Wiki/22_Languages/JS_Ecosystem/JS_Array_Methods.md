---
aliases:
  - array methods
  - Array.from
  - filter
  - forEach
  - map
  - reduce
  - Set
  - sort
  - localcompare
tags:
  - JavaScript
related:
  - "[[00_JS_Ecosystem_HomePage]]"
  - "[[JS_Object_Methods]]"
---
# JS_Array_Methods — 배열 메서드

> [!info] 
> 배열 메서드 = 함수를 인자로 받아, 그 함수를 각 요소에 실행하는 것.
>  `filter` · `map` · `some` · `find` — 전부 이 하나의 구조다.
>  Set은 중복 없는 값의 모음으로 배열과 자주 함께 쓰인다.

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
|`reduce`|값 하나|배열 → 합계·객체·Map|
|`flatMap`|새 배열|map 후 한 단계 펼치기|
|`forEach`|`undefined`|반복 실행 (반환 없음)|
|`sort`|원본 배열|비교 함수로 정렬 (원본 변경 주의)|

---

# 찾기 · 확인 — 존재 여부, 항목 꺼내기

## some · every — boolean 반환 ⭐️⭐️⭐️⭐️

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

const hasDmNotification = dms.some((dm) => dm.unread) || requests.length > 0;
setDmsUnread(hasDmNotification);
```

```txt
Boolean(room.unread):
  room.unread가 number | undefined 타입이면
  boolean 자리에 number를 넣으면 TS 에러
  → Boolean() 또는 !! 로 명시적으로 변환

!isRoomMuted(userId, room.id):
  isRoomMuted()가 true면 ! 로 false → "뮤트가 아닌"
  → [[JS_Operators]] ! · Boolean() · !! 섹션 참고
```

## find · findLast — 항목 하나 꺼내기 ⭐️⭐️⭐️

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

# 걸러내기 · 변환 — 새 배열 만들기

## filter ⭐️⭐️⭐️⭐️

```typescript
const nums   = [1, 2, 3, 4, 5];
const evens  = nums.filter((n) => n % 2 === 0);   // [2, 4]

const active = users.filter((u) => u.isActive);
const admins = users.filter((u) => u.role === 'admin');

// null / undefined 제거
const values  = [1, null, 2, undefined, 3];
const cleaned = values.filter((v) => v != null);  // [1, 2, 3]
// 타입 서술어로 → [[TS_Type_Guards]]
const cleaned = values.filter((v): v is number => v != null);
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
  if (prev.includes(item))       return prev;  // 이미 있으면 그대로
  if (prev.length >= MAX)        return prev;  // 꽉 찼으면 그대로
  return [...prev, item];                      // 여기까지 오면 추가
});
```

```txt
return prev의 의미:
  React는 setState(fn)에서 반환된 값과 이전 값을 Object.is()로 비교
  return prev  → 같은 참조 → "변경 없음" → 리렌더 없음
  return [...prev, item]  → 새 배열 → 다른 참조 → 리렌더 발생

  return [...prev]  ← 내용이 같아도 새 배열 → 리렌더 O
  return prev       ← 참조가 같으면 리렌더 X

  early return 패턴:
  "여기까지 왔으면 위의 조건에 하나도 안 걸린 것" → 안전하게 추가
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
  "이미 있나?"라는 불린 질문 → some이 정확한 선택
  있으면 prev를 그대로 반환 (새 배열 불필요)
  find를 쓰면 항목이 반환되는데 여기선 항목이 필요하지 않음
  filter를 쓰면 새 배열이 반환되는데 여기서도 불필요
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

---

# 조건 함수 (predicate) — return true/false의 의미 ⭐️⭐️⭐️⭐️

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
each return이 뜻하는 것:

  if (!q) return true
    빈 문자열은 falsy → !q = true → 검색어 없음 → 전부 보여줘야 함
    → filter에서 모든 방이 통과

  if (...includes(q)) return true
    이름에서 찾았으면 더 볼 필요 없음 → 즉시 반환 (조기 종료)

  return room.tags.some(...)
    여기까지 온 것 = 이름에는 없었음
    태그 중 하나라도 일치하면 true, 전부 불일치면 false
    이 한 줄이 두 경우를 모두 처리

early return 읽는 법:
  답이 확실한 순간 바로 반환 → 이후 검사를 안 해도 됨
  else 중첩 없이 위에서 아래로 평탄하게 읽힘
  "여기까지 왔으면" = 위의 return들에 걸리지 않은 상황
```

---

# sort — 비교 함수 규칙 ⭐️⭐️⭐️⭐️

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
names.sort((a, b) => a.localeCompare(b, 'ko'));  // 오름차순
```


## localeCompare — 문자열 비교 ⭐️⭐️⭐️⭐️

```typescript
'apple'.localeCompare('banana')     // -1  (apple이 banana보다 앞)
'banana'.localeCompare('apple')     // 1   (banana가 apple보다 뒤)
'apple'.localeCompare('apple')      // 0   (같음)

// sort와 조합
names.sort((a, b) => a.localeCompare(b, 'ko'))  // 오름차순 (가나다순)
names.sort((a, b) => b.localeCompare(a, 'ko'))  // 내림차순
```

```txt
localeCompare란:
  두 문자열을 비교해서 -1 · 0 · 1을 반환
  sort의 비교 함수가 기대하는 음수/0/양수를 그대로 반환
  → sort((a, b) => a.localeCompare(b))로 바로 사용 가능

왜 a - b 대신 localeCompare를 쓰는가:
  a - b는 숫자에서만 작동
  문자열을 < > 로 비교하면 유니코드 코드 포인트 순서 → 한글·특수문자가 이상하게 정렬됨
  localeCompare는 언어 규칙에 맞게 비교 (한국어는 'ko')

두 번째 인자 로케일:
  a.localeCompare(b)        → 브라우저/시스템 언어 기준
  a.localeCompare(b, 'ko')  → 한국어 정렬 규칙 (가나다순)
  a.localeCompare(b, 'en')  → 영어 정렬 규칙 (abc순)
```

## ISO 날짜 문자열 정렬 — localeCompare ⭐️⭐️⭐️⭐️

```typescript
// "YYYY-MM-DD" 또는 "YYYY-MM-DDTHH:mm:ss.sssZ" 형식은
// 문자열 사전순 = 날짜 순서가 일치

messages.sort((a, b) => b.createdAt.localeCompare(a.createdAt));
//                       ↑ b가 앞 → 최신순 (내림차순)

messages.sort((a, b) => a.createdAt.localeCompare(b.createdAt));
//                       ↑ a가 앞 → 오래된 순 (오름차순)
```

```txt
b.createdAt.localeCompare(a.createdAt) 읽는 법:
  localeCompare는 "호출한 것 vs 인자"를 비교
  b.localeCompare(a) → b가 a보다 크면(최신이면) 양수 → b가 앞 → 최신순

  new Date().getTime() 방식과의 차이:
    getTime() 방식  → Date 객체를 두 번 생성 → 약간 느림
    localeCompare   → 문자열 직접 비교 → 더 빠름

  단, ISO 형식이 아닌 날짜 문자열 ("2024년 1월 15일" 등)은 localeCompare로 정렬 불가
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

⚠️ sort는 원본 배열을 직접 바꿈
  React state에서는 [...prev].sort(...)로 복사 후 정렬
```

---

# reduce — 배열을 값 하나로 ⭐️⭐️⭐️

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

// 배열 → Map
const userMap = users.reduce((map, user) => {
  map.set(user.id, user);
  return map;
}, new Map<string, User>());
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
```

## 실전 예 — 배열 + Set 조합

```typescript
// 태그 목록 — 선택/해제 토글
setState((prev) =>
  prev.includes(tag)
    ? prev.filter(t => t !== tag)   // 있으면 제거
    : [...prev, tag]                 // 없으면 추가
);
```

---

# Set — 중복 없는 값의 모음 ⭐️⭐️⭐️⭐️

```txt
Set = 같은 값이 두 번 들어갈 수 없는 배열 같은 자료구조
주로 배열에서 중복 제거 또는 "있냐 없냐" 빠른 확인에 사용
```

```typescript
const set = new Set<string>();

set.add('apple')
set.add('banana')
set.add('apple')   // 이미 있으면 무시
set.size           // 2 (apple 하나만)

set.has('apple')   // true  — O(1) 탐색
set.has('grape')   // false
set.delete('apple')
```

## 배열 ↔ Set 변환 ⭐️⭐️⭐️⭐️

```typescript
// 배열 → Set (중복 제거)
const arr = ['a', 'b', 'a', 'c', 'b'];
const unique = [...new Set(arr)];   // ['a', 'b', 'c']
// 또는
Array.from(new Set(arr))

// Set → 배열
const set = new Set(['x', 'y', 'z']);
const arr2 = [...set];              // ['x', 'y', 'z']
```

## Set.has() vs Array.includes() ⭐️⭐️⭐️

```typescript
// 성능 비교
array.includes(id)  // O(n) — 처음부터 끝까지 순회
set.has(id)         // O(1) — 즉시 확인

// 반복 탐색이 많다면 Set으로 변환 후 사용
const friendIds = new Set(friends.map(f => f.id));

members.map(member => ({
  ...member,
  isFriend: friendIds.has(member.id),  // 매번 O(1)
}));
// friends.some(f => f.id === member.id) 로 하면 매번 O(n) → 전체 O(n²)
```

```txt
언제 Set을 쓰는가:
  배열에서 중복 제거 → [...new Set(arr)]
  "이 값이 목록에 있는가" 를 여러 번 확인할 때 → Set.has()
  순서가 필요 없는 고유 값들의 컬렉션

언제 배열을 쓰는가:
  순서가 중요할 때
  같은 값이 여러 번 나와야 할 때
  index로 접근이 필요할 때
```

## React state에서 Set ⭐️⭐️⭐️

```typescript
// Set을 직접 state로 쓰면 React가 변경 감지 못함
// → 배열로 저장, 필요할 때 Set 변환

const [selectedIds, setSelectedIds] = useState<string[]>([]);

// 토글 — 있으면 제거, 없으면 추가
const toggleId = (id: string) => {
  setSelectedIds(prev =>
    prev.includes(id) ? prev.filter(x => x !== id) : [...prev, id]
  );
};

// 탐색이 많은 경우 — 렌더 중 Set으로 변환해서 사용
const selectedSet = useMemo(() => new Set(selectedIds), [selectedIds]);

return items.map(item => (
  <Item key={item.id} isSelected={selectedSet.has(item.id)} />
));
```

---

# ⚠️ forEach — async와 함께 쓰면 안 됨 ⭐️⭐️⭐️

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