---
aliases:
  - 접근성
  - 스크린 리더
  - ARIA
  - aria-label
  - aria-labelledby
  - sr-only
  - role
  - tabIndex
  - 인터랙티브 요소 중첩 금지
tags:
  - HTML
  - React
related:
  - "[[00_JS_Ecosystem_HomePage]]"
---

# HTML_ARIA — 접근성 · ARIA

> [!info] 
> ARIA(Accessible Rich Internet Applications) = 스크린 리더 같은 보조 기술에게 HTML의 의미를 전달하는 속성.
>  `aria-hidden`으로 장식용 요소를 숨기고, `aria-label`로 시각적 텍스트 없는 요소에 이름을 붙이고, `role`로 의미를 명시한다.

---

# 접근성이란 — 왜 필요한가 ⭐️⭐️⭐️⭐️

```txt
웹을 사용하는 모든 사람 중에는:
  시각 장애인  → 스크린 리더로 내용을 듣는다
  키보드만 사용 → 마우스 없이 탭 키로 이동
  저시력     → 고대비 모드, 화면 확대

스크린 리더:
  화면의 내용을 소리로 읽어주는 프로그램
  HTML 구조와 ARIA 속성을 보고 "지금 어디에 있는지, 뭘 할 수 있는지" 읽어줌

ARIA:
  HTML이 의미를 충분히 전달 못할 때 추가 정보를 제공
  시각적으로는 보이지 않지만 스크린 리더에게 전달됨
```

---

# aria-hidden — 스크린 리더에서 숨기기 ⭐️⭐️⭐️⭐️

```tsx
// 읽을 필요 없는 장식용 요소에 사용
<span
  className="size-1.5 rounded-full bg-brand-primary"
  aria-hidden         // = aria-hidden="true"
/>

// 아이콘 — 텍스트로 이미 설명되어 있을 때
<button>
  <svg aria-hidden>...</svg>   {/* 아이콘은 숨기고 */}
  알림                          {/* 텍스트만 읽힘 */}
</button>
```

```txt
aria-hidden이 필요한 경우:
  장식용 점(·), 구분선(|), 아이콘처럼 "시각적 의미는 있지만 읽어줄 필요 없는 것"
  이미 같은 내용이 텍스트로 있을 때 중복 읽기 방지

  스크린 리더는 위 span을 만나면:
  aria-hidden 없음 → "dot" 또는 빈 내용을 읽으려 시도
  aria-hidden="true" → 완전히 무시

주의: aria-hidden="true"인 요소 안에 포커스 가능한 요소가 있으면 안 됨
  숨겨진 요소 안의 버튼은 키보드로는 갈 수 없지만 스크린 리더가 혼란
```

---

# aria-label — 요소에 이름 붙이기 ⭐️⭐️⭐️⭐️

```tsx
// 텍스트 없는 버튼 — 아이콘만 있어서 무슨 버튼인지 모름
<button aria-label="알림 설정">
  <BellIcon />   {/* 텍스트 없음 */}
</button>
// 스크린 리더: "알림 설정, 버튼"

// 닫기 버튼
<button aria-label="닫기" onClick={onClose}>
  ✕
</button>
// 스크린 리더: "닫기, 버튼"  (✕ 기호 대신)

// 상태에 따라 동적으로 변경
<button aria-label={notificationsUnread ? '알림 (안 읽음)' : '알림'}>
  <BellIcon />
</button>
// 읽지 않은 알림 있을 때: "알림 (안 읽음), 버튼"
// 전부 읽었을 때:         "알림, 버튼"

// input — label 없을 때
<input
  type="search"
  aria-label="게시글 검색"
  placeholder="검색어를 입력하세요"
/>
```

```txt
동적 aria-label:
  aria-label={조건 ? '상태 A' : '상태 B'}
  상태가 바뀌면 스크린 리더가 새 label을 읽음

  버튼의 시각적 모습이 같아도 상태가 다르면 aria-label로 구분:
    기본:    aria-label="알림"
    안읽음:  aria-label="알림 (안 읽음)"
  → 시각 장애인도 알림이 있는지 파악 가능
```

