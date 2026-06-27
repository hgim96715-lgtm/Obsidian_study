---
aliases:
  - PostgreSQL 전용
  - FILTER
  - ILIKE
  - ~ 연산자
  - ILIKE ANY
  - '"큰따옴표" 식별자'
  - (^| ) 패턴
tags:
  - SQL
related:
  - "[[00_SQL_HomePage]]"
  - "[[SQL_Aggregate]]"
  - "[[SQL_CASE_WHEN]]"
  - "[[SQL_SELECT]]"
---

# PG_Specific — PostgreSQL 전용 기능

## 한 줄 요약

```txt
PostgreSQL 에서만 쓸 수 있는 문법
MySQL 과 다른 점 정리
```

---

---

#  :: 캐스팅 — 타입 변환 ⭐️

```sql
-- :: 연산자로 타입 변환
값::타입

-- 예시
rating::numeric        -- integer → numeric (실수)
rating::float          -- integer → float
'2024-01-01'::date     -- 문자열 → date
'123'::integer         -- 문자열 → integer
```

```txt
MySQL 은 CAST(값 AS 타입) 만 사용
PostgreSQL 은 :: 단축형도 사용 가능

-- 동일한 결과
CAST(rating AS NUMERIC)   -- 표준 SQL
rating::numeric           -- PostgreSQL 단축형
```

## 왜 ::numeric 이 필요한가 ⭐️

```sql
-- 문제: integer / integer = integer (소수점 버림)
SELECT 1 / 3;          -- 결과: 0  ← 틀림!

-- 해결: ::numeric 으로 실수 변환
SELECT 1::numeric / 3; -- 결과: 0.3333...  ← 맞음

-- 실전 패턴
ROUND(AVG(rating::numeric / position), 2)
--          ↑ integer 를 numeric 으로 변환 후 나눗셈
--          → 소수점 포함 평균 계산 가능
```

---
## 비율 계산 패턴 — PostgreSQL vs MySQL ⭐️

```sql
-- PostgreSQL: ::numeric 없으면 결과가 0 이 됨
SELECT
    ROUND(COUNT(*)::numeric / (SELECT COUNT(*) FROM Users) * 100, 2) AS percentage
FROM Register r
GROUP BY r.contest_id;

-- MySQL: 캐스팅 없이 바로 나눗셈 가능
SELECT
    ROUND(COUNT(*) / (SELECT COUNT(*) FROM Users) * 100, 2) AS percentage
FROM Register r
GROUP BY r.contest_id;
```

```txt
PostgreSQL 나눗셈 주의:
  정수 / 정수 = 정수 (소수점 자동 버림)
  3 / 10 = 0  ← 비율이 0 이 됨!

  → 피제수를 ::numeric 으로 캐스팅 필수

MySQL 나눗셈:
  정수 / 정수 = 소수 자동 처리
  3 / 10 = 0.3  (캐스팅 불필요)

대안:
  COUNT(*) * 1.0 / 전체수   ← 1.0 곱해서 float 유도
  COUNT(*)::float / 전체수  ← float 으로 캐스팅
```

---

---

#  FILTER — 조건부 집계 ⭐️

```sql
-- FILTER (WHERE 조건) = 조건에 맞는 행만 집계
COUNT(*) FILTER (WHERE rating < 3)
SUM(amount) FILTER (WHERE status = 'approved')
AVG(score) FILTER (WHERE score IS NOT NULL)

-- CASE WHEN 과 비교
-- PostgreSQL (FILTER)
COUNT(*) FILTER (WHERE rating < 3)

-- 표준 SQL (CASE WHEN)
SUM(CASE WHEN rating < 3 THEN 1 ELSE 0 END)

-- 빼기도 가능 계산가능
SUM(timestamp) FILTER (WHERE activity_type = 'end')
-
SUM(timestamp) FILTER (WHERE activity_type = 'start') AS process_time

-- 결과 동일 / PostgreSQL 에서는 FILTER 가 더 간결
```

## 실전 — 비율 계산

```sql
-- poor_query_percentage: 평점 3 미만 비율
SELECT
    query_name,
    ROUND(AVG(rating::numeric / position), 2)               AS quality,
    ROUND(
        100.0 * COUNT(*) FILTER (WHERE rating < 3) / COUNT(*),
        2
    )                                                        AS poor_query_percentage
FROM Queries
GROUP BY query_name;
```

