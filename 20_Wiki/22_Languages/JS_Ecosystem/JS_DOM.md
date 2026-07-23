---
aliases:
  - DOM
  - document
  - createElement
  - appendChild
  - querySelector
  - scrollIntoView
tags:
  - JavaScript
related:
  - "[[00_JS_Ecosystem_HomePage]]"
  - "[[JS_BrowserAPI]]"
  - "[[TS_DOM_Events]]"
  - "[[TS_ImportType]]"
  - "[[JS_Canvas]]"
  - "[[React_useId]]"
---
# JS_DOM — DOM 조작

> [!info] 
> DOM(Document Object Model) = 브라우저가 HTML을 파싱해서 만든 트리 구조.
>  `document.querySelector` / `createElement` / `appendChild` 같은 메서드로 요소를 찾고, 만들고, 추가한다. 
>  React/Next.js를 써도 서드파티 스크립트 로드, 외부 라이브러리 연동 등에서 직접 DOM을 다뤄야 하는 경우가 있다.

---

# 요소 찾기 ⭐️⭐️⭐️⭐️

## querySelector / querySelectorAll

```javascript
// 첫 번째로 일치하는 요소 하나 — 없으면 null
document.querySelector('.btn-primary')
document.querySelector('#user-name')
document.querySelector('script[src="https://example.com/api.js"]')  // 속성 선택자

// 일치하는 모든 요소 — NodeList (배열처럼 iterate 가능, 없으면 빈 NodeList)
document.querySelectorAll('li.active')
document.querySelectorAll('[data-id]')   // data-id 속성이 있는 모든 요소
```

## CSS 선택자 문법 ⭐️⭐️⭐️

|선택자|의미|예시|
|---|---|---|
|`태그`|태그명|`script`|
|`.클래스`|클래스명|`.btn-primary`|
|`#아이디`|id|`#user-name`|
|`[속성]`|속성 존재 여부|`[async]`|
|`[속성="값"]`|속성 값 일치|`[src="https://..."]`|
|`[속성^="값"]`|속성 값이 ~로 시작|`[src^="https://"]`|
|`[속성*="값"]`|속성 값에 ~가 포함|`[src*="youtube"]`|
|`부모 자식`|하위 요소|`div .item`|
|`부모 > 자식`|직계 자식만|`ul > li`|
|`A, B`|A 또는 B|`.btn, .link`|

```javascript
// script[src="..."] 패턴 — 특정 src를 가진 script 태그가 이미 있는지 확인
if (!document.querySelector('script[src="https://www.youtube.com/iframe_api"]')) {
  // 없을 때만 추가
}
```

## getElementById / getElementsByClassName

```javascript
document.getElementById('user-name')         // id로 찾기 — querySelector('#user-name')과 동일
document.getElementsByClassName('btn')        // 클래스로 찾기 — 실시간 업데이트되는 HTMLCollection
document.getElementsByTagName('script')       // 태그명으로 찾기
```

```txt
querySelector vs getElementById:
  getElementById → id 하나만, 속도가 약간 빠름
  querySelector  → CSS 선택자 전부 사용 가능, 더 유연함
  → 실무에서는 querySelector로 통일하는 경우가 많음

querySelectorAll vs getElementsByClassName:
  querySelectorAll  → 정적 NodeList (DOM 변경돼도 안 바뀜)
  getElementsBy...  → 동적 HTMLCollection (DOM 변경되면 자동 업데이트)
  → 대부분 querySelectorAll이 더 예측 가능해서 권장
```

---

# 요소 만들기 ⭐️⭐️⭐️⭐️

```javascript
const el = document.createElement('태그명');

// 예시 — script 태그 동적 생성
const script = document.createElement('script');
script.src   = 'https://www.youtube.com/iframe_api';
script.async = true;

// 예시 — div 만들기
const div = document.createElement('div');
div.id        = 'my-container';
div.className = 'container active';
div.textContent = '안녕하세요';
```

## 속성 설정 방법 두 가지

```javascript
// 방법 1: 프로퍼티 직접 할당 (자주 쓰는 속성)
script.src   = 'https://example.com/api.js';
script.async = true;
script.defer = true;
input.value  = '초기값';
img.alt      = '이미지 설명';
a.href       = 'https://example.com';

// 방법 2: setAttribute (커스텀 속성, data-* 속성)
el.setAttribute('data-id', '123');
el.setAttribute('aria-label', '닫기 버튼');
el.getAttribute('data-id')  // '123'
el.removeAttribute('disabled');
el.hasAttribute('async')    // true / false
```

