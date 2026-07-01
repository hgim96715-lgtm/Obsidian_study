---
aliases: [마운트(mount), createPortal, Hydration, mounted pattern, Portal]
tags: [React]
related:
  - "[[00_JS_Ecosystem_HomePage]]"
  - "[[JS_BrowserAPI]]"
  - "[[NextJS_ServerClient]]"
  - "[[React_useId]]"
  - "[[React_useMemo_useCallback_useEffect]]"
---
# React_Portal — createPortal로 DOM 트리 밖에 렌더링하기

> [!info] 
> createPortal(children, container)는 React 컴포넌트 트리 구조는 그대로 유지하면서, 실제 DOM에는 다른 위치(보통 document.body)에 렌더링하는 API다. 모달/다이얼로그가 부모의 CSS(overflow, z-index 등)에 갇히지 않게 하려고 주로 쓴다.

---

# createPortal이 하는 일 ⭐️⭐️⭐️⭐️

```tsx
import { createPortal } from 'react-dom';

return createPortal(
  <div className="modal">...</div>,
  document.body,
);
```

```txt
첫 번째 인자: 렌더링할 JSX
두 번째 인자: 그걸 실제로 붙일 DOM 노드

컴포넌트 트리 관점(React 입장)에서는 여전히 원래 부모 안에 있는 것처럼 동작함
  → state/context/이벤트 버블링은 평소처럼 그대로 작동함
실제 DOM 위치만 다름 — 화면에 그려지는 자리가 document.body 바로 아래로 옮겨짐
```

---

# 왜 Dialog/Modal에 필요한가 ⭐️⭐️⭐️⭐️

```txt
모달이 평소처럼 컴포넌트 트리 깊은 곳(다른 div들 안)에서 그려지면:
  부모 중 하나가 overflow: hidden 이면 모달이 그 영역 밖으로 잘려서 안 보일 수 있음
  부모의 z-index 쌓임 순서에 갇혀서, 의도와 다르게 다른 요소 뒤에 깔릴 수 있음

document.body 바로 아래에 직접 그리면:
  부모들의 overflow/z-index 제약을 전부 건너뛰고, 화면 전체 기준으로 독립적으로 위치/겹침을 제어할 수 있음
  → 모달/다이얼로그/툴팁처럼 "어디서 호출되든 화면 맨 위에 떠야 하는" UI에 createPortal이 거의 표준처럼 쓰임
```

---

# document.body가 꼭 필요한가 — 두 번째 인자는 "어떤 DOM 노드든" 가능 ⭐️⭐️⭐️

```txt
document.body는 가장 간단한 선택일 뿐, 반드시 그래야 하는 건 아님
어떤 DOM 노드를 두 번째 인자로 넘기든 동일하게 동작함

더 안전하게 가려면 — 전용 컨테이너를 따로 만들기:
  document.body 자체에 다른 라이브러리나 스크립트가 자식을 추가하는 경우,
  여러 포털이 서로 자식 순서/스타일을 건드릴 위험이 있음
  → <div id="portal-root"></div> 를 layout에 미리 하나 두고, 그 노드를 두 번째 인자로 쓰는 패턴도 흔함

핵심은 "이 컴포넌트가 원래 위치한 DOM 트리 바깥의, 화면 전체 기준으로 자유로운 어떤 노드"라는 것
```

---

# mounted + useEffect 패턴 — 왜 꼭 필요한가 ⭐️⭐️⭐️⭐️

```txt
먼저 두 용어부터 정확히 짚기:

마운트(mount):
  JSX를 실제 DOM 노드로 변환 → 그 DOM 노드를 화면에 삽입 → 마운트 완료
  → 마운트가 완료되는 바로 그 시점에 useEffect가 실행됨 (이 둘은 사실상 같은 순간)

Hydration:
  서버가 HTML 생성 → 브라우저가 그 HTML을 먼저 화면에 표시 → (보통 동시에) JS 다운로드
  → React가 그 기존 HTML에 이벤트 리스너와 내부 상태를 "연결"함 (DOM을 다시 만드는 게 아니라 재사용)

이 둘의 관계: SSR된 컴포넌트는 "마운트"가 곧 "hydration이 끝나는 시점"과 맞물림 —
서버가 만든 HTML은 이미 화면에 있지만, useEffect 같은 React의 동작은 hydration이 끝나야 실행됨
```


```tsx
const [mounted, setMounted] = useState(false);

useEffect(() => {
  setMounted(true);
}, []);

if (!open || !mounted) return null;
```


```txt
document 자체가 서버에는 없음(SSR 환경) — 이 자체는 typeof window 체크와 같은 이유
([[JS_BrowserAPI]]의 "환경 감지" 패턴, [[NextJS_ServerClient]] 참고)

근데 여기서 typeof window === 'undefined' 를 인라인으로 바로 쓰는 대신
useState+useEffect로 한 단계 더 늦추는 이유가 따로 있음 — hydration mismatch까지 막기 위해서
```

