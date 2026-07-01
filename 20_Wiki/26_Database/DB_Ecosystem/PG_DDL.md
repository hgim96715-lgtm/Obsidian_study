---
aliases: [CREATE, DDL]
tags: [SQL, PostgreSQL]
related:
  - "[[00_DB_HomePage]]"
  - "[[NestJS_PostgreSQL]]"
  - "[[NestJS_Prisma]]"
  - "[[PG_DML]]"
  - "[[PG_Types]]"
---
# PG_DDL — 테이블 구조 관리

> [!info] 
> DDL(Data Definition Language) — 테이블을 만들고, 고치고, 지운다. 
> Prisma를 쓴다면 schema.prisma → migrate dev가 DDL을 대신 생성해주지만, DataGrip에서 직접 쿼리 날릴 때나 migrate 파일을 직접 수정할 때 필요

---

# CREATE TABLE ⭐️⭐️⭐️

## 기본 구조

```sql
CREATE TABLE item (
    id           BIGSERIAL PRIMARY KEY,
    title        VARCHAR(200)  NOT NULL,
    description  TEXT,
    price        NUMERIC(10,2) NOT NULL DEFAULT 0,
    is_active    BOOLEAN       NOT NULL DEFAULT TRUE,
    created_at   TIMESTAMPTZ(3) NOT NULL DEFAULT NOW(),
    updated_at   TIMESTAMPTZ(3) NOT NULL DEFAULT NOW()
);
```

```txt
BIGSERIAL:
  SERIAL   → INTEGER(4바이트) 자동 증가 — 약 21억까지
  BIGSERIAL → BIGINT(8바이트) 자동 증가 — 약 922경까지
  서비스 규모 모를 때는 처음부터 BIGSERIAL 선택이 안전

NUMERIC(10, 2):
  정밀 소수 — 금액/비율에 사용
  NUMERIC(전체자릿수, 소수점이하)
  FLOAT/REAL 은 부동소수점 오차 있어서 금액에 사용 금지

TIMESTAMPTZ(3):
  timezone-aware — UTC로 저장, 어떤 환경에서도 같은 instant
  안 붙이면 기본이 TIMESTAMP(timezone-naive) → 쓰지 말 것
  ([[PG_Types]] "timestamp vs timestamptz" 참고)
```

## FK 포함

```sql
CREATE TABLE post (
    id          BIGSERIAL PRIMARY KEY,
    user_id     BIGINT NOT NULL REFERENCES "user"(id) ON DELETE CASCADE,
    title       VARCHAR(300) NOT NULL,
    created_at  TIMESTAMPTZ(3) NOT NULL DEFAULT NOW()
);
```

```txt
ON DELETE 옵션:
  CASCADE   → 부모 삭제 시 자식도 삭제
  SET NULL  → 부모 삭제 시 FK를 NULL로 (컬럼이 nullable이어야 함)
  RESTRICT  → 자식이 있으면 부모 삭제 불가 (기본값)
  NO ACTION → RESTRICT와 거의 같음 (트랜잭션 끝에 체크)
```

## IF NOT EXISTS — 에러 방지

```sql
CREATE TABLE IF NOT EXISTS item (
    id BIGSERIAL PRIMARY KEY,
    ...
);
-- 이미 있으면 조용히 무시, 없으면 생성
```

---

# 제약조건 ⭐️⭐️⭐️

## 종류 한눈에

|제약조건|역할|예시|
|---|---|---|
|`PRIMARY KEY`|기본키 (UNIQUE + NOT NULL 자동)|`id BIGSERIAL PRIMARY KEY`|
|`NOT NULL`|NULL 불가|`title VARCHAR(200) NOT NULL`|
|`UNIQUE`|중복 불가|`email TEXT UNIQUE`|
|`CHECK`|값 조건|`CHECK (price >= 0)`|
|`DEFAULT`|기본값|`DEFAULT NOW()`|
|`REFERENCES`|FK (외래키)|`REFERENCES "user"(id)`|

## CHECK — 조건 제약

```sql
CREATE TABLE product (
    id       BIGSERIAL PRIMARY KEY,
    price    NUMERIC(10,2) CHECK (price >= 0),
    discount NUMERIC(5,2)  CHECK (discount BETWEEN 0 AND 100),
    -- 여러 컬럼에 걸친 CHECK
    CONSTRAINT price_check CHECK (sale_price <= price)
);
```

