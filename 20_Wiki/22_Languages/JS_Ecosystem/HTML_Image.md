---
aliases: [decoding async, fetchpriority, lazy loading, LCP, loading lazy, srcset]
tags: [HTML]
related:
  - "[[00_JS_Ecosystem_HomePage]]"
  - "[[HTML_ARIA]]"
---
# HTML_Image — 이미지 최적화 속성

> [!info]
> `<img>` 태그의 성능 관련 속성 모음.
> `loading` · `decoding` · `fetchpriority` — 브라우저가 이미지를 언제, 어떻게 처리할지 제어.

---

# `loading` — 네트워크 요청 시점 제어 ⭐️⭐️⭐️⭐️

```tsx
<img
  src={posterUrl}
  alt={movie.title}
  loading="lazy"
/>
```

```txt
loading="eager"  (기본값)
  → 페이지 로드 즉시 이미지 요청
  → 뷰포트 밖에 있어도 바로 네트워크 요청 발생

loading="lazy"
  → 뷰포트 근처까지 스크롤될 때까지 요청 연기
  → 초기 로드에 불필요한 네트워크 요청 차단
  → 브라우저 내부적으로 Intersection Observer API 활용
```

```txt
왜 쓰는가:
  검색 결과 포스터처럼 한 번에 수십 장이 렌더링되지만 전부 안 보이는 경우
  → lazy 없으면 초기에 N개 동시 요청 → 대역폭 낭비, 초기 로드 지연
  → lazy 적용 시 스크롤될 때마다 필요한 만큼만 요청
```

---

# `decoding` — 메인 스레드 블로킹 제어 ⭐️⭐️⭐️⭐️

```txt
이미지 파일(JPEG/PNG 등)은 다운로드 후 픽셀 데이터로 변환(디코딩)해야 화면에 표시됨

decoding="sync"  (기본값)
  → 메인 스레드에서 디코딩
  → 완료될 때까지 렌더링 파이프라인 블로킹

decoding="async"
  → 별도 워커 스레드에서 디코딩
  → 메인 스레드는 계속 실행 → 다른 렌더링·JS 작업에 영향 없음

decoding="auto"
  → 브라우저가 상황에 맞게 선택
```

```txt
브라우저 렌더링 파이프라인과의 연결:
  Parse HTML → DOM 구성 → Layout → Paint → Composite
                                      ↑
                              이 단계에서 이미지 픽셀 필요
                              decoding="async" → Paint와 분리
                              → 메인 스레드 블로킹 시간 감소
```

---

# 실전 조합 패턴 ⭐️⭐️⭐️⭐️

```tsx
// ✅ 스크롤 후 보이는 이미지 (검색결과, 카드, 리스트)
<img
  src={posterUrl}
  alt={movie.title}
  loading="lazy"
  decoding="async"
/>

// ✅ 첫 화면에 바로 보이는 이미지 (히어로, 배너)
<img
  src={heroUrl}
  alt="메인 배너"
  fetchpriority="high"   // 브라우저 다운로드 우선순위 상승
/>
// loading·decoding 기본값(eager·sync) 유지 — 생략
```

---

# ⚠️ LCP 이미지에는 lazy 금지 ⭐️⭐️⭐️⭐️⭐️

```txt
LCP (Largest Contentful Paint)
  = 뷰포트 안에서 가장 큰 콘텐츠가 렌더링되는 시점
  = Core Web Vitals 핵심 지표 (2.5s 이하 권장)

히어로 이미지, 상단 배너, 첫 화면 메인 포스터
→ loading="lazy" 쓰면 초기 로드에서 요청이 연기됨 → LCP 악화

선택 기준:
  스크롤해야 보이는 이미지  → loading="lazy"  + decoding="async"
  첫 화면에 바로 보이는 이미지 → lazy 쓰지 말 것 + fetchpriority="high"
```

---

# `fetchpriority` — 다운로드 우선순위 ⭐️⭐️⭐️

```txt
fetchpriority="high"   → 다른 리소스보다 먼저 다운로드 (LCP 이미지에 사용)
fetchpriority="low"    → 낮은 우선순위 (광고·장식 이미지 등)
fetchpriority="auto"   → 브라우저가 결정 (기본값)
```

```tsx
// preload 없이도 브라우저가 빠르게 요청하도록 힌트
<img
  src="/hero.jpg"
  alt="히어로"
  fetchpriority="high"
/>
```