```txt
언제 aria-label이 필요한가:
  버튼이나 링크에 텍스트가 없을 때 (아이콘만 있을 때)
  같은 텍스트가 여러 개일 때 구분 ("더 보기" 버튼이 10개면 → "홍길동의 게시글 더 보기")
  input에 연결된 <label>이 없을 때

  <label>이 있으면 aria-label 불필요:
  <label htmlFor="search">검색</label>
  <input id="search" type="text" />
```

# aria-labelledby — 다른 요소로 이름 연결 ⭐️⭐️⭐️

```tsx
// 이미 있는 텍스트를 label로 활용
<h2 id="dialog-title">삭제하시겠습니까?</h2>
<div role="dialog" aria-labelledby="dialog-title">
  ...
</div>
// 스크린 리더: "삭제하시겠습니까?, 대화상자"
```

```txt
aria-label vs aria-labelledby:
  aria-label        → 직접 텍스트 입력
  aria-labelledby   → 화면에 이미 있는 텍스트 요소의 id를 참조
  
  화면에 이미 제목이 있으면 aria-labelledby로 연결
  없으면 aria-label로 직접 제공
```

---

# role — 의미 명시 ⭐️⭐️⭐️

```tsx
// 기본 HTML 태그로 의미 전달이 어려울 때
<div role="dialog" aria-modal="true" aria-labelledby="title">
  모달 내용
</div>

<div role="alert">
  {/* 자동으로 읽힘 — 에러 메시지, 알림 */}
  저장에 실패했습니다.
</div>

<span role="status">
  {/* 작업 완료 상태 메시지 */}
  저장됐습니다.
</span>
```

## 자주 쓰는 role 값

|role|의미|
|---|---|
|`dialog`|모달 다이얼로그|
|`alert`|즉시 읽어주는 에러/경고|
|`status`|작업 완료 상태 (덜 급함)|
|`button`|div/span을 버튼처럼|
|`navigation`|nav 태그와 동일|
|`main`|main 태그와 동일|
|`list`·`listitem`|ul/li 대체|

```txt
가능하면 role 대신 의미 있는 HTML 태그 사용:
  <div role="button"> → <button>
  <div role="navigation"> → <nav>
  <div role="main"> → <main>
  
  HTML 태그 자체가 role을 내포하고 있음
  div/span에 role 붙이는 건 정말 어쩔 수 없을 때만
```

---

# 자주 쓰는 패턴 ⭐️⭐️⭐️⭐️

## 아이콘 버튼

```tsx
// ❌ 스크린 리더: "svg, 버튼" — 무슨 버튼인지 모름
<button onClick={onDelete}>
  <TrashIcon />
</button>

// ✅ 방법 1 — aria-label
<button onClick={onDelete} aria-label="게시글 삭제">
  <TrashIcon aria-hidden />
</button>

// ✅ 방법 2 — sr-only 텍스트
<button onClick={onDelete}>
  <TrashIcon aria-hidden />
  <span className="sr-only">게시글 삭제</span>  {/* 시각적으로 숨기되 스크린 리더는 읽음 */}
</button>
```

## 알림 뱃지 · 상태 점

```tsx
// 읽지 않은 알림 표시 — 점은 장식, 숫자로 전달
<button aria-label={`알림 ${unreadCount}개`}>
  <BellIcon aria-hidden />
  {unreadCount > 0 && (
    <span aria-hidden className="badge">{unreadCount}</span>
    // 숫자 뱃지도 aria-hidden — aria-label에서 이미 전달했으므로 중복 방지
  )}
</button>

// 읽음 여부 표시 점 — 순수 장식
<span
  className={`size-1.5 rounded-full ${n.readAt ? 'bg-transparent' : 'bg-brand-primary'}`}
  aria-hidden   // 점은 시각적 표현만, 읽음 여부는 다른 방법으로 전달
/>
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
  aria-labelledby="modal-title"    // 모달 제목과 연결
>
  <h2 id="modal-title">삭제 확인</h2>
  ...
</div>
```

