---
aliases: [브라우저 API, document, navigator, window]
tags:
  - JavaScript
related:
  - "[[00_JS_Ecosystem_HomePage]]"
  - "[[JS_CustomEvent]]"
  - "[[NextJS_Routing]]"
  - "[[NextJS_TokenStorage]]"
---
# JS_BrowserAPI — 브라우저 내장 API

> [!info] 
>  브라우저가 JS에 기본으로 제공하는 API들이다. 설치 없이 바로 사용할 수 있고, React/Next.js 어디서든 동일하게 동작한다.

# 빠르게 찾기 ⭐️⭐️⭐️

| 카테고리        | API                                                             |
| ----------- | --------------------------------------------------------------- |
| 환경/기능 감지 패턴 | `typeof window/navigator !== 'undefined'`                       |
| 다이얼로그       | `confirm` / `alert` / `prompt`                                  |
| 페이지 이동      | `location` (origin으로 절대 URL 만들기 포함)                             |
| 저장소         | `localStorage` / `sessionStorage`                               |
| DOM 조작      | `document`, `classList`, `dataset`, `style`                     |
| 시스템 설정 감지   | `matchMedia` (다크모드 등)                                           |
| 타이머         | `setTimeout` / `setInterval`                                    |
| 디바이스/사용자 정보 | `navigator` (클립보드, 공유, 언어 등), `geolocation`                     |
| URL/쿼리스트링   | `URLSearchParams`                                               |
| 파일          | `File` 미리보기 (`createObjectURL`)                                 |
| 창 제어        | `window.open` / `window.scroll`                                 |
| 컴포넌트 간 동기화  | → 별도 노트 [[JS_CustomEvent]] (`dispatchEvent`/`addEventListener`) |

---

# typeof X !== 'undefined' — 환경/기능 감지 패턴 ⭐️⭐️⭐️⭐️

```txt
이 패턴 자체는 Next.js 전용도, 브라우저 API 하나도 아님 —
"지금 이 전역 식별자가 이 환경에 존재하는가"를 안전하게 확인하는 범용 JS 패턴
window/document/navigator/localStorage 등 이 노트의 거의 모든 API 앞에 반복해서 등장함
```

## 왜 typeof로 확인하나 — ReferenceError 회피 ⭐️⭐️⭐️

```typescript
if (window) { ... }                          // ❌ window가 선언 안 된 환경(Node.js)에서
                                              //    ReferenceError로 그 즉시 코드가 죽음
if (typeof window !== 'undefined') { ... }   // ✅ 선언조차 안 된 식별자에도 안전
```

```txt
보통 선언 안 된 변수를 참조하면 ReferenceError가 남
하지만 typeof는 JS에서 유일하게 "선언조차 안 된 식별자"에 적용해도 에러를 안 던지고
그냥 문자열 'undefined'를 돌려주는 연산자임 — 그래서 존재 여부를 안전하게 먼저 물어볼 수 있음
```

## 세 가지 쓰임 — 문법은 같아도 목적은 다름 ⭐️⭐️⭐️⭐️

|상황|예시|확인하는 것|
|---|---|---|
|① 서버 환경 감지|`typeof window !== 'undefined'`|Node.js(SSR)에는 window/document/localStorage 자체가 없음|
|② 기능 지원 여부 감지|`navigator.share`처럼 값 자체로 체크, 또는 `typeof X === 'function'`|브라우저에서 실행 중이어도, 모든 브라우저가 그 기능을 지원하진 않음|
|③ 모듈 최상단 코드|파일 맨 위에서 바로 `window`를 참조하는 경우|import되는 순간(서버든 클라이언트든) 즉시 실행돼서 더 위험함|

```txt
①은 "이 코드가 지금 어디서 실행되는가"의 문제, ②는 "이 브라우저가 이 기능을 갖고 있는가"의 문제 —
완전히 다른 질문인데 똑같이 typeof/존재 확인 문법을 쓰기 때문에 헷갈리기 쉬움
```

## 의외의 포인트 — useEffect 안에서는 ①이 보통 필요 없음 ⭐️⭐️⭐️

```txt
useEffect의 콜백은 정의상 컴포넌트가 브라우저에 마운트된 "이후"에만 실행됨 — SSR 중에는 절대 안 돎
→ useEffect 안에서 window/document를 쓸 때, typeof window 체크(①)는 대부분 불필요

①이 진짜 필요한 자리:
  렌더링 중에 바로 실행되는 코드 (컴포넌트 함수 본문 자체)
  모듈 최상단 (import되는 즉시 실행되는 코드)
  누가 어디서 호출할지 모르는 범용 유틸 함수 (서버/클라이언트 양쪽에서 호출될 가능성이 있는 함수)

반면 ②(기능 지원 여부)는 useEffect/이벤트 핸들러 안이어도 여전히 필요함 —
"브라우저에서 실행 중"이라는 건 보장돼도, "이 브라우저가 이 API를 지원한다"는 보장은 별개이기 때문
(아래 navigator.share가 정확히 이 ②번 사례)
```

---

# 다이얼로그 — confirm / alert / prompt

```typescript
const ok   = window.confirm('정말 삭제하시겠습니까?');   // 확인/취소 → boolean
window.alert('저장되었습니다.');                          // 알림만
const name = window.prompt('이름을 입력하세요.');         // 입력 → string | null
```

