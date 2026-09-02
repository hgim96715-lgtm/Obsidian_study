---
aliases: [지연 초기화, state 초기화, state snapshot, useState]
tags: [React]
relations:
  - "[[00_JS_Ecosystem_HomePage]]"
  - "[[React_Component]]"
  - "[[React_useMemo_useCallback]]"
---
# React_useState — 상태 관리

> [!info]
> `useState`는 컴포넌트가 렌더 간에 값을 기억하게 해주는 훅.
> 일반 변수는 렌더마다 초기화되지만, state는 React가 별도로 관리해서 유지된다.
> setState를 호출하면 React가 재렌더를 예약하고, 다음 렌더에서 새 값이 반영된다.

---

# 왜 state가 필요한가 — 일반 변수와의 차이 ⭐️⭐️⭐️⭐️⭐️

```typescript
// ❌ 일반 변수 — 두 가지 문제
function Counter() {
  let count = 0;  // 렌더마다 0으로 초기화됨

  function handleClick() {
    count += 1;   // 값은 바뀌지만 화면은 그대로 (재렌더 없음)
  }

  return <button onClick={handleClick}>{count}</button>;
}

// ✅ state — 두 가지 문제 해결
function Counter() {
  const [count, setCount] = useState(0);

  function handleClick() {
    setCount(count + 1);  // 재렌더 예약 → 다음 렌더에서 count가 1로
  }

  return <button onClick={handleClick}>{count}</button>;
}
```

```txt
일반 변수의 문제:
  ① 렌더 간 유지 안 됨 — 컴포넌트 함수가 다시 실행되면 초기화됨
  ② 변경해도 재렌더 안 됨 — React가 변경을 감지하지 못해 화면 그대로

state가 다른 이유:
  React가 컴포넌트 외부에 state 값을 별도로 보관
  → 렌더가 반복돼도 값이 유지됨
  setState 호출 → React에게 "이 값이 바뀌었으니 다시 그려줘" 신호
  → 재렌더 예약 → 다음 렌더에서 새 state로 화면 업데이트
```

---

# state는 snapshot이다 ⭐️⭐️⭐️⭐️⭐️

```typescript
function Counter() {
  const [count, setCount] = useState(0);

  function handleClick() {
    setCount(count + 1);
    setCount(count + 1);  // count는 여전히 0 — 아직 재렌더 전
    setCount(count + 1);  // 결과: 1 (3이 아님)
  }
}
```

```txt
핵심: 각 렌더의 state는 그 렌더 시점에서 고정된 snapshot

  handleClick이 실행되는 순간의 count = 0 (고정)
  setCount(0 + 1)  → React에 예약
  setCount(0 + 1)  → React에 예약 (여전히 count=0으로 계산)
  setCount(0 + 1)  → React에 예약

  React는 배칭(batching)으로 한 번에 처리
  → 마지막 setCount(0 + 1) = 1이 최종 값

  이전 값 기반으로 여러 번 업데이트하려면 prev 패턴 사용:
```

```typescript
// ✅ prev 패턴 — 최신 값 기반으로 누적
function handleClick() {
  setCount(prev => prev + 1);  // prev = 0 → 1
  setCount(prev => prev + 1);  // prev = 1 → 2
  setCount(prev => prev + 1);  // prev = 2 → 3
}
// 결과: 3 ✅
```

```txt
setState는 즉시 반영되지 않는다:
  setCount(5) 다음 줄에서 count를 읽어도 여전히 이전 값
  → 변경은 다음 렌더에서 반영됨

  // ❌ 이런 실수
  setCount(count + 1);
  console.log(count);  // 여전히 이전 값 출력

  // ✅ 로컬 변수로 먼저 계산
  const next = count + 1;
  setCount(next);
  console.log(next);  // 올바른 값
```

---

# 기본 사용법 ⭐️⭐️⭐️⭐️

```typescript
const [state, setState] = useState(initialValue);
// state     — 현재 렌더의 snapshot 값
// setState  — 다음 렌더를 예약하는 함수
```

```typescript
// 원시값
const [count,  setCount]  = useState(0);
const [name,   setName]   = useState('');
const [isOpen, setIsOpen] = useState(false);

// 객체
const [user, setUser] = useState<{ name: string; age: number }>({ name: '', age: 0 });

// 배열
const [items, setItems] = useState<string[]>([]);

// null 허용 (API 응답 등)
const [data, setData] = useState<User | null>(null);
```

---

# 불변성 — mutation이 안 되는 이유 ⭐️⭐️⭐️⭐️⭐️

