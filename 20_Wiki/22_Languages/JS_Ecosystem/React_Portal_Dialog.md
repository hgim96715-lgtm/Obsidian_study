---
aliases:
  - 마운트(mount)
  - createPortal
  - Hydration
  - Modal
  - mounted pattern
  - Portal
tags:
  - React
related:
  - "[[00_JS_Ecosystem_HomePage]]"
  - "[[JS_BrowserAPI]]"
  - "[[NextJS_AuthState]]"
  - "[[NextJS_ServerClient]]"
  - "[[React_useId]]"
---
# React_Portal — Portal · 모달 다이얼로그

> [!info] 
> Portal = React 컴포넌트를 부모 DOM 밖의 다른 곳(보통 `document.body`)에 렌더링하는 방법.
>  모달·다이얼로그·토스트처럼 페이지 최상단에 떠야 하는 UI에 필수.
>   `createPortal(children, container)`로 사용.

---

# Portal이 왜 필요한가 ⭐️⭐️⭐️⭐️

```txt
모달을 컴포넌트 트리 안에 그냥 넣으면 문제가 생김

문제 1 — overflow: hidden:
  부모 컴포넌트에 overflow: hidden이 있으면
  그 안에 렌더링된 모달이 잘려서 보임

문제 2 — z-index 쌓임 맥락(stacking context):
  부모의 z-index 범위 안에 갇혀서
  다른 요소 위에 제대로 뜨지 않을 수 있음

해결:
  모달을 DOM 트리 최상단(document.body)에 직접 붙임
  부모의 overflow, z-index에 영향 받지 않음
  → createPortal
```

```tsx
// ❌ 컴포넌트 안에 그냥 렌더링
function Card() {
  return (
    <div style={{ overflow: 'hidden' }}>   {/* 모달이 잘림 */}
      <Modal />
    </div>
  );
}

// ✅ document.body에 Portal로 렌더링
createPortal(<Modal />, document.body);   // 부모 overflow 무관
```

---

# createPortal 기본 사용 ⭐️⭐️⭐️⭐️

```tsx
import { createPortal } from 'react-dom';

function Modal({ open }: { open: boolean }) {
  if (!open) return null;

  return createPortal(
    <div className="fixed inset-0 z-[100] bg-black/40">
      <div>모달 내용</div>
    </div>,
    document.body,   // ← 렌더링할 위치 (DOM 노드)
  );
}
```

```txt
createPortal(children, container):
  children   → 렌더링할 React 요소
  container  → 렌더링될 DOM 노드 (보통 document.body)

  React 트리에서는 부모-자식 관계 유지 (이벤트 버블링 등)
  DOM에서는 container(document.body) 아래에 붙음

  → Portal 안에서 useContext, props 전달 전부 정상 작동
```

## SSR 안전한 mounted 패턴 ⭐️⭐️⭐️

```tsx
export function Modal({ open }: { open: boolean }) {
  const [mounted, setMounted] = useState(false);

  useEffect(() => {
    setMounted(true);   // 클라이언트에서만 실행됨
  }, []);

  if (!open || !mounted) return null;

  return createPortal(<div>...</div>, document.body);
}
```

```txt
mounted 상태가 왜 필요한가:
  Next.js 같은 SSR 환경에서
  서버에서 렌더링할 때 document가 존재하지 않음
  createPortal(el, document.body) → document 없음 → 에러

  useEffect는 클라이언트에서만 실행됨
  → setMounted(true)가 실행되면 document 존재 확정
  → mounted가 true일 때만 createPortal 실행
```

---

# 모달 기본 구조 ⭐️⭐️⭐️⭐️

```tsx
function Modal({ open, onClose }: { open: boolean; onClose: () => void }) {
  const [mounted, setMounted] = useState(false);
  useEffect(() => { setMounted(true); }, []);

  // ESC 키로 닫기
  useEffect(() => {
    if (!open) return;
    function onKeyDown(e: KeyboardEvent) {
      if (e.key === 'Escape') onClose();
    }
    window.addEventListener('keydown', onKeyDown);
    return () => window.removeEventListener('keydown', onKeyDown);
  }, [open, onClose]);

  if (!open || !mounted) return null;

  return createPortal(
    // ① 배경 오버레이 — 클릭하면 닫힘
    <div
      className="fixed inset-0 z-[100] flex items-center justify-center bg-black/40 p-4"
      role="dialog"
      aria-modal="true"
      onClick={onClose}>           {/* 배경 클릭 → 닫기 */}

      {/* ② 다이얼로그 패널 — 클릭해도 안 닫힘 */}
      <div
        className="relative w-full max-w-sm rounded-xl bg-white p-6"
        onClick={(e) => e.stopPropagation()}>  {/* 버블링 차단 */}

        모달 내용
      </div>
    </div>,
    document.body,
  );
}
```