|함수|확인|취소/닫기|
|---|---|---|
|`confirm`|`true`|`false`|
|`prompt`|입력한 문자열|`null`|

```txt
window 생략 가능: confirm(...) === window.confirm(...)

React 사용 예:
  onClick={() => { if (confirm('삭제할까요?')) deleteItem(id); }}
```

## window.confirm vs 커스텀 모달

| |`window.confirm`|커스텀 모달|
|---|---|---|
|디자인|브라우저 기본, 변경 불가|자유|
|비동기 작업|어려움 (블로킹)|자연스러움|
|접근성(aria)|제어 불가|직접 제어|
|적합한 곳|내부 어드민, 빠른 구현|사용자 대면 서비스|

```tsx
// 커스텀 confirm 모달 — State 로 제어
const [confirmOpen, setConfirmOpen] = useState(false);
const [targetId, setTargetId] = useState<number | null>(null);

const handleDeleteClick = (id: number) => { setTargetId(id); setConfirmOpen(true); };
const handleConfirm = async () => {
  if (!targetId) return;
  await deleteItem(targetId);
  setConfirmOpen(false);
};

{confirmOpen && (
  <div role="dialog" aria-modal="true">
    <button onClick={handleConfirm}>확인</button>
    <button onClick={() => setConfirmOpen(false)}>취소</button>
  </div>
)}
```

```txt
role="dialog" / aria-modal="true" → 스크린리더에 모달임을 알림
```

---

# 페이지 이동 — window.location

```typescript
location.href        // 전체 URL
location.pathname    // 경로만 (/movie/1)
location.search       // 쿼리 (?title=abc)
location.origin      // 도메인 (https://example.com)

location.href = '/login';     // 이동 (히스토리 남음, 새로고침 발생)
location.replace('/login');   // 이동 (히스토리 교체)
location.reload();            // 새로고침
```

```txt
Next.js 에서는 보통 이걸 대신 사용 — 자세한 비교는 [[NextJS_Routing]] 참고:
  location.href = url   →  router.push(url)     (SPA, 새로고침 없음)
  location.reload()     →  router.refresh()

  의도적으로 "완전 새로고침" 이동이 필요할 때만 location 직접 사용
```

## location.origin — 절대 URL 만들기 ⭐️

```typescript
const shareUrl =
  typeof window !== 'undefined'
    ? `${window.location.origin}/posts/${id}`
    : `/posts/${id}`;
```

```txt
location.origin이 주는 것: 프로토콜+도메인+포트까지 합친 부분만 (예: 'https://example.com')
  pathname처럼 "지금 보고 있는 경로"가 아니라 "이 사이트의 기준 주소" 자체

왜 필요한가: /posts/3 같은 상대 경로는 그 사이트 안에서만 의미가 있음
  공유 버튼처럼 "이 링크를 메신저·SNS 등 바깥으로 그대로 복사해서 보낼 때"는
  상대 경로로는 안 되고 https://example.com/posts/3 같은 완전한 절대 URL이 필요함

typeof window 체크를 같이 두는 이유(위 "환경 감지" ① 패턴):
  이 함수가 서버에서도 호출될 가능성이 있다면(메타데이터 생성 등) origin을 미리 알 방법이 없어서
  상대 경로로 대신함 — 클라이언트에서만 호출된다는 게 확실하면 이 체크는 생략해도 됨
```

---

# 저장소 — localStorage / sessionStorage ⭐️

```txt
둘은 저장 기간만 다르고, 메서드(setItem/getItem/removeItem/clear)는 완전히 동일
→ 하나만 익히면 둘 다 쓸 수 있음
```

## 기본 4종 메서드

```typescript
// ① 저장 — setItem(key, value)
localStorage.setItem('token', 'eyJhb...');
sessionStorage.setItem('draft', JSON.stringify(formData));

// ② 읽기 — getItem(key) → 값이 string 으로 반환, 없으면 null
const token = localStorage.getItem('token');
const draft = sessionStorage.getItem('draft');

// ③ 삭제(1개) — removeItem(key)
localStorage.removeItem('token');
sessionStorage.removeItem('draft');

// ④ 전체 삭제 — clear()
localStorage.clear();
sessionStorage.clear();
```

|메서드|역할|반환값|
|---|---|---|
|`setItem(key, value)`|저장 (이미 있으면 덮어씀)|`undefined`|
|`getItem(key)`|읽기|`string` 또는 `null`(없을 때)|
|`removeItem(key)`|해당 key 1개만 삭제|`undefined`|
|`clear()`|저장된 것 전부 삭제|`undefined`|

## localStorage vs sessionStorage — 유지 기간만 다름

| |유지 기간|공유 범위|
|---|---|---|
|`localStorage`|브라우저를 완전히 닫아도 유지|같은 도메인이면 모든 탭에서 공유|
|`sessionStorage`|탭을 닫는 순간 삭제|그 탭 안에서만 (새 탭 열면 안 보임)|

```txt
선택 기준:
  로그인 토큰처럼 "다음에 다시 와도 유지"      → localStorage
  작성 중인 폼처럼 "이 탭에서만, 새로고침까지만" → sessionStorage

실제 토큰 저장 적용 사례(왜 함수로 감싸는지, SSR에서 왜 못 읽는지 등)는 [[NextJS_TokenStorage]] 참고
```

## 객체 / 숫자 저장 — 직렬화 필수

