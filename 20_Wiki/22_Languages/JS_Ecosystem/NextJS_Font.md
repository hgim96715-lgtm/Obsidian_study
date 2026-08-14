---
aliases:
  - Font
  - GoogleFonts
  - CSS
  - localFont
  - "@import"
tags:
  - NextJS
related:
  - "[[00_JS_Ecosystem_HomePage]]"
  - "[[NextJS_Concept]]"
  - "[[React_Styling]]"
---
# NextJS_Font — 폰트 설정

> [!info] 
> `next/font/google`으로 Google Fonts를 번들에 포함 — 외부 요청 없이 자체 서빙.
>  폰트를 CSS 변수(`--font-xxx`)로 등록하고 `body`나 특정 요소에 적용.
>   `tailwind.config`에 등록하면 클래스로 사용 가능.

---

# 왜 next/font인가 ⭐️⭐️⭐️

```txt
일반적인 Google Fonts 사용:
  <link href="https://fonts.googleapis.com/..." rel="stylesheet">
  → 페이지 로드 시 외부 서버에 요청 → 느림, 개인정보 문제

next/font/google:
  빌드 시 폰트 파일을 다운로드 → Next.js 서버에서 자체 서빙
  외부 네트워크 요청 없음 → 빠름, GDPR 문제 없음
  layout shift 방지 (폰트 로드 전/후 레이아웃 틀어짐 없음)
```

---

# 사용 순서 ⭐️⭐️⭐️⭐️

## 1. 폰트 이름 찾기

```txt
Google Fonts (fonts.google.com) 에서 원하는 폰트 검색
Next.js 폰트 이름 = Google Fonts 이름을 CamelCase + 언더스코어로 변환

Nanum Pen Script → Nanum_Pen_Script
Noto Sans KR     → Noto_Sans_KR
Roboto           → Roboto
Inter            → Inter
```

## 2. 폰트 설정 — CSS 변수 등록

```typescript
// app/layout.tsx 또는 lib/fonts.ts
import { Nanum_Pen_Script, Noto_Sans_KR } from 'next/font/google';

const nanumPen = Nanum_Pen_Script({
  variable: '--font-nanum-pen',  // CSS 변수 이름 (내가 정함)
  subsets:  ['latin'],           // 사용할 문자 범위
  weight:   '400',               // 폰트 굵기 (string 또는 배열)
});

const notoSansKR = Noto_Sans_KR({
  variable: '--font-noto-sans-kr',
  subsets:  ['latin'],
  weight:   ['400', '700'],      // 여러 굵기를 배열로
});
```

```txt
variable: '--font-xxx':
  CSS 커스텀 속성(변수) 이름 — 내가 원하는 이름으로 지정
  이후 CSS에서 font-family: var(--font-nanum-pen) 로 사용

subsets:
  'latin'     = 영문 기본 (용량 최소)
  'latin-ext' = 영문 확장
  한국어 폰트는 대부분 subset 없이 전체 사용 (용량 큼)

weight:
  고정 굵기 폰트: weight 지정 필수 ('400', '700' 등)
  가변 폰트(variable font): 생략 가능 (모든 굵기 포함)
  배열로 여러 굵기 동시 로드 가능
```

## 3. body 또는 html에 className 등록

```typescript
// app/layout.tsx
export default function RootLayout({ children }: { children: React.ReactNode }) {
  return (
    <html lang="ko" className={`${nanumPen.variable} ${notoSansKR.variable}`}>
    //               ↑ html에 붙이는 방식도 자주 씀
    //                 body에 붙이는 것과 차이:
    //                 html에 붙이면 :root CSS 변수처럼 최상위에서 접근 가능
      <body>
        {children}
      </body>
    </html>
  );
}

// body에 붙이는 방식
<body className={nanumPen.variable}>

// 둘 다 붙이는 방식 (공식 Next.js 예제)
<html className={nanumPen.variable}>
  <body className="font-sans">  {/* Tailwind 기본 폰트 클래스 */}
```

```txt
html vs body 어디에:
  html에 붙이기 → CSS에서 :root와 비슷하게 최상위 변수
  body에 붙이기 → body 하위 요소에서 접근 가능
  결과는 거의 같음 — 공식 Next.js 문서는 html에 붙임
```

---

# ⚠️ 한국어 폰트와 next/font — 주의사항 ⭐️⭐️⭐️⭐️

```txt
next/font의 구조:
  Google Fonts에서 폰트를 받아올 때 subset 단위로 다운로드
  'latin' subset = 영문·숫자·기본 기호만 포함
  한글 글리프는 latin subset에 없음

한글 폰트(Nanum Pen Script 등)를 next/font로 쓸 때 문제:
  subsets: ['latin']  → 한글 글자가 포함된 글리프 없음
                         → 한글이 표시 안 되거나 폴백 폰트로 대체됨

  한글 폰트는 subset 구성이 복잡
  (CJK 글리프 = 수만 개 → 파일이 매우 크고 subset 지정 어려움)
```