```txt
프로퍼티 직접 할당 vs setAttribute:
  src / href / async / defer / value / checked 같은 표준 속성
  → 프로퍼티로 직접 할당 (타입 안전, 자동완성 됨)

  data-* 속성, aria-* 속성, 커스텀 속성
  → setAttribute 사용 (프로퍼티로 접근 불가)

  script.async = true   → boolean 타입으로 설정됨 (권장)
  script.setAttribute('async', '')  → 빈 문자열로 설정됨 (HTML에서 async는 존재 자체가 true)
```

---

# 요소 추가 / 이동 / 삭제 ⭐️⭐️⭐️⭐️

## 추가

```javascript
// 맨 끝에 추가
document.body.appendChild(script);
parentEl.appendChild(childEl);

// 특정 위치에 추가
parentEl.prepend(el)                        // 맨 앞에
parentEl.insertBefore(newEl, referenceEl)   // referenceEl 앞에
referenceEl.insertAdjacentElement('afterend', newEl)  // referenceEl 뒤에
```

## insertAdjacentElement 위치 옵션

```javascript
el.insertAdjacentElement('beforebegin', newEl)  // el 바로 앞 (형제)
el.insertAdjacentElement('afterbegin', newEl)   // el 안, 첫 번째 자식 앞
el.insertAdjacentElement('beforeend', newEl)    // el 안, 마지막 자식 뒤
el.insertAdjacentElement('afterend', newEl)     // el 바로 뒤 (형제)
```

## 삭제

```javascript
el.remove()                        // 요소 자체를 제거
parentEl.removeChild(childEl)      // 부모에서 자식 제거 (el.remove()가 더 간결)
el.replaceWith(newEl)              // 다른 요소로 교체
```

---

# classList — 클래스 다루기 ⭐️⭐️⭐️

```javascript
el.classList.add('active')              // 클래스 추가
el.classList.remove('hidden')           // 클래스 제거
el.classList.toggle('open')             // 있으면 제거, 없으면 추가
el.classList.toggle('open', true)       // 강제 추가 (두 번째 인자로 조건 지정)
el.classList.toggle('open', false)      // 강제 제거
el.classList.contains('active')         // 있는지 확인 → true/false
el.classList.replace('old', 'new')      // 클래스 교체

// className으로 전체 교체 (기존 클래스 전부 날아감 — 주의)
el.className = 'btn btn-primary'
```

```txt
classList.toggle(클래스, 조건):
  조건이 true면 추가, false면 제거
  el.classList.toggle('active', isLoggedIn)  → isLoggedIn에 따라 추가/제거
```

---

# textContent vs innerHTML ⭐️⭐️⭐️

```javascript
el.textContent = '안녕하세요';    // 텍스트만 — HTML 태그를 문자 그대로 표시
el.innerHTML   = '<b>굵게</b>';  // HTML 파싱해서 렌더링
el.textContent                   // 읽기: 텍스트 내용 (태그 제거됨)
el.innerHTML                     // 읽기: HTML 문자열 그대로
```

```txt
⚠️ innerHTML에 사용자 입력을 그대로 넣으면 XSS 취약점
  el.innerHTML = userInput  ← 위험 (사용자가 <script>태그를 심을 수 있음)
  el.textContent = userInput ← 안전 (태그를 문자로 처리)

React에서도 dangerouslySetInnerHTML이 있는 이유가 이것
```

---

# 동적 스크립트 로드 패턴 ⭐️⭐️⭐️⭐️

```typescript
// 외부 스크립트를 중복 없이 동적으로 로드
function loadScript(src: string): void {
  // 이미 로드됐으면 다시 추가하지 않음
  if (document.querySelector(`script[src="${src}"]`)) return;

  const script = document.createElement('script');
  script.src   = src;
  script.async = true;
  document.body.appendChild(script);
}
```

