---
aliases: [마운트(mount), createPortal, Hydration, Modal, mounted pattern, Portal]
tags: [React]
related:
  - "[[00_JS_Ecosystem_HomePage]]"
  - "[[JS_BrowserAPI]]"
  - "[[NextJS_AuthCache]]"
  - "[[NextJS_ServerClient]]"
  - "[[React_useId]]"
  - "[[React_useMemo_useCallback_useEffect]]"
---
# React_Portal — createPortal & Dialog 패턴

> [!info] 
> `createPortal` = 컴포넌트를 DOM 트리의 다른 위치에 렌더링하는 방법.
>  Dialog·모달·툴팁처럼 부모의 `overflow:hidden`이나 `z-index`를 탈출해야 할 때 사용한다.

---

# createPortal이란 ⭐️⭐️⭐️⭐️

```tsx
import { createPortal } from 'react-dom';

function Modal({ children }: { children: React.ReactNode }) {
  return createPortal(
    children,        // 렌더링할 JSX
    document.body,   // 어디에 붙일지 (DOM 노드)
  );
}
```

```txt
일반 렌더링:
  <App>
    <Header />
    <Modal />   ← Modal은 App 안에 갇혀있음
  </App>

createPortal 사용:
  <App>
    <Header />
    (Portal → document.body 아래에 렌더링)
  </App>
  <div class="modal">...</div>  ← App 바깥, body 직속 자식

React 이벤트 버블링은 Portal 위치가 아니라 컴포넌트 트리 기준으로 동작
→ Portal 안의 이벤트가 컴포넌트 상위로 버블링됨 (DOM 위치와 무관)
```

## 왜 필요한가

```txt
문제 상황:
  부모 컴포넌트에 overflow:hidden 이 있으면 Modal이 잘림
  부모 컴포넌트의 z-index 컨텍스트에 갇혀서 다른 요소 뒤로 숨어버림
  position:fixed 가 부모의 transform/filter에 영향받음

해결:
  createPortal로 document.body 직속 자식으로 렌더링
  → 부모의 CSS 영향에서 완전히 탈출
  → z-index, overflow, position 모두 독립적으로 동작
```

---

# mounted 패턴 — SSR 안전하게 ⭐️⭐️⭐️⭐️

```tsx
const [mounted, setMounted] = useState(false);

useEffect(() => {
  setMounted(true);
}, []);

if (!mounted) return null;

return createPortal(<Dialog />, document.body);
```

```txt
왜 mounted 체크가 필요한가:
  createPortal(children, document.body)에서 document.body는 브라우저 전용
  SSR(서버사이드 렌더링) 중에는 document가 없음 → 에러

  mounted = false  → 서버 / 첫 렌더 (document 없음) → null 반환
  mounted = true   → useEffect 실행 = 브라우저에서 hydration 완료 → Portal 렌더

Next.js App Router나 SSR 환경에서 createPortal을 쓰면 항상 이 패턴 필요
```

---

# Dialog 전체 구현 ⭐️⭐️⭐️⭐️

