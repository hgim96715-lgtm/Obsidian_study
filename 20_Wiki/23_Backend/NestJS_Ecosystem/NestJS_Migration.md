---
aliases:
  - Database
  - DataGrip
  - Migration
  - PostgreSQL
  - Prisma
tags:
  - NestJS
  - PostgreSQL
  - SQL
related:
  - "[[00_NestJS_Ecosystem_HomePage]]"
  - "[[NestJS_Prisma]]"
  - "[[NestJS_Prisma_Patterns]]"
  - "[[Monorepo_PNPM]]"
  - "[[NestJS_PostgreSQL]]"
---
# NestJS_Migration — Prisma 마이그레이션

> [!info] 
> 마이그레이션 = schema.prisma 변경을 실제 DB에 반영하는 과정. 
> **설치 → 초기화 → 반복 루프**(schema 수정 → `migrate dev` → 사용)가 핵심 흐름이고, `_prisma_migrations` 테이블이 어떤 SQL이 이미 적용됐는지 추적한다.

---

# 설치 & 초기화

```bash
pnpm add @prisma/client
pnpm add -D prisma
pnpm add @prisma/adapter-pg pg   # Prisma 7+, PostgreSQL 사용 시 필수

npx prisma init   # prisma/schema.prisma + .env 생성
```

```txt
prisma init이 만들어주는 것:
  prisma/schema.prisma  — 스키마 정의 파일 (datasource + generator 포함)
  .env                  — DATABASE_URL 환경변수 예시

⚠️ 자동 생성된 DATABASE_URL은 Prisma 클라우드 주소 — 직접 쓰는 DB 주소로 반드시 교체
```

```txt
모노레포에서 설치 순서 주의:
  pnpm-workspace.yaml로 루트 워크스페이스가 잡힌 상태에서 설치해야 함
  워크스페이스 인식 전에 apps/api 폴더 안에서 설치하면 문제가 생김

증상                                    원인
apps/api 에 pnpm-lock.yaml 이 따로 생김  워크스페이스 인식 전에 그 폴더에서 설치
apps/api 에 .git 폴더가 따로 있음        nest new 를 루트 세팅 전에 실행

해결: 중복 lockfile / .git 삭제 후 루트에서 재설치
결론: pnpm-workspace.yaml이 루트에 있고 apps/api가 정상 인식된 상태라면
      이후 설치/명령 실행은 자유롭게 선택 가능
```

---

# 초기 설정 파일

## prisma.config.ts (Prisma 7)

```typescript
import "dotenv/config";
import { defineConfig, env } from "prisma/config";

export default defineConfig({
  schema: "prisma/schema.prisma",
  datasource: { url: env("DATABASE_URL") },
});
```

```txt
이 파일만 process.env 직접 사용 — CLI(migrate/generate)는 NestJS 부팅 없이 실행되어
ConfigService를 쓸 수 없음

NestJS 런타임 / Prisma CLI / Docker Compose가 .env를 각자 다르게 읽는 전체 그림 → [[NestJS_Env_Config]]
```

## DATABASE_URL 구조

```bash
postgresql://user:password@host:port/dbname?schema=public&sslmode=disable
```

|구간|의미|
|---|---|
|`postgresql://`|DB 종류|
|`user:password`|접속 계정|
|`host:port`|접속 주소|
|`dbname`|사용할 DB|
|`?schema=`|기본값 `public`, 보통 그대로 사용|
|`sslmode=`|로컬(Docker) → `disable` / 클라우드(Neon 등) → `require`|

## schema.prisma 기본 구성 (Prisma 7)

```prisma
// prisma/schema.prisma
datasource db {
  provider = "postgresql"
}

generator client {
  provider     = "prisma-client"
  output       = "../src/generated/prisma"
  moduleFormat = "cjs"
}
```

```txt
moduleFormat = "cjs" 누락 시 흔한 에러: "exports is not defined in ES module scope"
→ Prisma 7 기본 출력(ESM)과 NestJS 빌드(CJS) 형식 불일치가 원인

output 경로: schema.prisma 기준 상대 경로
  prisma/schema.prisma 기준 ../src/generated/prisma
  → 프로젝트 루트의 src/generated/prisma/ 에 Client 코드 생성됨
```