```txt
구조 설명:

  ① 배경 오버레이 (fixed inset-0):
    화면 전체를 덮는 반투명 배경
    onClick={onClose} → 배경 클릭 시 닫기

  ② 다이얼로그 패널:
    실제 모달 내용이 있는 영역
    e.stopPropagation() → 여기서 클릭해도 배경까지 이벤트가 올라가지 않음
    → 패널 클릭으로 실수로 닫히는 것 방지

  e.stopPropagation() 이 없으면:
    모달 안에서 버튼 클릭 → 이벤트가 배경까지 버블링 → onClose 실행 → 모달 닫힘
```

## 접근성 속성 ⭐️⭐️⭐️

```tsx
<div
  role="dialog"                          // 스크린 리더: "이건 다이얼로그야"
  aria-modal="true"                      // 스크린 리더: "모달 뒤의 내용 무시해"
  aria-labelledby="dialog-title">        // 제목 요소와 연결

  <h2 id="dialog-title">제목</h2>        // aria-labelledby와 id 매칭
</div>
```

---

# 재사용 가능한 베이스 다이얼로그 ⭐️⭐️⭐️⭐️

```tsx
type BaseDialogProps = {
  open:          boolean;
  onClose:       () => void;
  onConfirm:     () => void;
  title?:        string;
  description?:  string;
  confirmLabel?: string;
  pendingLabel?: string;
  cancelLabel?:  string;
  isPending?:    boolean;
  children?:     React.ReactNode;   // 추가 내용을 주입받는 슬롯
};

export function BaseDialog({
  open,
  onClose,
  onConfirm,
  title       = '확인',
  description,
  confirmLabel = '확인',
  pendingLabel,
  cancelLabel  = '취소',
  isPending    = false,
  children,
}: BaseDialogProps) {
  const [mounted, setMounted] = useState(false);
  useEffect(() => { setMounted(true); }, []);

  const actionLabel        = confirmLabel;
  const actionPendingLabel = pendingLabel ?? `${confirmLabel} 중…`;

  // ESC 키
  useEffect(() => {
    if (!open) return;
    const onKeyDown = (e: KeyboardEvent) => {
      if (e.key === 'Escape' && !isPending) onClose();
    };
    window.addEventListener('keydown', onKeyDown);
    return () => window.removeEventListener('keydown', onKeyDown);
  }, [open, isPending, onClose]);

  if (!open || !mounted) return null;

  return createPortal(
    <div
      className="fixed inset-0 z-[100] flex items-center justify-center bg-black/40 p-4"
      role="dialog"
      aria-modal="true"
      aria-labelledby="base-dialog-title"
      onClick={() => { if (!isPending) onClose(); }}>

      <div
        className="relative w-full max-w-sm rounded-xl bg-white p-6"
        onClick={(e) => e.stopPropagation()}>

        <h2 id="base-dialog-title" className="text-center text-lg font-semibold">
          {title}
        </h2>

        {description && (
          <p className="mt-2 text-center text-sm text-gray-500">{description}</p>
        )}

        {/* children 슬롯 — 추가 내용 주입 가능 */}
        {children && <div className="mt-4">{children}</div>}

        <div className="mt-6 flex flex-col gap-2">
          <button
            onClick={onConfirm}
            disabled={isPending}
            className="rounded-full bg-blue-500 py-2 text-white disabled:opacity-50">
            {isPending ? actionPendingLabel : actionLabel}
          </button>
          <button
            onClick={onClose}
            disabled={isPending}
            className="py-2 text-sm text-gray-400 disabled:opacity-50">
            {cancelLabel}
          </button>
        </div>
      </div>
    </div>,
    document.body,
  );
}
```

