---
aliases:
  - Tailwind
  - CSS Modules
  - 스타일링
  - lucide-react
tags:
  - Tailwind
related:
  - "[[00_JS_Ecosystem_HomePage]]"
  - "[[React_Component]]"
  - "[[NextJS_Routing]]"
---

# React_TailwindCSS — Tailwind CSS

```
Tailwind CSS = 유틸리티 클래스 기반 CSS 프레임워크
미리 정의된 클래스를 조합해서 스타일 적용
CSS 파일 따로 안 써도 됨
```

---

---

# 왜 쓰나 ⭐️

```
기존 CSS:
  .card { padding: 16px; background: white; border-radius: 8px; }
  .card-title { font-size: 18px; font-weight: bold; }
  → 클래스 이름 고민 / CSS 파일 왔다갔다

Tailwind:
  <div className="p-4 bg-white rounded-lg">
    <h2 className="text-lg font-bold">제목</h2>
  </div>
  → 클래스 이름 없이 JSX 안에서 바로 스타일 적용
  → CSS 파일 거의 안 씀

장점:
  클래스 이름 고민 불필요
  컴포넌트 안에서 스타일 바로 확인
  빌드 시 사용한 클래스만 포함 → 번들 작음
```

---

---

# 설치 (Next.js)

```bash
# create-next-app 시 Tailwind 선택하면 자동 설정
pnpm create next-app@latest my-app
# Tailwind CSS? → Yes

# 기존 프로젝트에 추가
pnpm add -D tailwindcss postcss autoprefixer
npx tailwindcss init -p
```

```css
/* app/globals.css */
@tailwind base;
@tailwind components;
@tailwind utilities;
```

---

---

# 자주 쓰는 유틸리티 클래스 ⭐️

## 여백

```
p-4    padding: 16px (전체)
px-4   padding 좌우
py-4   padding 상하
pt-4   padding 위
pb-4   padding 아래

m-4    margin: 16px (전체)
mx-auto margin 좌우 auto (가운데 정렬)
mt-4   margin 위

단위:
  1 = 4px / 2 = 8px / 4 = 16px / 6 = 24px / 8 = 32px
```

## 크기

```
w-full      width: 100%
w-1/2       width: 50%
w-[200px]   width: 200px  (임의값)
h-screen    height: 100vh
h-full      height: 100%
max-w-xl    max-width: 576px
```

## 텍스트

```
text-sm     font-size: 14px
text-base   font-size: 16px
text-lg     font-size: 18px
text-xl     font-size: 20px
text-2xl    font-size: 24px

font-normal  font-weight: 400
font-medium  font-weight: 500
font-bold    font-weight: 700

text-gray-500   color: gray
text-center     text-align: center
truncate        말줄임 (overflow hidden + ellipsis)
```

## 배경 & 테두리

```
bg-white        background: white
bg-gray-100     background: 연한 회색
bg-blue-500     background: 파란색

rounded         border-radius: 4px
rounded-lg      border-radius: 8px
rounded-full    border-radius: 9999px (원형)

border          border: 1px solid
border-gray-200 border-color: 연한 회색
```

## Flexbox

```
flex            display: flex
items-center    align-items: center
justify-center  justify-content: center
justify-between justify-content: space-between
gap-4           gap: 16px
flex-col        flex-direction: column
flex-wrap       flex-wrap: wrap
```

## Grid

```
grid            display: grid
grid-cols-2     grid-template-columns: repeat(2, 1fr)
grid-cols-3     3열
gap-4           gap: 16px
col-span-2      grid-column: span 2
```

---

---

# 반응형 ⭐️

```
접두사 없음   모바일 기본 (모든 크기)
sm:           640px 이상
md:           768px 이상
lg:           1024px 이상
xl:           1280px 이상
2xl:          1536px 이상

모바일 퍼스트:
  작은 화면 → 기본값
  큰 화면   → 접두사로 덮어쓰기
```

```tsx
// 모바일: 1열 / 태블릿: 2열 / 데스크탑: 3열
<div className="grid grid-cols-1 md:grid-cols-2 lg:grid-cols-3 gap-4">
  {items.map(item => <Card key={item.id} {...item} />)}
</div>

// 모바일: 숨김 / 데스크탑: 표시
<nav className="hidden lg:flex">...</nav>

// 모바일: 작은 텍스트 / 데스크탑: 큰 텍스트
<h1 className="text-xl lg:text-4xl font-bold">제목</h1>
```

---

---

# 상태 변형 ⭐️

```
hover:   마우스 올렸을 때
focus:   포커스됐을 때
active:  클릭 중
disabled: 비활성화
```

