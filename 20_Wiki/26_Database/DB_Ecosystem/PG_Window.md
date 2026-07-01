---
aliases:
  - 윈도우 함수
  - PARTITION BY
  - ORDER BY
  - ROWS BETWEEN
  - DENSE_RANK
  - RANK
  - ROW_NUMBER
  - NTILE
  - LAG
  - LEAD
  - FIRST_VALUE
  - LAST_VALUE
  - 집계 윈도우 함수
tags:
  - SQL
  - PostgreSQL
related:
  - "[[00_DB_HomePage]]"
  - "[[PG_Aggregate]]"
  - "[[PG_DML]]"
  - "[[NestJS_StatsBucket]]"
---
# PG_Window — 윈도우 함수

> [!info] 
> 윈도우 함수 = 행을 그룹으로 묶어 집계하되, 원본 행을 유지하는 함수. 
> GROUP BY는 행이 줄어들지만 윈도우 함수는 각 행에 집계 값을 덧붙인다.

---

# GROUP BY vs 윈도우 함수 ⭐️⭐️⭐️⭐️

```sql
-- GROUP BY: 그룹별 집계 → 행이 줄어듦
SELECT user_id, COUNT(*) AS post_count
FROM post
GROUP BY user_id;
-- 유저 수만큼의 행

-- 윈도우 함수: 각 행에 그룹 집계를 붙임 → 행 수 유지
SELECT
    id,
    user_id,
    title,
    COUNT(*) OVER (PARTITION BY user_id) AS user_post_count  -- 이 유저의 총 게시글 수
FROM post;
-- post 행 수 그대로 + user_post_count 컬럼 추가
```

```txt
선택 기준:
  "유저별 게시글 수만 필요" → GROUP BY
  "각 게시글 옆에 이 유저의 총 게시글 수도 보여줘" → 윈도우 함수

윈도우 함수가 필요한 상황:
  각 행과 그 행이 속한 그룹의 집계를 같이 보고 싶을 때
  순위(1등, 2등)를 매기되 원본 데이터도 유지하고 싶을 때
  이전/다음 행의 값을 현재 행과 비교하고 싶을 때
```

---

# 기본 문법 ⭐️⭐️⭐️

```sql
함수() OVER (
    PARTITION BY 그룹기준컬럼    -- 어떻게 묶을지 (선택)
    ORDER BY 정렬컬럼            -- 어떻게 정렬할지 (선택, 순위/누적에 필요)
    ROWS BETWEEN ... AND ...    -- 프레임 범위 (선택)
)
```

```txt
PARTITION BY:
  GROUP BY처럼 그룹을 나눔, 없으면 전체가 하나의 그룹
  각 파티션 안에서 독립적으로 계산됨

ORDER BY:
  파티션 안에서 정렬 기준 — 순위 함수와 누적 집계에 필수
  윈도우 전체 ORDER BY와 별개

ROWS BETWEEN:
  프레임 범위 — 계산에 포함할 행 범위 지정 (누적, 이동 평균 등)
  UNBOUNDED PRECEDING ~ CURRENT ROW (기본) / N PRECEDING ~ N FOLLOWING
```

---

# 순위 함수 ⭐️⭐️⭐️⭐️

## ROW_NUMBER · RANK · DENSE_RANK

|함수|동점 처리|특징|
|---|---|---|
|`ROW_NUMBER()`|동점도 다른 번호 (순서대로)|항상 1,2,3,4...|
|`RANK()`|동점 = 같은 순위, 다음은 건너뜀|1,2,2,4 (3 건너뜀)|
|`DENSE_RANK()`|동점 = 같은 순위, 다음은 연속|1,2,2,3 (건너뜀 없음)|

```sql
SELECT
    id,
    user_id,
    view_count,
    ROW_NUMBER() OVER (ORDER BY view_count DESC)                        AS row_num,
    RANK()       OVER (ORDER BY view_count DESC)                        AS rank,
    DENSE_RANK() OVER (ORDER BY view_count DESC)                        AS dense_rank,
    -- 카테고리별 순위
    RANK()       OVER (PARTITION BY category ORDER BY view_count DESC)  AS category_rank
FROM post;
```

## 상위 N개 추출 — 순위 함수 활용 ⭐️⭐️⭐️

