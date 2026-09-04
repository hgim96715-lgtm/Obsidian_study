---
aliases: [커스텀 드롭다운, click-outside, listbox, combobox]
tags: [react]
related:
  - "[[00_JS_Ecosystem_HomePage]]"
  - "[[HTML_ARIA]]"
  - "[[JS_DOM]]"
  - "[[React_useRef]]"
  - "[[TS_DOM_Events]]"
---
# React_CustomSelect — 커스텀 드롭다운 패턴

> [!info]
> 커스텀 드롭다운 = native `<select>` 대신 div·button으로 직접 구현한 드롭다운.
> 핵심 3가지: **ARIA 역할 구조** · **click-outside 감지** · **useEffect cleanup**.

---

# 왜 native `<select>`를 안 쓰는가 ⭐️⭐️⭐️⭐️

```txt
native <select>의 한계:
  OS·브라우저마다 기본 스타일이 다름 → 디자인 통일 불가
  드롭다운 열릴 때 애니메이션 커스텀 불가
  옵션 안에 이미지·아이콘·설명 텍스트 등 복잡한 레이아웃 불가
  멀티 선택 UI 커스텀 불가

커스텀 드롭다운으로 해결:
  div + button으로 완전한 UI 제어
  단, 접근성(ARIA)을 직접 구현해야 함 — native select는 기본 제공
  → ARIA 역할을 제대로 달아야 스크린리더가 "리스트박스"로 인식
```

---

# 컴포넌트 구조 — 3층 레이어 ⭐️⭐️⭐️⭐️

```txt
[1] 트리거 버튼 (trigger button)
    → 드롭다운을 여닫는 버튼
    → aria-haspopup="listbox" + aria-expanded={open}

[2] 리스트박스 컨테이너 (listbox)
    → 열렸을 때만 렌더링되는 목록 컨테이너
    → role="listbox"

[3] 옵션 버튼들 (option)
    → 각 선택 항목
    → role="option" + aria-selected={선택됐는지}
```

```tsx
// 구조 스켈레톤
<div ref={rootRef} className="relative">

  {/* [1] 트리거 버튼 */}
  <button
    type="button"
    aria-haspopup="listbox"
    aria-expanded={open}
    onClick={() => setOpen((v) => !v)}
  >
    {selectedLabel}
    <ChevronDown aria-hidden />
  </button>

  {/* [2] 리스트박스 */}
  {open && (
    <div role="listbox">
      {options.map((opt) => (
        /* [3] 옵션 버튼 */
        <button
          key={opt.value}
          type="button"
          role="option"
          aria-selected={value === opt.value}
          onClick={() => { onChange(opt.value); setOpen(false); }}
        >
          {opt.label}
        </button>
      ))}
    </div>
  )}
</div>
```

---

# click-outside 감지 ⭐️⭐️⭐️⭐️

```txt
문제:
  드롭다운이 열린 상태에서 바깥 영역을 클릭하면 자동으로 닫혀야 함
  React의 이벤트 시스템만으로는 "컴포넌트 바깥 클릭"을 감지하기 어려움

해결:
  document에 pointerdown 리스너를 달고
  클릭 좌표가 컴포넌트 rootRef 안인지 밖인지 판별
```

## rootRef — 컨테이너 참조

```tsx
const rootRef = useRef<HTMLDivElement>(null);

<div ref={rootRef} className="relative">
  ...
</div>
```

```txt
rootRef.current = 전체 드롭다운 컨테이너 DOM 요소
contains(node) = 이 요소가 node를 자손으로 포함하는지 확인

rootRef.current.contains(event.target as Node)
  → true  = 컴포넌트 안을 클릭 → 닫지 않음
  → false = 컴포넌트 밖을 클릭 → 닫음
```

## useEffect + document.addEventListener + cleanup ⭐️⭐️⭐️⭐️

```tsx
useEffect(() => {
  if (!open) return;  // 닫혀있으면 리스너 불필요

  function handlePointerDown(event: PointerEvent) {
    if (!rootRef.current?.contains(event.target as Node)) {
      setOpen(false);  // 바깥 클릭 → 닫기
    }
  }

  document.addEventListener("pointerdown", handlePointerDown);

  return () => {
    document.removeEventListener("pointerdown", handlePointerDown);
  };
}, [open]);
```

```txt
동작 흐름:
  open이 true로 바뀜
  → useEffect 실행 → document에 pointerdown 리스너 등록
  → 어디든 클릭 시 handlePointerDown 호출
  → rootRef.current.contains(클릭된 요소) 판별
  → 바깥이면 setOpen(false)
  → open이 false로 바뀜 → useEffect cleanup 실행 → 리스너 제거

cleanup (return () => ...) 이 필수인 이유:
  open=true일 때 리스너를 달았으면
  open=false가 됐을 때 (또는 컴포넌트 언마운트 시) 반드시 제거해야 함
  제거 안 하면 → 닫힌 드롭다운의 리스너가 계속 document에 쌓임
  → 메모리 누수 + 의도치 않은 setOpen 호출

open을 의존성에 넣는 이유:
  open이 true → false로 바뀔 때 cleanup + 재실행 (if !open return)
  open이 false → true로 바뀔 때 새 리스너 등록
```

## pointerdown vs click 사용 이유

