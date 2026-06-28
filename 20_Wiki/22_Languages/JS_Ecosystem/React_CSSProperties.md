---
aliases:
  - CSSProperties
  - CSS custom property
  - CSS variable
tags:
  - React
  - CSS
related:
  - "[[00_JS_Ecosystem_HomePage]]"
  - "[[JS_BrowserAPI]]"
  - "[[TS_TypeAssertion]]"
---

# React_CSSProperties — style 프롭과 CSS 커스텀 속성

> [!info] 
>  React의 `style={{...}}` 객체는 `CSSProperties` 타입으로 검사되는데, 이 타입은 `backgroundColor`/`fontSize` 같은 표준 CSS 속성만 알고 있다. 
>  `--card-back`처럼 `--`로 시작하는 CSS 커스텀 속성(변수)은 그 타입에 없어서, `as CSSProperties`로 강제 단언해야 컴파일이 통과된다.

---

# CSSProperties란 무엇인가 ⭐️⭐️⭐️

```typescriptreact
import type { CSSProperties } from 'react';

const style: CSSProperties = {
  backgroundColor: 'black', // CSS의 background-color → camelCase
  fontSize: '16px',
};
```

```txt
React의 style prop에 넣는 객체가 정확히 어떤 모양이어야 하는지를 정의해둔 타입
camelCase로 쓰는 규칙 자체는 React만의 것이 아니라, DOM의 el.style과 동일한 원리
(el.style.backgroundColor 같은 DOM 속성 자체에 대한 내용은 [[JS_BrowserAPI]]의 "style" 섹션 참고)

import type { ... } 인 이유: CSSProperties는 타입으로만 쓰이고 런타임에 실제로 존재하는
값이 아니라서, type-only import로 명시하면 컴파일 시 이 import 자체가 완전히 제거됨
```

---

# CSS 커스텀 속성(CSS Variables)이란 ⭐️⭐️⭐️⭐️

```css
:root {
  --primary-color: blue; /* -- 로 시작하는 이름으로 직접 정의 */
}

.button {
  color: var(--primary-color); /* var()로 어디서든 꺼내 쓸 수 있음 */
}
```

```txt
CSS 커스텀 속성 = 개발자가 직접 이름을 정해서 CSS 안에 값을 저장해두는 변수
  정의: --이름: 값;
  사용: var(--이름)

왜 쓰나:
  반복되는 값(색상, 간격 등)을 한 곳에서 관리(디자인 토큰)
  무엇보다 — JS에서 동적으로 값을 주입할 수 있는 통로가 됨 (바로 아래가 그 패턴)
```

---

# React에서 동적 값을 CSS로 넘기는 패턴 ⭐️⭐️⭐️⭐️

```typescriptreact
<div style={{ '--card-back': cardBack } as CSSProperties}>
```

```txt
이 한 줄이 하는 일:
  cardBack이라는 JS 변수(런타임에 정해지는 값)를 --card-back이라는 CSS 커스텀 속성에 주입함
  → 이후 CSS 파일에서는 .card { background-image: var(--card-back); } 처럼
    그 값을 그냥 꺼내 쓰면 됨 — 실제 스타일링 규칙은 CSS에 그대로 두고,
    "값"만 React가 동적으로 넘겨주는 역할 분담이 가능해짐

장점: 매번 인라인으로 모든 스타일을 다 적지 않고, "바뀌는 값만" React가 주고
나머지 스타일(hover, transition, 미디어쿼리 등 인라인 style로는 표현 못 하는 것들)은
CSS 파일에 그대로 작성할 수 있음
```

---

# 왜 as CSSProperties가 필요한가 ⭐️⭐️⭐️⭐️

```txt
CSSProperties 타입은 backgroundColor, fontSize 같은 "표준 CSS 속성 목록"만 키로 알고 있음
--card-back처럼 직접 만든 임의의 커스텀 속성 이름은 그 목록에 없어서,
TS가 "이런 속성은 모른다"며 타입 에러를 냄

→ { '--card-back': cardBack } 객체를 만들고 as CSSProperties로 "이건 CSSProperties로 취급해라"고
  강제로 단언해서 통과시키는 것 — as 자체가 검증이 아니라 단언이라는 점은 [[TS_TypeAssertion]] 참고
```

## 매번 as 없이 쓰는 법 — 커스텀 속성을 포함한 타입 만들기 ⭐️⭐️

```typescriptreact
type CardStyle = CSSProperties & {
  '--card-back'?: string; // 커스텀 속성 이름을 직접 타입에 추가
};

const style: CardStyle = { '--card-back': cardBack }; // as 없이도 통과됨
```

```txt
CSSProperties & { ... } — 교차 타입(&)으로 기존 CSSProperties에 내가 쓸 커스텀 속성을 추가
이렇게 한 번 타입을 만들어두면, 그 컴포넌트가 쓰는 커스텀 속성이 늘어나도
as를 매번 새로 쓰는 대신 이 타입 하나에 계속 추가하면 됨 — as보다 한 단계 더 안전한 방법
```

---

# 한눈에

| 개념                         | 핵심                                                                    |
| -------------------------- | --------------------------------------------------------------------- |
| `CSSProperties`            | React `style` prop 객체의 타입 — 표준 CSS 속성만 알고 있음                          |
| CSS 커스텀 속성(`--이름`)         | 직접 정의하는 CSS 변수, `var(--이름)`으로 어디서든 사용                                 |
| React에서 동적 주입              | `style={{ '--card-back': value } as CSSProperties}`로 JS 값을 CSS 변수에 전달 |
| `as CSSProperties`가 필요한 이유 | 커스텀 속성 이름이 표준 타입 목록에 없어서 강제 단언 필요                                     |
| 더 안전한 대안                   | `CSSProperties & { '--이름'?: string }`로 커스텀 속성을 타입에 직접 포함              |