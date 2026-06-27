---
aliases:
  - SELECT
  - SQL 기본 조회
  - ORDER BY
  - LIMIT
tags:
  - SQL
related:
  - "[[00_SQL_HomePage]]"
  - "[[SQL_WHERE]]"
  - "[[SQL_GROUP_BY]]"
  - "[[SQL_JOIN]]"
  - "[[SQL_JOIN_Advanced]]"
---

# SQL_SELECT — 기본 조회

## 한 줄 요약

```
데이터를 꺼내는 가장 기본 명령어
SELECT → FROM → WHERE → ORDER BY → LIMIT 순서로 작성
```

---

---

#  기본 구조 ⭐️

```sql
SELECT 컬럼1, 컬럼2
FROM 테이블명
WHERE 조건
ORDER BY 정렬기준
LIMIT 개수;
```

```sql
-- 전체 컬럼 조회
SELECT * FROM movies;

-- 특정 컬럼만
SELECT title, genre FROM movies;

-- 계산식도 가능
SELECT title, price * 0.9 AS discounted_price FROM products;
```

```
⚠️ SELECT * 주의:
  BigQuery 같은 클라우드 환경에서 SELECT *
  → 테이블 전체 스캔 → 비용 폭발
  → 항상 필요한 컬럼만 명시
```

---

---

# 실행 순서 ⭐️

```
작성 순서:       실제 실행 순서:
SELECT           ① FROM
FROM             ② WHERE
WHERE            ③ GROUP BY
GROUP BY         ④ HAVING
HAVING           ⑤ SELECT
ORDER BY         ⑥ ORDER BY
LIMIT            ⑦ LIMIT
```

```
왜 중요한가:
  SELECT 에서 만든 별칭(AS)은 WHERE 에서 못 씀
  → WHERE 가 SELECT 보다 먼저 실행되기 때문

  ❌ SELECT price * 0.9 AS sale WHERE sale < 100
  ✅ SELECT price * 0.9 AS sale WHERE price * 0.9 < 100
```

---

---

#  별칭 (AS)

```sql
-- 컬럼 별칭
SELECT title AS 제목, genre AS 장르 FROM movies;
SELECT price * 0.9 AS discounted_price FROM products;

-- AS 생략 가능 (권장하지 않음)
SELECT title 제목 FROM movies;

-- 공백 포함 별칭 → 큰따옴표
SELECT price * 0.9 AS "할인 가격" FROM products;
-- ⚠️ 홑따옴표('') 사용 금지 → 문자열로 인식됨
```

---

---

#  DISTINCT — 중복 제거

```sql
-- 중복 제거
SELECT DISTINCT genre FROM movies;
-- 드라마 / 액션 / 코미디 (각 1번씩)

-- 여러 컬럼 조합 중복 제거
SELECT DISTINCT genre, director FROM movies;
-- (장르, 감독) 조합이 같은 것 제거
```

---

---

#  ORDER BY — 정렬 ⭐️

```sql
-- 오름차순 (기본)
SELECT * FROM movies ORDER BY title;
SELECT * FROM movies ORDER BY title ASC;   -- ASC 생략 가능

-- 내림차순
SELECT * FROM movies ORDER BY price DESC;

-- 여러 컬럼 정렬
SELECT * FROM movies ORDER BY genre ASC, price DESC;
-- genre 오름차순 → 같은 genre 는 price 내림차순

-- 컬럼 번호로 정렬 (SELECT 순서)
SELECT title, price FROM movies ORDER BY 2 DESC;
--                                        ↑ price (두 번째 컬럼)

-- NULL 처리
SELECT * FROM movies ORDER BY price NULLS LAST;   -- NULL 마지막
SELECT * FROM movies ORDER BY price NULLS FIRST;  -- NULL 처음
```

---

---

#  LIMIT / OFFSET — 개수 제한

```sql
-- 상위 N개만
SELECT * FROM movies LIMIT 5;
SELECT * FROM movies ORDER BY price DESC LIMIT 3;   -- 가장 비싼 3개

-- OFFSET — 건너뛰기 (페이징)
SELECT * FROM movies LIMIT 10 OFFSET 20;
-- 21번째부터 30번째 (0부터 시작)

-- MySQL 에서 LIMIT offset, count 형태도 가능
SELECT * FROM movies LIMIT 20, 10;   -- OFFSET 20, 10개
```

```
LIMIT 없이 ORDER BY DESC + 조건:
  ORDER BY weight_sum DESC LIMIT 1
  → 가장 큰 값 1개 = 최대값 행
  → MAX() 와 비슷하지만 전체 행 정보를 가져올 수 있음
```

---

---

#  산술 / 문자열 연산

```sql
-- 산술
SELECT price * 1.1 AS price_with_tax FROM products;
SELECT (salary - 3000000) / 1000000 AS extra_million FROM employees;

-- 문자열 합치기
-- PostgreSQL
SELECT first_name || ' ' || last_name AS full_name FROM users;

-- MySQL
SELECT CONCAT(first_name, ' ', last_name) AS full_name FROM users;
```

---

---

#  NULL 처리

```sql
-- NULL 은 = 로 비교 불가
SELECT * FROM movies WHERE genre = NULL;    -- ❌ 항상 결과 없음
SELECT * FROM movies WHERE genre IS NULL;   -- ✅
SELECT * FROM movies WHERE genre IS NOT NULL;

-- COALESCE — NULL 이면 기본값
SELECT COALESCE(genre, '미분류') AS genre FROM movies;
-- genre 가 NULL 이면 '미분류' 반환
```

---


---

---

# 명령어 한눈에

|절|역할|예시|
|---|---|---|
|`SELECT 컬럼`|가져올 컬럼|`SELECT title, price`|
|`FROM 테이블`|대상 테이블|`FROM movies`|
|`WHERE 조건`|행 필터|`WHERE price > 10000`|
|`ORDER BY 컬럼 DESC`|정렬|`ORDER BY price DESC`|
|`LIMIT N`|개수 제한|`LIMIT 10`|
|`OFFSET N`|건너뛰기|`OFFSET 20`|
|`DISTINCT`|중복 제거|`SELECT DISTINCT genre`|
|`AS 별칭`|별칭|`price * 0.9 AS sale`|
|`IS NULL`|NULL 비교|`WHERE genre IS NULL`|
|`COALESCE(컬럼, 기본값)`|NULL 대체|`COALESCE(genre, '기타')`|