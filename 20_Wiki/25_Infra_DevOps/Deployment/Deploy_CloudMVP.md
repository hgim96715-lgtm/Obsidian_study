---
aliases:
  - 배포
  - Deploy
  - Dockerfile
  - Neon
  - Railway
  - Vercel
tags:
  - Deploy
  - NextJS
  - NestJS
related:
  - "[[00_Deployment_HomePage]]"
  - "[[Docker_Dockerfile]]"
  - "[[NestJS_Env_Config]]"
  - "[[NestJS_Migration]]"
  - "[[NestJS_Prisma_Monorepo]]"
  - "[[NestJS_Prisma]]"
  - "[[NextJS_AuthState]]"
  - "[[Web_XSS_CSRF]]"
---
# Deploy_CloudMVP — Vercel + Railway + Neon (pnpm 모노레포)

> [!info] 
>  Next.js(Web)는 Vercel, NestJS(API)는 Railway(Docker), Postgres는 Neon으로 배포하는 무료/저비용 풀스택 MVP 스택이다.
>   pnpm 모노레포 구조를 전제로 하며, 각 서비스가 서로 다른 origin이라 CORS 설정과 URL 형식이 핵심이다.

---

# 전체 구성 ⭐️⭐️⭐️⭐️

```mermaid
flowchart LR
    V["Vercel\nNext.js (Web)"] -->|HTTPS + Bearer| R["Railway\nNestJS (API)"]
    R --> DB[("Neon\nPostgres")]
```

|영역|서비스|역할|
|---|---|---|
|Web|Vercel|Next.js — `NEXT_PUBLIC_API_URL` = Railway API URL|
|API|Railway (Docker)|NestJS — `FRONTEND_URL` = Vercel origin · CORS|
|DB|Neon|Postgres — `DATABASE_URL` · 기동 시 `prisma migrate deploy`|

## ⚠️ Railway = API만, Web은 Vercel만 ⭐️⭐️⭐️⭐️

```txt
Railway 프로젝트에 API 서비스 하나면 충분 (railway.toml + apps/api/Dockerfile)

같은 레포에서 Railway가 Web(apps/web) 서비스를 또 만들거나
Web 빌드가 실패해도 무시해도 됨 — Web은 Vercel만 쓰면 됨

Railway Variables도 API용(DATABASE_URL · API_JWT_SECRET · FRONTEND_URL)만 신경 쓰면 됨
```

---

# ⚠️ URL 형식 — 경로 없이 origin만 ⭐️⭐️⭐️⭐️

```txt
CORS · fetch base URL 모두 origin(스킴 + 호스트)만 — 페이지 경로는 절대 넣지 않음
끝 슬래시(/)도 빼기
```

|변수|✅ 맞음|❌ 틀림|
|---|---|---|
|`FRONTEND_URL` (Railway)|`https://xxx.vercel.app`|`https://xxx.vercel.app/recommendations`|
|`FRONTEND_URL` (로컬)|`http://localhost:3031`|`http://localhost:3031/recommendations`|
|`NEXT_PUBLIC_API_URL` (Vercel)|`https://xxx.up.railway.app`|`https://xxx.up.railway.app/health`|

```txt
FRONTEND_URL에 /recommendations 같은 경로를 넣으면 CORS origin이 어긋나 브라우저에서 API 연결 실패
→ API 코드에서 new URL(FRONTEND_URL).origin으로 origin만 추출해서 CORS에 씀
  그래도 env에는 처음부터 origin만 넣는 게 혼란이 없음
```

---

# 배포 순서 — 왜 이 순서인가 ⭐️⭐️⭐️⭐️

```txt
0. 코드/설정 파일 준비 (로컬) ← 반드시 먼저
1. Neon DB 생성 → DATABASE_URL 확보
2. Railway API 배포 → API URL 확보         ← Web보다 먼저: Web이 API URL을 env로 필요로 함
3. Vercel Web 배포 → Web URL 확보
4. Railway FRONTEND_URL = Web origin → 재배포  ← CORS 반영: Web URL은 배포 후에야 알 수 있음
5. E2E 확인 (health · 가입 · 로그인)
```

