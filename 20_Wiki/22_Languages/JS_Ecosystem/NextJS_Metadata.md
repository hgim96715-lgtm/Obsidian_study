---
aliases:
  - Metadata
  - template
  - title
  - viewport
  - generateMetadata
tags:
  - NextJS
  - HTML
related:
  - "[[00_JS_Ecosystem_HomePage]]"
  - "[[NextJS_Concept]]"
---
# NextJS_Metadata — 메타데이터

>[!info]
>메타데이터 = HTML `<head>` 안에 들어가는 "페이지에 대한 정보". 
>브라우저 탭 이름·SEO·소셜 공유 미리보기를 제어한다. 
>Next.js App Router에서는 `export const metadata`로 선언하거나 `generateMetadata()`로 동적 생성.

---

# 메타데이터란 ⭐️⭐️⭐️⭐️

```txt
브라우저가 페이지를 보여줄 때 <head> 안의 태그들을 읽음:
  <title>CINEMO</title>
  <meta name="description" content="영화관 로비 소셜">
  <meta property="og:image" content="/og.png">

이 태그들을 "메타데이터"라고 함 — 페이지 내용이 아닌 페이지에 대한 정보

어디에 쓰이는가:
  브라우저 탭 이름            → <title>
  구글 검색 결과 제목·설명     → <title>, <meta name="description">
  카카오톡·슬랙에 링크 공유 시 → <meta property="og:*"> (Open Graph)
  트위터 카드                 → <meta name="twitter:*">
  파비콘 (탭 아이콘)          → <link rel="icon">
```

```txt
Next.js 이전에는:
  <head> 태그를 직접 작성하거나
  라이브러리(react-helmet 등)로 관리

Next.js App Router:
  export const metadata = { title: '...' } 한 줄로
  Next.js가 알아서 <head>에 넣어줌
  → 페이지마다 다른 제목·설명을 쉽게 관리 가능
```

---

# 왜 메타데이터가 필요한가 ⭐️⭐️⭐️

```txt
<title> — 브라우저 탭 이름, 검색엔진 결과 제목
<meta name="description"> — 검색엔진 결과 설명문

Open Graph (og:) — 카카오톡·슬랙·트위터 등에 공유할 때 미리보기
  og:title, og:description, og:image → 공유 카드에 표시

robots — 검색엔진이 이 페이지를 색인할지
```

---

# 정적 메타데이터 — export const metadata ⭐️⭐️⭐️⭐️

```typescript
// app/layout.tsx 또는 app/page.tsx
import type { Metadata } from 'next';

export const metadata: Metadata = {
  title: 'CINEMO',
  description: '영화관 로비 소셜 — 매표소 · 뽑기 · 후기방',
};
```

```txt
layout.tsx에 선언:
  모든 하위 페이지에 기본값으로 적용

page.tsx에 선언:
  그 페이지에서만 적용 (layout을 덮어씀)

두 곳 모두 있으면:
  page.tsx > layout.tsx (더 구체적인 쪽이 우선)
```

---

# Metadata 주요 필드 ⭐️⭐️⭐️⭐️

```typescript
export const metadata: Metadata = {
  // 기본
  title:       'CINEMO',
  description: '영화관 로비 소셜',

  // title 템플릿 — 하위 페이지에서 활용
  title: {
    default:  'CINEMO',            // 하위 page에 title 없으면 이것
    template: '%s | CINEMO',       // 하위 page.title → "영화 목록 | CINEMO"
  },

  // Open Graph — SNS 공유 미리보기
  openGraph: {
    title:       'CINEMO',
    description: '영화관 로비 소셜',
    url:         'https://cinemo.example.com',
    siteName:    'CINEMO',
    images: [
      {
        url:    '/og-image.png',   // public/ 폴더 기준
        width:  1200,
        height: 630,
        alt:    'CINEMO 로비',
      },
    ],
    locale: 'ko_KR',
    type:   'website',
  },

  // 트위터 카드
  twitter: {
    card:        'summary_large_image',
    title:       'CINEMO',
    description: '영화관 로비 소셜',
    images:      ['/og-image.png'],
  },

  // 검색엔진 색인 제어
  robots: {
    index:  true,
    follow: true,
  },

  // 파비콘 · 앱 아이콘
  icons: {
    icon:  '/favicon.ico',
    apple: '/apple-icon.png',
  },

  // 정규 URL (중복 페이지 처리)
  alternates: {
    canonical: 'https://cinemo.example.com',
  },
};
```

---

# title 템플릿 패턴 ⭐️⭐️⭐️⭐️

```typescript
// app/layout.tsx — 루트 레이아웃에 템플릿 설정
export const metadata: Metadata = {
  title: {
    default:  'CINEMO',         // 하위 페이지에 title 없으면 이것
    template: '%s | CINEMO',    // %s = 하위 page.tsx의 title
  },
};

// app/posts/page.tsx — title만 지정
export const metadata: Metadata = {
  title: '후기방',              // → 탭에 "후기방 | CINEMO" 로 표시
};

// app/page.tsx — title 없으면 default 사용
// → 탭에 "CINEMO" 로 표시
```

```txt
%s 자리에 하위 page의 title이 들어감
모든 페이지마다 " | CINEMO" 붙이지 않아도 됨
하위 페이지에서 title만 지정하면 자동으로 템플릿 적용
```

---

# 동적 메타데이터 — generateMetadata ⭐️⭐️⭐️⭐️

```typescript
// app/posts/[id]/page.tsx — 동적 라우트
import type { Metadata } from 'next';

type Props = {
  params: { id: string };
};

// 서버에서 실행 — params로 데이터 fetch 후 메타데이터 생성
export async function generateMetadata({ params }: Props): Promise<Metadata> {
  const post = await fetchPost(params.id);

  return {
    title:       post.title,
    description: post.summary,
    openGraph: {
      title:  post.title,
      images: [post.thumbnailUrl],
    },
  };
}

export default function PostPage({ params }: Props) {
  // ...
}
```

```txt
generateMetadata vs export const metadata:
  export const metadata  → 빌드 시 고정값 (정적)
  generateMetadata()     → 요청마다 실행 (동적, params·fetch 가능)
  → 동적 라우트([id])처럼 페이지마다 다른 제목이 필요할 때 사용

generateMetadata도 async 함수:
  fetch로 DB·API 조회 가능
  Next.js가 데이터 페칭을 자동으로 중복 제거 (page.tsx와 같은 fetch면 캐시 재사용)
```

---

# 파일 기반 메타데이터

```txt
app/ 폴더 안에 특정 파일명으로 두면 자동 적용:

  favicon.ico     → /favicon.ico (브라우저 탭 아이콘)
  icon.png        → 앱 아이콘
  apple-icon.png  → iOS 홈화면 아이콘
  opengraph-image.png → OG 이미지 (코드 없이 파일만으로)
  robots.txt      → 검색엔진 크롤링 규칙
  sitemap.ts      → 사이트맵 자동 생성
```

```typescript
// app/sitemap.ts
import type { MetadataRoute } from 'next';

export default function sitemap(): MetadataRoute.Sitemap {
  return [
    { url: 'https://cinemo.example.com',       lastModified: new Date() },
    { url: 'https://cinemo.example.com/posts', lastModified: new Date() },
  ];
}
```