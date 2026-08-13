---
aliases: [마운트(mount), createPortal, Hydration, Modal, mounted pattern, Portal]
tags: [React, NextJS]
related:
  - "[[00_JS_Ecosystem_HomePage]]"
  - "[[JS_BrowserAPI]]"
  - "[[NextJS_AuthState]]"
  - "[[NextJS_ServerClient]]"
  - "[[React_useId]]"
---
# React_Portal_Dialog — Portal · 모달

>[!info]
>Portal = React 컴포넌트를 부모 DOM 트리 밖에 렌더링하는 방법. 
>`createPortal(children, container)`으로 React 트리는 원래 위치에 두면서 DOM만 `document.body`에 붙인다.
> `z-index`·`overflow: hidden` 탈출 문제를 해결한다. 
> SSR(Next.js)에서는 `document`가 없어서 mounted 가드 필요.

---

# Portal이란 — React 트리 vs DOM 트리 ⭐️⭐️⭐️⭐️

```txt
React 트리:
  부모 컴포넌트 → 자식 컴포넌트 계층
  Context, 이벤트 버블링이 이 트리를 따름

DOM 트리:
  실제 브라우저가 화면을 그리는 HTML 구조
  z-index, overflow, stacking context가 이 트리를 따름

문제:
  React 트리 깊은 곳에 Modal이 있으면
  부모가 overflow: hidden 이거나 z-index가 낮으면
  Modal이 잘리거나 다른 요소에 가려짐

Portal:
  React 트리는 그대로 (Context·이벤트 유지)
  DOM만 document.body에 직접 붙임
  → z-index·overflow 부모 영향 탈출

  "React 입장에서는 자식이지만
   DOM 입장에서는 body 자식"
```

---

# 왜 필요한가 ⭐️⭐️⭐️⭐️

```txt
문제 1 — overflow: hidden:
  <div style="overflow: hidden">
    <Modal />  ← 이 Modal이 div 밖으로 못 나감 (잘림)
  </div>

문제 2 — z-index 격리:
  <section style="z-index: 1; position: relative">
    <Modal style="z-index: 9999" />
    ← 부모의 stacking context에 갇혀서
      z-index 9999라도 다른 section 뒤에 가릴 수 있음
  </section>

Portal 해결:
  Modal이 DOM에서 document.body의 자식으로 붙음
  → 어떤 부모의 overflow/z-index에도 영향받지 않음
  → fixed + z-index 9999로 화면 최상단에 안전하게 표시
```

---

# createPortal 기본 사용법 ⭐️⭐️⭐️⭐️

```typescript
import { createPortal } from 'react-dom';

function Modal({ children }: { children: React.ReactNode }) {
  return createPortal(
    <div className="modal-overlay">
      <div className="modal-content">
        {children}
      </div>
    </div>,
    document.body,  // DOM 붙을 위치 — 보통 document.body
  );
}
```

```txt
createPortal(children, container):
  children  = 렌더링할 React 요소
  container = DOM에서 붙을 위치 (document.body, 특정 div 등)

  React 트리: Modal은 원래 부모 컴포넌트 안에 있음
              → Context, 이벤트 버블링은 React 트리를 따름
  DOM 트리:   Modal의 HTML은 container(document.body) 안에 붙음
              → z-index, overflow는 DOM 트리를 따름
```

---

# mounted 가드 — SSR(Next.js) 주의 ⭐️⭐️⭐️⭐️

```typescript
// ❌ SSR에서 에러 — 서버에는 document가 없음
function Modal() {
  return createPortal(<div>모달</div>, document.body);
  // ReferenceError: document is not defined
}

// ✅ mounted 가드 — 클라이언트에서만 portal 렌더링
'use client';
import { useState, useEffect } from 'react';
import { createPortal }        from 'react-dom';

function Modal({ children }: { children: React.ReactNode }) {
  const [mounted, setMounted] = useState(false);

  useEffect(() => {
    setMounted(true);   // 클라이언트에서 마운트됐을 때만 true
  }, []);

  if (!mounted) return null;  // 서버·마운트 전 → 렌더링 안 함

  return createPortal(
    <div>{children}</div>,
    document.body,
  );
}
```

