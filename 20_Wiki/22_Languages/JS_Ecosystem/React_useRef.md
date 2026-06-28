---
aliases: [current, ref, useRef]
tags:
  - React
related:
  - "[[00_JS_Ecosystem_HomePage]]"
  - "[[React_useMemo_useCallback_useEffect]]"
  - "[[TS_DOM_Events]]"
---
# React_useRef — 렌더링과 무관하게 값을 들고 있기

> [!info] 
> `useRef`는 "리렌더를 안 일으키면서 값을 기억해두는 상자"다. 
> 가장 흔한 용도는 ① DOM 엘리먼트에 직접 접근하기, ② 렌더링과 무관한 값(타이머 ID, 이전 값 등)을 들고 있기 — 둘 다 `.current`라는 하나의 속성을 통해 값을 읽고 쓴다.

---

# ref란 무엇인가 — useState와 다른 점 ⭐️⭐️⭐️

|구분|`useState`|`useRef`|
|---|---|---|
|값이 바뀌면|리렌더 발생|리렌더 안 일어남|
|값을 읽는 법|변수 그대로|`.current`를 통해서만|
|적합한 값|화면에 "보여줘야" 하는 값|화면 표시와 무관하게 그냥 "기억"만 해두면 되는 값|

```tsx
const countRef = useRef(0);
countRef.current += 1;
// 값은 바뀌었지만 화면은 안 바뀜 — 리렌더를 일으키는 신호 자체가 없기 때문
```

```txt
useRef가 반환하는 객체({ current: ... })는 컴포넌트가 리렌더돼도 항상 같은 객체임
→ 그 안의 .current 값만 바뀌는 것이고, useRef를 호출한 변수(countRef) 자체는 매 렌더마다 동일함
  (useState처럼 "새 값으로 교체"가 아니라, 같은 박스 안의 내용물만 바꾸는 것)
```

---

# `useRef<T>`(initialValue) — 제네릭과 초기값 ⭐️⭐️⭐️

```tsx
const rootRef = useRef<HTMLDivElement>(null);
//              ↑ <T> = .current에 들어갈 값의 타입
//                        초기값은 null
```

```txt
<HTMLDivElement> — 이 ref를 <div>에 연결할 거라는 타입 선언
  타입스크립트는 ref.current의 실제 타입을 HTMLDivElement | null 로 추론함

왜 초기값이 항상 null인가:
  useRef가 처음 호출되는 시점(첫 렌더링 중)엔, 그 ref를 매달 DOM 노드가
  아직 실제로 화면에 만들어지지(커밋되지) 않은 상태임
  → React가 렌더링을 마치고 실제 DOM에 반영한 "직후"에야 .current를 그 노드로 채워줌
  → 그 사이(첫 렌더링 도중)에는 가리킬 게 없으니 null이 정직한 시작값

타입이 HTMLDivElement | null 인 이유, 그래서 매번 ?. 가 필요한 이유:
  rootRef.current.contains(...)   ❌ "null일 수도 있다"는 경고 — 컴파일 에러
  rootRef.current?.contains(...)  ✅ null이면 그냥 undefined로 안전하게 빠짐
  → null인 경우(아직 마운트 전, 또는 조건부 렌더링으로 그 엘리먼트가 없는 경우)를
    항상 같이 고려해야 한다는 걸 타입이 강제해주는 것
```

---

# DOM 엘리먼트에 직접 접근하기 — 가장 흔한 용도 ⭐️⭐️⭐️

```tsx
const inputRef = useRef<HTMLInputElement>(null);

return <input ref={inputRef} />;

// 나중에 — 진짜 DOM 메서드를 직접 호출
inputRef.current?.focus();
```

```txt
JSX에 ref={inputRef} 를 달아두면, React가 마운트를 마친 뒤
실제 DOM 노드를 자동으로 inputRef.current에 채워줌

그 뒤로는 React가 추상화해주지 않는, 브라우저가 원래 제공하는 진짜 DOM 메서드를
직접 호출할 수 있게 됨 — .focus() / .scrollIntoView() / .contains() / .getBoundingClientRect() 등
(React는 "무엇을 화면에 그릴지"만 관리하고, 이런 명령형 동작은 ref로 DOM에 직접 요청해야 함)
```

---

# 렌더링과 무관한 값 저장 — 두 번째 용도 ⭐️⭐️