```txt
click 이벤트:
  pointerdown → pointerup → click 순서로 발생
  blur가 먼저 실행돼서 드롭다운이 닫힌 뒤에 click이 오는 경우 있음

pointerdown:
  가장 먼저 발생 → 빠르게 외부 클릭 감지
  마우스·터치·펜 모두 커버 (Pointer Events 통합 API)
  → 외부 클릭 감지 패턴에서 pointerdown이 관례
```

---

# ARIA 역할 구조 ⭐️⭐️⭐️⭐️

```txt
ARIA 개념 상세 → [[HTML_ARIA]]
```

|속성/역할|위치|역할|
|---|---|---|
|`aria-haspopup="listbox"`|트리거 버튼|"이 버튼이 listbox를 열어" — 스크린리더에 팝업 종류 알림|
|`aria-expanded={open}`|트리거 버튼|현재 열림/닫힘 상태 알림|
|`role="listbox"`|드롭다운 컨테이너|"이 div가 선택 목록이야"|
|`role="option"`|각 옵션 버튼|"이 버튼이 선택 항목이야"|
|`aria-selected={...}`|각 옵션 버튼|현재 선택된 항목 표시|
|`aria-hidden`|장식 아이콘(ChevronDown)|스크린리더가 읽지 않도록|
|`type="button"`|모든 버튼|`<form>` 안에 있을 때 submit 방지|

---

# 전체 구현 — 범용 컴포넌트 ⭐️⭐️⭐️⭐️

```tsx
import { useEffect, useRef, useState } from "react";
import { ChevronDown } from "lucide-react";

interface Option<T extends string> {
  value: T;
  label: string;
}

interface CustomSelectProps<T extends string> {
  value: T | null;
  onChange: (value: T) => void;
  options: Option<T>[];
  placeholder?: string;
  "aria-label"?: string;
}

function CustomSelect<T extends string>({
  value,
  onChange,
  options,
  placeholder = "선택하세요",
  "aria-label": ariaLabel,
}: CustomSelectProps<T>) {
  const [open, setOpen] = useState(false);
  const rootRef = useRef<HTMLDivElement>(null);

  const selectedLabel = options.find((o) => o.value === value)?.label;

  useEffect(() => {
    if (!open) return;

    function handlePointerDown(event: PointerEvent) {
      if (!rootRef.current?.contains(event.target as Node)) {
        setOpen(false);
      }
    }

    document.addEventListener("pointerdown", handlePointerDown);
    return () => {
      document.removeEventListener("pointerdown", handlePointerDown);
    };
  }, [open]);

  return (
    <div ref={rootRef} className="relative">
      <button
        type="button"
        aria-haspopup="listbox"
        aria-expanded={open}
        aria-label={ariaLabel}
        onClick={() => setOpen((v) => !v)}
        className="flex items-center gap-2"
      >
        <span>{selectedLabel ?? placeholder}</span>
        <ChevronDown
          aria-hidden
          className={`transition-transform ${open ? "rotate-180" : ""}`}
        />
      </button>

      {open && (
        <div
          role="listbox"
          aria-label={ariaLabel}
          className="absolute z-10 mt-1 w-full rounded border bg-white shadow"
        >
          {options.map((opt) => (
            <button
              key={opt.value}
              type="button"
              role="option"
              aria-selected={value === opt.value}
              onClick={() => {
                onChange(opt.value);
                setOpen(false);
              }}
              className="w-full px-3 py-2 text-left hover:bg-gray-100"
            >
              {opt.label}
            </button>
          ))}
        </div>
      )}
    </div>
  );
}
```

```tsx
// 사용 예
type Genre = "action" | "romance" | "horror";

const GENRE_OPTIONS: Option<Genre>[] = [
  { value: "action", label: "액션" },
  { value: "romance", label: "로맨스" },
  { value: "horror", label: "공포" },
];

function MovieFilter() {
  const [genre, setGenre] = useState<Genre | null>(null);

  return (
    <CustomSelect
      value={genre}
      onChange={setGenre}
      options={GENRE_OPTIONS}
      aria-label="장르 선택"
    />
  );
}
```

---

# 주의사항 ⭐️⭐️⭐️

```txt
type="button" 누락 주의:
  <form> 안에 있을 때 type 없는 button은 기본값이 type="submit"
  → 옵션 클릭 시 폼이 제출돼버림
  → 트리거 버튼 + 옵션 버튼 모두 type="button" 명시

role="option"은 role="listbox" 안에만:
  role="option"은 반드시 role="listbox" 컨테이너 안에 있어야 함
  → ARIA 규칙 위반 시 스크린리더가 옵션을 인식 못함

contains(event.target as Node):
  event.target은 EventTarget 타입 → Node로 단언 필요
  contains()는 Node 메서드이기 때문
  rootRef.current가 null일 수 있으므로 ?. 옵셔널 체이닝 사용
```

| 핵심 | 설명 |
|---|---|
| `rootRef` | 컨테이너 전체를 가리키는 ref — contains()로 내/외부 판별 |
| `document.addEventListener` | React 이벤트 시스템 밖 — 전체 문서 클릭 감지 |
| `cleanup` | open=false·언마운트 시 리스너 반드시 제거 — 누수 방지 |
| `aria-haspopup="listbox"` | 트리거 버튼이 listbox를 열어준다는 신호 |
| `role="listbox"` + `role="option"` | 스크린리더가 드롭다운 구조를 인식하는 핵심 |
