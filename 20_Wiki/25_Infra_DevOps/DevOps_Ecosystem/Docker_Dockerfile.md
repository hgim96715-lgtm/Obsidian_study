---
aliases:
  - Docker
  - Dockerfile
  - 이미지 빌드
tags:
  - DevOps
  - Docker
related:
  - "[[00_DevOps_HomePage]]"
  - "[[Docker_Compose]]"
  - "[[Deploy_CloudMVP]]"
---
# Docker_Dockerfile — 이미지 빌드

> [!info]
>  Dockerfile = 컨테이너 이미지를 만드는 레시피
>  어떤 OS 위에서, 어떤 파일을 넣고, 어떤 명령어를 실행해서 앱을 실행할지 정의한다. 멀티스테이지 빌드로 "빌드 환경"과 "실행 환경"을 분리하면 이미지 크기를 크게 줄일 수 있다.

---

# 명령어 한눈에

|명령어|역할|레이어 생성|
|---|---|---|
|`FROM`|베이스 이미지 지정|✅|
|`WORKDIR`|작업 디렉토리 설정 (없으면 자동 생성)|✅|
|`COPY`|호스트 파일 → 이미지로 복사|✅|
|`ADD`|COPY + URL/tar 압축 해제 지원 (COPY 권장)|✅|
|`RUN`|이미지 빌드 시 명령어 실행|✅|
|`ENV`|환경 변수 설정 (빌드 + 런타임)|✅|
|`ARG`|빌드 시점에만 사용하는 변수 (`--build-arg`로 전달)|❌|
|`EXPOSE`|문서용 포트 명시 (실제 포트 오픈 아님)|❌|
|`CMD`|컨테이너 시작 시 기본 명령 (덮어쓰기 가능)|❌|
|`ENTRYPOINT`|컨테이너 시작 시 고정 명령 (덮어쓰기 어려움)|❌|

```txt
레이어 생성:
  Dockerfile 명령어 중 레이어를 만드는 것들은 Docker가 캐싱함
  이전 빌드와 같은 레이어면 다시 실행하지 않고 캐시 사용 → 빌드 빠름
  COPY한 파일이 바뀌면 그 이후 레이어는 전부 캐시 무효화 → 다시 실행
```

---

# 기본 흐름 ⭐️⭐️⭐️

```dockerfile
# 1. 베이스 이미지
FROM node:22-alpine

# 2. 작업 디렉토리 설정
WORKDIR /app

# 3. 의존성 파일만 먼저 복사 (캐시 활용)
COPY package.json package-lock.json ./

# 4. 의존성 설치
RUN npm ci --only=production

# 5. 소스 코드 복사
COPY . .

# 6. 빌드
RUN npm run build

# 7. 포트 문서화 (실제 오픈 아님)
EXPOSE 3000

# 8. 실행
CMD ["node", "dist/main"]
```

```txt
package.json을 먼저 복사하는 이유 (캐시 최적화):
  소스 코드는 자주 바뀌지만 package.json은 상대적으로 안 바뀜
  COPY . . 하나로 합치면 소스 코드가 바뀔 때마다 npm install도 다시 실행됨 → 느림

  COPY package.json ./  →  RUN npm install  →  COPY . .
  └ package.json 안 바뀌면 npm install 캐시 사용          └ 소스코드만 다시 COPY
  → 소스 코드를 아무리 바꿔도 npm install은 캐시에서 가져옴 → 빌드 빠름
```

---

# CMD vs ENTRYPOINT ⭐️⭐️⭐️⭐️

```txt
둘 다 컨테이너 시작 시 실행할 명령을 정의하지만 동작이 다름
```

| |`CMD`|`ENTRYPOINT`|
|---|---|---|
|역할|기본 명령 (덮어쓰기 가능)|고정 명령 (덮어쓰기 어려움)|
|`docker run 이미지 명령어`|CMD 무시하고 새 명령 실행|ENTRYPOINT는 유지, 인자로 추가됨|
|용도|기본값이 있지만 바꿀 수 있게|항상 같은 프로그램으로 고정|
|예시|개발/운영 모드 전환|웹 서버, API 서버|

