---
aliases:
  - 집계
  - GROUP BY
  - HAVING
  - FILTER
  - generate_series
  - 날짜 기반 그룹핑
tags:
  - SQL
  - PostgreSQL
related:
  - "[[00_DB_HomePage]]"
  - "[[NestJS_StatsBucket]]"
  - "[[NestJS_PostgreSQL]]"
  - "[[PG_DML]]"
  - "[[PG_Window]]"
---
# PG_Aggregate — 집계 & GROUP BY

> [!info] 
> GROUP BY · HAVING · 집계 함수 · FILTER · generate_series(빈 구간 채우기)
> generate_series는 PostgreSQL 전용 — "데이터 없는 날도 0으로" 표시하는 핵심 패턴.

---

# 집계 함수 한눈에

|함수|의미|NULL 처리|
|---|---|---|
|`COUNT(*)`|전체 행 수|NULL 포함|
|`COUNT(컬럼)`|NULL 아닌 행 수|NULL 제외|
|`COUNT(DISTINCT 컬럼)`|중복 제거 후 행 수|NULL 제외|
|`SUM(컬럼)`|합계|NULL 제외|
|`AVG(컬럼)`|평균|NULL 제외|
|`MAX(컬럼)`|최댓값|NULL 제외|
|`MIN(컬럼)`|최솟값|NULL 제외|

```sql
SELECT
    COUNT(*)                        AS total,
    COUNT(deleted_at)               AS deleted_count,   -- NULL 아닌 것만
    COUNT(DISTINCT user_id)         AS unique_users,
    SUM(amount)                     AS revenue,
    ROUND(AVG(amount)::numeric, 2)  AS avg_amount,      -- ::numeric → 소수점 오차 방지
    MAX(created_at)                 AS latest
FROM payment;
```

```txt
PostgreSQL 정수 나눗셈 주의:
  COUNT(*) / 전체수   → 정수 / 정수 = 정수 (소수점 버림)
  3 / 10 = 0          ← 0이 나옴
  → ::numeric 또는 * 1.0으로 캐스팅 필수

  COUNT(*)::numeric / 전체수   → 0.3  (정상)
  COUNT(*) * 1.0 / 전체수      → 0.3  (정상)
```

---

# GROUP BY 기본 ⭐️⭐️⭐️

```sql
-- 카테고리별 게시글 수
SELECT category, COUNT(*) AS cnt
FROM post
WHERE is_active = TRUE
GROUP BY category
ORDER BY cnt DESC;

-- 여러 컬럼 그룹핑 — (category, month) 쌍 기준
SELECT
    category,
    DATE_TRUNC('month', created_at AT TIME ZONE 'Asia/Seoul') AS month,
    COUNT(*) AS cnt
FROM post
GROUP BY category, month
ORDER BY month, cnt DESC;
```

```txt
GROUP BY 기준 = 식별자(PK), 출력용 컬럼은 덤

❌ 이름으로 그룹핑 — 동명이인 있으면 틀림
  GROUP BY u.name  → 같은 이름 두 사람이 하나로 합쳐짐

✅ id로 그룹핑, 이름은 출력용
  GROUP BY u.id, u.name  → id가 사람을 구분, name은 SELECT에 필요해서 같이 포함
```

---

# HAVING — 집계 결과 필터링 ⭐️⭐️⭐️

```sql
-- 게시글이 10개 이상인 카테고리만
SELECT category, COUNT(*) AS cnt
FROM post
GROUP BY category
HAVING COUNT(*) >= 10;

-- WHERE(집계 전) + HAVING(집계 후) 함께 쓰기
SELECT user_id, COUNT(*) AS post_count
FROM post
WHERE created_at >= '2025-01-01'   -- 집계 전 필터 (개별 행)
GROUP BY user_id
HAVING COUNT(*) >= 5;              -- 집계 후 필터 (그룹 결과)
```

```txt
WHERE vs HAVING:
  WHERE  → 집계 전 필터 (개별 행), 집계 함수 사용 불가
  HAVING → 집계 후 필터 (그룹 결과), 집계 함수 사용 가능

  ❌ WHERE COUNT(*) >= 5   → 에러
  ✅ HAVING COUNT(*) >= 5  → 정상
```

---

# FILTER — 조건부 집계 ⭐️⭐️⭐️ (PostgreSQL 전용)

```txt
집계 함수 안에 WHERE 조건을 붙이는 문법 — PostgreSQL 전용
같은 결과를 CASE WHEN보다 훨씬 읽기 쉽게 표현
```

