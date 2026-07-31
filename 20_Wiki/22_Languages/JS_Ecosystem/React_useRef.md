---
aliases:
  - current
  - ref
  - useRef
  - 렌더마다 재생성 안 되는 상자
tags:
  - React
related:
  - "[[00_JS_Ecosystem_HomePage]]"
  - "[[JS_DOM]]"
  - "[[React_useMemo_useCallback_useEffect]]"
  - "[[TS_DOM_Events]]"
---
# React_useRef — 렌더마다 재생성 안 되는 상자

> [!info] 
> useRef가 주는 것: 렌더마다 재생성되지 않는 `{ current: 값 }` 상자 하나.
>  `.current`를 바꿔도 리렌더 없음. DOM 접근과 렌더링 무관 값 보관 — 둘 다 이 한 가지 성질에서 나온다.

---

# useRef의 정체 — 왜 필요한가 ⭐️⭐️⭐️⭐️

```txt
함수 컴포넌트는 렌더링할 때마다 함수 전체가 다시 실행됨
→ 안에 선언한 변수는 전부 새로 만들어짐
```

```typescript
function Counter() {
  let count = 0;  // 렌더링마다 0으로 초기화됨 — 이전 값이 사라짐

  function increment() {
    count += 1;   // 값을 바꿔도 리렌더가 안 일어남 → 화면도 안 바뀜
                  // 다음 렌더링 때 또 0으로 초기화
  }
}
```

```txt
두 가지 문제:
  ① 렌더링 사이에 값을 유지하고 싶음 (이전 값이 사라지지 않게)
  ② 값을 바꿔도 리렌더는 일어나지 않았으면 함 (화면에 보이는 값이 아닌 경우)

useState: ①은 해결 (값 유지), ②는 안 됨 (바뀌면 리렌더)
useRef:   ①도 해결, ②도 해결
```

## useRef가 주는 것 — `{ current: 초기값 }` 객체 하나

```typescript
const ref = useRef(0);
// React가 만들어주는 것: { current: 0 }
// 이 객체는 컴포넌트가 언마운트될 때까지 항상 같은 객체임
// 렌더링마다 재생성되지 않음
```

```txt
React는 useRef로 만든 객체를 컴포넌트의 생애 동안 보관함
매 렌더링 때 useRef(0)을 호출해도 React는 처음에 만든 그 객체를 그대로 돌려줌

ref.current = 42 로 값을 바꿔도:
  → React에게 "바뀌었다"고 알리지 않음
  → 리렌더 없음
  → 값은 ref.current 안에 그대로 유지됨
```

## useState와의 차이

```typescript
const [count, setCount] = useState(0);  // 바뀌면 리렌더 발생
const countRef = useRef(0);             // 바뀌어도 리렌더 없음
```

| |`useState`|`useRef`|
|---|---|---|
|값 변경 시 리렌더|✅ 발생|❌ 없음|
|값 읽기|`count`|`countRef.current`|
|값 쓰기|`setCount(n)`|`countRef.current = n`|
|화면에 보여야 할 때|✅|❌ (바꿔도 화면 안 바뀜)|

```txt
판단 기준 한 줄:
  "이 값이 바뀌면 화면도 바뀌어야 하는가?"
    → 예: useState
    → 아니오: useRef
```

---

# 용도 1 — DOM 요소에 직접 접근 ⭐️⭐️⭐️⭐️

```txt
React가 <input ref={inputRef} /> 처럼 ref 속성을 보면
마운트 후에 그 DOM 요소를 자동으로 inputRef.current에 넣어줌

이것도 같은 원리:
  ref 상자(객체)가 렌더링 사이에 유지되므로
  React가 DOM 요소를 거기 넣어두면 언제든 꺼낼 수 있음
```

```typescript
const inputRef = useRef<HTMLInputElement>(null);
//                                        ↑ 초기값 null — 마운트 전엔 DOM이 없음

<input ref={inputRef} />  // React에게 "마운트 후 inputRef.current에 이 요소를 넣어줘"

// 마운트 전:  inputRef.current === null
// 마운트 후:  inputRef.current === <input> DOM 요소
// 언마운트:   inputRef.current === null (자동으로 되돌아감)
```

