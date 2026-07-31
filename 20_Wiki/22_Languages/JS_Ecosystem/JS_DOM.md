---
aliases:
  - DOM
  - textarea
  - requestSubmit
  - currentTarget
  - scrollIntoView
  - getBoundingClientRect
  - Pointer Events
  - e.nativeEvent
tags:
  - JavaScript
related:
  - "[[00_JS_Ecosystem_HomePage]]"
  - "[[JS_BrowserAPI]]"
  - "[[TS_DOM_Events]]"
  - "[[TS_ImportType]]"
  - "[[JS_Canvas]]"
  - "[[React_useId]]"
  - "[[React_useRef]]"
---
# JS_DOM — DOM 조작

> [!info]
>  DOM = 브라우저가 HTML을 파싱해서 만든 트리 구조의 객체들. 
>  JavaScript로 이 객체들을 읽고, 바꾸고, 추가할 수 있다. 
>  React가 대부분을 처리해주지만 포커스·스크롤·외부 라이브러리 연동에서는 직접 다뤄야 할 때가 있다.

---

# DOM이란 ⭐️⭐️⭐️⭐️

```txt
브라우저가 HTML 파일을 받으면:
  1. HTML 텍스트를 파싱
  2. 각 태그를 JavaScript 객체(Node)로 변환
  3. 이 객체들을 부모-자식 관계로 연결 → 트리 구조 완성
  4. 이 트리가 DOM (Document Object Model)
```

```txt
HTML:
  <body>
    <div id="app">
      <h1>제목</h1>
      <p>내용</p>
    </div>
  </body>

↓ 파싱 후 메모리 안:

  body (HTMLBodyElement 객체)
    └── div#app (HTMLDivElement 객체)
          ├── h1 (HTMLHeadingElement 객체) → "제목" 텍스트
          └── p (HTMLParagraphElement 객체)  → "내용" 텍스트
```

```txt
JavaScript는 이 객체들에 직접 접근 가능:
  const div = document.getElementById('app');
  div.textContent = '변경됨';  // 객체 속성을 바꾸면
  // → 브라우저가 화면을 다시 그림 (리렌더)

document = 이 트리 전체의 진입점 (루트)
  document.body    → <body> 객체
  document.head    → <head> 객체
  document.title   → 페이지 제목 (읽기/쓰기)
```

## React에서도 직접 DOM이 필요한 경우

```txt
React는 가상 DOM(Virtual DOM)을 유지하며 최소한의 실제 DOM 변경만 함
→ 대부분의 UI 조작은 React가 알아서 처리

하지만 직접 DOM 접근이 필요한 경우:
  포커스      inputRef.current.focus()  — React는 언제 포커스를 줄지 모름
  스크롤      scrollIntoView()          — 채팅 맨 아래로 이동
  크기 측정   getBoundingClientRect()   — 요소가 화면 어디 있는지
  외부 라이브러리  YouTube IFrame API처럼 직접 DOM 요소를 요구하는 경우
  드래그/그리기    pointermove 좌표를 실시간으로 처리

→ 이런 경우에 useRef + DOM API를 같이 씀
  ref.current = 실제 DOM 요소
  ref.current.focus() / ref.current.getBoundingClientRect() 등
```

---

# 요소 찾기 ⭐️⭐️⭐️⭐️

```javascript
// querySelector — 첫 번째로 일치하는 요소 하나
// 없으면 null 반환
document.querySelector('.btn-primary')
document.querySelector('#user-name')
document.querySelector('script[src="https://example.com/api.js"]')

// querySelectorAll — 일치하는 모든 요소
// 없으면 빈 NodeList (null 아님)
document.querySelectorAll('li.active')
document.querySelectorAll('[data-id]')  // data-id 속성이 있는 모든 요소
```

## CSS 선택자 문법

