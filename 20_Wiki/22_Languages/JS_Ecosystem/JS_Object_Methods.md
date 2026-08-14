---
aliases:
  - key
  - value
  - 객체 메서드
  - Map
  - values
  - entries
  - keys
  - fromEntries
  - O(1) vs O(n) — 탐색 속도 차이
tags:
  - JavaScript
related:
  - "[[00_JS_Ecosystem_HomePage]]"
  - "[[JS_Array_Methods]]"
---
# JS_Object_Methods — 객체 메서드 · Map

> [!info] 
> Object 메서드 = 객체의 key-value 쌍을 꺼내거나, 합치거나, 변환하는 도구.
>  Map은 같은 key-value 구조지만 동적 키·빠른 탐색이 필요할 때 Object 대신 쓴다.

---

# 객체란 — key-value 쌍의 모음 ⭐️⭐️⭐️

```typescript
const user = {
  id:   1,
  name: '홍길동',
  role: 'admin',
};
// 키(id, name, role)와 값(1, '홍길동', 'admin')의 쌍들
```

```txt
언제 객체를 쓰는가:
  구조가 미리 정해진 데이터 (User, Post, Config 등)
  점 표기법(user.name)으로 편리하게 접근
  JSON 직렬화가 필요할 때

언제 Map을 쓰는가 (아래 Map 섹션 참고):
  키가 런타임에 동적으로 결정될 때
  id → 데이터 처럼 빠른 탐색 테이블이 필요할 때
  키-값 쌍을 자주 추가/삭제할 때
```

---

# 속성 접근 — 점 표기법 vs 대괄호 ⭐️⭐️⭐️

```typescript
const user = { id: 1, name: '홍길동' };

// 점 표기법 — 키를 미리 알 때
user.name       // '홍길동'

// 대괄호 표기법 — 키가 변수일 때
const key = 'name';
user[key]       // '홍길동'
user['name']    // '홍길동' (동일)

// 동적 키 접근
function getValue(obj: Record<string, unknown>, field: string) {
  return obj[field];  // 런타임에 결정되는 키 → 대괄호 필수
}
```

```txt
점 표기법: 자동완성 · 타입 체크 ✅
대괄호: 변수로 키를 결정해야 할 때 ✅

계산된 속성명 (객체 생성 시) → [[JS_Operators]] [key] 섹션
```

---

# Object.keys / values / entries ⭐️⭐️⭐️⭐️

```typescript
const user = { id: 1, name: '홍길동', role: 'admin' };

Object.keys(user)    // ['id', 'name', 'role']         → 키 배열
Object.values(user)  // [1, '홍길동', 'admin']          → 값 배열
Object.entries(user) // [['id',1], ['name','홍길동'], ...] → [키, 값] 쌍 배열
```

```txt
언제 뭘 쓰는가:
  키만 필요할 때    → Object.keys()
  값만 필요할 때    → Object.values()
  둘 다 필요할 때   → Object.entries() + 구조분해

세 메서드 모두:
  반환값이 배열 → map · filter · reduce 등 배열 메서드를 바로 사용 가능
  for...of와 조합해도 됨
```

```typescript
// 실전 — entries로 변환/필터링
const scores = { math: 90, english: 85, science: 78 };

// 값 변환
const doubled = Object.fromEntries(
  Object.entries(scores).map(([k, v]) => [k, v * 2])
);
// { math: 180, english: 170, science: 156 }

// 조건 필터
const passing = Object.fromEntries(
  Object.entries(scores).filter(([, v]) => v >= 80)
);
// { math: 90, english: 85 }
```

```txt
Object.entries(obj).map(([k, v]) => ...) 읽는 법:
  entries → [['math', 90], ['english', 85], ...]
  map(([k, v]) => ...) → 각 [키, 값] 쌍을 구조분해해서 처리
  fromEntries → 다시 객체로 조립
```

---

# Object.fromEntries — 배열/Map → 객체 ⭐️⭐️⭐️

```typescript
// [키, 값] 쌍 배열 → 객체
const pairs = [['name', '홍길동'], ['role', 'admin']] as const;
const obj = Object.fromEntries(pairs);
// { name: '홍길동', role: 'admin' }

// Map → 객체
const map = new Map([['a', 1], ['b', 2]]);
const obj2 = Object.fromEntries(map);
// { a: 1, b: 2 }
```

---

# Object.assign / 스프레드 — 객체 합치기 ⭐️⭐️⭐️

```typescript
const defaults = { theme: 'light', lang: 'ko', size: 'md' };
const overrides = { theme: 'dark' };

// 스프레드 (권장 — 불변)
const config = { ...defaults, ...overrides };
// { theme: 'dark', lang: 'ko', size: 'md' }

// Object.assign (대상 객체를 변경)
Object.assign(defaults, overrides);  // defaults 자체가 바뀜 ⚠️
```

