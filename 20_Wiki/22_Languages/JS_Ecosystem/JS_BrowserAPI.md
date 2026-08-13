---
aliases:
  - 브라우저 API
  - window
  - addEventListener
  - IntersectionObserver
tags:
  - JavaScript
related:
  - "[[00_JS_Ecosystem_HomePage]]"
  - "[[NextJS_Concept]]"
  - "[[JS_URL_Encoding]]"
  - "[[JS_DOM]]"
  - "[[React_useEffect]]"
---
# JS_BrowserAPI — 브라우저 API · window

> [!info] 
> `window` = 브라우저의 전역 객체. 
> 이벤트 리스너·타이머·뷰포트 등 브라우저가 제공하는 API의 진입점. 
> Next.js(SSR)에서는 서버에 `window`가 없으므로 `useEffect` 안에서만 사용.
>  URL·URLSearchParams → [[JS_URL_Encoding]], DOM 조작 → [[JS_DOM]]

---

# window란 ⭐️⭐️⭐️

```txt
브라우저에서 JavaScript를 실행하면 전역 스코프 = window 객체
window. 을 생략해도 동작:
  setTimeout(fn, 1000)   = window.setTimeout(fn, 1000)
  addEventListener(...)   = window.addEventListener(...)
  location.href           = window.location.href

window. 를 명시하면 "이건 브라우저 API"임이 코드에서 명확해짐
```

## SSR(Next.js)에서 window 주의 ⭐️⭐️⭐️⭐️

```typescript
// ❌ 서버에서 window가 없어서 에러
const w = window.innerWidth;

// ✅ useEffect 안에서만 (클라이언트에서만 실행됨)
useEffect(() => {
  const w = window.innerWidth;
}, []);

// ✅ 조건 체크
if (typeof window !== 'undefined') { ... }
```

---

# addEventListener — 이벤트 등록과 해제 ⭐️⭐️⭐️⭐️

```typescript
// 등록
window.addEventListener('keydown', handler);

// 해제 — 반드시 같은 함수 참조여야 함
window.removeEventListener('keydown', handler);
```

## cleanup 패턴 — useEffect와 함께 ⭐️⭐️⭐️⭐️

```typescript
useEffect(() => {
  if (!open) return;

  // ① 핸들러를 변수에 저장 (같은 참조 유지)
  const onKeyDown = (e: KeyboardEvent) => {
    if (e.key === 'Escape' && !isPending) onClose();
  };

  // ② 등록
  window.addEventListener('keydown', onKeyDown);

  // ③ cleanup — 재실행 전 또는 언마운트 시 해제
  return () => window.removeEventListener('keydown', onKeyDown);

}, [open, isPending, onClose]);
```

```txt
핸들러를 변수에 저장하는 이유:
  addEventListener와 removeEventListener에 정확히 같은 함수 참조 필요

  ❌ 해제 안 됨:
  window.addEventListener('keydown', (e) => handler(e));
  window.removeEventListener('keydown', (e) => handler(e));  // 다른 참조!

  ✅ 정상:
  const fn = (e: KeyboardEvent) => handler(e);
  window.addEventListener('keydown', fn);
  window.removeEventListener('keydown', fn);  // 같은 참조

removeEventListener를 안 하면:
  컴포넌트가 언마운트되어도 리스너가 남아있음
  → 메모리 누수 + 언마운트된 컴포넌트의 state 변경 시도 → 에러
```

## 자주 쓰는 window 이벤트

|이벤트|언제|
|---|---|
|`keydown` / `keyup`|키보드 입력 — Escape, Enter 처리|
|`resize`|브라우저 창 크기 변경|
|`scroll`|페이지 스크롤|
|`focus` / `blur`|탭 전환, 창 포커스 변경|
|`online` / `offline`|네트워크 상태 변화|
|`beforeunload`|페이지 이탈 전 (저장 확인 팝업)|

---

