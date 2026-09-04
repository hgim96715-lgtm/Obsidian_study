---
aliases: [스크린 리더, 인터랙티브 요소 중첩 금지, 접근성, ARIA, aria-label, aria-labelledby, aria-live, role, sr-only, tabIndex]
tags: [HTML, React]
relations:
  - "[[00_JS_Ecosystem_HomePage]]"
  - "[[React_Portal_Dialog]]"
  - "[[React_useId]]"
---

# HTML_ARIA — 접근성 · ARIA

>[!info]
> ARIA(Accessible Rich Internet Applications) = 스크린 리더 같은 보조 기술에게 HTML의 의미를 전달하는 속성 모음.
> ARIA는 3가지 역할을 한다: **숨기기** · **이름 붙이기** · **상태/역할 알리기**.
> 가능하면 ARIA보다 의미 있는 HTML 태그를 먼저 쓴다 — `<button>`, `<nav>`, `<main>` 이미 role을 내포함.

---

# 접근성이란 — 왜 필요한가 ⭐️⭐️⭐️⭐️

```txt
웹을 사용하는 모든 사람 중에는:
  시각 장애인  → 스크린 리더로 내용을 듣는다
  키보드만 사용 → 마우스 없이 탭 키로 이동
  저시력       → 고대비 모드, 화면 확대

스크린 리더:
  화면의 내용을 소리로 읽어주는 프로그램
  HTML 구조와 ARIA 속성을 보고 "지금 어디에 있는지, 뭘 할 수 있는지" 읽어줌

ARIA 속성이 하는 일:
  1. 숨기기   → aria-hidden, sr-only    "이건 읽지 마"/"이것만 읽어"
  2. 이름 붙이기 → aria-label, aria-labelledby  "이게 뭔지 알려줌"
  3. 상태·역할  → aria-live, aria-expanded, role  "지금 어떤 상태인지 알려줌"
```

---

# 숨기기 — aria-hidden · sr-only · display:none

## 세 가지 숨기기 비교 ⭐️⭐️⭐️⭐️

```txt
                  시각적으로 보임   스크린 리더가 읽음
display: none         ✕                 ✕        → 완전히 없앰
aria-hidden="true"    ✅                ✕        → 보조 기술에서만 숨김
sr-only (class)       ✕                ✅        → 스크린 리더만을 위한 텍스트

언제:
  display:none   → 진짜로 숨길 때 (접힌 메뉴, 비활성 탭)
  aria-hidden    → 장식 요소, 아이콘 — 시각적으론 있지만 읽을 필요 없음
  sr-only        → 스크린 리더 전용 설명 텍스트
```

## aria-hidden ⭐️⭐️⭐️⭐️

```tsx
// 장식용 점 · 구분선
<span className="size-1.5 rounded-full bg-brand-primary" aria-hidden />

// 아이콘 — 텍스트로 이미 설명될 때
<button>
  <svg aria-hidden>...</svg>   {/* 아이콘은 숨기고 */}
  알림                          {/* 텍스트만 읽힘 */}
</button>

// 숫자 뱃지 — aria-label에서 이미 전달했을 때 중복 방지
<button aria-label={`알림 ${unreadCount}개`}>
  <BellIcon aria-hidden />
  {unreadCount > 0 && (
    <span aria-hidden className="badge">{unreadCount}</span>
  )}
</button>
```

```txt
⚠️ 주의: aria-hidden="true"인 요소 안에 포커스 가능한 요소가 있으면 안 됨
  숨겨진 영역 안의 button은 키보드로 접근 불가 → 혼란 발생
```

## sr-only ⭐️⭐️⭐️⭐️

```tsx
// Tailwind CSS — 시각적으로는 안 보이지만 스크린 리더는 읽음
<button onClick={onDelete}>
  <TrashIcon aria-hidden />
  <span className="sr-only">게시글 삭제</span>
</button>
// 스크린 리더: "게시글 삭제, 버튼"
// 화면:        아이콘만 보임
```

```css
/* sr-only 구현 원리 */
.sr-only {
  position: absolute;
  width: 1px;
  height: 1px;
  padding: 0;
  margin: -1px;
  overflow: hidden;
  clip: rect(0,0,0,0);
  white-space: nowrap;
  border-width: 0;
}
/* display:none이 아님 → 스크린 리더는 렌더 트리에 있으면 읽음 */
/* 1px × 1px → 화면에서 사실상 안 보임 */
```