```tsx
'use client';

import { useEffect, useState } from 'react';
import { createPortal } from 'react-dom';

type DialogProps = {
  open:         boolean;
  onClose:      () => void;
  onConfirm:    () => void;
  title?:       string;
  description?: string;
  confirmLabel?: string;
  /** @deprecated 오타 — confirmLabel 사용 */
  comfirmLabel?: string;
  pendingLabel?: string;
  cancelLabel?:  string;
  isPending?:    boolean;
};

export function Dialog({
  open,
  onClose,
  onConfirm,
  title         = '삭제하시겠습니까?',
  description   = '삭제하면 되돌릴 수 없어요.',
  confirmLabel,
  comfirmLabel,
  pendingLabel,
  cancelLabel   = '취소',
  isPending     = false,
}: DialogProps) {
  // ① SSR 안전
  const [mounted, setMounted] = useState(false);
  useEffect(() => { setMounted(true); }, []);

  // ② 레이블 우선순위 (deprecate 처리 포함)
  const actionLabel        = confirmLabel ?? comfirmLabel ?? '삭제';
  const actionPendingLabel = pendingLabel ?? `${actionLabel} 중…`;

  // ③ Escape 키로 닫기
  useEffect(() => {
    if (!open) return;
    function onKeyDown(e: KeyboardEvent) {
      if (e.key === 'Escape' && !isPending) onClose();
    }
    window.addEventListener('keydown', onKeyDown);
    return () => window.removeEventListener('keydown', onKeyDown);
  }, [open, isPending, onClose]);

  if (!open || !mounted) return null;

  return createPortal(
    // ④ 배경(overlay) — 클릭하면 닫기
    <div
      className="fixed inset-0 z-[100] flex items-center justify-center bg-black/40 p-4"
      role="dialog"
      aria-modal="true"
      aria-labelledby="dialog-title"
      onClick={() => { if (!isPending) onClose(); }}
    >
      {/* ⑤ 패널 — 클릭 이벤트 버블링 차단 */}
      <div
        className="relative w-full max-w-sm"
        onClick={(e) => e.stopPropagation()}
      >
        <h2 id="dialog-title" className="text-lg font-semibold">
          {title}
        </h2>
        <p className="mt-2 text-sm">{description}</p>
        <div className="mt-6 flex flex-col gap-2">
          <button onClick={onConfirm} disabled={isPending}>
            {isPending ? actionPendingLabel : actionLabel}
          </button>
          <button onClick={onClose} disabled={isPending}>
            {cancelLabel}
          </button>
        </div>
      </div>
    </div>,
    document.body,
  );
}
```

---

# 패턴별 상세 설명

## ① 배경 클릭 닫기 + 패널 클릭 차단 ⭐️⭐️⭐️⭐️

```tsx
{/* 배경 — 클릭하면 닫힘 */}
<div onClick={() => { if (!isPending) onClose(); }}>

  {/* 패널 — 클릭이 배경까지 버블링되지 않도록 차단 */}
  <div onClick={(e) => e.stopPropagation()}>
    ...내용...
  </div>

</div>
```

```txt
왜 stopPropagation이 필요한가:
  패널 안을 클릭하면 클릭 이벤트가 버블링으로 배경까지 전파됨
  → 배경의 onClick이 실행돼서 모달이 닫혀버림
  → 패널에서 stopPropagation()으로 차단해야 패널 클릭이 배경에 안 닿음

isPending 체크:
  API 요청 중에 배경 클릭으로 닫히면 요청이 도중에 끊길 수 있음
  → isPending일 때는 배경 클릭으로 닫기 차단
```

## ② Escape 키 처리 ⭐️⭐️⭐️

```tsx
useEffect(() => {
  if (!open) return;  // 열려있을 때만 리스너 등록

  function onKeyDown(e: KeyboardEvent) {
    if (e.key === 'Escape' && !isPending) onClose();
  }

  window.addEventListener('keydown', onKeyDown);
  return () => window.removeEventListener('keydown', onKeyDown);
}, [open, isPending, onClose]);
```

```txt
open이 deps에 있는 이유:
  닫혀있을 때 이벤트 리스너를 등록할 필요 없음
  open이 false면 early return

isPending이 deps에 있는 이유:
  Escape를 눌러도 isPending 중에는 닫히면 안 됨
  isPending이 바뀌면 effect를 재실행해서 최신 isPending 값으로 핸들러 갱신

cleanup 필수:
  return () => removeEventListener(...)
  컴포넌트가 사라지거나 open이 false가 되면 리스너 제거
```

## ③ @deprecated JSDoc 패턴 ⭐️⭐️

```tsx
type DialogProps = {
  confirmLabel?: string;
  /** @deprecated 오타 — confirmLabel 사용 */
  comfirmLabel?: string;
};

// 우선순위 체인으로 두 props 모두 허용 (하위 호환)
const actionLabel = confirmLabel ?? comfirmLabel ?? '삭제';
```

