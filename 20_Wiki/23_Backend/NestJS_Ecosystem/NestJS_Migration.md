---
aliases: [Database, DataGrip, Migration, PostgreSQL, Prisma]
tags: [NestJS, PostgreSQL, SQL]
related:
  - "[[00_NestJS_Ecosystem_HomePage]]"
  - "[[DB_MigrationPattern]]"
  - "[[Deploy_CloudMVP]]"
  - "[[NestJS_PostgreSQL]]"
  - "[[NestJS_Prisma_Monorepo]]"
  - "[[NestJS_Prisma]]"
---
# NestJS_Migration — Prisma 마이그레이션

> [!info] 
> 마이그레이션 = schema.prisma 변경을 실제 DB에 반영하는 과정
>  `migrate dev`(로컬 개발) → `migrate deploy`(운영 배포) 가 기본 흐름이고, `_prisma_migrations` 테이블이 어떤 SQL이 이미 적용됐는지 추적한다.

---

# 명령어 한눈에

|명령|언제|동작|
|---|---|---|
|`migrate dev --name x`|로컬 개발|마이그레이션 파일 생성 + 로컬 DB 적용 + Client 재생성|
|`migrate dev --create-only`|커스텀 SQL 추가 전|파일만 생성, DB 적용 안 함 — SQL 직접 편집 후 `migrate dev` 재실행|
|`migrate deploy`|배포/CI|미적용 마이그레이션만 순서대로 적용 (생성 안 함)|
|`migrate reset`|로컬 초기화|DB 드롭 → 재생성 → 전체 마이그레이션 재적용 → seed 실행|
|`migrate status`|현황 확인|적용된/미적용 마이그레이션 목록|
|`migrate resolve`|실패 복구|실패한 마이그레이션을 수동으로 적용됨/롤백됨으로 표시|
|`migrate diff`|스키마 비교|두 스키마 간 차이를 SQL로 출력|
|`db push`|프로토타이핑|마이그레이션 파일 없이 schema → DB 즉시 동기화|
|`db seed`|초기 데이터|seed 스크립트 실행|
|`generate`|타입만|DB 변경 없이 Prisma Client 타입만 재생성|

---

# migrate dev — 로컬 개발의 핵심 ⭐️⭐️⭐️⭐️

```bash
npx prisma migrate dev --name add_user_role
```

```txt
--name: 마이그레이션 파일 이름 (타임스탬프_이름 형식으로 생성됨)
  예: 20250701120000_add_user_role

한 번 실행이 세 가지 일을 함:
  ① prisma/migrations/타임스탬프_이름/migration.sql 생성
  ② 로컬 DB에 SQL 적용
  ③ Prisma Client 재생성 (generate 자동 실행)

schema.prisma만 고치면 DB는 안 바뀜 → migrate dev 필수
타입 자동완성도 ③ 이후에 반영됨 → 서버 재시작까지 해야 실제로 보임
([[NestJS_Prisma]] "타입이 안 보일 때 체크리스트" 참고)
```

---

# migrate dev vs db push — 언제 뭘 쓰나 ⭐️⭐️⭐️⭐️

| |`migrate dev`|`db push`|
|---|---|---|
|마이그레이션 파일|생성됨 (`prisma/migrations/`)|생성 안 됨|
|이력 추적|`_prisma_migrations` 테이블에 기록|기록 안 됨|
|배포 가능|✅ `migrate deploy`로 운영에 적용|❌ 재현 불가|
|데이터 보존|기존 데이터 유지|파괴적 변경 시 경고 또는 데이터 손실|
|용도|**실무 표준 — 항상 이걸**|빠른 프로토타이핑, 버릴 DB에서만|

```txt
db push를 쓰면 안 되는 이유:
  마이그레이션 파일이 없어서 팀원/운영 서버에 동일한 변경을 재현할 방법이 없음
  → 로컬에서 뭔가 달라졌는데 운영 DB에는 안 들어가는 상황 발생
  → 개인 학습용 임시 실험에서만 사용
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
  → CI/CD나 서버 시작 전에 실행하는 용도

migrate dev와의 차이:
  dev   → 개발 환경 전용. 스키마와 DB 불일치 감지 + 파일 생성 + 적용
  deploy → 운영 환경 전용. 이미 만들어진 파일만 순서대로 실행
```

## 배포 시 실행 위치

