---
aliases:
  - 집계 함수
  - COUNT
  - SUM
  - AVG
  - MAX
  - MIN
  - STRING_AGG
  - GROUP_CONCAT
  - 문자열 집계
tags:
  - SQL
related:
  - "[[00_SQL_HomePage]]"
  - "[[SQL_GROUP_BY]]"
  - "[[SQL_CASE_WHEN]]"
---
# SQL_Aggregate — 집계 함수

## 한 줄 요약

```txt
여러 행을 하나의 값으로 요약하는 함수
COUNT / SUM / AVG / MAX / MIN
GROUP BY 와 함께 쓰는 것이 핵심
문자열 집계: STRING_AGG (PostgreSQL) / GROUP_CONCAT (MySQL)
```

---

---

# 기본 집계 함수

```sql
COUNT(*)              -- 전체 행 수 (NULL 포함)
COUNT(컬럼)           -- NULL 제외한 행 수
COUNT(DISTINCT 컬럼)  -- 중복 제거 후 행 수

SUM(컬럼)             -- 합계
AVG(컬럼)             -- 평균 (NULL 제외)
MAX(컬럼)             -- 최댓값
MIN(컬럼)             -- 최솟값
```

```sql
SELECT
    COUNT(*)         AS 전체건수,
    COUNT(country)   AS 국가있는건수,
    SUM(amount)      AS 총금액,
    AVG(amount)      AS 평균금액,
    MAX(amount)      AS 최대금액,
    MIN(amount)      AS 최소금액
FROM Transactions;
```

---

---

#  COUNT(`*`) vs COUNT(컬럼) ⭐️

```txt
COUNT(*)       전체 행 수  (NULL 포함)
COUNT(컬럼)    NULL 이 아닌 행 수  ← NULL 이면 세지 않음

예시 데이터:
  id | country | amount
   1 | KR      | 100
   2 | NULL    | 200    ← country 가 NULL
   3 | US      | 300

COUNT(*)       → 3
COUNT(country) → 2   (NULL 행 제외)
```

## COUNT(컬럼) — NULL 이면 자동 0 ⭐️

```txt
LEFT JOIN 결과에서 매칭 안 된 행은 컬럼이 NULL
COUNT(컬럼) → NULL 이면 세지 않음 → 자동으로 0

즉, COALESCE 없이도 0 이 나옴!
```

```sql
-- CROSS JOIN + LEFT JOIN 패턴 핵심
SELECT
    s.student_id,
    sub.subject_name,
    COUNT(e.subject_name) AS attended_exams  -- ← e 가 NULL 이면 자동 0
FROM Students s
CROSS JOIN Subjects sub
LEFT JOIN Examinations e
    ON s.student_id = e.student_id
   AND sub.subject_name = e.subject_name
GROUP BY s.student_id, sub.subject_name;

-- COUNT(e.subject_name):
--   e 가 매칭됨  → e.subject_name = 'Math' → COUNT +1
--   e 가 없음    → e.subject_name = NULL   → COUNT 무시 → 0

-- COUNT(*) 로 하면 안 됨:
-- COUNT(*) 는 NULL 행도 1 로 셈 → 0 이 아닌 1 나옴!
```

```txt
핵심 차이:
  COUNT(*)          NULL 포함 → 시험 안 봐도 1 나옴 ❌
  COUNT(e.컬럼)     NULL 제외 → 시험 안 보면 0    ✅
  COUNT(DISTINCT 컬럼)  중복 제거 후 → 같은 과목 여러 번 시험봐도 1
```

---

---

#  조건부 집계 ⭐️

## PostgreSQL — FILTER

```sql
-- FILTER (WHERE 조건): 조건 맞는 행만 집계
SELECT
    COUNT(*) FILTER (WHERE state = 'approved')            AS approved_count,
    SUM(amount) FILTER (WHERE state = 'approved')         AS approved_total
FROM Transactions;
```

## MySQL / 공통 — CASE WHEN

```sql
-- SUM(CASE WHEN 조건 THEN 1 ELSE 0): 건수
-- SUM(CASE WHEN 조건 THEN amount ELSE 0): 금액 합계
SELECT
    SUM(CASE WHEN state = 'approved' THEN 1 ELSE 0 END)      AS approved_count,
    SUM(CASE WHEN state = 'approved' THEN amount ELSE 0 END)  AS approved_total
FROM Transactions;
```

```txt
FILTER vs CASE WHEN:
  FILTER      PostgreSQL 전용 / 직관적
  CASE WHEN   모든 DB 호환 / 표준
  결과 동일
```