|선택자|의미|예시|
|---|---|---|
|`태그`|태그명|`script`|
|`.클래스`|클래스명|`.btn-primary`|
|`#아이디`|id|`#user-name`|
|`[속성]`|속성 존재 여부|`[async]`|
|`[속성="값"]`|속성 값 일치|`[src="https://..."]`|
|`[속성^="값"]`|속성 값이 ~로 시작|`[src^="https://"]`|
|`[속성*="값"]`|속성 값에 ~가 포함|`[src*="youtube"]`|
|`부모 자식`|하위 요소 전체|`div .item`|
|`부모 > 자식`|직계 자식만|`ul > li`|
|`A, B`|A 또는 B|`.btn, .link`|

```javascript
// 실전 — 특정 src를 가진 script 태그가 이미 있는지 확인
if (!document.querySelector('script[src="https://www.youtube.com/iframe_api"]')) {
  // 없을 때만 추가
}
```

```txt
querySelector vs getElementById:
  getElementById  → id 하나만, 약간 빠름
  querySelector   → CSS 선택자 전부 사용 가능, 더 유연
  → 실무에서는 querySelector로 통일하는 경우가 많음

querySelectorAll vs getElementsByClassName:
  querySelectorAll    → 정적 NodeList (DOM 변경돼도 결과 안 바뀜)
  getElementsBy...    → 동적 HTMLCollection (DOM 변경되면 자동 업데이트)
  → 예측 가능한 querySelectorAll 권장
```

---

# 요소 만들기 ⭐️⭐️⭐️⭐️

```javascript
// createElement — 메모리 안에 새 DOM 객체 생성 (아직 화면에 없음)
const script = document.createElement('script');
script.src   = 'https://www.youtube.com/iframe_api';
script.async = true;

const div = document.createElement('div');
div.id          = 'my-container';
div.className   = 'container active';
div.textContent = '안녕하세요';
```

```txt
createElement로 만든 요소는 트리에 연결되기 전까지 화면에 안 보임
appendChild/prepend 등으로 트리에 붙여야 렌더링됨
```

## 속성 설정 방법

```javascript
// 방법 1 — 프로퍼티 직접 할당 (표준 속성)
script.src   = 'https://example.com/api.js';
script.async = true;
input.value  = '초기값';
img.alt      = '이미지 설명';

// 방법 2 — setAttribute (커스텀 속성, data-*, aria-*)
el.setAttribute('data-id', '123');
el.setAttribute('aria-label', '닫기 버튼');
el.getAttribute('data-id')    // '123'
el.removeAttribute('disabled');
el.hasAttribute('async')      // true / false
```

```txt
언제 뭘 쓰는가:
  src / href / async / defer / value / checked 같은 표준 속성
  → 프로퍼티로 직접 할당 (타입 안전, 자동완성)

  data-* 속성, aria-* 속성, 커스텀 속성
  → setAttribute 사용 (프로퍼티로 접근 불가)
```

---

# 요소 추가 / 이동 / 삭제 ⭐️⭐️⭐️⭐️

```javascript
// 추가
document.body.appendChild(script);      // body 맨 끝에 추가
parentEl.appendChild(childEl);          // 맨 끝에
parentEl.prepend(el);                   // 맨 앞에

// 특정 위치에
parentEl.insertBefore(newEl, referenceEl);  // referenceEl 앞에
el.insertAdjacentElement('afterend', newEl) // el 바로 뒤에
```

```javascript
// insertAdjacentElement 위치
el.insertAdjacentElement('beforebegin', newEl)  // el 바로 앞 (형제)
el.insertAdjacentElement('afterbegin', newEl)   // el 안, 첫 번째 자식 앞
el.insertAdjacentElement('beforeend', newEl)    // el 안, 마지막 자식 뒤
el.insertAdjacentElement('afterend', newEl)     // el 바로 뒤 (형제)

// 삭제
el.remove()                       // 요소 자체를 트리에서 제거
parentEl.removeChild(childEl);    // 부모에서 자식 제거 (el.remove()가 더 간결)
el.replaceWith(newEl);            // 다른 요소로 교체
```