```txt
FRONTEND_URL을 두 번 설정하는 이유:
  첫 Railway 배포 시점엔 Vercel URL이 아직 없음 → placeholder로 일단 배포
  Vercel 배포 완료 후 실제 origin을 Railway env에 업데이트 → 재배포(CORS 반영)
```

---

# 0단계 — 코드 준비 (push 전 로컬에서)

## start:deploy 스크립트 ⭐️⭐️⭐️⭐️

```json
// apps/api/package.json
{
  "scripts": {
    "start:prod": "node dist/src/main",
    "start:deploy": "prisma migrate deploy && node dist/src/main"
  }
}
```

```txt
⚠️ dist/src/main — dist/main이 아님
  Nest 프로젝트 루트에 prisma.config.ts 같은 파일이 있으면
  nest build 출력이 dist/src/ 아래로 들어감 (tsconfig 설정에 따라 다를 수 있으므로 빌드 후 확인)

migrate를 빌드 단계에 넣지 않는 이유:
  DATABASE_URL이 런타임에 주입되어야 DB 접속 가능 — 빌드 단계(Docker build)에선 연결 불가
  → 컨테이너 기동 시점에 migrate 먼저 실행하고 서버 올리는 것
```

## ⚠️ prisma generate vs prisma migrate deploy — 역할 구분 ⭐️⭐️⭐️⭐️

```txt
prisma generate:
  Prisma Client(타입·쿼리 빌더) 코드를 생성 — generated/ 폴더에 출력
  DB 접속 없음 · 테이블 변경 없음
  → Railway Build 단계에서 실행 (Dockerfile 또는 build 스크립트)

prisma migrate deploy:
  migrations/ 폴더의 pending(미적용) migration을 실제 DB에 적용
  테이블 생성·컬럼 추가·인덱스 등 스키마 변경이 여기서 일어남
  → Railway 컨테이너 기동 시점에 실행 (start:deploy 스크립트)
  → DB에 접속해야 하므로 DATABASE_URL 필요

500 "데이터베이스 오류" 원인:
  API 코드는 배포됐지만 새 테이블·컬럼이 DB에 없는 상태
  → start:deploy 스크립트에 prisma migrate deploy가 포함됐는지 확인
  → 또는 Railway 재배포(재시작)가 필요한 경우
```

```txt
Neon DB에 migration을 로컬에서 직접 적용해야 할 때:
  (Railway 재배포 없이 즉시 반영하고 싶을 때 또는 초기 세팅 시)
```

```bash
# apps/api/.env 에 Neon DATABASE_URL이 설정된 상태에서
pnpm --filter api exec prisma migrate deploy
```

```txt
  → migrations/ 안의 미적용 migration이 Neon에 즉시 적용됨
  → Railway 재배포 없이 테이블 구조 반영 가능
  ⚠️ 로컬 .env의 DATABASE_URL이 Neon 진짜 URL인지 확인 후 실행
```

## env 검증 — 클라우드 전용 변수는 optional로 ⭐️⭐️⭐️

```typescript
// apps/api/src/config/env.validation.ts
[EnvKeys.DATABASE_URL]: Joi.string().uri().required(),
[EnvKeys.API_JWT_SECRET]: Joi.string().required(),
[EnvKeys.FRONTEND_URL]: Joi.string().uri().required(),
[EnvKeys.PORT]: Joi.number().default(3000),

// Docker Compose 로컬 전용 — 클라우드에선 불필요하므로 optional
[EnvKeys.POSTGRES_PORT]: Joi.number().port().optional(),
[EnvKeys.POSTGRES_USER]: Joi.string().optional(),
[EnvKeys.POSTGRES_PASSWORD]: Joi.string().optional(),
[EnvKeys.POSTGRES_DB]: Joi.string().optional(),
```

```txt
⚠️ 흔한 실수: POSTGRES_*를 required로 두면 Neon 환경(DATABASE_URL 하나만 씀)에서
  "필수 env 없음" 에러로 기동 실패 → optional로
```

## CORS — origin만 추출 ⭐️⭐️⭐️⭐️