---

# sr-only — 시각적으로 숨기되 스크린 리더는 읽음

```tsx
// Tailwind CSS
<span className="sr-only">현재 페이지: 홈</span>

// sr-only의 CSS (시각적으로 완전히 숨김)
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
```

```txt
display: none vs aria-hidden vs sr-only:
  display: none    → 시각적으로 숨김 + 스크린 리더도 못 읽음
  aria-hidden      → 시각적으로 보임 + 스크린 리더는 못 읽음
  sr-only(class)   → 시각적으로 숨김 + 스크린 리더만 읽음
```

---

# 키보드 접근성 ⭐️⭐️⭐️

```tsx
// tabIndex — 키보드 탭 순서 제어
<div tabIndex={0}>   {/* 탭으로 포커스 가능하게 */}
<div tabIndex={-1}>  {/* 탭으로는 못 가지만 JS로 focus() 가능 */}

// div를 버튼처럼 쓸 때 — 키보드도 지원해야 함
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
// → 그냥 <button>을 쓰는 게 훨씬 낫고, 이 모든 게 기본 지원됨
```

----
# 인터랙티브 요소 중첩 금지 ⭐️⭐️⭐️⭐️

```txt
HTML 규칙: button 안에 button 올 수 없음
interactive 요소(button, a, input, select, textarea) 안에
또 다른 interactive 요소를 넣으면 안 됨

증상:
  클릭이 안 먹거나 부모 이벤트에 먹힘
  카드만 뒤집히고 내부 버튼 토글이 안 됨
  button cannot be a descendant of button (hydration 경고)
```

## 플립 카드 + 마크 버튼 패턴

```tsx
// ❌ 플립 컨테이너(button/role="button") 안에 button 중첩
<button className="room-flip" onClick={handleFlip}>
  {/* 앞면 */}
  <div className="room-flip-front">...</div>
  {/* 뒷면 */}
  <div className="room-flip-back">...</div>
  {/* ❌ 여기 button 넣으면 안 됨 */}
  <button onClick={handleMark}>찜</button>
  <button onClick={handleWatch}>봤어요</button>
</button>
```

```tsx
// ✅ 마크 버튼을 플립 밖으로 분리
<li className="room-movie">
  {/* 플립 컨테이너 — 앞/뒤 면만 */}
  <div role="button" tabIndex={0} className="room-flip" onClick={handleFlip}>
    <div className="room-flip-front">...</div>
    <div className="room-flip-back">...</div>
  </div>

  {/* 마크 버튼 — 플립 밖 */}
  <div className="room-flip-marks">
    <button onClick={handleMark}>찜</button>
    <button onClick={handleWatch}>봤어요</button>
  </div>
</li>
```

```txt
왜 클릭이 부모로 먹히는가:
  button 안의 button을 클릭하면
  브라우저가 HTML 규칙 위반으로 DOM을 재해석
  → 내부 button이 부모 밖으로 빠져나가거나 무시됨
  → 클릭 이벤트가 의도한 곳에 안 닿음

가챠 결과 모달도 동일:
  모달 안에 플립 카드가 있고 마크 버튼이 필요하면
  → 마크는 모달 actions 영역(플립 밖)에 배치

기준:
  플립 컨테이너(또는 카드)가 클릭 가능하면
  → 그 안에 다른 클릭 가능한 요소 넣지 말 것
  → 분리할 수 없으면 e.stopPropagation() 고려
    (근본 해결은 아님 — HTML 구조 수정이 우선)
```

## 자주 만나는 중첩 금지 케이스

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

// ❌ a 안에 a
<a href="/user/1">
  <a href="/post/1">게시글</a>  {/* 안 됨 */}
</a>

// ✅ e.stopPropagation (임시 방편)
<a href="/user/1">
  <a href="/post/1" onClick={e => e.stopPropagation()}>게시글</a>
</a>
// → 구조 개선이 먼저
```