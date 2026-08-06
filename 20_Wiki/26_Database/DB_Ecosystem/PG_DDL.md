---
aliases:
  - CREATE
  - DDL
tags:
  - SQL
  - PostgreSQL
related:
  - "[[00_DB_HomePage]]"
  - "[[NestJS_PostgreSQL]]"
  - "[[NestJS_Prisma]]"
  - "[[PG_DML]]"
  - "[[PG_Types]]"
  - "[[NestJS_Migration]]"
---
# PG_DDL — PostgreSQL 데이터 정의 언어

>[!info]
>DDL(Data Definition Language) = 테이블·인덱스·제약조건 등 데이터베이스 **구조**를 정의하는 SQL.
> `CREATE`·`ALTER`·`DROP`이 핵심. 
> DML(SELECT·INSERT 등)이 데이터를 다룬다면 DDL은 데이터가 담길 **그릇**을 만드는 것.
>  Prisma에서는 schema.prisma가 DDL을 대신 생성 → [[NestJS_Migration]]

---

# DDL이란 ⭐️⭐️⭐️⭐️

```txt
SQL은 크게 두 가지:

  DDL (Data Definition Language) — 구조 정의
    CREATE TABLE   테이블 생성
    ALTER TABLE    테이블 수정 (컬럼 추가/변경/삭제)
    DROP TABLE     테이블 삭제
    CREATE INDEX   인덱스 생성

  DML (Data Manipulation Language) — 데이터 조작  → [[PG_DML]]
    SELECT   조회
    INSERT   삽입
    UPDATE   수정
    DELETE   삭제

비유:
  DDL = 건물 설계·건축 (구조를 만듦)
  DML = 건물 안에서 물건을 넣고 꺼내고 수정하는 것
```

---

# CREATE TABLE ⭐️⭐️⭐️⭐️

```sql
CREATE TABLE posts (
  id         UUID         PRIMARY KEY DEFAULT gen_random_uuid(),
  title      VARCHAR(200) NOT NULL,
  content    TEXT,                          -- NULL 허용
  view_count INTEGER      NOT NULL DEFAULT 0,
  author_id  UUID         NOT NULL,
  created_at TIMESTAMPTZ  NOT NULL DEFAULT NOW(),
  deleted_at TIMESTAMPTZ                    -- 소프트 삭제용, NULL 허용
);
```

```txt
CREATE TABLE 구조:
  컬럼명 → 타입 → 제약조건 순서

  타입을 먼저 정하고, 그 다음 제약조건을 붙임
  여러 제약조건은 공백으로 나열
  컬럼마다 쉼표로 구분, 마지막 컬럼엔 쉼표 없음
```

---

# 주요 컬럼 타입 ⭐️⭐️⭐️⭐️

```txt
문자열:
  VARCHAR(n)   최대 n글자 문자열 (초과 시 에러)
  TEXT         길이 제한 없는 문자열 (PostgreSQL에서 VARCHAR와 성능 차이 없음)
  CHAR(n)      정확히 n글자 (짧으면 공백 채움 — 거의 안 씀)

숫자:
  INTEGER      4바이트 정수 (-21억 ~ +21억)
  BIGINT       8바이트 정수 (매우 큰 수)
  SMALLINT     2바이트 정수 (-32768 ~ +32767)
  NUMERIC(p,s) 정확한 소수 (p=전체 자리, s=소수 자리) — 금액에 사용
  FLOAT        부동소수 (정확도 낮음 — 금액에 쓰면 안 됨)

불린:
  BOOLEAN      true / false / NULL

날짜·시간:
  DATE         날짜만 (2024-01-15)
  TIME         시간만
  TIMESTAMP    날짜+시간 (타임존 없음)
  TIMESTAMPTZ  날짜+시간+타임존 ← 권장 (UTC로 저장, 조회 시 변환)

식별자:
  UUID         128비트 고유 ID (gen_random_uuid()로 생성)
  SERIAL       자동 증가 정수 (구식 — 요즘은 GENERATED ALWAYS AS IDENTITY)

기타:
  JSONB        JSON 저장 + 인덱스 가능 (JSON보다 빠름)
  TEXT[]       텍스트 배열
```

---

# 제약조건 (Constraints) ⭐️⭐️⭐️⭐️

## PRIMARY KEY