```tsx
<button
  className="
    bg-blue-500 text-white px-4 py-2 rounded
    hover:bg-blue-600
    active:bg-blue-700
    disabled:bg-gray-300 disabled:cursor-not-allowed
  "
>
  저장
</button>

<input
  className="
    border border-gray-300 rounded px-3 py-2
    focus:outline-none focus:ring-2 focus:ring-blue-500
  "
/>
```

---

---

# 다크모드 ⭐️

```
dark: 접두사 = 다크모드일 때 적용
```

```tsx
// tailwind.config.ts
module.exports = {
  darkMode: 'class',  // 'class' = 수동 / 'media' = 시스템 설정 자동
}

// 루트에 dark 클래스 추가 시 다크모드 적용
<html className="dark">

// 컴포넌트
<div className="bg-white dark:bg-gray-900 text-black dark:text-white">
  <h1 className="text-gray-900 dark:text-gray-100">제목</h1>
</div>
```

---

---

# 조건부 클래스 ⭐️

```tsx
// 삼항 연산자
<div className={`p-4 rounded ${isActive ? 'bg-blue-500 text-white' : 'bg-gray-100'}`}>

// clsx / cn 유틸리티 (더 깔끔)
import { clsx } from 'clsx';

<div className={clsx(
  'p-4 rounded',
  isActive && 'bg-blue-500 text-white',
  !isActive && 'bg-gray-100',
  hasError && 'border border-red-500',
)}>

// shadcn/ui 의 cn 함수 (tailwind-merge + clsx)
import { cn } from '@/lib/utils';

<div className={cn(
  'p-4 rounded',
  isActive ? 'bg-blue-500' : 'bg-gray-100',
)}>
```

```
문자열 템플릿 리터럴로 조건부 클래스:
  공백 실수 / 클래스 충돌 위험 있음

clsx / cn 권장:
  가독성 좋음
  undefined / false 값 자동 무시
  tailwind-merge 로 중복 클래스 충돌 방지
```
---
---
# 아이콘 — lucide-react ⭐️

```
lucide-react = Tailwind 와 함께 가장 많이 쓰는 아이콘 라이브러리
SVG 아이콘을 React 컴포넌트로 제공
className 으로 Tailwind 클래스 바로 적용 가능
```

```bash
pnpm add lucide-react
```

```tsx
import { ArrowRight, Search, X, ChevronDown, Heart } from 'lucide-react';

// 기본 사용
<ArrowRight />

// Tailwind 로 크기 / 색상 조절
<ArrowRight className="w-4 h-4" />            // 16px
<ArrowRight className="w-5 h-5 text-blue-500" /> // 20px + 파란색
<ArrowRight className="w-6 h-6 text-gray-400" /> // 24px + 회색

// 버튼 안에 아이콘
<button className="flex items-center gap-2">
  더보기 <ArrowRight className="w-4 h-4" />
</button>
```

```
크기 기준:
  w-4 h-4   16px  (작은 인라인 아이콘)
  w-5 h-5   20px  (기본)
  w-6 h-6   24px  (버튼 아이콘)
  w-8 h-8   32px  (큰 아이콘)

아이콘 목록:
  https://lucide.dev/icons 에서 검색
  컴포넌트 이름은 PascalCase
  예: arrow-right → ArrowRight / chevron-down → ChevronDown
```

---

---

# 실전 카드 컴포넌트 예시

```tsx
type ExhibitionCardProps = {
  title:     string;
  dateRange: string;
  status:    string;
  statusColor: string;
};

function ExhibitionCard({ title, dateRange, status, statusColor }: ExhibitionCardProps) {
  return (
    <div className="bg-white rounded-xl shadow-sm border border-gray-100 p-5 hover:shadow-md transition-shadow">
      <div className="flex items-start justify-between gap-2">
        <h2 className="text-base font-semibold text-gray-900 line-clamp-2">
          {title}
        </h2>
        <span className={`text-xs px-2 py-1 rounded-full shrink-0 ${statusColor}`}>
          {status}
        </span>
      </div>
      <p className="mt-2 text-sm text-gray-500">{dateRange}</p>
    </div>
  );
}
```

---

---

# 한눈에

| 카테고리   | 예시                                                       |
| ------ | -------------------------------------------------------- |
| 여백     | `p-4` `px-6` `m-auto` `gap-4`                            |
| 크기     | `w-full` `h-screen` `max-w-xl`                           |
| 텍스트    | `text-lg` `font-bold` `text-gray-500` `truncate`         |
| 배경·테두리 | `bg-white` `rounded-lg` `border`                         |
| Flex   | `flex` `items-center` `justify-between`                  |
| Grid   | `grid` `grid-cols-3` `col-span-2`                        |
| 반응형    | `md:grid-cols-2` `lg:hidden`                             |
| 상태     | `hover:bg-blue-600` `focus:ring-2` `disabled:opacity-50` |
| 다크모드   | `dark:bg-gray-900` `dark:text-white`                     |