```typescript
// ❌ 그냥 넣으면 안 됨 — Storage 는 string 만 저장 가능
localStorage.setItem('count', 5);          // 내부적으로 "5" 로 강제 변환됨 (객체는 "[object Object]" 가 되어버려 깨짐)

// ✅ 객체/숫자는 JSON.stringify 로 문자열화해서 저장
localStorage.setItem('count', JSON.stringify(5));
localStorage.setItem('user', JSON.stringify({ id: 1, name: '공이' }));

// ✅ 꺼낼 때는 JSON.parse 로 복원
const count = JSON.parse(localStorage.getItem('count') ?? '0');        // null 대비 기본값
const user  = JSON.parse(localStorage.getItem('user')  ?? 'null');
```

```txt
?? '0' / ?? 'null' 을 붙이는 이유:
  getItem 은 키가 없으면 null 을 반환 → JSON.parse(null) 은 에러가 아니라 null 을 그대로 반환하지만,
  숫자/객체가 와야 할 자리에 null 이 들어가면 이후 로직에서 타입 에러 위험 → 기본값으로 방어
```

## 키 존재 여부 확인하는 두 가지 방법

```typescript
// 방법 1 — null 체크 (가장 흔함)
if (localStorage.getItem('token') !== null) { /* 로그인 상태 */ }

// 방법 2 — Storage.key() 와 length 로 전체 순회 (거의 안 씀, 참고용)
for (let i = 0; i < localStorage.length; i++) {
  console.log(localStorage.key(i));
}
```

```txt
⚠️ 보안: httpOnly 쿠키와 달리 JS 로 직접 접근 가능 → XSS 공격에 취약
   저장 위치별 트레이드오프(localStorage vs httpOnly 쿠키)는 [[NextJS_TokenStorage]] 참고
```

## React 에서 사용 — 초기값 / 이벤트 동기화

```typescript
// 초기값으로 사용 — 함수로 감싸면 첫 렌더링 1번만 실행됨 (lazy initial state)
const [theme, setTheme] = useState(() => localStorage.getItem('theme') ?? 'light');

// 값 바뀔 때마다 저장도 같이
const toggleTheme = () => {
  const next = theme === 'light' ? 'dark' : 'light';
  setTheme(next);
  localStorage.setItem('theme', next);
};
```

```txt
주의: localStorage.setItem 으로 값을 바꿔도 같은 탭의 React state 는 자동으로 안 바뀜
     (storage 이벤트는 "다른 탭"에서 바뀌었을 때만 발생) → 같은 탭이면 setState 를 직접 같이 호출해야 함
     같은 탭 안에서 여러 컴포넌트에 동기화해야 한다면 [[JS_CustomEvent]] 패턴이 더 적합
```

---

# DOM 조작 — document

```typescript
document.title = '영화 목록 | MyApp';

const el  = document.getElementById('my-input');
const el2 = document.querySelector('.card');

el.classList.add('active');
el.classList.toggle('active');
```

```txt
React 에서는 보통 useRef 로 대체, document.title 은 useEffect 안에서:
  useEffect(() => { document.title = `${movie.title} | MyApp`; }, [movie.title]);
```

## document.documentElement — `<html>` 태그 자체 ⭐️

```typescript
document.documentElement   // <html> 엘리먼트
document.body               // <body> 엘리먼트
document.head               // <head> 엘리먼트
```

```txt
document.body 와 다른 점:
  documentElement = <html> (body 의 부모, head 와 형제)
  body            = <body> 만

다크모드 같은 "페이지 전체" 설정을 줄 때 documentElement 를 쓰는 이유:
  html 에 클래스/속성을 걸어야 head 안의 <style> 선택자(html.dark, [data-theme]) 와
  body 전체에 동시에 영향을 줄 수 있음
  body 에만 걸면 head/scrollbar 같은 body 바깥 영역엔 적용 안 됨
```

## classList.toggle 의 두 번째 인자 — force ⭐️

```typescript
el.classList.toggle('dark');          // 있으면 제거, 없으면 추가 (그냥 반전)
el.classList.toggle('dark', isDark);  // isDark 값에 따라 "확정"
//                          ↑ true 면 무조건 추가, false 면 무조건 제거 (반전 아님)
```

```txt
2번째 인자(force) 없이 그냥 toggle('dark') 만 쓰면 안 되는 이유:
  toggle 은 "현재 상태를 뒤집기" 가 기본 동작
  → 같은 코드가 두 번 실행되면 클래스가 있다/없다를 왔다갔다 반복함 (의도와 다를 수 있음)

  toggle('dark', isDark) 처럼 두 번째 인자를 주면:
  → isDark 가 true 인 동안 이 코드를 몇 번 실행해도 항상 'dark' 가 "있는" 상태로 고정됨
  → "현재 변수 값에 맞춰 강제로 설정" 하고 싶을 때는 항상 force 인자를 같이 씀
```

## dataset — data-* 속성 다루기 ⭐️

```typescript
el.dataset.theme = 'dark';        // <html data-theme="dark"> 로 설정됨
el.dataset.theme                  // 'dark' 읽기

// camelCase ↔ kebab-case 자동 변환
el.dataset.userId = '42';         // <html data-user-id="42">
el.dataset.userId                 // '42'
```