## generator json — Json 필드에 직접 타입 붙이기 (선택)

```prisma
generator json {
  provider = "prisma-json-types-generator"
}
```

```bash
pnpm add -D prisma-json-types-generator
```

```txt
이 블록을 추가하고 패키지를 설치해야, 필드 뒤에 붙이는 /// [TypeName] 주석이 실제로 동작함
generator 블록 없이 주석만 적어두면 무시되는 일반 텍스트일 뿐

추가 후 prisma generate(또는 migrate dev)를 다시 실행해야 적용됨
사용법 → [[NestJS_Prisma]] "Json 필드" 섹션 참고
```

---

# 반복 루프 — 개발 워크플로우 ⭐️⭐️⭐️⭐️

```txt
① schema.prisma 수정
② pnpm prisma migrate dev --name 설명용_이름
   예: pnpm prisma migrate dev --name add_user_role
③ (보통 자동) pnpm prisma generate
④ 서버 재시작
⑤ this.prisma.모델명.메서드() — 타입 자동완성 즉시 적용
```

```txt
② 한 줄이 하는 일:
  변경분만 SQL 마이그레이션 파일 생성
  → 로컬 DB에 적용
  → Prisma Client 재생성 (prisma generate 자동 포함)

schema.prisma만 고치면 DB는 안 바뀜 — migrate dev 반드시 실행해야 반영됨
```

## migrate dev / migrate deploy / generate 비교

|명령|언제|동작|
|---|---|---|
|`migrate dev --name x`|로컬 개발|마이그레이션 파일 생성 + DB 적용 + Client 재생성|
|`migrate deploy`|배포/CI|기존 마이그레이션만 순서대로 적용 (새로 생성 안 함)|
|`generate`|Client만 다시|DB 변경 없이 타입만 재생성|

## 타입이 안 보일 때 체크리스트 ⭐️⭐️

```txt
① schema에 모델/필드 정확히 들어갔는지
② migrate dev (또는 generate) 실행했는지
③ 서버 재시작 — Node는 require한 모듈을 메모리에 캐싱해서,
   파일이 새로 생겨도 이미 떠 있는 프로세스는 재시작 전까지 옛 버전을 그대로 씀
   (가장 자주 빠뜨리는 단계)
④ (그래도 안 되면) 에디터 TS 서버 재시작
```

## 모노레포에서 명령 실행 — cd vs --filter ⭐️⭐️⭐️

|방법|예시|
|---|---|
|cd 후 직접|`cd apps/api && pnpm prisma migrate dev --name x`|
|`--filter` (루트에서)|`pnpm --filter api exec prisma migrate dev --name x`|

```txt
둘 다 결과 동일 — pnpm이 그 워크스페이스의 로컬 prisma 바이너리를 찾아 실행하는 것은 같음
선택 기준은 "지금 어디 있는가" 뿐:
  이미 apps/api 안에 있다 → cd 후 그냥 실행
  루트에서 그대로 (스크립트 / CI 등) → --filter exec 가 편리
```

---

# Prisma 6 → 7 주요 변경 ⭐️⭐️

|항목|6 (옛 튜토리얼)|7 (현재)|
|---|---|---|
|DB url 위치|`schema.prisma` 안|`prisma.config.ts`|
|generator provider|`prisma-client-js`|`prisma-client` + `output` 필수|
|import 경로|`@prisma/client`|`output` 지정 경로 기준|
|DB 연결|`new PrismaClient()`|adapter 필요 (`@prisma/adapter-pg` 등)|

```txt
인터넷 튜토리얼 대부분 6 기준 — "따라했는데 안 됨"의 흔한 원인이 이 표의 차이들
이런 식으로 라이브러리 메이저 버전이 바뀌며 import 경로/타입 위치가 통째로 바뀌는 건
Prisma만의 특징이 아님
```

---

# migrate dev — 로컬 개발의 핵심 ⭐️⭐️⭐️⭐️