```txt
같은 키가 있으면 나중에 오는 것이 이김:
  { ...defaults, ...overrides }
  → overrides의 theme: 'dark'가 defaults의 theme: 'light'를 덮어씀

스프레드 vs Object.assign:
  스프레드  → 새 객체 반환 (원본 불변) → 권장
  assign   → 첫 번째 인자(대상)를 직접 수정 → React state에서 쓰면 안 됨
```

```typescript
// React state 업데이트 패턴
setState(prev => ({ ...prev, theme: 'dark' }));
// prev를 건드리지 않고 새 객체 반환 → React가 변경 감지 가능
```

---

# Map — 동적 key-value 저장소 ⭐️⭐️⭐️⭐️

```txt
Map = "키로 값을 꺼내는 상자"

Object와 비슷하지만 다른 점:
  Object: 키가 항상 string (또는 Symbol)
  Map:    키가 숫자, 객체 등 어떤 타입이든 가능

언제 Map을 선택하는가:
  키가 숫자일 때     → movie.id(number)로 영화를 찾을 때
  키가 런타임에 결정 → API 응답의 id를 키로
  여러 번 id 탐색    → .find()보다 빠름 (O(1))
  순서 보장 필요     → 삽입 순서 유지

언제 Object를 선택하는가:
  구조가 고정됨      → { title, content, createdAt }
  JSON 직렬화 필요   → Map은 JSON.stringify 지원 안 함
  알려진 키만        → { id, name, email }
```

## TypeScript 제네릭 `new Map<K, V>()` ⭐️⭐️⭐️⭐️

```typescript
// new Map<키 타입, 값 타입>()
const map = new Map<number, GachaMovie>();
//                  ↑ 키     ↑ 값
//   map.set(123, movieObj)  → number 키, GachaMovie 값
//   map.get(123)            → GachaMovie | undefined

const userMap  = new Map<string, User>();
const scoreMap = new Map<string, number>();
const tagMap   = new Map<number, string[]>();  // 키:숫자, 값:문자열 배열
```

```txt
제네릭 <K, V>를 지정하는 이유:
  map.set(123, '잘못된 값')  → TS 에러 (값 타입이 GachaMovie가 아님)
  map.get(123)               → GachaMovie | undefined (타입 안전)

  undefined가 포함되는 이유:
  키가 없으면 .get()은 undefined를 반환
  → 반드시 if (movie) 체크 후 사용
```

## 점진적으로 채우는 패턴 ⭐️⭐️⭐️

```typescript
// 조건에 따라 Map을 점진적으로 채우기
const map = new Map<number, GachaMovie>();

if (today.status === 'used' && today.movie) {
  map.set(today.movie.id, today.movie);
  //           ↑ 키 (number)  ↑ 값 (GachaMovie)
}
if (yesterday.movie) {
  map.set(yesterday.movie.id, yesterday.movie);
}

// 이후 ID로 즉시 조회
const found = map.get(targetId);  // GachaMovie | undefined
if (found) { /* 사용 */ }
```

```txt
왜 배열이 아닌 Map인가:
  배열: [movie1, movie2] → id로 찾으려면 .find(m => m.id === id)
        → 배열이 커질수록 느림 (O(n))

  Map: map.get(id) → 즉시 반환 (O(1))
       나중에 id로 자주 찾아야 할 때 Map이 유리

왜 Object가 아닌 Map인가:
  키가 number → Object 키는 string으로 변환됨
               → map[123]은 실제로 map["123"]
  Map<number, ...>은 진짜 number 키
```


```typescript
// 기본 API
const map = new Map<string, User>();

map.set('user-1', { id: 'user-1', name: '홍길동' });
map.get('user-1')     // { id: 'user-1', name: '홍길동' } — 없으면 undefined
map.has('user-1')     // true
map.delete('user-1')  // 삭제
map.size              // 항목 수
map.clear()           // 전체 삭제

// 초기값으로 생성
const map2 = new Map([
  ['a', 1],
  ['b', 2],
]);
```

## Object vs Map — 언제 뭘 쓰는가 ⭐️⭐️⭐️⭐️

|상황|Object|Map|
|---|---|---|
|구조가 고정됨 (User, Config)|✅|❌ 과함|
|키가 런타임에 결정됨|△ (인덱스 시그니처)|✅|
|JSON 직렬화 필요|✅|❌ 지원 안 함|
|빠른 탐색 (O(1) 조회)|○|✅ 더 최적화|
|순서 보장 필요|❌|✅ 삽입 순서 유지|
|키가 문자열이 아님 (객체, 숫자)|❌|✅|

```typescript
// ❌ Object로 동적 키 관리 — 타입 불안전
const cache: { [key: string]: User } = {};
cache[userId] = user;

// ✅ Map — 더 명확하고 안전
const cache = new Map<string, User>();
cache.set(userId, user);
```

