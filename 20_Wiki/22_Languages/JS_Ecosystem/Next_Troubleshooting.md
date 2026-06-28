---
aliases:
  - Load failed
  - Next.js 오류 모음
  - Next.js 트러블슈팅
  - Safari 쿠키
tags:
  - React
  - NextJS
related:
  - "[[00_JS_Ecosystem_HomePage]]"
  - "[[NestJS_Deploy]]"
  - "[[NextJS_Concept]]"
  - "[[NextJS_Routing]]"
---
# Next_Troubleshooting — 실제 겪은 버그 & 경고 모음

# 한 줄 요약

```bash
Next.js + NestJS 모노레포(Artinerary) 개발·배포 중 실제로 만난 문제들
# 재사용 가능한 해결 패턴은 → [[NestJS_Deploy#모바일 Safari — same-site 프록시 패턴]] 참고
이 노트는 "어떤 증상이었고 무엇이 원인이었는지" 기록 중심
```

---

---

# 📱 배포 환경 — 모바일 Safari 버그

## 전체 아키텍처 (문제 발생 당시)

```txt
[Browser]
    ↓ credentials (httpOnly JWT)
[Next.js web :3001]
    │  prod: 브라우저 → /api/nest (same-site 프록시)
    │  서버(RSC) → API_INTERNAL_URL 로 Railway 직접
    └─fetch──→  [Nest API :3000]
                    ↓
               [PostgreSQL]
                    ↑
[문화 API] ←── Collector (ADMIN) ────┘
[Claude API] ← ExhibitionAiService (ADMIN edit)
```

## 버그 1 — 모바일만 로그인·찜 안 됨 (401) ⭐️

|항목|내용|
|---|---|
|**증상**|회원가입·로그인 API 응답은 200(성공)인데, 헤더 닉네임 표시·찜하기·`GET /auth/me` 가 실패. **PC 브라우저는 전혀 문제없음**|
|**원인**|`NEXT_PUBLIC_API_URL` 이 Railway **절대 URL** → 브라우저가 Vercel(프론트)과 Railway(API) 라는 다른 도메인에 직접 요청 → iOS Safari 가 서드파티(cross-site) 쿠키로 인식해서 차단|
|**조치**|Vercel 환경변수를 `NEXT_PUBLIC_API_URL=/api/nest`(상대경로) 로 변경, 실제 Railway 주소는 `API_INTERNAL_URL` 로 분리. 재배포|
|**확인**|iPhone Safari 로그인 후 → 쿠키가 **vercel.app 도메인**에 `artinerary-auth-token` 으로 있는지|

```txt
왜 헷갈렸나:
  FRONTEND_URL 도 맞고 credentials: 'include' 도 이미 있었는데
  모바일에서만 안 됐던 이유 → CORS 설정과 iOS Safari 의 ITP(서드파티 쿠키 차단)는
  서로 다른 차원의 문제였기 때문
  CORS 헤더가 맞다고 → 쿠키가 저장/전송된다는 보장은 없음

근본 해결책 → [[NestJS_Deploy#모바일 Safari — same-site 프록시 패턴]]
  브라우저가 cross-origin 요청 자체를 안 하도록 프록시로 우회
```

## 버그 2 — 마이페이지 목록만 "Load failed" ⭐️

|항목|내용|
|---|---|
|**증상**|찜하기·관람 후기 **등록은 정상**. `/mypage/wishlist`, `/mypage/visits` **조회만** "Load failed" (Safari 의 fetch 네트워크 단계 실패 메시지)|
|**헷갈렸던 부분**|포스터/`photoUrl` 이미지 깨짐과는 다른 증상 — API 응답 자체가 도착 못 한 것|
|**원인**|버그1 해결용 프록시(`route.ts`)가, Railway 응답을 Node `fetch` 가 **이미 압축 해제**했는데 `content-encoding: gzip` 헤더는 그대로 브라우저에 전달 → 헤더-바디 불일치 → 응답 큰 목록 API 에서 iOS Safari 가 디코딩 실패|
|**조치**|프록시에서: upstream 요청 시 `accept-encoding` 제거 / 응답 전달 시 `content-encoding`·`content-length`·`transfer-encoding` 제거 / body `arrayBuffer()` 로 재전달 / `dynamic = 'force-dynamic'`|
|**확인**|Network 탭 `GET /api/nest/me/wishlist` status **200** + 정상 JSON 배열|

```txt
왜 헷갈렸나:
  "등록은 되는데 조회만 안 됨" → 인증 문제(버그1 재발)로 오해하기 쉬움
  실제론 인증과 무관, 헤더·바디 불일치 문제
  응답 작은 API(로그인 등)는 우연히 안 드러나고 → 응답 큰 목록 API 에서만 증상

왜 Node fetch 가 "이미 풀어버리는지":
  Node 의 fetch 는 압축 응답을 자동으로 해제해서 body 를 줌
  하지만 원본 응답 헤더(content-encoding) 는 그대로 유지
  → 검사 없이 그대로 복사 전달하면 "헤더는 압축, 실제는 이미 풀림" 불일치
  → Chrome 은 관대하게 처리 / iOS Safari 는 엄격하게 처리해서 실패
```

```txt
에러 문구로 원인 가늠하기:
  "Load failed"                      → fetch 자체 실패 (프록시·압축 헤더 의심)
  "목록을 불러오지 못했습니다."(앱 메시지) → catch 블록의 일반 네트워크 에러
  "Unauthorized" / "로그인이 필요합니다."  → 쿠키·인증 문제 (버그1 계열)
```

---

---

# 📦 로컬 개발 — pnpm 모노레포 / lockfile