```dockerfile
# Dockerfile CMD 예시 — 앱 시작 전에 migrate deploy 실행
CMD ["sh", "-c", "npx prisma migrate deploy && node dist/main"]
```

```txt
Railway/Neon 배포 시:
  migrate deploy → 운영 DB에 미적용 마이그레이션 적용
  node dist/main → NestJS 앱 시작

주의: migrate deploy는 DATABASE_URL이 운영 DB를 가리켜야 함
  Railway 환경변수에 Neon 연결 문자열이 설정돼 있어야 함
  ([[Deploy_CloudMVP]] Dockerfile 섹션 참고)
```

---

# _prisma_migrations 테이블 — 이력 추적 ⭐️⭐️⭐️

```txt
migrate dev / migrate deploy를 실행하면 DB에 _prisma_migrations 테이블이 자동 생성됨
어떤 마이그레이션이 언제 적용됐는지를 기록하는 이력 테이블
```

```sql
-- DataGrip에서 직접 확인 가능
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
    migration.sql          ← 실제 실행될 SQL
  20250710090000_add_role/
    migration.sql
  migration_lock.toml      ← DB 종류(provider) 기록, 충돌 방지용
```

```txt
왜 Git에 올려야 하는가:
  마이그레이션 파일 = "이 시점에 DB가 이렇게 바뀌었다"는 이력
  팀원이 pull하면 migrate dev로 같은 상태로 맞출 수 있음
  운영 배포 시 migrate deploy가 이 파일들을 순서대로 실행함

.gitignore에 prisma/migrations/ 넣으면 안 됨 ← 흔한 실수

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

→ --create-only로 파일만 생성 → SQL 직접 편집 → migrate dev로 적용
```

```bash
# 1. 파일만 생성 (DB 적용 안 함)
npx prisma migrate dev --name add_partial_index --create-only

# 2. 생성된 파일에 커스텀 SQL 추가
# prisma/migrations/20250701_add_partial_index/migration.sql
```

```sql
-- Prisma가 생성한 기본 내용 아래에 추가
CREATE INDEX idx_active_post ON "Post"(created_at)
WHERE is_active = TRUE;
-- ↑ 부분 인덱스 — Prisma 스키마로 표현 불가, SQL 직접 작성
```

```bash
# 3. 수동 편집한 SQL로 DB 적용
npx prisma migrate dev
```

## ENUM 값 추가 — 특수 케이스

```sql
-- ENUM 값 추가는 트랜잭션 안에서 못 씀 (PostgreSQL 제약)
-- Prisma가 생성한 migration.sql에서 트랜잭션 블록 밖으로 꺼내야 함

-- ❌ 자동 생성 시 트랜잭션 안에 들어가서 에러
BEGIN;
ALTER TYPE "UserRole" ADD VALUE 'MODERATOR';
COMMIT;

-- ✅ 트랜잭션 밖으로 이동
ALTER TYPE "UserRole" ADD VALUE IF NOT EXISTS 'MODERATOR';
```

```txt
IF NOT EXISTS:
  값이 이미 있어도 에러 안 남 → 멱등성 보장
  migrate deploy를 여러 번 실행해도 안전
```

---

# migrate reset — 로컬 DB 초기화 ⭐️⭐️⭐️

```bash
npx prisma migrate reset
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
⚠️ 프롬프트로 확인 요청 → --force 플래그로 건너뛸 수 있음 (CI 환경)
```

---

# migrate resolve — 실패한 마이그레이션 복구 ⭐️⭐️

```bash
# 이 마이그레이션을 "성공적으로 적용됐다"고 표시
npx prisma migrate resolve --applied 20250701120000_add_user_role

# 이 마이그레이션을 "롤백됐다"고 표시
npx prisma migrate resolve --rolled-back 20250701120000_add_user_role
```

```txt
언제 필요한가:
  마이그레이션 실행 중 오류 발생 → _prisma_migrations에 실패 기록이 남음
  → 수동으로 SQL을 고쳐서 DB에 직접 적용했을 때
  → Prisma에게 "이거 이미 처리했어"라고 알려주는 용도

--applied:  성공 처리 → 다음 migrate deploy에서 건너뜀
--rolled-back: 롤백 처리 → 다음 migrate deploy에서 다시 시도

일반적인 복구 흐름:
  ① DataGrip에서 실패 원인 파악 + 수동 SQL 실행
  ② migrate resolve --applied로 Prisma 이력 정리
  ③ 이후 migrate deploy 정상 동작 확인
```

