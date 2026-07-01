---
aliases: [서브쿼리, CTE(WITH), DELETE, INSERT, JOIN, SELECT, UPDATE]
tags: [SQL, PostgreSQL]
related:
  - "[[00_DB_HomePage]]"
  - "[[NestJS_PostgreSQL]]"
  - "[[NestJS_Prisma]]"
  - "[[PG_Aggregate]]"
  - "[[PG_DDL]]"
  - "[[PG_Types]]"
---
# PG_DML — 데이터 조작 & 조회

> [!info]
>  SELECT · JOIN · 서브쿼리 · CTE(WITH) · INSERT / UPDATE / DELETE. RETURNING과 ON CONFLICT는 PostgreSQL 특화 — Prisma도 내부적으로 이 SQL을 생성

---

# 따옴표 규칙 ⭐️⭐️⭐️

```txt
작은따옴표 '  → 문자열 값 (string literal)
큰따옴표   "  → 식별자 (테이블명, 컬럼명)

PostgreSQL에서 따옴표 없는 식별자 → 자동으로 소문자 변환
  User   → user  (다른 테이블로 인식될 수 있음)
  "User" → User  (대소문자 그대로 유지)

Prisma가 생성한 테이블명은 PascalCase → DataGrip에서 쓸 때 항상 "User"처럼 " 로 감싸기
```

```sql
-- ❌ 잘못됨 — 큰따옴표로 값을 지정
UPDATE "User" SET nickname = "공이" WHERE email = "admin@example.com";
-- "공이" → 컬럼명으로 해석 → 에러

-- ✅ 올바름 — 값은 항상 작은따옴표
UPDATE "User" SET nickname = '공이' WHERE email = 'admin@example.com';
```

---

# SELECT 기본 ⭐️

```sql
SELECT id, title, created_at
FROM post
WHERE is_active = TRUE
  AND created_at >= '2025-01-01'
ORDER BY created_at DESC
LIMIT 10
OFFSET 20;   -- OFFSET = (page - 1) * limit
```

```txt
실행 순서 (중요):
  FROM → WHERE → GROUP BY → HAVING → SELECT → ORDER BY → LIMIT/OFFSET

  SELECT에서 정의한 별칭(AS)은 WHERE에서 사용 불가 (아직 SELECT가 실행 안 됨)
  → WHERE price * 0.9 > 1000  (직접 표현식 써야 함)
  → CTE나 인라인 뷰로 우회 가능
```

---

# JOIN ⭐️⭐️⭐️⭐️

## JOIN 종류

|JOIN|포함하는 행|
|---|---|
|`INNER JOIN`|양쪽 모두 매칭되는 행만|
|`LEFT JOIN`|왼쪽 전부 + 오른쪽 매칭 (없으면 NULL)|
|`RIGHT JOIN`|오른쪽 전부 + 왼쪽 매칭 (없으면 NULL)|
|`FULL JOIN`|양쪽 전부 (없으면 NULL)|
|`CROSS JOIN`|모든 조합 (카테시안 곱)|

```sql
-- INNER JOIN: 게시글 + 작성자 (작성자 없으면 게시글도 안 나옴)
SELECT p.title, u.nickname
FROM post p
INNER JOIN "user" u ON p.user_id = u.id;

-- LEFT JOIN: 모든 게시글 + 작성자 (작성자 정보 없어도 게시글 나옴)
SELECT p.title, u.nickname
FROM post p
LEFT JOIN "user" u ON p.user_id = u.id;

-- 다중 JOIN
SELECT p.title, u.nickname, COUNT(c.id) AS comment_count
FROM post p
LEFT JOIN "user" u    ON p.user_id = u.id
LEFT JOIN comment c   ON c.post_id = p.id
GROUP BY p.id, p.title, u.nickname;
```

## ON vs WHERE — LEFT JOIN에서의 함정 ⭐️⭐️⭐️

