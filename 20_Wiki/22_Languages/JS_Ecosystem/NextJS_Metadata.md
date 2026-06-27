---
aliases:
  - Metadata
  - generateMetadata
  - title
  - template
  - viewport
  - SEO
tags:
  - NextJS
related:
  - "[[00_JS_Ecosystem_HomePage]]"
---

# NextJS_Metadata — `<head>` 태그를 선언적으로 관리하기

> [!info] 
>  App Router에서는 `<Head>` JSX를 직접 안 쓰고, `layout.tsx`/`page.tsx`에서 `metadata` 객체(정적) 또는 `generateMetadata()` 함수(동적)를 export하면 Next.js가 그걸 읽어서 `<head>`에 `<title>`/`<meta>` 태그를 자동으로 만들어준다. 
>  Server Component에서만 동작하고, 부모 layout의 metadata는 자식 page의 metadata와 합쳐진다(병합, 덮어쓰기 아님).

```txt
metadata를 안 적어도 항상 들어가는 기본 태그 2개:
  <meta charset="utf-8" />
  <meta name="viewport" content="width=device-width, initial-scale=1" />
```

---

# 정적 메타데이터 — export const metadata ⭐️⭐️⭐️

```tsx
// app/layout.tsx 또는 app/page.tsx
import type { Metadata } from 'next';

export const metadata: Metadata = {
  title: 'My App',
  description: '사람이 추천하는 음악 커뮤니티',
};
```

```txt
이 값은 빌드 타임에 결정됨 — 페이지 내용이 요청마다 달라지지 않는다면(런타임 데이터에
의존하지 않는다면) 이 방식이 기본값임. 비용이 거의 안 들어서 가능하면 이쪽을 먼저 고려

⚠️ Server Component(layout.tsx/page.tsx)에서만 동작함 — 'use client' 컴포넌트에서는 못 씀
   (metadata는 페이지 자체의 속성이라, 클라이언트에서 나중에 바뀌는 것과는 다른 층위의 개념)
```

---

# 동적 메타데이터 — generateMetadata() ⭐️⭐️⭐️

```tsx
// app/posts/[id]/page.tsx — 게시글 제목을 메타데이터에도 그대로 쓰고 싶을 때
import type { Metadata } from 'next';

type Props = { params: Promise<{ id: string }> };

export async function generateMetadata({ params }: Props): Promise<Metadata> {
  const { id } = await params;
  const post = await fetchPost(id); // 이미 page 컴포넌트에서도 호출하는 함수 — fetch는 자동 중복 제거됨

  return {
    title: post.title,
    description: post.content.slice(0, 100),
  };
}

export default async function Page({ params }: Props) {
  const { id } = await params;
  const post = await fetchPost(id);
  return <article>{post.title}</article>;
}
```

```txt
generateMetadata와 page 컴포넌트가 같은 fetchPost(id)를 따로 호출해도,
Next.js가 같은 요청을 자동으로 중복 제거(memoization)해줌 — 두 번 호출한다고 두 번 요청 가는 게 아님

언제 generateMetadata를 쓰나:
  제목/설명이 라우트 파라미터나 DB/API에서 가져온 데이터에 따라 달라질 때(블로그 글, 상품 상세 등)
  런타임 데이터에 의존하지 않는다면 굳이 안 쓰고 정적 metadata 객체로 충분함 — 원칙은 "필요할 때만"
```

---

# layout과 page의 상속/병합 — cascade ⭐️⭐️⭐️

```txt
metadata는 라우트 트리를 따라 "위에서 아래로" 내려오면서 합쳐짐(cascade)
  root layout.tsx 의 metadata = 사이트 전체 기본값
  그 아래 page.tsx 의 metadata = 그 페이지만의 값 — 같은 필드가 있으면 자식 값이 부모 값을 덮어씀

⚠️ "덮어쓰기"가 전체 교체가 아니라 부분 병합(shallow merge)이라는 점이 중요:
  자식이 title만 새로 정의하면 title만 바뀌고, description처럼 자식이 안 적은 필드는
  부모 값이 그대로 유지됨 — 부모의 metadata 전체가 사라지는 게 아님
```

|위치|정의한 필드|결과|
|---|---|---|
|`app/layout.tsx`|`title: 'My App'`, `description: '...'`|사이트 기본값|
|`app/posts/page.tsx`|`title: '게시글 목록'`|title만 교체, description은 layout 값 그대로 유지|

---

# title — 문자열 vs 템플릿 객체 ⭐️⭐️⭐️