```typescript
// 로드 완료까지 기다려야 할 때 — Promise 래핑
function loadScriptAsync(src: string): Promise<void> {
  // 이미 있으면 즉시 resolve
  if (document.querySelector(`script[src="${src}"]`)) {
    return Promise.resolve();
  }

  return new Promise((resolve, reject) => {
    const script = document.createElement('script');
    script.src    = src;
    script.async  = true;
    script.onload  = () => resolve();
    script.onerror = () => reject(new Error(`스크립트 로드 실패: ${src}`));
    document.body.appendChild(script);
  });
}

// 사용
await loadScriptAsync('https://www.youtube.com/iframe_api');
```

```txt
script[src="..."] 선택자로 중복 확인하는 이유:
  컴포넌트가 여러 번 마운트/언마운트되면 스크립트가 중복으로 추가될 수 있음
  이미 <script> 태그가 있으면 추가하지 않음

script.async = true:
  HTML 파싱을 막지 않고 병렬로 다운로드 → 다운로드 완료 즉시 실행
  script.defer = true는 HTML 파싱이 끝난 후 실행 (순서 보장)

Promise 래핑과 prev?.() 콜백 보존 패턴 → [[JS_Promise]] / [[JS_OptionalChaining]] 참고
```

---
# input type="color" — 네이티브 색상 선택기 ⭐️⭐️⭐️

```typescript
// 기본 — 기본 UI 그대로
<input type="color" value={color} onChange={(e) => setColor(e.target.value)} />
```

## label로 커스텀 스타일 적용 — opacity-0 패턴 ⭐️⭐️⭐️⭐️


```tsx
// 네이티브 input을 숨기고 label로 클릭 영역 만들기
<label
  className="relative size-9 cursor-pointer overflow-hidden rounded-full"
  aria-label="색 직접 선택">

  {/* 보이는 아이콘 */}
  <Palette className="absolute inset-0 m-auto size-4 text-neutral-400" aria-hidden />

  {/* 실제 input — 완전히 투명하게, 클릭 영역은 유지 */}
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
  display:none 또는 visibility:hidden 하면 클릭도 안 됨
  opacity:0 은 보이지 않지만 클릭은 됨 → 커스텀 UI 뒤에 깔아두는 패턴

  적용 예시:
    <input type="file">   — 파일 선택 버튼을 커스텀 버튼으로
    <input type="color">  — 색 선택기를 커스텀 아이콘으로
    <input type="range">  — 슬라이더를 커스텀 슬라이더로

  구조:
    label(상대 위치, overflow-hidden)
      ↑ 커스텀 UI (아이콘, 텍스트 등) — absolute, inset-0
      ↑ input (absolute, inset-0, size-full, opacity-0) — 클릭 영역 전체 커버

aria-hidden on 아이콘:
  스크린리더가 <Palette> 아이콘을 읽지 않도록
  label의 aria-label="색 직접 선택"으로 대신 설명
```
---
# aria-pressed — 선택 상태 접근성 ⭐️⭐️⭐️

```tsx
// 현재 선택된 항목을 접근성으로 알리기
<button
  aria-pressed={value === color}  // true면 "눌린 상태", false면 "눌리지 않은 상태"
  onClick={() => onChange(color)}
>
  {color}
</button>
```

```txt
aria-pressed:
  토글 버튼에서 현재 "선택됨/선택 안 됨" 상태를 스크린리더에 알림
  true  → "눌림" — 현재 선택된 상태
  false → "눌리지 않음" — 선택 안 된 상태

  언제 쓰는가:
    색상 팔레트에서 현재 선택된 색상 버튼
    탭/필터 버튼에서 현재 활성 탭
    좋아요/즐겨찾기 토글 버튼

  aria-selected와 차이:
    aria-pressed  → 버튼(button role)에서 on/off 상태
    aria-selected → 목록/탭(listbox, tablist)에서 선택된 항목
```

---

# Pointer Events — 마우스 · 터치 · 펜 통합 ⭐️⭐️⭐️⭐️

```typescript
// Pointer Events = 마우스 / 터치 / 펜을 하나의 API로 처리
// onMouseDown/onTouchStart를 따로 쓰지 않아도 됨

element.addEventListener('pointerdown', (e: PointerEvent) => {
  e.pointerId   // 이 포인터의 고유 ID (멀티터치 구분)
  e.clientX     // 뷰포트 기준 X
  e.clientY     // 뷰포트 기준 Y
  e.pressure    // 펜 압력 (0~1, 마우스는 0 또는 0.5)
  e.pointerType // 'mouse' | 'touch' | 'pen'
});
```

