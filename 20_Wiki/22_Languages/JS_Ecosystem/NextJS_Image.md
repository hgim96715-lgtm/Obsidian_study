---
aliases: [이미지 최적화, fill, Image 컴포넌트, next/image, priority, remotePatterns]
tags: [NextJS, React]
related:
  - "[[00_JS_Ecosystem_HomePage]]"
  - "[[HTML_Image]]"
  - "[[NextJS_Config]]"
---
# NextJS_Image — next/image 이미지 최적화

---

# 왜 next/image인가 ⭐️⭐️⭐️⭐️⭐️

```txt
일반 <img>:
  브라우저가 원본 그대로 다운로드
  → JPEG 1MB 이미지를 200×200px 썸네일로 써도 1MB 전부 전송
  → 지연 로딩(lazy loading) 수동 처리 필요
  → WebP 변환 없음

next/image <Image />:
  → 서버에서 WebP/AVIF로 자동 변환 (파일 크기 30~50% 감소)
  → 뷰포트에 맞는 크기로 리사이징 후 전송
  → lazy loading 기본 적용 (화면 밖 이미지는 나중에 로드)
  → 이미지 영역 사전 확보 → CLS(레이아웃 이동) 방지
  → LCP(최대 콘텐츠 페인트) 점수 개선
```

| | `<img>` | `<Image />` (next/image) |
|--|:-------:|:------------------------:|
| WebP 자동 변환 | ❌ | ✅ |
| 크기 최적화 | ❌ | ✅ |
| lazy loading | 수동 | ✅ 기본 |
| CLS 방지 | ❌ | ✅ |
| 외부 도메인 | 자유 | next.config 허용 필요 |

---

# 설치 및 import

```typescript
import Image from 'next/image';
// next 설치 시 포함 — 별도 설치 불필요
```

---

# 기본 사용 — width/height 명시 ⭐️⭐️⭐️⭐️⭐️

```tsx
<Image
  src="/images/poster.jpg"   // public/ 기준 경로 or 외부 URL
  alt="영화 포스터"
  width={342}                // 렌더링 크기 (px)
  height={513}
/>
```

```txt
width/height:
  렌더링될 픽셀 크기 — CSS로 다시 줄일 수도 있지만 기준값 필요
  이 값으로 이미지 영역을 미리 확보 → CLS 방지

src:
  /로 시작 → public/ 폴더 기준
  https://로 시작 → 외부 URL (next.config에 도메인 허용 필요)
```

---

# fill — 부모 컨테이너에 꽉 채우기 ⭐️⭐️⭐️⭐️

```tsx
// 부모에 position: relative + 크기 지정 필수
<div style={{ position: 'relative', width: '100%', height: '300px' }}>
  <Image
    src={posterUrl}
    alt="포스터"
    fill
    style={{ objectFit: 'cover' }}  // 비율 유지하며 채우기
  />
</div>
```

```txt
fill 모드:
  width/height 대신 부모 크기에 맞춰 자동 채움
  → 부모에 position: relative 필수
  → objectFit: 'cover' | 'contain' | 'fill' 로 비율 제어

objectFit:
  cover   → 비율 유지, 넘치는 부분 잘라냄 (썸네일에 자주 사용)
  contain → 비율 유지, 여백 생길 수 있음
```

---

# 외부 이미지 — remotePatterns 설정 ⭐️⭐️⭐️⭐️⭐️

```typescript
// next.config.ts
const config: NextConfig = {
  images: {
    remotePatterns: [
      {
        protocol: 'https',
        hostname: 'image.tmdb.org',   // 허용할 외부 도메인
        pathname: '/t/p/**',          // 허용할 경로 패턴 (* = 한 단계, ** = 전체)
      },
    ],
  },
};
```

```txt
외부 URL을 <Image src="https://..."> 로 쓰려면 반드시 remotePatterns 등록 필요
등록 안 하면 런타임 에러: "hostname is not configured"

pathname:
  '/t/p/**' → /t/p/로 시작하는 모든 경로 허용
  생략 시 해당 도메인의 모든 경로 허용
```

---

# priority — LCP 이미지 최적화 ⭐️⭐️⭐️⭐️

```tsx
// 페이지 첫 화면에 바로 보이는 핵심 이미지
<Image
  src={heroImage}
  alt="메인 배너"
  width={1200}
  height={600}
  priority       // lazy loading 비활성화 → 즉시 로드
/>
```

```txt
기본값: lazy loading (뷰포트 밖은 나중에 로드)
priority: 페이지 로드 즉시 다운로드 시작

→ 히어로 이미지, 배너처럼 첫 화면에 보이는 이미지에만 사용
→ 남발하면 초기 로딩 속도 저하
```

---

# 일반 `<img>` 써도 되는 경우 ⭐️⭐️⭐️

```tsx
// ESLint 경고: @next/next/no-img-element
// → next/image 쓰도록 권장하는 규칙

// 아래 주석으로 억제
// eslint-disable-next-line @next/next/no-img-element
<img src={poster} alt={`${movie.title} 포스터`} />
```

```txt
일반 <img>가 허용되는 상황:
  ✅ 동적 외부 URL이 many하고 remotePatterns 패턴화가 불가능할 때
  ✅ 이미 최적화된 CDN 이미지 (TMDB, Cloudinary 등 — 서버에서 리사이징 제공)
  ✅ 간단한 프로토타입, 어드민 도구
  ✅ 이미지 URL이 blob: 또는 data: 형식일 때

일반 <img> 피해야 하는 상황:
  ❌ 서비스 대표 이미지, 히어로 이미지
  ❌ 사용자가 많아 대역폭 최적화가 중요한 프로덕션
  ❌ 원본 크기가 큰 이미지를 작게 표시하는 경우
```

---

# 자주 쓰는 패턴 — 포스터 이미지

```tsx
// tmdbPosterUrl() 같은 유틸로 URL 생성 후 조건부 렌더링
const poster = tmdbPosterUrl(movie.poster_path, 'w342');

{poster ? (
  <Image
    src={poster}
    alt={`${movie.title} 포스터`}
    width={342}
    height={513}
  />
) : (
  <span>포스터 없음</span>
)}
```

```txt
poster가 null일 수 있을 때:
  → 조건부 렌더링으로 fallback UI 처리
  → <Image src={null}> 는 에러
```

---

# placeholder — 로딩 중 블러 처리

```tsx
import Image from 'next/image';
import posterImg from '/public/poster.jpg';  // 로컬 이미지만 가능

<Image
  src={posterImg}
  alt="포스터"
  placeholder="blur"   // 로딩 중 흐릿한 미리보기
/>
```

```txt
placeholder="blur":
  로컬 이미지(import) → 빌드 시 자동으로 blur 데이터 생성
  외부 URL → blurDataURL 직접 제공 필요 (base64 인라인 이미지)

→ 외부 이미지 blur는 구현이 복잡해서 생략하는 경우가 많음
```