```sql
-- ❌ WHERE에 오른쪽 테이블 조건 → NULL 행이 사라짐 (INNER JOIN처럼 동작)
SELECT p.product_id, AVG(p.price * u.units)
FROM price p
LEFT JOIN units_sold u ON p.product_id = u.product_id
WHERE u.sale_date BETWEEN p.start_date AND p.end_date;  -- NULL 행 제거됨

-- ✅ ON에 AND로 조건 → NULL 행 유지 (LEFT JOIN 의도대로)
SELECT p.product_id, AVG(p.price * u.units)
FROM price p
LEFT JOIN units_sold u
    ON  p.product_id = u.product_id
    AND u.sale_date BETWEEN p.start_date AND p.end_date;  -- 조건 불만족 시 u = NULL
```

```txt
판단 기준:
  조건 불만족 시 오른쪽 테이블을 NULL로 남기고 싶다 → ON + AND
  조건 불만족 시 그 행 자체를 제거하고 싶다         → WHERE
```

## Anti-Join — 매칭 안 된 행 찾기 ⭐️⭐️⭐️

```sql
-- 댓글 없는 게시글 찾기
SELECT p.id, p.title
FROM post p
LEFT JOIN comment c ON c.post_id = p.id
WHERE c.id IS NULL;   -- c.id = PK 컬럼 (일반 컬럼이면 안 됨)
```

```txt
⚠️ IS NULL 검사는 반드시 오른쪽 테이블의 PK(또는 NOT NULL 컬럼)에 걸어야 함
  일반 컬럼에 걸면 "매칭은 됐지만 그 컬럼이 원래 NULL인 경우"와
  "매칭 자체가 안 된 경우"를 구분 못 함

동작 원리:
  LEFT JOIN → 댓글 없는 게시글의 c.* 자리는 NULL로 채워짐
  WHERE c.id IS NULL → 그 NULL인 행 = 댓글이 없는 게시글만 남김
```

## CROSS JOIN + LEFT JOIN — 빈 조합도 0으로 ⭐️⭐️

```sql
-- 학생별 × 과목별 응시 횟수 (안 본 과목도 0으로 표시)
SELECT
    s.student_id,
    sub.subject_name,
    COUNT(e.subject_name) AS attended
FROM student s
CROSS JOIN subject sub                      -- 모든 학생 × 과목 조합 생성
LEFT JOIN exam e
    ON  s.student_id = e.student_id
    AND sub.subject_name = e.subject_name   -- ON에! (WHERE에 넣으면 0이 사라짐)
GROUP BY s.student_id, sub.subject_name;
```

---

# 서브쿼리 ⭐️⭐️⭐️

## 위치별 종류

|위치|이름|용도|
|---|---|---|
|SELECT 절|스칼라 서브쿼리|행마다 단일 값 계산|
|FROM 절|인라인 뷰|가상 테이블|
|WHERE 절|조건 서브쿼리|동적 필터|

```sql
-- 스칼라 서브쿼리 — 전체 평균과 비교
SELECT title, price,
    (SELECT AVG(price) FROM product) AS avg_price   -- 모든 행에 같은 값
FROM product
WHERE price > (SELECT AVG(price) FROM product);

-- 인라인 뷰 — FROM 안에서 가상 테이블로 (반드시 AS 필요)
SELECT dept, avg_salary
FROM (
    SELECT dept, AVG(salary)::numeric AS avg_salary
    FROM employee
    GROUP BY dept
) AS dept_stats
WHERE avg_salary > 5000000;

-- IN 서브쿼리
SELECT nickname FROM "user"
WHERE id IN (SELECT user_id FROM post WHERE is_active = TRUE);
```

## EXISTS / NOT EXISTS ⭐️⭐️

```sql
-- 게시글이 하나라도 있는 유저
SELECT u.nickname
FROM "user" u
WHERE EXISTS (
    SELECT 1 FROM post p WHERE p.user_id = u.id
);

-- 게시글이 없는 유저
SELECT u.nickname
FROM "user" u
WHERE NOT EXISTS (
    SELECT 1 FROM post p WHERE p.user_id = u.id
);
```

```txt
EXISTS:
  서브쿼리 결과가 하나라도 있으면 TRUE → 즉시 중단 (성능 효율적)
  반환 값이 중요하지 않아서 SELECT 1 관례
  Anti-Join(LEFT JOIN + IS NULL)과 결과는 같음 — 의도가 명확할 때 EXISTS가 더 읽기 쉬움
```