## setPointerCapture — 포인터 캡처 ⭐️⭐️⭐️⭐️

```typescript
element.addEventListener('pointerdown', (e: PointerEvent) => {
  element.setPointerCapture(e.pointerId);
  // 이 시점부터 포인터가 element 밖으로 나가도
  // pointermove / pointerup 이벤트가 계속 element에 전달됨
});
```


```txt
setPointerCapture가 필요한 이유:
  드래그 중 마우스가 element 밖으로 나가면
  pointermove 이벤트가 다른 element로 넘어가서 추적이 끊김
  → setPointerCapture(pointerId) 로 "이 포인터를 내가 잡겠다" 선언
  → pointerup/pointercancel 이 올 때까지 이 element가 이벤트를 독점 수신

해제:
  pointerup 이나 pointercancel 이 오면 자동 해제됨
  또는 element.releasePointerCapture(e.pointerId) 수동 해제
```

## React에서 Pointer Events


```typescript
import { type PointerEvent as ReactPointerEvent } from 'react';
//              ↑ as 로 이름 변경 — 전역 PointerEvent (Web API)와 이름 충돌 방지

function DrawingCanvas() {
  const onPointerDown = (e: ReactPointerEvent<HTMLDivElement>) => {
    e.currentTarget.setPointerCapture(e.pointerId);
    // 그리기 시작
  };

  const onPointerMove = (e: ReactPointerEvent<HTMLDivElement>) => {
    if (e.buttons !== 1) return;  // 마우스 버튼 안 눌렸으면 무시
    // 그리기 중
  };

  const onPointerUp = (e: ReactPointerEvent<HTMLDivElement>) => {
    e.currentTarget.releasePointerCapture(e.pointerId);
    // 그리기 완료
  };

  return (
    <div
      onPointerDown={onPointerDown}
      onPointerMove={onPointerMove}
      onPointerUp={onPointerUp}
      style={{ touchAction: 'none' }}  // 모바일에서 스크롤 방지
    />
  );
}
```


```txt
type PointerEvent as ReactPointerEvent:
  React의 PointerEvent 타입과 브라우저 전역 PointerEvent 타입 이름 충돌
  as 로 별칭을 붙여서 구분 → [[TS_ImportType]] 참고

touchAction: 'none':
  모바일에서 pointer 이벤트와 스크롤이 충돌
  none으로 설정하면 브라우저 기본 터치 동작 비활성화
  → 드래그/그리기가 의도대로 동작

e.buttons:
  현재 눌린 마우스 버튼 비트마스크
  0   버튼 없음
  1   좌클릭
  2   우클릭
  → onPointerMove에서 if (e.buttons !== 1) return 으로 좌클릭 드래그만 처리
```

---

# getBoundingClientRect — 요소의 화면 위치 ⭐️⭐️⭐️

```typescript
const el = document.querySelector('.my-element')!;
const rect = el.getBoundingClientRect();

rect.left    // 뷰포트 왼쪽 기준 X 시작
rect.top     // 뷰포트 위쪽 기준 Y 시작
rect.right   // left + width
rect.bottom  // top + height
rect.width   // 요소 너비
rect.height  // 요소 높이
rect.x       // left와 동일
rect.y       // top과 동일
```

```typescript
// 실전 — 클릭 위치를 요소 내부 좌표로 변환
function getRelativePosition(e: PointerEvent, el: HTMLElement) {
  const rect = el.getBoundingClientRect();
  return {
    x: (e.clientX - rect.left) / rect.width,   // 0~1 정규화
    y: (e.clientY - rect.top)  / rect.height,
  };
}
```
ReactNode

```txt
getBoundingClientRect():
  뷰포트(현재 보이는 화면) 기준 위치를 반환
  스크롤하면 값이 달라짐 → 클릭할 때마다 새로 호출해야 함

clientX/Y vs pageX/Y:
  clientX/Y   뷰포트 기준 (스크롤과 무관)
  pageX/Y     문서 기준 (스크롤 포함)
  → getBoundingClientRect()도 뷰포트 기준 → clientX/Y와 함께 쓸 것
```
---

# 스크롤 — scrollIntoView ⭐️⭐️⭐️⭐️

