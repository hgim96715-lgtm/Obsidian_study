---
aliases:
  - CSS custom property
  - CSS variable
  - CSSProperties
tags:
  - React
  - CSS
  - Tailwind
related:
  - "[[00_JS_Ecosystem_HomePage]]"
  - "[[JS_BrowserAPI]]"
  - "[[TS_TypeAssertion]]"
  - "[[React_Styling]]"
---
# React_CSSProperties — 인라인 스타일 타입

> [!info] 
> `CSSProperties` = React 인라인 `style` 속성의 TypeScript 타입.
>  `import { type CSSProperties } from 'react'`.
>   camelCase로 CSS 속성을 쓰고, CSS 커스텀 속성(`--variable`)을 인라인으로 넘길 때 `as CSSProperties`로 타입 단언이 필요하다.

---

# CSSProperties란 ⭐️⭐️⭐️⭐️

```txt
React의 style prop은 문자열이 아닌 객체를 받음

  HTML: style="color: red; font-size: 14px"  (문자열)
  React: style={{ color: 'red', fontSize: 14 }}  (객체)

CSSProperties = 이 style 객체의 TypeScript 타입
  어떤 CSS 속성이 있는지, 타입이 뭔지 정의
  오타 방지 + 자동완성

import { type CSSProperties } from 'react';
```

---

# 기본 사용법 ⭐️⭐️⭐️⭐️

```typescript
import { type CSSProperties } from 'react';

// style 객체 타입 지정
const cardStyle: CSSProperties = {
  color:           'red',
  fontSize:        14,         // px 단위는 숫자로 (단위 생략)
  backgroundColor: '#fff',    // camelCase (background-color → backgroundColor)
  borderRadius:    '8px',     // 단위 있으면 문자열
  display:         'flex',
  zIndex:          10,
};

// JSX에서 사용
<div style={cardStyle}>...</div>

// 인라인으로
<div style={{ color: 'red', fontSize: 14 }}>...</div>
```

```txt
CSS → CSSProperties 변환 규칙:
  background-color → backgroundColor  (하이픈 → camelCase)
  font-size        → fontSize
  border-radius    → borderRadius
  z-index          → zIndex

숫자 vs 문자열:
  fontSize: 14       → '14px'로 자동 변환 (px 단위 자동 추가)
  fontSize: '1rem'   → 단위 있으면 문자열로
  zIndex: 10         → 숫자
  opacity: 0.5       → 숫자
```

---

# CSS 커스텀 속성 (`--variable`) + `as CSSProperties` ⭐️⭐️⭐️⭐️

```typescript
// CSS 커스텀 속성(변수)을 인라인 style로 넘기기
<div
  style={{
    left:              layout.left,
    top:               layout.top,
    '--wall-cap-hue':  layout.hue,    // CSS 변수
    '--wall-cap-delay': layout.delay, // CSS 변수
    '--wall-cap-rot':  layout.rotate, // CSS 변수
    zIndex:            layout.z,
  } as CSSProperties}
/>
```

```txt
왜 as CSSProperties가 필요한가:

  CSSProperties 타입은 알려진 CSS 속성만 포함
  (color, fontSize, zIndex ...)

  '--wall-cap-hue' 같은 CSS 커스텀 속성(--로 시작)은
  CSSProperties 타입에 없음 → TypeScript 에러

  as CSSProperties:
  "이 객체를 CSSProperties로 간주해라"
  → 커스텀 속성이 있어도 타입 에러 없음

  CSS 커스텀 속성(CSS 변수):
  CSS에서 --이름으로 선언, var(--이름)으로 사용
  JavaScript에서 인라인 style로 값을 동적으로 바꿀 때 유용
```

## CSS 변수와 함께 쓰는 이유

```css
/* globals.css — CSS 변수를 받아서 사용 */
.wall-cap {
  transform: rotate(calc(var(--wall-cap-rot) * 1deg))
             scale(var(--wall-cap-scale));
  animation-delay: calc(var(--wall-cap-delay) * 1s);
  filter: hue-rotate(calc(var(--wall-cap-hue) * 1deg));
}
```

```typescript
// React — 동적으로 CSS 변수 값만 바꿈
<div
  className="wall-cap"
  style={{
    '--wall-cap-rot':   45,    // CSS에서 계산식으로 처리
    '--wall-cap-scale': 1.2,
    '--wall-cap-delay': 0.3,
    '--wall-cap-hue':   180,
  } as CSSProperties}
/>
```

```txt
왜 이 방식을 쓰는가:
  CSS 클래스는 고정값 — 동적으로 바꾸려면 새 클래스가 필요
  CSS 변수는 JS에서 값만 바꾸면 CSS가 나머지 처리
  → className은 그대로, style로 변수값만 주입

  예: 카드마다 다른 색상·각도·딜레이
  → 클래스 수십 개 만들 필요 없이 CSS 변수로 해결
  → 애니메이션 타이밍, 회전, 색조 등 유동적 값에 많이 씀
```

---

# 타입을 변수로 선언하는 패턴

```typescript
// 여러 곳에서 같은 스타일 쓸 때
const overlayStyle: CSSProperties = {
  position:  'fixed',
  inset:     0,
  zIndex:    9999,
  background: 'rgba(0,0,0,0.5)',
};

// 커스텀 속성 포함 — 타입 명시
const cardStyle = {
  '--card-hue': hue,
  zIndex: 10,
} as CSSProperties;

// 함수로 동적 생성
function getStyle(hue: number, delay: number): CSSProperties {
  return {
    '--hue':   hue,
    '--delay': delay,
  } as CSSProperties;
}
```

---

# CSS 속성 주요 타입

```typescript
type CSSProperties = {
  color?:           string;
  fontSize?:        string | number;  // 14 또는 '1rem'
  fontWeight?:      string | number;  // 700 또는 'bold'
  display?:         string;           // 'flex' | 'grid' | 'block' ...
  position?:        string;           // 'fixed' | 'absolute' | 'relative' ...
  zIndex?:          string | number;
  backgroundColor?: string;
  opacity?:         number;           // 0~1
  // ... CSS 속성 전부
};
```

```txt
선택적 속성(?)인 이유:
  style 객체에 모든 CSS 속성을 다 쓸 필요가 없음
  필요한 것만 넣으면 됨
```