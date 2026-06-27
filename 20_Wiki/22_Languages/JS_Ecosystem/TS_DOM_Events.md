---
aliases:
  - MouseEvent
  - KeyboardEvent
  - SyntheticEvent
  - target vs currentTarget
tags:
  - TypeScript
related:
  - "[[00_JS_Ecosystem_HomePage]]"
  - "[[React_useRef]]"
---

# TS_DOM_Events — MouseEvent 같은 이벤트 타입

> [!info] 한마디로: 네이티브 DOM 이벤트(`document.addEventListener`로 직접 등록)와 React 합성 이벤트(JSX의 `onClick` 등으로 받는 것)는 타입이 다르다. `MouseEvent`는 전자, `React.MouseEvent<T>`는 후자다 — React를 거치지 않고 직접 등록한 리스너라면 항상 네이티브 타입을 써야 한다.

---

# 네이티브 DOM 이벤트 vs React 합성 이벤트 ⭐️⭐️⭐️⭐️

||네이티브 DOM 이벤트|React 합성 이벤트|
|---|---|---|
|등록 방법|`addEventListener('click', fn)`|JSX `onClick={fn}`|
|타입 출처|`lib.dom.d.ts` (브라우저 표준)|`react` 패키지가 자체 정의|
|타입 예시|`MouseEvent`, `KeyboardEvent`|`React.MouseEvent<T>`, `React.KeyboardEvent<T>`|
|예시 코드|`function onPointerDown(e: PointerEvent) {}`|`onClick={(e: React.MouseEvent<HTMLButtonElement>) => {}}`|

```txt
왜 둘이 다른가:
  React는 브라우저마다 이벤트 동작이 미묘하게 다른 것을 감추기 위해
  자기만의 합성 이벤트(SyntheticEvent)로 한번 감싸서 컴포넌트에 전달함

  document.addEventListener처럼 React를 거치지 않고 직접 등록한 리스너는
  이 합성 이벤트 레이어를 안 지나감 — 브라우저가 원래 주는 네이티브 이벤트 객체를 그대로 받음
  → 그래서 타입도 React가 감싸기 전의 원본 타입(MouseEvent 등)을 그대로 써야 함

헷갈리는 지점: 이름이 겹침 (React.MouseEvent도 줄여서 그냥 "MouseEvent"라고 부르는 경우가 많음)
  → import 안 했는데 자동완성으로 MouseEvent가 잡혔다면, 지금 import된 게
    네이티브 lib.dom.d.ts의 것인지 React 쪼인지 확인할 것
```

---

# 자주 보는 이벤트 타입 ⭐️⭐️⭐️

|상황|네이티브 타입|React 타입|
|---|---|---|
|마우스 클릭/누름|`MouseEvent`|`React.MouseEvent<T>`|
|포인터(마우스+터치+펜 통합)|`PointerEvent`|`React.PointerEvent<T>`|
|키보드|`KeyboardEvent`|`React.KeyboardEvent<T>`|
|포커스 이동|`FocusEvent`|`React.FocusEvent<T>`|
|터치(모바일)|`TouchEvent`|`React.TouchEvent<T>`|
|마우스 휠|`WheelEvent`|`React.WheelEvent<T>`|
|`<input>` 값 변경|(없음 — input 이벤트는 보통 React로만 다룸)|`React.ChangeEvent<HTMLInputElement>`|
|`<form>` 제출|(없음)|`React.FormEvent<HTMLFormElement>`|

```txt
<T> 자리에는 보통 그 이벤트가 발생하는 엘리먼트 타입을 넣음
  버튼 클릭   → React.MouseEvent<HTMLButtonElement>
  입력창 변경 → React.ChangeEvent<HTMLInputElement>
  폼 제출     → React.FormEvent<HTMLFormElement>
```

---

# e.target vs e.currentTarget ⭐️⭐️⭐️

```tsx
<div onClick={(e) => {
  console.log(e.target);        // 실제로 클릭이 시작된 가장 안쪽 요소 (자식의 자식일 수도 있음)
  console.log(e.currentTarget); // 이 핸들러가 등록된 요소 그 자체 (여기서는 이 div)
}}>
  <button>클릭</button>
</div>
```

```txt
target        이벤트가 실제로 발생한 지점 — 버블링을 타고 올라오는 그 시작점
              버튼을 클릭했다면 target은 버튼, 그 바깥 div가 아님

currentTarget 지금 이 이벤트 핸들러가 달려있는 요소 — 핸들러 입장에서는 "나 자신"
              위 예시에서 div에 onClick을 달았으니 currentTarget은 항상 div

→ "지금 클릭된 게 정확히 이 요소인가"를 알고 싶으면 currentTarget
  "이벤트가 어디서 시작됐는가"를 알고 싶으면 target
```

---

# `e.target as Node` — 타입 단언이 필요한 이유 ⭐️⭐️⭐️

```txt
e.target의 선언 타입은 EventTarget — 이벤트를 주고받을 수 있다는 것만 보장하는 아주 넓은 타입
.contains(), .closest() 같은 실제 DOM 메서드는 EventTarget에는 없고 Node(또는 Element)에 있음

→ "이건 실제로 Node일 거야"라고 직접 알려줘야(as Node) 그 메서드를 호출할 수 있음
```

```tsx
if (!rootRef.current?.contains(e.target as Node)) setOpen(false);
//                              ^^^^^^^^^^^^^^^^
//                              EventTarget → Node로 단언
```

```txt
⚠️ as는 검증이 아니라 단언일 뿐임 — 런타임에 진짜 Node인지 확인해주지 않음
   (실제로 클릭 이벤트의 target은 항상 Node이긴 해서 이 경우는 안전하지만,
    "타입을 우긴다"는 점에서는 다른 as 단언들과 동일한 주의가 필요함)
```

---

# 한눈에

| 개념                                    | 핵심                                                               |
| ------------------------------------- | ---------------------------------------------------------------- |
| 네이티브 이벤트(`MouseEvent` 등)              | `addEventListener`로 직접 등록할 때, React를 거치지 않은 원본 타입                |
| React 합성 이벤트(`React.MouseEvent<T>` 등) | JSX의 `onClick` 등으로 받을 때, React가 감싼 타입                            |
| `<T>`                                 | 이벤트가 발생하는 엘리먼트 타입 (예: `HTMLButtonElement`)                       |
| `e.target`                            | 이벤트가 실제로 시작된 가장 안쪽 요소                                            |
| `e.currentTarget`                     | 지금 이 핸들러가 달려있는 요소 자체                                             |
| `e.target as Node`                    | `EventTarget`은 DOM 메서드가 없어서 `Node`로 단언해야 `.contains()` 등을 쓸 수 있음 |
| `PointerEvent`                        | 마우스+터치+펜을 통합 — 모바일까지 잡고 싶을 때 `MouseEvent`보다 선호                   |