---

# migrate diff — 스키마 비교

```bash
# 현재 schema.prisma와 실제 DB 차이를 SQL로 출력
npx prisma migrate diff \
  --from-schema-datamodel prisma/schema.prisma \
  --to-schema-datasource prisma/schema.prisma

# 두 마이그레이션 파일 간 차이
npx prisma migrate diff \
  --from-migrations prisma/migrations \
  --to-schema-datamodel prisma/schema.prisma
```

```txt
언제 유용한가:
  migrate dev 실행 전 "어떤 SQL이 생성될지" 미리 확인
  schema.prisma와 DB가 얼마나 차이나는지 점검
  변경이 의도한 대로인지 검토 후 migrate dev 실행
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
// package.json에 추가 필요
{
  "prisma": {
    "seed": "ts-node prisma/seed.ts"
  }
}
```

```txt
seed 실행 시점:
  migrate reset 실행 시 자동으로 seed도 같이 실행됨
  수동으로 npx prisma db seed로 단독 실행 가능

upsert를 쓰는 이유:
  seed를 여러 번 실행해도 중복 데이터가 생기지 않음 (멱등성)
  create 대신 upsert → 있으면 update, 없으면 create

모노레포에서 명령 위치 → [[NestJS_Prisma_Monorepo]] 참고
```

---

# migrate status — 현황 확인

```bash
npx prisma migrate status
```

```txt
출력 예시:
  ✔ Database schema is up to date!         ← 모든 마이그레이션 적용됨
  ✖ 1 migration not yet applied           ← 미적용 마이그레이션 있음

배포 전 점검용으로 유용:
  "운영 DB에 아직 안 들어간 마이그레이션이 있는가" 확인
```

---

# 자주 만나는 에러

|증상|원인|해결|
|---|---|---|
|`There are N migrations that have not yet been applied`|migrate deploy 안 함|`npx prisma migrate deploy` 실행|
|`Migration failed to apply cleanly`|SQL 실행 중 에러|DataGrip에서 원인 파악 → 수동 수정 → `migrate resolve`|
|`Drift detected`|DB와 schema가 다른데 마이그레이션 파일이 없음|`migrate dev`로 차이를 새 파일로 만들거나 `migrate reset`|
|ENUM ADD VALUE 트랜잭션 에러|PostgreSQL ENUM 제약|migration.sql에서 트랜잭션 블록 밖으로 이동|
|`migration_lock.toml` 충돌|다른 provider로 migrate 시도|provider 확인 + lock 파일 재생성|

---

# 한눈에

```txt
기본 흐름:
  schema.prisma 수정
  → migrate dev --name 이름  (로컬: 파일 생성 + DB 적용 + Client 재생성)
  → Git 커밋 (prisma/migrations/ 폴더 포함)
  → 배포: migrate deploy  (운영: 미적용 파일만 순서대로 실행)

Prisma가 못 하는 SQL 추가:
  migrate dev --create-only → 파일 직접 편집 → migrate dev

migrate dev vs db push:
  dev   → 파일 생성 + 이력 추적 → 운영 배포 가능 (항상 이걸)
  push  → 파일 없음, 이력 없음 → 프로토타입 전용

_prisma_migrations 테이블:
  어떤 마이그레이션이 적용됐는지 추적
  deploy는 이 테이블을 보고 미적용 것만 실행

migrate reset:
  ⚠️ 로컬 전용 — DB 전체 드롭 + 재생성 + 전체 재적용 + seed

migrate resolve:
  실패한 마이그레이션을 수동 복구 후 Prisma 이력 정리

ENUM ADD VALUE:
  트랜잭션 밖으로 꺼내기 + IF NOT EXISTS

seed:
  upsert로 멱등성 보장 — 여러 번 실행해도 중복 없음
  migrate reset 시 자동 실행

운영 중 스키마를 안전하게 변경해야 한다면 (컬럼 이름 변경, 타입 변경, NOT NULL 추가 등):
  → [[DB_MigrationPattern]] Expand-Contract 패턴 참고
  (migrate deploy 한 번으로 끝내는 게 아니라 배포를 단계별로 분리하는 전략)
```