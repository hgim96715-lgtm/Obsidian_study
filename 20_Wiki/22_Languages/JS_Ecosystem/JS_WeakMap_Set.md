---
aliases: [자료구조, 중복제거, Map, Set, WeakMap, WeakSet]
tags:
  - JavaScript
related:
  - "[[00_JS_Ecosystem_HomePage]]"
  - "[[JS_Array_Methods]]"
  - "[[JS_Date]]"
  - "[[React_DatePicker]]"
---


# JS_WeakMap_Set — Map / Set / WeakMap / WeakSet

# 한 줄 요약

```txt
Set      중복 없는 "값" 모음 (배열 중복 제거에 자주 씀)
Map      key-value 저장 (객체보다 자유로운 key, 순서 보장)
WeakSet / WeakMap  Set/Map 과 거의 같지만, 참조가 사라지면 자동으로 메모리 정리됨
```

---

---

# Set — 중복 없는 값 모음 ⭐️

## 기본 사용

```javascript
const set = new Set();

set.add(1);
set.add(2);
set.add(1);          // 이미 있음 → 무시됨

set.size             // 2 (중복 제거됨)
set.has(1)            // true
set.delete(1);
set.has(1)            // false
```

| 메서드             | 의미                     |
| --------------- | ---------------------- |
| `add(value)`    | 값 추가                   |
| `has(value)`    | 값이 있는지 확인 (true/false) |
| `delete(value)` | 값 제거                   |
| `size`          | 들어있는 값의 개수             |

```javascript
// 배열로 바로 생성 — 가장 흔한 사용법
const arr = [1, 2, 2, 3, 3, 3];
const unique = new Set(arr);
[...unique]             // [1, 2, 3]
// 또는
Array.from(new Set(arr))
```

## 순회

```javascript
const set = new Set(['a', 'b', 'c']);

for (const value of set) { console.log(value); }
set.forEach(value => console.log(value));
```

## ⚠️ Set 은 "참조"로 중복을 판단함 (객체/Date 주의) ⭐️

```javascript
new Set([1, 1, 2]).size          // 2  ← 원시값은 값 자체로 비교됨

new Set([{}, {}]).size           // 2  ← 객체는 참조로 비교 → 다른 객체로 취급
new Set([new Date('2026-06-16'), new Date('2026-06-16')]).size
// → 2  ← 같은 날짜를 가리켜도 "다른 Date 인스턴스" → 중복 제거 안 됨!
```

```bash
이유: Set 은 SameValueZero 비교 알고리즘을 씀
  원시값(number, string)  → 값 자체로 비교 (1 === 1)
  객체(object, Date 포함) → 참조로 비교 ({} === {} → false)
  # → [[React_useMemo_useCallback]] 의 "원시값 vs 참조값" 과 정확히 같은 원리

그래서 Date 를 직접 Set 에 넣어 "같은 날" 을 걸러내려 하면 실패함
→ 해결: Date 대신 "날짜를 나타내는 문자열(YYYY-MM-DD)" 을 Set 의 기준으로 사용
```

## 실전 — 같은 날짜 중복 제거 (visitedDates 패턴) ⭐️

```typescript
// Artinerary 관람 기록 — 같은 날 여러 전시를 봤어도 달력엔 점 하나만
const visitedDates = useMemo(() => {
  const seen = new Set<string>();      // ← 날짜 "문자열" 을 기준으로 중복 체크
  const dates: Date[] = [];

  for (const visit of visits) {
    const key = visit.visitedAt.slice(0, 10);   // 'YYYY-MM-DD' 문자열만 추출
    if (seen.has(key)) continue;                // 이미 처리한 날짜면 건너뜀
    seen.add(key);
    dates.push(toVisitDate(visit.visitedAt));    // 실제로 쓸 Date 는 따로 변환해서 push
  }
  return dates;
}, [visits]);
```

```txt
왜 Set<string> 이고 Set<Date> 가 아닌지:
  visit.visitedAt 는 ISO 문자열 ('2026-06-18T03:00:00.000Z')
  slice(0, 10) 으로 'YYYY-MM-DD' 부분만 뽑으면 → 원시값(string)
  → 원시값은 Set 이 "값"으로 비교하므로 같은 날짜 문자열은 정확히 중복 제거됨

  반대로 Date 객체를 바로 Set 에 넣었다면
  → 위에서 본 것처럼 "참조"로 비교되어 중복 제거가 안 됐을 것

흐름:
  1. seen (문자열 Set) 으로 "이 날짜를 이미 처리했는지" 빠르게 확인 (has 는 O(1))
  2. 처음 보는 날짜면 seen 에 추가 + dates 배열에 실제 Date 를 push
  3. 이미 처리한 날짜면 continue 로 건너뜀 → 같은 날 여러 전시가 있어도 한 번만 push

  Set 없이 visits.map(v => toVisitDate(v.visitedAt)) 만 하면:
  → 같은 날짜가 여러 번 들어간 배열이 됨 (중복 제거 안 됨)
  # → 달력에 같은 날짜의 modifier 가 중복 적용됨 (자세한 사용처는 [[React_DatePicker]] 참고)

seen 이라는 이름:
  "이미 본(seen) 항목들" 이라는 의미의 흔한 관용적 변수명
  → 중복 체크용 Set 에 자주 붙이는 이름일 뿐, 꼭 이 이름이어야 하는 건 아님
```

