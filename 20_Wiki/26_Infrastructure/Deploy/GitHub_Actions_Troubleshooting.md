---
aliases:
  - GitHub Actions 오류
  - CI/CD 트러블슈팅
tags:
  - Git
  - Deploy
related:
  - "[[00_Deploy_HomePage]]"
  - "[[GitHub_Deploy]]"
  - "[[NestJS_Migration]]"
---

# GitHub_Actions_Troubleshooting — CI/CD 오류 모음

```txt
GitHub Actions 에서 만난 에러 / 원인 / 해결법 기록
```

---

---

# TS5011 / TS2564 — 빌드 에러

## 오류

```txt
error TS5011: 'rootDir' is expected to contain all source files.
error TS2564: Property has no initializer and is not definitely assigned
```

## 원인 

```txt
TS5011:
  tsconfig.build.json 에 rootDir / include 가 없으면
  TypeScript 가 컴파일 범위를 src/ 로 한정하지 못함
  → src/ 밖의 파일도 포함하려다 rootDir 범위 초과 에러

TS2564:
  NestJS 에서 클래스 프로퍼티를 constructor 가 아닌
  DI(의존성 주입) 로 초기화하는 패턴을 사용
  → TypeScript strict 모드가 "초기값 없다" 고 에러
  → strictPropertyInitialization: false 로 해제 필요
```

## 해결

```json
// tsconfig.build.json — rootDir / include 추가
{
  "extends": "./tsconfig.json",
  "compilerOptions": {
    "rootDir": "./src",
    "outDir": "./dist"
  },
  "include": ["src/**/*"],
  "exclude": ["node_modules", "test", "dist"]
}
```

```json
// tsconfig.json — strictPropertyInitialization 추가
{
  "compilerOptions": {
    "strictPropertyInitialization": false
  }
}
```

## Prisma generated 파일 포함 시 TS5011 재발 ⭐️

```txt
Prisma output 이 src 바깥 (generated/prisma/...) 에 있을 때

import { Role } from '../../generated/prisma/prisma/client'
                        ↑ src 바깥 폴더

rootDir: './src' 이면
  src 바깥 파일을 참조하는 순간 TS5011 다시 발생
  "rootDir 에 포함되지 않은 파일을 참조함"
```

## 해결 — rootDir 을 프로젝트 루트로 변경

```json
// tsconfig.build.json
{
  "extends": "./tsconfig.json",
  "compilerOptions": {
    "rootDir": ".",                             // src 대신 프로젝트 루트
    "tsBuildInfoFile": "./dist/.tsbuildinfo"
  },
  "include": ["src/**/*"],
  "exclude": ["node_modules", "test", "dist", "**/*spec.ts"]
}
```

```txt
rootDir: "." 로 바꾸면
  src 바깥의 generated/ 경로 import 허용
  Prisma generated client import 정상 동작

⚠️ Procfile 경로도 같이 바꿔야 함
  rootDir: "./src"  → dist/main.js     → web: node dist/main.js
  rootDir: "."      → dist/src/main.js → web: node dist/src/main.js
  빌드 후 dist/ 폴더 구조 확인 후 맞춰서 작성
```

---
---
# package.json scripts 맞추기 ⭐️

```txt
tsconfig.build.json 의 rootDir 을 바꾸면
dist/ 폴더 구조가 바뀌기 때문에
package.json scripts 경로도 전부 맞춰야 함

rootDir: "."  → dist/src/main.js
rootDir: "./src" → dist/main.js
```

```json
{
  "scripts": {
    "build":              "prisma generate && nest build",
    //                     ↑ Prisma Client 먼저 생성 후 빌드
    "migration:run":      "typeorm migration:run -d ./dist/src/database/data-source.js",
    "migration:deploy":   "prisma migrate deploy",
    "start":              "node dist/src/main.js",
    //                     ↑ rootDir: "." 이면 dist/src/ 경로
    "start:dev":          "nest start --watch",
    "start:dev:worker":   "export TYPE=worker && export PORT=3002 && nest start --watch",
    "start:dev:clean":    "pnpm run clean && nest start --watch",
    "start:debug":        "nest start --debug --watch",
    "start:prod":         "node dist/src/main"
  }
}
```

```txt
build 에 prisma generate 추가하는 이유:
  Prisma Client 가 generated/ 에 생성되어 있어야 빌드 가능
  CI 에서 generate 없이 build 하면 import 에러

start / start:prod 경로 차이:
  rootDir: "."  → dist/src/main.js
  rootDir: "./src" → dist/main.js
  → tsconfig.build.json 과 반드시 일치해야 함

Procfile 도 동일하게:
  web: node dist/src/main.js
```
---