---

# classList — 클래스 다루기 ⭐️⭐️⭐️

```javascript
el.classList.add('active')              // 클래스 추가
el.classList.remove('hidden')           // 클래스 제거
el.classList.toggle('open')             // 있으면 제거, 없으면 추가
el.classList.toggle('open', true)       // 조건이 true면 추가
el.classList.toggle('open', false)      // 조건이 false면 제거
el.classList.contains('active')         // 있는지 확인 → true/false
el.classList.replace('old', 'new')      // 클래스 교체

// className으로 전체 교체 (기존 클래스 전부 날아감 — 주의)
el.className = 'btn btn-primary';
```

```javascript
// 실전 패턴 — 조건에 따라 추가/제거
el.classList.toggle('active', isLoggedIn);  // isLoggedIn이 true면 추가, false면 제거
```

---

# textContent vs innerHTML ⭐️⭐️⭐️

```javascript
el.textContent = '안녕하세요';   // 텍스트만 — HTML 태그를 문자 그대로 표시
el.innerHTML   = '<b>굵게</b>'; // HTML 파싱해서 렌더링

el.textContent  // 읽기: 텍스트 내용 (태그 제거됨)
el.innerHTML    // 읽기: HTML 문자열 그대로
```

```txt
⚠️ innerHTML에 사용자 입력을 그대로 넣으면 XSS 취약점:
  el.innerHTML = userInput  ← 위험 — 사용자가 <script>를 심을 수 있음
  el.textContent = userInput ← 안전 — 태그를 문자로 처리

React에 dangerouslySetInnerHTML이 있는 이유가 바로 이것
```

---

# 키보드 이벤트 ⭐️⭐️⭐️⭐️

```typescript
<input
  onKeyDown={(e) => {
    if (e.key === 'Enter') submitForm();
    if (e.key === 'Escape') closeModal();
    if (e.key === 'k' && (e.metaKey || e.ctrlKey)) openSearch(); // Cmd+K / Ctrl+K
  }}
/>
```

## 자주 쓰는 키 이름

|키|`e.key` 값|
|---|---|
|Enter|`'Enter'`|
|Escape|`'Escape'`|
|Tab|`'Tab'`|
|백스페이스|`'Backspace'`|
|방향키|`'ArrowUp'` / `'ArrowDown'` / `'ArrowLeft'` / `'ArrowRight'`|
|조합키|`e.shiftKey` / `e.metaKey` / `e.ctrlKey` / `e.altKey`|

## isComposing — 한글 입력 중 Enter 문제 ⭐️⭐️⭐️⭐️

```typescript
onKeyDown={(e) => {
  if (e.key === 'Enter' && !e.nativeEvent.isComposing) {
    e.preventDefault();
    commit();
  }
}}
```

```txt
왜 필요한가:
  한글은 자음+모음을 조합해서 한 글자를 만듦 (IME: Input Method Editor)
  '분' 입력할 때: 'ㅂ' → '부' → '분' 조합 중

  조합 중에 Enter를 누르면 두 이벤트가 발생:
    1. IME가 조합 중인 글자를 "확정"하는 Enter
    2. 우리가 처리하려는 "제출"용 Enter

  isComposing = true  → 조합 중 Enter (확정용) — 무시해야 함
  isComposing = false → 조합 끝난 뒤 Enter — 처리할 Enter

  체크 안 하면: '분' 입력 → Enter(조합확정) → commit() 실행 → 의도 않은 제출
```

## e.nativeEvent란

```txt
React는 브라우저 이벤트를 SyntheticEvent로 감싸서 브라우저 간 일관성을 보장
isComposing은 SyntheticEvent에 없음 → e.nativeEvent (원본 DOM 이벤트)로 접근
```

---

# Pointer Events — 마우스 · 터치 · 펜 통합 ⭐️⭐️⭐️⭐️