---

# CTE — WITH ⭐️⭐️⭐️⭐️

```txt
CTE(Common Table Expression) = 쿼리에 이름을 붙여서 재사용
인라인 뷰(FROM 서브쿼리)와 결과는 같지만 → 읽기가 훨씬 쉬워짐
복잡한 쿼리를 단계별로 쪼개서 쓸 수 있음
```

## 기본 문법

```sql
WITH 이름 AS (
    SELECT ...
)
SELECT * FROM 이름 WHERE ...;
```

## 실전 — 단계별로 쪼개기

```sql
-- 월별 게시글 수 + 이전 달 대비 증감
WITH monthly_count AS (
    SELECT
        DATE_TRUNC('month', created_at AT TIME ZONE 'Asia/Seoul') AS month,
        COUNT(*) AS cnt
    FROM post
    WHERE is_active = TRUE
    GROUP BY month
),
with_prev AS (
    SELECT
        month,
        cnt,
        LAG(cnt) OVER (ORDER BY month) AS prev_cnt  -- 이전 달 값
    FROM monthly_count
)
SELECT
    month,
    cnt,
    cnt - COALESCE(prev_cnt, 0) AS diff
FROM with_prev
ORDER BY month;
```

## 여러 CTE 연결

```sql
WITH
active_users AS (
    SELECT id FROM "user" WHERE is_active = TRUE
),
recent_posts AS (
    SELECT user_id, COUNT(*) AS cnt
    FROM post
    WHERE created_at >= NOW() - INTERVAL '30 days'
      AND user_id IN (SELECT id FROM active_users)  -- 앞의 CTE 참조
    GROUP BY user_id
)
SELECT u.nickname, rp.cnt
FROM "user" u
JOIN recent_posts rp ON rp.user_id = u.id
ORDER BY rp.cnt DESC;
```

```txt
CTE vs 인라인 뷰:
  (SELECT ... FROM ...) AS alias  → 인라인 뷰, 중첩되면 읽기 어려움
  WITH name AS (SELECT ...)       → CTE, 단계별로 이름 붙여서 읽기 쉬움

재귀 CTE(WITH RECURSIVE):
  트리 구조(카테고리 계층, 조직도 등) 조회에 사용
  → 별도 노트에서 다룰 내용
```

---

# INSERT ⭐️

```sql
-- 단건
INSERT INTO post (user_id, title, is_active)
VALUES (1, '첫 번째 게시글', TRUE);

-- 여러 행 한 번에 (batch insert — 개별 INSERT보다 훨씬 빠름)
INSERT INTO post (user_id, title)
VALUES
    (1, '게시글 1'),
    (1, '게시글 2'),
    (2, '게시글 3');

-- 다른 테이블에서 복사
INSERT INTO post_archive (user_id, title, created_at)
SELECT user_id, title, created_at
FROM post
WHERE created_at < NOW() - INTERVAL '1 year';
```

---

# UPDATE ⭐️⭐️

```sql
-- 기본
UPDATE post
SET title = '수정된 제목', updated_at = NOW()
WHERE id = 1;

-- 계산식 적용
UPDATE product
SET price = price * 1.1      -- 10% 인상
WHERE category = 'premium';

-- 다른 테이블 값으로 UPDATE (PostgreSQL: FROM 절)
UPDATE post p
SET view_count = stats.cnt
FROM post_stats stats
WHERE stats.post_id = p.id;
```

```txt
⚠️ WHERE 없으면 전체 행 수정 → 항상 WHERE 먼저 확인
  UPDATE post SET is_active = FALSE;  -- 모든 게시글 비활성화 ← 위험
```

---

# DELETE ⭐️

```sql
-- 기본
DELETE FROM post WHERE id = 1;

-- 삭제 전 SELECT로 먼저 확인하는 습관 ⭐️
SELECT * FROM post WHERE created_at < '2024-01-01' AND is_active = FALSE;
-- 확인 후 DELETE
DELETE FROM post WHERE created_at < '2024-01-01' AND is_active = FALSE;
```

