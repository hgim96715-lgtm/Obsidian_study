---
aliases:
  - normalize embed url
  - embed URL 정규화
  - YouTube Spotify embed normalize
tags:
  - Snippet
related:
  - "[[00_Tools_Ecosystem_HomePage]]"
  - "[[00_Project_HomePage]]"
  - "[[00_NestJS_Ecosystem_HomePage]]"
  - "[[Snippet_youtube-iframe-embed-react]]"
  - "[[MusicCommunity_DB]]"
---

# Snippet_normalize-embed-url — 공유 링크 → embed URL 정규화

> **범용 패턴 메모** — Nest·Express·Next Server Action 어디든 **복사 가능** (의존성 없음).  
> **첫 적용:** music-community — `apps/api/src/recommendations/normalize-embed-url.ts` · POST 올리기 시 1회

```txt
이게 뭔가
  사용자가 붙여넣는 URL 형태는 제각각 (watch?v= · youtu.be · open.spotify.com/track/…)
  DB에는 iframe에 바로 넣을 수 있는 embed URL 한 형태로 통일해 두는 함수

언제 호출하나
  추천·게시글 "등록(POST)" 시점에 한 번 — 피드 조회마다 다시 돌리지 않음

이게 안 하는 것
  임베드 가능 여부 검사 (YouTube Data API embeddable) ❌
  올리기 거절 ❌ — 막힌 영상도 글은 저장, 재생은 Web fallback ([[Snippet_youtube-iframe-embed-react]])
```

---

## 꼭 기억할 것

| 주제 | 채택 ✅ | 피하기 ❌ |
| --- | --- | --- |
| **시점** | **저장(쓰기) 시** normalize → DB `embedUrl` | 매 GET·피드 렌더마다 변환 |
| **입력** | `watch` · `youtu.be` · `/shorts/` · Spotify `/track/` 등 | embed 불가 URL POST 거절 |
| **출력** | `https://www.youtube.com/embed/{id}` · `open.spotify.com/embed/...` | 원본 그대로만 저장 (형태 섞임) |
| **실패** | 모르는 호스트 → `raw.trim()` 그대로 반환 | throw로 올리기 막기 |
| **Web과 역할** | Nest: **canonical embed URL 저장** | Web `parseYouTubeVideoId`: 썸네일·fallback용 id 추출 (비슷한 파싱, 목적 다름) |

---

## Nest vs Web — 역할 나누기

```txt
POST /recommendations
  └ normalizeEmbedUrl(dto.embedUrl)  → DB 저장

피드 FeedCardMedia
  └ parseYouTubeVideoId(embedUrl)     → 썸네일 · YT.Player videoId · watch?v= 링크
```

| | Nest `normalizeEmbedUrl` | Web `parseYouTubeVideoId` |
| --- | --- | --- |
| **목적** | DB에 넣을 **최종 embed URL** | **videoId**만 뽑기 |
| **Spotify** | `/track/id` → `/embed/track/id` | `embed` 경로에서 kind 파싱 |
| **중복** | YouTube host·path 분기가 비슷함 | 통합 패키지로 빼도 되나, 지금은 30줄이라 스니펫 복사로 충분 |

---

## 범용 함수 — `normalizeEmbedUrl`

> 파일명·경로는 자유. music-community는 `recommendations/normalize-embed-url.ts`.

