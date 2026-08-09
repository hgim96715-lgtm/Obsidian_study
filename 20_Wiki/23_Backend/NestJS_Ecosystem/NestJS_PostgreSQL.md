---
aliases:
  - NestJS
  - PostgreSQL
  - Docker
  - DataGrip
  - Database
tags:
  - NestJS
  - SQL
related:
  - "[[00_NestJS_Ecosystem_HomePage]]"
  - "[[00_DB_HomePage]]"
  - "[[NestJS_Prisma]]"
  - "[[NestJS_Migration]]"
---
# NestJS_PostgreSQL — PostgreSQL 연결 · 로컬 환경

>[!info]
>NestJS에서 PostgreSQL 연결 = Prisma를 통한 연결이 표준. 
>로컬 개발은 Docker로 PostgreSQL을 띄우고, DataGrip으로 DB를 시각적으로 관리한다.
> `PrismaService`가 연결·해제를 담당하고 `PrismaModule @Global()`로 전역 등록한다.
>  Prisma 쿼리·스키마 → [[NestJS_Prisma]], 마이그레이션 → [[NestJS_Migration]]

---

# 용어 정리 ⭐️⭐️⭐️⭐️

|이름|역할|
|---|---|
|SQL|DB에 보내는 질의 언어 (`CREATE TABLE`, `INSERT` …)|
|PostgreSQL|SQL을 실행하는 DB 서버 (여기선 Docker로 실행)|
|Prisma|schema.prisma → 마이그레이션 SQL 생성 + TypeScript 클라이언트|
|migration|스키마 변경을 DB에 적용하는 SQL 기록 (`prisma/migrations/`)|
|generate|`schema.prisma` → `src/generated/prisma` 클라이언트 코드 생성|

---

# 설치 순서 ⭐️⭐️⭐️⭐️

```bash
# 1. 패키지 설치 (루트에서)
pnpm --filter api add @prisma/client
pnpm --filter api add -D prisma
pnpm --filter api add dotenv
# dotenv 필요: prisma.config.ts가 import "dotenv/config"를 사용

# 2. Prisma 초기화
pnpm --filter api exec prisma init
```

```txt
prisma init이 만드는 것 (Prisma 7 기준):
  apps/api/prisma/schema.prisma  → 모델 정의
  apps/api/prisma.config.ts      → datasource URL 설정 (Prisma 7 변경사항)

Prisma 7 변경사항:
  이전: schema.prisma의 datasource db { url = env("DATABASE_URL") }
  이후: prisma.config.ts에서 URL 관리
  → schema.prisma에 url을 안 넣고 prisma.config.ts에서 읽음
```

---
# NestJS에서 DB 연결 방법 선택 ⭐️⭐️⭐️⭐️

```txt
NestJS + PostgreSQL 연결 옵션:

① Prisma (이 프로젝트 — 권장)
  schema.prisma → PrismaClient 타입 자동 생성
  쿼리 자동완성·타입 안전
  마이그레이션이 직관적
  현재 Node.js 생태계 표준

② TypeORM (@nestjs/typeorm)
  NestJS 공식 지원, 데코레이터 기반
  설정이 복잡하고 타입 안전성이 상대적으로 약함

③ pg (직접 연결)
  SQL 문자열 직접 작성 → 타입 안전성 없음
  거의 안 씀
```

---

# DATABASE_URL — DB 연결 문자열 ⭐️⭐️⭐️⭐️

```bash
DATABASE_URL="postgresql://유저:비밀번호@호스트:포트/DB이름?schema=public"
```

```bash
# 로컬 Docker (포트 5444 — Mac 기본 5432 충돌 회피)
DATABASE_URL="postgresql://USER:PASSWORD@localhost:5444/DB?schema=public&sslmode=disable"
POSTGRES_USER=USER
POSTGRES_PASSWORD=PASSWORD
POSTGRES_DB=DB
POSTGRES_PORT=5444

# Neon (클라우드)
DATABASE_URL="postgresql://user:pass@ep-xxx.ap-southeast-1.aws.neon.tech/neondb?sslmode=require"

# Railway (배포)
DATABASE_URL="postgresql://postgres:AbCdEf@containers-us-west-xxx.railway.app:1234/railway"
```

```txt
DATABASE_URL vs POSTGRES_*:
  DATABASE_URL       → Prisma가 DB에 접속할 때 사용
  POSTGRES_USER/PASSWORD/DB → Docker가 컨테이너 생성 시 DB 초기 설정에 사용

  둘의 user·password·db 이름이 반드시 같아야 함
  DATABASE_URL 포트 = docker-compose ports의 왼쪽(호스트) 포트

  예시 (포트 5444):
  docker-compose: ports: '5444:5432'   ← 호스트 5444, 컨테이너 5432
  DATABASE_URL: ...@localhost:5444/...  ← 호스트 포트 5444로 접근
```