```txt
isPending 상태:
  API 요청 중이면 true
  버튼 disabled 처리 → 중복 클릭 방지
  배경 클릭·ESC 닫기도 isPending 중엔 막음
  확인 버튼 문구가 "삭제" → "삭제 중…"으로 변경
```

---

# 컴포지션 — 베이스 다이얼로그 확장 ⭐️⭐️⭐️⭐️

```txt
BaseDialog를 그대로 쓰면 단순 확인/취소 모달
children에 추가 내용을 주입하면 커스텀 모달로 확장
```

```tsx
// 신고 다이얼로그 — BaseDialog의 children에 textarea 주입
export function ReportDialog({
  open,
  isPending = false,
  onClose,
  onSubmit,
}: {
  open:      boolean;
  isPending?: boolean;
  onClose:   () => void;
  onSubmit:  (reason: string) => Promise<void>;
}) {
  const [reason,      setReason]      = useState('');
  const [reasonError, setReasonError] = useState('');

  // 닫힐 때 내부 상태 초기화
  useEffect(() => {
    if (!open) {
      setReason('');
      setReasonError('');
    }
  }, [open]);

  async function handleConfirm() {
    const trimmed = reason.trim();
    if (trimmed.length < 2) {
      setReasonError('신고 사유를 입력해 주세요.');
      return;
    }
    setReasonError('');
    try {
      await onSubmit(trimmed);
      setReason('');
    } catch (err) {
      setReasonError(err instanceof Error ? err.message : '신고에 실패했어요.');
    }
  }

  return (
    <BaseDialog
      open={open}
      title="신고할까요?"
      description="운영 검토용이에요. 허위 신고는 제재될 수 있어요."
      confirmLabel="신고"
      pendingLabel="신고 중…"
      isPending={isPending}
      onClose={() => { if (!isPending) onClose(); }}
      onConfirm={() => void handleConfirm()}>

      {/* children으로 textarea 주입 */}
      <div className="space-y-1.5">
        <textarea
          value={reason}
          onChange={(e) => {
            setReason(e.target.value);
            if (reasonError) setReasonError('');
          }}
          placeholder="신고 사유 (필수, 2자 이상)"
          rows={3}
          maxLength={500}
          disabled={isPending}
          className="w-full rounded border p-2 text-sm"
        />
        {reasonError && (
          <p className="text-xs text-red-500" role="alert">{reasonError}</p>
        )}
      </div>
    </BaseDialog>
  );
}
```

```txt
컴포지션 패턴의 장점:
  BaseDialog — 모달 껍데기(오버레이·패널·버튼) 공통화
  각 다이얼로그 — children으로 필요한 내용만 추가

  BaseDialog를 수정하면 모든 다이얼로그에 적용
  새 다이얼로그 만들 때 중복 코드 없음
```

---

# 사용 패턴 ⭐️⭐️⭐️

```tsx
// 페이지나 상위 컴포넌트에서 open 상태 관리
function PostPage({ post }: { post: Post }) {
  const [isDeleteOpen, setDeleteOpen] = useState(false);
  const [isReportOpen, setReportOpen] = useState(false);

  const handleDelete = async () => {
    await deletePost(post.id);
    setDeleteOpen(false);
  };

  return (
    <>
      <button onClick={() => setDeleteOpen(true)}>삭제</button>
      <button onClick={() => setReportOpen(true)}>신고</button>

      {/* 삭제 — 단순 확인 모달 */}
      <BaseDialog
        open={isDeleteOpen}
        onClose={() => setDeleteOpen(false)}
        onConfirm={() => void handleDelete()}
        title="삭제할까요?"
        description="삭제하면 되돌릴 수 없어요."
        confirmLabel="삭제"
      />

      {/* 신고 — 사유 입력 포함 */}
      <ReportDialog
        open={isReportOpen}
        onClose={() => setReportOpen(false)}
        onSubmit={async (reason) => {
          await reportPost(post.id, reason);
          setReportOpen(false);
        }}
      />
    </>
  );
}
```

```txt
모달 open 상태 위치:
  모달을 쓰는 컴포넌트에서 useState로 관리
  여러 곳에서 같은 모달을 써야 하면 Context나 전역 상태로 올리기

void handleConfirm():
  onConfirm은 () => void 타입인데
  handleConfirm은 async function → Promise를 반환
  void 키워드로 "반환값 무시" 처리 → TypeScript 에러 없음
```