```tsx
// app/layout.tsx — 루트에서 템플릿을 한 번만 정의
export const metadata: Metadata = {
  title: {
    default: 'My App',        // 자식이 title을 따로 안 적었을 때 쓰이는 기본값
    template: '%s | My App',  // 자식이 title을 적으면 %s 자리에 그 값이 들어감
  },
};
```

```tsx
// app/posts/page.tsx — 자식은 이름만 적으면 됨
export const metadata: Metadata = {
  title: '게시글 목록', // 최종 결과: "게시글 목록 | My App"
};
```

```txt
이 패턴이 유용한 이유:
  사이트 이름을 모든 페이지마다 "게시글 목록 | My App" 처럼 반복해서 적지 않아도 됨
  사이트 이름이 바뀌면 루트 layout 한 곳만 고치면 모든 페이지에 반영됨

title.default는 "자식이 title 자체를 아예 안 적었을 때"의 fallback이고,
title.template은 "자식이 title을 적었을 때, 거기에 추가로 씌우는 틀" — 역할이 다름
```

---

# Metadata와 같이 알아야 하는 것들

## viewport — metadata에서 분리된 별도 export ⭐️⭐️

```txt
예전엔 viewport/themeColor/colorScheme도 metadata 객체 안에 같이 넣었는데,
지금은 별도로 분리된 viewport export(또는 generateViewport 함수)를 씀
metadata 안에 그대로 넣으면 deprecated 경고가 뜸
```

```tsx
// app/layout.tsx
import type { Viewport } from 'next';

export const viewport: Viewport = {
  width: 'device-width',
  initialScale: 1,
  themeColor: '#000000',
};
```

## metadataBase — 상대경로의 기준점 ⭐️

```tsx
export const metadata: Metadata = {
  metadataBase: new URL('https://example.com'),
  openGraph: {
    images: ['/og-image.png'], // metadataBase 덕분에 https://example.com/og-image.png 로 해석됨
  },
};
```

```txt
openGraph.images 같은 필드에 절대 URL이 아니라 상대 경로만 적어도,
metadataBase를 한 번 지정해두면 Next.js가 자동으로 합쳐서 완전한 URL을 만들어줌
보통 루트 layout.tsx에 한 번만 지정해두는 값
```

## 파일 기반 메타데이터 — 코드 대신 파일로 ⭐️⭐️

```txt
metadata 객체 말고, app/ 폴더에 정해진 이름의 파일을 두면 자동으로 인식되는 방식도 있음
```

|파일명|역할|
|---|---|
|`favicon.ico`, `icon.png`|브라우저 탭/북마크에 쓰이는 아이콘|
|`apple-icon.png`|iOS 홈 화면 추가 시 쓰이는 아이콘|
|`opengraph-image.png` (또는 `.tsx`)|소셜 공유 시 보이는 미리보기 이미지 — `.tsx`로 만들면 동적 생성 가능|
|`twitter-image.png`|트위터(X) 공유 전용 이미지 (없으면 opengraph-image 재사용)|
|`robots.ts`|검색엔진 크롤러에게 어떤 경로를 허용/차단할지 알려줌|
|`sitemap.ts`|사이트의 전체 URL 목록을 검색엔진에 제공|

```txt
이 파일들도 .tsx로 만들면(opengraph-image.tsx 등) 코드로 동적 생성 가능 —
정적 이미지 파일이면 그냥 그대로 두면 됨, 둘 다 같은 위치(라우트 폴더 안)에 둠
```

---

# 한눈에

| 키워드                                | 한 줄 정리                                                               |
| ---------------------------------- | -------------------------------------------------------------------- |
| `export const metadata`            | 정적 메타데이터 — 빌드 타임에 결정, 비용 거의 없음                                       |
| `generateMetadata()`               | 동적 메타데이터 — params/fetch 데이터에 따라 달라질 때만                               |
| 적용 위치                              | `layout.tsx`/`page.tsx` (Server Component) — `'use client'`에서는 불가    |
| 상속/병합                              | 부모(layout) → 자식(page)으로 cascade, 부분 병합(자식이 안 적은 필드는 부모 값 유지)         |
| `title.default` / `title.template` | 기본값(자식이 안 적었을 때) / 틀(자식이 적었을 때 씌우는 형식)                               |
| `viewport`                         | metadata에서 분리된 별도 export — themeColor 등도 여기로 이동                      |
| `metadataBase`                     | 상대 경로 이미지/URL을 절대 URL로 자동 변환하는 기준점                                   |
| 파일 기반 메타데이터                        | favicon, opengraph-image, robots.ts, sitemap.ts — 코드 대신 정해진 파일명으로 처리 |