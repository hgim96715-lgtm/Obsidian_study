---
aliases:
  - YouTube
  - IFrameAPI
tags:
  - TypeScript
related:
  - "[[00_JS_Ecosystem_HomePage]]"
  - "[[JS_DOM]]"
  - "[[JS_Promise]]"
  - "[[JS_OptionalChaining]]"
  - "[[React_useRef]]"
  - "[[Snippet_youtube-iframe-embed-react]]"
---
# TS_YouTube — YouTube IFrame API 타입

> [!info]
>  `@types/youtube`는 YouTube IFrame Player API의 TypeScript 타입 선언 패키지
>   `window.YT`, `YT.Player`, `YT.PlayerState` 등에 타입이 붙어 자동완성과 타입 체크가 가능

---

# 설치

```bash
pnpm add -D @types/youtube
```

```txt
@types/youtube:
  YouTube가 공식 제공하는 타입이 아니라 DefinitelyTyped 커뮤니티 패키지
  devDependency로 설치 — 런타임에 실제 코드가 포함되는 게 아니라 타입 정보만 제공

설치 후 tsconfig.json에 별도 추가 없이 자동으로 전역 타입으로 인식됨
  → YT.Player / YT.PlayerState 등을 import 없이 바로 사용 가능
```

---

# window 타입 확장 — ambient declaration ⭐️⭐️⭐️⭐️

```txt
@types/youtube는 YT 네임스페이스를 전역으로 선언하지만,
window.YT / window.onYouTubeIframeAPIReady는 별도로 선언해줘야 타입 에러가 안 남
```

```typescript
// types/youtube.d.ts  (또는 global.d.ts)
declare global {
  interface Window {
    YT: typeof YT;
    onYouTubeIframeAPIReady: (() => void) | undefined;
  }
}

export {};  // 이 파일을 모듈로 만들어야 declare global이 동작함
```

```txt
export {}가 왜 필요한가:
  TypeScript에서 import/export 문이 없는 파일은 "스크립트"로 취급 — declare global이 필요 없음
  import/export 문이 하나라도 있으면 "모듈"로 취급 — declare global로 전역 확장 가능
  아무것도 export 안 해도 export {}로 "나는 모듈이다"를 선언할 수 있음

window.onYouTubeIframeAPIReady:
  YouTube 스크립트가 로드 완료되면 이 전역 콜백을 호출함
  처음엔 없다가(undefined) 나중에 할당되는 구조 → | undefined 포함
  (prev?.() 패턴으로 기존 콜백 보존 → [[JS_OptionalChaining]] 참고)
```

---

# YT.Player — 플레이어 생성 ⭐️⭐️⭐️⭐️

## 생성자

```typescript
const player = new YT.Player(elementOrId, options);
```

```typescript
// 기본 예시
const player = new YT.Player('player-container', {
  videoId:    'dQw4w9WgXcQ',
  width:      '100%',
  height:     '360',
  playerVars: {
    autoplay:   0,         // 0: 수동 재생 / 1: 자동 재생
    controls:   1,         // 0: 컨트롤 숨김 / 1: 표시
    rel:        0,         // 0: 관련 동영상 숨김 / 1: 표시
    modestbranding: 1,     // YouTube 로고 최소화
    playsinline:    1,     // iOS에서 인라인 재생 (전체화면 자동 전환 방지)
  },
  events: {
    onReady:       (e) => e.target.playVideo(),
    onStateChange: (e) => handleStateChange(e.data),
    onError:       (e) => console.error('YouTube 에러:', e.data),
  },
});
```

## 첫 번째 인자 — elementOrId

```typescript
// 방법 1: id 문자열 (해당 id를 가진 요소를 YouTube가 <iframe>으로 교체)
new YT.Player('player-container', options);

// 방법 2: HTMLElement (React의 ref.current 등)
const ref = useRef<HTMLDivElement>(null);
new YT.Player(ref.current!, options);
```

```txt
방법 2 사용 시 주의:
  ref.current가 null일 수 있어서 ! (non-null assertion) 또는 null 체크 필요
  React에서는 useRef + useEffect 조합으로 마운트 후에 생성해야 안전
  (ref 패턴 → [[React_useRef]] 참고)
```