```bash
pnpm prisma migrate dev --name add_user_role
# 예: pnpm prisma migrate dev --name user_withdraw_soft
```

```txt
--name: 마이그레이션 파일 이름에 붙는 설명 (타임스탬프_이름 형식으로 저장)
  예: 20250701120000_add_user_role

실행하면 세 가지 일을 순서대로 함:
  ① prisma/migrations/타임스탬프_이름/migration.sql 생성
  ② 로컬 DB에 SQL 적용
  ③ Prisma Client 재생성 (prisma generate 자동 실행)

적용 후 서버를 재시작해야 타입 자동완성에 반영됨 (위 체크리스트 참고)
```

## migrate dev vs db push — 언제 뭘 쓰나 ⭐️⭐️⭐️

| |`migrate dev`|`db push`|
|---|---|---|
|마이그레이션 파일|생성됨 (`prisma/migrations/`)|생성 안 됨|
|이력 추적|`_prisma_migrations` 테이블에 기록|기록 안 됨|
|배포 가능|✅ `migrate deploy`로 운영에 적용|❌ 재현 불가|
|데이터 보존|기존 데이터 유지|파괴적 변경 시 경고 또는 데이터 손실|
|용도|**실무 표준 — 항상 이걸**|빠른 프로토타이핑, 버릴 DB에서만|

```txt
db push를 쓰면 안 되는 이유:
  마이그레이션 파일이 없어서 운영 서버에 동일한 변경을 재현할 방법이 없음
  → 로컬에서 뭔가 달라졌는데 운영 DB에는 안 들어가는 상황 발생
  → 개인 학습용 임시 실험에서만 허용
```

---

# migrate deploy — 배포 흐름 ⭐️⭐️⭐️⭐️

```bash
npx prisma migrate deploy
```

```txt
동작:
  _prisma_migrations 테이블을 보고 "아직 적용 안 된 마이그레이션"만 순서대로 적용
  새 파일 생성 안 함 / Client 재생성 안 함
  CI/CD 파이프라인 또는 서버 시작 직전에 실행
```

## Dockerfile에서 실행

```dockerfile
# 앱 시작 전에 migrate deploy 실행
CMD ["sh", "-c", "npx prisma migrate deploy && node dist/main"]
```

```txt
Railway/Neon 배포 시:
  migrate deploy → 운영 DB에 미적용 마이그레이션 적용
  node dist/main → NestJS 앱 시작

⚠️ DATABASE_URL이 운영 DB를 가리켜야 함
   Railway 환경변수에 Neon 연결 문자열이 설정돼 있어야 함
```

---

# ` _prisma_migrations` 테이블 — 이력 추적 ⭐️⭐️⭐️

```txt
migrate dev / migrate deploy를 실행하면 DB에 _prisma_migrations 테이블이 자동 생성됨
어떤 마이그레이션이 언제 적용됐는지를 기록하는 이력 테이블
```

```sql
SELECT * FROM _prisma_migrations ORDER BY finished_at DESC;
```

|컬럼|의미|
|---|---|
|`migration_name`|파일 이름 (타임스탬프_이름)|
|`finished_at`|적용 완료 시각|
|`applied_steps_count`|실행된 SQL 개수|
|`rolled_back_at`|롤백된 경우 시각|
|`logs`|실패 시 에러 메시지|

```txt
migrate deploy가 "이미 적용된 것"을 건너뛰는 원리:
  migration_name이 이 테이블에 있으면 → 이미 적용됨 → 건너뜀
  없으면 → 미적용 → SQL 실행 후 테이블에 기록

→ 같은 마이그레이션이 두 번 실행되는 일이 없는 이유
```

---

# 마이그레이션 파일 구조 — Git에 반드시 커밋 ⭐️⭐️⭐️

```txt
prisma/migrations/
  20250701120000_init/
    migration.sql
  20250710090000_add_role/
    migration.sql
  migration_lock.toml
```