---

# 이름 붙이기 — aria-label · aria-labelledby · aria-describedby

## aria-label ⭐️⭐️⭐️⭐️

```tsx
// 텍스트 없는 버튼 (아이콘만)
<button aria-label="알림 설정">
  <BellIcon />
</button>
// 스크린 리더: "알림 설정, 버튼"

// 닫기 버튼 — ✕ 기호 대신 의미 있는 이름으로
<button aria-label="닫기" onClick={onClose}>✕</button>

// 상태에 따라 동적으로 변경
<button aria-label={unread ? '알림 (안 읽음)' : '알림'}>
  <BellIcon />
</button>

// input — label 없을 때
<input type="search" aria-label="게시글 검색" placeholder="검색어 입력" />
```

```txt
언제 aria-label이 필요한가:
  버튼·링크에 텍스트가 없을 때 (아이콘만 있을 때)
  같은 텍스트가 여러 개일 때  ("더 보기" 10개 → "홍길동의 게시글 더 보기")
  <label>이 없는 input

  <label htmlFor="id">가 있으면 aria-label 불필요 — label이 이미 이름 역할
```

## aria-labelledby · aria-describedby ⭐️⭐️⭐️

```tsx
// aria-labelledby — 화면에 이미 있는 텍스트로 이름 연결
<h2 id="dialog-title">삭제하시겠습니까?</h2>
<div role="dialog" aria-modal="true" aria-labelledby="dialog-title">
  ...
</div>
// 스크린 리더: "삭제하시겠습니까?, 대화상자"

// aria-describedby — 보조 설명 연결 (이름 아닌 설명)
<input
  id="email"
  type="email"
  aria-describedby={error ? 'email-error' : undefined}
  aria-invalid={!!error}
/>
{error && (
  <p id="email-error" role="alert">{error}</p>
)}
// 스크린 리더: "이메일, 입력 필드, 유효하지 않음, 올바른 이메일 형식이 아닙니다"
```

```txt
aria-label        → 직접 텍스트 입력
aria-labelledby   → 화면에 이미 있는 요소의 id를 이름으로 참조
aria-describedby  → 이름이 아닌 보충 설명 (에러 메시지, 힌트)

  요소 이름  → aria-label / aria-labelledby
  보충 설명  → aria-describedby
```

---

# 상태 알리기 — aria-live · aria-expanded · aria-invalid

## aria-live — 동적 업데이트 알림 ⭐️⭐️⭐️⭐️

```txt
문제:
  스크린 리더는 기본적으로 포커스가 있는 곳만 읽음
  → 날짜 선택 후 달력 상세가 바뀌어도 스크린 리더는 변화를 모름
  → 에러 메시지가 나타나도 포커스가 없으면 못 읽음

해결:
  aria-live = "이 영역의 내용이 바뀌면 알아서 읽어줘"
```

|값|언제 읽음|사용 상황|
|---|---|---|
|`"polite"`|현재 읽던 것 끝나고|날짜 선택 결과, 검색 결과 수, 저장 완료|
|`"assertive"`|즉시 중단하고|에러, 타임아웃, 긴급 알림|
|`"off"`|읽지 않음|기본값, 계속 바뀌는 영역|

```tsx
// 달력 상세 — 날짜 선택 시 조용히 업데이트 알림
<section
  className="movie-calendar-detail"
  aria-live="polite"
  aria-atomic="true"    // 영역 전체를 한 번에 읽음 (일부만 읽지 않음)
>
  {selectedDate ? (
    <MovieList date={selectedDate} movies={movies} />
  ) : (
    <p>날짜를 선택해주세요</p>
  )}
</section>
// 날짜가 바뀔 때마다 스크린 리더가 영역 전체를 다시 읽어줌

// 검색 결과 수
<p aria-live="polite">
  {count}개의 결과가 있습니다
</p>

// 에러 메시지 — 즉시 읽어야 함
<div aria-live="assertive" role="alert">
  {errorMessage}
</div>
```