```txt
dataset 이 가리키는 것:
  HTML data-속성이름="값"  ↔  JS 에서는 el.dataset.속성이름 (camelCase)
  data-user-id  ↔  dataset.userId
  data-theme    ↔  dataset.theme

왜 className 대신 data-* 를 쓰기도 하나:
  className 은 "있다/없다" 정보만 표현하기 좋음 (다크면 dark 클래스 있음/없음)
  data-* 는 "어떤 값인지" 까지 표현 가능 (data-theme="system"/"light"/"dark" 3가지 값)
  → CSS 에서도 [data-theme="dark"] { ... } 처럼 값 기준 선택자 사용 가능
```

## style — 인라인 스타일 직접 조작

```typescript
el.style.color = 'red';
el.style.backgroundColor = 'black';     // CSS background-color → camelCase
el.style.colorScheme = isDark ? 'dark' : 'light';
```

```txt
CSS 속성명(kebab-case) → JS 에서는 camelCase 로 변환해서 씀
  background-color → backgroundColor
  font-size        → fontSize
  border-radius    → borderRadius
```

|CSS 속성|`el.style.*`|예시 값|
|---|---|---|
|`color`|`color`|`'red'`, `'#333'`|
|`background-color`|`backgroundColor`|`'black'`, `'#fff'`|
|`font-size`|`fontSize`|`'16px'`, `'1.2rem'`|
|`font-weight`|`fontWeight`|`'bold'`, `600`|
|`display`|`display`|`'none'`, `'flex'`|
|`width` / `height`|`width` / `height`|`'100px'`, `'50%'`|
|`margin` / `padding`|`margin` / `padding`|`'10px'`, `'1rem 2rem'`|
|`border-radius`|`borderRadius`|`'8px'`|
|`opacity`|`opacity`|`'0.5'`|
|`transform`|`transform`|`'translateX(10px)'`|
|`z-index`|`zIndex`|`'10'`|
|`color-scheme`|`colorScheme`|`'dark'`, `'light'`|

```txt
⚠️ 숫자만 쓰면 안 되는 경우가 많음 — 단위 포함한 문자열이어야 함
  el.style.width = 100        ❌ 적용 안 됨
  el.style.width = '100px'    ✅
```

## classList.toggle vs style.colorScheme — 둘 다 쓰는 이유

|방법|역할|적용 대상|
|---|---|---|
|`classList.toggle('dark', isDark)`|내가 작성한 CSS 클래스(`.dark { ... }`)를 켜고 끔|내가 직접 스타일링한 요소들|
|`style.colorScheme = 'dark'`|브라우저에게 "지금 다크/라이트 모드" 라고 알림|스크롤바·체크박스·셀렉트 박스 같은 브라우저 기본 UI|

```txt
classList 만 쓰면: 내가 만든 컴포넌트는 다크모드 되는데
                   브라우저가 그려주는 기본 UI(스크롤바 등)는 그대로 라이트로 남음
→ 다크모드를 제대로 구현하려면 보통 이 둘을 함께 씀
```

## 종합 — applyTheme 함수 ⭐️

```typescript
export const applyTheme = (preference: ThemePreference) => {
  const root = document.documentElement;        // <html> 전체에 적용
  const isDark = resolveIsDark(preference);      // 'system'이면 OS 설정 확인 → boolean

  root.classList.toggle('dark', isDark);         // 내가 만든 .dark CSS 규칙 on/off
  root.dataset.theme = preference;               // data-theme="system"/"light"/"dark" 값 기록
  root.style.colorScheme = isDark ? 'dark' : 'light';  // 브라우저 기본 UI도 맞춤
};
```

```txt
한 줄씩 역할이 다름:
  classList.toggle  → 내가 정의한 CSS 클래스 켜고 끄기 (boolean 만 표현)
  dataset.theme     → 실제 선택된 값 자체를 기록 (system 인지 직접 선택한 건지 구분 가능)
  style.colorScheme → 브라우저 네이티브 요소까지 다크/라이트 일치시킴

→ 셋 다 같은 root(html) 에 적용하는 이유는 documentElement 가
  페이지 전체에 영향을 줄 수 있는 가장 바깥 요소이기 때문
→ resolveIsDark 의 'system' 분기 처리는 아래 "시스템 설정 감지 — matchMedia" 섹션 참고
```

---

# 시스템 설정 감지 — window.matchMedia ⭐️

```txt
CSS 미디어 쿼리를 JS 코드 안에서 그대로 검사하는 API
"지금 이 미디어 쿼리에 해당하는 상태인가?" 를 true/false 로 알려줌
```

```typescript
const isDark = window.matchMedia('(prefers-color-scheme: dark)').matches;
// true  → OS/브라우저 설정이 다크 모드
// false → 라이트 모드
```

```txt
matchMedia(쿼리) 가 반환하는 MediaQueryList 객체:
  .matches    현재 그 쿼리에 해당하는지 (boolean)
  .media      넘긴 쿼리 문자열 그대로
  .addEventListener('change', cb)   ← 사용자가 설정을 바꾸면 실시간 감지 가능
```

## 자주 쓰는 미디어 쿼리

```typescript
window.matchMedia('(prefers-color-scheme: dark)').matches      // 다크 모드 선호
window.matchMedia('(max-width: 768px)').matches                 // 화면 너비 768px 이하
window.matchMedia('(prefers-reduced-motion: reduce)').matches   // 애니메이션 최소화 선호
```