---

---

#  DISTINCT ON — 그룹별 첫 번째 행 ⭐️

```sql
-- 각 user_id 의 가장 최근 주문 1건
SELECT DISTINCT ON (user_id)
    user_id, order_date, amount
FROM orders
ORDER BY user_id, order_date DESC;
--        ↑ DISTINCT ON 기준과 ORDER BY 첫 컬럼 일치 필요
```

```txt
MySQL 에서는 ROW_NUMBER() OVER () 로 대체

-- PostgreSQL
SELECT DISTINCT ON (user_id) ...

-- MySQL 동일 결과
SELECT * FROM (
    SELECT *, ROW_NUMBER() OVER (PARTITION BY user_id ORDER BY order_date DESC) AS rn
    FROM orders
) t WHERE rn = 1;
```

---

---

#  ILIKE — 대소문자 무시 검색

```txt
LIKE  = 대소문자 구분 (case-sensitive)
ILIKE = 대소문자 무시 (case-insensitive) — PostgreSQL 전용
```

```sql
-- LIKE = 대소문자 구분
WHERE title LIKE '%MOVIE%'   -- 'movie' 는 매칭 안 됨

-- ILIKE = 대소문자 무시 (PostgreSQL 전용)
WHERE title ILIKE '%movie%'  -- 'MOVIE', 'Movie', 'movie' 모두 매칭

-- MySQL 은 기본이 대소문자 무시
WHERE title LIKE '%movie%'   -- MySQL 에서는 대소문자 무시
```

## LIKE vs ILIKE 비교 ⭐️

```sql
-- 데이터: ['Hello', 'hello', 'HELLO', 'world']

WHERE name LIKE 'hello'    -- 'hello' 만 매칭
WHERE name ILIKE 'hello'   -- 'Hello' / 'hello' / 'HELLO' 모두 매칭
```

```txt
MySQL 과 차이:
  MySQL      LIKE = 기본으로 대소문자 무시 (collation 따라 다름)
  PostgreSQL LIKE = 구분 / ILIKE = 무시

  → PostgreSQL 에서 대소문자 무시 검색은 반드시 ILIKE 사용
```

## 와일드카드 패턴

```sql
-- % = 0개 이상의 문자
WHERE title ILIKE '%movie%'   -- movie 포함 (어디든)
WHERE title ILIKE 'the%'      -- the 로 시작
WHERE title ILIKE '%2024'     -- 2024 로 끝남

-- _ = 정확히 1개 문자
WHERE code ILIKE 'A_C'        -- A로 시작, C로 끝, 중간 1글자
-- 'ABC' / 'aXc' / 'AZC' 매칭

-- NOT ILIKE
WHERE title NOT ILIKE '%spam%'   -- spam 포함 제외 (대소문자 무관)
```

## 여러 패턴 한 번에 — ILIKE ANY (PostgreSQL 특화)

```sql
-- ❌ OR 반복 (지저분)
WHERE email ILIKE '%gmail%' OR email ILIKE '%naver%'

-- ✅ ILIKE ANY (깔끔)
WHERE email ILIKE ANY (ARRAY['%gmail%', '%naver%'])
```

```txt
ILIKE ANY:
  배열 안의 패턴 중 하나라도 일치하면 반환
  OR 여러 개와 같은 결과 / 더 간결
  PostgreSQL 전용
```

---

---
# 정규식 패턴 매칭 — ~ 연산자 ⭐️

```sql
-- ~ : 대소문자 구분, 매칭됨
WHERE mail ~ '^[A-Za-z][A-Za-z0-9._-]*@leetcode[.]com$'

-- ~* : 대소문자 무시, 매칭됨
WHERE name ~* '^(A|B)'

-- !~ : 대소문자 구분, 매칭 안 됨
WHERE email !~ '@spam.com$'
```

|연산자|대소문자 구분|의미|
|---|---|---|
|`~`|✅ 구분|매칭됨|
|`~*`|❌ 무시|매칭됨|
|`!~`|✅ 구분|매칭 안 됨|
|`!~*`|❌ 무시|매칭 안 됨|