# setTimeout · clearTimeout ⭐️⭐️⭐️⭐️

```typescript
// 일정 시간 후 실행
const id = window.setTimeout(() => doSomething(), 1000);

// 취소 (실행 전이면 취소됨)
window.clearTimeout(id);

// useEffect cleanup에서 취소 패턴
useEffect(() => {
  const id = window.setTimeout(() => setVisible(false), 3000);
  return () => window.clearTimeout(id);  // 언마운트 시 취소
}, []);
```

## setTimeout(fn, 0) — "한 틱 뒤 실행" ⭐️⭐️⭐️⭐️

```typescript
window.setTimeout(() => setLoginOpen(true), 0);
```

```txt
0ms지만 즉시 실행이 아님:
  JavaScript는 단일 스레드
  현재 실행 중인 코드가 끝난 뒤 → 다음 이벤트 루프에서 실행

실전 사용 이유:
  버튼 클릭 → 로그인 모달 열기인데
  그 클릭 이벤트가 모달 배경 클릭 닫기까지 전달될 수 있음
  → setTimeout 0으로 한 틱 뒤에 모달 오픈
  → 현재 클릭 이벤트 처리가 완전히 끝난 뒤에 열림

  if (!user) {
    window.setTimeout(() => setLoginOpen(true), 0);
    return;
  }
```

---

# window.visualViewport — 모바일 뷰포트 ⭐️⭐️⭐️

```typescript
window.visualViewport?.width     // 실제 보이는 가로
window.visualViewport?.height    // 실제 보이는 세로 (모바일 키보드 제외)
window.visualViewport?.offsetTop // 스크롤 오프셋
```

```typescript
// 팝업·드롭다운 위치 계산 — resize/scroll 둘 다 구독
useLayoutEffect(() => {
  if (!open) { setPos(null); return; }

  updatePos();

  window.addEventListener('resize', updatePos);
  window.visualViewport?.addEventListener('resize', updatePos);
  window.visualViewport?.addEventListener('scroll', updatePos);

  return () => {
    window.removeEventListener('resize', updatePos);
    window.visualViewport?.removeEventListener('resize', updatePos);
    window.visualViewport?.removeEventListener('scroll', updatePos);
  };
}, [open, updatePos]);
```

```txt
window.resize vs window.visualViewport:
  window resize        → 브라우저 창 크기 변경
  visualViewport resize → 모바일 소프트 키보드가 올라와서 뷰포트가 줄어들 때도 발생
  → 팝업처럼 화면에 정확히 붙는 UI는 둘 다 구독 필요

?.  옵셔널 체이닝:
  구형 브라우저에서 visualViewport 미지원 → 없으면 건너뜀

useLayoutEffect를 쓰는 이유:
  DOM 측정 후 위치 계산 → 화면에 그려지기 전에 완료해야 깜빡임 없음
  useEffect는 그린 뒤 실행 → 위치가 잘못된 상태로 잠깐 보임
```

---
# IntersectionObserver — 요소가 화면에 보이는지 감지 ⭐️⭐️⭐️⭐️

```txt
IntersectionObserver = "이 요소가 뷰포트(또는 특정 컨테이너)에 보이는지"를 감지하는 API

scroll 이벤트 대신 쓰는 이유:
  scroll → 스크롤마다 getBoundingClientRect() 호출 → 비쌈
  IntersectionObserver → 브라우저가 알아서 감지 → 성능 좋음

주요 사용처:
  무한 스크롤 (load more trigger)
  이미지 지연 로딩 (화면에 보일 때 src 설정)
  애니메이션 진입 효과 (보이기 시작할 때 클래스 추가)
```