## 실전 — 시스템 다크모드 감지 함수 ⚠️

```typescript
// ⚠️ 자주 하는 실수 — 화살표 함수 { } 블록에서 return 빠뜨리기
export const getSystemPreferDark = () => {
  typeof window !== 'undefined' &&
    window.matchMedia('(prefers-color-scheme: dark)').matches;
  // ↑ return 이 없어서 이 줄의 결과가 그냥 버려짐 → 함수는 항상 undefined 반환
};

// ✅ return 추가
export const getSystemPreferDark = () => {
  return (
    typeof window !== 'undefined' &&
    window.matchMedia('(prefers-color-scheme: dark)').matches
  );
};
```

```txt
typeof window !== 'undefined' 를 같이 붙이는 이유는 위 "typeof X !== 'undefined' — 환경/기능 감지 패턴"
섹션의 ①(서버 환경 감지) 그대로임 — matchMedia도 confirm/localStorage처럼 브라우저 전용 API라
서버(Next.js Server Component, 빌드 타임 등)에는 window 자체가 없어서 먼저 확인이 필요함
```

## 실시간 변경 감지 — change 이벤트

```tsx
useEffect(() => {
  const mq = window.matchMedia('(prefers-color-scheme: dark)');

  const handleChange = (e: MediaQueryListEvent) => {
    setIsDark(e.matches);   // 사용자가 OS 설정을 바꾸는 순간 즉시 반영
  };

  mq.addEventListener('change', handleChange);
  return () => mq.removeEventListener('change', handleChange);
}, []);
```

```txt
ThemePreference 가 'system' 일 때 실제 적용 테마를 정하는 흐름:
  theme === 'system'  → getSystemPreferDark() 결과로 다크/라이트 결정
  theme === 'dark'    → 무조건 다크
  theme === 'light'   → 무조건 라이트
```

---

# 타이머 — setTimeout / setInterval

```typescript
const timer = setTimeout(() => console.log('3초 후'), 3000);
clearTimeout(timer);

const interval = setInterval(() => console.log('1초마다'), 1000);
clearInterval(interval);
```

```tsx
// React — 클린업 필수 (언마운트 시 취소)
useEffect(() => {
  const timer = setTimeout(() => setShowToast(false), 3000);
  return () => clearTimeout(timer);
}, []);
```

---

# 디바이스/사용자 정보 — navigator

```typescript
await navigator.clipboard.writeText('복사할 텍스트');   // 클립보드 복사
navigator.onLine      // true / false
navigator.language    // 'ko-KR'
```

```typescript
async function handleCopy(text: string) {
  try {
    await navigator.clipboard.writeText(text);
    alert('복사되었습니다.');
  } catch {
    alert('복사에 실패했습니다.');
  }
}
```

## navigator.share — 네이티브 공유 시트 ⭐️⭐️⭐️

```typescript
async function shareLink(url: string, title: string) {
  if (typeof navigator !== 'undefined' && navigator.share) {
    await navigator.share({ url, title });
    return;
  }
  // 지원 안 하면 클립보드 복사로 대체
  await navigator.clipboard.writeText(url);
}
```

```txt
navigator.share가 있을 때만 호출하는 이유 — 위 "기능 지원 여부 감지"(②) 패턴의 실제 사례:
  모바일 Safari/Chrome 등 일부 환경에서만 OS 네이티브 공유 시트(메시지·카톡 등 선택창)를 띄워주는 API
  데스크톱 브라우저 다수는 navigator.share 자체가 존재하지 않음 (undefined)
  → 있는지 먼저 확인하고, 없으면 클립보드 복사 같은 대체 동작으로 자연스럽게 전환(progressive enhancement)

typeof navigator !== 'undefined' 까지 같이 붙이는 이유:
  navigator.share를 확인하기도 전에 navigator 자체가 없는 환경(SSR)일 수 있음
  → ①(환경 감지) + ②(기능 감지) 두 체크가 동시에 필요한 경우
  (navigator.share만 확인하면 SSR 환경에서 "navigator is not defined"로 먼저 죽어버림)
```

## navigator — 실무에서 자주 보는 나머지들 ⭐️⭐️⭐️

|API|용도|
|---|---|
|`navigator.clipboard.readText()`|클립보드 읽기 — `writeText`의 반대, 붙여넣기 감지 등|
|`navigator.userAgent`|브라우저/OS 식별 문자열 — 분석/로깅에 흔히 쓰지만, 기능 분기에는 위 "기능 지원 여부 감지"(②)가 더 안전함|
|`navigator.vibrate(ms)`|모바일 기기 진동 — 알림/게임 피드백|
|`navigator.serviceWorker`|PWA 오프라인 캐싱·푸시 알림의 기반|
|`navigator.mediaDevices.getUserMedia()`|카메라/마이크 접근 — QR 스캔, 사진 업로드, 화상통화|
|`navigator.permissions.query()`|권한 상태를 프롬프트 없이 미리 확인|
|`navigator.sendBeacon()`|페이지를 떠나도 안전하게 전송되는 로그/분석 요청|

```typescript
// 클립보드 읽기 — writeText와 짝
const pasted = await navigator.clipboard.readText();
```

