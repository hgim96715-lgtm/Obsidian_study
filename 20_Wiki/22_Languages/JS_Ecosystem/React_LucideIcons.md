---
aliases:
  - Lucide
  - Icons
  - UI
tags:
  - React
related:
  - "[[00_JS_Ecosystem_HomePage]]"
  - "[[React_Component]]"
  - "[[TS_Generics]]"
---
# React_LucideIcons — lucide-react 아이콘

> [!info] 
> lucide-react = React용 아이콘 라이브러리. 
> 각 아이콘이 React 컴포넌트로 제공된다. https://lucide.dev/icons 에서 아이콘 이름 검색 가능.

---

# 설치 ⭐️⭐️⭐️

```bash
pnpm add lucide-react
```

---

# 기본 사용 ⭐️⭐️⭐️⭐️

```typescript
// 각 아이콘을 named import로 가져옴
import { Home, Settings, Search, X, ChevronDown } from 'lucide-react';

// JSX에서 컴포넌트처럼 사용
<Home />
<Settings />
<Search className="size-4" />
```

```txt
아이콘 이름 규칙:
  lucide 사이트 이름 그대로 PascalCase로 import
  https://lucide.dev/icons 에서 검색 → 이름 확인
  예: "chevron-down" → ChevronDown, "arrow-left" → ArrowLeft

각 아이콘 = React 함수 컴포넌트:
  <Home />  →  <svg ...> 를 렌더링
  props으로 크기, 색상, 두께 조절
```

---

# 주요 props ⭐️⭐️⭐️⭐️

```typescript
<Home
  size={24}              // 아이콘 크기 (px) — 기본값 24
  color="currentColor"   // 색상 — 기본값 currentColor (부모 색상 상속)
  strokeWidth={2}        // 선 두께 — 기본값 2
  className="size-4"     // Tailwind 클래스
  aria-hidden            // 스크린리더에서 숨김 (장식용 아이콘에 필수)
/>
```

```txt
size vs className:
  size={20}       → width="20" height="20" 속성으로 설정
  className="size-4"  → Tailwind w-4 h-4 (1rem = 16px) 로 설정
  둘 중 하나만 — className 방식이 Tailwind 프로젝트에서 더 일관성 있음

color="currentColor":
  기본값 — 부모의 CSS color를 그대로 상속
  text-blue-500 클래스를 부모에 주면 아이콘도 파란색이 됨
  특정 색을 고정하려면 color="#ff0000" 또는 className="text-red-500"

aria-hidden:
  아이콘 옆에 텍스트 레이블이 있을 때 → aria-hidden (중복 읽기 방지)
  아이콘만 있고 텍스트 없을 때 → aria-label="설명" 필요
```

## Tailwind 크기 조합

|className|크기|
|---|---|
|`size-3`|12px|
|`size-3.5`|14px|
|`size-4`|16px|
|`size-5`|20px|
|`size-6`|24px (lucide 기본)|
|`size-8`|32px|

---

# 아이콘을 변수로 — typeof ⭐️⭐️⭐️⭐️

```typescript
import { Home, Settings, type LucideIcon } from 'lucide-react';

// 방법 1 — typeof로 타입 추출
type NavItem = {
  icon: typeof Home;  // Home의 타입 = lucide 아이콘 컴포넌트 타입
};

// 방법 2 — lucide-react의 전용 타입 (더 명시적)
type NavItem = {
  icon: LucideIcon;   // lucide-react가 export하는 아이콘 타입
};
```

```typescript
// 변수에 담아서 동적으로 사용
const items = [
  { label: '홈', icon: Home },
  { label: '설정', icon: Settings },
];

// 구조분해 시 반드시 대문자로 rename
items.map(({ label, icon: Icon }) => (
  //                  ↑ icon을 Icon으로 rename — 소문자면 HTML 태그로 해석됨
  <div key={label}>
    <Icon className="size-4" aria-hidden />  {/* ← 대문자라 컴포넌트로 인식 */}
    <span>{label}</span>
  </div>
));
```

```txt
icon: Icon rename이 필요한 이유:
  JSX는 소문자 시작 = HTML 태그, 대문자 시작 = 컴포넌트
  const icon = Home; → <icon /> → ❌ HTML 태그
  const Icon = Home; → <Icon /> → ✅ React 컴포넌트

LucideIcon vs typeof Home:
  typeof Home        → Home 컴포넌트의 타입 (편리하지만 특정 아이콘에 의존)
  LucideIcon         → lucide-react 공식 타입, 의미가 명확
  둘은 사실상 동일하지만 LucideIcon이 더 자기 설명적
```

---

# prop으로 아이콘 받기 ⭐️⭐️⭐️

```typescript
import { type LucideIcon } from 'lucide-react';

type ButtonProps = {
  icon?:  LucideIcon;
  label:  string;
};

function IconButton({ icon: Icon, label }: ButtonProps) {
  return (
    <button>
      {Icon && <Icon className="size-4" aria-hidden />}
      <span>{label}</span>
    </button>
  );
}

// 사용
<IconButton icon={Settings} label="설정" />
<IconButton label="텍스트만" />          // icon 없어도 됨
```

---

# 자주 쓰는 아이콘 모음

```typescript
import {
  // 탐색
  Home, ArrowLeft, ArrowRight, ChevronDown, ChevronUp, ChevronLeft, ChevronRight,
  X, Menu,

  // 액션
  Search, Plus, Minus, Edit, Trash2, Save, Download, Upload, Copy, Share2,
  Send, RefreshCw,

  // 상태
  Check, CheckCircle, AlertCircle, AlertTriangle, Info, Loader2,
  Eye, EyeOff,

  // 사용자
  User, UserRound, Users, UserPlus, Settings,

  // 미디어
  Image, Camera, Music, Play, Pause, Volume2, VolumeX,
  Palette, Pen,

  // 데이터
  BarChart3, TrendingUp, Star, Heart, Bookmark, Flag,

  // 기타
  Globe, Lock, Unlock, Bell, Mail, Phone,
  Calendar, Clock, MapPin, Link, ExternalLink,
} from 'lucide-react';
```

```txt
아이콘 검색:
  https://lucide.dev/icons 에서 영어로 검색
  원하는 아이콘 클릭 → 이름 확인 → import

Loader2 스피너 애니메이션:
  <Loader2 className="size-4 animate-spin" />
  Tailwind animate-spin으로 회전 애니메이션

strokeWidth 조절:
  strokeWidth={1}    → 얇고 가벼운 느낌
  strokeWidth={2}    → 기본값
  strokeWidth={2.5}  → 굵고 강조된 느낌
```

---

# 한눈에

```txt
설치:   pnpm add lucide-react
import: import { Home, Settings } from 'lucide-react'
사용:   <Home className="size-4" aria-hidden />

주요 props:
  size={20}          크기 (px)
  strokeWidth={2}    선 두께
  className="size-4" Tailwind 크기
  aria-hidden        스크린리더 숨김 (텍스트 레이블이 있을 때)
  color="currentColor" 부모 색상 상속 (기본)

타입:
  LucideIcon                아이콘 컴포넌트 타입 (공식)
  typeof Home               Home과 동일한 타입

동적 사용:
  { icon: Icon }           구조분해 시 대문자 rename 필수
  <Icon />                 대문자여야 컴포넌트로 인식

아이콘 검색: https://lucide.dev/icons
```