---

# YT.PlayerState — 재생 상태 ⭐️⭐️⭐️

```txt
YT.PlayerState는 숫자 enum — onStateChange 이벤트의 e.data 값을 이 상수로 비교
```

|상수|값|의미|
|---|---|---|
|`YT.PlayerState.UNSTARTED`|`-1`|초기화됨, 아직 재생 안 함|
|`YT.PlayerState.ENDED`|`0`|재생 완료|
|`YT.PlayerState.PLAYING`|`1`|재생 중|
|`YT.PlayerState.PAUSED`|`2`|일시 정지|
|`YT.PlayerState.BUFFERING`|`3`|버퍼링 중|
|`YT.PlayerState.CUED`|`5`|큐에 올라옴 (재생 대기)|

```typescript
const handleStateChange = (state: number) => {
  switch (state) {
    case YT.PlayerState.PLAYING:
      console.log('재생 시작');
      break;
    case YT.PlayerState.PAUSED:
      console.log('일시 정지');
      break;
    case YT.PlayerState.ENDED:
      console.log('재생 완료');
      break;
  }
};
```

---

# 이벤트 타입 ⭐️⭐️

|이벤트|타입|설명|
|---|---|---|
|`onReady`|`YT.PlayerEvent`|플레이어 준비 완료|
|`onStateChange`|`YT.OnStateChangeEvent`|재생 상태 변경 (`e.data`에 PlayerState 값)|
|`onError`|`YT.OnErrorEvent`|에러 발생 (`e.data`에 에러 코드)|
|`onPlaybackQualityChange`|`YT.OnPlaybackQualityChangeEvent`|화질 변경|
|`onPlaybackRateChange`|`YT.OnPlaybackRateChangeEvent`|재생 속도 변경|

```typescript
// 이벤트 핸들러 타입 명시
const onReady = (event: YT.PlayerEvent) => {
  const player = event.target;   // YT.Player 인스턴스
  player.playVideo();
};

const onStateChange = (event: YT.OnStateChangeEvent) => {
  const state: number = event.data;   // YT.PlayerState 값
};
```

---

# YT.Player 주요 메서드 ⭐️⭐️⭐️

## 재생 제어

|메서드|설명|
|---|---|
|`player.playVideo()`|재생|
|`player.pauseVideo()`|일시 정지|
|`player.stopVideo()`|정지 (버퍼 초기화)|
|`player.seekTo(seconds, allowSeekAhead)`|특정 시점으로 이동|
|`player.mute()` / `player.unMute()`|음소거 / 해제|
|`player.setVolume(volume)`|볼륨 설정 (0~100)|

## 동영상 로드

|메서드|설명|
|---|---|
|`player.loadVideoById(videoId)`|새 동영상 로드 + 즉시 재생|
|`player.cueVideoById(videoId)`|새 동영상 로드 (재생 안 함)|

## 상태 조회

|메서드|반환|
|---|---|
|`player.getPlayerState()`|`YT.PlayerState` 값 (number)|
|`player.getCurrentTime()`|현재 재생 시간 (초)|
|`player.getDuration()`|전체 길이 (초)|
|`player.getVolume()`|현재 볼륨 (0~100)|
|`player.isMuted()`|boolean|

## 정리

```typescript
player.destroy()   // 플레이어 인스턴스 제거 + iframe 삭제
```

---

# React에서 사용하는 전체 패턴 ⭐️⭐️⭐️⭐️

