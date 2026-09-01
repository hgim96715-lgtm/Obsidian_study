---
aliases:
  - HTMLDetailsElement
  - HTMLInputElement
  - HTMLElement 타입
  - DOM 타입 계층
  - useRef 타입
tags:
  - typescript
  - html
  - dom
  - ref
  - react
relations:
  - "[[00_JS_Ecosystem_HomePage]]"
  - "[[React_useRef]]"
  - "[[React_Input]]"
  - "[[TS_DOM_Events]]"
  - "[[JS_DOM]]"
---
# HTML Element Types — TypeScript DOM 타입

> [!info]
> TypeScript에서 HTML 요소에 접근할 때 정확한 타입을 지정해야
> `.open`, `.value`, `.files` 등 요소 고유 프로퍼티에 타입 안전하게 접근 가능

---

## DOM 타입 계층 구조

```
EventTarget
  └─ Node
       └─ Element
            └─ HTMLElement                    ← 모든 HTML 요소 공통 타입
                 ├─ HTMLDivElement            (<div>, <p>, <span> 등 대부분)
                 ├─ HTMLInputElement          (<input>)       → .value .checked .files .type
                 ├─ HTMLTextAreaElement       (<textarea>)    → .value .rows .cols
                 ├─ HTMLButtonElement         (<button>)      → .disabled .type .form
                 ├─ HTMLFormElement           (<form>)        → .submit() .reset() .elements
                 ├─ HTMLSelectElement         (<select>)      → .value .selectedIndex .options
                 ├─ HTMLDetailsElement        (<details>)     → .open
                 ├─ HTMLVideoElement          (<video>)       → .play() .pause() .currentTime
                 ├─ HTMLAudioElement          (<audio>)       → .play() .pause() .volume
                 ├─ HTMLCanvasElement         (<canvas>)      → .getContext() .width .height
                 ├─ HTMLImageElement          (<img>)         → .src .alt .naturalWidth
                 ├─ HTMLAnchorElement         (<a>)           → .href .target .download
                 └─ HTMLDialogElement         (<dialog>)      → .open .showModal() .close()
```

```txt
HTMLElement로 쓰면 → 하위 타입 고유 프로퍼티(.open, .value 등) 접근 시 TS 에러
정확한 타입을 지정해야 IntelliSense + 타입 안전성 확보
```

---

## 자주 쓰는 타입 한눈에

| 요소 | TypeScript 타입 | 고유 프로퍼티 / 메서드 |
|---|---|---|
| `<div>`, `<p>`, `<span>` | `HTMLDivElement` | — (HTMLElement 공통만) |
| `<input>` | `HTMLInputElement` | `.value` `.checked` `.files` `.type` `.focus()` |
| `<textarea>` | `HTMLTextAreaElement` | `.value` `.rows` `.selectionStart` |
| `<button>` | `HTMLButtonElement` | `.disabled` `.type` `.form` |
| `<form>` | `HTMLFormElement` | `.submit()` `.reset()` `.elements` |
| `<select>` | `HTMLSelectElement` | `.value` `.selectedIndex` `.options` |
| `<details>` | `HTMLDetailsElement` | `.open` (boolean 하나) |
| `<dialog>` | `HTMLDialogElement` | `.open` `.showModal()` `.close()` |
| `<video>` | `HTMLVideoElement` | `.play()` `.pause()` `.currentTime` `.duration` |
| `<audio>` | `HTMLAudioElement` | `.play()` `.pause()` `.volume` `.muted` |
| `<canvas>` | `HTMLCanvasElement` | `.getContext()` `.width` `.height` |
| `<img>` | `HTMLImageElement` | `.src` `.alt` `.naturalWidth` `.complete` |
| `<a>` | `HTMLAnchorElement` | `.href` `.target` `.download` |
| 타입 모를 때 | `HTMLElement` | 공통 프로퍼티만 |

---

## HTMLElement 공통 프로퍼티 — 모든 요소에서 사용 가능

