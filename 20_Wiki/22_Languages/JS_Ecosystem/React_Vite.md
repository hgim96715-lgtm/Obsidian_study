---
aliases:
  - 빌드도구
  - CRA
  - create-react-app
  - Vite
tags:
  - React
related:
  - "[[00_JS_Ecosystem_HomePage]]"
  - "[[NextJS_Concept]]"
  - "[[React_Concept]]"
---
# React_Vite — Vite 빌드 도구

# 한 줄 요약

```txt
Vite = React(또는 다른 프레임워크) 프로젝트를 빠르게 개발·빌드하는 도구
Next.js 와 달리 SSR/라우팅이 기본 내장되지 않은 "순수 빌드 도구 + 개발 서버"
```

---

---

# Vite 가 뭔가 ⭐️

```txt
Vite(프랑스어로 "빠르다") 는 프론트엔드 빌드 도구

이전 세대(Webpack, Create React App):
  개발 서버 켤 때 프로젝트 전체를 한 번에 번들링(bundle)
  → 프로젝트가 커질수록 서버 시작이 느려짐

Vite 방식:
  브라우저의 네이티브 ES 모듈(import/export) 기능을 그대로 활용
  개발 중에는 번들링 없이 필요한 파일만 그때그때 변환해서 제공
  → 프로젝트 크기와 거의 무관하게 빠른 시작 속도

빌드(배포용)할 때는:
  Rollup 기반으로 번들링 → 운영 환경에 최적화된 결과물 생성
```

---

---

# Vite vs Next.js vs CRA ⭐️

```bash
CRA (Create React App):
  과거 React 공식 시작 도구였지만 2025년 공식 deprecated(지원 종료)
  → 현재는 신규 프로젝트에 권장되지 않음

Vite + React:
  순수 SPA(Single Page Application) 만들 때 적합
  라우팅(react-router 등) / SSR 직접 추가해야 함
  빠른 개발 서버 / 빌드 속도가 강점
  백엔드(NestJS 등)와 분리된 프론트엔드 전용 프로젝트에 자주 사용

Next.js:
  React 기반 "풀스택 프레임워크"
  SSR / SSG / 폴더 기반 라우팅 / API Route 가 기본 내장


언제 무엇을 쓰나:
  SEO 필요 / 서버 렌더링 필요 / API Route 도 같이 쓰고 싶다  → Next.js
  단순 SPA / 관리자 대시보드 / 백엔드(NestJS)가 이미 따로 있다 → Vite + React
  Artinerary 처럼 NestJS 백엔드가 이미 있는 구조라면
  → 프론트도 Vite 로 가볍게 갈지, Next.js 로 SSR 이점을 가져갈지가 선택 포인트
```

---

---

# 프로젝트 생성

```bash
pnpm create vite my-app --template react-ts
cd my-app
pnpm install
pnpm dev
```

```txt
--template react-ts:
  React + TypeScript 템플릿
  다른 옵션: react (JS), vue, svelte, vanilla 등 다양한 프레임워크 지원

pnpm dev 실행 시:
  Vite 개발 서버가 즉시 켜짐 (Next.js 의 next dev 와 비슷한 역할)
  기본 포트 5173
```

---

---

# 프로젝트 구조

```txt
my-app/
├── index.html       ← Vite 는 HTML 이 진입점 (Next.js 는 폴더 구조가 진입점)
├── src/
│   ├── main.tsx      ← React 앱을 index.html 에 마운트하는 시작점
│   ├── App.tsx
│   └── ...
├── vite.config.ts    ← Vite 설정 파일
└── package.json
```

```txt
index.html 이 진입점인 이유:
  Vite 는 프레임워크가 아니라 "빌드 도구"
  HTML 파일에서 <script type="module" src="/src/main.tsx"> 로 시작
  → Next.js 처럼 폴더 구조 자체가 라우팅이 되는 방식과 다름
```

---

---

# vite.config.ts — 기본 설정

```typescript
import { defineConfig } from 'vite';
import react from '@vitejs/plugin-react';

export default defineConfig({
  plugins: [react()],

  server: {
    port: 5173,
    proxy: {
      // 개발 중 NestJS API 로 요청 우회 (CORS 문제 회피)
      '/api': {
        target: 'http://localhost:3001',
        changeOrigin: true,
      },
    },
  },
});
```

```bash
plugins: [react()]:
  Vite 자체는 프레임워크에 종속되지 않음
  React 의 JSX 변환 등을 처리하려면 플러그인 필요

server.proxy 가 필요한 이유:
  프론트(Vite, localhost:5173) 와 백엔드(NestJS, localhost:3001) 가 다른 포트
  → 그냥 fetch('/api/...') 하면 CORS 에러 날 수 있음
  proxy 설정하면 Vite 가 /api 로 시작하는 요청을 NestJS 로 대신 전달
  → 개발 중엔 마치 같은 origin 인 것처럼 동작 (운영 배포 시엔 별도 CORS 설정 필요)
```

---

---

# 환경변수 — import.meta.env ⭐️

```bash
# .env (Vite 프로젝트 루트)
VITE_API_URL=http://localhost:3001
```

```typescript
// 코드에서 사용
const apiUrl = import.meta.env.VITE_API_URL;
```