```txt
React가 state 변경을 감지하는 방법: Object.is() 비교 (얕은 비교)

  Object.is(old, new)
  원시값: 값이 같으면 동일 → true = 변경 없음, 재렌더 스킵
  객체/배열: 참조가 같으면 동일 → 내용이 바뀌어도 같은 참조면 재렌더 안 함

  // ❌ mutation — 같은 객체 참조를 그대로 넘김
  const arr = items;
  arr.push('새 항목');   // 내용은 바뀌었지만
  setItems(arr);         // 참조가 같음 → Object.is(old, new) = true → 재렌더 안 됨

  // ✅ 새 참조 생성
  setItems([...items, '새 항목']);  // 새 배열 → 참조 다름 → 재렌더 됨
```

## 객체 업데이트 패턴

```typescript
// ✅ spread로 새 객체 생성
setUser(prev => ({ ...prev, name: '공이' }));

// ✅ 중첩 객체
setUser(prev => ({
  ...prev,
  address: { ...prev.address, city: 'Seoul' },
}));

// ❌ mutation — 재렌더 안 됨
user.name = '공이';
setUser(user);  // 같은 참조
```

## 배열 업데이트 패턴

```typescript
// 추가
setItems(prev => [...prev, newItem]);

// 삭제
setItems(prev => prev.filter(item => item.id !== id));

// 수정
setItems(prev => prev.map(item =>
  item.id === id ? { ...item, name: '수정됨' } : item
));

// ❌ mutation
items.push(newItem);   // 재렌더 안 됨
items[0].name = '수정'; // 재렌더 안 됨
```

---

# setState 함수형 업데이트 — prev 패턴 ⭐️⭐️⭐️⭐️

```typescript
// 언제 prev를 써야 하는가

// ① 연속 setState 호출 시
setCount(prev => prev + 1);  // 누적 보장

// ② 비동기 컨텍스트 (setTimeout, fetch 콜백 등)
setTimeout(() => {
  setCount(prev => prev + 1);  // ✅ 최신 값 기반
  // setCount(count + 1);       // ❌ 클로저에 잡힌 오래된 count
}, 1000);

// ③ 배열/객체 — 항상 prev 패턴 권장
setItems(prev => [...prev, newItem]);  // ✅
setItems([...items, newItem]);          // ⚠️ items가 최신인지 보장 안 됨
```

---

# state를 쪼갤까, 합칠까 ⭐️⭐️⭐️

```txt
함께 바뀌는 값은 하나의 객체로:
  const [position, setPosition] = useState({ x: 0, y: 0 });
  → x, y는 항상 같이 움직이므로 묶는 게 자연스러움

독립적으로 바뀌는 값은 분리:
  const [isLoading, setIsLoading] = useState(false);
  const [error,     setError]     = useState<string | null>(null);
  const [data,      setData]      = useState<User | null>(null);
  → 각각 독립적으로 바뀜

파생 값은 state에 넣지 말 것:
  const [items, setItems]         = useState<Item[]>([]);
  const [count, setCount]         = useState(0);  // ❌ items.length와 동기화 어려움
  const count = items.length;                      // ✅ 파생 값 — 그냥 계산
  const count = useMemo(() => items.length, [items]); // 비싼 계산이면 useMemo
```

---

# 지연 초기화 — () => 함수 전달 ⭐️⭐️⭐️⭐️

```txt
useState(initialValue)에 값을 직접 넘기면
컴포넌트가 렌더링될 때마다 initialValue 표현식이 실행됨
→ 첫 렌더에만 쓰고 이후에는 무시되는데 계속 실행되는 낭비
```

```typescript
// ❌ 매 렌더마다 new Set() 실행
const [ids, setIds] = useState(new Set<string>());

// ✅ 마운트 시 딱 한 번만 실행
const [ids, setIds] = useState<Set<string>>(() => new Set());
```

```typescript
// new Set / new Map
const [expandedIds, setExpandedIds] = useState<Set<string>>(() => new Set());
const [cache, setCache]             = useState<Map<string, unknown>>(() => new Map());

// 무거운 계산
const [data, setData] = useState(() => expensiveCalculation());

// localStorage — SSR 주의
const [theme, setTheme] = useState<'light' | 'dark'>(() => {
  if (typeof window === 'undefined') return 'light';
  return (localStorage.getItem('theme') as 'light' | 'dark') ?? 'light';
});

// 단순 원시값은 그냥 넘겨도 됨
const [count, setCount] = useState(0);   // ✅
const [name, setName]   = useState('');  // ✅
```