```sql
-- 카테고리별 조회수 상위 3개 게시글
WITH ranked AS (
    SELECT
        id,
        category,
        title,
        view_count,
        RANK() OVER (PARTITION BY category ORDER BY view_count DESC) AS rnk
    FROM post
)
SELECT id, category, title, view_count
FROM ranked
WHERE rnk <= 3;
```

```txt
왜 WHERE에서 바로 RANK()를 못 쓰는가:
  윈도우 함수는 SELECT 절에서 계산됨
  WHERE는 SELECT보다 먼저 실행 → 아직 rnk 값이 없음
  → CTE나 인라인 뷰로 한 번 감싸서 결과를 테이블처럼 만든 뒤 WHERE 적용

ROW_NUMBER vs RANK vs DENSE_RANK 선택:
  "동점도 다른 번호, 페이지네이션용" → ROW_NUMBER
  "실제 순위 (2등이 두 명이면 4등 존재)" → RANK
  "압축 순위 (2등이 두 명이면 바로 3등)" → DENSE_RANK
```

## NTILE — 균등 분할

```sql
-- 조회수 기준으로 4분위 나누기 (1=하위 25%, 4=상위 25%)
SELECT
    id,
    title,
    view_count,
    NTILE(4) OVER (ORDER BY view_count) AS quartile
FROM post;
```

---

# LAG / LEAD — 이전/다음 행 참조 ⭐️⭐️⭐️⭐️

```sql
LAG(컬럼, N, 기본값)  OVER (...)  -- N번째 이전 행
LEAD(컬럼, N, 기본값) OVER (...)  -- N번째 다음 행
```

```sql
-- 전월 대비 매출 증감
SELECT
    month,
    revenue,
    LAG(revenue) OVER (ORDER BY month)                         AS prev_revenue,
    revenue - LAG(revenue) OVER (ORDER BY month)              AS diff,
    ROUND(
        (revenue - LAG(revenue) OVER (ORDER BY month))
        / LAG(revenue) OVER (ORDER BY month) * 100, 1
    )                                                          AS growth_pct
FROM monthly_revenue
ORDER BY month;
```

```sql
-- 유저별 직전 로그인 시각 (파티션 적용)
SELECT
    user_id,
    logged_in_at,
    LAG(logged_in_at) OVER (PARTITION BY user_id ORDER BY logged_in_at) AS prev_login
FROM login_log;
```

```txt
LAG/LEAD 기본값:
  세 번째 인자로 기본값 지정 → 이전/다음 행이 없을 때(첫 행, 마지막 행) 사용
  LAG(revenue, 1, 0) → 이전 행이 없으면 0
  LAG(revenue, 1, NULL) → 기본값 (생략하면 NULL)

증감률 계산 시 주의:
  이전 달 매출이 0이면 나눗셈 에러 → NULLIF 사용
  LAG(revenue) OVER (...) = 0이면 → NULLIF(LAG_값, 0)로 0 → NULL 변환
  (0으로 나누면 에러, NULL로 나누면 NULL 반환 — 에러보단 나음)
```

---

# FIRST_VALUE / LAST_VALUE ⭐️⭐️

```sql
-- 카테고리별 첫 번째(최저가) 게시글 제목을 각 행에 붙이기
SELECT
    id,
    category,
    price,
    FIRST_VALUE(title) OVER (
        PARTITION BY category
        ORDER BY price ASC
        ROWS BETWEEN UNBOUNDED PRECEDING AND CURRENT ROW
    ) AS cheapest_in_category
FROM product;
```

```txt
LAST_VALUE 사용 시 주의:
  기본 프레임이 CURRENT ROW까지라서 "현재 행"이 마지막이 됨
  진짜 마지막 값을 보려면 프레임을 명시해야 함:
  LAST_VALUE(title) OVER (
      PARTITION BY category ORDER BY price
      ROWS BETWEEN UNBOUNDED PRECEDING AND UNBOUNDED FOLLOWING
  )
```

---

# 집계 윈도우 함수 — 누적/이동 계산 ⭐️⭐️⭐️⭐️

## 누적 합계 (Running Total)

