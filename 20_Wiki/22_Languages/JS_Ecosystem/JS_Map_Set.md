---
aliases: [그루핑, Map, new Map, new Set, Set]
tags: [JavaScript, TypeScript]
relations:
  - "[[00_JS_Ecosystem_HomePage]]"
  - "[[React_Calendar]]"
  - "[[React_useMemo_useCallback]]"
  - "[[React_useState]]"
---
# JS_Map_Set — Map과 Set

> [!info]
> `Map` — 키-값 쌍 저장. 일반 객체 `{}`와 비슷하지만 키 타입이 자유롭고 O(1) 조회가 보장됨.
> `Set` — 중복 없는 값 집합. 배열에서 특정 값 존재 여부를 O(1)로 확인할 때 사용.
> "언제 `{}`를 쓰고 언제 `Map`을 쓰는가"가 핵심 판단 기준.

---

# Map vs 일반 객체 `{}` — 언제 뭘 쓰는가 ⭐️⭐️⭐️⭐️⭐️

| 기준 | 일반 객체 `{}` | `Map` |
|---|---|---|
| 키 타입 | string·Symbol만 | 모든 타입 (숫자, 객체, Date 등) |
| 키 동적 생성 | 가능하지만 타입 불안정 | 자연스러움 |
| 조회 | `obj[key]` — O(1) | `map.get(key)` — O(1) |
| 존재 확인 | `key in obj` (프로토타입 포함) | `map.has(key)` (정확함) |
| 크기 | `Object.keys(obj).length` | `map.size` |
| 순서 | 삽입 순서 보장 안 됨 (숫자 키 먼저) | 삽입 순서 항상 보장 |
| 이터레이션 | `Object.entries(obj)` | `map.forEach` · `for...of map` |
| 직렬화 | `JSON.stringify` 가능 | 직접 변환 필요 |

```txt
일반 객체 {} 를 쓰는 경우:
  ① 키가 컴파일 시점에 고정된 정적 구조 (DTO, config, enum-like)
     const user = { name: '공이', age: 28 };
  ② JSON으로 직렬화해서 API에 보낼 때
  ③ TypeScript interface/type으로 타입 정의가 이미 있을 때

Map을 쓰는 경우:
  ① 키가 런타임에 동적으로 생성됨 (날짜 문자열, DB에서 온 ID 등)
  ② 키 타입이 string이 아님 (숫자, Date, 객체)
  ③ 특정 키가 존재하는지 자주 확인해야 함 (has)
  ④ 삽입 순서가 중요한 경우
  ⑤ 크기를 자주 확인해야 하는 경우 (.size)
```

---

# Map 기본 API ⭐️⭐️⭐️⭐️

```typescript
const map = new Map<string, number>();

// 추가·수정
map.set('apple', 3);
map.set('banana', 5);

// 조회
map.get('apple');    // 3
map.get('cherry');   // undefined (없으면 undefined)

// 존재 확인
map.has('apple');    // true
map.has('cherry');   // false

// 삭제
map.delete('apple'); // true (삭제 성공)

// 크기
map.size;  // 1

// 전체 삭제
map.clear();

// 이터레이션
for (const [key, value] of map) { ... }
map.forEach((value, key) => { ... });

// 배열로 변환
Array.from(map.entries());  // [[key, value], ...]
Array.from(map.keys());     // [key, ...]
Array.from(map.values());   // [value, ...]
```

---

# Map<string, T[]> — 그루핑 패턴 ⭐️⭐️⭐️⭐️⭐️

```txt
배열을 특정 기준(날짜, 카테고리, 상태)으로 묶어야 할 때
→ 배열을 순회하면서 각 항목을 키별로 모아 Map에 쌓는 패턴
```

```typescript
type Item = { date: string; title: string };

const items: Item[] = [
  { date: '2025-01-15', title: '영화 A' },
  { date: '2025-01-15', title: '영화 B' },
  { date: '2025-01-20', title: '영화 C' },
];

// ✅ Map 그루핑
const grouped = new Map<string, Item[]>();

for (const item of items) {
  const list = grouped.get(item.date) ?? [];  // 없으면 빈 배열
  list.push(item);
  grouped.set(item.date, list);
}

// 결과:
// Map {
//   '2025-01-15' → [{ date, title: '영화 A' }, { date, title: '영화 B' }],
//   '2025-01-20' → [{ date, title: '영화 C' }],
// }

// 조회
grouped.get('2025-01-15');  // [영화 A, 영화 B]
grouped.get('2025-01-20');  // [영화 C]
grouped.get('2025-01-99');  // undefined
```

