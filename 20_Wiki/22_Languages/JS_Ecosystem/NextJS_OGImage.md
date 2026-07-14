---
aliases:
  - OGImage
tags:
  - NextJS
related:
  - "[[00_JS_Ecosystem_HomePage]]"
  - "[[NextJS_Concept]]"
  - "[[NextJS_Routing]]"
---
# NextJS_OGImage — OG 이미지 & 아이콘 생성

> [!info] 
> `next/og`의 `ImageResponse` = JSX → PNG 이미지를 서버에서 동적으로 생성하는 API. 
> 소셜 공유 썸네일(OG 이미지), Apple 터치 아이콘, 파비콘을 코드로 만들 때 사용한다.

---

# ImageResponse란 ⭐️⭐️⭐️⭐️

```typescript
import { ImageResponse } from 'next/og';

new ImageResponse(
  element,   // JSX 엘리먼트 — 이미지로 변환할 UI
  options,   // width, height 등
)
```

```txt
동작 원리:
  JSX → Satori(라이브러리)가 SVG로 변환 → PNG로 렌더링
  Edge Runtime에서 실행 (Node.js 런타임 아님)
  → 빠름, 전 세계 CDN 엣지에서 실행 가능

주요 제약:
  ① CSS: Tailwind 클래스 사용 불가 → 반드시 style={{}} 인라인 스타일
  ② 지원 CSS: flex, grid, position, font, border 등 부분 지원 (전부는 아님)
  ③ 외부 이미지: 절대 URL만 가능 (상대경로 불가)
  ④ 폰트: 시스템 폰트 기본, 한글 폰트는 직접 로드 필요
```

---

# 파일명 규칙 — App Router 특수 파일 ⭐️⭐️⭐️⭐️

```txt
App Router는 특정 파일명을 자동으로 메타데이터로 처리

app/
  icon.tsx              → /favicon.ico (파비콘)
  icon.png              → /icon.png (정적 파비콘)
  apple-icon.tsx        → /apple-touch-icon.png (iOS 홈화면 아이콘)
  opengraph-image.tsx   → OG 이미지 (소셜 공유 썸네일)
  twitter-image.tsx     → 트위터 카드 이미지

  app/(route)/
    opengraph-image.tsx → 그 경로 전용 OG 이미지
```

```txt
.tsx 파일이면 ImageResponse로 동적 생성
.png/.jpg/.ico 파일이면 정적 이미지 그대로 사용
→ 동적이 필요 없다면 그냥 PNG 파일을 넣는 게 더 간단
```

---

# Apple 아이콘 — apple-icon.tsx ⭐️⭐️⭐️

```tsx
// app/apple-icon.tsx
import { ImageResponse } from 'next/og';

export const size = { width: 180, height: 180 };
export const contentType = 'image/png';

export default function AppleIcon() {
  return new ImageResponse(
    (
      <div
        style={{
          width: '100%',
          height: '100%',
          display: 'flex',
          alignItems: 'center',
          justifyContent: 'center',
          background: 'linear-gradient(145deg, #d6e8f2 0%, #b9d4e4 100%)',
          borderRadius: '50%',
        }}
      >
        <span style={{ fontSize: 100, lineHeight: 1 }}>🌊</span>
      </div>
    ),
    { ...size },
  );
}
```

```txt
export const size: Next.js가 이미지 크기를 알아야 <meta> 태그 생성 가능
export const contentType: MIME 타입 — 브라우저가 이미지 종류를 알 수 있게

Apple Touch Icon 권장 크기: 180×180px
파비콘 권장 크기: 32×32px 또는 48×48px
```

---

# OG 이미지 — opengraph-image.tsx ⭐️⭐️⭐️⭐️

```tsx
// app/opengraph-image.tsx (전체 사이트 기본 OG 이미지)
import { ImageResponse } from 'next/og';

export const size = { width: 1200, height: 630 };
export const contentType = 'image/png';
export const alt = '사이트 이름';

export default function OGImage() {
  return new ImageResponse(
    (
      <div
        style={{
          width: '100%',
          height: '100%',
          display: 'flex',
          flexDirection: 'column',
          alignItems: 'center',
          justifyContent: 'center',
          backgroundColor: '#0f172a',
          padding: '60px',
        }}
      >
        <h1
          style={{
            fontSize: 72,
            fontWeight: 700,
            color: '#f8fafc',
            textAlign: 'center',
            lineHeight: 1.2,
          }}
        >
          사이트 이름
        </h1>
        <p style={{ fontSize: 36, color: '#94a3b8', marginTop: 24 }}>
          슬로건 또는 설명
        </p>
      </div>
    ),
    { ...size },
  );
}
```

```txt
OG 이미지 표준 크기: 1200×630px
  카카오, 트위터, 페이스북, 슬랙 모두 이 비율(1.91:1)에 최적화

alt 값:
  스크린리더, 이미지 로드 실패 시 표시되는 텍스트
  export const alt로 선언하면 Next.js가 자동으로 <meta> 태그에 적용
```

---

# 동적 OG 이미지 — 페이지별 다른 이미지 ⭐️⭐️⭐️⭐️