## 복합 UNIQUE — 쌍으로 유니크

```sql
CREATE TABLE user_follow (
    follower_id  BIGINT NOT NULL REFERENCES "user"(id),
    following_id BIGINT NOT NULL REFERENCES "user"(id),
    UNIQUE (follower_id, following_id)   -- 같은 쌍 중복 방지
);
```

```txt
Prisma에서의 복합 UNIQUE → @@unique([follower_id, following_id])
조회 시 → findUnique({ where: { follower_id_following_id: { follower_id, following_id } } })
([[NestJS_Prisma]] "@@unique" 섹션 참고)
```

## 인덱스 — CREATE INDEX

```sql
-- 단일 인덱스
CREATE INDEX idx_post_user_id ON post(user_id);

-- 복합 인덱스 (순서 중요 — 앞 컬럼으로 단독 검색도 인덱스 탐)
CREATE INDEX idx_post_user_created ON post(user_id, created_at DESC);

-- UNIQUE 인덱스 (= UNIQUE 제약과 동일 효과)
CREATE UNIQUE INDEX idx_user_email ON "user"(email);

-- 부분 인덱스 (조건 만족하는 행만) — PostgreSQL 특화
CREATE INDEX idx_active_item ON item(created_at) WHERE is_active = TRUE;
```

```txt
부분 인덱스(Partial Index):
  WHERE 조건을 붙여서 일부 행에만 인덱스 생성
  → 전체 행 중 is_active=TRUE인 것만 자주 검색한다면
    전체 인덱스보다 훨씬 작고 빠른 인덱스가 만들어짐
  → Prisma에서 직접 지원 안 됨 → migrate dev 후 SQL 직접 추가
```

---

# ALTER TABLE — 테이블 수정 ⭐️⭐️

```sql
-- 컬럼 추가
ALTER TABLE item ADD COLUMN thumbnail_url TEXT;

-- 컬럼 삭제
ALTER TABLE item DROP COLUMN thumbnail_url;

-- 타입 변경 (데이터 변환 가능한 경우)
ALTER TABLE item ALTER COLUMN title TYPE TEXT;

-- 타입 변경 + 명시적 USING (변환 방법 지정)
ALTER TABLE item ALTER COLUMN price TYPE NUMERIC(12,2)
    USING price::NUMERIC(12,2);

-- 컬럼 이름 변경
ALTER TABLE item RENAME COLUMN title TO name;

-- NOT NULL 추가 / 제거
ALTER TABLE item ALTER COLUMN price SET NOT NULL;
ALTER TABLE item ALTER COLUMN price DROP NOT NULL;

-- DEFAULT 설정 / 제거
ALTER TABLE item ALTER COLUMN is_active SET DEFAULT TRUE;
ALTER TABLE item ALTER COLUMN is_active DROP DEFAULT;

-- 테이블 이름 변경
ALTER TABLE item RENAME TO product;

-- FK 추가
ALTER TABLE post
    ADD CONSTRAINT fk_post_user
    FOREIGN KEY (user_id) REFERENCES "user"(id) ON DELETE CASCADE;

-- 제약조건 삭제
ALTER TABLE post DROP CONSTRAINT fk_post_user;
```

```txt
timestamp → timestamptz 마이그레이션 (Prisma schema 수정 후 migrate dev가 생성하는 SQL):
  ALTER TABLE "User"
    ALTER COLUMN "createdAt" TYPE TIMESTAMPTZ(3)
    USING "createdAt" AT TIME ZONE 'UTC';
  
  USING ... AT TIME ZONE 'UTC': 기존 데이터가 UTC로 들어갔다고 가정
  ([[NestJS_PostgreSQL]] 참고)
```

---

# DROP TABLE ⭐️

```sql
-- 기본 삭제
DROP TABLE item;

-- 없으면 무시 (에러 방지) — 스크립트에서 자주 씀
DROP TABLE IF EXISTS item;

-- FK로 참조되는 테이블도 함께 삭제
DROP TABLE item CASCADE;
-- ⚠️ CASCADE: 이 테이블을 참조하는 다른 테이블의 FK 제약까지 같이 삭제됨
--             데이터가 삭제되는 건 아님, 제약조건이 삭제됨
```

