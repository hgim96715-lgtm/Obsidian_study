---
aliases:
  - YouTube embed
  - YT.Player
  - IFrame Player API
tags:
  - Snippet
related:
  - "[[00_Tools_Ecosystem_HomePage]]"
  - "[[00_Project_HomePage]]"
  - "[[00_JS_Ecosystem_HomePage]]"
  - "[[TS_YouTube]]"
---

# Snippet_youtube-iframe-embed-react — YouTube IFrame API · React

> **범용 패턴 메모** — Next.js `'use client'` · 다른 React 앱에도 복사 가능.  
> **첫 적용:** music-community — `YouTubeFeedEmbed.tsx` · `EmbedPlaybackFallback.tsx` · `embedMedia.ts`

```txt
왜 plain <iframe>만 쓰면 안 되나
  임베드 막힌 영상 → YouTube 검은 에러 UI만 보임 · JS로 구분 어려움
  IFrame Player API + onError 101·150 → fallback UI로 전환 가능

세 층 (헷갈리지 말 것)
  1. iframe_api 스크립트     → 런타임 window.YT 생성
  2. @types/youtube (dev)    → TS에게 YT.Player · destroy() 설명
  3. 우리 헬퍼·컴포넌트       → watch URL · 101/150 판별 · UI
```

---

## 꼭 기억할 것

| 주제 | 채택 ✅ | 피하기 ❌ |
| --- | --- | --- |
| **준비 완료** | `onYouTubeIframeAPIReady` 에서만 `resolve(YT)` | `script.onload` 만으로 `resolve(win.YT)` (아직 YT 없을 수 있음) |
| **타입** | `pnpm add -D @types/youtube` · `useRef<YT.Player \| null>` | `type YTPlayer = { destroy() }` 손수 정의 |
| **임베드 막힘** | `onError` **101 · 150** → 외부 `watch?v=` 링크 | 서비스 「에러」 토스트 · 올리기 API 거절 |
| **정리** | unmount·접기 시 `playerRef.current?.destroy()` | Player 놔두고 재마운트만 반복 |
| **스크립트** | `document`에 `iframe_api` **한 번만** 추가 | 카드마다 script 태그 중복 |

---

## 공식 문서

| 문서 | URL |
| --- | --- |
| IFrame Player API 전체 | https://developers.google.com/youtube/iframe_api_reference |
| `onError` (101·150) | https://developers.google.com/youtube/iframe_api_reference#onError |
| 플레이어 로드 | https://developers.google.com/youtube/iframe_api_reference#Loading_a_Video_Player |
| 스크립트 URL | https://www.youtube.com/iframe_api |

| `onError` 코드 | 의미 |
| --- | --- |
| 101 | 업로더가 임베드 재생 불허 |
| 150 | 101과 동일 |
| 153 | Referer 등 클라이언트 식별 (WebView 등) — fallback 조건에 넣을지는 별도 판단 |

---

## 설치 (프로젝트마다)

```bash
pnpm add -D @types/youtube
```

---

## 1. 헬퍼 — `lib/youtubeEmbed.ts` (이름은 자유)

> music-community는 `lib/embedMedia.ts`에 같이 둠.

```ts
export function parseYouTubeVideoId(urlString: string): string | null {
  try {
    const url = new URL(urlString.trim());
    const host = url.hostname.replace(/^www\./, '');

    if (host === 'youtu.be') {
      const id = url.pathname.slice(1).split('/')[0];
      return id || null;
    }

    if (host === 'youtube.com' || host === 'm.youtube.com') {
      if (url.pathname.startsWith('/embed/')) {
        return url.pathname.split('/')[2] ?? null;
      }
      const watchId = url.searchParams.get('v');
      if (watchId) return watchId;
      const shortsId = url.pathname.match(/^\/shorts\/([^/?#]+)/)?.[1];
      if (shortsId) return shortsId;
    }
  } catch {
    return null;
  }
  return null;
}

export function youtubeWatchUrl(videoId: string): string {
  return `https://www.youtube.com/watch?v=${videoId}`;
}

export function youtubeThumbnailUrl(videoId: string): string {
  return `https://img.youtube.com/vi/${videoId}/hqdefault.jpg`;
}

/** 공식 onError 101·150 — 또는 YT.PlayerError enum 사용 가능 */
export function isYouTubeEmbedBlockedError(code: number): boolean {
  return code === 101 || code === 150;
}
```

---

## 2. 스크립트 로드 — `loadYoutubeIframeApi.ts`

```ts
/** iframe_api 한 번 로드 · onYouTubeIframeAPIReady에서만 resolve */
export function loadYoutubeIframeApi(): Promise<typeof YT> {
  const win = window as Window & {
    YT?: typeof YT;
    onYouTubeIframeAPIReady?: () => void;
  };

  if (win.YT?.Player) return Promise.resolve(win.YT);

  return new Promise((resolve, reject) => {
    const prev = win.onYouTubeIframeAPIReady;
    win.onYouTubeIframeAPIReady = () => {
      prev?.();
      if (win.YT?.Player) resolve(win.YT);
      else reject(new Error('YouTube IFrame API 로드 실패'));
    };

    if (
      !document.querySelector('script[src="https://www.youtube.com/iframe_api"]')
    ) {
      const script = document.createElement('script');
      script.src = 'https://www.youtube.com/iframe_api';
      script.async = true;
      document.head.appendChild(script);
    }
  });
}
```

---

## 3. 컴포넌트 — `YouTubeFeedEmbed.tsx`

```tsx
'use client';