```typescript
element.scrollIntoView();                           // 기본 — 요소가 보이도록 스크롤
element.scrollIntoView({ behavior: 'smooth' });     // 부드러운 애니메이션
element.scrollIntoView({ behavior: 'instant' });    // 즉시 (기본값)
element.scrollIntoView({ block: 'end' });           // 요소를 뷰포트 하단에 맞춤
element.scrollIntoView({ block: 'start' });         // 요소를 뷰포트 상단에 맞춤
element.scrollIntoView({ block: 'center' });        // 요소를 뷰포트 중앙에 맞춤
```

|옵션|값|의미|
|---|---|---|
|`behavior`|`'smooth'` / `'instant'`|스크롤 애니메이션 여부|
|`block`|`'start'` / `'center'` / `'end'` / `'nearest'`|수직 정렬 위치|
|`inline`|`'start'` / `'center'` / `'end'` / `'nearest'`|수평 정렬 위치|

## 채팅 "맨 아래로 스크롤" 패턴 ⭐️⭐️⭐️⭐️

```tsx
// 메시지 목록 맨 아래에 빈 div를 두고 ref로 참조
function ChatRoom() {
  const bottomRef = useRef<HTMLDivElement>(null);

  // messages.length가 바뀔 때마다 맨 아래로 스크롤
  useEffect(() => {
    bottomRef.current?.scrollIntoView({ behavior: 'smooth' });
  }, [messages.length]);

  return (
    <div className="overflow-y-auto">
      {messages.map((m) => <MessageItem key={m.id} message={m} />)}
      <div ref={bottomRef} />  {/* 스크롤 타깃 — 눈에 안 보이는 빈 div */}
    </div>
  );
}
```

```txt
빈 div를 마지막에 두는 이유:
  마지막 메시지 자체에 ref를 달면 메시지가 바뀔 때마다 ref 재연결 필요
  빈 div 하나를 항상 맨 아래에 두고 그 요소를 scrollIntoView 하면
  메시지 수와 무관하게 항상 맨 아래로 스크롤됨

?.  (옵셔널 체이닝):
  bottomRef.current가 아직 null일 수 있음 (마운트 전)
  ?.scrollIntoView()로 null이면 조용히 무시

behavior: 'smooth':
  애니메이션으로 부드럽게 스크롤
  새 메시지가 쏟아질 때는 'instant'가 나을 수 있음
  (smooth면 이전 스크롤이 끝나기 전에 다음이 시작돼서 버벅일 수 있음)

messages.length가 deps인 이유:
  messages 배열 자체를 deps에 넣으면 배열 참조가 바뀔 때마다 실행
  messages.length (숫자)를 deps에 넣으면 실제로 개수가 바뀔 때만 실행
  새 메시지가 추가됐을 때(length 증가)만 스크롤 → 의도에 맞음
```

---

# 자주 쓰는 document 속성

```javascript
document.title                    // 페이지 제목 (읽기/쓰기)
document.head                     // <head> 요소
document.body                     // <body> 요소
document.documentElement          // <html> 요소
document.readyState               // 'loading' / 'interactive' / 'complete'
```

---

# 한눈에

```txt
요소 찾기:
  querySelector(선택자)      → 첫 번째 일치, 없으면 null
  querySelectorAll(선택자)   → 전부, 없으면 빈 NodeList
  [속성="값"] 선택자         → 특정 속성값을 가진 요소 탐색

요소 만들기:
  document.createElement('태그')  → 새 요소 생성
  el.src / el.async = ...        → 표준 속성은 프로퍼티로
  el.setAttribute('data-x', 'y') → 커스텀/aria 속성은 setAttribute

요소 추가/삭제:
  parentEl.appendChild(el)   → 맨 끝에 추가
  parentEl.prepend(el)       → 맨 앞에 추가
  el.remove()                → 자신을 제거

classList:
  add / remove / toggle / contains / replace

textContent vs innerHTML:
  textContent → 텍스트만 (XSS 안전)
  innerHTML   → HTML 파싱 (사용자 입력 직접 넣으면 XSS 위험)

동적 스크립트 로드:
  querySelector로 중복 확인 → createElement → src/async 설정 → body.appendChild
  완료 대기 필요 시 → Promise + onload/onerror 래핑 ([[JS_Promise]] 참고)

스크롤:
  el.scrollIntoView({ behavior: 'smooth' })  요소가 보이도록 스크롤
  채팅 맨 아래로 → 빈 div ref + useEffect([messages.length])
```