---

# TRUNCATE — 전체 초기화 ⭐️⭐️⭐️

## DELETE vs TRUNCATE

|
|`DELETE`|`TRUNCATE`|
|---|---|---|
|속도|느림 (행마다 로그 기록)|빠름 (통째로 비움)|
|롤백|가능|불가 (PostgreSQL에서도)|
|WHERE 조건|가능|불가 — 항상 전체|
|SERIAL/ID 초기화|❌ 이어서 증가|✅ RESTART IDENTITY로 가능|
|용도|일부 삭제|전체 초기화|

## PostgreSQL TRUNCATE 옵션

```sql
-- 기본 (데이터 삭제, id는 이어서 증가)
TRUNCATE TABLE item;

-- id를 1부터 다시 시작
TRUNCATE TABLE item RESTART IDENTITY;

-- FK로 연결된 자식 테이블도 같이 비움
TRUNCATE TABLE item CASCADE;

-- 둘 다 ⭐️ — 개발 중 가장 많이 씀
TRUNCATE TABLE item RESTART IDENTITY CASCADE;
```

```txt
언제 쓰나:
  개발 중 테스트 데이터가 쌓여 id가 1, 5, 12...처럼 뒤죽박죽될 때
  → 데이터 전부 지우고 id를 1부터 다시 시작하고 싶을 때

여러 테이블 한꺼번에 초기화:
  TRUNCATE TABLE post, comment, tag RESTART IDENTITY CASCADE;
  (순서는 FK 관계 무관 — CASCADE가 처리해줌)

⚠️ 롤백 불가 + CASCADE는 연관 데이터 전부 삭제
  → 로컬 개발 / 테스트 환경에서만
  → 프로덕션 절대 금지
```

---

# Prisma와의 관계 ⭐️⭐️

```txt
Prisma를 쓰면 DDL을 직접 쓸 일이 많이 줄어듦:
  schema.prisma 수정 → prisma migrate dev → DDL SQL 자동 생성 + 실행

그래도 DDL 직접 필요한 경우:
  ① 부분 인덱스(Partial Index) — Prisma가 직접 지원 안 함
     → migrate dev로 빈 마이그레이션 파일 생성 후 SQL 직접 작성

  ② 기존 컬럼 타입 변경 (timestamp → timestamptz 같은 경우)
     → migrate dev가 생성하는 SQL을 검토/수정

  ③ DataGrip에서 직접 구조 확인하거나 임시 수정이 필요할 때

생성된 마이그레이션 SQL은 prisma/migrations/ 폴더에 저장됨
→ Git에 커밋해서 팀원과 공유, 운영 배포 시 prisma migrate deploy로 적용
([[NestJS_Prisma]] "migrate dev / migrate deploy" 섹션 참고)
```

---

# 한눈에

```txt
CREATE TABLE:
  BIGSERIAL → 큰 서비스 고려하면 SERIAL 대신 BIGSERIAL
  TIMESTAMPTZ(3) → timezone 포함 필수 (TIMESTAMP 쓰지 말 것)
  NUMERIC(10,2) → 금액/비율 (FLOAT 사용 금지)
  IF NOT EXISTS → 스크립트에서 에러 방지

제약조건:
  PRIMARY KEY / NOT NULL / UNIQUE / CHECK / DEFAULT / REFERENCES
  복합 UNIQUE → UNIQUE (col1, col2)
  부분 인덱스 → CREATE INDEX ... WHERE 조건 (Prisma 직접 지원 안 됨)

ALTER TABLE:
  ADD COLUMN / DROP COLUMN / ALTER COLUMN TYPE / RENAME COLUMN
  SET NOT NULL / DROP NOT NULL / SET DEFAULT / DROP DEFAULT
  ADD CONSTRAINT FK / DROP CONSTRAINT

TRUNCATE:
  RESTART IDENTITY → id 1부터 재시작
  CASCADE → FK 연결된 자식 테이블도 같이 비움
  둘 다: TRUNCATE TABLE xxx RESTART IDENTITY CASCADE
  ⚠️ 롤백 불가 — 개발/테스트 환경에서만

Prisma 연결:
  schema.prisma → migrate dev → SQL 자동 생성
  부분 인덱스 등 Prisma 미지원 → 마이그레이션 SQL 직접 작성
```