```ts
/**
 * 사용자가 붙여넣은 음악 공유 URL → iframe용 embed URL로 통일.
 * POST(등록) 시 1회 호출해 DB에 저장. 임베드 가능 여부는 검사하지 않음.
 *
 * 지원: YouTube (watch · youtu.be · shorts · embed), Spotify (track/album/playlist/episode)
 * 미지원 호스트: trim만 하고 그대로 반환
 */
export function normalizeEmbedUrl(raw: string): string {
  const url = new URL(raw.trim());
  const host = url.hostname.replace(/^www\./, '');

  // youtu.be/VIDEO_ID
  if (host === 'youtu.be') {
    const id = url.pathname.slice(1).split('/')[0];
    if (id) return `https://www.youtube.com/embed/${id}`;
  }

  // youtube.com — embed / watch?v= / shorts
  if (host === 'youtube.com' || host === 'm.youtube.com') {
    if (url.pathname.startsWith('/embed/')) {
      const id = url.pathname.split('/')[2];
      if (id) return `https://www.youtube.com/embed/${id}`;
    }
    const watchId = url.searchParams.get('v');
    if (watchId) return `https://www.youtube.com/embed/${watchId}`;
    const shortsId = url.pathname.match(/^\/shorts\/([^/?#]+)/)?.[1];
    if (shortsId) return `https://www.youtube.com/embed/${shortsId}`;
  }

  // 이미 Spotify embed URL
  if (host === 'open.spotify.com' && url.pathname.startsWith('/embed/')) {
    return url.toString();
  }

  // open.spotify.com/track|album|playlist|episode/ID → embed 경로
  const spotify = url.pathname.match(
    /^\/(track|album|playlist|episode)\/([^/?#]+)/,
  );
  if (host === 'open.spotify.com' && spotify) {
    return `https://open.spotify.com/embed/${spotify[1]}/${spotify[2]}`;
  }

  return raw.trim();
}
```

---

## Nest에서 쓰는 방법

```ts
// recommendations.service.ts — create 시
const normalizedEmbedUrl = normalizeEmbedUrl(embedUrl);

await this.prisma.recommendation.create({
  data: {
    // ...
    embedUrl: normalizedEmbedUrl,
  },
});
```

```txt
DTO에서는 @IsUrl() 등으로 http(s)만 검증
normalize는 서비스 레이어에서 — "형태 통일"은 비즈니스 규칙에 가깝고 Pipe보다 service가 읽기 쉬운 경우가 많음
```

---

## 입력 → 출력 예시

| 사용자 붙여넣기 | DB에 저장되는 형태 |
| --- | --- |
| `https://www.youtube.com/watch?v=dQw4w9WgXcQ` | `https://www.youtube.com/embed/dQw4w9WgXcQ` |
| `https://youtu.be/dQw4w9WgXcQ` | `https://www.youtube.com/embed/dQw4w9WgXcQ` |
| `https://youtube.com/shorts/abc123` | `https://www.youtube.com/embed/abc123` |
| `https://open.spotify.com/track/6rqhFgbbKwnb9MLmUQDhG6` | `https://open.spotify.com/embed/track/6rqhFgbbKwnb9MLmUQDhG6` |
| `https://example.com/audio.mp3` | `https://example.com/audio.mp3` (그대로) |

---

## VS Code User Snippet (선택)

`typescript.json` — prefix 예: `norm-embed`

```json
"normalizeEmbedUrl": {
  "prefix": "norm-embed-url",
  "body": [
    "/**",
    " * 공유 URL → embed URL (POST 시 DB 저장용). 임베드 가능 여부는 검사하지 않음.",
    " */",
    "export function normalizeEmbedUrl(raw: string): string {",
    "  const url = new URL(raw.trim());",
    "  const host = url.hostname.replace(/^www\\./, '');",
    "  $0",
    "  return raw.trim();",
    "}"
  ],
  "description": "YouTube/Spotify embed URL normalize (skeleton)"
}
```

등록 방법 → [[VSCode_User_Snippets]]

---

## 프로젝트별로 바꿀 것

| 항목 | music-community 예 |
| --- | --- |
| 호출 위치 | `RecommendationsService.create` |
| DB 필드 | `Recommendation.embedUrl` |
| Web 재생 | [[Snippet_youtube-iframe-embed-react]] · `FeedCardMedia` |
| Apple Music | ⬜ 스키마 확정 후 분기 추가 |

---

## Wiki와의 경계

| 여기 (Snippet) | Wiki / Project |
| --- | --- |
| 복붙 가능한 normalize 함수 | 제품 정책 「올리기 막지 않음」→ `30_Project` · 레포 `apps/docs` |
| POST 시 1회 | [[NestJS_Controller]] · [[NestJS_DTO]] |
| `embedUrl` 컬럼 | [[MusicCommunity_DB]] |
| Web `parseYouTubeVideoId` | [[Snippet_youtube-iframe-embed-react]] § 헬퍼 |
