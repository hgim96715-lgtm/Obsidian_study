---
aliases:
  - Expand
  - Migrate
  - Contract
tags:
  - SQL
  - PostgreSQL
related:
  - "[[00_DB_HomePage]]"
  - "[[Deploy_CloudMVP]]"
  - "[[NestJS_Migration]]"
  - "[[PG_DDL]]"
---
# DB_MigrationPattern — 무중단 마이그레이션 패턴

> [!info] 
> 운영 중인 DB 스키마를 변경할 때 서비스를 내리지 않고 안전하게 진행하는 전략 모음
>  핵심은 "한 번에 다 바꾸지 않는다" — 애플리케이션 배포와 스키마 변경을 작은 단계로 분리한다.

---

# 왜 단순 migrate deploy로 부족한가 ⭐️⭐️⭐️

```txt
개발 환경: migrate dev → migrate deploy → 끝 (단순)

운영 환경에서 문제가 되는 상황:
  예) users 테이블의 name 컬럼을 username으로 이름 변경

  ① 순진한 방법: ALTER TABLE users RENAME COLUMN name TO username
     → 배포 순간 구 버전 앱이 users.name을 읽으려 하면 에러 → 서비스 장애

  ② 점검 시간 잡고 배포:
     서비스를 잠시 내리고 변경 → 소규모 서비스에서는 가능
     하지만 24시간 운영해야 하거나 SLA가 있으면 불가능

→ 무중단 마이그레이션 패턴이 필요한 이유
```

```txt
핵심 원칙:
  DB 변경과 앱 코드 변경을 같은 배포에 묶지 않는다
  각 단계는 "배포 롤백이 가능한" 상태여야 한다
  → 배포 단계마다 구버전 앱과 신버전 앱이 동시에 동작 가능해야 함
```

---

# Expand-Contract 패턴 ⭐️⭐️⭐️⭐️

```txt
Expand-Contract = "먼저 넓히고(Expand), 나중에 좁힌다(Contract)"

3단계:
  ① Expand   새 구조를 기존 구조 옆에 추가 (기존 건드리지 않음)
  ② Migrate  데이터를 새 구조로 옮기고, 앱이 새 구조를 사용하도록 전환
  ③ Contract 기존 구조 제거
```

## 예시 — name 컬럼을 username으로 이름 변경

### ① Expand 단계 — 새 컬럼 추가

```sql
-- migration: add_username_column
ALTER TABLE users ADD COLUMN username VARCHAR(100);
-- nullable로 추가 — 기존 데이터에 영향 없음
-- 기존 name 컬럼은 그대로 유지
```

```typescript
// Prisma schema — 두 컬럼 모두 존재
model User {
  id       Int     @id
  name     String  // 기존 (아직 삭제 안 함)
  username String? // 새로 추가 (nullable)
}
```

```typescript
// 앱 코드 — 쓸 때 두 컬럼 모두 채움 (읽기는 아직 name에서)
await prisma.user.create({
  data: {
    name:     '홍길동',
    username: '홍길동',  // 새 컬럼에도 같이 씀
  },
});
```

```txt
이 상태의 특징:
  기존 앱(name 사용)과 새 앱(username 사용)이 동시에 동작 가능
  롤백 가능 — username 컬럼을 DROP해도 name은 멀쩡함
```

### ② Migrate 단계 — 데이터 이전 + 앱 전환

```sql
-- 기존 데이터 백필(Backfill)
-- 이미 있는 rows에 username 채우기
UPDATE users SET username = name WHERE username IS NULL;
```

```typescript
// Prisma schema — username을 non-null로 변경
model User {
  id       Int    @id
  name     String // 아직 유지
  username String // nullable 제거
}
```

```typescript
// 앱 코드 — 읽기를 username으로 전환, 쓰기는 여전히 둘 다
const user = await prisma.user.findUnique({ where: { id } });
console.log(user.username); // ← 이제 username으로 읽음
```

```txt
백필(Backfill) 주의:
  대용량 테이블에서 UPDATE를 한 번에 실행하면 DB 락 발생
  → 배치로 나눠서 실행

  -- 배치 백필 예시 (1000건씩)
  UPDATE users SET username = name
  WHERE username IS NULL
  LIMIT 1000;
  -- 위를 반복
```

### ③ Contract 단계 — 기존 컬럼 제거

```typescript
// 앱 코드에서 name 컬럼 완전히 제거 확인 후
model User {
  id       Int    @id
  username String // name 컬럼 제거
}
```

```sql
-- migration: remove_name_column
ALTER TABLE users DROP COLUMN name;
```

```txt
이 단계 전 확인:
  모든 앱 인스턴스가 username을 사용하고 있는가
  어디서도 name 컬럼을 참조하지 않는가 (로그, 쿼리 확인)
  롤백 시 name 복구가 가능한 상태인가 (백업 여부)
```

