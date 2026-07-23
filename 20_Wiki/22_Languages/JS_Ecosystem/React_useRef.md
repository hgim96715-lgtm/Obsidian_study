---
aliases:
  - current
  - ref
  - useRef
tags:
  - React
related:
  - "[[00_JS_Ecosystem_HomePage]]"
  - "[[React_useMemo_useCallback_useEffect]]"
  - "[[TS_DOM_Events]]"
  - "[[JS_DOM]]"
  - "[[TS_Utility_Types]]"
---
# React_useRef — DOM 접근 & 렌더링 무관 값 보관

> [!info]
>  useRef 는 "렌더링에 영향을 주지 않으면서 값을 유지하는 상자." 
>  useState 와 달리 `.current` 를 바꿔도 리렌더가 일어나지 않는다.
>   주된 용도: ① DOM 요소에 직접 접근 ② 타이머 ID·이전 값처럼 렌더링과 무관한 값 보관.

---

# useState vs useRef ⭐️⭐️⭐️⭐️

```typescript
const [count, setCount] = useState(0);  // 바뀌면 리렌더 O
const countRef = useRef(0);             // 바뀌어도 리렌더 X
```

| |`useState`|`useRef`|
|---|---|---|
|값 변경 시 리렌더|✅ 리렌더 발생|❌ 리렌더 없음|
|값 읽기|`count`|`countRef.current`|
|값 쓰기|`setCount(n)`|`countRef.current = n`|
|렌더링 결과에 반영|✅ JSX에 표시됨|❌ `.current` 를 JSX에 써도 업데이트 안 됨|

```txt
"화면에 보여야 하는 값" → useState
"내부에서만 쓰고 화면에 안 보이는 값" → useRef
```

---

# 기본 형태

```typescript
import { useRef } from 'react';

const ref = useRef(초기값);

ref.current          // 현재 값 읽기
ref.current = 새값   // 값 쓰기 (리렌더 없음)
```

---

# 용도 1 — DOM 요소에 직접 접근 ⭐️⭐️⭐️⭐️

```typescript
// useRef<타입>(null) — 초기값은 null (마운트 전에 DOM이 없음)
const inputRef = useRef<HTMLInputElement>(null);
const divRef   = useRef<HTMLDivElement>(null);

// JSX에서 ref 속성으로 연결
<input ref={inputRef} />
<div ref={divRef} />
```

```txt
React가 요소를 렌더링한 뒤에 ref.current에 그 DOM 요소를 자동으로 넣어줌
마운트 전:  ref.current = null
마운트 후:  ref.current = 실제 DOM 요소 (HTMLInputElement 등)
언마운트 후: ref.current = null 으로 되돌아감
```

## 포커스

```typescript
const inputRef = useRef<HTMLInputElement>(null);

function handleOpen() {
  // DOM이 준비된 이후에만 실행
  inputRef.current?.focus();
}

return <input ref={inputRef} />;
```

## 스크롤 — 채팅 맨 아래로 ⭐️⭐️⭐️⭐️

```typescript
const bottomRef = useRef<HTMLDivElement>(null);

useEffect(() => {
  // messages가 추가될 때마다 맨 아래로 스크롤
  bottomRef.current?.scrollIntoView({ behavior: 'smooth' });
}, [messages.length]);

return (
  <div className="overflow-y-auto">
    {messages.map((m) => <Message key={m.id} message={m} />)}
    <div ref={bottomRef} />  {/* 스크롤 타깃 — 항상 맨 아래에 있는 빈 div */}
  </div>
);
```

```txt
빈 div를 ref 타깃으로 쓰는 이유:
  마지막 메시지에 ref를 달면 메시지가 바뀔 때마다 ref가 재연결됨
  항상 맨 아래에 있는 빈 div 하나를 두고 그것만 참조 → 더 안정적

ref.current?.scrollIntoView:
  ?. (옵셔널 체이닝) — null이면 조용히 무시
  scrollIntoView 옵션 → [[JS_DOM]] 참고
```

## 타입 — 요소별 타입