```ts
el.id                // id 어트리뷰트
el.className         // class 어트리뷰트 (공백 구분 문자열)
el.classList         // DOMTokenList — .add() .remove() .toggle() .contains()
el.style             // CSSStyleDeclaration — el.style.height = '100px'
el.dataset           // data-* 어트리뷰트 — el.dataset.userId
el.hidden            // display:none 토글
el.innerHTML         // 내부 HTML 문자열
el.textContent       // 내부 텍스트
el.offsetWidth       // 렌더링된 너비(px) — border 포함, margin 제외
el.scrollHeight      // 스크롤 포함 전체 높이
el.getBoundingClientRect()  // 뷰포트 기준 위치·크기 DOMRect
el.contains(other)   // 자식 포함 여부 — 외부 클릭 감지에 사용
el.focus()           // 포커스 이동
el.blur()            // 포커스 해제
el.click()           // 클릭 이벤트 강제 발생
el.scrollIntoView({ behavior: 'smooth' })
```

---

## `<details>` / `<summary>` — 네이티브 드롭다운

```html
<!-- JS 없이 브라우저 기본으로 열리고 닫힘 -->
<details>
  <summary>메뉴 열기</summary>  <!-- 클릭 트리거 -->
  <ul>
    <li>설정</li>
    <li>로그아웃</li>
  </ul>
</details>

<!-- 처음부터 열린 상태 -->
<details open>
  <summary>펼쳐진 섹션</summary>
  ...
</details>
```

```txt
<details>  — 접히는 컨테이너. open 어트리뷰트 유무로 열림/닫힘 결정
<summary>  — 클릭 트리거. 없으면 브라우저 기본 텍스트("Details") 표시
open 어트리뷰트 — boolean 어트리뷰트: 있으면 열림, 없으면 닫힘

활용:
  CSS-only 아코디언 / 드롭다운 메뉴
  JS 없이 동작하므로 접근성(a11y) 기본 지원
  열림/닫힘 시 toggle 이벤트 발생
```

### HTMLDetailsElement 인터페이스

```ts
// lib.dom.d.ts (TypeScript 내장 타입)
interface HTMLDetailsElement extends HTMLElement {
  open: boolean;  // <details open> 어트리뷰트와 동기화. true=열림 false=닫힘
}
```

```txt
HTMLDetailsElement가 HTMLElement에 추가하는 것: open: boolean 딱 하나
ref.current.open = false  →  <details>를 프로그래밍으로 닫기
ref.current.open = true   →  <details>를 프로그래밍으로 열기
```

---

## useRef 제네릭 타입 패턴

```tsx
// 기본 패턴 — 초기값 null, 마운트 후 DOM 요소가 자동으로 들어옴
const inputRef   = useRef<HTMLInputElement>(null);
const menuRef    = useRef<HTMLDetailsElement>(null);
const videoRef   = useRef<HTMLVideoElement>(null);
const dialogRef  = useRef<HTMLDialogElement>(null);
```

```txt
초기값을 null로 주는 이유:
  마운트 전에는 DOM 요소가 없음
  React가 마운트 후 ref.current에 DOM 요소를 자동으로 넣어줌
  언마운트 시 자동으로 null로 되돌아감

접근 시 항상 null 체크:
  ref.current?.focus()   — optional chaining
  if (ref.current) { ... }
```

### HTMLInputElement — 주요 패턴

```tsx
const inputRef = useRef<HTMLInputElement>(null);

// 포커스
inputRef.current?.focus();

// 비제어 방식으로 값 읽기 (제출 시 한 번만)
const value = inputRef.current?.value ?? '';

// 파일 인풋
const fileRef = useRef<HTMLInputElement>(null);
const file = fileRef.current?.files?.[0];  // FileList → File

// 높이 자동 조절 (textarea)
const textareaRef = useRef<HTMLTextAreaElement>(null);
// onChange 안에서
e.target.style.height = 'auto';
e.target.style.height = `${e.target.scrollHeight}px`;
// 이벤트 밖(비동기 후 초기화)에서
if (textareaRef.current) {
  textareaRef.current.style.height = 'auto';
}
```

### HTMLDetailsElement — 외부 클릭 시 닫기