---

# Expand-Contract가 필요한 대표 상황 ⭐️⭐️⭐️

|변경 유형|위험도|패턴 적용 필요|
|---|---|---|
|새 테이블 추가|낮음|불필요 (기존에 영향 없음)|
|새 컬럼 추가 (nullable)|낮음|불필요|
|새 컬럼 추가 (NOT NULL, default 없음)|높음|필요|
|컬럼 이름 변경|높음|필요|
|컬럼 타입 변경|높음|필요|
|컬럼 삭제|높음|필요|
|테이블 분리/합치기|매우 높음|필요|
|인덱스 추가 (운영 중)|중간|`CREATE INDEX CONCURRENTLY` 사용|

---

# PostgreSQL 전용 — 무중단 DDL ⭐️⭐️⭐️

## CREATE INDEX CONCURRENTLY

```sql
-- ❌ 일반 인덱스 생성 — 테이블 전체 잠금 → 쓰기 차단
CREATE INDEX idx_users_email ON users(email);

-- ✅ 무중단 인덱스 생성 — 잠금 없이 백그라운드 생성 (시간은 더 걸림)
CREATE INDEX CONCURRENTLY idx_users_email ON users(email);
```

```txt
CONCURRENTLY 주의사항:
  트랜잭션 안에서 실행 불가 (단독 실행만 가능)
  실패하면 INVALID 상태 인덱스가 남음 → DROP 후 재시도
  Prisma migrate에서는 자동으로 CONCURRENTLY를 안 씀
  → --create-only 플래그로 migration.sql을 직접 편집해서 추가해야 함
  (→ [[NestJS_Migration]] "커스텀 SQL 추가 패턴" 참고)
```

## NOT NULL 컬럼 추가 — 두 단계로

```sql
-- ❌ 한 번에 NOT NULL 추가 — 기존 rows가 있으면 에러 또는 전체 테이블 갱신 잠금
ALTER TABLE users ADD COLUMN score INT NOT NULL DEFAULT 0;

-- ✅ 두 단계로 분리
-- 1단계: nullable로 추가
ALTER TABLE users ADD COLUMN score INT;

-- 2단계: 백필 후 NOT NULL 추가 (PostgreSQL 11+는 DEFAULT 있으면 즉시 가능)
UPDATE users SET score = 0 WHERE score IS NULL;
ALTER TABLE users ALTER COLUMN score SET NOT NULL;
```

---

# Prisma에서 Expand-Contract 적용 ⭐️⭐️

```bash
# 1. Expand 마이그레이션 파일 생성 (DB 적용 전 편집을 위해 --create-only)
npx prisma migrate dev --name add_username_column --create-only

# 2. 생성된 migration.sql 편집 후 적용
npx prisma migrate dev

# 3. 앱 코드 수정 → 배포 → 데이터 백필

# 4. Contract 마이그레이션
npx prisma migrate dev --name remove_name_column --create-only
# migration.sql 확인 후 적용
npx prisma migrate dev
```

```txt
각 마이그레이션 사이에 앱 배포가 들어간다는 점이 핵심
migrate dev 한 번으로 끝내는 게 아니라
  마이그레이션 → 배포 → 확인 → 마이그레이션 → 배포 ... 의 반복

운영 환경 적용:
  npx prisma migrate deploy (운영 DB)
  → [[NestJS_Migration]] "배포 시 migrate deploy" 참고
```

---

# 한눈에

```txt
Expand-Contract 3단계:
  ① Expand   기존 구조 옆에 새 구조 추가 (nullable, 기존 건드리지 않음)
              → 앱은 양쪽에 모두 쓰기 시작
  ② Migrate  백필(backfill)로 데이터 이전, 앱 읽기를 새 구조로 전환
  ③ Contract 기존 구조 제거 (앱 코드에서 완전히 사라진 것 확인 후)

각 단계 사이에 앱 배포 + 확인이 들어감
→ 롤백 가능 상태를 항상 유지

PostgreSQL 무중단 DDL:
  인덱스   CREATE INDEX CONCURRENTLY (트랜잭션 밖에서)
  NOT NULL  nullable 추가 → 백필 → NOT NULL 설정 순서

Prisma와 함께:
  --create-only로 SQL 파일을 먼저 생성해 검토 후 적용
  커스텀 SQL(CONCURRENTLY 등)은 migration.sql 직접 편집
  → [[NestJS_Migration]] 참고

적용이 필요한 경우:
  컬럼 이름 변경 / 타입 변경 / 컬럼 삭제 / 테이블 구조 변경
  NOT NULL 컬럼 추가 (기존 데이터 있을 때)
  대용량 테이블 인덱스 추가
```