```typescript
useRef<HTMLInputElement>(null)       // <input>
useRef<HTMLTextAreaElement>(null)    // <textarea>
useRef<HTMLDivElement>(null)         // <div>
useRef<HTMLButtonElement>(null)      // <button>
useRef<HTMLFormElement>(null)        // <form>
useRef<HTMLVideoElement>(null)       // <video>
useRef<HTMLCanvasElement>(null)      // <canvas>

// 모르면 → HTMLElement (공통 타입)
useRef<HTMLElement>(null)
```

---

# 용도 2 — 렌더링과 무관한 값 보관 ⭐️⭐️⭐️⭐️

## 타이머 ID 보관

```typescript
const timerRef = useRef<ReturnType<typeof setTimeout> | null>(null);

function start() {
  timerRef.current = setTimeout(() => {
    doSomething();
  }, 3000);
}

function cancel() {
  if (timerRef.current !== null) {
    clearTimeout(timerRef.current);
    timerRef.current = null;
  }
}

// 언마운트 시 타이머 정리
useEffect(() => {
  return () => {
    if (timerRef.current !== null) clearTimeout(timerRef.current);
  };
}, []);
```

```txt
왜 useState가 아니라 useRef인가:
  timerRef.current = setTimeout(...)  → 상태 변경이 아님, 리렌더 필요 없음
  useState로 하면 clearTimeout 직전에 불필요한 리렌더 발생

ReturnType<typeof setTimeout>:
  환경(브라우저/Node)에 따라 반환 타입이 다름 → 직접 적지 않고 추출
  → [[TS_Utility_Types]] 참고
```

## 진행 중인 작업 임시 보관 — 드래그 / 그리기 ⭐️⭐️⭐️⭐️

```typescript
const stageRef = useRef<HTMLDivElement>(null);    // DOM ref — 이벤트 대상
const drawing  = useRef<LyricStroke | null>(null); // 현재 그리는 중인 스트로크

const onPointerDown = (e: ReactPointerEvent<HTMLDivElement>) => {
  e.currentTarget.setPointerCapture(e.pointerId);
  const p = toNorm(e.clientX, e.clientY);
  drawing.current = {           // 그리기 시작 — ref에 저장 (리렌더 없음)
    id:     `${uid}-stroke-${Date.now()}`,
    color:  penColor,
    width:  penWidth,
    points: [p],
  };
};

const onPointerMove = (e: ReactPointerEvent<HTMLDivElement>) => {
  if (!drawing.current) return;
  const p = toNorm(e.clientX, e.clientY);
  drawing.current.points.push(p);  // ref에 직접 추가 (리렌더 없음)
};

const onPointerUp = () => {
  if (!drawing.current) return;
  setStrokes(prev => [...prev, drawing.current!]);  // 완성된 획 → state로
  drawing.current = null;                            // ref 리셋
};
```

```txt
drawing.current를 ref로 보관하는 이유:
  pointermove는 초당 수십~수백 번 발생
  매 이벤트마다 setState를 호출하면 → 초당 수백 번 리렌더 → 성능 문제

  ref에 쌓다가 pointerup 때 state에 한 번만 옮기는 패턴:
    pointermove  → ref에 점 추가 (리렌더 없음, 빠름)
    pointerup    → 완성된 획을 state에 추가 (리렌더 1번)

  stageRef(DOM)와 drawing(데이터) 둘 다 useRef — 목적이 다름:
    stageRef   → DOM 요소 참조 (이벤트 등록, 포인터 캡처 등)
    drawing    → 그리기 진행 중 임시 데이터 (리렌더 없이 업데이트)
```

## 이전 값 기억

```typescript
function usePrevious<T>(value: T): T | undefined {
  const prevRef = useRef<T | undefined>(undefined);

  // 렌더링 후 현재 값을 저장 (다음 렌더에서 "이전 값"이 됨)
  useEffect(() => {
    prevRef.current = value;
  });

  return prevRef.current;  // 렌더 시점엔 아직 이전 값
}

// 사용
const prevCount = usePrevious(count);
console.log(`이전: ${prevCount}, 현재: ${count}`);
```