```txt
@deprecated를 쓰는 상황:
  오타 수정, 이름 변경 등으로 기존 prop 이름을 바꿔야 하는데
  이미 여러 곳에서 사용 중이라 한 번에 바꾸기 어려울 때

  → @deprecated 달아두면 VSCode에서 취소선으로 표시
  → 기존 코드는 그대로 동작하면서 새 코드는 새 이름 사용

  동작 방식: confirmLabel ?? comfirmLabel ?? '삭제'
  → confirmLabel 있으면 사용 / 없으면 comfirmLabel(구버전) / 없으면 기본값
```

## ④ 접근성(aria) 속성 ⭐️⭐️⭐️

```tsx
<div
  role="dialog"           // 스크린리더에 "이건 다이얼로그야"
  aria-modal="true"       // 배경은 비활성 상태임을 알림
  aria-labelledby="dialog-title"  // 제목 요소와 연결
>
  <h2 id="dialog-title">...</h2>  // aria-labelledby가 참조하는 id
```

|속성|역할|
|---|---|
|`role="dialog"`|스크린리더가 다이얼로그로 인식|
|`aria-modal="true"`|배경 콘텐츠는 접근 불가 상태임을 선언|
|`aria-labelledby="id"`|해당 id의 요소가 다이얼로그 제목임을 연결|
|`aria-describedby="id"`|설명 텍스트 연결 (선택)|

---
# 실전 패턴 — 목록에서 특정 항목 삭제 ⭐️⭐️⭐️⭐️

```tsx
// ❌ 상태 두 개 — open 따로, 대상 id 따로
const [open, setOpen]               = useState(false);
const [targetId, setTargetId]       = useState<string | null>(null);

// ✅ id | null 하나로 — open 여부 + 대상 id를 동시에 표현
const [removeTargetId, setRemoveTargetId] = useState<string | null>(null);
// null    → Dialog 닫힘
// 'abc'   → Dialog 열림 + 삭제 대상은 'abc'
```

## 전체 흐름

```typescript
// 상태
const [removeTargetId, setRemoveTargetId] = useState<string | null>(null);
const [removing, setRemoving]             = useState(false);
const [error, setError]                   = useState('');

// ① 삭제 버튼 클릭 → Dialog 열기만 (API 호출 ❌)
function handleRemove(otherId: string) {
  setRemoveTargetId(otherId);  // id를 저장 = Dialog 열림
}

// ② 확인 버튼 → 실제 API 호출
async function confirmRemove() {
  if (!removeTargetId) return;  // 안전 가드

  setRemoving(true);
  setError('');
  try {
    await removeFriend(removeTargetId);
    setRemoveTargetId(null);   // 성공 → Dialog 닫기
    await load();              // 목록 새로고침
  } catch (err) {
    setError(err instanceof Error ? err.message : '친구 삭제에 실패했습니다.');
  } finally {
    setRemoving(false);
  }
}
```

```tsx
{/* ③ Dialog 렌더링 */}
<FeedDialog
  open={removeTargetId !== null}  // id가 있으면 열림
  title="친구를 삭제할까요?"
  description="삭제하면 다시 친구 요청을 보내야 해요."
  confirmLabel="삭제"
  pendingLabel="삭제 중…"
  isPending={removing}
  onClose={() => !removing && setRemoveTargetId(null)}
  onConfirm={() => void confirmRemove()}
/>
```


```txt
포인트별 설명:

  open={removeTargetId !== null}
    id가 null이면 false(닫힘), 값이 있으면 true(열림)
    별도의 boolean 상태 없이 id 하나로 열기/닫기를 표현

  handleRemove(id): Dialog 열기만
    삭제 버튼 클릭 → API 호출 안 함 → Dialog 열어서 확인 받기
    "버튼 클릭 = 즉시 삭제"가 아니라 "버튼 클릭 = 확인 Dialog 열기"

  confirmRemove(): 실제 삭제
    if (!removeTargetId) return  → 혹시 null이면 방어
    성공 → setRemoveTargetId(null)으로 Dialog 닫기 + 목록 새로고침

  onClose={() => !removing && setRemoveTargetId(null)}
    removing 중에는 닫기 차단 (API 진행 중 닫히면 상태 꼬임)

  onConfirm={() => void confirmRemove()}
    confirmRemove는 async → Promise 반환
    void로 floating promise 처리 → [[JS_Promise]] 참고
```

