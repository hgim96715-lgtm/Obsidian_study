---
aliases:
  - DML
  - INSERT
  - UPDATE
  - DELETE
  - UPSERT
  - ON CONFLICT
  - RETURNING
  - 따옴표 규칙
tags:
  - SQL
related:
  - "[[00_SQL_HomePage]]"
  - "[[SQL_DDL]]"
  - "[[SQL_SELECT]]"
---

# SQL_DML — 데이터 조작

## 한 줄 요약

```
INSERT    → 행 추가
UPDATE    → 행 수정
DELETE    → 행 삭제
UPSERT    → 있으면 UPDATE / 없으면 INSERT (ON CONFLICT)
RETURNING → 변경된 행 바로 반환 (PostgreSQL)
```

---
---
# 따옴표 규칙 — 가장 자주 하는 실수 ⭐️

```
작은따옴표 '  → 문자열 값 (string literal)
큰따옴표   "  → 식별자 (테이블명, 컬럼명)

혼동하면 에러 또는 의도치 않은 동작 발생
```

```sql
-- ❌ 잘못된 방법 — 큰따옴표로 값 지정
UPDATE "User"
SET nickname = "공이"
WHERE email = "admin@example.com";
-- "공이" → 컬럼명으로 해석 → 에러

-- ✅ 올바른 방법 — 문자열 값은 작은따옴표
UPDATE "User"
SET nickname = '공이'
WHERE email = 'admin@example.com';
```

```
언제 큰따옴표 " 를 쓰나:
  테이블명에 대문자 / 예약어 / 공백이 있을 때만
  → "User" / "Order" / "my table"
  → Prisma 가 생성한 테이블은 대소문자 그대로 → " 로 감싸야 함

  ⚠️ PostgreSQL 따옴표 없는 식별자 → 자동으로 소문자 변환
     User   → user  (다른 테이블로 인식)
     "User" → User  (대소문자 유지)

언제 작은따옴표 ' 를 쓰나:
  문자열 값은 항상 작은따옴표
  '공이' / 'drama' / 'admin@example.com' / '2024-01-01'
```

---

---

#  INSERT — 행 추가 ⭐


```
INSERT INTO 테이블명 (컬럼1, 컬럼2, ...)
VALUES (값1, 값2, ...);
```

```sql
-- 컬럼 지정 (권장)
INSERT INTO movies (title, genre, rating)
VALUES ('기생충', 'drama', 8.5);

-- 여러 행 한번에
INSERT INTO movies (title, genre, rating)
VALUES
  ('기생충', 'drama', 8.5),
  ('아바타', 'sf',    7.8),
  ('극한직업', 'comedy', 9.1);

-- 전체 컬럼 (컬럼 순서 일치 필수 / 비권장)
INSERT INTO movies VALUES (1, '기생충', 'drama', 8.5);
```

```
컬럼 지정 방식 권장:
  테이블 컬럼 순서 바뀌어도 안전
  지정 안 한 컬럼 → NULL 또는 DEFAULT
```

## INSERT ... SELECT

```sql
-- 다른 테이블에서 데이터 가져와서 삽입
INSERT INTO movies_backup (title, genre)
SELECT title, genre
FROM movies
WHERE genre = 'drama';
```

---

---

#  UPDATE — 행 수정 ⭐

## 기본 문법

```
UPDATE 테이블명
SET 컬럼1 = 값1, 컬럼2 = 값2
WHERE 조건;
```

```sql
-- 기본
UPDATE movies
SET rating = 9.0
WHERE id = 1;

-- 여러 컬럼 동시 수정
UPDATE movies
SET rating = 9.0,
    genre  = 'thriller'
WHERE id = 1;

-- 계산식
UPDATE products
SET price = price * 1.1   -- 10% 인상
WHERE category = 'premium';
```

```
⚠️ WHERE 없으면 전체 수정
  UPDATE movies SET rating = 0;   -- 모든 행 rating = 0 ← 위험!
  항상 WHERE 조건 확인 후 실행
```

## UPDATE ... FROM (PostgreSQL)

```sql
-- 다른 테이블 값으로 업데이트
UPDATE movies m
SET director_name = d.name
FROM directors d
WHERE m.director_id = d.id;
```

---

---

# DELETE — 행 삭제 ⭐️

## 기본 문법

```
DELETE FROM 테이블명
WHERE 조건;
```


```sql
DELETE FROM movies WHERE id = 1;

DELETE FROM movies
WHERE genre = 'drama' AND rating < 5.0;
```