```sql
-- 방법 1 — 컬럼에 직접 (단일 컬럼)
id UUID PRIMARY KEY DEFAULT gen_random_uuid()

-- 방법 2 — 테이블 레벨 (복합 키)
CREATE TABLE room_members (
  room_id UUID NOT NULL,
  user_id UUID NOT NULL,
  PRIMARY KEY (room_id, user_id)   -- 두 컬럼을 합쳐서 PK
);
```

```txt
PRIMARY KEY:
  NOT NULL + UNIQUE를 동시에 보장
  테이블당 하나만 가능
  자동으로 인덱스 생성됨
```

## NOT NULL

```sql
title VARCHAR(200) NOT NULL   -- 반드시 값 있어야 함
body  TEXT                    -- NULL 허용 (기본)
```

## DEFAULT

```sql
view_count INTEGER      NOT NULL DEFAULT 0
status     VARCHAR(20)  NOT NULL DEFAULT 'active'
created_at TIMESTAMPTZ  NOT NULL DEFAULT NOW()
id         UUID         PRIMARY KEY DEFAULT gen_random_uuid()
```

## UNIQUE

```sql
-- 단일 컬럼 UNIQUE
email VARCHAR(200) UNIQUE

-- 복합 UNIQUE (두 컬럼 조합이 유일)
UNIQUE (date, hour)    -- 같은 날짜+시간 조합 중복 불가
```

```txt
⚠️ UNIQUE + NULL 함정:
  NULL은 중복으로 취급되지 않음 → NULL이 여러 개 들어갈 수 있음
  → 센티넬 값(-1) 또는 NULLS NOT DISTINCT (PG 15+) 사용
  → [[PG_Patterns]] NULL UNIQUE 섹션 참고
```

## FOREIGN KEY

```sql
CREATE TABLE comments (
  id        UUID PRIMARY KEY DEFAULT gen_random_uuid(),
  post_id   UUID NOT NULL,
  author_id UUID NOT NULL,

  FOREIGN KEY (post_id)   REFERENCES posts(id)   ON DELETE CASCADE,
  FOREIGN KEY (author_id) REFERENCES users(id)    ON DELETE RESTRICT
);

-- 컬럼에 직접 쓰는 방법
post_id UUID NOT NULL REFERENCES posts(id) ON DELETE CASCADE
```

## ON DELETE 옵션

|옵션|동작|예시|
|---|---|---|
|`CASCADE`|부모 삭제 → 자식도 삭제|게시글 삭제 시 댓글 삭제|
|`RESTRICT`|자식 있으면 부모 삭제 불가|답글 있는 댓글 삭제 불가|
|`SET NULL`|부모 삭제 → FK를 NULL로|작성자 탈퇴 시 게시글 유지|
|`SET DEFAULT`|부모 삭제 → FK를 기본값으로|기본값이 설정된 경우|
|`NO ACTION`|기본값 — RESTRICT와 유사|명시 안 하면 이 동작|

## CHECK

```sql
CREATE TABLE products (
  price    NUMERIC(10,2) NOT NULL CHECK (price >= 0),
  quantity INTEGER       NOT NULL CHECK (quantity >= 0),
  status   VARCHAR(20)   CHECK (status IN ('active', 'inactive', 'deleted'))
);
```

---

# ALTER TABLE — 구조 변경 ⭐️⭐️⭐️⭐️

```sql
-- 컬럼 추가
ALTER TABLE posts ADD COLUMN is_pinned BOOLEAN NOT NULL DEFAULT false;

-- 컬럼 삭제
ALTER TABLE posts DROP COLUMN old_field;

-- 컬럼 타입 변경
ALTER TABLE posts ALTER COLUMN title TYPE TEXT;

-- NOT NULL 추가
ALTER TABLE posts ALTER COLUMN author_id SET NOT NULL;

-- NOT NULL 제거
ALTER TABLE posts ALTER COLUMN description DROP NOT NULL;

-- 기본값 추가
ALTER TABLE posts ALTER COLUMN status SET DEFAULT 'draft';

-- 기본값 제거
ALTER TABLE posts ALTER COLUMN status DROP DEFAULT;

-- 컬럼 이름 변경
ALTER TABLE posts RENAME COLUMN old_name TO new_name;

-- 테이블 이름 변경
ALTER TABLE posts RENAME TO articles;
```