```typescript
// apps/api/src/main.ts
const frontendUrl = configService.get<string>('FRONTEND_URL');
const frontendOrigin = frontendUrl ? new URL(frontendUrl).origin : undefined;

app.enableCors({
  origin: frontendOrigin
    ? [
        'http://localhost:3031',   // 로컬 Web 포트에 맞게 수정
        'http://127.0.0.1:3031',
        frontendOrigin,
      ]
    : undefined,
  credentials: true,
});
```

```txt
new URL(frontendUrl).origin 으로 origin만 추출하는 이유:
  FRONTEND_URL env에 실수로 경로(/recommendations 등)가 붙더라도
  CORS에 쓰이는 값은 항상 origin만 되도록 코드 레벨에서 방어
  그래도 env에는 처음부터 origin만 넣는 게 좋음 (위 URL 형식 규칙 참고)
```

## .gitignore — Dockerfile 커밋되어 있어야 함

```bash
# 이 줄이 있으면 제거
# apps/api/Dockerfile

# Railway는 GitHub에서 Dockerfile을 직접 읽으므로 반드시 커밋되어 있어야 함
```

## .dockerignore (루트)

```txt
node_modules
.pnpm-store
.git
**/.env
**/.env.local
**/.next
**/dist
apps/web        ← API 이미지에 Web 앱은 포함시키지 않음
```

## 로컬 빌드 검증 — push 전 필수

```bash
pnpm install

# API
cd apps/api && pnpm exec prisma generate && pnpm build

# Web
cd ../.. && pnpm --filter web build
```

```txt
둘 다 exit 0이어야 배포 성공 가능
특히 dist/src/main 경로 확인 — 빌드 후 실제 출력 위치 눈으로 확인
```

---

# ⚠️ Dockerfile — 더미 DATABASE_URL이 필요한 이유 ⭐️⭐️⭐️⭐️

```dockerfile
FROM node:20-alpine

RUN corepack enable && corepack prepare pnpm@9 --activate

WORKDIR /app

COPY pnpm-lock.yaml pnpm-workspace.yaml package.json ./
COPY apps/api ./apps/api

RUN pnpm install --frozen-lockfile --filter api

WORKDIR /app/apps/api

# ⚠️ 더미 DATABASE_URL — Prisma 7 prisma.config.ts가 빌드 시점에도 env를 요구함
# 실제 DB 접속은 안 하지만 형식만 맞는 URL이 있어야 generate 실행 가능
RUN DATABASE_URL="postgresql://build:build@127.0.0.1:5432/build?schema=public" \
    pnpm exec prisma generate && pnpm build

ENV NODE_ENV=production
EXPOSE 3000   # 문서용 — 실제 listen은 Railway가 주입하는 PORT

CMD ["pnpm", "run", "start:deploy"]
```

## DATABASE_URL이 두 군데 나오는 이유 — 헷갈리기 쉬움 ⭐️⭐️⭐️⭐️

```txt
Dockerfile 더미 URL과 Railway Variables의 진짜 URL은 완전히 다른 역할임
```

|시점|어디에|값|DB 접속?|
|---|---|---|---|
|Docker build|Dockerfile `RUN DATABASE_URL=...`|더미 (형식만 맞는 가짜)|❌ prisma generate만 — 스키마 읽어 클라이언트 생성|
|컨테이너 실행|Railway Variables `DATABASE_URL`|Neon Connect에서 복사한 진짜 URL|✅ migrate deploy · API 쿼리|

```txt
왜 빌드에 더미가 필요한가:
  Prisma 7의 prisma.config.ts가 config 로딩 시 env('DATABASE_URL')을 요구함
  Railway 빌드 단계에는 Variables가 없거나, 있어도 generate는 DB에 안 붙음
  → Dockerfile에 형식만 맞는 가짜 URL을 RUN 한 줄에만 주입

⚠️ Railway Variables의 DATABASE_URL에 더미를 넣으면 migrate 실패 · API DB 연결 불가
   반드시 Neon Console → Connect에서 복사한 진짜 URL을 Variables에 넣을 것
```