```typescript
// userAgent — "이 코드가 모바일에서 도는가"를 대략 판단할 때 흔히 보임
const isMobile = /Mobile|Android|iPhone/i.test(navigator.userAgent);
// ⚠️ userAgent 문자열은 브라우저마다 형식이 다르고 위조도 쉬움
//    "이 기능을 쓸 수 있는가"가 궁금하면 위 ②(기능 지원 여부 감지)가 더 정확하고 안전함
```

```typescript
// vibrate — 숫자(ms) 하나 또는 패턴 배열([진동, 멈춤, 진동...])
navigator.vibrate(200);           // 200ms 한 번
navigator.vibrate([100, 50, 100]); // 진동-멈춤-진동 패턴
```

```typescript
// serviceWorker 등록 — PWA의 시작점
if ('serviceWorker' in navigator) {
  navigator.serviceWorker.register('/sw.js');
}
```

```typescript
// getUserMedia — 카메라 스트림을 받아서 <video>에 연결
const stream = await navigator.mediaDevices.getUserMedia({ video: true });
videoEl.srcObject = stream;
```

```typescript
// permissions — 권한 요청 전에 현재 상태 확인 (요청 자체를 보내지 않음)
const status = await navigator.permissions.query({ name: 'geolocation' });
status.state; // 'granted' | 'denied' | 'prompt'
```

```typescript
// sendBeacon — 페이지를 떠나는 순간에도 끝까지 전송이 보장되는 분석/로그 요청
window.addEventListener('pagehide', () => {
  navigator.sendBeacon('/api/log', JSON.stringify({ event: 'leave' }));
});
```

```txt
이 중 'serviceWorker' in navigator 처럼 in 연산자로 체크하는 것도 위 "기능 지원 여부 감지"(②)의
또 다른 표현 방식임 — navigator.share처럼 값 자체를 truthy 체크하는 것과 같은 목적, 다른 문법일 뿐

serviceWorker/getUserMedia는 각자 PWA·미디어 스트림이라는 더 큰 주제의 입구일 뿐이라
실제로 그 기능을 쓰게 되면 별도 노트로 깊게 다룰 만함 — 지금은 "이런 게 있다"는 입구만 정리
```

## navigator.geolocation — 위치 정보

```typescript
navigator.geolocation.getCurrentPosition(
  (position) => {
    const { latitude, longitude, accuracy } = position.coords;
  },
  (error) => {
    console.error(error.code, error.message);
  },
);
```

|`error.code`|의미|
|---|---|
|1|`PERMISSION_DENIED` — 사용자 거부|
|2|`POSITION_UNAVAILABLE` — 위치 사용 불가|
|3|`TIMEOUT` — 시간 초과|

### pos.coords 주요 속성

```typescript
pos.coords.latitude;    // 위도
pos.coords.longitude;   // 경도
pos.coords.accuracy;    // 정확도(m) — 낮을수록 정확
```

```txt
날씨/지도는 보통 latitude·longitude 만 사용
accuracy 참고치: GPS 수 미터 / Wi-Fi 수십 미터 / IP 수 km
```

### 옵션

```typescript
navigator.geolocation.getCurrentPosition(successCb, errorCb, {
  enableHighAccuracy: false,    // true=GPS(정확·느림·배터리 소모) / false=Wi-Fi·IP(빠름)
  timeout:            10000,    // 최대 대기 시간(ms), 넘으면 TIMEOUT 에러
  maximumAge:         300000,   // 캐시된 위치 재사용 허용 시간(ms). 5분=배터리 절약
});
```

```txt
날씨 앱처럼 정밀도가 중요하지 않으면:
  enableHighAccuracy: false, maximumAge: 300000 정도가 적당
```

### async/await 패턴

```typescript
// getCurrentPosition 자체는 Promise 가 아님(콜백 기반)
// → 콜백 함수에 async 를 붙여서 내부에서 await 사용
navigator.geolocation.getCurrentPosition(
  async (pos) => {
    try {
      const { latitude, longitude } = pos.coords;
      const res  = await fetch(`/api/weather?lat=${latitude}&lon=${longitude}`);
      const data = await res.json();
      setWeather(data);
      setStatus('ready');
    } catch {
      setStatus('error');
    }
  },
  (err) => setStatus(err.code === 1 ? 'denied' : 'error'),
  { enableHighAccuracy: false, timeout: 10000, maximumAge: 300000 },
);
```

```txt
특징: 한 번만 가져옴(실시간 추적은 watchPosition) / HTTPS 필수(localhost 예외) / 사용자 허용 필요
```

### status 상태 머신 패턴

```typescript
// ❌ boolean 여러 개 — 조합이 늘어나 복잡해짐
const [isLoading, setIsLoading] = useState(false);
const [isError, setIsError]     = useState(false);

// ✅ 하나의 status 로 통합
const [status, setStatus] = useState<'idle'|'loading'|'error'|'ready'|'denied'>('idle');
```

```txt
idle → loading → ready / error / denied

장점: isLoading+isError 동시 true 같은 불가능한 조합 자체가 안 생김
     JSX 에서 if/switch 로 깔끔하게 분기

이 status 패턴 자체는 geolocation 전용이 아니라 비동기 작업 전반에 재사용 가능한 일반 패턴
```

### 실전 — WeatherByLocation