---
## O(1) vs O(n) — 탐색 속도 차이 ⭐️⭐️⭐️⭐️

```txt
O(Big O) = 데이터가 늘어날 때 작업 시간이 얼마나 늘어나는가

O(1) — 상수 시간:
  데이터가 100개든 100만 개든 걸리는 시간이 같음
  Map.get(key) → 키를 직접 해시로 계산해서 즉시 찾음

O(n) — 선형 시간:
  데이터가 n개이면 최악의 경우 n번 확인
  Array.find() → 앞에서부터 하나씩 비교하며 찾음

실제 차이:
  데이터 100개  → find() 최대 100번 비교, Map.get() 1번
  데이터 100만 개 → find() 최대 100만 번 비교, Map.get() 1번
```

```typescript
const users = [/* 100만 개 */];

// O(n) — 매번 처음부터 찾음
users.find(u => u.id === targetId);   // 100만 번 비교할 수 있음

// O(1) — 키로 즉시 찾음
const userMap = new Map(users.map(u => [u.id, u]));
userMap.get(targetId);   // 항상 1번 (해시 계산)
```

```txt
Map이 O(1)인 이유 — 해시 테이블:
  key를 해시 함수로 계산 → 값이 있는 위치(주소)를 바로 계산
  "홍길동"을 찾으려면 → 해시("홍길동") → 위치 → 즉시 반환
  몇 개가 있든 이 계산 과정은 동일

배열이 O(n)인 이유:
  순서대로 저장된 것들을 앞에서부터 하나씩 비교
  운 좋으면 첫 번째에서 찾고, 운 나쁘면 마지막에서 찾음
  평균적으로 n/2번 비교
```

## ID 인덱싱 패턴 — 빠른 탐색 테이블 ⭐️⭐️⭐️⭐️

```typescript
// 배열에서 특정 id 찾기 — O(n): 매번 전체 순회
const user = users.find(u => u.id === targetId);

// Map으로 인덱스 만들기 — O(1): 즉시 찾음
const userMap = new Map(users.map(u => [u.id, u]));
// 또는
const userMap = users.reduce((map, u) => {
  map.set(u.id, u);
  return map;
}, new Map<string, User>());

const user = userMap.get(targetId);  // 즉시 반환
```

```txt
언제 Map 인덱스를 만드는가:
  같은 배열에서 id 기반 탐색을 여러 번 해야 할 때
  users.find()를 루프 안에서 반복하면 O(n²) → Map으로 O(n)으로 개선

  배열 한 번 탐색으로 충분하면 → find/filter
  여러 번 탐색이 필요하면 → Map 인덱스
```

## Map 순회 ⭐️⭐️⭐️

```typescript
const map = new Map([['a', 1], ['b', 2], ['c', 3]]);

// for...of — 가장 기본
for (const [key, value] of map) {
  console.log(key, value);  // 'a' 1, 'b' 2, 'c' 3 (삽입 순서)
}

// 배열로 변환 → 배열 메서드 사용
[...map.entries()]   // [['a',1], ['b',2], ['c',3]]
[...map.keys()]      // ['a', 'b', 'c']
[...map.values()]    // [1, 2, 3]

// Map → Object
Object.fromEntries(map)  // { a: 1, b: 2, c: 3 }

// Object → Map
new Map(Object.entries(obj))
```

---

# WeakMap — 메모리 누수 없는 연결 ⭐️⭐️

```typescript
const cache = new WeakMap<object, ComputedResult>();

function getResult(key: object): ComputedResult {
  if (cache.has(key)) return cache.get(key)!;
  const result = computeExpensive(key);
  cache.set(key, result);
  return result;
}
```

```txt
Map vs WeakMap:
  Map    → 키가 참조되는 한 GC(가비지 컬렉션)가 메모리 해제 안 함
  WeakMap → 키 객체가 다른 곳에서 더 이상 참조되지 않으면 자동 GC

  WeakMap 제약:
    키는 반드시 객체 (문자열, 숫자 불가)
    size 속성 없음 / 순회 불가

  사용하는 경우:
    DOM 요소에 데이터를 연결 (요소가 삭제되면 데이터도 자동 해제)
    라이브러리 내부 캐싱 (외부에서 접근 불가한 private 데이터)
```

---

# 자주 쓰는 패턴 정리

```typescript
// 객체 배열 → Map (ID 인덱싱)
const map = new Map(items.map(item => [item.id, item]));

// Map → 객체
Object.fromEntries(map)

// 객체 → Map
new Map(Object.entries(obj))

// 객체 특정 키만 꺼내기 (pick)
const { id, name } = user;
const picked = { id, name };

// 객체 특정 키 제거 (omit)
const { password, ...rest } = user;  // password 제외한 나머지

// 객체가 비어있는지
Object.keys(obj).length === 0

// 중첩 객체 안전하게 접근
const city = user?.address?.city ?? '주소 없음';
```