---
aliases:
  - DDL
  - CREATE TABLE
  - ALTER TABLE
  - DROP TABLE
  - TRUNCATE
  - RESTART IDENTITY CASCADE
tags:
  - SQL
related:
  - "[[00_SQL_HomePage]]"
  - "[[SQL_DML]]"
  - "[[SQL_Data_Types]]"
---
# SQL_DDL — 테이블 구조 관리

## 한 줄 요약

```txt
DDL = Data Definition Language
테이블 생성 / 수정 / 삭제
CREATE / ALTER / DROP / TRUNCATE
```

---

---

# ① CREATE TABLE — 테이블 생성 ⭐️

## PostgreSQL

```sql
CREATE TABLE movie (
    id          SERIAL PRIMARY KEY,          -- 자동 증가 PK
    title       VARCHAR(100) NOT NULL,
    genre       VARCHAR(50),
    detail      TEXT,
    created_at  TIMESTAMP DEFAULT NOW(),
    is_active   BOOLEAN DEFAULT TRUE
);

-- FK 포함
CREATE TABLE movie_detail (
    id          SERIAL PRIMARY KEY,
    movie_id    INTEGER REFERENCES movie(id) ON DELETE CASCADE,
    description TEXT
);
```

## MySQL

```sql
CREATE TABLE movie (
    id          INT AUTO_INCREMENT PRIMARY KEY,  -- 자동 증가 PK
    title       VARCHAR(100) NOT NULL,
    genre       VARCHAR(50),
    detail      TEXT,
    created_at  DATETIME DEFAULT CURRENT_TIMESTAMP,
    is_active   TINYINT(1) DEFAULT 1
);

-- FK 포함
CREATE TABLE movie_detail (
    id          INT AUTO_INCREMENT PRIMARY KEY,
    movie_id    INT,
    description TEXT,
    FOREIGN KEY (movie_id) REFERENCES movie(id) ON DELETE CASCADE
);
```

```txt
PostgreSQL vs MySQL:
  자동 증가: SERIAL          vs AUTO_INCREMENT
  현재 시간: NOW()           vs CURRENT_TIMESTAMP
  불리언:    BOOLEAN         vs TINYINT(1)
  FK 선언:   인라인 REFERENCES vs 별도 FOREIGN KEY 절
```

---

---

# ② 주요 제약조건

```sql
CREATE TABLE users (
    id          SERIAL PRIMARY KEY,           -- 기본키
    email       VARCHAR(255) UNIQUE NOT NULL, -- 유일 + 필수
    name        VARCHAR(100) NOT NULL,        -- 필수
    age         INTEGER CHECK (age >= 0),     -- 조건 검사
    role        VARCHAR(20) DEFAULT 'user',   -- 기본값
    created_at  TIMESTAMP DEFAULT NOW()
);
```

|제약조건|설명|
|---|---|
|`PRIMARY KEY`|기본키 (유일 + NOT NULL 자동)|
|`NOT NULL`|NULL 불가|
|`UNIQUE`|중복 불가|
|`DEFAULT 값`|기본값|
|`CHECK (조건)`|값 범위 제한|
|`REFERENCES 테이블(컬럼)`|외래키 (FK)|

---

---

# ③ ALTER TABLE — 테이블 수정

```sql
-- 컬럼 추가
ALTER TABLE movie ADD COLUMN director VARCHAR(100);

-- 컬럼 삭제
ALTER TABLE movie DROP COLUMN director;

-- 컬럼 타입 변경
-- PostgreSQL
ALTER TABLE movie ALTER COLUMN title TYPE TEXT;
-- MySQL
ALTER TABLE movie MODIFY COLUMN title TEXT;

-- 컬럼 이름 변경
-- PostgreSQL / MySQL 8.0+
ALTER TABLE movie RENAME COLUMN title TO movie_title;

-- NOT NULL 추가
-- PostgreSQL
ALTER TABLE movie ALTER COLUMN title SET NOT NULL;
-- MySQL
ALTER TABLE movie MODIFY COLUMN title VARCHAR(100) NOT NULL;

-- 기본값 설정
-- PostgreSQL
ALTER TABLE movie ALTER COLUMN genre SET DEFAULT 'drama';
-- MySQL
ALTER TABLE movie ALTER COLUMN genre SET DEFAULT 'drama';

-- 테이블 이름 변경
ALTER TABLE movie RENAME TO films;
```