```txt
ALTER TABLE 주의사항:
  운영 중인 테이블에 NOT NULL 컬럼 추가 시:
    기존 행의 값이 NULL → 에러 발생
    → DEFAULT를 함께 지정하거나, 먼저 NULL 허용으로 추가 후 데이터 채우고 NOT NULL 추가

  타입 변경 시:
    기존 데이터가 변환 가능해야 함 (TEXT → INTEGER는 숫자 문자열만 가능)
    운영 중 대용량 테이블 타입 변경은 락(lock) 발생 → 서비스 중단 가능
```

---

# DROP — 삭제 ⭐️⭐️⭐️

```sql
-- 테이블 삭제
DROP TABLE posts;

-- 존재할 때만 삭제 (없으면 에러 없음)
DROP TABLE IF EXISTS posts;

-- 참조 관계 있어도 강제 삭제 (FOREIGN KEY도 함께 삭제)
DROP TABLE posts CASCADE;

-- 컬럼 삭제
ALTER TABLE posts DROP COLUMN description;
ALTER TABLE posts DROP COLUMN IF EXISTS description;

-- 인덱스 삭제
DROP INDEX idx_posts_user_id;
DROP INDEX IF EXISTS idx_posts_user_id;
```

```txt
⚠️ DROP은 되돌릴 수 없음
  트랜잭션 안에서 DROP 후 확인하고 COMMIT하는 것이 안전
  운영 DB에서는 백업 후 실행
```

---

# CREATE INDEX — 인덱스 ⭐️⭐️⭐️⭐️

```sql
-- 기본 인덱스
CREATE INDEX idx_posts_user_id ON posts (user_id);

-- 내림차순
CREATE INDEX idx_posts_created_at ON posts (created_at DESC);

-- UNIQUE 인덱스 (UNIQUE 제약과 동일 효과)
CREATE UNIQUE INDEX idx_users_email ON users (email);

-- 복합 인덱스 (순서 중요!)
CREATE INDEX idx_posts_user_created ON posts (user_id, created_at DESC);
-- → WHERE user_id = ? ORDER BY created_at DESC 쿼리에 최적

-- Partial Index (조건부 인덱스)
CREATE INDEX idx_active_posts ON posts (created_at DESC)
WHERE deleted_at IS NULL;
-- → 삭제 안 된 게시글만 인덱스 → 더 작고 빠름
```

```txt
복합 인덱스 순서 규칙:
  (user_id, created_at) 인덱스는:
    WHERE user_id = ?                    ← 사용 가능 (앞 컬럼만)
    WHERE user_id = ? AND created_at > ? ← 사용 가능 (앞+뒤)
    WHERE created_at > ?                 ← 비효율 (앞 컬럼 없음)

  → 등호(=) 조건 컬럼을 앞에, 범위 조건 컬럼을 뒤에

Prisma schema에서:
  @@index([userId])
  @@index([userId, createdAt(sort: Desc)])
```

---

# Prisma와 DDL ⭐️⭐️⭐️

```prisma
// schema.prisma가 DDL을 대신 작성
model Post {
  id        String   @id @default(cuid())          // PRIMARY KEY + DEFAULT
  title     String   @db.VarChar(200)              // VARCHAR(200) NOT NULL
  content   String?                                // TEXT NULL
  viewCount Int      @default(0)                   // INTEGER DEFAULT 0
  authorId  String                                 // UUID NOT NULL
  createdAt DateTime @default(now())               // TIMESTAMPTZ DEFAULT NOW()
  author    User     @relation(fields: [authorId], references: [id], onDelete: Cascade)

  @@index([authorId])                              // CREATE INDEX
  @@unique([date, hour])                           // UNIQUE(date, hour)
}
```

```txt
prisma migrate dev 를 실행하면:
  schema.prisma의 변경사항을 분석해서
  필요한 DDL(ALTER TABLE, CREATE INDEX 등)을 자동 생성·실행
  → 직접 DDL을 쓸 일이 많지 않음

직접 DDL을 쓰는 경우:
  Prisma가 지원 안 하는 PostgreSQL 기능 (Partial Index, NULLS NOT DISTINCT 등)
  prisma/migrations/*.sql 파일에 직접 추가
  데이터 마이그레이션 (기존 데이터를 변환하면서 구조 변경)
```