```sql
SELECT
    created_at::date AS day,
    daily_revenue,
    SUM(daily_revenue) OVER (ORDER BY created_at::date) AS cumulative_revenue
FROM daily_stats;
-- 날짜가 지날수록 누적 합계가 쌓임
```

## 이동 평균 (Moving Average)

```sql
-- 7일 이동 평균
SELECT
    day,
    revenue,
    ROUND(
        AVG(revenue) OVER (
            ORDER BY day
            ROWS BETWEEN 6 PRECEDING AND CURRENT ROW  -- 현재 포함 7행
        )::numeric, 2
    ) AS moving_avg_7d
FROM daily_revenue;
```

## 파티션 내 비율

```sql
-- 카테고리별 매출 비중
SELECT
    category,
    revenue,
    ROUND(
        revenue::numeric / SUM(revenue) OVER (PARTITION BY category) * 100, 1
    ) AS share_pct
FROM product_sales;
```

---

# 프레임 범위 (ROWS vs RANGE) ⭐️⭐️

```txt
ROWS BETWEEN A AND B  → 물리적 행 위치 기준
RANGE BETWEEN A AND B → 값 기준 (같은 ORDER BY 값이 있으면 같이 포함)

기준값:
  UNBOUNDED PRECEDING → 파티션의 첫 행
  N PRECEDING         → 현재 행에서 N행 앞
  CURRENT ROW         → 현재 행
  N FOLLOWING         → 현재 행에서 N행 뒤
  UNBOUNDED FOLLOWING → 파티션의 마지막 행
```

|프레임 표현|의미|
|---|---|
|`ROWS BETWEEN UNBOUNDED PRECEDING AND CURRENT ROW`|처음부터 현재까지 (누적)|
|`ROWS BETWEEN 6 PRECEDING AND CURRENT ROW`|7행 이동 윈도우|
|`ROWS BETWEEN UNBOUNDED PRECEDING AND UNBOUNDED FOLLOWING`|파티션 전체|
|`ROWS BETWEEN 3 PRECEDING AND 3 FOLLOWING`|앞 3행 + 현재 + 뒤 3행|

---

# 실전 패턴 — 종합 ⭐️⭐️⭐️

```sql
-- 유저별 게시글: 순위 + 누적 조회수 + 이전 게시글 제목
SELECT
    u.nickname,
    p.title,
    p.view_count,
    p.created_at,
    RANK()           OVER (PARTITION BY p.user_id ORDER BY p.view_count DESC) AS rank_in_user,
    SUM(p.view_count) OVER (PARTITION BY p.user_id ORDER BY p.created_at)    AS cumulative_views,
    LAG(p.title)     OVER (PARTITION BY p.user_id ORDER BY p.created_at)     AS prev_post_title
FROM post p
JOIN "user" u ON u.id = p.user_id
WHERE p.is_active = TRUE;
```

---

# 한눈에

```txt
핵심 구분:
  GROUP BY → 행 수 줄어듦 (집계 결과만)
  윈도우   → 행 수 유지 (각 행에 집계값 덧붙임)

OVER() 구성:
  PARTITION BY → 그룹 나누기 (없으면 전체 하나의 그룹)
  ORDER BY     → 파티션 내 정렬 (순위/누적에 필수)
  ROWS BETWEEN → 계산 범위

순위 함수:
  ROW_NUMBER() → 동점도 다른 번호 (1,2,3,4)
  RANK()       → 동점 같은 번호, 다음 건너뜀 (1,2,2,4)
  DENSE_RANK() → 동점 같은 번호, 연속 (1,2,2,3)
  NTILE(N)     → N등분

이전/다음 행:
  LAG(col, N, default)  → N번째 이전 행 (없으면 default)
  LEAD(col, N, default) → N번째 다음 행

집계 윈도우:
  SUM OVER (ORDER BY col)                           → 누적 합계
  AVG OVER (ROWS BETWEEN 6 PRECEDING AND CURRENT ROW) → 7일 이동 평균
  SUM OVER (PARTITION BY col)                       → 파티션 합계 (비율 계산에)

상위 N개 추출:
  CTE에서 RANK() 계산 → 바깥 WHERE rnk <= N
  (WHERE에서 바로 윈도우 함수 쓰기 불가 — SELECT보다 WHERE가 먼저 실행됨)

집계와 비교 → [[PG_Aggregate]]
```