```txt
주요 정규식 패턴:
  ^         줄 시작
  $         줄 끝
  [A-Za-z]  영문 대소문자
  [0-9]     숫자
  *         0개 이상
  +         1개 이상
  [.]       리터럴 점 (. 자체)
```

## (^| ) 패턴 — 단어 시작 찾기 ⭐️

```sql
-- DIAB1 로 시작하거나, 공백 바로 뒤에 DIAB1 이 오는 경우
WHERE conditions ~ '(^| )DIAB1'

-- 예시 데이터:
-- 'DIAB100 HEART2'   → 맨 앞 DIAB1 → 매칭 ✅
-- 'HEART2 DIAB100'   → 공백 뒤 DIAB1 → 매칭 ✅
-- 'ADIAB100'         → 앞에 문자 있음 → 매칭 안 됨 ❌
```

```txt
(^| ) 의미:
  ^ 또는 공백( ) 중 하나로 시작
  → 단어의 시작 위치에 패턴이 오는지 확인

왜 LIKE '%DIAB1%' 로 안 되나:
  'ADIAB100' 처럼 앞에 다른 문자가 붙어도 매칭됨
  → 'DIAB1 로 시작하는 단어' 를 정확히 찾으려면 정규식 필요
```

---
---
# "큰따옴표" 식별자 — TypeORM camelCase 컬럼 주의

```sql
-- [[NestJS_TypeORM]] [[NestJS_TypeORM_QueryBuilder]] 참고 
-- TypeORM Entity 에서 camelCase 컬럼은 DB 에 큰따옴표로 생성됨
-- likeCount → "likeCount"

-- ❌ 큰따옴표 없이 → 에러
SELECT likeCount FROM movie;    -- PostgreSQL 이 likecount 로 변환 → 컬럼 없음

-- ✅ 큰따옴표 필수
SELECT "likeCount" FROM movie;
SELECT m."likeCount", m."dislikeCount" FROM movie m;
```

```txt
왜 이런 문제가 생기나:
  PostgreSQL 은 따옴표 없는 식별자를 모두 소문자로 변환
  TypeORM 이 likeCount → "likeCount" 로 DB 에 생성
  → SELECT 할 때도 "likeCount" 큰따옴표 필요

TypeORM QueryBuilder:
  자동 처리 → 신경 안 써도 됨

Raw Query / DataGrip:
  직접 써야 함 → "likeCount" 큰따옴표 명시
  에러 나면 컬럼명에 큰따옴표 붙여보기
```
---
---
# 튜플 비교 — 다중 컬럼 

```sql
-- OR 방식 (장황함)
WHERE "likeCount" < 20
   OR ("likeCount" = 20 AND id < 35)

-- 튜플 비교 (간결 / PostgreSQL 전용)
WHERE ("likeCount", id) < (20, 35)
--     ↑ (likeCount < 20) OR (likeCount = 20 AND id < 35)
```

```bash
튜플 비교 동작:
  (a, b) < (x, y)
  = a < x  OR  (a = x AND b < y)

  (a, b) > (x, y)
  = a > x  OR  (a = x AND b > y)

언제 쓰나:
  Cursor Based Pagination 다중 컬럼 기준
  ORDER BY "likeCount" DESC, id DESC 정렬 시
  → WHERE ("likeCount", id) < (커서likeCount, 커서id)

⚠️ MySQL 에서는 동작 안 함 → OR 방식 사용
# [[NestJS_Pagination]] 참고 
```

---
---

# MySQL vs PostgreSQL 주요 차이

| 기능         | MySQL                    | PostgreSQL                    |
| ---------- | ------------------------ | ----------------------------- |
| 타입 변환      | `CAST(값 AS 타입)`          | `값::타입` 또는 CAST               |
| 조건부 집계     | `SUM(CASE WHEN ... END)` | `COUNT(*) FILTER (WHERE ...)` |
| 대소문자 무시 검색 | `LIKE` (기본 무시)           | `ILIKE`                       |
| 그룹별 첫 행    | ROW_NUMBER()             | `DISTINCT ON`                 |
| 배열 타입      | 없음                       | `int[]`, `text[]`             |
| NULL 처리    | `IFNULL()`               | `COALESCE()`                  |
| 날짜 포맷      | `DATE_FORMAT()`          | `TO_CHAR()`                   |