```dockerfile
# CMD 단독 — docker run 시 명령어로 덮어쓰기 가능
CMD ["node", "dist/main"]
# docker run 이미지 node dist/other → CMD 무시하고 other 실행

# ENTRYPOINT 단독 — 항상 node 실행, 인자만 바뀜
ENTRYPOINT ["node"]
CMD ["dist/main"]          # 기본 인자
# docker run 이미지 dist/other → node dist/other 실행 (ENTRYPOINT 유지)

# 실무 패턴 — ENTRYPOINT + CMD 조합
ENTRYPOINT ["node"]
CMD ["dist/main.js"]
# 기본: node dist/main.js
# 커스텀: docker run 이미지 dist/other.js → node dist/other.js
```

```txt
Node.js 앱에서 권장 패턴:
  CMD ["node", "dist/main"] 단독 사용이 가장 일반적
  → docker run 시 bash나 다른 명령어로 디버깅하기 편함
  → docker run 이미지 sh → 컨테이너 내부 진입 가능

JSON 배열 형식 vs 문자열 형식:
  CMD ["node", "dist/main"]  → exec 형식 (권장) — PID 1로 직접 실행
  CMD node dist/main         → shell 형식 — /bin/sh -c 로 감싸서 실행
  exec 형식이 시그널(SIGTERM) 처리가 더 확실해서 권장
```

---

# 멀티스테이지 빌드 ⭐️⭐️⭐️⭐️

```txt
문제:
  TypeScript 컴파일에는 tsc, ts-node 같은 개발 도구가 필요함
  그런데 실제 실행에는 컴파일된 JS 파일만 있으면 됨
  → 개발 도구를 포함한 채로 배포하면 이미지 크기가 수백 MB

해결:
  스테이지 1 (builder): 개발 도구 포함 → 컴파일만 담당
  스테이지 2 (runner):  node만 있는 가벼운 이미지 → 컴파일 결과물만 복사해서 실행
  → 최종 이미지에 개발 도구가 포함되지 않음
```

```dockerfile
# ── 스테이지 1: 빌드 ──────────────────────────────
FROM node:22-alpine AS builder

WORKDIR /app

# pnpm 사용 시
RUN npm install -g pnpm
COPY package.json pnpm-lock.yaml ./
RUN pnpm install --frozen-lockfile

COPY . .
RUN pnpm run build     # TypeScript → JavaScript 컴파일


# ── 스테이지 2: 실행 ──────────────────────────────
FROM node:22-alpine AS runner

WORKDIR /app

# 운영에 필요한 파일만 복사 (개발 의존성 제외)
RUN npm install -g pnpm
COPY package.json pnpm-lock.yaml ./
RUN pnpm install --frozen-lockfile --prod   # production 의존성만

# 빌드 결과물만 복사 (소스 코드, tsc 등은 안 옴)
COPY --from=builder /app/dist ./dist

EXPOSE 3000
CMD ["node", "dist/main"]
```

```txt
--from=builder:
  스테이지 1(builder)에서 만들어진 파일을 가져옴
  builder 스테이지 전체가 아니라 지정한 경로만 복사됨

--frozen-lockfile:
  lock 파일(pnpm-lock.yaml)과 package.json이 일치하지 않으면 에러
  재현 가능한 빌드 보장 — CI/CD에서 중요

--prod:
  devDependencies 제외하고 dependencies만 설치
  TypeScript, ts-node, eslint 등 개발 도구가 최종 이미지에 포함 안 됨

이미지 크기 비교:
  단일 스테이지 (개발도구 포함): 수백 MB ~ 1GB
  멀티스테이지 (실행 파일만):    수십 MB ~ 수백 MB

실제 배포 예시 → [[Deploy_CloudMVP]] 참고
```

---

# .dockerignore ⭐️⭐️⭐️

```txt
.gitignore처럼 Docker 빌드 시 제외할 파일/폴더 목록
COPY . . 할 때 이 파일들은 이미지에 포함되지 않음

왜 중요한가:
  node_modules를 포함하면 → 빌드 컨텍스트가 수백 MB → 느려짐
  .env 파일을 포함하면   → 비밀 값이 이미지에 박힘 → 보안 위험
  .git을 포함하면        → 불필요한 용량 증가
```