```tsx
export const WeatherByLocation = () => {
  const [weather, setWeather] = useState<Weather | null>(null);
  const [status, setStatus] = useState<'idle'|'loading'|'error'|'ready'|'denied'>('idle');

  useEffect(() => {
    if (!navigator.geolocation) { setStatus('error'); return; }
    setStatus('loading');

    navigator.geolocation.getCurrentPosition(
      async (pos) => {
        const { latitude: lat, longitude: lon } = pos.coords;
        const res  = await fetch(`/api/weather?lat=${lat}&lon=${lon}`);
        setWeather(await res.json());
        setStatus('ready');
      },
      (err) => setStatus(err.code === 1 ? 'denied' : 'error'),
    );
  }, []);

  if (status === 'idle' || status === 'loading') return <p>위치 확인 중...</p>;
  if (status === 'denied') return <p>위치 권한을 허용해주세요.</p>;
  if (status === 'error')  return <p>위치 정보를 가져올 수 없습니다.</p>;
  if (!weather) return null;

  return <div>{weather.weather} {weather.temperature}°C · 습도 {weather.humidity}%</div>;
};
```

---

# URL/쿼리스트링 — URLSearchParams

```txt
문자열로 직접 만들면: `?area=${area}` → 한글/특수문자에서 깨짐, 조건부 처리 불편
URLSearchParams 쓰면: 자동 인코딩 + 조건부 추가/삭제 메서드 내장 (브라우저+Node 18+ 기본 제공)
```

```typescript
const sp = new URLSearchParams();
sp.set('area', '서울');
sp.toString()   // 'area=%EC%84%9C%EC%9A%B8'  ← 자동 인코딩, encodeURIComponent 직접 안 써도 됨
```

|메서드|역할|
|---|---|
|`sp.set(k, v)`|설정 (덮어쓰기)|
|`sp.append(k, v)`|추가 (중복 허용)|
|`sp.get(k)`|읽기 (없으면 `null`)|
|`sp.has(k)`|존재 여부|
|`sp.delete(k)`|삭제|
|`sp.toString()`|`'k=v&k2=v2'` 문자열 변환|

```typescript
// 실전 — 조건부 파라미터 빌드
export function buildExhibitionParams(params: { area?: string; status?: string }) {
  const sp = new URLSearchParams();
  if (params.area)   sp.set('area', params.area);     // undefined/''는 추가 안 됨
  if (params.status) sp.set('status', params.status);
  return sp;
}

fetch(`/exhibitions?${buildExhibitionParams({ area: '서울' }).toString()}`);
```

```typescript
// 현재 URL 쿼리 읽기 — 브라우저
const sp = new URLSearchParams(window.location.search);
sp.get('area');

// Next.js Client Component — 자세한 내용은 [[NextJS_Routing]] 참고
import { useSearchParams } from 'next/navigation';
useSearchParams().get('area');
```

---

# 파일 — File 미리보기 (URL.createObjectURL)

```txt
File = <input type="file"> / 드래그 드롭으로 받은 파일 객체
서버 업로드 전에 화면에 미리보기를 띄우고 싶을 때 사용
```

```javascript
const file = e.target.files[0];
const previewUrl = URL.createObjectURL(file);
// 'blob:http://localhost:3000/...'  ← 브라우저 메모리의 파일을 가리키는 임시 URL
// <img src={previewUrl} /> 로 즉시 표시, 서버 업로드 불필요
```

```javascript
URL.revokeObjectURL(previewUrl);   // 메모리 해제
```

```txt
revoke 안 하면: 파일을 여러 번 바꿀 때마다 메모리에 계속 쌓임 (메모리 누수)
호출 시점: 업로드 끝났을 때 / 새 파일로 교체할 때
```

## 실전 — 업로드 + 미리보기 교체

```typescript
const applyPhotoFile = async (file: File) => {
  const localPreview = URL.createObjectURL(file);   // ① 즉시 로컬 미리보기
  setPhotoPreview(localPreview);

  try {
    const { url } = await uploadVisitPhoto(file);    // ② 서버 업로드
    setPhotoPreview(url);                            // ③ 서버 URL 로 교체
  } catch {
    setPhotoPreview('');
  } finally {
    URL.revokeObjectURL(localPreview);               // ④ 로컬 blob 은 항상 정리
  }
};
```

```txt
finally 에서 revoke 하는 이유: 성공(서버 URL 로 교체) / 실패(어차피 안 씀) 둘 다 local 정리 필요
```

---

# 창 제어 — window.open / window.scroll

```typescript
window.open('https://example.com', '_blank');                          // 새 탭
window.open('https://example.com', 'popup', 'width=600,height=400');   // 팝업
```

```txt
보안: window.open 으로 연 탭은 opener 접근 가능
  rel="noopener" 와 같은 효과를 코드로:
  const popup = window.open('...');
  if (popup) popup.opener = null;
```

```typescript
window.scrollTo({ top: 0, behavior: 'smooth' });
window.scrollY   // 현재 세로 스크롤 위치
```

```tsx
// React — 스크롤 감지 패턴 (클린업 필수)
useEffect(() => {
  const handleScroll = () => setScrollY(window.scrollY);
  window.addEventListener('scroll', handleScroll);
  return () => window.removeEventListener('scroll', handleScroll);
}, []);
```

---
# 데이터 복사 — structuredClone (깊은 복사) ⭐️⭐️⭐️⭐️

```txt
어디서 import하는 함수가 아니라, 브라우저와 최신 Node.js(17+)가 기본으로 제공하는 전역 함수
fetch/setTimeout처럼 그냥 바로 호출하면 됨 — import 없이 쓸 수 있어서
"어디서 가져오는 거지" 하고 찾아도 안 나오는 게 당연함
```

