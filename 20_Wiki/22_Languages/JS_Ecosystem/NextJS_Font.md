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

# 사용자 설정 폰트 시스템 ⭐️⭐️⭐️⭐️

```txt
패턴: localStorage 저장 → CSS 변수 주입 → CSS calc/var 적용
  JS에서 fontId, scale을 저장하고
  CSS --room-chat-font-family, --room-chat-font-scale 변수로 주입
  CSS에서 var()로 읽어 실제 스타일 적용
```

## 상수 & 타입 정의

```typescript
// as const 배열 → T[number] 타입 → 타입 서술어 조합 패턴
export const CHAT_FONT_IDS    = ['default', 'gothic', 'serif', 'hand'] as const;
export const CHAT_FONT_SCALES = ['S', 'M', 'L'] as const;

export type ChatFontId    = (typeof CHAT_FONT_IDS)[number];
// → 'default' | 'gothic' | 'serif' | 'hand'
export type ChatFontScale = (typeof CHAT_FONT_SCALES)[number];
// → 'S' | 'M' | 'L'

export type ChatFontPrefs = {
  fontId: ChatFontId;
  scale:  ChatFontScale;
};
```

## Record 룩업 테이블


```typescript
/** 화면에 표시할 레이블 */
export const CHAT_FONT_LABELS: Record<ChatFontId, string> = {
  default: '기본',
  gothic:  '고딕',
  serif:   '세리프',
  hand:    '손글씨',
};

export const CHAT_FONT_SCALE_LABELS: Record<ChatFontScale, string> = {
  S: '작게', M: '보통', L: '크게',
};

/** CSS에 주입할 배율 */
export const CHAT_FONT_SCALE_VALUE: Record<ChatFontScale, number> = {
  S: 0.92, M: 1, L: 1.12,
};

/** CSS font-family 값 (폰트 스택 포함) */
export const CHAT_FONT_FAMILY: Record<ChatFontId, string> = {
  default: 'inherit',
  gothic:  '"Pretendard", "Apple SD Gothic Neo", "Noto Sans KR", sans-serif',
  serif:   '"Noto Serif KR", "Apple SD Gothic Neo", serif',
  hand:    '"napkin-hand", "Nanum Pen Script", cursive',
};
```

```txt
Record<K, V> + as const 조합이 좋은 이유:
  CHAT_FONT_IDS에 새 폰트를 추가하면
  TypeScript가 CHAT_FONT_FAMILY 등 Record에서 누락 항목을 에러로 잡아줌
  → 상수 하나 추가 시 자동으로 모든 테이블 동기화 강제
  → [[TS_Utility_Types]] Record 참고
```

## localStorage 저장/읽기 — 타입 검증 포함


```typescript
const PREFIX = 'chat-font:';

function storageKey(userId: string) { return `${PREFIX}${userId}`; }

// 타입 서술어 — 유효한 값인지 검증
function isFontId(v: string): v is ChatFontId {
  return (CHAT_FONT_IDS as readonly string[]).includes(v);
}
function isScale(v: string): v is ChatFontScale {
  return (CHAT_FONT_SCALES as readonly string[]).includes(v);
}

function defaultPrefs(): ChatFontPrefs { return { fontId: 'default', scale: 'M' }; }

// 읽기
export function getChatFontPrefs(userId: string): ChatFontPrefs {
  if (typeof window === 'undefined') return defaultPrefs();  // SSR 가드
  const raw = localStorage.getItem(storageKey(userId));
  if (!raw) return defaultPrefs();
  try {
    const parsed = JSON.parse(raw) as Partial<ChatFontPrefs>;
    return {
      fontId: isFontId(parsed.fontId ?? '') ? parsed.fontId! : 'default',
      scale:  isScale(parsed.scale  ?? '') ? parsed.scale!  : 'M',
    };
  } catch {
    return defaultPrefs();
  }
}

// 쓰기 — 저장 전에도 검증
export function setChatFontPrefs(userId: string, prefs: ChatFontPrefs) {
  if (typeof window === 'undefined') return;
  localStorage.setItem(storageKey(userId), JSON.stringify({
    fontId: isFontId(prefs.fontId) ? prefs.fontId : 'default',
    scale:  isScale(prefs.scale)   ? prefs.scale  : 'M',
  }));
}
```


```txt
이 패턴이 다루는 세 가지 방어:
  1. SSR 가드  typeof window === 'undefined'  서버에서 실행 시 기본값
  2. 없는 값   !raw → defaultPrefs()           처음 방문 시 기본값
  3. 잘못된 값 isFontId/isScale 검증          저장값이 손상됐을 때 기본값

parsed.fontId ?? '':
  Partial<T>이라 fontId가 없을 수 있음 → ?? '' → isPresetId('') → false → 기본값
  → [[TS_Utility_Types]] Partial / [[TS_Type_Guards]] as const 타입 서술어 참고
```

---

## CSS 변수로 동적 폰트 적용 ⭐️⭐️⭐️⭐️


```css
/* globals.css 또는 컴포넌트 스코프 CSS */
.room-chat-font .room-bubble,
.room-chat-font .room-composer textarea {
  font-family: var(--room-chat-font-family, inherit);
  font-size:   calc(15px * var(--room-chat-font-scale, 1));
}
```

```typescript
// JS에서 CSS 변수 주입
function applyChatFontPrefs(element: HTMLElement, prefs: ChatFontPrefs) {
  element.style.setProperty(
    '--room-chat-font-family',
    CHAT_FONT_FAMILY[prefs.fontId],
  );
  element.style.setProperty(
    '--room-chat-font-scale',
    String(CHAT_FONT_SCALE_VALUE[prefs.scale]),
  );
}

// React에서 사용
const containerRef = useRef<HTMLDivElement>(null);

useEffect(() => {
  if (!containerRef.current) return;
  applyChatFontPrefs(containerRef.current, prefs);
}, [prefs]);

return <div ref={containerRef} className="room-chat-font">...</div>;
```

```txt
CSS 변수 + calc() 패턴:
  var(--room-chat-font-family, inherit)
    → 변수가 없으면 inherit(기본값) 사용

  calc(15px * var(--room-chat-font-scale, 1))
    → scale: S(0.92) → 13.8px
    → scale: M(1)    → 15px
    → scale: L(1.12) → 16.8px
    → JS에서 숫자(배율)만 바꾸면 CSS가 알아서 계산

.room-chat-font 클래스 범위:
  전체 앱에 적용하지 않고 채팅 컨테이너(.room-chat-font)에만 적용
  CSS 변수는 해당 요소와 자식에만 영향

setProperty vs className:
  className    → 클래스를 통째로 바꿔야 해서 경우의 수가 많음
  setProperty  → 변수 값만 바꾸면 CSS가 자동으로 반영 → 조합이 자유로움
```

---