```txt
왜 Git에 올려야 하는가:
  마이그레이션 파일 = "이 시점에 DB가 이렇게 바뀌었다"는 이력
  팀원이 pull하면 migrate dev로 같은 상태로 맞출 수 있음
  운영 배포 시 migrate deploy가 이 파일들을 순서대로 실행함

⚠️ .gitignore에 prisma/migrations/ 넣으면 안 됨 — 흔한 실수

migration_lock.toml:
  provider = "postgresql" 같은 DB 종류를 기록
  다른 provider로 실수로 migrate하는 걸 방지
```

---

# --create-only — 커스텀 SQL 추가 ⭐️⭐️⭐️⭐️

```txt
Prisma가 자동으로 생성하는 SQL로는 표현 못 하는 것들:
  - 부분 인덱스 (CREATE INDEX ... WHERE 조건)
  - ENUM 값 추가 (ALTER TYPE ... ADD VALUE)
  - 커스텀 함수, 트리거
  - 기존 데이터 변환 로직
```

```bash
# 1. 파일만 생성 (DB 적용 안 함)
pnpm prisma migrate dev --name add_partial_index --create-only

# 2. 생성된 migration.sql에 커스텀 SQL 직접 추가

# 3. 수동 편집한 SQL로 DB 적용
pnpm prisma migrate dev
```

```sql
-- Prisma가 생성한 기본 내용 아래에 추가
CREATE INDEX idx_active_post ON "Post"(created_at)
WHERE is_active = TRUE;
-- ↑ 부분 인덱스 — Prisma 스키마로 표현 불가, SQL 직접 작성
```

## ENUM 값 추가 — 특수 케이스

```sql
-- ❌ 자동 생성 시 트랜잭션 안에 들어가서 에러 (PostgreSQL 제약)
BEGIN;
ALTER TYPE "UserRole" ADD VALUE 'MODERATOR';
COMMIT;

-- ✅ 트랜잭션 밖으로 이동 + IF NOT EXISTS
ALTER TYPE "UserRole" ADD VALUE IF NOT EXISTS 'MODERATOR';
```

```txt
PostgreSQL에서 ENUM 값 추가는 트랜잭션 안에서 실행 불가
--create-only로 파일을 만든 뒤 BEGIN ~ COMMIT 밖으로 꺼내야 함

IF NOT EXISTS:
  값이 이미 있어도 에러 안 남 → 멱등성 보장
  migrate deploy를 여러 번 실행해도 안전
```

---

# migrate reset — 로컬 DB 초기화 ⭐️⭐️⭐️

```bash
pnpm prisma migrate reset
```

```txt
동작 순서:
  ① DB 전체 드롭
  ② DB 재생성
  ③ 전체 마이그레이션 처음부터 순서대로 재적용
  ④ seed 스크립트 실행 (있으면)

언제 쓰나:
  마이그레이션이 꼬여서 개발 DB 상태가 이상할 때
  완전히 처음부터 깨끗하게 다시 시작하고 싶을 때

⚠️ 로컬 전용 — 운영 DB에 절대 실행 금지 (데이터 전부 삭제)
⚠️ 실행 전 확인 프롬프트 → CI 환경에서 건너뛰려면 --force 플래그
```

---

# migrate resolve — 실패한 마이그레이션 복구 ⭐️⭐️

```bash
# 성공적으로 적용됐다고 표시
npx prisma migrate resolve --applied 20250701120000_add_user_role

# 롤백됐다고 표시
npx prisma migrate resolve --rolled-back 20250701120000_add_user_role
```

```txt
언제 필요한가:
  마이그레이션 실행 중 오류 발생 → _prisma_migrations에 실패 기록이 남음
  → 수동으로 SQL을 고쳐서 DB에 직접 적용했을 때
  → Prisma에게 "이거 이미 처리했어"라고 알려주는 용도

일반적인 복구 흐름:
  ① DataGrip에서 실패 원인 파악 + 수동 SQL 실행
  ② migrate resolve --applied로 Prisma 이력 정리
  ③ migrate deploy 또는 migrate status로 정상 확인
```

---

# migrate diff — 스키마 비교

```bash
# schema.prisma와 실제 DB 사이의 차이를 SQL로 출력
npx prisma migrate diff \
  --from-schema-datamodel prisma/schema.prisma \
  --to-schema-datasource prisma/schema.prisma
```

