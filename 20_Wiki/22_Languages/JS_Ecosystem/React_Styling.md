---
aliases: [CSS, DesignToken, globals.css, JSDoc 주석, React, Tailwind]
tags: [React, CSS, Tailwind]
related:
  - "[[00_JS_Ecosystem_HomePage]]"
  - "[[JS_BrowserAPI]]"
  - "[[JS_DOM]]"
  - "[[React_CSSProperties]]"
---
# React_Styling — 스타일링 전략

> [!info] 
> Next.js + Tailwind 프로젝트에서 스타일을 관리하는 두 축 
>  `globals.css`(커스텀 클래스 · CSS 변수 · 전역 초기화)와 클래스명 상수 파일(오타 방지 · 리팩토링 · 의도 문서화)
>  Tailwind로 못 하는 것은 globals.css에, 단순 유틸리티는 Tailwind에 역할을 나눈다.

---

# globals.css 기본 구조 ⭐️⭐️⭐️⭐️

```css
/* app/globals.css */

/* 1. Tailwind 세 레이어 — 반드시 이 순서로 */
@tailwind base;
@tailwind components;
@tailwind utilities;

/* 2. CSS 변수 (디자인 토큰) */
/* 3. 전역 초기화 */
/* 4. 커스텀 컴포넌트 클래스 */
/* 5. 애니메이션 */
```

## @layer — 우선순위 제어 ⭐️⭐️⭐️

```css
/* @layer 없이 작성하면 → 파일 순서에 따라 우선순위 결정 (예측하기 어려움) */
/* @layer로 감싸면 → 레이어 순서(base < components < utilities)로 우선순위 고정 */

@layer base {
  /* HTML 기본 태그 스타일 초기화 · 전역 기본값 */
  *, *::before, *::after { box-sizing: border-box; }
  body { font-family: var(--font-sans); }
  a { color: inherit; text-decoration: none; }
}

@layer components {
  /* 재사용 가능한 컴포넌트 클래스 — Tailwind utilities보다 낮은 우선순위 */
  .post-card-shell { ... }
  .play-button { ... }
}

@layer utilities {
  /* 커스텀 유틸리티 — Tailwind utilities와 같은 레벨 */
  .text-balance { text-wrap: balance; }
}
```

```txt
레이어 우선순위: base < components < utilities
→ utilities(Tailwind bg-red-500 등)가 components(.post-card-shell)보다 항상 이김
→ JSX에서 className="post-card-shell bg-red-500"처럼 쓸 때
  bg-red-500이 post-card-shell 안의 background를 오버라이드할 수 있음

@layer components 안에 넣어야 하는 것:
  커스텀 컴포넌트 클래스 — Tailwind 유틸리티로 덮어쓸 수 있어야 할 때

@layer utilities 안에 넣어야 하는 것:
  Tailwind처럼 단일 속성 유틸리티 — 오히려 남들한테 안 덮어써지길 원할 때
```

---

# CSS 변수 — 디자인 토큰 ⭐️⭐️⭐️⭐️

```css
/* globals.css */

:root {
  /* 브랜드 컬러 */
  --color-primary:      #335b73;
  --color-primary-soft: #e4eff5;
  --color-primary-muted: color-mix(in srgb, #335b73 70%, transparent);

  /* 타이포그래피 */
  --font-sans: 'Pretendard', 'Inter', system-ui, sans-serif;
  --font-mono: 'JetBrains Mono', monospace;

  /* 간격/크기 */
  --radius-card: 1rem;
  --radius-btn:  9999px;

  /* 그림자 */
  --shadow-card: 0 2px 8px rgb(0 0 0 / 0.08), 0 0 0 1px rgb(0 0 0 / 0.04);
  --shadow-card-hover: 0 8px 24px rgb(0 0 0 / 0.12), 0 0 0 1px rgb(0 0 0 / 0.06);
}
```

```css
/* 다크모드 — classList.toggle('dark') 또는 data-theme 패턴 */
/* ([[JS_BrowserAPI]] applyTheme 함수 참고) */

/* 방법 1: .dark 클래스 기반 */
.dark {
  --color-primary:      #7ab3d0;
  --color-primary-soft: #1a2f3d;
}

/* 방법 2: data-theme 속성 기반 */
[data-theme="dark"] {
  --color-primary:      #7ab3d0;
  --color-primary-soft: #1a2f3d;
}
```