```txt
왜 useEffect 안에서 setMounted(true)인가:
  useEffect는 클라이언트(브라우저)에서만 실행됨
  서버 렌더링 시 useEffect 실행 안 됨 → mounted = false → null 반환
  브라우저에서 마운트 완료 → mounted = true → portal 렌더링

  typeof window !== 'undefined' 체크로 대체 가능하지만
  useState + useEffect 패턴이 hydration 불일치를 더 안전하게 처리
```

---

# 실전 Modal 패턴 ⭐️⭐️⭐️⭐️

```typescript
// components/Modal.tsx
'use client';
import { useEffect, useState } from 'react';
import { createPortal }        from 'react-dom';

type ModalProps = {
  open:     boolean;
  onClose:  () => void;
  children: React.ReactNode;
};

export function Modal({ open, onClose, children }: ModalProps) {
  const [mounted, setMounted] = useState(false);
  useEffect(() => { setMounted(true); }, []);

  // ESC 키로 닫기
  useEffect(() => {
    if (!open) return;
    const handleKey = (e: KeyboardEvent) => {
      if (e.key === 'Escape') onClose();
    };
    document.addEventListener('keydown', handleKey);
    return () => document.removeEventListener('keydown', handleKey);
  }, [open, onClose]);

  // 스크롤 잠금
  useEffect(() => {
    if (!open) return;
    const prev = document.body.style.overflow;  // 기존 값 저장
    document.body.style.overflow = 'hidden';
    return () => {
      document.body.style.overflow = prev;       // 닫을 때 원래 값으로 복원
    };
  }, [open]);

  // prev를 저장하는 이유:
  // 모달이 열리기 전에 body.overflow가 이미 다른 값이었을 수 있음
  // 예: 다른 오버레이가 먼저 열려있어서 'hidden'이었다가 이 모달을 닫으면
  //     '' 로 되돌리면 → 첫 번째 오버레이의 스크롤 잠금도 풀려버림
  // prev 복원 = "내가 바꾸기 전 상태로만 돌아가기"

  if (!mounted || !open) return null;

  return createPortal(
    <div
      className="fixed inset-0 z-[9999] flex items-center justify-center"
      onClick={onClose}              // 배경 클릭 → 닫기
    >
      <div
        className="bg-white rounded-xl p-6 shadow-2xl max-w-md w-full"
        onClick={(e) => e.stopPropagation()}  // 모달 내부 클릭 → 닫기 방지
      >
        {children}
      </div>
    </div>,
    document.body,
  );
}

// 사용
function Page() {
  const [open, setOpen] = useState(false);

  return (
    <>
      <button onClick={() => setOpen(true)}>모달 열기</button>
      <Modal open={open} onClose={() => setOpen(false)}>
        <h2>모달 제목</h2>
        <p>내용</p>
        <button onClick={() => setOpen(false)}>닫기</button>
      </Modal>
    </>
  );
}
```

---

# 이벤트 버블링 — Portal의 특이한 동작 ⭐️⭐️⭐️

```tsx
function Parent() {
  return (
    <div onClick={() => console.log('부모 클릭')}>
      <Modal>
        <button onClick={() => console.log('버튼 클릭')}>클릭</button>
      </Modal>
    </div>
  );
}
```

```txt
Portal의 이벤트 버블링:
  DOM 트리: button → document.body (Portal container)
  React 트리: button → Modal → div(부모)

  React 이벤트는 DOM이 아닌 React 트리를 따라 버블링
  → 버튼 클릭 시 "버튼 클릭" + "부모 클릭" 둘 다 출력

  이게 의도치 않으면:
  모달 배경 클릭 닫기 패턴에서 e.stopPropagation() 필요
  → 위 Modal 예제의 stopPropagation 참고
```

---

# Portal 사용 기준

```txt
✅ Portal을 써야 할 때:
  모달·다이얼로그 (화면 전체를 덮어야 함)
  툴팁·팝오버 (부모의 overflow: hidden을 탈출해야 함)
  토스트 알림 (화면 고정 위치)
  드롭다운 (스크롤 컨테이너 밖에 표시)

❌ Portal이 필요 없을 때:
  부모의 overflow나 z-index 영향이 없는 경우
  단순 조건부 렌더링으로 충분한 경우
  → 불필요하게 복잡도만 높아짐

container 선택:
  document.body       → 기본, 대부분의 경우
  특정 div#portal     → 여러 Portal을 한 컨테이너에 모을 때
  scrollRef.current   → 특정 스크롤 영역 안에서만 portal할 때
```