```tsx
// app/posts/[id]/opengraph-image.tsx
import { ImageResponse } from 'next/og';
import { getPost } from '@/lib/api';

export const size        = { width: 1200, height: 630 };
export const contentType = 'image/png';

type Props = {
  params: Promise<{ id: string }>;
};

export default async function OGImage({ params }: Props) {
  const { id } = await params;
  const post   = await getPost(id);  // DB 또는 API에서 글 정보 조회

  return new ImageResponse(
    (
      <div
        style={{
          width: '100%',
          height: '100%',
          display: 'flex',
          flexDirection: 'column',
          justifyContent: 'flex-end',
          padding: '60px 80px',
          backgroundColor: '#1e293b',
        }}
      >
        <p style={{ fontSize: 24, color: '#64748b', marginBottom: 16 }}>
          {post.category}
        </p>
        <h1
          style={{
            fontSize: 64,
            fontWeight: 700,
            color: '#f1f5f9',
            lineHeight: 1.3,
          }}
        >
          {post.title}
        </h1>
        <p style={{ fontSize: 28, color: '#94a3b8', marginTop: 24 }}>
          {post.author}
        </p>
      </div>
    ),
    { ...size },
  );
}
```

```txt
동적 OG 이미지가 작동하는 방식:
  /posts/123에 접근 → Next.js가 /posts/123/opengraph-image를 자동 생성
  카카오톡/슬랙 링크 공유 시 이 이미지가 썸네일로 표시

params가 Promise인 이유 (Next.js 15+):
  App Router의 params가 비동기로 변경됨
  → const { id } = await params;
```

---

# 한글 폰트 로드 ⭐️⭐️⭐️

```tsx
// 한글이 포함된 OG 이미지는 폰트를 직접 로드해야 함
// 기본 폰트는 한글을 렌더링하지 못함 (네모 박스로 나옴)

import { ImageResponse } from 'next/og';

async function loadFont() {
  const res = await fetch(
    new URL('/fonts/NotoSansKR-Bold.ttf', process.env.NEXT_PUBLIC_BASE_URL!)
  );
  return res.arrayBuffer();
}

export default async function OGImage() {
  const fontData = await loadFont();

  return new ImageResponse(
    <div style={{ fontFamily: 'Noto Sans KR' }}>한글 텍스트</div>,
    {
      width: 1200,
      height: 630,
      fonts: [
        {
          name: 'Noto Sans KR',
          data: fontData,
          weight: 700,
        },
      ],
    },
  );
}
```

```txt
한글 OG 이미지에서 텍스트가 □□□로 나온다면:
  폰트 파일(.ttf/.woff)을 public/ 폴더에 넣고
  위 패턴으로 ArrayBuffer로 로드해서 fonts 옵션에 전달

fonts 옵션:
  name    CSS font-family에서 사용할 이름
  data    ArrayBuffer (폰트 파일 바이너리)
  weight  폰트 굵기 (400, 700 등)
```

---

# CSS 지원 범위 — 주의사항 ⭐️⭐️⭐️

```tsx
// ✅ 잘 동작하는 CSS
style={{
  display: 'flex',           // flex 레이아웃
  flexDirection: 'column',
  alignItems: 'center',
  justifyContent: 'center',
  backgroundColor: '#000',
  color: '#fff',
  fontSize: 48,
  fontWeight: 700,
  padding: '20px 40px',
  borderRadius: 16,
  lineHeight: 1.5,
  backgroundImage: 'linear-gradient(...)',
}}

// ❌ 동작 안 하거나 제한적인 CSS
// - CSS 변수 (var(--color))
// - Tailwind 클래스
// - box-shadow가 복잡한 경우
// - calc()
// - CSS animation
// - overflow: hidden + border-radius 조합 (불안정)
```

```txt
Satori 지원 CSS 목록:
  https://github.com/vercel/satori#css

기본 원칙:
  복잡한 CSS보다 단순한 레이아웃이 안전
  문제가 생기면 해당 스타일을 제거하고 어디서 깨지는지 확인
```

---

# generateImageMetadata — 여러 크기 아이콘 ⭐️⭐️

```tsx
// app/icon.tsx — 여러 크기 아이콘 동시 생성
import { ImageResponse } from 'next/og';

export function generateImageMetadata() {
  return [
    { id: 'small',  size: { width: 48,  height: 48  }, contentType: 'image/png' },
    { id: 'medium', size: { width: 72,  height: 72  }, contentType: 'image/png' },
    { id: 'large',  size: { width: 192, height: 192 }, contentType: 'image/png' },
  ];
}

export default function Icon({ id }: { id: string }) {
  const size = id === 'small' ? 24 : id === 'medium' ? 48 : 96;

  return new ImageResponse(
    <div style={{ /* ... */ }}>
      <span style={{ fontSize: size }}>🌊</span>
    </div>,
  );
}
```

---

# 한눈에

```txt
import { ImageResponse } from 'next/og'

JSX → PNG 이미지 생성
Edge Runtime에서 실행 (빠름)

파일명 규칙 (App Router):
  apple-icon.tsx       → Apple 터치 아이콘 (180×180)
  icon.tsx             → 파비콘
  opengraph-image.tsx  → OG 이미지 (1200×630)
  동적 경로 폴더 안에도 넣으면 그 페이지 전용 OG 이미지

export:
  export const size = { width, height }   이미지 크기
  export const contentType = 'image/png'  MIME 타입
  export const alt = '설명'              OG alt 텍스트

CSS 주의:
  Tailwind 클래스 안 됨 → 반드시 style={{}} 인라인 스타일
  모든 CSS가 지원되는 건 아님 → 단순하게 유지

한글 폰트:
  기본 폰트로 한글 렌더링 불가 → TTF 파일 로드 → fonts 옵션에 전달

동적 OG:
  async 함수로 선언 → params await → DB/API 조회 → 글 제목/내용으로 이미지 생성
```