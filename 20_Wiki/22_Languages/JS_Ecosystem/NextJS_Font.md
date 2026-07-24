---
aliases:
  - Font
  - GoogleFonts
  - CSS
tags:
  - NextJS
related:
  - "[[00_JS_Ecosystem_HomePage]]"
  - "[[NextJS_Concept]]"
  - "[[React_Styling]]"
---
# NextJS_Font — 폰트 관리

> [!info]
>  Next.js에서 폰트를 쓰는 방법은 두 가지 — `next/font`(자동 최적화)와 CSS `@import`(직접 로드). 
>  한글 폰트처럼 라틴 서브셋만으로는 안 되는 경우 `globals.css @import`를 쓴다.

---

# next/font vs CSS @import ⭐️⭐️⭐️⭐️

| |`next/font`|CSS `@import`|
|---|---|---|
|설정 위치|`layout.tsx` 또는 컴포넌트|`globals.css`|
|최적화|✅ 자동 (preload, swap)|❌ 직접 설정 필요|
|번들 방식|서버에서 처리 → 외부 요청 없음|브라우저가 Google Fonts 직접 요청|
|한글 지원|⚠️ Latin subset 기본 — 한글 글리프 누락|✅ 전체 글리프 포함 가능|
|언제|영문 폰트, 서브셋 지원 폰트|한글 폰트, 전체 글리프가 필요한 폰트|

```txt
next/font가 한글 폰트에 부적합한 이유:
  next/font는 Google Fonts에서 폰트를 받아올 때 latin subset을 기본으로 사용
  한글 글리프는 latin subset에 없음 → 한글이 표시 안 되거나 폴백 폰트로 대체됨

  Nanum Pen Script 같은 한글 손글씨 폰트는 subset 지정이 복잡
  → globals.css에 @import로 직접 로드하는 것이 더 안전
```

---

# next/font — 영문 폰트 (권장) ⭐️⭐️⭐️

```typescript
// app/layout.tsx
import { Inter, Playfair_Display } from 'next/font/google';

const inter = Inter({
  subsets: ['latin'],
  variable: '--font-inter',   // CSS 변수로 등록
});

const playfair = Playfair_Display({
  subsets: ['latin'],
  variable: '--font-playfair',
});

export default function RootLayout({ children }) {
  return (
    <html className={`${inter.variable} ${playfair.variable}`}>
      <body>{children}</body>
    </html>
  );
}
```

```css
/* globals.css */
body {
  font-family: var(--font-inter), sans-serif;
}
```

```txt
next/font 장점:
  빌드 타임에 폰트 다운로드 → CDN에 올림
  font-display: swap 자동 적용 → 폴백 폰트 → 웹폰트 전환
  외부 Google Fonts 요청 없음 → 성능 + 프라이버시
  variable로 CSS 변수 등록 → Tailwind의 fontFamily와 연결 가능
```

---

# CSS @import — 한글 폰트 ⭐️⭐️⭐️⭐️

```css
/* app/globals.css */
@import url('https://fonts.googleapis.com/css2?family=Nanum+Pen+Script&display=swap');
```

```txt
display=swap:
  폰트 로딩 전에 폴백 폰트(시스템 폰트)를 먼저 보여줌
  폰트가 로드되면 교체 — 레이아웃 시프트는 있지만 텍스트가 숨지 않음

@import 위치:
  globals.css 최상단에 두어야 함
  다른 CSS 규칙 아래에 있으면 일부 브라우저에서 무시됨
```

---

# 커스텀 폰트 클래스 패턴 ⭐️⭐️⭐️⭐️

```css
/* globals.css */
@import url('https://fonts.googleapis.com/css2?family=Nanum+Pen+Script&display=swap');

.napkin-hand {
  font-family: 'Nanum Pen Script', 'Apple SD Gothic Neo', cursive;
  font-weight: 400;
  letter-spacing: 0.01em;
}
```