```gitignore
# .dockerignore

# 의존성 — 컨테이너 안에서 새로 설치
node_modules
**/node_modules

# 빌드 결과물 — 컨테이너 안에서 빌드
dist
**/dist

# 환경 변수 — 런타임에 주입
.env
**/.env
.env.*

# Git
.git
.gitignore

# 개발 도구 설정
.eslintrc*
.prettierrc*
tsconfig*.json   # 빌드 후에는 불필요

# 로컬 개발 파일
docker-compose*.yml
README.md
*.md
```

```txt
.dockerignore vs .gitignore:
  문법은 같지만 목적이 다름
  .gitignore → Git에 올리지 않을 파일
  .dockerignore → Docker 이미지에 포함하지 않을 파일
  둘 다 있어야 하고, 내용이 달라도 됨

tsconfig.json을 제외하는 이유:
  멀티스테이지 빌드에서 빌드는 builder 스테이지에서 완료됨
  runner 스테이지에서는 이미 컴파일된 JS만 실행 → tsconfig 불필요
  단일 스테이지라면 COPY . . 이전에 tsconfig가 필요하니 제외하면 안 됨
```

---

# 빌드 & 실행 명령어

```bash
# 이미지 빌드
docker build -t 앱이름:태그 .
docker build -t my-api:latest .
docker build -t my-api:1.0.0 --no-cache .   # 캐시 무시하고 처음부터

# 이미지 확인
docker images

# 컨테이너 실행
docker run -p 3000:3000 my-api:latest
docker run -d -p 3000:3000 --name my-api my-api:latest   # 백그라운드

# 실행 중인 컨테이너 확인
docker ps

# 컨테이너 로그
docker logs my-api
docker logs -f my-api   # 실시간 follow

# 컨테이너 내부 진입
docker exec -it my-api sh

# 정리
docker stop my-api
docker rm my-api
docker rmi my-api:latest   # 이미지 삭제
```

```txt
Docker Compose 환경에서는 대부분 docker compose 명령어 사용
docker build / docker run은 단독으로 테스트할 때 주로 씀
([[Docker_Compose]] 참고)
```

---

# alpine 이미지를 쓰는 이유 ⭐️⭐️

```txt
node:22         → Debian 기반 — 크고 범용적
node:22-slim    → Debian에서 불필요한 패키지 제거
node:22-alpine  → Alpine Linux 기반 — 5MB 수준의 초경량 OS

alpine 선택 이유:
  이미지 크기 차이: node:22 ≈ 1GB / node:22-alpine ≈ 180MB
  → 빌드 시간 단축, 레지스트리 용량 절약, 배포 속도 향상

alpine 주의점:
  glibc 대신 musl libc 사용 → 일부 네이티브 패키지 호환 문제
  (bcrypt, sharp 같은 네이티브 모듈은 alpine에서 문제 생길 수 있음)
  → 문제 생기면 -slim 버전으로 교체
```

---

# 한눈에

```txt
명령어 순서 (권장):
  FROM → WORKDIR → COPY package.json → RUN install → COPY . . → RUN build → CMD

캐시 최적화:
  package.json 먼저 COPY → install → 소스코드 COPY
  → 소스 변경 시 install은 캐시 재사용

CMD vs ENTRYPOINT:
  CMD   → 기본 명령, docker run 시 덮어쓰기 가능 (유연)
  ENTRYPOINT → 고정 명령, 인자만 바꿀 수 있음 (엄격)
  Node.js 앱: CMD ["node", "dist/main"] 단독이 일반적

멀티스테이지 빌드:
  AS builder → 컴파일 (개발도구 포함)
  AS runner  → 실행 (컴파일 결과물만)
  COPY --from=builder /app/dist ./dist
  → 최종 이미지에 tsc/ts-node 등 포함 안 됨 → 이미지 크기 대폭 감소

.dockerignore:
  node_modules / dist / .env / .git 반드시 제외
  보안(비밀값)과 성능(빌드 컨텍스트 크기) 모두 영향

alpine:
  node:22-alpine → 경량, 대부분 호환
  네이티브 모듈 문제 생기면 node:22-slim으로 교체
```