---

---

# Map — key-value 저장 ⭐️

## 기본 사용

```javascript
const map = new Map();

map.set('a', 1);
map.set('b', 2);

map.get('a')     // 1
map.has('a')     // true
map.delete('a');
map.size         // 1

// 생성자에 배열로 바로 초기화
const map2 = new Map([
  ['a', 1],
  ['b', 2],
]);
```

| 메서드               | 의미                               |
| ----------------- | -------------------------------- |
| `set(key, value)` | key-value 추가 (이미 있는 key 면 값 덮어씀) |
| `get(key)`        | key 로 value 조회 (없으면 undefined)   |
| `has(key)`        | key 가 있는지 확인 (true/false)        |
| `delete(key)`     | key 제거                           |
| `size`            | 들어있는 key-value 쌍의 개수             |

## Map vs 일반 Object — 언제 Map 을 쓰나

```txt
Object                            Map
key 가 string / Symbol 만 가능      key 로 객체 / 함수 / Date 등 아무 값이나 가능
순서 보장이 명세상 핵심은 아님        삽입 순서 그대로 보장
size 속성 없음 (Object.keys 필요)    map.size 로 바로 개수 확인
프로토타입 체인과 섞여 있음           순수한 key-value 저장소

Map 이 유리한 경우:
  key 가 객체/함수여야 할 때 (예: 컴포넌트 인스턴스별 데이터)
  key-value 쌍이 자주 추가/삭제될 때
  순서 보장이 중요한 경우
```

## 순회

```javascript
const map = new Map([['a', 1], ['b', 2]]);

for (const [key, value] of map) { console.log(key, value); }
map.forEach((value, key) => console.log(key, value));

[...map.keys()]     // ['a', 'b']
[...map.values()]   // [1, 2]
[...map.entries()]  // [['a', 1], ['b', 2]]
```

## 실전 — 카운팅 패턴 ⭐️

```typescript
// 전시별 관람 횟수 집계
function countByExhibition(visits: Visit[]) {
  const counts = new Map<number, number>();

  for (const v of visits) {
    counts.set(v.exhibitionId, (counts.get(v.exhibitionId) ?? 0) + 1);
    //                          ↑ 없으면 0, 있으면 현재 값 +1
  }
  return counts;
}
```

---

---

# WeakMap / WeakSet — 약한 참조 ⭐️

```txt
일반 Map/Set 의 문제:
  map.set(someObject, data) 로 객체를 key 로 저장하면
  → map 이 그 객체를 계속 참조하게 됨
  → someObject 를 다른 곳에서 다 안 써도, map 에 남아있는 한 GC(가비지 컬렉션) 안 됨
  → 의도치 않은 메모리 누수 가능

WeakMap / WeakSet 은:
  key(WeakMap) 또는 값(WeakSet) 으로 "객체만" 허용
  그 객체가 다른 곳에서 더는 참조되지 않으면 → GC 대상이 됨 (Map 에 남아있어도 자동 정리)
```

```javascript
const cache = new WeakMap();

function getMetadata(domNode) {
  if (!cache.has(domNode)) {
    cache.set(domNode, { clicks: 0 });
  }
  return cache.get(domNode);
}

// domNode 가 화면에서 제거되고 다른 곳에서 참조도 안 하면
// → cache 안의 항목도 자동으로 GC 됨 (직접 delete 안 해도 됨)
```

## 일반 Map/Set 과의 차이 ⭐️

```txt
WeakMap / WeakSet 은:
  key(또는 값)로 객체만 가능 (원시값 X)
  순회 불가 — forEach, for...of, size 없음
    (객체가 언제 GC 될지 알 수 없어서, 순회 자체가 의미 없음)
  has / get / set / delete 만 가능

→ "이 객체에 대한 부가 데이터를 저장하고 싶은데,
   객체가 사라지면 그 데이터도 같이 사라져야 한다" 는 상황에 적합
```

---

---

# 한눈에

| |Set|Map|WeakSet|WeakMap|
|---|---|---|---|---|
|저장|값만|key-value|값만 (객체만)|key-value (key 는 객체만)|
|중복|허용 안 함|key 중복 허용 안 함|허용 안 함|key 중복 허용 안 함|
|비교 방식|SameValueZero (객체는 참조 비교)|key 도 동일|참조 비교|key 참조 비교|
|순회|가능|가능|불가능|불가능|
|size|있음|있음|없음|없음|
|GC|일반 참조 (메모리에 계속 남음)|일반 참조|약한 참조 (자동 정리)|약한 참조 (자동 정리)|

```txt
배열 중복 제거           → [...new Set(arr)]
객체/Date 중복 제거       → 원시값(문자열 등)으로 변환 후 Set 사용 ⭐️
key-value 저장           → Map (key 가 객체/함수일 수도 있을 때 특히 유리)
객체별 부가 데이터 + 메모리 누수 방지 → WeakMap
```