## 해결 — globals.css에서 직접 import ⭐️⭐️⭐️

```css
/* globals.css */
/* 한글 폰트는 Google Fonts @import로 직접 로드 */
@import url('https://fonts.googleapis.com/css2?family=Nanum+Pen+Script&display=swap');

/* 사용 */
.handwriting {
  font-family: 'Nanum Pen Script', cursive;
}
```

```txt
next/font 대신 @import 장점:
  한글 글리프 전체 포함 (subset 문제 없음)
  설정이 단순

단점:
  외부 요청 발생 (Google 서버에 의존)
  next/font처럼 자체 서빙 아님

절충안:
  영문 폰트 → next/font (latin subset 문제 없음)
  한글 폰트 → globals.css @import 또는 localFont
```

## localFont — 직접 다운받아 자체 서빙 ⭐️⭐️⭐️

```typescript
// 폰트 파일을 직접 다운받아 /public/fonts/ 또는 /app/fonts/에 두기
import localFont from 'next/font/local';

const nanumPen = localFont({
  src: './fonts/NanumPenScript-Regular.ttf',
  variable: '--font-nanum-pen',
});
```

```txt
localFont가 한글 폰트에 적합한 이유:
  파일 전체를 그대로 사용 → subset 문제 없음
  자체 서빙 → 외부 요청 없음
  Next.js가 자동으로 최적화 (woff2 변환 등)

폰트 파일 얻는 방법:
  Google Fonts → 폰트 선택 → Download family
  .ttf 또는 .otf 파일을 /public/fonts/ 또는 /app/fonts/ 에 복사
```

## 4. CSS에서 사용

```css
/* globals.css */

/* 전체 앱 기본 폰트 */
body {
  font-family: var(--font-noto-sans-kr), sans-serif;
}

/* 특정 요소에만 */
.handwriting {
  font-family: var(--font-nanum-pen), cursive;
}

.title {
  font-family: var(--font-nanum-pen);
  font-size: 2rem;
}
```

---

# Tailwind CSS와 함께 ⭐️⭐️⭐️

```typescript
// tailwind.config.ts
import type { Config } from 'tailwindcss';

export default {
  theme: {
    extend: {
      fontFamily: {
        'nanum-pen': ['var(--font-nanum-pen)', 'cursive'],
        'noto-kr':   ['var(--font-noto-sans-kr)', 'sans-serif'],
      },
    },
  },
} satisfies Config;
```

```tsx
// 클래스로 사용
<h1 className="font-nanum-pen text-3xl">손글씨 제목</h1>
<p className="font-noto-kr">본문 텍스트</p>
```

---

# 전체 예시 — Nanum Pen Script

```typescript
// app/layout.tsx
import { Nanum_Pen_Script } from 'next/font/google';

const nanumPen = Nanum_Pen_Script({
  variable: '--font-nanum-pen',
  subsets:  ['latin'],
  weight:   '400',
  // display: 'swap',  // 폰트 로드 전 대체 폰트 표시 (기본값)
});

export default function RootLayout({ children }: { children: React.ReactNode }) {
  return (
    <html lang="ko">
      <body className={nanumPen.variable}>
        {children}
      </body>
    </html>
  );
}
```

```css
/* globals.css */
.pen-font {
  font-family: var(--font-nanum-pen), cursive;
}
```

```tsx
/* 사용 */
<p className="pen-font">손글씨처럼 보이는 텍스트</p>
```

---

# 자주 쓰는 한국어 폰트

```typescript
import {
  Noto_Sans_KR,          // 본고딕 계열 — 가독성 좋음
  Nanum_Gothic,          // 나눔고딕
  Nanum_Myeongjo,        // 나눔명조 (serif)
  Nanum_Pen_Script,      // 손글씨
  Gowun_Dodum,           // 고운돋움 — 부드러운 느낌
  IBM_Plex_Sans_KR,      // IBM Plex 한국어
} from 'next/font/google';
```

```txt
한국어 폰트 주의사항:
  한국어는 글자 수가 많아서 폰트 파일이 큼 (수 MB)
  subset 지정으로 용량 줄이기 어려움
  display: 'swap' (기본값) — 폰트 로드 전 시스템 폰트 먼저 표시
  
  wght (가변 폰트) 지원 확인:
    Noto Sans KR — weight 배열로 여러 굵기
    Nanum Gothic — 고정 굵기 지정 필요
```

---

# 로컬 폰트

```typescript
// 직접 다운받은 폰트 파일 사용
import localFont from 'next/font/local';

const myFont = localFont({
  src: './fonts/MyFont.woff2',  // public/ 기준 경로
  variable: '--font-my',
});

// 여러 파일 (굵기별)
const myFont = localFont({
  src: [
    { path: './fonts/MyFont-Regular.woff2', weight: '400' },
    { path: './fonts/MyFont-Bold.woff2',    weight: '700' },
  ],
  variable: '--font-my',
});
```