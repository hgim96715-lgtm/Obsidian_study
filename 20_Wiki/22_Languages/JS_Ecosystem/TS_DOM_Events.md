---
aliases:
  - MouseEvent
  - KeyboardEvent
  - SyntheticEvent
  - target vs currentTarget
  - preventDefault vs stopPropagation
tags:
  - TypeScript
  - React
related:
  - "[[00_JS_Ecosystem_HomePage]]"
  - "[[React_useRef]]"
---
# TS_DOM_Events — MouseEvent 같은 이벤트 타입

> [!info] 
> 네이티브 DOM 이벤트(`document.addEventListener`로 직접 등록)와 React 합성 이벤트(JSX의 `onClick` 등으로 받는 것)는 타입이 다르다. `MouseEvent`는 전자, `React.MouseEvent<T>`는 후자다 — React를 거치지 않고 직접 등록한 리스너라면 항상 네이티브 타입을 써야 한다.

---

# 네이티브 DOM 이벤트 vs React 합성 이벤트 ⭐️⭐️⭐️⭐️

| |네이티브 DOM 이벤트|React 합성 이벤트|
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
|`<form>` 제출|`SubmitEvent` (`submitter` 속성 포함)|`React.SubmitEvent<HTMLFormElement>` ⚠️|

```txt
⚠️ React 19.2.10부터 React.FormEvent / React.FormEventHandler는 deprecated됨
   대체: React.SubmitEvent<T> / React.SubmitEventHandler<T>
   옛 타입(FormEvent)도 당장 동작은 하지만 deprecation 경고가 뜨므로, 새로 작성하는 코드는
   SubmitEvent/SubmitEventHandler를 쓸 것

이 변경으로 폼 제출 이벤트는 네이티브와 React가 이름까지 같아짐(둘 다 "SubmitEvent") —
다만 출처(lib.dom.d.ts vs react)는 여전히 다른 타입이므로, 위 "이름이 겹침" 주의사항이 그대로 적용됨
```

## Event 타입 vs EventHandler 타입 ⭐️

```tsx
// 이벤트 객체 자체의 타입을 직접 쓰는 방식
function handleSubmit(e: React.SubmitEvent<HTMLFormElement>) { e.preventDefault(); }

// "함수 전체"의 타입을 변수에 미리 선언하는 방식 — *EventHandler 계열
const handleSubmit: React.SubmitEventHandler<HTMLFormElement> = (e) => { e.preventDefault(); };
```

```txt
React.SubmitEvent<T>          이벤트 객체 하나의 타입 (매개변수 자리에 직접 쓸 때)
React.SubmitEventHandler<T>   "이 타입의 이벤트를 받는 함수" 전체의 타입
                               (= (e: React.SubmitEvent<T>) => void 를 미리 정의해둔 축약형)

재사용 가능한 핸들러를 변수로 먼저 선언해두고 나중에 onSubmit={handleSubmit}로 연결할 때는
*EventHandler 쪽이 더 짧고 의도가 분명함 — 이 둘은 같은 종류의 정보를 다른 위치에서 표현하는 것뿐
```

```txt
<T> 자리에는 보통 그 이벤트가 발생하는 엘리먼트 타입을 넣음
  버튼 클릭   → React.MouseEvent<HTMLButtonElement>
  입력창 변경 → React.ChangeEvent<HTMLInputElement>
  폼 제출     → React.SubmitEvent<HTMLFormElement> (React 19.2.10+, 예전엔 FormEvent)
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
# preventDefault() — 브라우저 기본 동작 막기 ⭐️⭐️⭐️⭐️

```tsx
const handleSubmit: React.SubmitEventHandler<HTMLFormElement> = (e) => {
  e.preventDefault(); // 폼 제출 시 브라우저가 페이지를 새로고침하는 기본 동작을 막음
  // ... 직접 fetch로 제출 처리
};
```

```txt
많은 DOM 이벤트는 "브라우저가 원래 알아서 해주는 동작"을 같이 갖고 있음 —
preventDefault()는 그 기본 동작만 취소하고, 내가 등록한 핸들러 코드는 그대로 실행되게 둠
```

|이벤트|기본 동작|`preventDefault()`로 막는 것|
|---|---|---|
|`<form>` 제출|페이지 새로고침(또는 다른 페이지로 이동)|새로고침 막고, JS로 직접 제출 처리|
|`<a>` 클릭|그 링크로 페이지 이동|이동 막고, 직접 라우팅 처리 등|
|`<input type="checkbox">` 클릭|체크 상태 토글|토글 자체를 막음(드물게 씀)|
|마우스 우클릭(`contextmenu`)|브라우저 기본 컨텍스트 메뉴 표시|커스텀 메뉴를 직접 띄울 때 기본 메뉴 차단|

```txt
⚠️ 모든 이벤트에 "기본 동작"이 있는 건 아님 — 예를 들어 일반 <div>의 onClick에는
   막을 기본 동작 자체가 없어서 preventDefault()를 호출해도 아무 효과가 없음
   (폼 제출, 링크 클릭처럼 "브라우저가 원래 뭔가를 한다"는 이벤트에서만 의미가 있음)
```

---
# stopPropagation() — preventDefault와 가장 헷갈리는 짝꿍 ⭐️⭐️⭐️⭐️

```txt
preventDefault()와 stopPropagation()은 이름이 비슷해 보여도 막는 대상이 완전히 다름
```

|메서드|막는 것|
|---|---|
|`preventDefault()`|브라우저가 그 이벤트에 대해 "원래 하려던 동작"(새로고침, 페이지 이동 등)|
|`stopPropagation()`|이 이벤트가 부모 요소로 계속 전파(버블링)되는 것|


```tsx
// 바깥 클릭 감지(Click Outside) 패턴에서 흔히 같이 보이는 코드 — [[React_useRef]] 참고
<div onClick={() => setOpen(false)}>
  <button onClick={(e) => {
    e.stopPropagation(); // 이 클릭이 바깥 div의 onClick까지 전파되는 걸 막음
    doSomething();
  }}>
    버튼
  </button>
</div>
```

```txt
버블링(bubbling)이란: 자식 요소에서 발생한 이벤트가 부모 → 부모의 부모 순으로
계속 위로 전파되는 기본 동작 — 버튼을 클릭하면 그 클릭 이벤트가 바깥 div에도 또 전달됨

stopPropagation()을 안 부르면: 위 예시에서 버튼을 눌러도 그 클릭이 바깥 div까지 전파돼서
  div의 onClick(메뉴 닫기)까지 같이 실행되는, 의도치 않은 동작이 생길 수 있음

→ "이 이벤트가 일으키려던 브라우저 동작을 막을지"는 preventDefault,
  "이 이벤트가 부모에게도 알려질지"는 stopPropagation — 서로 완전히 독립적인 두 가지 결정
  (하나만 호출하고 다른 하나는 그대로 둬도 됨, 둘 다 호출해도 됨)
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
| `e.preventDefault()`                  | 그 이벤트의 브라우저 기본 동작(새로고침, 페이지 이동 등)을 막음                            |
| `e.stopPropagation()`                 | 이벤트가 부모로 버블링되는 것을 막음 — `preventDefault`와 독립적                     |