```txt
grouped.get(key) ?? []:
  Map에 해당 키가 없으면 get()은 undefined 반환
  ?? [] — nullish coalescing: undefined이면 빈 배열로 대체
  list.push(item) 후 grouped.set(key, list) 로 다시 저장 필수
  → push는 배열 내용을 바꾸지만, Map에는 참조가 이미 들어있어서
    사실 set을 다시 안 해도 동작하긴 하지만 명시적으로 set하는 게 안전

왜 {} 대신 Map인가:
  { [key: string]: Item[] } 로도 가능하지만:
    타입이 넓어짐 (Record<string, Item[]> 로 좁힐 수 있음)
    in 연산자가 프로토타입 체인까지 검색
    Map은 has/get이 정확하고 타입 안전

React에서 useMemo와 함께:
  달력처럼 렌더마다 조회가 발생하는 경우 → useMemo로 한 번만 생성
  (→ [[React_Calendar]] 참고)
```

## reduce로 그루핑 (대안)

```typescript
// reduce 버전 — 한 줄이지만 가독성 낮음
const grouped = items.reduce((map, item) => {
  const list = map.get(item.date) ?? [];
  list.push(item);
  return map.set(item.date, list);
}, new Map<string, Item[]>());

// Object.groupBy — ES2024 (최신 환경만)
const grouped = Object.groupBy(items, item => item.date);
// 결과는 일반 객체 {} (Map 아님)
```

---

# Map 초기화 패턴 ⭐️⭐️⭐️

```typescript
// 빈 Map
const map = new Map<string, number>();

// 초기값으로 생성
const map = new Map<string, number>([
  ['apple',  3],
  ['banana', 5],
]);

// 배열 → Map 변환
const entries: [string, number][] = [['a', 1], ['b', 2]];
const map = new Map(entries);

// 객체 → Map
const obj = { apple: 3, banana: 5 };
const map = new Map(Object.entries(obj));

// Map → 객체 (JSON 직렬화가 필요할 때)
const obj = Object.fromEntries(map);
```

---

# Set — 중복 없는 집합 ⭐️⭐️⭐️⭐️

```typescript
const set = new Set<string>();

set.add('apple');
set.add('banana');
set.add('apple');  // 중복 — 무시됨

set.size;          // 2
set.has('apple');  // true — O(1)
set.delete('apple');
set.has('apple');  // false

// 배열 → Set (중복 제거)
const arr = ['a', 'b', 'a', 'c'];
const unique = new Set(arr);          // Set {'a', 'b', 'c'}
const uniqueArr = [...new Set(arr)];  // ['a', 'b', 'c']

// Set → 배열
Array.from(set);
[...set];
```

## Set을 쓰는 경우

```typescript
// ✅ 1. 중복 없는 선택 상태 (체크박스 등)
const [selectedIds, setSelectedIds] = useState<Set<string>>(() => new Set());

function toggle(id: string) {
  setSelectedIds(prev => {
    const next = new Set(prev);  // 불변성: 새 Set 생성
    if (next.has(id)) next.delete(id);
    else next.add(id);
    return next;
  });
}

// ✅ 2. 빠른 존재 확인 — 배열.includes() 대신
const allowedIds = new Set(['id1', 'id2', 'id3']);
allowedIds.has('id1');  // O(1)
// vs
['id1', 'id2', 'id3'].includes('id1');  // O(n)

// ✅ 3. 중복 제거
const tags = ['react', 'js', 'react', 'ts'];
const uniqueTags = [...new Set(tags)];  // ['react', 'js', 'ts']
```

```txt
Set 불변성 업데이트 (React state):
  const next = new Set(prev)  — 기존 Set을 복사해서 새 Set 생성
  → React는 참조가 바뀌어야 재렌더 트리거
  prev.add(id) 직접 호출 후 setSelectedIds(prev) → 재렌더 안 됨 ❌

배열 vs Set 선택 기준:
  순서가 중요하거나 중복이 필요 → 배열
  존재 여부 확인이 빈번하거나 중복 없어야 → Set
```

---

# TypeScript 타입 ⭐️⭐️⭐️

```typescript
// Map 타입
const map1: Map<string, number>    = new Map();
const map2: Map<string, string[]>  = new Map();
const map3: Map<number, User>      = new Map();

// Set 타입
const set1: Set<string>  = new Set();
const set2: Set<number>  = new Set();

// 읽기 전용
const readMap: ReadonlyMap<string, number> = new Map([['a', 1]]);
const readSet: ReadonlySet<string>         = new Set(['a', 'b']);
// readMap.set(...)  → ❌ 타입 에러

// 함수 파라미터 — 읽기만 할 때
function process(map: ReadonlyMap<string, Item[]>) { ... }
```