```typescript
//web/lib/fontname.ts
// 클래스 이름을 상수로 관리 — 오타 방지, 변경 시 한 곳만 수정
export const napkinHandClassName = 'napkin-hand';

// 커스텀 클래스 + Tailwind 조합
export const napkinMoodInputClassName = [
  napkinHandClassName,
  'w-[9.5rem]',
  'border-0 border-b border-dashed border-[#c9a66b]/40',
  'bg-transparent px-0.5 py-1',
  'text-[1.2rem] leading-none text-[#c9a66b]',
  'outline-none',
  'placeholder:text-[#a89880]/45',
  'focus:border-[#c9a66b]',
].join(' ');

// 사용
<input className={napkinMoodInputClassName} />
```

```txt
클래스 이름을 상수로 분리하는 이유:
  'napkin-hand' 문자열을 여러 컴포넌트에 직접 쓰면 오타 발생 가능
  napkinHandClassName 상수로 관리하면 IDE 자동완성 + 변경 시 한 곳만 수정

커스텀 클래스 + Tailwind 조합:
  .napkin-hand       → globals.css에 font-family 등 폰트 속성
  Tailwind 클래스    → 크기, 여백, 색상 등 나머지 스타일
  분리하는 이유: font-family 폴백 스택은 Tailwind arbitrary value로 표현하기 어려움
```

## 폰트 스택 (fallback) 설계

```css
.napkin-hand {
  font-family:
    'Nanum Pen Script',     /* 1순위: 웹폰트 */
    'Apple SD Gothic Neo',  /* 2순위: Mac/iOS 시스템 한글 폰트 */
    cursive;                /* 3순위: 시스템 손글씨 계열 (최후 폴백) */
}
```

```txt
폰트 스택이 필요한 이유:
  웹폰트가 아직 로드 안 됐거나 실패했을 때 대체 폰트가 없으면
  브라우저 기본 폰트(보통 맑은 고딕/Apple SD Gothic)로 표시됨
  → 손글씨 느낌이 전혀 없는 폰트로 나옴

  'Apple SD Gothic Neo' 를 중간에 두는 이유:
  cursive(시스템 손글씨)보다 한글이 더 자연스럽게 나오는 경우가 있어서

자주 쓰는 한글 시스템 폰트:
  'Apple SD Gothic Neo'   Mac/iOS
  'Noto Sans KR'          Android/Web (설치 시)
  '맑은 고딕'             Windows
  sans-serif              전체 폴백
```

---

# Tailwind에 커스텀 폰트 등록 ⭐️⭐️

```javascript
// tailwind.config.ts
export default {
  theme: {
    extend: {
      fontFamily: {
        hand: ['var(--font-napkin)', 'Apple SD Gothic Neo', 'cursive'],
        // → className="font-hand"로 사용 가능
      },
    },
  },
};
```

```css
/* globals.css */
:root {
  --font-napkin: 'Nanum Pen Script';
}
```

```txt
Tailwind에 등록하면:
  className="font-hand" 로 간결하게 사용
  arbitrary value [font-family:...] 보다 읽기 쉬움

next/font variable과 연결:
  const nanum = localFont({ src: './NanumPenScript.woff2', variable: '--font-napkin' });
  → Tailwind의 var(--font-napkin)과 자동 연결
```

---

# 한눈에

```txt
영문 폰트  → next/font/google (자동 최적화, Latin subset)
한글 폰트  → globals.css @import (전체 글리프)

@import:
  globals.css 최상단
  url('...?display=swap')  로딩 전 폴백 폰트 표시

커스텀 클래스 패턴:
  globals.css → .napkin-hand { font-family: '웹폰트', '시스템폰트', cursive; }
  TS 상수     → export const napkinHandClassName = 'napkin-hand'
  조합        → [napkinHandClassName, ...tailwindClasses].join(' ')

폰트 스택:
  웹폰트 → 시스템 한글 폰트 → cursive (최후 폴백)

Tailwind 등록:
  tailwind.config.ts fontFamily → className="font-hand"
```