```sql
-- PostgreSQL: FILTER (WHERE 조건)
SELECT
    TO_CHAR(created_at AT TIME ZONE 'Asia/Seoul', 'YYYY-MM') AS month,
    COUNT(*)                                                   AS total,
    COUNT(*) FILTER (WHERE status = 'approved')               AS approved,
    SUM(amount)                                               AS total_amount,
    SUM(amount) FILTER (WHERE status = 'approved')            AS approved_amount
FROM payment
GROUP BY month
ORDER BY month;
```

```sql
-- MySQL 등 다른 DB: CASE WHEN으로 동일 표현 (더 복잡)
SELECT
    DATE_FORMAT(created_at, '%Y-%m') AS month,
    COUNT(*) AS total,
    SUM(CASE WHEN status = 'approved' THEN 1 ELSE 0 END) AS approved,
    SUM(amount) AS total_amount,
    SUM(CASE WHEN status = 'approved' THEN amount ELSE 0 END) AS approved_amount
FROM payment
GROUP BY month;
```

---

# 날짜 기반 그룹핑 ⭐️⭐️⭐️

## 주요 날짜 함수

|함수|반환|예시|
|---|---|---|
|`DATE_TRUNC('month', ts)`|월 시작 timestamp|`2025-07-01 00:00:00+00`|
|`DATE_TRUNC('week', ts)`|주 시작 (월요일)|`2025-06-30 00:00:00+00`|
|`DATE_TRUNC('day', ts)`|일 시작|`2025-07-01 00:00:00+00`|
|`DATE(ts AT TIME ZONE 'Asia/Seoul')`|KST 날짜|`2025-07-01`|
|`TO_CHAR(ts, 'YYYY-MM')`|문자열 년월|`'2025-07'`|
|`EXTRACT(HOUR FROM ts)`|시간대(0-23)|`14`|

```sql
-- 월별 가입자 수 (KST 기준)
SELECT
    DATE_TRUNC('month', created_at AT TIME ZONE 'Asia/Seoul') AS month,
    COUNT(*) AS new_users
FROM "user"
GROUP BY month
ORDER BY month;

-- 요일별 활동량 (0=일요일, 1=월요일, ...)
SELECT
    EXTRACT(DOW FROM created_at AT TIME ZONE 'Asia/Seoul') AS day_of_week,
    COUNT(*) AS cnt
FROM post
GROUP BY day_of_week
ORDER BY day_of_week;
```

---

# generate_series — 빈 구간 채우기 ⭐️⭐️⭐️⭐️ (PostgreSQL 전용)

```txt
GROUP BY만 쓰면 데이터가 없는 날/달은 결과에서 사라짐
→ 차트에서 "7/3 데이터 없음" → 날짜가 빠져서 그래프가 이상하게 연결됨
→ generate_series로 날짜 목록을 먼저 만들고 LEFT JOIN해서 0으로 채움

([[NestJS_StatsBucket]] — 애플리케이션 레벨에서 버킷 생성 후 Map으로 채우는 패턴도 참고)
```

## 날짜 시리즈 생성

```sql
-- 2025-01-01 ~ 2025-01-07까지 하루씩
SELECT generate_series(
    '2025-01-01'::date,
    '2025-01-07'::date,
    INTERVAL '1 day'
) AS day;

-- 이번 달 전체 날짜 생성
SELECT generate_series(
    DATE_TRUNC('month', NOW()),
    DATE_TRUNC('month', NOW()) + INTERVAL '1 month' - INTERVAL '1 day',
    INTERVAL '1 day'
)::date AS day;
```

## 빈 날짜 채우기 패턴

```sql
-- 최근 30일 일별 가입자 수 (가입자 없는 날도 0으로)
WITH date_series AS (
    SELECT generate_series(
        NOW()::date - INTERVAL '29 days',
        NOW()::date,
        INTERVAL '1 day'
    )::date AS day
),
daily_signup AS (
    SELECT
        DATE(created_at AT TIME ZONE 'Asia/Seoul') AS day,
        COUNT(*) AS cnt
    FROM "user"
    WHERE created_at >= NOW() - INTERVAL '30 days'
    GROUP BY 1
)
SELECT
    ds.day,
    COALESCE(du.cnt, 0) AS cnt   -- 데이터 없으면 0
FROM date_series ds
LEFT JOIN daily_signup du ON du.day = ds.day
ORDER BY ds.day;
```