```mermaid-beautiful
flowchart TB
    subgraph BUILD["Docker build (DB 접속 없음)"]
        DUMMY["Dockerfile 더미 DATABASE_URL"]
        GEN["prisma generate · nest build"]
        DUMMY --> GEN
    end

    subgraph RUN["Railway runtime (실제 DB)"]
        NEON["Neon Connect → 진짜 DATABASE_URL (Railway Variables)"]
        MIG["prisma migrate deploy"]
        API["Nest API 서버"]
        NEON --> MIG --> API
    end

    BUILD --> RUN
```

---

# 1. Neon (DB) ⭐️⭐️⭐

```txt
Neon은 PostgreSQL 데이터베이스를 클라우드에서 제공하는 관리형 서버리스 데이터베이스 서비스

PostgreSQL = 데이터베이스 소프트웨어
Neon       = PostgreSQL을 클라우드에서 생성·운영해 주는 서비스
```

## 프로젝트 생성 및 DATABASE_URL 복사

```txt
1. Neon Console 로그인 → New project
2. 프로젝트 Dashboard 열림
3. 상단/우측 Connect 버튼 클릭
4. Connection string 탭 선택 (Railway에는 이걸로 충분)
5. Connection pooling: Pooled 권장 (URL에 -pooler 호스트가 붙는 형태)
6. 한 줄짜리 문자열 복사 → Railway Variables에 붙여넣기
```

```bash
# 복사된 URL 모양 (비밀번호·호스트는 프로젝트마다 다름)
postgresql://<role>:<password>@<host>-pooler.<region>.aws.neon.tech/neondb?sslmode=require
#            ───┬───  ───┬───  ──────────────────┬──────────────── ───┬───
#              user    password                  host               database
```

```txt
URL 구조 파싱:
  user     → <role>                              (Neon Console에서 확인)
  password → <password>                          (Neon Console에서 확인)
  host     → <host>-pooler.<region>.aws.neon.tech  ← -pooler. 까지 전부 host
  port     → 5432                                (명시 없으면 기본값)
  database → neondb
  SSL      → require
```

```txt
DataGrip (또는 다른 DB 클라이언트) 연결 시:
  Host     → <host>-pooler.<region>.aws.neon.tech  ← @ 뒤부터 / 앞까지 전체
  Port     → 5432
  User     → <role>
  Password → <password>
  Database → neondb
  SSL mode → require (Advanced 탭 또는 sslmode 옵션에서 설정)

  ⚠️ 흔한 실수: host 필드에 <host> 부분만 입력
     -pooler.<region>.aws.neon.tech 까지 포함해야 연결됨
```

```txt
⚠️ sslmode=require — Neon 프로덕션에 필요, 빼지 않기
⚠️ 끝에 / 없음 — .../neondb 까지
⚠️ Git 커밋 절대 금지 — Railway Variables · Neon Console에서만 관리
```

---

# 2. Railway (API) ⭐️⭐️⭐️⭐

```txt
Railway는 서버·데이터베이스·Redis 같은 백엔드 인프라를 간편하게 배포하고 운영하도록 도와주는 클라우드 플랫폼
```

```txt
Railway → New Project → Deploy from GitHub repo

서비스는 API 하나만 (railway.toml + apps/api/Dockerfile)
Settings → Root Directory: 비워둘 것 (레포 루트)
```

| 변수               | 값                                                          | 필수     |
| ---------------- | ---------------------------------------------------------- | ------ |
| `DATABASE_URL`   | Neon Connect → Connection string (진짜 URL — 더미 아님)          | ✅      |
| `API_JWT_SECRET` | `openssl rand -base64 32`                                  | ✅      |
| `FRONTEND_URL`   | Vercel origin만 — `https://xxx.vercel.app` (경로 ❌, 3단계 후 확정) | ✅      |
| `PORT`           | Railway 자동 주입 (Variables에 직접 입력해도 됨 — 아래 참고)              | 설정 불필요 |

## ⚠️ PORT — NestJS가 반드시 process.env.PORT로 listen해야 함 ⭐️⭐️⭐️⭐️

```typescript
// apps/api/src/main.ts
const port = process.env.PORT ?? 3000;
await app.listen(port);
```