```typescript
// Tailwind에서 CSS 변수 사용 — tailwind.config.ts
export default {
  theme: {
    extend: {
      colors: {
        primary:      'var(--color-primary)',
        'primary-soft': 'var(--color-primary-soft)',
      },
      borderRadius: {
        card: 'var(--radius-card)',
        btn:  'var(--radius-btn)',
      },
    },
  },
};

// JSX에서: className="bg-primary-soft text-primary rounded-card"
```

```txt
CSS 변수의 장점:
  JS에서도 접근 가능: getComputedStyle(el).getPropertyValue('--color-primary')
  다크모드 전환 시 변수만 바꾸면 → 그 변수를 쓰는 모든 곳이 자동으로 바뀜
  Tailwind config에 연결하면 유틸리티 클래스로도 사용 가능
```

---

# 커스텀 클래스 — globals.css에 정의 ⭐️⭐️⭐️⭐️

```txt
Tailwind로 표현하기 어려운 것들을 globals.css에 클래스로 정의:
  - 여러 레이어 box-shadow
  - :hover/:active와 조합된 transition
  - 복잡한 그라디언트
  - 의사 요소(::before, ::after)
  - 스크롤바 스타일
```

```css
@layer components {

  /* 카드 계층 — 입체감 3단계 */
  .post-card-shell {
    border-radius: var(--radius-card);
    box-shadow: var(--shadow-card);
    transition: box-shadow 0.2s ease, transform 0.2s ease;
    overflow: hidden;
  }

  .post-card-shell:hover {
    box-shadow: var(--shadow-card-hover);
    transform: translateY(-2px);
  }

  /* 재생 버튼 — 브랜드 그라디언트 */
  .play-button {
    background: linear-gradient(135deg, var(--color-primary) 0%, #5a8fa8 100%);
    border-radius: 50%;
    transition: opacity 0.15s ease, transform 0.15s ease;
  }

  .play-button:hover  { opacity: 0.9; transform: scale(1.05); }
  .play-button:active { opacity: 0.8; transform: scale(0.97); }

  /* 브랜드 필 버튼 */
  .brand-pill-btn {
    border-radius: var(--radius-btn);
    background-color: var(--color-primary);
    color: white;
    padding: 0.5rem 1.25rem;
    font-weight: 600;
    transition: background-color 0.15s ease;
  }

  .brand-pill-btn:hover { background-color: color-mix(in srgb, var(--color-primary) 85%, black); }

}
```

## 스크롤바 스타일링

```css
/* @layer 밖에 — 브라우저 전용 선택자라 레이어 관리 불필요 */

/* Chrome / Safari / Edge */
::-webkit-scrollbar       { width: 6px; height: 6px; }
::-webkit-scrollbar-track { background: transparent; }
::-webkit-scrollbar-thumb {
  background: rgb(0 0 0 / 0.2);
  border-radius: 9999px;
}
::-webkit-scrollbar-thumb:hover { background: rgb(0 0 0 / 0.35); }

/* Firefox */
* { scrollbar-width: thin; scrollbar-color: rgb(0 0 0 / 0.2) transparent; }
```

---

# @keyframes — 애니메이션 ⭐️⭐️

```css
/* globals.css */

@keyframes fade-in {
  from { opacity: 0; transform: translateY(8px); }
  to   { opacity: 1; transform: translateY(0); }
}

@keyframes spin {
  to { transform: rotate(360deg); }
}

@keyframes shimmer {
  from { background-position: -200% 0; }
  to   { background-position: 200% 0; }
}
```

```css
@layer components {
  /* 애니메이션 클래스 */
  .animate-fade-in {
    animation: fade-in 0.25s ease both;
  }

  /* 스켈레톤 로딩 */
  .skeleton {
    background: linear-gradient(90deg, #f0f0f0 25%, #e0e0e0 50%, #f0f0f0 75%);
    background-size: 200% 100%;
    animation: shimmer 1.5s infinite;
    border-radius: 0.25rem;
  }
}
```

```typescript
// Tailwind에 @keyframes 연결 — tailwind.config.ts
export default {
  theme: {
    extend: {
      animation: {
        'fade-in': 'fade-in 0.25s ease both',
      },
      keyframes: {
        'fade-in': {
          from: { opacity: '0', transform: 'translateY(8px)' },
          to:   { opacity: '1', transform: 'translateY(0)' },
        },
      },
    },
  },
};
// → className="animate-fade-in" 으로 사용 가능
```

---

# 클래스명 상수 관리 ⭐️⭐️⭐️⭐️

```txt
globals.css에서 정의한 클래스명을 문자열로 JSX에 직접 쓰면:
  오타 위험 / 리팩토링 어려움 / 의도 파악 어려움

→ 상수 파일로 추출해서 import해서 사용
```