import { isYouTubeEmbedBlockedError, parseYouTubeVideoId } from '@/lib/youtubeEmbed';
import { loadYoutubeIframeApi } from '@/lib/loadYoutubeIframeApi';
import { useEffect, useId, useRef } from 'react';

type YouTubeFeedEmbedProps = {
  embedUrl: string;
  title: string;
  /** onError 101·150 등 — 부모가 fallback UI로 전환 */
  onEmbedBlocked: () => void;
};

export function YouTubeFeedEmbed({
  embedUrl,
  title,
  onEmbedBlocked,
}: YouTubeFeedEmbedProps) {
  const playerRef = useRef<YT.Player | null>(null);
  const containerId = `yt-${useId().replace(/:/g, '')}`;

  useEffect(() => {
    const videoId = parseYouTubeVideoId(embedUrl);
    if (!videoId) {
      onEmbedBlocked();
      return;
    }

    let cancelled = false;

    loadYoutubeIframeApi()
      .then((YT) => {
        if (cancelled) return;
        playerRef.current = new YT.Player(containerId, {
          videoId,
          playerVars: { autoplay: 1, rel: 0 },
          events: {
            onError: (e) => {
              if (isYouTubeEmbedBlockedError(e.data)) onEmbedBlocked();
            },
          },
        });
      })
      .catch(() => {
        if (!cancelled) onEmbedBlocked();
      });

    return () => {
      cancelled = true;
      playerRef.current?.destroy();
      playerRef.current = null;
    };
  }, [containerId, embedUrl, onEmbedBlocked]);

  return (
    <div className="aspect-video w-full">
      <div id={containerId} title={title} className="h-full w-full" />
    </div>
  );
}
```

---

## 4. fallback UI (선택) — props 예시

```ts
type EmbedPlaybackFallbackProps = {
  thumbnailUrl?: string | null;
  message: string;
  externalHref: string;   // youtubeWatchUrl(videoId)
  externalLabel: string;  // "YouTube에서 듣기"
  onClose: () => void;    // 프로젝트 convention: onClose
};
```

부모 흐름:

```txt
▶ 클릭 → playing true
  ├ YouTubeFeedEmbed
  │    └ onEmbedBlocked → blocked true
  └ blocked ? Fallback + watch 링크 : Player
접기 → playing false · blocked false
```

---

## VS Code User Snippet (선택)

`typescriptreact.json` — prefix 예: `ytiframe`

```json
"YouTube loadIframeApi": {
  "prefix": "yt-load-iframe-api",
  "body": [
    "export function loadYoutubeIframeApi(): Promise<typeof YT> {",
    "  const win = window as Window & {",
    "    YT?: typeof YT;",
    "    onYouTubeIframeAPIReady?: () => void;",
    "  };",
    "  if (win.YT?.Player) return Promise.resolve(win.YT);",
    "  return new Promise((resolve, reject) => {",
    "    const prev = win.onYouTubeIframeAPIReady;",
    "    win.onYouTubeIframeAPIReady = () => {",
    "      prev?.();",
    "      if (win.YT?.Player) resolve(win.YT);",
    "      else reject(new Error('YouTube IFrame API unavailable'));",
    "    };",
    "    if (!document.querySelector('script[src=\"https://www.youtube.com/iframe_api\"]')) {",
    "      const script = document.createElement('script');",
    "      script.src = 'https://www.youtube.com/iframe_api';",
    "      script.async = true;",
    "      document.head.appendChild(script);",
    "    }",
    "  });",
    "}"
  ],
  "description": "YouTube IFrame API script loader (onYouTubeIframeAPIReady)"
}
```

등록 방법 → [[VSCode_User_Snippets]]

---

## 프로젝트별로 바꿀 것

| 항목 | music-community 예 |
| --- | --- |
| import 경로 | `@/lib/embedMedia` |
| fallback 컴포넌트 | `EmbedPlaybackFallback` · `brandPillBtn` |
| 부모 | `FeedCardMedia.tsx` |
| 제품 문서 | `apps/docs/youtube.md` (로컬) |

---

## Wiki와의 경계

| 여기 (Snippet) | Wiki / Project |
| --- | --- |
| 복붙 가능한 React·로더 패턴 | 제품 정책 「올리기 막지 않음」 |
| `@types/youtube` 설치 | music-community `overview` · `ui` §1 |
| Postman·DataGrip 아님 | DB `embedUrl` 컬럼 → [[MusicCommunity_DB]] |