---

# Missing script: "typeorm"

## 오류

```txt
npm error Missing script: "typeorm"
```

## 원인

```txt
npm install -g typeorm 으로 전역 설치해도
npm run typeorm 은 package.json scripts 에 있어야 동작
전역 설치와 npm run 은 별개
```

## 해결

```json
{
  "scripts": {
    "migration:run": "pnpm build && typeorm migration:run -d ./dist/database/data-source.js"
  }
}
```

---

---

# no pg_hba.conf entry — RDS SSL 오류 ⭐️

## 오류

```txt
no pg_hba.conf entry for host "172.184.x.x", user "...", database "...", no encryption
```

## 원인

```txt
GitHub Actions runner 가 RDS 에 SSL 없이 접속 시도
RDS 는 SSL 연결만 허용 → 거절

app.module.ts 는 ENV=prod 일 때 SSL 을 켜는데
migration 용 data-source.ts 에는 SSL 설정이 없었음
GitHub Secrets 에 ENV=dev 가 있으면 SSL 꺼짐 → 에러
```

## 해결 — DB_HOST 기반으로 SSL 자동 판단 ⭐️

```typescript
// ❌ 기존 — ENV 값에 의존 (ENV=dev 면 SSL 꺼짐)
const isProd = process.env.ENV === 'prod';
...(isProd ? { ssl: { rejectUnauthorized: false } } : {})

// ✅ 수정 — localhost 가 아니면 SSL 자동 활성화
const isRemoteDB = process.env.DB_HOST !== 'localhost'
                && process.env.DB_HOST !== '127.0.0.1';
...(isRemoteDB ? { ssl: { rejectUnauthorized: false } } : {})
```

```typescript
// data-source.ts 최종
import * as dotenv from 'dotenv';
import { DataSource } from 'typeorm';

dotenv.config();

const isRemoteDB = process.env.DB_HOST !== 'localhost'
                && process.env.DB_HOST !== '127.0.0.1';

export default new DataSource({
  type:     process.env.DB_TYPE as 'postgres',
  host:     process.env.DB_HOST,
  port:     Number(process.env.DB_PORT || 5432),
  username: process.env.DB_USERNAME,
  password: process.env.DB_PASSWORD,
  database: process.env.DB_DATABASE,
  logging:  false,
  entities:   [__dirname + '/../**/*.entity{.ts,.js}'],
  synchronize: false,
  migrations: [__dirname + '/../database/migrations/*.{ts,js}'],
  ...(isRemoteDB ? { ssl: { rejectUnauthorized: false } } : {}),
});
```

---

---

# ncu (npm-check-updates) CI 에서 금지 ⭐️

```txt
배포 파이프라인에서 ncu -u 같은 자동 업그레이드 절대 금지

이유:
  CI 실행마다 패키지 버전이 바뀜
  로컬과 다른 TS / 패키지 → 빌드 실패 / 재현 불가 에러

올바른 방법:
  npm install --legacy-peer-deps
  또는 pnpm install --frozen-lockfile  (lock 파일 기반 정확한 버전)
```

---
---
# EB nginx 502 — 앱 기동 실패 ⭐

## 오류

```txt
connect() failed (111: Connection refused) upstream: "http://127.0.0.1:8080/"
```

## 원인

```txt
EB nginx 는 8080 포트로 Node 앱에 프록시
그 포트에 아무것도 listen 하지 않음
→ 앱이 기동 전에 크래시됐거나 아직 배포 전 버전이 돌고 있는 것

eb.stdout.log 에서 실제 에러 원인 확인
```

---
---
# Cannot find module '@redis/client/dist/lib/commands/reply-utils'

## 오류

```txt
Cannot find module '@redis/client/dist/lib/commands/reply-utils'
```

## 원인

```txt
redis@6.0.0 과 @keyv/redis@5.1.6 이 서로 다른 @redis/client 버전 요구

  패키지                요구 버전
  redis@6.0.0          @redis/client@6.0.0 (reply-utils 포함)
  @keyv/redis@5.1.6    @redis/client@5.12.1 (reply-utils 없음)

로컬(pnpm): 패키지별 의존성 분리 → 정상 동작
CI(npm):    flat node_modules → v5.12.1 최상위 → reply-utils 없음 → 크래시
```

## 해결

```json
// package.json
{
  "overrides": {
    "@redis/client": "6.0.0"
  }
}
```