## 포커스

```typescript
const inputRef = useRef<HTMLInputElement>(null);

function handleOpen() {
  inputRef.current?.focus();  // ?. — null이면 조용히 무시
}

return <input ref={inputRef} />;
```

## 스크롤 — 채팅 맨 아래로 ⭐️⭐️⭐️

```typescript
const bottomRef = useRef<HTMLDivElement>(null);

useEffect(() => {
  bottomRef.current?.scrollIntoView({ behavior: 'smooth' });
}, [messages.length]);  // 메시지 추가될 때마다 실행

return (
  <div className="overflow-y-auto">
    {messages.map((m) => <Message key={m.id} message={m} />)}
    <div ref={bottomRef} />  {/* 항상 맨 아래에 있는 빈 div — 스크롤 타깃 */}
  </div>
);
```

```txt
마지막 메시지 요소에 ref를 달지 않는 이유:
  메시지가 추가될 때마다 ref가 다른 요소로 재연결됨
  항상 맨 아래에 있는 빈 div 하나에 고정 → 더 안정적
```

## e.target vs ref — 언제 ref가 필요한가 ⭐️⭐️⭐️⭐️

```typescript
const inputRef = useRef<HTMLTextAreaElement>(null);

// onChange — 이벤트가 있으므로 e.target 사용 가능 (ref 불필요)
onChange={(e) => {
  setBody(e.target.value);
  e.target.style.height = 'auto';
  e.target.style.height = `${e.target.scrollHeight}px`;
}}

// 전송 후 — 이벤트 없음 → e.target 없음 → ref 필요
async function handleSubmit() {
  await sendMessage(body);
  setBody('');  // state 초기화 → textarea 내용 비워짐

  if (inputRef.current) {
    inputRef.current.style.height = 'auto';  // 높이 초기화 (style은 state가 아님)
    inputRef.current.focus();                // 포커스 복귀
  }
}
```

```txt
이벤트 핸들러 안   → e.target 사용 (이벤트 대상 요소가 넘어옴)
이벤트 밖          → ref 사용 (비동기 이후, 다른 함수에서 접근 등)

setBody('')로 textarea 내용을 비울 수 있지만
style.height는 React state가 아니라 DOM 속성 → ref로 직접 초기화해야 함
```

## 요소별 타입

```typescript
useRef<HTMLInputElement>(null)       // <input>
useRef<HTMLTextAreaElement>(null)    // <textarea>
useRef<HTMLDivElement>(null)         // <div>
useRef<HTMLButtonElement>(null)      // <button>
useRef<HTMLVideoElement>(null)       // <video>
useRef<HTMLCanvasElement>(null)      // <canvas>
useRef<HTMLElement>(null)            // 정확한 타입 모를 때 — 공통 타입
```

---

# 용도 2 — 렌더링과 무관한 값 보관 ⭐️⭐️⭐️⭐️

```txt
DOM과 무관하게, 렌더링 사이에 값을 유지해야 하는데 리렌더를 유발하면 안 되는 경우
```

## 타이머 ID 보관

```typescript
const timerRef = useRef<ReturnType<typeof setTimeout> | null>(null);

function start() {
  timerRef.current = setTimeout(() => doSomething(), 3000);
}

function cancel() {
  if (timerRef.current !== null) {
    clearTimeout(timerRef.current);
    timerRef.current = null;
  }
}

useEffect(() => {
  return () => {
    if (timerRef.current !== null) clearTimeout(timerRef.current);
  };
}, []);
```

```txt
useState가 아닌 이유:
  타이머 ID 자체는 화면에 보여줄 값이 아님
  setState를 하면 불필요한 리렌더가 발생함

ReturnType<typeof setTimeout>:
  브라우저(number)와 Node(NodeJS.Timeout)에서 반환 타입이 다름
  → 직접 적지 않고 타입 추출
```

## 요청 일련번호 — 경쟁 조건 방지

```typescript
const reqIdRef = useRef(0);

useEffect(() => {
  const reqId = ++reqIdRef.current;  // 이 effect 실행 시 번호 캡처

  fetchData().then((data) => {
    if (reqId !== reqIdRef.current) return;  // 더 새로운 요청이 있으면 무시
    setData(data);
  });
}, [query]);
```