```txt
각 부분 설명:
  postgresql://  → 드라이버 (postgres:// 도 동일)
  postgres       → DB 사용자 이름
  password       → 비밀번호
  localhost      → 호스트 (원격이면 도메인/IP)
  5444           → 포트 (docker-compose hosts 포트)
  myapp_dev      → 데이터베이스 이름
  ?sslmode=disable → 로컬은 SSL 불필요, 클라우드는 require

⚠️ DATABASE_URL은 .env에만 저장, 절대 git에 올리면 안 됨
```

---

# Docker로 PostgreSQL 로컬 실행 ⭐️⭐️⭐️⭐️

```txt
로컬에 PostgreSQL을 직접 설치하는 대신 Docker로 실행하면:
  설치 없이 바로 사용 가능
  버전 변경 쉬움
  팀원 모두 같은 환경
  docker-compose.yml 하나로 공유
```

## docker-compose.yml

```yaml
# docker-compose.yml ← 반드시 저장소 루트에 위치
# apps/docker-compose.yml에 두면 루트에서 docker compose up 시 "not found" 에러
services:
  db:
    image: postgres:17-alpine          # PostgreSQL 17 경량 이미지
    container_name: music-community-db
    env_file:
      - apps/api/.env                  # .env에서 POSTGRES_USER, POSTGRES_DB 등을 읽음
    ports:
      - '5433:5432'                    # 호스트 5433 → 컨테이너 5432
    volumes:
      - db_data:/var/lib/postgresql/data  # 데이터 영구 보존
    healthcheck:
      test: ['CMD-SHELL', 'pg_isready -U "$$POSTGRES_USER" -d "$$POSTGRES_DB"']
      interval: 10s
      timeout: 5s
      retries: 5
      start_period: 40s               # 컨테이너 시작 후 40초 뒤부터 체크
    networks:
      - music-community-network

volumes:
  db_data:

networks:
  music-community-network:
    driver: bridge
```

```txt
env_file: apps/api/.env:
  .env 파일에서 환경변수를 읽어 컨테이너에 주입
  PostgreSQL은 다음 환경변수로 초기 설정:
  POSTGRES_USER     → DB 사용자 이름
  POSTGRES_PASSWORD → 비밀번호
  POSTGRES_DB       → 생성할 DB 이름
  → .env에 이 세 값이 있으면 컨테이너 시작 시 자동 설정

ports: '5433:5432':
  왼쪽(5433) = 내 컴퓨터 포트  ← DATABASE_URL에서 이 포트 사용
  오른쪽(5432) = 컨테이너 포트
  5432가 아닌 5433을 쓰는 이유:
  로컬에 PostgreSQL이 이미 설치돼 있으면 5432 충돌 → 5433으로 회피

healthcheck:
  pg_isready로 DB가 실제로 준비됐는지 주기적으로 확인
  DB에 의존하는 서비스(api 등)가 DB 준비 전에 시작하는 것을 방지
  start_period: 40s → 처음 시작 40초는 체크 안 함 (초기화 시간 여유)

networks:
  컨테이너끼리 통신하는 가상 네트워크
  같은 네트워크 안의 컨테이너끼리 서비스 이름으로 통신 가능
  (api 컨테이너에서 db 컨테이너를 'db' 이름으로 접근)
```

```bash
# apps/api/.env — docker-compose가 읽는 PostgreSQL 환경변수
POSTGRES_USER=postgres
POSTGRES_PASSWORD=password
POSTGRES_DB=myapp_dev

# Prisma가 읽는 DATABASE_URL (포트 5433 주의)
DATABASE_URL="postgresql://postgres:password@localhost:5433/myapp_dev"
```

## 실행 명령어

```bash
# 반드시 저장소 루트에서 실행
docker compose up -d

# 특정 서비스만 실행
docker compose up -d db

# 중지
docker compose down

# 중지 + 데이터 삭제 (초기화)
docker compose down -v

# 로그 확인
docker compose logs db
docker compose logs -f db   # -f: 실시간 추적

# 컨테이너 상태 확인 (healthy 뜨면 DB 준비 완료)
docker ps
```

```txt
docker compose 실행 위치:
  루트에서 실행해야 docker-compose.yml을 찾음
  apps/ 폴더에서 실행하면 "no configuration file provided" 에러
```

---

# migrate · generate 워크플로우 ⭐️⭐️⭐️⭐️

```bash
# 1. 마이그레이션 실행 (스키마 → DB 적용)
pnpm --filter api exec prisma migrate dev --name init

# 2. 클라이언트 생성 (스키마 → TypeScript 타입)
pnpm --filter api exec prisma generate
# migrate dev 실행 시 보통 generate도 함께 실행됨
# client가 없으면 따로 한 번 더 실행
```

