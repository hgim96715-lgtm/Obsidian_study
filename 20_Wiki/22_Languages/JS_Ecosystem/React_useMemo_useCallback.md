---
aliases:
  - useMemo
  - useCallback
  - 메모이제이션
tags:
  - React
related:
  - "[[00_JS_Ecosystem_HomePage]]"
  - "[[React_useRef]]"
  - "[[React_useEffect]]"
---
# React_useMemo_useCallback — 메모이제이션

>[!info]
>메모이제이션 = 이전에 계산한 결과를 기억해서 재사용. 
>`useMemo`는 값을, `useCallback`은 함수를 기억한다. 
>`React.memo`로 감싼 자식에게 함수를 넘길 때, 또는 `useEffect` 의존성에 함수가 들어갈 때 `useCallback`이 필요하다.
> deps를 빠뜨리면 stale closure 버그 발생.

---

# 메모이제이션이란 ⭐️⭐️⭐️⭐️

```txt
React 컴포넌트는 state/props가 바뀔 때마다 함수 전체를 다시 실행
→ 함수 안의 계산, 객체, 배열, 함수 전부 매 렌더마다 새로 만들어짐

대부분은 문제 없음
하지만 두 가지 상황에서 문제가 됨:

① 계산 비용이 클 때
  const result = heavyCalc(data);  // data가 안 바뀌었는데 매번 재계산
  → useMemo로 기억

② 참조가 바뀌면 자식이 재렌더될 때
  const options = { size: 'lg' };  // 매 렌더마다 새 객체 → 참조 바뀜
  const onClick = () => doSomething();  // 매 렌더마다 새 함수 → 참조 바뀜
  → useMemo / useCallback으로 참조 안정화
```

---

# useMemo — 값 메모이제이션 ⭐️⭐️⭐️⭐️

```typescript
const memoizedValue = useMemo(
  () => expensiveCalculation(data),  // 계산 함수
  [data],                            // 의존성 배열 — data가 바뀔 때만 재계산
);
```

```txt
동작:
  첫 렌더 → 계산 실행 → 결과 저장
  이후 렌더 → data 바뀌었나? → 아니면 저장된 값 재사용
                               → 맞으면 재계산 후 저장

의존성 배열:
  [] → 최초 1회만 계산
  [a, b] → a 또는 b가 바뀔 때마다 재계산
  없으면 → 매 렌더마다 계산 (useMemo 의미 없음)
```

## 언제 useMemo ⭐️⭐️⭐️⭐️

```typescript
// ✅ 1. 계산 비용이 클 때
const sortedList = useMemo(
  () => items.sort((a, b) => b.score - a.score),  // items가 많으면 비쌈
  [items],
);

// ✅ 2. 객체/배열을 자식에게 props로 줄 때 (참조 안정화)
const queryOptions = useMemo(
  () => ({ page: 1, size: 20, q: searchQuery }),
  [searchQuery],   // searchQuery가 바뀔 때만 새 객체 생성
);

// ✅ 3. 다른 useMemo/useCallback의 의존성으로 쓸 때
const filteredIds = useMemo(
  () => new Set(items.filter(i => i.active).map(i => i.id)),
  [items],
);

// ❌ 이런 건 useMemo 불필요
const double = useMemo(() => count * 2, [count]);  // 단순 계산은 그냥 쓰면 됨
const double = count * 2;  // ✅ 이게 더 읽기 쉬움
```

---

# useCallback — 함수 메모이제이션 ⭐️⭐️⭐️⭐️

```typescript
const memoizedFn = useCallback(
  () => {
    doSomething(value);
  },
  [value],   // value가 바뀔 때만 새 함수 생성
);
```

```txt
useMemo(() => fn, deps) 와 useCallback(fn, deps) 는 동일:
  useMemo   → 어떤 값이든 메모이제이션
  useCallback → 함수 메모이제이션의 단축형 (가독성을 위한 것)
```

## 언제 useCallback ⭐️⭐️⭐️⭐️

```typescript
// ✅ React.memo로 감싼 자식 컴포넌트에 함수 props로 줄 때
const handleClick = useCallback(() => {
  deleteItem(item.id);
}, [item.id]);

// List.tsx
export const List = React.memo(({ onDelete }) => {
  // handleClick 참조가 안 바뀌면 List가 재렌더 안 됨
  return <button onClick={onDelete}>삭제</button>;
});

// ✅ useEffect 의존성에 함수가 들어갈 때
const fetchData = useCallback(async () => {
  const data = await api.get('/items');
  setItems(data);
}, []);  // 참조가 안정적 → useEffect가 무한루프 안 됨

useEffect(() => {
  void fetchData();
}, [fetchData]);  // fetchData가 안 바뀌면 한 번만 실행
```