```txt
# .npmrc
legacy-peer-deps=true
```

### yaml

```yaml
# deploy.yml
- name: Install dependencies
  run: npm ci --legacy-peer-deps
```

---

---

# ECONNREFUSED 127.0.0.1:6379 — Redis 연결 실패

## 오류

```txt
ECONNREFUSED 127.0.0.1:6379
```

## 원인

```txt
EB 인스턴스에는 로컬 Redis 없음
REDIS_HOST 가 localhost / 127.0.0.1 로 설정된 것

GitHub Secrets 에서 확인:
  REDIS_HOST → ElastiCache 엔드포인트로 변경
  예: my-redis.xxxxx.apn2.cache.amazonaws.com
```

---

---

# Prisma P1011 — RDS SSL self-signed certificate

## 오류

```txt
P1011: self-signed certificate in certificate chain
→ 500 Internal Server Error
```

## 원인

```txt
RDS 는 SSL 연결 강제
Prisma 가 RDS 의 자체 서명 인증서를 신뢰하지 않아서 거부
```

## 해결 - isLocalDb 패턴 ⭐️

```typescript
// prisma.service.ts — 로컬 아닐 때 SSL 설정
const dbHost = process.env.DB_HOST ?? '';
const isLocalDb = ['localhost', '127.0.0.1', 'postgres'].includes(dbHost);

const adapter = new PrismaPg({
  connectionString: process.env.DATABASE_URL,
  ...(!isLocalDb
    ? { ssl: { rejectUnauthorized: false } }
    : {}),
});
```

```txt
const dbHost = process.env.DB_HOST ?? '':
  DB_HOST 환경변수 읽기
  ?? '' → undefined / null 이면 빈 문자열로 대체

isLocalDb = ['localhost', '127.0.0.1', 'postgres'].includes(dbHost):
  dbHost 가 배열 안에 있으면 true (로컬 환경)
  없으면 false (RDS 등 원격)

  왜 'postgres' 도 포함하나:
    Docker Compose 에서 서비스 이름이 'postgres' 이면
    컨테이너끼리 통신할 때 hostname 이 'postgres' 로 잡힘
    → 로컬 Docker 환경도 SSL 불필요로 처리

...(!isLocalDb ? { ssl: ... } : {}):
  isLocalDb 가 false (RDS) → ssl 옵션 추가
  isLocalDb 가 true (로컬) → 빈 객체 → ssl 옵션 없음
  스프레드 연산자로 조건부 병합
```

```txt
rejectUnauthorized: false 란:
  SSL 연결은 하되 인증서 검증을 건너뜀
  RDS 는 자체 서명 인증서(self-signed) 를 사용
  → Node.js 기본 SSL 검증이 이 인증서를 신뢰 안 함
  → rejectUnauthorized: false 로 검증 스킵

  true  인증서 유효성 검사 O (공인 CA 서명 인증서 필요)
  false 인증서 유효성 검사 X (연결만 암호화, 검증 안 함)

  ⚠️ 보안 주의:
    완전히 안전한 방법은 RDS CA 인증서를 직접 등록하는 것
    포트폴리오 / 개인 프로젝트 수준에서는 false 로 충분
```

### DATABASE_URL 추가

```txt
DATABASE_URL 에도 추가:
  postgresql://...?sslmode=no-verify
  sslmode=no-verify = Prisma CLI (migrate 등) 가 사용하는 URL 파라미터
  rejectUnauthorized: false = PrismaPg 어댑터 코드 레벨 설정
  → 둘 다 설정해야 CLI 와 앱 모두 정상 동작
```

---

---

# Prisma P1013 — invalid port number in DATABASE_URL

## 오류

```txt
P1013: The provided database string is invalid. invalid port number in database URL
```

## 원인

```txt
DB_PASSWORD 에 : 같은 특수문자가 있으면
postgresql://user:pass@host:5432/db 형태의 URL 이 깨짐
Prisma 가 : 를 포트 구분자로 잘못 읽음

TypeORM 은 DB_PASSWORD 를 개별 파라미터로 쓰므로 문제없음
Prisma 는 DATABASE_URL 전체를 파싱하므로 특수문자 주의
```

## 해결

```yaml
# deploy.yml Create Env File 단계에서 인코딩
- name: Create Env File
  run: |
    printf 'DB_PASSWORD=%s\n' "${DB_PASSWORD}"
    # echo 대신 printf 사용 — 특수문자 안전하게 처리
```

---

---

# Prisma P3005 — 기존 DB 스키마 충돌 (baseline)

## 오류