```txt
실행 후 생기는 것:
  apps/api/prisma/migrations/20240101_init/migration.sql  → 적용된 SQL 기록
  apps/api/src/generated/prisma/                         → Prisma 클라이언트

migrate가 생성하는 SQL 예시 (migration.sql):
  -- CreateTable
  CREATE TABLE "users" (
    "id" TEXT NOT NULL,
    "email" TEXT NOT NULL,
    "created_at" TIMESTAMP(3) NOT NULL DEFAULT CURRENT_TIMESTAMP,
    CONSTRAINT "users_pkey" PRIMARY KEY ("id")
  );
  CREATE UNIQUE INDEX "users_email_key" ON "users"("email");

  → schema.prisma의 model User { @@map("users") } 가 위 SQL로 변환됨

앱 코드에서는:
  prisma.user.create({ data: { email: 'a@b.com' } })
  → Prisma Client가 SQL INSERT를 만들어 PostgreSQL에 전송
```

---

---

# Dockerfile — API 빌드 ⭐️⭐️⭐️

```dockerfile
# Dockerfile (apps/api 또는 루트)
FROM node:20-alpine

# pnpm 활성화 (corepack = Node.js 내장 패키지 매니저 관리 도구)
RUN corepack enable && corepack prepare pnpm@9 --activate

WORKDIR /app

# 의존성 먼저 복사 (캐싱 활용 — 코드 변경 시 재설치 방지)
COPY pnpm-lock.yaml pnpm-workspace.yaml package.json ./
COPY apps/api ./apps/api

# api 앱만 설치 (frozen-lockfile = lockfile 변경 금지)
RUN pnpm install --frozen-lockfile --filter api

WORKDIR /app/apps/api

# Prisma generate — DB 접속 없이 타입만 생성
# 빌드 시에는 실제 DB가 없으므로 더미 DATABASE_URL 사용
RUN DATABASE_URL="postgresql://build:build@127.0.0.1:5432/build?schema=public" \
    pnpm exec prisma generate && pnpm build

ENV NODE_ENV=production
EXPOSE 3030

CMD ["pnpm", "run", "start:deploy"]
```

```txt
corepack enable && corepack prepare pnpm@9 --activate:
  corepack = Node.js 20+ 내장 패키지 매니저 버전 관리 도구
  pnpm을 별도 설치 없이 특정 버전으로 활성화

COPY 순서가 중요한 이유 (Docker 레이어 캐싱):
  1. lockfile·package.json 복사
  2. pnpm install (의존성 설치)
  3. 소스 코드 복사
  → 소스만 바뀌면 1·2 단계를 캐시에서 재사용
  → 소스 바뀔 때마다 전체 재설치 방지 → 빌드 속도 향상

--frozen-lockfile:
  pnpm-lock.yaml이 변경되는 것을 금지
  lockfile과 package.json이 일치하지 않으면 에러
  → 배포 환경에서 예상치 못한 버전 변경 방지

prisma generate의 더미 DATABASE_URL:
  prisma generate = schema.prisma → PrismaClient 타입 생성
  DB에 실제 접속하지 않음 → 더미 URL이어도 됨
  하지만 DATABASE_URL 환경변수 자체는 있어야 에러 안 남
  → 빌드 시에만 쓰는 의미 없는 값으로 통과

start:deploy:
  운영 환경 시작 스크립트 (package.json에 정의)
  보통 "node dist/main.js" 또는 "nest start --prod"
```

---

# DataGrip — DB GUI 관리 도구 ⭐️⭐️⭐️

```txt
DataGrip (JetBrains):
  PostgreSQL을 시각적으로 탐색·쿼리하는 IDE
  테이블 구조 확인, SQL 실행, 데이터 편집
  개발 중 DB 상태를 직접 확인할 때 편함
```

## DataGrip 연결 설정

```txt
New Connection → PostgreSQL 선택:

  Host:     localhost
  Port:     5432
  Database: myapp_dev
  User:     postgres
  Password: password

Test Connection 클릭 → 성공 확인 → OK
```

## 자주 쓰는 기능

```txt
DB Explorer (왼쪽 패널):
  테이블 목록 → 컬럼 구조 확인
  우클릭 → View Data → 현재 데이터 조회

Console (SQL 실행):
  Ctrl+Enter / Cmd+Enter → 선택한 쿼리 실행
  SELECT * FROM "User" LIMIT 10;

테이블 이름 주의:
  Prisma가 생성한 테이블은 대소문자 구분
  → "User"처럼 대문자가 있으면 따옴표로 감싸야 함
  → SELECT * FROM "User" (O)
  → SELECT * FROM User (X — user 예약어로 오해)
```