```txt
⚠️ Next.js 의 process.env 와 다름:
  Vite        → import.meta.env.VITE_접두사
  Next.js     → process.env.NEXT_PUBLIC_접두사 (클라이언트 노출용)

  접두사 필수:
    VITE_ 로 시작하는 변수만 클라이언트(브라우저) 코드에 노출됨
    VITE_ 없는 변수는 빌드 결과물에 포함 안 됨 (보안)
    → API_SECRET_KEY 같은 건 절대 VITE_ 접두사 붙이면 안 됨 (노출됨)
```

---

---

# 빌드 & 배포

```bash
pnpm build     # dist/ 폴더에 정적 파일 생성
pnpm preview   # 빌드 결과물 로컬에서 미리보기
```

```txt
빌드 결과물(dist/) 은 순수 정적 파일(HTML/JS/CSS):
  Vercel / Netlify / Cloudflare Pages 같은 정적 호스팅에 바로 배포 가능
  Next.js 처럼 Node.js 서버 실행이 필수가 아님 (SSR 안 쓰는 경우)

  → [[Next_Deploy]] 의 Vercel 배포와 달리
    Vite 결과물은 어떤 정적 파일 호스팅에도 올릴 수 있어 더 자유로움
```

---

---
# Docker로 실행 ⭐️

```txt
Vite 도 Docker 로 실행 가능
개발용(Dockerfile.dev) 과 배포용(Dockerfile) 을 분리하는 게 일반적
```

## 개발 모드 ⭐️

```dockerfile
# Dockerfile.dev
FROM node:20-alpine
WORKDIR /app
COPY package*.json ./
RUN npm install
COPY . .
EXPOSE 5173
CMD ["npm", "run", "dev", "--", "--host"]
```

```bash
docker build -f Dockerfile.dev -t my-app-dev .
docker run -p 5173:5173 -v $(pwd):/app -v /app/node_modules my-app-dev
```

```txt
--host 가 필요한 이유 ⭐️:
  Vite 개발 서버는 기본적으로 localhost(컨테이너 내부) 에만 바인딩됨
  컨테이너 밖(호스트 PC 브라우저) 에서 접근하려면
  0.0.0.0 으로 열어줘야 함 → --host 옵션이 그 역할
  (이거 안 하면 포트는 열려있는데 브라우저에서 안 들어가짐 ⚠️)

-v $(pwd):/app 볼륨 마운트:
  로컬에서 코드 수정 → 컨테이너 안에도 즉시 반영 (HMR 동작)
  컨테이너 안에서만 다시 빌드할 필요 없음

-v /app/node_modules (익명 볼륨):
  호스트의 node_modules 가 컨테이너 안의 것을 덮어쓰지 않게 보호
  (OS 다르면 native 모듈이 깨질 수 있어서 분리)
```

## 배포용 — 멀티스테이지 빌드 ⭐️

```dockerfile
# Dockerfile
# 1단계: 빌드만 담당
FROM node:20-alpine AS build
WORKDIR /app
COPY package*.json ./
RUN npm install
COPY . .
RUN npm run build

# 2단계: 정적 파일만 nginx 로 서빙
FROM nginx:alpine
COPY --from=build /app/dist /usr/share/nginx/html
EXPOSE 80
```

```bash
docker build -t my-app .
docker run -p 8080:80 my-app
```

```txt
왜 멀티스테이지로 나누나 ⭐️:
  Vite 빌드 결과물(dist/) 은 순수 정적 파일(HTML/JS/CSS)
  Node.js 런타임이 운영에는 필요 없음
  → 1단계(build) 에서 Node 로 빌드만 하고
    2단계(nginx) 에는 결과물만 복사 → node_modules·소스코드 전부 제외
  → 최종 이미지 크기가 훨씬 작아짐 (보통 수백 MB → 수십 MB)

COPY --from=build:
  이전 스테이지(build)의 결과물 중 /app/dist 만 가져옴
  나머지(빌드 도구, 캐시 등)는 최종 이미지에 안 남음
```

## API 서버(NestJS)와 같이 docker-compose ⭐️

```yaml
# docker-compose.yml
services:
  frontend:
    build:
      context: ./web
      dockerfile: Dockerfile.dev
    ports:
      - '5173:5173'
    volumes:
      - ./web:/app
      - /app/node_modules

  backend:
    build: ./api
    ports:
      - '3001:3001'
    environment:
      - DATABASE_URL=postgresql://...
```

```txt
이렇게 묶으면:
  docker compose up 한 번으로 프론트(Vite) + 백엔드(NestJS) 동시 실행
  서비스 이름(backend)으로 컨테이너 간 통신 가능
  → Vite 의 server.proxy target 을 'http://backend:3001' 로 설정 가능
    (컨테이너 안에서는 localhost 대신 서비스 이름으로 접근)
```

---

---

# 한눈에

```txt
Vite       빌드 도구 + 개발 서버 (빠름) / SSR·라우팅 직접 추가
Next.js    풀스택 프레임워크 / SSR·라우팅·API Route 내장

생성:        pnpm create vite my-app --template react-ts
설정 파일:    vite.config.ts
환경변수:     import.meta.env.VITE_접두사 (Next.js 의 NEXT_PUBLIC_ 과 유사한 역할)
개발 중 API:  server.proxy 로 CORS 우회
빌드 결과:    dist/ (정적 파일) → 어디든 호스팅 가능

선택 기준:
  SEO/SSR 필요 + API Route 도 같이      → Next.js
  순수 SPA + 백엔드(NestJS)가 이미 따로 있음 → Vite

Docker:
  개발용  CMD npm run dev -- --host  (0.0.0.0 바인딩 필수)
  배포용  멀티스테이지 (build → nginx) → dist/ 만 최종 이미지에 포함
```