```typescript
// hooks/useYouTubePlayer.ts
import { useEffect, useRef } from 'react';

// YouTube API를 한 번만 로드하는 함수
function loadYouTubeAPI(): Promise<typeof YT> {
  // 이미 로드됐으면 바로 반환 ([[JS_Promise]] Promise.resolve 패턴)
  if (window.YT?.Player) return Promise.resolve(window.YT);

  return new Promise((resolve, reject) => {
    const prev = window.onYouTubeIframeAPIReady;  // 기존 콜백 보존

    window.onYouTubeIframeAPIReady = () => {
      prev?.();                                    // ([[JS_OptionalChaining]] prev?.() 패턴)
      if (window.YT?.Player) resolve(window.YT);
      else reject(new Error('YouTube IFrame API를 사용할 수 없습니다.'));
    };

    // 스크립트가 없을 때만 추가 ([[JS_DOM]] 동적 스크립트 로드 패턴)
    if (!document.querySelector('script[src="https://www.youtube.com/iframe_api"]')) {
      const script = document.createElement('script');
      script.src   = 'https://www.youtube.com/iframe_api';
      script.async = true;
      document.body.appendChild(script);
    }
  });
}

export function useYouTubePlayer(videoId: string) {
  const containerRef = useRef<HTMLDivElement>(null);
  const playerRef    = useRef<YT.Player | null>(null);

  useEffect(() => {
    if (!containerRef.current) return;

    let cancelled = false;

    loadYouTubeAPI().then((yt) => {
      if (cancelled || !containerRef.current) return;

      playerRef.current = new yt.Player(containerRef.current, {
        videoId,
        playerVars: { controls: 1, rel: 0, playsinline: 1 },
        events: {
          onStateChange: (e) => {
            if (e.data === YT.PlayerState.ENDED) {
              console.log('재생 완료');
            }
          },
        },
      });
    });

    return () => {
      cancelled = true;
      playerRef.current?.destroy();  // 언마운트 시 정리
      playerRef.current = null;
    };
  }, [videoId]);

  return containerRef;
}
```

```tsx
// 사용
export function VideoPlayer({ videoId }: { videoId: string }) {
  const containerRef = useYouTubePlayer(videoId);
  return <div ref={containerRef} />;
}
```

```txt
cancelled 플래그:
  비동기 로드 중 컴포넌트가 언마운트되면 player를 생성하면 안 됨
  → useEffect cleanup에서 cancelled = true로 설정
  → then 콜백 안에서 cancelled 확인 후 조기 종료

playerRef.current?.destroy():
  .destroy() 호출 전 null 체크 — ?.으로 안전하게 처리 ([[JS_OptionalChaining]] 참고)
```

---

# playerVars 주요 옵션

|옵션|값|설명|
|---|---|---|
|`autoplay`|`0` / `1`|자동 재생 (모바일은 정책상 대부분 차단)|
|`controls`|`0` / `1` / `2`|컨트롤 바 숨김 / 표시|
|`rel`|`0` / `1`|재생 후 관련 동영상 표시 여부|
|`loop`|`0` / `1`|반복 재생|
|`mute`|`0` / `1`|음소거 (autoplay와 함께 쓸 때 필요)|
|`playsinline`|`0` / `1`|iOS에서 인라인 재생 (1 권장)|
|`modestbranding`|`0` / `1`|YouTube 로고 최소화|
|`start`|number|시작 시간 (초)|
|`end`|number|종료 시간 (초)|
|`cc_lang_pref`|`'ko'` 등|자막 언어 기본값|

---

# 한눈에

```txt
설치: pnpm add -D @types/youtube

window 타입 확장 (global.d.ts):
  interface Window { YT: typeof YT; onYouTubeIframeAPIReady: (() => void) | undefined; }
  export {}  ← 모듈로 만들어야 declare global이 동작

YT.Player 생성:
  new YT.Player(elementOrId, { videoId, playerVars, events })
  React에서는 useRef + useEffect 조합으로 마운트 후 생성

YT.PlayerState (숫자 enum):
  UNSTARTED(-1) / ENDED(0) / PLAYING(1) / PAUSED(2) / BUFFERING(3) / CUED(5)

주요 메서드:
  playVideo / pauseVideo / stopVideo / seekTo
  loadVideoById / cueVideoById
  destroy → 언마운트 시 반드시 호출

패턴 연결:
  API 로드 → [[JS_Promise]] (new Promise + Promise.resolve early return)
  prev?.() → [[JS_OptionalChaining]] (기존 콜백 보존)
  동적 스크립트 삽입 → [[JS_DOM]] (createElement + querySelector 중복 확인)
  ref + useEffect → [[React_useRef]]
```