---
---
# 문자열 집계 — STRING_AGG / GROUP_CONCAT ⭐️

```txt
여러 행의 값을 하나의 문자열로 합치는 집계 함수
GROUP BY 와 함께 사용
```

## PostgreSQL — STRING_AGG

```sql
SELECT
    sell_date,
    COUNT(DISTINCT product)                                   AS num_sold,
    STRING_AGG(DISTINCT product, ',' ORDER BY product)        AS products
FROM Activities
GROUP BY sell_date
ORDER BY sell_date;

-- 결과
-- sell_date   | num_sold | products
-- 2020-05-30  | 3        | Headphone,Keyboard,Mouse
```

```txt
STRING_AGG(컬럼, 구분자 ORDER BY 정렬기준):
  첫 번째 인자  합칠 컬럼
  두 번째 인자  구분자 (',' / ' / ' / ' ')
  ORDER BY     합칠 순서 (문자열 내부 정렬)
  DISTINCT     중복 제거

STRING_AGG(product, ',')               → 순서 랜덤
STRING_AGG(product, ',' ORDER BY product)  → 알파벳 순
STRING_AGG(DISTINCT product, ',')      → 중복 제거 (ORDER BY 같이 쓸 수 있음)
```

## MySQL — GROUP_CONCAT

```sql
SELECT
    sell_date,
    COUNT(DISTINCT product)                                         AS num_sold,
    GROUP_CONCAT(DISTINCT product ORDER BY product SEPARATOR ',')   AS products
FROM Activities
GROUP BY sell_date
ORDER BY sell_date;
```

```txt
GROUP_CONCAT(DISTINCT 컬럼 ORDER BY 정렬기준 SEPARATOR '구분자'):
  DISTINCT     중복 제거
  ORDER BY     합칠 순서
  SEPARATOR    구분자 (기본값 ',')
```

## PostgreSQL vs MySQL 비교

| 목적     | PostgreSQL                 | MySQL                      |
| ------ | -------------------------- | -------------------------- |
| 함수명    | `STRING_AGG()`             | `GROUP_CONCAT()`           |
| 구분자 위치 | 두 번째 인자 `','`              | `SEPARATOR ','`            |
| 중복 제거  | `DISTINCT product`         | `DISTINCT product`         |
| 내부 정렬  | `ORDER BY product` (인자 안에) | `ORDER BY product` (인자 안에) |


---

---

#  NULL 주의사항 ⭐️

## SUM / AVG / MIN / MAX 의 NULL 처리

```txt
SUM · AVG · MIN · MAX 는 NULL 을 투명인간 취급
→ NULL 은 계산에서 아예 빠짐 (0 이 아님!)

COUNT(*) 와 다르게:
  NULL 행이 있어도 SUM / AVG 에는 영향 없음
  하지만 결과 자체가 NULL 이 될 수 있음
```

```sql
-- 데이터: [100, NULL, 300]
SUM(amount)   → 400   (NULL 은 더하지 않음)
AVG(amount)   → 200   (NULL 제외 → 100+300 / 2)
COUNT(amount) → 2     (NULL 제외)
COUNT(*)      → 3     (NULL 포함)
```

## NULL 이 아예 없거나 전부 NULL 일 때

|상황|COUNT(*)|SUM / AVG / MIN / MAX|
|---|:-:|:-:|
|행이 아예 없을 때|**0**|**NULL**|
|대상 컬럼이 전부 NULL 일 때|**행 수 그대로**|**NULL**|

```sql
-- 빈 테이블에서
SELECT COUNT(*), SUM(amount) FROM empty_table;
-- COUNT(*) = 0  / SUM = NULL (0 이 아님!)
```

## AVG 함정 — NULL 을 0 으로 포함하고 싶을 때

```sql
-- 기본 AVG → NULL 제외하고 평균
AVG(amount)

-- NULL 을 0 으로 포함해서 평균 내고 싶으면
AVG(COALESCE(amount, 0))   -- NULL → 0 변환 후 평균

-- 예시: [100, NULL, 300]
AVG(amount)                → (100+300)/2 = 200   (NULL 제외)
AVG(COALESCE(amount, 0))   → (100+0+300)/3 = 133  (NULL → 0 포함)
```

## NULL 전파 ⚠️ — 집계 결과를 연산에 쓸 때

```sql
-- SUM 이 NULL 이면 이 계산 전체가 NULL
SELECT SUM(bonus) + 1000 FROM employees;
-- SUM = NULL 이면 → NULL + 1000 = NULL!

-- 안전하게: COALESCE 로 NULL → 0 처리
SELECT COALESCE(SUM(bonus), 0) + 1000 FROM employees;
-- SUM = NULL 이면 → 0 + 1000 = 1000 ✅
```