---

---

# ④ DROP TABLE — 테이블 삭제

```sql
-- 기본 삭제
DROP TABLE movie;

-- 없으면 무시 (에러 방지)
DROP TABLE IF EXISTS movie;

-- FK 로 연결된 테이블도 같이 삭제
-- PostgreSQL
DROP TABLE movie CASCADE;

-- MySQL (FK 제약 해제 후 삭제)
SET FOREIGN_KEY_CHECKS = 0;
DROP TABLE movie;
SET FOREIGN_KEY_CHECKS = 1;
```

---

---

# ⑤ TRUNCATE — 테이블 데이터 초기화 ⭐️

## DELETE vs TRUNCATE 차이

| |`DELETE`|`TRUNCATE`|
|---|---|---|
|속도|느림 (행마다 로그)|빠름|
|롤백|가능|불가|
|WHERE 조건|가능|불가|
|ID 초기화|❌ 유지|✅ 가능|
|용도|일부 삭제|전체 초기화|

## PostgreSQL — RESTART IDENTITY CASCADE ⭐️

```sql
-- 기본 (데이터만 삭제, id 이어서 증가)
TRUNCATE TABLE movie;

-- id 를 1 부터 다시 시작
TRUNCATE TABLE movie RESTART IDENTITY;

-- FK 연결된 자식 테이블도 같이 비움
TRUNCATE TABLE movie CASCADE;

-- 둘 다 (가장 많이 씀) ⭐️
TRUNCATE TABLE movie RESTART IDENTITY CASCADE;
```

```txt
언제 쓰나:
  개발 중 테스트 데이터 쌓여서 id 가 1,5,12... 뒤죽박죽
  → 데이터 다 지우고 id 를 1부터 다시 시작하고 싶을 때

  FK 로 연결된 테이블이 있으면 그냥 TRUNCATE 하면 에러
  → CASCADE 추가 → 연관 테이블도 같이 비워줌

실전 패턴 (여러 테이블 초기화):
  TRUNCATE TABLE movie        RESTART IDENTITY CASCADE;
  TRUNCATE TABLE director     RESTART IDENTITY CASCADE;
  TRUNCATE TABLE movie_detail RESTART IDENTITY CASCADE;
```

```txt
⚠️ 주의:
  롤백 불가 → 복구 안 됨
  CASCADE → 연관 데이터 전부 삭제
  → 로컬 개발 / 테스트 환경에서만 사용
  → 프로덕션(실서버) 절대 금지
```

## MySQL

```sql
-- MySQL 은 RESTART IDENTITY 없음
TRUNCATE TABLE movie;
-- → AUTO_INCREMENT 자동으로 1로 초기화

-- FK 있으면 제약 해제 후
SET FOREIGN_KEY_CHECKS = 0;
TRUNCATE TABLE movie;
SET FOREIGN_KEY_CHECKS = 1;
```

---

---

# 명령어 한눈에

|명령어|역할|
|---|---|
|`CREATE TABLE 테이블 (...)`|테이블 생성|
|`ALTER TABLE 테이블 ADD COLUMN 컬럼`|컬럼 추가|
|`ALTER TABLE 테이블 DROP COLUMN 컬럼`|컬럼 삭제|
|`ALTER TABLE 테이블 RENAME TO 새이름`|테이블 이름 변경|
|`DROP TABLE IF EXISTS 테이블`|테이블 삭제|
|`TRUNCATE TABLE 테이블 RESTART IDENTITY CASCADE`|전체 초기화 + id 리셋|