---

# Prisma 설치 및 초기 설정 ⭐️⭐️⭐️⭐️

```bash
# Prisma 설치 (apps/api 에 추가)
pnpm --filter api add @prisma/client
pnpm --filter api add -D prisma

# Prisma 초기화
cd apps/api
npx prisma init
```

```txt
prisma init이 만드는 것:
  prisma/schema.prisma  → 스키마 파일 (모델 정의)
  .env에 DATABASE_URL 주석 추가

apps/api/.env에 설정:
  DATABASE_URL="postgresql://postgres:password@localhost:5432/myapp_dev"
```

---

# PrismaService — DB 연결 담당 ⭐️⭐️⭐️⭐️

```typescript
// src/prisma/prisma.service.ts
import { Injectable, OnModuleInit, OnModuleDestroy } from '@nestjs/common';
import { PrismaClient } from '@prisma/client';

@Injectable()
export class PrismaService
  extends PrismaClient
  implements OnModuleInit, OnModuleDestroy
{
  async onModuleInit() {
    await this.$connect();
    // 앱 시작 시 DB 연결
  }

  async onModuleDestroy() {
    await this.$disconnect();
    // 앱 종료 시 연결 해제 (연결 누수 방지)
  }
}
```

```txt
extends PrismaClient:
  PrismaService가 PrismaClient를 상속
  → this.user.findMany(), this.post.create() 등 직접 사용 가능

OnModuleInit / OnModuleDestroy:
  NestJS 라이프사이클 훅
  onModuleInit  → 앱이 시작될 때 자동 실행
  onModuleDestroy → 앱이 꺼질 때 자동 실행
  → 수동으로 connect/disconnect 호출 불필요
```

---

# PrismaModule — @Global() 전역 등록 ⭐️⭐️⭐️⭐️

```typescript
// src/prisma/prisma.module.ts
import { Global, Module } from '@nestjs/common';
import { PrismaService } from './prisma.service';

@Global()
@Module({
  providers: [PrismaService],
  exports:   [PrismaService],
})
export class PrismaModule {}
```

```typescript
// app.module.ts
@Module({
  imports: [
    ConfigModule.forRoot({ isGlobal: true }),
    PrismaModule,   // 한 번만 등록 → 전체 앱에서 PrismaService 주입 가능
    AuthModule,
    PostsModule,
    ...
  ],
})
export class AppModule {}
```

```txt
@Global() 효과:
  AppModule에 한 번만 import하면
  PostsModule, UsersModule 등 모든 모듈에서
  imports: [PrismaModule] 없이 PrismaService 주입 가능

다른 서비스에서 사용:
  @Injectable()
  export class PostsService {
    constructor(private readonly prisma: PrismaService) {}

    findAll() {
      return this.prisma.post.findMany();
    }
  }
```

---

# 연결 확인 방법

```bash
# 1. Prisma로 DB 연결 테스트
pnpm --filter api exec prisma db pull
# → DB에 연결해서 테이블을 schema.prisma로 가져옴
# → 에러 없으면 연결 성공

# 2. 앱 실행 시 확인
pnpm --filter api start:dev
# → onModuleInit의 $connect()가 실행됨
# → 연결 실패 시 에러 로그 + 앱 시작 안 됨

# 3. Prisma Studio로 확인
pnpm --filter api exec prisma studio
# → 브라우저에서 DB 테이블 시각적으로 확인
```

---

# 자주 만나는 에러

| 에러                                | 원인                            | 해결                                               |
| --------------------------------- | ----------------------------- | ------------------------------------------------ |
| `Can't reach database server`     | Docker 미실행 또는 DATABASE_URL 틀림 | `docker ps` 확인, DATABASE_URL 재확인                 |
| `Authentication failed`           | DB 사용자·비밀번호 틀림                | docker-compose.yml의 POSTGRES_PASSWORD 확인         |
| `database "myapp" does not exist` | DB가 없음                        | `docker compose down -v && docker compose up -d` |
| `SSL connection required`         | Neon 등 클라우드 DB는 SSL 필수        | DATABASE_URL 끝에 `?sslmode=require` 추가            |
| `no configuration file provided`  | 루트가 아닌 곳에서 docker compose 실행  | 저장소 루트에서 `docker compose up -d` 실행               |
| `Connection refused (포트 5444 등)`  | URL 포트 ≠ compose hosts 포트     | DATABASE_URL 포트와 compose ports 좌측 포트 일치 확인       |
| user/password 인증 실패 (볼륨 있을 때)     | 기존 볼륨에 이전 비밀번호가 고정됨           | `docker compose down -v` 후 재생성                   |
| DataGrip 연결 실패                    | Docker가 안 켜져 있음               | `docker compose up -d` 실행 후 재시도                  |