```txt
Railway는 서비스마다 PORT를 자동 주입함
NestJS가 하드코딩된 3000으로 listen하면 Railway 라우터가 기대하는 포트와 달라져
→ 외부 도메인(https://xxx.up.railway.app) 접근 실패
→ health check 실패 → 배포 실패 처리 또는 Actions curl 에러

Variables 탭에 PORT가 안 보일 때:
  Railway 자동 주입 변수는 Variables 탭에서 안 보일 수 있음
  → Variables 탭에서 PORT = 3000 (또는 원하는 값) 직접 입력하면
    Generate Domain 입력 창에서 그 값을 넣으면 됨
  → 직접 지정하지 않으면 Railway가 할당한 값으로 app.listen(process.env.PORT) 동작
```

```txt
Deploy 완료 후 공개 도메인 발급:
  서비스 클릭 → Settings 탭 → Networking → Public Networking
  → Generate Domain 버튼 클릭
  → "Enter the port your app is listening on" 입력 창 뜸

PORT 값 확인:
  Variables 탭 → PORT 값 확인 (직접 설정했으면 그 값 / 없으면 3000이 일반적)
  → 해당 숫자 입력 → Generate
  ⚠️ 여기 입력하는 포트와 app.listen(port)의 포트가 반드시 일치해야 함

발급된 도메인 형식: https://<service-name>.up.railway.app
→ NEXT_PUBLIC_API_URL, GitHub Secret CINEMO_API_URL 등에 사용
```

```txt
확인: GET https://<railway-domain>/health → {"ok":true}
```

---

# railway.toml (루트)

```toml
[build]
builder = "DOCKERFILE"
dockerfilePath = "apps/api/Dockerfile"

[deploy]
startCommand = "pnpm run start:deploy"
healthcheckPath = "/health"
healthcheckTimeout = 120
restartPolicyType = "ON_FAILURE"
```

```txt
healthcheckPath: Railway가 이 경로로 GET 요청을 보내서 서버 생존 확인
→ NestJS에 GET /health → { ok: true } 컨트롤러가 없으면 health check 실패로 배포 실패 처리됨
```

```typescript
// apps/api/src/health/health.controller.ts
@Controller('health')
export class HealthController {
  @Get()
  check() { return { ok: true }; }
}
```

---

# 3. Vercel (Web) ⭐️⭐️⭐

```txt
Vercel은 웹 프론트엔드, 특히 Next.js 애플리케이션을 간편하게 빌드하고 배포하는 클라우드 플랫폼
```


```txt
Vercel → Import GitHub repo
Root Directory: apps/web
```

|변수|값|
|---|---|
|`NEXT_PUBLIC_API_URL`|`https://<railway-domain>` (끝 `/` 없음, 경로 없음)|

```txt
Deploy → Web URL 복사 (예: https://xxx.vercel.app)
Railway FRONTEND_URL = 위 URL origin만 (/recommendations 없이) → Redeploy
```

---

# apps/web/vercel.json — 언제 필요한가 ⭐️⭐️⭐️

```txt
필요한 경우:
  Vercel Root Directory = apps/web 으로 설정했을 때
  install/build를 모노레포 루트에서 실행해야 workspace가 동작하는 구조

  → Vercel이 apps/web 안에서 pnpm install을 실행하면 루트 workspace를 못 찾아 빌드 실패
  → vercel.json으로 cd ../.. 후 루트에서 실행하도록 오버라이드

필요 없는 경우 (이쪽이 더 깔끔):
  Vercel Dashboard > Settings > Root Directory = . (루트)
  Build Command: pnpm --filter web build
  Install Command: pnpm install
  → vercel.json 없이도 Dashboard에서 직접 설정 가능
  → Vercel이 pnpm workspace를 자동 감지해서 처리

⚠️ 무조건 만들어야 하는 파일이 아님
   Dashboard에서 Build/Install Command를 직접 지정하면 불필요
```

```json
// Root Directory = apps/web 방식일 때만 필요
{
  "$schema": "https://openapi.vercel.sh/vercel.json",
  "installCommand": "cd ../.. && pnpm install",
  "buildCommand": "cd ../.. && pnpm --filter web build"
}
```

---

# 환경변수 정리

## Railway (API)