## 여러 종류의 Dialog를 한 화면에 둘 때


```tsx
// 각각 id | null 상태로 관리
const [removeTargetId, setRemoveTargetId] = useState<string | null>(null);
const [blockTargetId,  setBlockTargetId]  = useState<string | null>(null);

// 목록 아이템
{friends.map(friend => (
  <div key={friend.id}>
    <button onClick={() => setRemoveTargetId(friend.id)}>삭제</button>
    <button onClick={() => setBlockTargetId(friend.id)}>차단</button>
  </div>
))}

{/* Dialog는 각각 독립적 */}
<Dialog
  open={removeTargetId !== null}
  onClose={() => setRemoveTargetId(null)}
  onConfirm={() => void confirmRemove()}
  title="친구를 삭제할까요?"
  confirmLabel="삭제"
/>
<Dialog
  open={blockTargetId !== null}
  onClose={() => setBlockTargetId(null)}
  onConfirm={() => void confirmBlock()}
  title="차단하시겠습니까?"
  confirmLabel="차단"
/>
```


---

# 사용법 ⭐️⭐️⭐️⭐️

```tsx
function DeleteButton({ id }: { id: string }) {
  const [open, setOpen]         = useState(false);
  const [isPending, setIsPending] = useState(false);

  const handleConfirm = async () => {
    setIsPending(true);
    try {
      await deleteItem(id);
      setOpen(false);
    } finally {
      setIsPending(false);
    }
  };

  return (
    <>
      <button onClick={() => setOpen(true)}>삭제</button>

      <Dialog
        open={open}
        onClose={() => setOpen(false)}
        onConfirm={handleConfirm}
        title="삭제하시겠습니까?"
        description="삭제하면 되돌릴 수 없어요."
        confirmLabel="삭제"
        pendingLabel="삭제 중…"
        cancelLabel="취소"
        isPending={isPending}
      />
    </>
  );
}
```

```txt
open/onClose 관리는 부모가 담당:
  Dialog 자체는 "열려있나/닫아라/확인했다"를 props로만 받음
  언제 열고 닫을지는 부모가 결정 → 재사용성이 높아짐

isPending은 Dialog 바깥에서 관리:
  API 호출 로직이 Dialog 안에 없어서 Dialog는 순수하게 UI만 담당
  비동기 상태 패턴 → [[React_AsyncUI]] 참고
```

---

# Portal 대상 커스터마이징

```tsx
// document.body 대신 특정 요소에 렌더링
const portalTarget = document.getElementById('modal-root');
return createPortal(<Dialog />, portalTarget ?? document.body);
```

```html
<!-- index.html 또는 layout.tsx -->
<body>
  <div id="root"></div>
  <div id="modal-root"></div>  <!-- Dialog 전용 컨테이너 -->
</body>
```

---

# 한눈에

```txt
createPortal(children, domNode):
  children을 domNode 위치에 렌더링 (DOM 위치 ≠ 컴포넌트 트리 위치)
  이벤트는 컴포넌트 트리 기준으로 버블링 (DOM 위치 무관)

mounted 패턴:
  useState(false) → useEffect에서 true
  SSR에서 document.body 없음 → mounted 전엔 null 반환

배경 클릭 닫기 + 패널 클릭 차단:
  배경 div: onClick={() => onClose()}
  패널 div: onClick={(e) => e.stopPropagation()}

Escape 키:
  useEffect에서 keydown 등록 → open/isPending deps
  cleanup으로 리스너 제거 필수

@deprecated:
  오타/이름 변경 시 하위 호환 유지 + VSCode 취소선 표시
  ?? 체인으로 두 props 모두 지원

접근성:
  role="dialog" / aria-modal="true" / aria-labelledby 세트

isPending 중 닫기 차단:
  배경 클릭 / Escape 키 모두 isPending 체크
```