```tsx
const timerRef = useRef<ReturnType<typeof setTimeout> | null>(null);

function startTimer() {
  timerRef.current = setTimeout(() => console.log('done'), 1000);
}

function cancelTimer() {
  if (timerRef.current) clearTimeout(timerRef.current);
}
```

```txt
타이머 id, 이전 렌더의 값(previous value), 렌더링 횟수 카운트처럼
"로직에는 필요하지만 화면에 직접 보여줄 건 아닌 값"을 들고 있을 때 씀
useState로 만들면 그 값이 바뀔 때마다 불필요한 리렌더가 같이 일어남 — ref는 그게 없음
```

---

# 실전 예시 — 바깥 클릭 감지(Click Outside) 패턴 ⭐️⭐️⭐️⭐️

```txt
드롭다운/메뉴/팝오버처럼 "이 영역 바깥을 클릭하면 닫는다"는 패턴은
ref로 그 영역의 실제 DOM 노드를 들고 있어야 "지금 클릭한 곳이 이 안인지 밖인지" 판단할 수 있음
```

```tsx
const rootRef = useRef<HTMLDivElement>(null);

useEffect(() => {
  if (!open) return;

  function onPointerDown(e: PointerEvent) {
    if (!rootRef.current?.contains(e.target as Node)) setOpen(false);
  }

  document.addEventListener('pointerdown', onPointerDown);
  return () => document.removeEventListener('pointerdown', onPointerDown);
}, [open]);

return <div ref={rootRef}>{/* 메뉴/드롭다운 내용 */}</div>;
```

```txt
⚠️ 자주 나는 실수 — addEventListener/cleanup이 핸들러 함수 "안"에 들어가는 경우:

  function onPointerDown(e) {
    if (...) setOpen(false);
    document.addEventListener('pointerdown', onPointerDown);  // ❌ 여기 있으면 안 됨
    return () => document.removeEventListener(...);            // ❌ 이것도
  }

  이렇게 두면 addEventListener 호출 자체가 onPointerDown "함수 본문 안"에 갇혀버림
  → onPointerDown은 정의만 되고 아무도 호출하지 않으니, 그 안의 addEventListener 줄은
    영원히 실행되지 않음 — 리스너가 실제로는 한 번도 등록되지 않는 조용한 버그가 됨

  addEventListener와 그 클린업(return)은 항상 useEffect 콜백의 "바로 안"(핸들러 함수 정의와
  같은 들여쓰기 레벨)에 있어야 함 — 핸들러 함수 정의와 그 핸들러를 등록하는 코드는 서로 다른 줄
```

```txt
rootRef.current?.contains(e.target as Node) 한 줄씩:

  rootRef.current        지금 이 컴포넌트가 그린 실제 <div> DOM 노드 (없으면 null)
  .contains(다른노드)     그 다른 노드가 이 노드의 자손(또는 자기 자신)인지 확인하는 네이티브 DOM 메서드
  e.target               이번 클릭이 실제로 발생한 가장 안쪽 요소
  as Node                e.target의 선언 타입(EventTarget)은 .contains에 안 맞아서 타입 단언 필요
                         (자세한 이유는 [[TS_DOM_Events]] 참고)

  → "지금 클릭한 지점이 내가 들고 있는 영역(rootRef) 안에 없다면 닫아라"는 뜻
```

```txt
PointerEvent를 쓰는 이유(MouseEvent 대신):
  마우스 클릭뿐 아니라 터치(모바일)·펜 입력까지 한 가지 이벤트로 통합해서 잡을 수 있음
  → "바깥을 눌렀다"를 마우스 환경에만 한정하고 싶지 않다면 pointerdown이 더 범용적
  네이티브 이벤트 타입과 React 합성 이벤트 타입의 차이 전반은 [[TS_DOM_Events]] 참고
```

---

# 한눈에

|개념|역할|
|---|---|
|`useRef(초기값)`|리렌더를 안 일으키는 값 저장 상자 — `.current`로 읽고 씀|
|`useRef<T>(null)`|DOM 엘리먼트용 — 마운트 후 React가 실제 노드로 `.current`를 채워줌|
|`.current?.xxx`|타입이 `T \| null`이라 항상 null 가능성을 처리해야 함|
|DOM 접근 용도|`.focus()`, `.contains()`, `.scrollIntoView()` 등 React가 추상화 안 한 명령형 동작|
|값 저장 용도|타이머 id, 이전 값, 카운트처럼 화면 표시와 무관한 값|
|흔한 실수|이벤트 등록(`addEventListener`)과 클린업을 핸들러 함수 안에 잘못 넣는 것|