```txt
마우스 이벤트(mousedown/mousemove/mouseup)와 터치 이벤트(touchstart 등)를
하나의 API로 통합한 것이 Pointer Events
→ onMouseDown/onTouchStart를 따로 쓰지 않아도 됨
```

```typescript
element.addEventListener('pointerdown', (e: PointerEvent) => {
  e.pointerId    // 이 포인터의 고유 ID (멀티터치 구분)
  e.clientX      // 뷰포트 기준 X 좌표
  e.clientY      // 뷰포트 기준 Y 좌표
  e.pointerType  // 'mouse' | 'touch' | 'pen'
  e.pressure     // 펜 압력 (0~1, 마우스는 0 또는 0.5)
});
```

## setPointerCapture — 포인터 캡처 ⭐️⭐️⭐️⭐️

```typescript
element.addEventListener('pointerdown', (e: PointerEvent) => {
  element.setPointerCapture(e.pointerId);
  // 이 시점부터 포인터가 element 밖으로 나가도
  // pointermove / pointerup 이벤트가 계속 element로 들어옴
});
```

```txt
왜 필요한가:
  드래그 중 마우스가 element 밖으로 나가면
  pointermove 이벤트가 다른 element로 넘어가서 추적이 끊김

  setPointerCapture(pointerId) → "이 포인터를 내가 잡겠다" 선언
  → pointerup/pointercancel이 올 때까지 이 element가 이벤트를 독점 수신

해제:
  pointerup / pointercancel 시 자동 해제
  또는 element.releasePointerCapture(e.pointerId) 수동 해제
```

## React에서 Pointer Events

```typescript
import { type PointerEvent as ReactPointerEvent } from 'react';
// as ReactPointerEvent — 전역 PointerEvent (Web API)와 이름 충돌 방지

function DrawingCanvas() {
  const canvasRef = useRef<HTMLDivElement>(null);

  const onPointerDown = (e: ReactPointerEvent<HTMLDivElement>) => {
    e.currentTarget.setPointerCapture(e.pointerId);
    // 그리기 시작
  };

  const onPointerMove = (e: ReactPointerEvent<HTMLDivElement>) => {
    if (e.buttons !== 1) return;  // 마우스 왼쪽 버튼 안 눌렸으면 무시
    // 그리기 중
  };

  const onPointerUp = () => {
    // 그리기 완료
  };

  return (
    <div
      ref={canvasRef}
      onPointerDown={onPointerDown}
      onPointerMove={onPointerMove}
      onPointerUp={onPointerUp}
      style={{ touchAction: 'none' }}  // 모바일에서 스크롤 방지
    />
  );
}
```

```txt
e.buttons:
  현재 눌린 마우스 버튼의 비트마스크
  0 = 버튼 없음, 1 = 왼쪽 클릭
  onPointerMove에서 if (e.buttons !== 1) return → 드래그 중일 때만 처리

touchAction: 'none':
  모바일에서 pointer 이벤트와 스크롤이 충돌
  none으로 설정하면 브라우저 기본 터치 동작 비활성화 → 드래그가 의도대로 동작
```

---

# getBoundingClientRect — 요소의 화면 위치 ⭐️⭐️⭐️

```typescript
const rect = el.getBoundingClientRect();

rect.left    // 뷰포트 왼쪽 기준 X 시작
rect.top     // 뷰포트 위쪽 기준 Y 시작
rect.width   // 요소 너비
rect.height  // 요소 높이
rect.right   // left + width
rect.bottom  // top + height
```

```typescript
// 실전 — 클릭 위치를 요소 내부 좌표(0~1)로 변환
function getRelativePosition(e: PointerEvent, el: HTMLElement) {
  const rect = el.getBoundingClientRect();
  return {
    x: (e.clientX - rect.left) / rect.width,
    y: (e.clientY - rect.top)  / rect.height,
  };
}
```