## 월별 빈 구간 채우기

```sql
WITH month_series AS (
    SELECT generate_series(
        DATE_TRUNC('month', '2025-01-01'::date),
        DATE_TRUNC('month', '2025-06-01'::date),
        INTERVAL '1 month'
    ) AS month
),
monthly_revenue AS (
    SELECT
        DATE_TRUNC('month', created_at AT TIME ZONE 'Asia/Seoul') AS month,
        SUM(amount)::numeric AS revenue
    FROM payment
    WHERE created_at >= '2025-01-01'
    GROUP BY 1
)
SELECT
    TO_CHAR(ms.month AT TIME ZONE 'Asia/Seoul', 'YYYY-MM') AS month,
    COALESCE(mr.revenue, 0) AS revenue
FROM month_series ms
LEFT JOIN monthly_revenue mr ON mr.month = ms.month
ORDER BY ms.month;
```

```txt
핵심 패턴:
  ① generate_series로 원하는 기간의 날짜/월 목록 CTE로 만들기
  ② 실제 데이터를 GROUP BY로 집계하는 CTE 만들기
  ③ 날짜 CTE LEFT JOIN 실제 데이터 CTE
  ④ COALESCE(값, 0)으로 NULL → 0

::date 캐스팅:
  generate_series는 TIMESTAMPTZ를 반환
  날짜(DATE)로 비교하려면 ::date로 캐스팅 필요

NestJS 애플리케이션에서:
  generate_series 방법: DB가 빈 구간을 채워서 줌 → 코드 간단
  버킷 방법(NestJS_StatsBucket): 앱에서 Map으로 채움 → DB 쿼리 단순, 앱에서 처리
  → 데이터가 많을수록 generate_series 방법이 DB에서 처리해서 더 효율적
```

---

# ROLLUP / GROUPING SETS — 소계·합계 ⭐️⭐️

```sql
-- ROLLUP: 계층적 소계 (category별, 전체 합계)
SELECT
    COALESCE(category, '전체') AS category,
    COUNT(*) AS cnt
FROM post
GROUP BY ROLLUP(category);
-- 결과: 각 카테고리별 + 마지막에 전체 합계 행 추가

-- GROUPING SETS: 여러 그룹 기준을 한 번에
SELECT category, user_id, COUNT(*) AS cnt
FROM post
GROUP BY GROUPING SETS (
    (category),          -- 카테고리별
    (user_id),           -- 유저별
    ()                   -- 전체 합계
);
```

---

# GROUP BY vs Window 함수 선택 기준 ⭐️⭐️⭐️

```txt
"그룹별 집계 결과만 필요" → GROUP BY (행 수가 줄어듦)
"각 행을 유지하면서 그룹 정보도 함께" → Window 함수 (원본 행 유지)

예: 유저별 게시글 수 → GROUP BY
예: 각 게시글에 "이 유저의 총 게시글 수"를 같이 보여주기 → Window 함수 (SUM OVER PARTITION BY)

([[PG_Window]] 참고)
```

---

# 한눈에

```txt
집계 함수:
  COUNT(*) → NULL 포함 / COUNT(컬럼) → NULL 제외 / COUNT(DISTINCT) → 중복 제거
  정수 나눗셈 시 ::numeric 또는 * 1.0 캐스팅 필수 (PostgreSQL)

GROUP BY:
  기준은 PK(식별자) — 이름/텍스트 단독 기준은 중복 위험
  SELECT에 있는 집계 아닌 컬럼은 GROUP BY에 전부 포함

HAVING:
  집계 후 필터 (WHERE의 집계 버전)
  WHERE에 집계 함수 사용 불가

FILTER (WHERE ...) ← PostgreSQL 전용:
  COUNT(*) FILTER (WHERE 조건) — CASE WHEN보다 읽기 쉬움

generate_series ← PostgreSQL 전용:
  generate_series(시작, 끝, 간격)으로 날짜/시간 목록 생성
  LEFT JOIN + COALESCE(값, 0)로 빈 구간을 0으로 채움
  NestJS_StatsBucket과 비교: generate_series=DB처리 / 버킷=앱처리

날짜 그룹핑:
  DATE_TRUNC('month', ts AT TIME ZONE 'Asia/Seoul') → KST 월 시작
  DATE(ts AT TIME ZONE 'Asia/Seoul')                → KST 날짜
  TO_CHAR(ts, 'YYYY-MM')                           → 문자열 년월
```