```txt
언제 유용한가:
  migrate dev 실행 전 "어떤 SQL이 생성될지" 미리 확인
  schema.prisma와 DB가 얼마나 차이나는지 점검
  변경이 의도한 대로인지 검토 후 migrate dev 실행
```

---

# migrate status — 현황 확인

```bash
npx prisma migrate status
```

```txt
출력 예시:
  ✔ Database schema is up to date!     → 모든 마이그레이션 적용됨
  ✖ 1 migration not yet applied        → 미적용 마이그레이션 있음

배포 전 점검: "운영 DB에 아직 안 들어간 마이그레이션이 있는가" 확인용
```

---

# db seed — 초기 데이터 삽입 ⭐️⭐️

```bash
npx prisma db seed
```

```typescript
// prisma/seed.ts
import { PrismaClient } from '../src/generated/prisma';

const prisma = new PrismaClient();

async function main() {
  await prisma.user.upsert({
    where: { email: 'admin@example.com' },
    create: { email: 'admin@example.com', role: 'ADMIN' },
    update: {},
  });
}

main()
  .catch(console.error)
  .finally(() => prisma.$disconnect());
```

```json
// package.json — prisma 키 추가 필요
{
  "prisma": {
    "seed": "ts-node prisma/seed.ts"
  }
}
```

```txt
seed 실행 시점:
  migrate reset 시 자동으로 seed도 함께 실행됨
  npx prisma db seed으로 단독 실행도 가능

upsert를 쓰는 이유:
  seed를 여러 번 실행해도 중복 데이터가 생기지 않음 (멱등성)
  create 대신 upsert → 있으면 update, 없으면 create

자세한 seed 전략(DRY_RUN · cleanup · $transaction) → [[NestJS_Seed]]
```

---

# 명령어 전체

|명령|언제|동작|
|---|---|---|
|`migrate dev --name x`|로컬 개발|마이그레이션 파일 생성 + DB 적용 + Client 재생성|
|`migrate dev --create-only`|커스텀 SQL 추가 전|파일만 생성, DB 적용 안 함|
|`migrate deploy`|배포/CI|미적용 마이그레이션만 순서대로 적용|
|`migrate reset`|로컬 초기화|DB 드롭 → 재생성 → 전체 재적용 → seed|
|`migrate status`|현황 확인|적용된/미적용 목록|
|`migrate resolve`|실패 복구|수동 처리 후 Prisma 이력 정리|
|`migrate diff`|스키마 비교|두 스키마 간 차이를 SQL로 출력|
|`db push`|프로토타이핑|마이그레이션 없이 즉시 동기화 (운영 금지)|
|`db seed`|초기 데이터|seed 스크립트 실행|
|`generate`|타입만|DB 변경 없이 Prisma Client 재생성|

---

# 자주 만나는 에러

|증상|원인|해결|
|---|---|---|
|`There are N migrations that have not yet been applied`|migrate deploy 안 함|`npx prisma migrate deploy` 실행|
|`Migration failed to apply cleanly`|SQL 실행 중 에러|DataGrip에서 원인 파악 → 수동 수정 → `migrate resolve`|
|`Drift detected`|DB와 schema가 다른데 마이그레이션 파일이 없음|`migrate dev`로 차이를 새 파일로 만들거나 `migrate reset`|
|ENUM ADD VALUE 트랜잭션 에러|PostgreSQL ENUM 제약|migration.sql에서 트랜잭션 블록 밖으로 이동|
|`migration_lock.toml` 충돌|다른 provider로 migrate 시도|provider 확인 + lock 파일 재생성|
|`exports is not defined in ES module scope`|`moduleFormat = "cjs"` 누락|schema.prisma generator 블록에 추가|

```txt
운영 중 스키마를 안전하게 변경해야 한다면 (컬럼 이름 변경, 타입 변경, NOT NULL 추가 등):
→ [[DB_MigrationPattern]] Expand-Contract 패턴 참고
  (migrate deploy 한 번으로 끝내는 게 아니라 배포를 단계별로 분리하는 전략)
```