```txt
aria-atomic="true":
  변경된 부분만 읽지 않고 aria-live 영역 전체를 읽음
  달력처럼 날짜·영화 목록이 같이 바뀔 때 → 전체를 한 덩어리로 읽는 게 자연스러움

role="alert" = aria-live="assertive" + aria-atomic="true" 단축
role="status" = aria-live="polite"   + aria-atomic="true" 단축

  → 짧은 상태 메시지는 role="status", 에러는 role="alert"로 단축 가능
```

## aria-expanded · aria-pressed · aria-selected ⭐️⭐️⭐️

```tsx
// 드롭다운 열림/닫힘 상태
<button
  aria-expanded={isOpen}
  aria-controls="menu-list"   // 제어하는 요소의 id
  onClick={() => setIsOpen(!isOpen)}
>
  메뉴
</button>
<ul id="menu-list" hidden={!isOpen}>...</ul>
// 스크린 리더: "메뉴, 버튼, 접혀있음" / "메뉴, 버튼, 펼쳐짐"

// 토글 버튼 (on/off 상태 유지)
<button aria-pressed={isMuted} onClick={toggleMute}>
  {isMuted ? '음소거 해제' : '음소거'}
</button>

// 탭·선택 항목
<button role="tab" aria-selected={activeTab === 'info'}>
  정보
</button>
```

```txt
aria-expanded  → 드롭다운, 아코디언, 서브메뉴 — 펼침/접힘
aria-pressed   → 토글 버튼 — 눌림/안눌림 (isMuted, isBookmarked)
aria-selected  → 탭, 리스트 선택 — 선택됨/안됨
aria-invalid   → 입력값 유효성 — true면 "유효하지 않음" 읽힘
```

## aria-invalid — 입력값 에러 ⭐️⭐️⭐️

```tsx
<input
  type="email"
  aria-invalid={!!error}           // 에러 있으면 "유효하지 않음"
  aria-describedby="email-error"   // 에러 메시지와 연결
/>
{error && (
  <p id="email-error" role="alert" className="text-red-500">
    {error}
  </p>
)}
```

---

# 역할 명시 — role ⭐️⭐️⭐️

```tsx
// HTML 태그로 의미 전달이 어려울 때
<div role="dialog" aria-modal="true" aria-labelledby="title">
  모달 내용
</div>

<div role="alert">
  {/* 즉시 읽힘 — role="alert" = aria-live="assertive" + aria-atomic="true" */}
  저장에 실패했습니다.
</div>

<span role="status">
  {/* role="status" = aria-live="polite" + aria-atomic="true" */}
  저장됐습니다.
</span>
```

## 자주 쓰는 role

|role|의미|HTML 대체|
|---|---|---|
|`dialog`|모달 다이얼로그|없음 (dialog 태그는 지원 제한)|
|`alert`|즉시 읽는 에러·경고|없음|
|`status`|작업 완료 상태 (덜 급함)|없음|
|`button`|div/span을 버튼처럼|`<button>` 쓰는 게 훨씬 나음|
|`navigation`|nav 영역|`<nav>`|
|`main`|메인 콘텐츠|`<main>`|
|`list`·`listitem`|목록·항목|`<ul>` · `<li>`|
|`tab`·`tabpanel`|탭 UI|없음|

```txt
가능하면 role 대신 의미 있는 HTML 태그 사용:
  <div role="button">     → <button>
  <div role="navigation"> → <nav>
  <div role="main">       → <main>

  HTML 태그 자체가 role을 내포하고 있음
  div/span에 role 붙이는 건 정말 어쩔 수 없을 때만
```

---

# 키보드 접근성 — tabIndex ⭐️⭐️⭐️

```tsx
// tabIndex
<div tabIndex={0}>    {/* 탭으로 포커스 가능하게 */}
<div tabIndex={-1}>   {/* 탭으로는 못 가지만 JS로 .focus() 가능 */}

// div를 클릭 가능하게 만들 때 — 키보드도 지원해야 함
<div
  role="button"
  tabIndex={0}
  onClick={handleClick}
  onKeyDown={(e) => {
    if (e.key === 'Enter' || e.key === ' ') handleClick();
  }}
>
  클릭
</div>
// ✅ 차라리 <button>을 쓰면 이 모든 게 기본 지원됨
```