```txt
⚠️ WHERE 없으면 전체 삭제 → TRUNCATE와 달리 롤백은 가능하지만 그래도 위험
  실서버에서 DELETE 전 SELECT로 반드시 확인
```

---

# RETURNING — 변경 결과 바로 반환 ⭐️⭐️⭐️ (PostgreSQL 전용)

```sql
-- INSERT 후 생성된 id 즉시 확인
INSERT INTO post (user_id, title)
VALUES (1, '새 게시글')
RETURNING id, title, created_at;

-- UPDATE 후 변경된 값 확인
UPDATE "user"
SET last_active_at = NOW()
WHERE id = 1
RETURNING id, last_active_at;

-- DELETE 후 삭제된 행 데이터 보존
DELETE FROM post
WHERE id = 1
RETURNING *;
```

```txt
RETURNING의 실용적 가치:
  별도 SELECT 없이 한 번에 처리 → DB 왕복 횟수 감소
  INSERT 후 DB가 생성한 id, default 값 등 즉시 확인

Prisma에서는 내부적으로 RETURNING을 사용:
  create() → INSERT ... RETURNING *
  update() → UPDATE ... RETURNING *
  → 반환 타입이 정확한 이유
```

---

# ON CONFLICT — UPSERT ⭐️⭐️⭐️ (PostgreSQL 전용)

```sql
-- 충돌 시 무시
INSERT INTO user_follow (follower_id, following_id)
VALUES (1, 2)
ON CONFLICT (follower_id, following_id) DO NOTHING;

-- 충돌 시 UPDATE
INSERT INTO post_stat (post_id, view_count)
VALUES (1, 1)
ON CONFLICT (post_id) DO UPDATE
    SET view_count = post_stat.view_count + EXCLUDED.view_count,
        --                 ↑ 기존 값              ↑ 새로 넣으려던 값
        updated_at = NOW();
```

```txt
EXCLUDED:
  INSERT하려다 충돌된 "새 값"을 담은 가상 테이블
  EXCLUDED.view_count = 방금 넣으려던 값 (1)
  post_stat.view_count = 현재 DB에 있는 값 → 더해서 누적

ON CONFLICT (컬럼):
  어떤 컬럼이 충돌할 때 처리할지 지정
  해당 컬럼에 UNIQUE 또는 PK 제약 필요

Prisma upsert()와의 관계:
  this.prisma.xxx.upsert({ where, create, update })
  → 내부적으로 INSERT ... ON CONFLICT ... DO UPDATE 실행
```

---

# 한눈에

```txt
따옴표 규칙:
  '값'    → 문자열 항상 작은따옴표
  "테이블" → Prisma 생성 테이블(PascalCase) + 예약어 포함 시 큰따옴표

SELECT 실행 순서:
  FROM → WHERE → GROUP BY → HAVING → SELECT → ORDER BY → LIMIT

JOIN 선택:
  양쪽 다 있는 것만 → INNER
  왼쪽 무조건 전부 → LEFT
  LEFT JOIN + WHERE 오른쪽.PK IS NULL → Anti-Join (매칭 안 된 것 찾기)
  LEFT JOIN + ON AND 조건 → 조건 불만족 시 NULL로 유지
  CROSS JOIN + LEFT JOIN → 빈 조합도 0으로

서브쿼리:
  SELECT절 → 스칼라 / FROM절 → 인라인 뷰 / WHERE절 → 조건
  EXISTS/NOT EXISTS → 존재 여부 확인 (상관 서브쿼리)

CTE (WITH):
  이름 붙인 가상 테이블 → 인라인 뷰보다 읽기 쉬움
  여러 CTE 연결 → 앞의 CTE를 다음 CTE에서 참조 가능

PostgreSQL 전용:
  RETURNING  → INSERT/UPDATE/DELETE 후 바로 결과 반환
  ON CONFLICT → UPSERT (충돌 시 DO NOTHING 또는 DO UPDATE SET)
  UPDATE ... FROM → 다른 테이블 값으로 업데이트
```