```txt
뷰포트 기준 — 스크롤하면 값이 달라짐
→ 클릭/드래그 이벤트마다 새로 호출해야 함

clientX/Y vs pageX/Y:
  clientX/Y  → 뷰포트 기준 (스크롤 위치 미포함)
  pageX/Y    → 문서 기준 (스크롤 위치 포함)
  getBoundingClientRect()도 뷰포트 기준 → clientX/Y와 함께 쓸 것
```

---

# scrollIntoView — 스크롤 제어 ⭐️⭐️⭐️

```javascript
element.scrollIntoView({ behavior: 'smooth' });  // 부드럽게
element.scrollIntoView({ behavior: 'instant' }); // 즉시 (기본값)
element.scrollIntoView({ block: 'start' });       // 뷰포트 상단에 맞춤
element.scrollIntoView({ block: 'end' });         // 뷰포트 하단에 맞춤
element.scrollIntoView({ block: 'center' });      // 뷰포트 중앙에 맞춤
```

---

# 실전 패턴

## 동적 스크립트 로드 ⭐️⭐️⭐️⭐️

```typescript
// 외부 스크립트를 중복 없이 동적으로 로드
function loadScript(src: string): void {
  if (document.querySelector(`script[src="${src}"]`)) return;  // 이미 있으면 스킵

  const script = document.createElement('script');
  script.src   = src;
  script.async = true;
  document.body.appendChild(script);
}

// 로드 완료까지 기다려야 할 때 — Promise 래핑
function loadScriptAsync(src: string): Promise<void> {
  if (document.querySelector(`script[src="${src}"]`)) return Promise.resolve();

  return new Promise((resolve, reject) => {
    const script = document.createElement('script');
    script.src    = src;
    script.async  = true;
    script.onload  = () => resolve();
    script.onerror = () => reject(new Error(`스크립트 로드 실패: ${src}`));
    document.body.appendChild(script);
  });
}

await loadScriptAsync('https://www.youtube.com/iframe_api');
```

```txt
querySelector로 중복 확인하는 이유:
  컴포넌트가 여러 번 마운트/언마운트되면 스크립트가 중복으로 추가될 수 있음
  이미 <script> 태그가 있으면 추가하지 않음

script.async = true:
  HTML 파싱을 막지 않고 병렬로 다운로드 → 다운로드 완료 즉시 실행
  script.defer = true는 HTML 파싱 끝난 후 실행 (순서 보장)
```

## textarea 자동 높이 + Enter 전송 ⭐️⭐️⭐️⭐️

```tsx
const inputRef = useRef<HTMLTextAreaElement>(null);

<textarea
  ref={inputRef}
  value={body}
  rows={1}
  onChange={(e) => {
    setBody(e.target.value);

    // 자동 높이 조절
    e.target.style.height = 'auto';                                          // ① 먼저 초기화
    e.target.style.height = `${Math.min(e.target.scrollHeight, 120)}px`;    // ② 콘텐츠 높이로
  }}
  onKeyDown={(e) => {
    if (
      e.key === 'Enter' &&
      !e.shiftKey &&                        // Shift 없음 → 전송
      !e.nativeEvent.isComposing            // 한글 조합 중 아님
    ) {
      e.preventDefault();                   // 기본 줄바꿈 막기
      e.currentTarget.form?.requestSubmit();
    }
    // Shift+Enter → 줄바꿈 (기본 동작 유지)
  }}
  className="resize-none overflow-y-auto"
/>
```