| 변수               | 로컬 `.env`               | 프로덕션                                     |
| ---------------- | ----------------------- | ---------------------------------------- |
| `DATABASE_URL`   | `localhost:5432/...`    | Neon Connect 복사값 (`?sslmode=require` 포함) |
| `API_JWT_SECRET` | 로컬 시크릿                  | `openssl rand -base64 32` (로컬과 분리)       |
| `FRONTEND_URL`   | `http://localhost:3031` | `https://xxx.vercel.app` (경로·슬래시 없음)     |
| `PORT`           | `3000`                  | Railway 자동 주입                            |
| `POSTGRES_*`     | Docker Compose용         | 불필요 (Neon은 DATABASE_URL만)                |

## Vercel (Web)

|변수|로컬 `.env.local`|프로덕션|
|---|---|---|
|`NEXT_PUBLIC_API_URL`|`http://localhost:3000`|`https://<railway-domain>` (경로·슬래시 없음)|

---

# railway.toml — 언제 필요한가 ⭐️⭐️⭐️⭐️

```txt
필요한 경우:
  Dockerfile로 빌드할 때 → dockerfilePath 지정
  Start Command를 파일로 관리하고 싶을 때
  Health check 경로·타임아웃을 커스텀할 때
  → 코드 레벨에서 배포 설정을 관리할 때 유용

필요 없는 경우 (Railway Dashboard로 충분):
  Railway Dashboard > Settings > Build Command / Start Command 직접 입력
  Build Command: pnpm --filter api build
  Start Command: node dist/src/main  (또는 pnpm run start:deploy)
  → railway.toml 없이도 배포 가능, Dashboard 설정이 우선 적용됨

⚠️ 무조건 만들어야 하는 파일이 아님
   toml을 쓴다면 dockerfilePath · startCommand가 실제 파일·스크립트와 정확히 일치해야 함
   경로가 틀리면 (Dockerfile 없음 / 스크립트 없음) 배포 즉시 실패
```

```toml
# Dockerfile 방식일 때만 사용
[build]
builder = "DOCKERFILE"
dockerfilePath = "apps/api/Dockerfile"

[deploy]
startCommand = "pnpm run start:deploy"
healthcheckPath = "/health"
```

```txt
Dockerfile 없이 Dashboard Build Command만 쓰는 경우:
  apps/api/package.json build 스크립트에 prisma generate 포함 필수
  → Railway 깨끗한 환경에는 generated/ 폴더가 없음 (gitignore 대상)
```

```json
// apps/api/package.json — Dockerfile 없는 방식
{
  "scripts": {
    "build": "prisma generate --schema prisma/schema.prisma && rm -f tsconfig.build.tsbuildinfo && nest build"
  }
}
```

---

# 트러블슈팅 ⭐️⭐️⭐️⭐️

|증상|원인|해결|
|---|---|---|
|`Property 'xxx' does not exist on type 'PrismaService'` / `Cannot find module '../generated/prisma/client'`|`generated/` 폴더가 `.gitignore` 대상 → Railway 깨끗한 빌드 환경에 Prisma Client 없음|`apps/api/package.json` build 스크립트에 `prisma generate --schema prisma/schema.prisma &&` 먼저 추가|
|`https://<railway-domain>` 접근 실패 / Actions curl 에러|`app.listen(3000)` 하드코딩 → Railway가 주입한 PORT와 불일치|`main.ts` → `app.listen(process.env.PORT ?? 3000)` + Generate Domain 입력값과 일치 확인|
|500 "데이터베이스 오류" — API는 뜨는데 쿼리 실패|`prisma generate`는 Client만 생성, DB 테이블은 `migrate deploy`가 적용 — migration 누락|`start:deploy`에 `prisma migrate deploy &&` 포함 확인 / 또는 로컬에서 `pnpm --filter api exec prisma migrate deploy` 직접 실행|
|`Cannot find module .../dist/main`|Nest 빌드 출력이 `dist/src/main.js`인데 `dist/main`으로 참조|`start:deploy` → `node dist/src/main`으로 수정|
|`PrismaConfigEnvError: DATABASE_URL` (Docker build)|Prisma 7 `prisma.config.ts`가 빌드 시점에도 env 요구|Dockerfile `RUN`에 더미 URL 추가 (§ Dockerfile 참고)|
|Railway 빌드 성공인데 migrate/API 연결 실패|Variables에 `DATABASE_URL` 없거나 더미 URL 넣음|Neon Connect → 복사 → Railway Variables에 진짜 URL|
|CORS 에러 — `FRONTEND_URL`에 경로 포함|`https://xxx.vercel.app/recommendations` 처럼 경로가 들어감|origin만 — `https://xxx.vercel.app` · Railway Redeploy|
|Railway 빌드 실패 — Dockerfile 없음|Dockerfile이 `.gitignore`에 있어서 커밋 안 됨|`.gitignore`에서 해당 줄 제거 후 커밋·push|
|API 기동 직후 종료|`DATABASE_URL` 오류 또는 migrate 실패|Railway 로그 확인 · Neon URL · `sslmode=require` 확인|
|CORS 에러 (그 외)|`FRONTEND_URL` ≠ 실제 Vercel origin|브라우저 주소창 origin과 동일하게 · Redeploy|
|`POSTGRES_USER required` 에러|env validation에서 로컬 전용 변수를 `required`로 선언|`POSTGRES_*`를 `optional()`로 변경|
|Vercel 빌드 실패 — workspace 못 찾음|`vercel.json`의 `cd ../..`가 없어서 루트 install 안 됨|`installCommand` 확인|
|Railway health check 실패|`/health` 엔드포인트 없음|`HealthController` 추가|