## 세 가지 형태

```typescript
// styles/classNames.ts

// 형태 1: 단순 클래스명 상수
/** 브랜드 입체감 — 3단계 (카드 쉘 · 트랙 · 재생 버튼) · 클래스는 globals.css */
export const postCardShell = 'post-card-shell';
export const postCard      = 'post-card';
export const playButton    = 'play-button';
export const brandPillBtn  = 'brand-pill-btn';

// 형태 2: 복합 클래스 (여러 클래스 조합)
export const savedCardFace   = 'saved-card-face';
export const savedCardFaceLg = 'saved-card-face saved-card-face-lg';
// → savedCardFaceLg를 쓰면 두 클래스를 한 번에 적용

// 형태 3: 객체 — 테마/변형을 한 묶음으로
/** 태그 없을 때 헤더 — primary-soft */
export const neoHeaderDefault = {
  band:     'bg-[#e4eff5]',          // Tailwind arbitrary value
  text:     'text-[#335b73]',
  muted:    'text-[#335b73]/70',
  cardBack: '#d8e5ee',               // 색상값 (인라인 스타일용, 클래스 아님)
};
```

```txt
/** JSDoc 주석 */의 실용적 가치:
  VSCode에서 상수에 hover하면 주석 내용이 팝업으로 보임
  "이 클래스가 왜 있는지, 어디서 정의됐는지" 파일 이동 없이 바로 확인
```

## JSX에서 사용

```tsx
import { postCardShell, playButton, neoHeaderDefault } from '@/styles/classNames';

// 단독 사용
<div className={postCardShell}>

// Tailwind와 혼합
<div className={`${postCardShell} mt-4 cursor-pointer`}>

// clsx / cn과 조합 (조건부 클래스)
<div className={cn(postCardShell, isActive && 'ring-2 ring-primary')}>

// 객체 테마
<header className={neoHeaderDefault.band}>
  <h1 className={neoHeaderDefault.text}>제목</h1>
  <div style={{ backgroundColor: neoHeaderDefault.cardBack }} />
</header>
```

---

# globals.css vs Tailwind — 언제 어느 쪽 ⭐️⭐️⭐️⭐️

|상황|globals.css|Tailwind|
|---|---|---|
|복잡한 box-shadow 레이어|✅|어렵 (arbitrary value로 가능하지만 길어짐)|
|:hover/:active + transition 조합|✅|hover: 접두사로 가능하지만 JSX가 길어짐|
|::before, ::after 의사 요소|✅|before: / after: 접두사로 가능|
|@keyframes 애니메이션|✅|config 연결 후 animate- 클래스로|
|스크롤바 스타일|✅|불가|
|단순 색상 / 간격 / 크기|—|✅ `bg-[#e4eff5]` `mt-4` `rounded-lg`|
|다크모드 단순 변형|—|✅ `dark:bg-gray-900`|
|CSS 변수 정의|✅ (`:root` 에서)|tailwind.config에 연결 후 사용|

```txt
판단 기준:
  "이 스타일을 Tailwind로 쓰면 className이 너무 길어지거나 JSX가 지저분해지는가?"
  → Yes → globals.css에 클래스로 정의
  → No  → Tailwind arbitrary value나 유틸리티로 바로 작성

기본 규칙:
  컴포넌트 정체성을 가진 스타일 (카드, 버튼 등 반복 사용) → globals.css 클래스
  일회성 조정 (mt-4, text-sm 등) → Tailwind inline
```

---

# 한눈에

```txt
globals.css 구조:
  @tailwind base / components / utilities  (이 순서 필수)
  :root { --css-변수 }                    디자인 토큰
  @layer components { .클래스명 }         커스텀 컴포넌트 클래스
  @keyframes                              애니메이션 정의
  ::-webkit-scrollbar                     스크롤바

CSS 변수:
  :root에서 선언 → .dark / [data-theme="dark"]에서 덮어쓰기
  tailwind.config에 연결 → Tailwind 유틸리티로도 사용 가능

클래스명 상수:
  단순: export const name = 'css-class'
  복합: export const name = 'class1 class2'
  객체: export const theme = { band: 'bg-...', cardBack: '#hex' }
  /** JSDoc */ → hover 시 VSCode 팝업으로 의도 확인

globals.css vs Tailwind:
  복잡한 shadow·transition·의사요소·스크롤바 → globals.css
  단순 색상·간격·크기 → Tailwind
  반복 사용 컴포넌트 스타일 → globals.css 클래스 + 상수 파일
```