```txt
e.currentTarget.form?.requestSubmit() 분해:

  e.currentTarget
    이벤트 리스너가 달린 요소 자체 (이 경우 textarea)
    e.target과 헷갈리기 쉬운데:
      e.target      = 실제로 이벤트가 발생한 요소 (버블링 중에 바뀔 수 있음)
      e.currentTarget = 리스너가 달린 요소 (항상 이 textarea)

  .form
    HTML form 요소들(<input> <textarea> <select> 등)이 가진 속성
    이 요소를 감싸고 있는 부모 <form> 요소를 가리킴
    <form>이 없으면 null

  ?.requestSubmit()
    ?.  → form이 null이면 조용히 무시 (에러 없음)
    requestSubmit() vs submit():
      form.submit()        → HTML 유효성 검사 안 함, onSubmit 이벤트 안 발생
      form.requestSubmit() → HTML 유효성 검사 실행, onSubmit 이벤트 발생
      → React에서 onSubmit 핸들러로 처리하려면 requestSubmit() 써야 함

  e.preventDefault()를 먼저 하는 이유:
    Enter의 기본 동작(textarea에 줄바꿈 삽입)을 막은 뒤
    form 제출을 직접 트리거
```

```txt
height = 'auto' 먼저 하는 이유:
  내용을 지웠을 때 높이가 줄어야 하는데
  바로 scrollHeight로 바꾸면 이전 height 기준으로 계산 → 안 줄어듦
  → 'auto'로 초기화 후 scrollHeight 측정 → 정확한 콘텐츠 높이

전송 후 포커스 & 높이 초기화:
  async function handleSubmit() {
    await sendMessage(body);
    setBody('');  // state 초기화 → 내용 비워짐

    if (inputRef.current) {
      inputRef.current.style.height = 'auto';  // 높이는 ref로 직접 초기화
      inputRef.current.focus();                // 포커스 복귀
    }
  }

  style.height는 React state가 아니라 DOM 속성
  → setBody('')만으로는 안 바뀜 → ref로 직접 초기화 필요
```

## input type="color" — opacity-0 커스텀 패턴 ⭐️⭐️⭐️

```tsx
<label className="relative size-9 cursor-pointer overflow-hidden rounded-full" aria-label="색 선택">
  <Palette className="absolute inset-0 m-auto size-4" aria-hidden />

  <input
    type="color"
    value={value}
    onChange={(e) => onChange(e.target.value)}
    className="absolute inset-0 size-full cursor-pointer opacity-0"
  />
</label>
```

```txt
opacity-0 패턴:
  display:none 또는 visibility:hidden → 클릭도 안 됨
  opacity:0 → 보이지 않지만 클릭은 됨

  → 커스텀 UI(아이콘) 뒤에 실제 input을 투명하게 깔아두는 패턴
    <input type="file">   — 파일 선택 버튼을 커스텀 버튼으로
    <input type="color">  — 색 선택기를 커스텀 아이콘으로
    <input type="range">  — 슬라이더를 커스텀 슬라이더로

aria-hidden on 아이콘:
  스크린리더가 <Palette> 아이콘을 읽지 않도록
  label의 aria-label="색 선택"으로 대신 설명
```

## 말풍선 줄바꿈 표시 — whitespace-pre-wrap ⭐️⭐️⭐️

```tsx
<span className="whitespace-pre-wrap break-words">
  {message.body}
</span>
```

```txt
whitespace-pre-wrap 없으면:
  "안녕\n반가워" → HTML에서 "안녕반가워"로 표시
  HTML은 기본적으로 \n을 공백 하나로 처리

whitespace-pre-wrap 있으면:
  \n → 실제 줄바꿈으로 표시

break-words:
  긴 URL 같은 단어가 컨테이너를 벗어나지 않도록 줄바꿈
  채팅 말풍선에는 둘 다 필요
```

## aria-pressed — 선택 상태 접근성 ⭐️⭐️

```tsx
<button
  aria-pressed={value === color}  // true = 선택됨, false = 선택 안 됨
  onClick={() => onChange(color)}
>
  {color}
</button>
```

```txt
스크린리더에 "이 버튼이 현재 눌린(선택된) 상태인지"를 알림
  색상 팔레트에서 현재 선택된 색상 버튼
  탭/필터에서 현재 활성 탭
  좋아요/즐겨찾기 토글 버튼

aria-selected와 차이:
  aria-pressed  → button role에서 on/off 상태
  aria-selected → listbox, tablist에서 선택된 항목
```