```typescript
// 기본 구조
const observer = new IntersectionObserver(
  (entries) => {
    // entries = 감시 중인 요소들의 교차 상태 배열
    entries.forEach((entry) => {
      if (entry.isIntersecting) {
        // 요소가 뷰포트에 진입함
        doSomething();
      }
    });
  },
  {
    root:       null,    // null = 브라우저 뷰포트 기준 (기본값)
    rootMargin: '0px',   // root 경계 확장 — 미리 감지하려면 '120px'
    threshold:  0,       // 0 = 1px이라도 보이면 감지, 1 = 완전히 보일 때
  },
);

observer.observe(element);   // 감시 시작
observer.unobserve(element); // 특정 요소 감시 해제
observer.disconnect();       // 전체 감시 해제 (cleanup)
```

```typescript
// 옵션 설명
{
  root: scrollRef.current,  // 특정 스크롤 컨테이너 기준 (null이면 뷰포트)
  rootMargin: '120px',      // 아직 뷰포트 밖 120px에 있을 때부터 감지
  //  → 화면에 닿기 전 미리 loadMore() 호출 (끊김 없는 무한 스크롤)
  threshold: 0.5,           // 50% 이상 보일 때 감지
}
```

## 무한 스크롤 — useEffect 패턴

```typescript
// 리스트 맨 아래에 빈 div를 두고 그게 보이면 loadMore() 호출
const loadMoreTriggerRef = useRef<HTMLDivElement>(null);
const scrollRef          = useRef<HTMLDivElement>(null);

useEffect(() => {
  const node = loadMoreTriggerRef.current;
  const root = scrollRef.current;
  if (!node || !root || !hasMore) return;
  // hasMore가 false면 더 불러올 것 없음 → observer 등록 안 함

  const observer = new IntersectionObserver(
    (entries) => {
      if (entries[0]?.isIntersecting) void loadMore();
      //                ↑ 트리거 div가 뷰포트에 보임 → loadMore 호출
    },
    { root, rootMargin: '120px' },
  );

  observer.observe(node);
  return () => observer.disconnect();  // 클린업: 언마운트 or deps 변경 시
}, [hasMore, loadMore, items.length]);
//           ↑ loadMore가 바뀌거나, 아이템이 추가될 때마다 observer 재등록
```

```tsx
// JSX
<div ref={scrollRef} style={{ overflow: 'auto', height: '500px' }}>
  {items.map(item => <Item key={item.id} {...item} />)}
  <div ref={loadMoreTriggerRef} />  {/* 트리거 — 화면에 보이면 loadMore */}
</div>
```

```txt
왜 items.length를 deps에 넣는가:
  아이템이 추가되면 트리거 div 위치가 바뀜
  observer를 재등록해야 새 위치에서 감지

rootMargin: '120px':
  트리거 div가 아직 화면 밖 120px에 있을 때부터 isIntersecting = true
  → 사용자가 맨 아래에 닿기 전에 미리 로딩 시작
  → 끊김 없는 무한 스크롤

void loadMore():
  loadMore()가 async함수인 경우 Promise 반환
  void = "이 Promise 무시" (useEffect 콜백은 Promise를 반환하면 안 됨)
```
---

# 자주 쓰는 패턴

```typescript
// ESC 키로 닫기
useEffect(() => {
  if (!open) return;
  const fn = (e: KeyboardEvent) => {
    if (e.key === 'Escape') onClose();
  };
  window.addEventListener('keydown', fn);
  return () => window.removeEventListener('keydown', fn);
}, [open, onClose]);

// 모달 열릴 때 스크롤 잠금
useEffect(() => {
  document.body.style.overflow = open ? 'hidden' : '';
  return () => { document.body.style.overflow = ''; };
}, [open]);

// 디바운스 — 타이핑이 멈춘 뒤에 검색
const timerRef = useRef<ReturnType<typeof setTimeout> | null>(null);
const handleChange = (value: string) => {
  if (timerRef.current) window.clearTimeout(timerRef.current);
  timerRef.current = window.setTimeout(() => search(value), 300);
};

// 한 틱 뒤 실행 (이벤트 전파 문제 우회)
window.setTimeout(() => doSomething(), 0);
```