## typeof window를 인라인으로 직접 쓰면 생기는 문제 ⭐️⭐️⭐️⭐️

```txt
만약 if (typeof window === 'undefined') return null; 을 렌더링 중에 바로 쓴다면:

  서버에서 렌더링할 때    → window가 없음 → false 평가 → null 렌더 (HTML에 모달 없음)
  클라이언트가 hydration할 때 → window는 이미 있음(클라이언트는 처음부터 window가 존재) →
                              true로 평가돼서 곧바로 모달 내용을 렌더링하려고 함

→ 서버가 만든 HTML(모달 없음)과 클라이언트가 처음 그리려는 결과(모달 있음)가 서로 달라서
  React가 "Hydration failed" 경고를 내거나, 강제로 클라이언트 기준으로 다시 그려야 하는 상황이 생김
```

## mounted+useEffect가 이 문제를 피하는 방식 ⭐️⭐️⭐️⭐️

```txt
useEffect는 컴포넌트가 마운트되고 hydration까지 끝난 "이후"에만 실행됨
([[React_useMemo_useCallback_useEffect]]의 useEffect 타이밍 참고)

  서버에서 렌더링할 때         → mounted 초기값 false → null 렌더
  클라이언트가 hydration할 때  → 이 시점엔 아직 useEffect가 실행 전이라 mounted는 여전히 false
                                → null 렌더 → 서버 결과와 정확히 일치 (mismatch 없음)
  hydration이 끝난 뒤          → useEffect 실행 → setMounted(true) → 그제서야 다시 렌더링되며
                                  비로소 createPortal로 모달이 실제로 나타남

→ "서버와 클라이언트의 첫 렌더 결과를 강제로 일치시켜둔 뒤, 그 다음 턴에서 실제 내용을 보여주는" 트릭
  typeof window 인라인 체크보다 한 단계 느리지만(모달이 한 틱 늦게 나타남), hydration 경고가 없음
```

|방법|hydration mismatch|모달이 보이는 시점|
|---|---|---|
|typeof window === 'undefined' 인라인|위험 있음|즉시(hydration 시점부터)|
|mounted + useEffect|없음|hydration 완료 직후(거의 체감 안 되는 지연)|

---

# 실전 예시 한 줄씩 ⭐️⭐️⭐️

```tsx
export function Dialog({
  open,
  onClose,
  title = '삭제하시겠습니까?',
  description = '삭제하면 되돌릴 수 없어요.',
  confirmLabel = '삭제',
  isPending = false,
  onConfirm,
}: FeedDialogProps) {
  const [mounted, setMounted] = useState(false);

  useEffect(() => {
    setMounted(true);
  }, []);

  if (!open || !mounted) return null;

  return createPortal(
    <div role="dialog" aria-modal="true">
      <h2>{title}</h2>
      <p>{description}</p>
      <button onClick={onConfirm} disabled={isPending}>{confirmLabel}</button>
      <button onClick={onClose}>취소</button>
    </div>,
    document.body,
  );
}
```

```txt
if (!open || !mounted) return null; 에 두 조건이 같이 있는 이유:
  !open    다이얼로그를 열어야 할 상황이 아니면 애초에 그릴 필요가 없음 (일반적인 조건부 렌더링)
  !mounted 아직 클라이언트 마운트(hydration) 전이면, document.body 자체가 위험하거나
           hydration mismatch가 생길 수 있으므로 무조건 null

  → 둘 중 하나라도 해당하면 그릴 필요가 없거나 그리면 안 되는 상태이므로 OR(||)로 묶임
```

---

# 한눈에

| 개념                                | 핵심                                                       |
| --------------------------------- | -------------------------------------------------------- |
| createPortal(children, container) | React 트리 구조는 유지하면서 실제 DOM은 다른 위치에 렌더링                    |
| 왜 모달에 쓰나                          | 부모의 overflow/z-index 제약을 피해 화면 맨 위에 독립적으로 띄우기 위해         |
| 두 번째 인자                           | document.body가 흔하지만 필수는 아님 — 전용 # portal-root 노드도 흔히 씀   |
| mounted + useEffect               | typeof window 인라인 체크보다 한 단계 늦지만, hydration mismatch를 막아줌 |
| typeof window 인라인의 위험             | 클라이언트는 hydration 시점에 이미 window가 있어서 서버 결과와 즉시 불일치할 수 있음  |
| if (!open \| !mounted)            | "열려야 함"과 "마운트 끝남" 둘 다 필요 — 하나라도 아니면 렌더 안 함               |