```tsx
const menuRef = useRef<HTMLDetailsElement>(null);

useEffect(() => {
  const handleClick = (e: MouseEvent) => {
    // contains — <details> 외부를 클릭했는지 판단
    if (menuRef.current && !menuRef.current.contains(e.target as Node)) {
      menuRef.current.open = false;  // 프로그래밍으로 닫기
    }
  };
  document.addEventListener('click', handleClick);
  return () => document.removeEventListener('click', handleClick);
}, []);

return (
  <details ref={menuRef}>
    <summary>메뉴</summary>
    <ul>
      <li>설정</li>
      <li>로그아웃</li>
    </ul>
  </details>
);
```

```txt
el.contains(e.target as Node)
  → <details> 내부를 클릭했으면 true (메뉴 닫지 않음)
  → 외부를 클릭했으면 false (open = false 로 닫음)

e.target as Node
  → MouseEvent.target은 EventTarget 타입
  → contains()는 Node 타입을 요구하므로 타입 단언 필요
```

### toggle 이벤트 감지

```tsx
useEffect(() => {
  const el = menuRef.current;
  if (!el) return;

  const handleToggle = () => {
    console.log('열림 상태:', el.open); // true | false
  };

  el.addEventListener('toggle', handleToggle);
  return () => el.removeEventListener('toggle', handleToggle);
}, []);
```

### HTMLDialogElement — 모달

```tsx
const dialogRef = useRef<HTMLDialogElement>(null);

// 모달 열기 (backdrop 포함)
dialogRef.current?.showModal();

// 모달 닫기
dialogRef.current?.close();

// 열린 상태 확인
const isOpen = dialogRef.current?.open; // boolean
```

```tsx
<dialog ref={dialogRef}>
  <h2>모달 제목</h2>
  <button onClick={() => dialogRef.current?.close()}>닫기</button>
</dialog>
```

### HTMLVideoElement

```tsx
const videoRef = useRef<HTMLVideoElement>(null);

videoRef.current?.play();
videoRef.current?.pause();

const current = videoRef.current?.currentTime;  // 현재 재생 위치 (초)
const duration = videoRef.current?.duration;    // 전체 길이 (초)
const isPaused = videoRef.current?.paused;      // boolean
```

---

## e.target 타입 단언 — 이벤트 핸들러에서

```tsx
// HTMLInputElement
onChange={(e) => {
  const input = e.target as HTMLInputElement;
  console.log(input.value);
}}

// HTMLSelectElement
onChange={(e) => {
  const select = e.target as HTMLSelectElement;
  console.log(select.value);
}}

// currentTarget vs target
// currentTarget — 이벤트 리스너가 붙은 요소 (타입이 명확)
// target        — 실제로 클릭된 요소 (자식일 수 있어서 EventTarget 타입)
onInput={(e) => {
  const input = e.currentTarget; // HTMLInputElement — 타입 단언 불필요
  input.value = normalized;
}}
```

---

## 안티패턴

```tsx
// ❌ HTMLElement로 통일 — 고유 프로퍼티 접근 불가
const ref = useRef<HTMLElement>(null);
ref.current?.open;  // TS 에러: 'open' does not exist on 'HTMLElement'

// ✅ 정확한 타입 지정
const ref = useRef<HTMLDetailsElement>(null);
ref.current?.open;  // OK

// ❌ null 체크 없이 직접 접근
ref.current.focus();  // TypeError: Cannot read properties of null

// ✅ optional chaining
ref.current?.focus();

// ❌ 렌더링 중에 ref.current 읽기
function Component() {
  const ref = useRef<HTMLInputElement>(null);
  const val = ref.current?.value;  // 마운트 전이라 항상 undefined
  return <input ref={ref} defaultValue={val} />;  // 의미 없음
}

// ✅ useEffect 안에서 접근 (마운트 후)
useEffect(() => {
  const val = ref.current?.value;
}, []);
```

---

> [!note] 핵심 정리
> `useRef<HTML요소타입>(null)` → 요소 고유 프로퍼티 타입 안전 접근
> `HTMLDetailsElement` — `.open: boolean` 하나, `ref.current.open = false`로 프로그래밍 닫기
> `HTMLDialogElement` — `.showModal()` / `.close()` 로 모달 제어
> `e.currentTarget` 은 타입 확정, `e.target` 은 `EventTarget` 이라 단언 필요