```txt
reqIdRef가 ref인 이유:
  reqId 자체는 화면에 안 보임
  바뀌어도 리렌더가 일어나면 안 됨 (리렌더 → effect 재실행 → 무한 루프)

자세한 설명 → [[React_AsyncUI]] reqIdRef 섹션
```

## 드래그 / 그리기 진행 중 데이터 ⭐️⭐️⭐️

```typescript
const drawing = useRef<Stroke | null>(null);  // 현재 그리는 중인 획

const onPointerDown = (e: PointerEvent) => {
  drawing.current = {             // 그리기 시작 — ref에 저장 (리렌더 없음)
    id: Date.now().toString(),
    points: [toNorm(e.clientX, e.clientY)],
  };
};

const onPointerMove = (e: PointerEvent) => {
  if (!drawing.current) return;
  drawing.current.points.push(toNorm(e.clientX, e.clientY));  // ref에 직접 추가
};

const onPointerUp = () => {
  if (!drawing.current) return;
  setStrokes(prev => [...prev, drawing.current!]);  // 완성된 획만 state로
  drawing.current = null;
};
```

```txt
ref에 쌓다가 pointerup 때 state에 한 번만 옮기는 이유:
  pointermove는 초당 수십~수백 번 발생
  매 이벤트마다 setState → 초당 수백 번 리렌더 → 성능 문제

  ref에 쌓다가 완료 시 state에 한 번:
    pointermove → ref에 점 추가 (리렌더 없음)
    pointerup   → state에 추가 (리렌더 1번)
```

---

# ⚠️ 주의사항 ⭐️⭐️⭐️

## 렌더링 중에 ref.current를 읽거나 쓰지 말 것

```typescript
// ❌ 렌더링 중에 직접 접근 — 예측 불가
function Component() {
  const ref = useRef(0);
  ref.current += 1;              // 렌더링 중에 변경
  return <div>{ref.current}</div>;  // 렌더링 중에 읽기
}

// ✅ useEffect 안에서
function Component() {
  const ref = useRef(0);
  useEffect(() => {
    ref.current += 1;  // 렌더링이 끝난 후에 접근
  });
  return <div>...</div>;
}
```

## ref.current를 JSX에 표시해도 자동 업데이트 안 됨

```typescript
// ❌ ref.current가 바뀌어도 화면이 업데이트 안 됨
const countRef = useRef(0);
return <div>{countRef.current}</div>;  // 항상 초기값만 표시

// ✅ 화면에 보여야 하는 값은 useState
const [count, setCount] = useState(0);
return <div>{count}</div>;
```

---

# useRef vs useRouter — 전혀 다른 것 ⭐️⭐️⭐️

```typescript
// useRef — React 내장, 상자를 줌
import { useRef } from 'react';
const inputRef = useRef<HTMLInputElement>(null);

// useRouter — Next.js, 라우팅 객체를 줌
import { useRouter } from 'next/navigation';
const router = useRouter();
router.push('/login');
```

```txt
이름이 비슷해서 헷갈릴 수 있지만 완전히 다른 것:

  useRef    → { current: 값 } 상자 반환. DOM 접근 또는 값 보관용
  useRouter → 페이지 이동(push/replace/back) 기능 반환. 렌더와 무관

  둘 다 use로 시작하고 컴포넌트 최상위에서 쓰는 훅이지만
  목적도 반환하는 것도 완전히 다름
```

---

# 자주 만나는 에러

| 증상                               | 원인                           | 해결                                        |
| -------------------------------- | ---------------------------- | ----------------------------------------- |
| `ref.current is null`            | 마운트 전에 접근 (useEffect 밖)      | `useEffect` 안에서 접근, 또는 `ref.current?.` 사용 |
| 값이 바뀌는데 화면 업데이트 안 됨              | 화면에 보여야 하는 값을 useRef로 관리     | `useState`로 변경                            |
| `Cannot read properties of null` | 초기값을 null로 줬는데 null 체크 없이 접근 | `if (ref.current)` 또는 `ref.current?.` 추가  |