```txt
NULL 전파 규칙:
  NULL 이 포함된 연산 결과는 모두 NULL
  NULL + 1000 = NULL
  NULL * 2    = NULL
  NULL = NULL = NULL  (비교도 NULL)

  → 집계 결과 바로 연산할 때는 COALESCE 필수
```

---

---

#  MIN / MAX + HAVING ⭐️

```sql
-- 그룹 전체 범위 검증
SELECT
    department,
    MIN(salary) AS min_salary,
    MAX(salary) AS max_salary
FROM employees
GROUP BY department
HAVING MAX(salary) > 5000;  -- 최댓값 조건으로 그룹 필터
-- 최고 연봉이 5000 초과인 부서만
```

---

---

#  중첩 집계 불가 — SQL 집계 한 번의 법칙 ⭐️

```txt
SQL 은 한 번의 단계에서 오직 한 번의 집계만 가능

❌ 중첩 집계 시도 (에러):
  AVG(SUM(amount))       -- 집계 안에 집계 불가
  MAX(COUNT(*))          -- 에러
  SUM(AVG(score))        -- 에러
```

```sql
-- ❌ 에러: 중첩 집계
SELECT AVG(SUM(amount))
FROM orders
GROUP BY product_id;

-- ✅ 해결 1: CTE 로 단계 분리
WITH product_totals AS (
    SELECT product_id, SUM(amount) AS total
    FROM orders
    GROUP BY product_id
)
SELECT AVG(total) FROM product_totals;

-- ✅ 해결 2: 서브쿼리
SELECT AVG(total) FROM (
    SELECT SUM(amount) AS total
    FROM orders
    GROUP BY product_id
) t;
```

```txt
중첩 집계 만나면:
  1. 안쪽 집계 → CTE 또는 서브쿼리
  2. 바깥 집계 → 그 결과에 적용

  한 쿼리에서 집계는 딱 한 번 = GROUP BY 한 레벨
```

---

---

#  비율 계산 — 정수 나눗셈 주의 ⭐️

```txt
정수 / 정수 = 정수 (소수점 버림)
→ 비율 계산 시 반드시 실수로 변환 필요

100 * 1 / 3  = 33     ← 정수 나눗셈 (틀림)
100.0 * 1/3  = 33.33  ← 실수 나눗셈 (맞음)
```

## MySQL 방식 — 100 * SUM(CASE WHEN) / COUNT(*)

```sql
SELECT
    query_name,
    ROUND(AVG(rating / position), 2) AS quality,
    ROUND(
        100 * SUM(CASE WHEN rating < 3 THEN 1 ELSE 0 END) / COUNT(*),
        2
    ) AS poor_query_percentage
FROM Queries
GROUP BY query_name;
```

```txt
주의:
  100 * SUM(...) / COUNT(*)
  MySQL 은 자동 실수 변환 → 보통 동작
  하지만 DB 에 따라 정수 나눗셈 결과 나올 수 있음
  → 안전하게 100.0 쓰는 습관 권장
```

## PostgreSQL 방식 — ::numeric + FILTER ⭐️

```sql
SELECT
    query_name,
    ROUND(AVG(rating::numeric / position), 2) AS quality,
    ROUND(
        100.0 * COUNT(*) FILTER (WHERE rating < 3) / COUNT(*),
        2
    ) AS poor_query_percentage
FROM Queries
GROUP BY query_name;
```

```txt
rating::numeric:
  rating 이 integer 타입 → integer / integer = integer
  ::numeric 으로 캐스팅 → 실수 나눗셈
  AVG(rating::numeric / position) = 소수점 포함 평균

100.0 * COUNT(...) / COUNT(*):
  100.0 (실수) * COUNT → 실수
  → 실수 / 정수 = 실수 → 소수점 나옴

FILTER (WHERE rating < 3):
  PostgreSQL 전용
  COUNT(*) FILTER (WHERE 조건) = 조건에 맞는 건수
  → CASE WHEN 보다 간결

두 쿼리 비교:
  MySQL:      100 * SUM(CASE WHEN rating < 3 THEN 1 ELSE 0 END) / COUNT(*)
  PostgreSQL: 100.0 * COUNT(*) FILTER (WHERE rating < 3) / COUNT(*)
  → 결과 동일 / 문법만 다름
```

## 실수 변환 방법 정리