```txt
P3005: The database schema is not empty.
```

## 원인

```txt
RDS 에 TypeORM 등으로 테이블이 이미 있는데
Prisma _prisma_migrations 이력이 없을 때 migrate deploy 거부
```

## 해결 — baseline 등록

```yaml
- name: Run Prisma migrations (RDS)
  run: |
    set +e   # P3005 실패해도 다음 줄 실행
    npm run migration:deploy > migrate.log 2>&1
    status=$?
    cat migrate.log
    if [ "$status" -ne 0 ] && grep -q 'P3005' migrate.log; then
      echo "Baselining Prisma migration history..."
      npx prisma migrate resolve --applied "20260605052743_init"
      npm run migration:deploy
    elif [ "$status" -ne 0 ]; then
      exit "$status"
    fi
```

```txt
set +e 이유:
  GitHub Actions 기본 set -e → P3005 실패 시 step 즉시 종료
  set +e 로 exit code 직접 처리 → baseline 로직 실행 가능

prisma migrate resolve --applied "마이그레이션명":
  해당 migration 을 "이미 적용됨" 으로 등록
  SQL 재실행 없이 이력만 생성

migration 이름 확인:
  prisma/migrations/ 폴더 안 디렉토리 이름
```

---

---

# RDS 테이블 없음 → npx prisma db push / migrate deploy

```txt
로컬에서 RDS 에 직접 스키마 반영 (빠른 테스트):
  export DATABASE_URL="postgresql://...?sslmode=no-verify"
  npx prisma db push --skip-generate

DataGrip 연결:
  jdbc:postgresql://rds-endpoint:5432/prisma

운영 배포 시 GitHub Actions 에 추가:
  - name: Run Prisma migrations
    run: npx prisma migrate deploy
    env:
      DATABASE_URL: ${{ secrets.DATABASE_URL }}

migrate deploy vs db push:
  migrate deploy  migration 파일 기반 / 운영 권장
  db push         즉시 반영 / 프로토타입·빠른 테스트용
```
---
---
# CI — Cannot find module .../generated/prisma

## 오류

```txt
Cannot find module '../../generated/prisma/prisma/client'
```

## 원인

```txt
generated/ 폴더는 .gitignore 에 포함
CI 환경에는 생성된 Prisma Client 가 없음
빌드 전에 prisma generate 를 실행해야 함
```

## 해결

```json
// package.json
{
  "scripts": {
    "build": "prisma generate && nest build"
  }
}
```

---

---

# migration already exists — RDS 스키마 충돌

## 오류

```txt
Table already exists / migration already exists
```

## 원인

```txt
synchronize: true 로 TypeORM 이 이미 테이블 생성
그 상태에서 migration:run / migration:deploy 실행
→ 이미 있는 테이블을 다시 만들려고 충돌
```

## 해결

```txt
빈 RDS 라면:
  DROP SCHEMA public CASCADE;
  CREATE SCHEMA public;
  → migration:run 또는 migration:deploy 재실행

기존 테이블이 있고 날리기 어려우면:
  → Prisma P3005 baseline 처리 참고
```

---

---

# S3 deploy AccessDenied

## 오류

```txt
AccessDenied — Upload To S3 step 실패
```

## 원인

```txt
IAM 사용자에 s3:PutObject 권한 없음
또는 IAM 정책의 Resource ARN 이 실제 버킷명과 다름
  예: arn:aws:s3:::my-placeholder-bucket/* ← 실제 버킷 아님
```

## 해결


```json
// IAM 인라인 정책 Resource 확인
{
  "Resource": "arn:aws:s3:::실제-버킷-이름/*"
}
```

---

---

# EB 502 Bad Gateway

## 원인 체크리스트

```txt
1. .env 에 필수 환경변수 누락
   AWS_REGION / AWS_S3_BUCKET / DATABASE_URL

2. Redis 연결 실패
   REDIS_HOST 가 localhost → ElastiCache endpoint 로 변경

3. RDS 연결 실패
   DATABASE_URL sslmode=no-verify 누락

4. 앱 기동 자체 실패
   EB Logs → eb.stdout.log 에서 실제 에러 확인
   dist/src/main.js 경로 Procfile 과 일치하는지 확인
```

---

---

# Swagger — authorization required

```txt
Try it out 클릭 후 Authorize 먼저 해야 함

Authorize 방법:
  Authorize 버튼 클릭
  → Basic Auth 선택
  → Username: 이메일 / Password: 비밀번호 입력
  → Body JSON 아님 — Authorize 버튼에서 입력
```