```txt
tabIndex={-1} 활용:
  모달이 열릴 때 모달 컨테이너에 .focus() → 탭 포커스를 모달 안으로 가두기
  드롭다운 목록 항목에 tabIndex={-1} → JS로만 포커스 이동, 직접 탭은 안 됨
```

---

# 인터랙티브 요소 중첩 금지 ⭐️⭐️⭐️⭐️

```txt
HTML 규칙: button 안에 button 올 수 없음
interactive 요소 (button, a, input, select, textarea) 안에
또 다른 interactive 요소를 넣으면 안 됨

증상:
  클릭이 안 먹거나 부모 이벤트에만 먹힘
  카드만 뒤집히고 내부 버튼 토글이 안 됨
  "button cannot be a descendant of button" (hydration 경고)
```

```tsx
// ❌ 플립 컨테이너(button) 안에 button 중첩
<button className="room-flip" onClick={handleFlip}>
  <div className="room-flip-front">...</div>
  <div className="room-flip-back">...</div>
  <button onClick={handleMark}>찜</button>     {/* ❌ */}
  <button onClick={handleWatch}>봤어요</button> {/* ❌ */}
</button>

// ✅ 마크 버튼을 플립 밖으로 분리
<li className="room-movie">
  <div role="button" tabIndex={0} className="room-flip" onClick={handleFlip}>
    <div className="room-flip-front">...</div>
    <div className="room-flip-back">...</div>
  </div>
  <div className="room-flip-marks">
    <button onClick={handleMark}>찜</button>
    <button onClick={handleWatch}>봤어요</button>
  </div>
</li>
```

```tsx
// ❌ a 안에 button
<a href="/post/1">
  <button>공유</button>  {/* 안 됨 */}
</a>

// ✅ 구조 분리
<div className="post-item">
  <a href="/post/1">제목</a>
  <button>공유</button>
</div>
```

```txt
왜 클릭이 부모로 먹히는가:
  button 안의 button을 클릭하면
  브라우저가 HTML 규칙 위반으로 DOM을 재해석
  → 내부 button이 부모 밖으로 빠져나가거나 무시됨
  → 클릭 이벤트가 의도한 곳에 안 닿음

기준:
  카드·플립 컨테이너가 클릭 가능하면 → 그 안에 다른 클릭 가능한 요소 넣지 말 것
  분리할 수 없으면 e.stopPropagation() 고려
    (근본 해결은 아님 — HTML 구조 수정이 우선)
```

---

# 실전 패턴 모음

## 아이콘 버튼

```tsx
// ❌ 스크린 리더: "svg, 버튼" — 무슨 버튼인지 모름
<button onClick={onDelete}><TrashIcon /></button>

// ✅ 방법 1 — aria-label
<button onClick={onDelete} aria-label="게시글 삭제">
  <TrashIcon aria-hidden />
</button>

// ✅ 방법 2 — sr-only 텍스트
<button onClick={onDelete}>
  <TrashIcon aria-hidden />
  <span className="sr-only">게시글 삭제</span>
</button>
```

## 달력 상세 업데이트

```tsx
// 날짜 선택 시 영화 목록이 바뀌는 영역 — aria-live="polite"
<section
  className="movie-calendar-detail"
  aria-live="polite"
  aria-atomic="true"
>
  {selectedDate
    ? <MovieList date={selectedDate} movies={movies} />
    : <p>날짜를 선택해주세요</p>
  }
</section>
```

## 폼 에러 메시지

```tsx
<input
  id="email"
  type="email"
  aria-describedby={error ? 'email-error' : undefined}
  aria-invalid={!!error}
/>
{error && (
  <p id="email-error" role="alert" className="text-red-500">
    {error}
  </p>
)}
```

## 모달 (Portal 패턴과 함께)

```tsx
<div
  role="dialog"
  aria-modal="true"
  aria-labelledby="modal-title"
>
  <h2 id="modal-title">삭제 확인</h2>
  ...
</div>
```

## 드롭다운 열림/닫힘

```tsx
<button
  aria-expanded={isOpen}
  aria-controls="dropdown-menu"
  onClick={() => setIsOpen(!isOpen)}
>
  메뉴 <ChevronIcon />
</button>
<ul id="dropdown-menu" hidden={!isOpen}>
  ...
</ul>
```