---

# 다른 pnpm 모노레포에 옮길 때 체크리스트 ⭐️⭐️⭐️

```txt
코드 변경:
  □ apps/api/package.json — start:deploy: "prisma migrate deploy && node dist/src/main"
    (빌드 후 실제 dist 경로 확인 — dist/main vs dist/src/main)
  □ env 검증 — POSTGRES_* optional로
  □ CORS — new URL(frontendUrl).origin 추출 + 로컬 포트 배열로
  □ GET /health 엔드포인트 추가
  □ .gitignore — Dockerfile 커밋되어 있는지 확인
  □ .dockerignore — Web 앱 폴더 제외

파일 추가:
  □ apps/api/Dockerfile — 더미 DATABASE_URL RUN (Dockerfile 방식일 때)
  □ railway.toml (루트) — Dockerfile 방식일 때만 / Dashboard 방식이면 불필요
  □ apps/web/vercel.json — Root Directory=apps/web 방식일 때만 / Dashboard 설정이면 불필요

로컬 검증:
  □ pnpm install
  □ API: prisma generate + pnpm build → dist 출력 경로 확인
  □ Web: pnpm --filter web build
  □ 둘 다 exit 0 확인 후 push

배포 순서:
  □ Neon DB 생성 → Connect → DATABASE_URL 복사 (진짜 URL)
  □ Railway API만 배포 → API URL 확보 (Web 서비스 Railway에 올리지 않기)
  □ Vercel Web 배포 → Web URL 확보
  □ Railway FRONTEND_URL = Web origin만 (경로 없음) → Redeploy
  □ GET {API}/health → { ok: true } 확인
  □ Web에서 가입 · 로그인 E2E 확인
```

---

# 한눈에

```txt
구성: Web(Vercel) + API(Railway/Docker) + DB(Neon) — Railway는 API만, Web은 Vercel만

URL 규칙: FRONTEND_URL / NEXT_PUBLIC_API_URL 모두 origin만 — 경로·끝 슬래시 금지
CORS: new URL(frontendUrl).origin으로 origin 추출해서 배열에 넣기

Dockerfile 핵심:
  더미 DATABASE_URL이 RUN 줄에 필요 (Prisma 7 prisma.config.ts 요구사항 — DB 접속 안 함)
  Railway Variables에는 Neon 진짜 URL (절대 더미 아님)
  dist/src/main — dist/main이 아닐 수 있음 (빌드 후 확인)

배포 순서: DB → API → Web → Railway FRONTEND_URL 재설정

흔한 실수 Top 5:
  ① dist/main → dist/src/main 경로 오류
  ② Dockerfile 더미 DATABASE_URL 누락 → prisma generate 실패
  ③ Railway Variables에 더미 URL → migrate 실패
  ④ FRONTEND_URL에 경로 포함 → CORS 에러
  ⑤ Dockerfile이 gitignore → Railway 빌드 실패
```