```sql
-- 방법 1: 100.0 곱하기 (가장 간단)
100.0 * COUNT(*) FILTER (WHERE rating < 3) / COUNT(*)

-- 방법 2: ::numeric 캐스팅 (PostgreSQL)
rating::numeric / position

-- 방법 3: CAST 함수 (표준 SQL)
CAST(rating AS NUMERIC) / position

-- 방법 4: 1.0 곱하기
1.0 * rating / position
```


---

---

# 비율 계산 패턴 ⭐️

```sql
-- 목표: poor_query_percentage = 평점 3 미만 비율 (%)
-- 문제: 정수 / 정수 = 정수 (소수점 버림)

-- ❌ 정수끼리 나누면 소수점 버림
100 * SUM(CASE WHEN rating < 3 THEN 1 ELSE 0 END) / COUNT(*)
-- COUNT 가 10, SUM 이 3 이면 → 100 * 3 / 10 = 30 (정수)

-- ✅ 방법 1: 100.0 으로 실수 변환 (MySQL / PostgreSQL 공통)
ROUND(
  100.0 * SUM(CASE WHEN rating < 3 THEN 1 ELSE 0 END) / COUNT(*),
  2
)

-- ✅ 방법 2: FILTER (PostgreSQL 전용 / 더 직관적)
ROUND(
  100.0 * COUNT(*) FILTER (WHERE rating < 3) / COUNT(*),
  2
)
```

```txt
핵심:
  정수 / 정수 = 정수  ← 소수점 버림!
  해결: 100 대신 100.0 사용 → 실수 나눗셈
  또는 ::numeric / ::float 캐스팅

  ROUND(값, 2) = 소수점 둘째 자리까지 반올림
```

## 실전 예시 — quality & poor_query_percentage

```sql
-- MySQL 호환 방식
SELECT
  query_name,
  ROUND(AVG(rating / position), 2)                            AS quality,
  ROUND(
    100 * SUM(CASE WHEN rating < 3 THEN 1 ELSE 0 END) / COUNT(*),
    2
  )                                                            AS poor_query_percentage
FROM Queries
GROUP BY query_name;

-- PostgreSQL 방식 (::numeric + FILTER)
SELECT
  query_name,
  ROUND(AVG(rating::numeric / position), 2)                   AS quality,
  ROUND(
    100.0 * COUNT(*) FILTER (WHERE rating < 3) / COUNT(*),
    2
  )                                                            AS poor_query_percentage
FROM Queries
GROUP BY query_name;
```

```txt
rating::numeric 왜 쓰나:
  rating, position 이 정수(integer) 타입이면
  rating / position = 정수 나눗셈 → 소수점 버림
  rating::numeric 으로 실수 변환 → 소수점 유지
```

> 자세한 내용 → [[PG_Specific]] :: 캐스팅 섹션

---
---
# SUM vs COUNT — 언제 뭘 쓰나 ⭐️

```txt
SUM   숫자를 더하는 것  (금액, 수량 등)
COUNT 행 수를 세는 것  (몇 번, 몇 개)

문자열 컬럼에 SUM → 에러!
COUNT 는 어떤 컬럼이든 가능
```

```sql
-- ❌ SUM(문자열) — 에러
SUM(e.subject_name)    -- 문자열은 SUM 불가

-- ✅ COUNT(컬럼) — 행 수 세기
COUNT(e.subject_name)  -- NULL 이면 0, 값 있으면 +1

-- SUM: 숫자 더하기
SUM(amount)        -- 금액 합계
SUM(units)         -- 수량 합계
SUM(price * units) -- 계산 후 합계

-- COUNT: 행 수 세기
COUNT(*)             -- NULL 포함 전체 행
COUNT(컬럼)          -- NULL 제외 행 수 ← LEFT JOIN 에서 자동 0
COUNT(DISTINCT 컬럼) -- 중복 제거 후 행 수
```

## LEFT JOIN 에서 COUNT(컬럼) 활용 ⭐️


```sql
-- 시험 응시 횟수 구하기
COUNT(e.subject_name)
-- e.subject_name 이 NULL (시험 안 봄) → 카운트 안 됨 → 0
-- e.subject_name 이 'Math'            → 카운트 +1

-- COUNT(*) 로 하면 안 됨:
-- NULL 행도 1 로 셈 → 0 이 아닌 1 나옴
```

```txt
선택 기준:
  시험 몇 번?  응시 횟수?    → COUNT(컬럼) ✅
  금액 얼마?  매출 합계?    → SUM(컬럼)   ✅
  문자열 컬럼 → SUM 불가     → COUNT 사용
```

>→ LEFT JOIN + CROSS JOIN 패턴은 [[SQL_JOIN]] 참고