## 최초 마운트 여부 확인

```typescript
function useIsFirstRender(): boolean {
  const isFirstRef = useRef(true);

  useEffect(() => {
    isFirstRef.current = false;  // 첫 렌더 이후 false
  }, []);

  return isFirstRef.current;
}

// 사용 — 마운트 직후는 건너뜀
const isFirst = useIsFirstRender();

useEffect(() => {
  if (isFirst) return;  // 마운트 직후는 건너뜀
  fetchSearch(query);
}, [query]);
```

## 렌더링 횟수 카운터 (디버깅)

```typescript
const renderCount = useRef(0);
renderCount.current += 1;
console.log(`렌더 횟수: ${renderCount.current}`);
// useState로 하면 setCount가 또 리렌더를 유발 → 무한루프
```

---

# ⚠️ 주의사항 ⭐️⭐️⭐️⭐️

## 렌더링 중에 ref.current를 읽거나 쓰지 말 것

```typescript
// ❌ 렌더링 중 ref 접근 — 예측 불가한 동작
function Component() {
  const ref = useRef(0);
  ref.current += 1;              // 렌더링 중에 직접 변경

  return <div>{ref.current}</div>; // 렌더링 중에 읽기
}

// ✅ useEffect 안에서만 접근
function Component() {
  const ref = useRef(0);

  useEffect(() => {
    ref.current += 1;  // 렌더링이 끝난 후에 접근 — 안전
  });

  return <div>...</div>;
}
```

```txt
이유:
  React의 렌더링은 순수 함수여야 함 — 같은 입력에 같은 출력
  렌더링 중 ref.current를 바꾸면 렌더링이 반복될 때 다른 결과가 나올 수 있음
  → DOM 접근이나 부수효과는 useEffect 안에서
```

## ref.current를 JSX에 직접 쓰면 업데이트 안 됨

```typescript
// ❌ ref.current가 바뀌어도 화면이 업데이트 안 됨
const countRef = useRef(0);
return <div>{countRef.current}</div>;  // 항상 초기값만 표시

// ✅ 화면에 표시해야 하는 값은 useState
const [count, setCount] = useState(0);
return <div>{count}</div>;
```

---

# useRef vs useCallback (함수 참조 안정화) ⭐️⭐️

```typescript
// useCallback — 값이 바뀌면 함수도 새로 생성, deps 관리
const fn = useCallback(() => { ... }, [deps]);

// useRef — 항상 같은 함수 참조 유지, 최신 클로저를 ref에 저장
const fnRef = useRef(fn);
useEffect(() => { fnRef.current = fn; });
const stableFn = useCallback((...args) => fnRef.current(...args), []);
```

```txt
useRef로 함수를 안정화하는 게 유용한 경우:
  deps가 너무 많아서 useCallback이 너무 자주 바뀔 때
  이벤트 리스너처럼 딱 한 번만 등록하고 항상 최신 클로저를 실행하고 싶을 때
```

---

# 한눈에

```txt
useRef(초기값):
  .current로 읽고 쓰는 "상자"
  .current를 바꿔도 리렌더 없음 → 화면에 보이면 안 되는 값에 사용

DOM 접근:
  useRef<HTMLInputElement>(null) → <input ref={inputRef} />
  마운트 후 ref.current에 DOM 요소가 들어옴
  useEffect 안에서만 접근 (렌더링 중 접근 금지)

주요 사용 패턴:
  포커스       inputRef.current?.focus()
  스크롤       bottomRef.current?.scrollIntoView({ behavior: 'smooth' })
  타이머 ID    clearTimeout(timerRef.current)
  이전 값      prevRef.current = value (useEffect 안에서)
  마운트 여부  isFirstRef.current = false
  드래그/그리기 drawing.current = { points: [...] }  (매 이벤트마다 ref에 쌓다가 완료 시 state로)

useState와 차이:
  화면에 보여야 함 → useState
  내부에서만 쓰고 리렌더 필요 없음 → useRef

scrollIntoView 옵션 → [[JS_DOM]]
```