```typescript
// ❌ 이런 건 useCallback 불필요
// React.memo 안 쓰는 일반 자식 → 어차피 부모 렌더 시 자식도 렌더
const handleClick = useCallback(() => {
  setCount(c => c + 1);
}, []);

// ✅ 그냥 이렇게
const handleClick = () => setCount(c => c + 1);
```

---

# useCallback이 필요한지 판단 ⭐️⭐️⭐️⭐️

```txt
아래 질문에 하나라도 해당되면 useCallback 고려:

  ① 이 함수를 React.memo로 감싼 자식에게 props로 넘기는가?
  ② 이 함수가 useEffect의 의존성 배열에 들어가는가?
  ③ 이 함수가 다른 useMemo/useCallback의 의존성에 들어가는가?

하나도 해당 없으면 → 그냥 일반 함수로 쓰면 됨

useCallback을 남용하면:
  메모이제이션 자체에도 비용이 있음
  의존성 배열 관리가 복잡해짐
  코드가 불필요하게 길어짐
```

---

# React.memo — 컴포넌트 메모이제이션 ⭐️⭐️⭐️

```typescript
// props가 바뀌지 않으면 재렌더 하지 않는 컴포넌트
export const ExpensiveList = React.memo(function ExpensiveList({ items, onDelete }) {
  return (
    <ul>
      {items.map(item => (
        <li key={item.id} onClick={() => onDelete(item.id)}>{item.name}</li>
      ))}
    </ul>
  );
});
```

```txt
React.memo + useCallback이 세트인 이유:
  React.memo: props가 바뀌지 않으면 재렌더 안 함
  하지만 함수 props는 매 렌더마다 새 참조 → React.memo가 "바뀌었다"고 판단
  → useCallback으로 함수 참조를 안정화해야 React.memo가 효과 있음

  React.memo 없이 useCallback만: 의미 없음
  useCallback 없이 React.memo만: 함수 props 있으면 효과 없음
```

---

# Stale Closure — deps 빠뜨렸을 때 ⭐️⭐️⭐️⭐️

```typescript
// ❌ stale closure — count가 deps에 없음
const handleClick = useCallback(() => {
  console.log(count);  // 항상 초기값 0 출력 (오래된 값을 잡음)
}, []);               // count가 없어서 최초 값으로 고정

// ✅ deps에 count 추가
const handleClick = useCallback(() => {
  console.log(count);  // 최신 count
}, [count]);
```

```txt
stale closure = 오래된 값을 잡고 있는 클로저
  useCallback(() => {...}, [])으로 만든 함수는
  의존성에 없는 값은 최초 생성 시점의 값으로 고정됨

ESLint react-hooks/exhaustive-deps:
  의존성 누락을 자동으로 경고해줌 — 무시하면 stale closure 버그
```

## useRef로 최신 값 유지 ⭐️⭐️⭐️

```typescript
// count가 자주 바뀌는데 handleClick도 계속 새 참조가 되면 자식이 계속 재렌더
// → ref로 최신 값 유지하면서 함수 참조는 안정적으로
const countRef = useRef(count);
useEffect(() => { countRef.current = count; }, [count]);  // 항상 최신값 동기화

const handleClick = useCallback(() => {
  console.log(countRef.current);  // ref로 최신 count 접근
}, []);  // 의존성 없음 → 함수 참조 안정적
```

```txt
언제 이 패턴을 쓰는가:
  콜백이 매우 자주 호출되거나 (마우스 이벤트, 애니메이션 루프)
  deps 변경이 잦아 useCallback이 의미 없어질 때
  일반적인 경우엔 그냥 deps에 넣는 게 더 단순
```

---

# 함께 쓰는 패턴

```typescript
function ParentComponent() {
  const [items, setItems] = useState<Item[]>([]);
  const [query, setQuery] = useState('');

  // ① 필터링 — 비싼 계산 → useMemo
  const filteredItems = useMemo(
    () => items.filter(item => item.name.includes(query)),
    [items, query],
  );

  // ② 이벤트 핸들러 — React.memo 자식에게 전달 → useCallback
  const handleDelete = useCallback((id: string) => {
    setItems(prev => prev.filter(item => item.id !== id));
  }, []);  // setItems는 안정적이라 의존성 불필요

  return (
    <ItemList
      items={filteredItems}    // useMemo로 안정화
      onDelete={handleDelete}  // useCallback으로 안정화
    />
  );
}

// React.memo로 감싼 자식
const ItemList = React.memo(({ items, onDelete }) => {
  // handleDelete 참조가 안 바뀌므로 부모 재렌더에도 이 컴포넌트는 재렌더 안 됨
  return items.map(item => (
    <div key={item.id} onClick={() => onDelete(item.id)}>{item.name}</div>
  ));
});
```