## multiple lockfiles 경고 ⭐️

|항목|내용|
|---|---|
|**증상**|`multiple lockfiles` 경고 — 루트와 `web/` 양쪽에 `pnpm-workspace.yaml` 존재|
|**원인**|`web/` 은 루트 repo 안의 하위 패키지인데 자체 workspace 파일을 따로 갖고 있어서 lockfile 충돌|

```txt
경고 메시지 예:
  multiple lockfiles
  - /Users/gong/artinerary/pnpm-workspace.yaml      (루트)
  - /Users/gong/artinerary/web/pnpm-workspace.yaml  (web)
```

### 해결 1 — turbopack.root 지정 (가장 빠름)

```typescript
// web/next.config.ts
import type { NextConfig } from 'next';

const nextConfig: NextConfig = {
  turbopack: {
    root: __dirname,   // web/ 폴더를 번들러 루트로 지정
  },
};

export default nextConfig;
```

```txt
web/ 안에서 파일 찾을 때 __dirname(web 폴더) 기준으로 지정
pnpm-workspace.yaml 은 그대로 두되 경고만 없앰
```

### 해결 2 — web/pnpm-workspace.yaml 삭제 (깔끔)

```bash
rm web/pnpm-workspace.yaml
```

```txt
web 은 루트 repo 의 하위 패키지 → 자체 workspace 파일 불필요
삭제하면 lockfile 하나로 정리됨
단, pnpm install 재실행으로 lockfile 구조 재정리 필요할 수 있음
```

### 해결 3 — 루트 workspace 에 web 등록 (장기 / 권장)

```yaml
# 루트 pnpm-workspace.yaml
packages:
  - 'web'
```

```bash
# 1. web 의 lockfile / workspace 파일 삭제
rm web/pnpm-lock.yaml
rm web/pnpm-workspace.yaml

# 2. 루트에서 재설치 (web 의존성도 루트 lock 에 합침)
pnpm install
```

```txt
왜 이 파일들을 지우나:
  web/pnpm-lock.yaml      lockfile 은 루트 하나만 있어야 함 (버전 충돌 방지)
  web/pnpm-workspace.yaml workspace 정의는 루트만 (하위 패키지는 자체 정의 불필요)

루트에서 pnpm install 하면:
  루트 pnpm-workspace.yaml 의 packages 에 web 이 있으므로
  web/ 의존성도 루트 lockfile 에 통합됨
  이후 루트에서 pnpm dev:web 으로 web 실행 가능

→ 구조가 가장 깔끔, 장기적으로 패키지가 늘어날 때 적합
```

### 세 가지 해결법 비교

|상황|선택|
|---|---|
|지금 당장 경고만 없애고 싶다|해결 1 — `turbopack.root`|
|web 구조를 정리하고 싶다|해결 2 — `web/pnpm-workspace.yaml` 삭제|
|모노레포로 장기 운영할 계획|해결 3 — 루트 workspace 등록 + lockfile 통합|

---

---

# 🔧 로컬 개발 — TypeScript / tsconfig

## "Cannot use JSX unless the '--jsx' flag is provided" ⭐️

|항목|내용|
|---|---|
|**증상**|`Cannot use JSX unless the '--jsx' flag is provided` 에러가 수십 개(예: 68개) 한꺼번에 발생|
|**원인**|NestJS 루트 `tsconfig.json` 의 `exclude` 에 `web` 이 없음 → TypeScript 가 루트 아래 `.ts`/`.tsx` 전부를 검사 → `web/` 의 `.tsx` 파일도 Nest watcher 에 잡힘 → Nest 는 JSX 설정이 없으므로 에러|

```json
// 루트 tsconfig.json — 문제 상황
{
  "exclude": ["node_modules", "test", "dist", "**/*spec.ts"]
  // web 이 없음 → web/ 포함됨 → JSX 에러 발생
}
```

```json
// 루트 tsconfig.json — 수정
{
  "exclude": ["node_modules", "test", "dist", "**/*spec.ts", "web"]
  //                                                           ↑ 추가
}
```

```txt
설정        의미
web 없음    web/ 포함됨 → Nest 빌드에 .tsx 잡힘 → JSX 에러 대량 발생
web 있음    web/ 제외됨 → Nest 는 src/ 만 봄 → 정상

예: web/components/NotFoundMessage.tsx 가 Nest watcher 에 잡히면
   "jsx 플래그 없음" 에러가 그 파일 안의 JSX 문법 개수만큼 쏟아짐
→ exclude 에 "web" 추가하면 한 번에 해결
```

---

---

# 한눈에

```txt
📱 배포(모바일 Safari) 증상별 의심 순서:
  1. 로그인은 되는데 인증이 안 풀린다
     → NEXT_PUBLIC_API_URL 이 절대 URL(Railway)인지 상대경로(/api/nest)인지 확인
  2. 프록시 도입 후 일부 API만 "Load failed"
     → content-encoding 등 압축 관련 헤더를 그대로 복사하고 있는지 확인

📦 모노레포 lockfile 경고:
  빠른 봉합     → next.config.ts 의 turbopack.root
  구조 정리     → web/pnpm-workspace.yaml 삭제
  장기 운영     → 루트 workspace 에 web 등록 + lockfile 통합

🔧 NestJS 빌드에 JSX 에러 대량 발생:
  → 루트 tsconfig.json exclude 에 "web" 추가

공통 디버깅 팁:
  PC 에서 재현 안 되면 → 반드시 실제 iPhone Safari 로 확인
  Network 탭에서 응답 헤더와 실제 body 가 일치하는지 비교
```