```
⚠️ WHERE 없으면 전체 삭제
  DELETE FROM movies;   -- 모든 행 삭제 ← 매우 위험!
  TRUNCATE 와 달리 롤백 가능

DELETE vs TRUNCATE:
  DELETE    조건 가능 / 롤백 가능 / 느림 (행 하나씩)
  TRUNCATE  전체 삭제만 / 빠름 / 주의 필요
```

## 실수 방지 패턴

```sql
-- 삭제 전 SELECT 로 확인
SELECT * FROM movies WHERE genre = 'drama' AND rating < 5.0;
-- 확인 후 DELETE 실행
DELETE FROM movies WHERE genre = 'drama' AND rating < 5.0;
```

---

---
# RETURNING — 변경된 행 바로 반환 ⭐️ (PostgreSQL)

```
INSERT / UPDATE / DELETE 뒤에 붙여서
변경된 행 데이터를 바로 반환
별도 SELECT 없이 한 번에 처리
```

## UPDATE + RETURNING

```sql
UPDATE "user"
SET role = 2
WHERE email = 'test@user.com'
RETURNING id, email, role;
-- 수정된 행의 id / email / role 바로 반환
```

## INSERT + RETURNING

```sql
-- INSERT 후 생성된 id 즉시 확인
INSERT INTO movies (title, genre)
VALUES ('기생충', 'drama')
RETURNING id, title;
-- { id: 5, title: '기생충' }
```

## DELETE + RETURNING

```sql
DELETE FROM movies
WHERE id = 1
RETURNING *;
-- 삭제된 행 모든 컬럼 반환
```

```
RETURNING 사용 이유:
  변경 후 결과 확인 → 별도 SELECT 불필요
  DataGrip 에서 쿼리 결과 바로 확인
  INSERT 후 생성된 id 즉시 확인

⚠️ PostgreSQL 전용
  MySQL → LAST_INSERT_ID() 사용
```


---
---

#  UPSERT — ON CONFLICT ⭐️

```
UPSERT = INSERT + UPDATE 합성어
이미 존재하면 UPDATE / 없으면 INSERT
```

## PostgreSQL — ON CONFLICT

```sql
-- 충돌 시 무시 (아무것도 안 함)
INSERT INTO movies (id, title, rating)
VALUES (1, '기생충', 8.5)
ON CONFLICT (id) DO NOTHING;

-- 충돌 시 UPDATE
INSERT INTO movies (id, title, rating)
VALUES (1, '기생충', 9.0)
ON CONFLICT (id) DO UPDATE
  SET rating = EXCLUDED.rating,
  --            ↑ 새로 넣으려던 값
      title  = EXCLUDED.title;
```

```
EXCLUDED:
  INSERT 하려다 충돌된 새 값을 담은 가상 테이블
  EXCLUDED.rating = 방금 넣으려던 9.0

ON CONFLICT (컬럼):
  어떤 컬럼 충돌 시 처리할지 지정
  해당 컬럼에 UNIQUE 또는 PK 제약 필요
```

## MySQL — ON DUPLICATE KEY UPDATE

```sql
INSERT INTO movies (id, title, rating)
VALUES (1, '기생충', 9.0)
ON DUPLICATE KEY UPDATE
  rating = VALUES(rating),
  title  = VALUES(title);
```

## TypeORM upsert()

```typescript
await this.movieRepository.upsert(
  { id: 1, title: '기생충', rating: 9.0 },
  ['id']   // 충돌 감지 기준 컬럼
);
```

---

---

# 한눈에

|명령어|역할|주의|
|---|---|---|
|`INSERT INTO ... VALUES`|행 추가||
|`INSERT INTO ... SELECT`|다른 테이블에서 삽입||
|`UPDATE ... SET ... WHERE`|행 수정|WHERE 없으면 전체 수정 ⚠️|
|`DELETE FROM ... WHERE`|행 삭제|WHERE 없으면 전체 삭제 ⚠️|
|`RETURNING`|변경 행 바로 반환|PostgreSQL 전용|
|`ON CONFLICT DO UPDATE`|UPSERT (PostgreSQL)||
|`ON DUPLICATE KEY UPDATE`|UPSERT (MySQL)||
```
따옴표 규칙 요약:
  '값'   → 문자열 값 (항상)
  "컬럼" → 식별자 (대문자 테이블/컬럼명, Prisma 테이블)
```