```typescript
const original = { user: { name: '홍길동' }, tags: ['a', 'b'] };
const copy = structuredClone(original);

copy.user.name = '김철수';
original.user.name; // '홍길동' — 원본은 안 바뀜 (완전히 독립된 복사본)
```

## 왜 스프레드(...)로는 부족할 때가 있나 — 얕은 복사 vs 깊은 복사 ⭐️⭐️⭐️⭐️


```typescript
const shallow = { ...original };
shallow.user.name = '김철수';
original.user.name; // '김철수' — 안쪽 객체는 같은 참조라 원본도 같이 바뀜 (의도와 다를 수 있음)
```

|방법|중첩된 객체/배열도 독립적으로 복사되는가|
|---|---|
|`{ ...obj }` / `[...arr]`|아니요 — 한 단계만 (얕은 복사) — [[JS_Operators]] 참고|
|`structuredClone(obj)`|네 — 몇 단계든 전부 새로 복사 (깊은 복사)|


```txt
{ ...obj }는 "한 단계"까지만 새로 복사함 — 중첩된 객체/배열은 원본과 같은 참조를 그대로 공유함
얕은 복사로 충분한 경우(중첩이 없는 평평한 객체)엔 스프레드가 더 짧고 흔하게 쓰임 —
"중첩된 부분까지 완전히 독립적인 복사본이 필요한가"가 둘을 가르는 기준
```

## 실전 — "되돌리기" 가능한 편집 상태 만들기 ⭐️⭐️⭐️⭐️

```tsx
useEffect(() => {
  if (!open || !card) return;
  setMode('view');
  setCustomization(structuredClone(card.customization));
}, [open, card]);
```


```txt
card.customization을 structuredClone 없이 그대로 setCustomization에 넘기면:
  state(customization)와 card.customization이 같은 객체를 가리키게 됨
  → 편집 폼에서 customization의 중첩 필드를 직접 바꾸면, 아직 저장도 안 했는데
    원본 card 데이터까지 같이 바뀌어버리는 위험이 생김 (둘이 같은 객체를 공유해서)

structuredClone으로 감싸면:
  card.customization과 완전히 독립된 "복사본"을 state로 들고 시작함
  → 편집 중에 마음대로 고쳐도 원본 card는 전혀 안 바뀜
  → 사용자가 "취소"를 누르면 그냥 이 state를 버리면 됨 (원본은 멀쩡하니까)
```

## 한계 — 복사할 수 없는 것들 ⭐️⭐️⭐️

```txt
structuredClone이 복사 못 하는 것들:
  함수(function) — 복사 시도하면 에러
  DOM 노드 — 에러
  클래스 인스턴스의 메서드(프로토타입 체인) — 데이터 필드는 복사되지만 메서드는 안 따라옴

→ 순수 데이터(객체/배열/문자열/숫자/Date/Map/Set 등)를 복사할 때만 적합
  함수나 클래스 인스턴스가 섞인 객체는 structuredClone으로 못 복사함
```

```txt
비교적 최신 API(2022년경부터 주요 브라우저/Node 17+ 지원) — 아주 오래된 환경까지 지원해야 한다면
typeof structuredClone === 'function'으로 존재 여부를 먼저 확인하는 게 안전함
(이런 기능 지원 여부 감지 패턴 자체는 위 "typeof X !== 'undefined' — 환경/기능 감지 패턴" 참고)
```


---

# 한눈에

|API|용도|반환|
|---|---|---|
|`typeof window !== 'undefined'`|서버 환경 감지 — ReferenceError 없이 안전하게 확인|boolean|
|`navigator.share`처럼 값으로 체크|기능 지원 여부 감지 — 환경 감지와는 다른 질문|boolean(존재 여부)|
|`confirm(msg)`|확인/취소|boolean|
|`prompt(msg)`|입력받기|string \| null|
|`location.href = url`|페이지 이동(새로고침)|-|
|`location.origin`|프로토콜+도메인+포트 — 절대 URL 만들 때|string|
|`localStorage.setItem/getItem`|영구 저장/읽기|- / string\|null|
|`sessionStorage.setItem/getItem`|탭 한정 저장/읽기|- / string\|null|
|`document.title = ...`|탭 제목 변경|-|
|`window.open(url, '_blank')`|새 탭 열기|-|
|`window.matchMedia(query).matches`|미디어 쿼리(다크모드 등) 검사 ⭐️|boolean|
|`navigator.clipboard.writeText(v)`|클립보드 복사|Promise|
|`navigator.share({ url, title })`|네이티브 공유 시트 (지원 안 하면 fallback 필요)|Promise|
|`navigator.geolocation.getCurrentPosition(cb)`|현재 위치|void|
|`URL.createObjectURL(file)`|파일 미리보기 URL|string(blob:)|
|`URL.revokeObjectURL(url)`|미리보기 URL 해제|void|
|`window.scrollTo(...)`|스크롤 이동|-|
|`setTimeout/setInterval`|지연/반복 실행|timerId|
|`new URLSearchParams()`|쿼리 파라미터 빌드|URLSearchParams|

```txt
컴포넌트 간 상태 동기화(dispatchEvent/addEventListener/CustomEvent)는 [[JS_CustomEvent]] 로 분리됨
```