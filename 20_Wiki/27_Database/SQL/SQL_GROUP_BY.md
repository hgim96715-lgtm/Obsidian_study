---
aliases:
  - GROUP BY
  - HAVING
  - GROUP BY 기준
tags:
  - SQL
related:
  - "[[00_SQL_HomePage]]"
  - "[[SQL_Aggregate]]"
  - "[[SQL_CASE_WHEN]]"
  - "[[SQL_CTE]]"
---

# SQL_GROUP_BY — GROUP BY & HAVING

## 한 줄 요약

```txt
GROUP BY  = 특정 컬럼 기준으로 묶어서 집계
HAVING    = 집계 결과를 필터링 (WHERE 의 집계 버전)
```

---

---

# ① GROUP BY 기본

```sql
SELECT 컬럼, 집계함수()
FROM 테이블
GROUP BY 컬럼;
```

```sql
-- 국가별 거래 건수
SELECT country, COUNT(*) AS trans_count
FROM Transactions
GROUP BY country;

-- 여러 컬럼으로 그룹핑
SELECT country, state, COUNT(*) AS cnt
FROM Transactions
GROUP BY country, state;
--       ↑ (country, state) 쌍 기준으로 묶임
```

```txt
GROUP BY A, B 의 의미:
  (A, B) 조합이 같은 행끼리 묶임
  country='KR', state='approved'  → 1그룹
  country='KR', state='declined'  → 다른 그룹
```

---

---

# ② GROUP BY + 표현식 ⭐️

```sql
-- 날짜를 연-월 형식으로 변환 후 그룹핑
-- PostgreSQL
SELECT
    TO_CHAR(trans_date, 'YYYY-MM') AS month,
    country,
    COUNT(*) AS trans_count
FROM Transactions
GROUP BY TO_CHAR(trans_date, 'YYYY-MM'), country;
--       ↑ SELECT 의 표현식과 동일하게

-- MySQL
SELECT
    DATE_FORMAT(trans_date, '%Y-%m') AS month,
    country,
    COUNT(*) AS trans_count
FROM Transactions
GROUP BY DATE_FORMAT(trans_date, '%Y-%m'), country;
```

```txt
GROUP BY 에 별칭(AS) 사용:
  PostgreSQL: GROUP BY month  ← 별칭 사용 가능
  MySQL:      GROUP BY month  ← 별칭 사용 가능
  표준 SQL:   GROUP BY TO_CHAR(...) ← 표현식 그대로

  안전하게: 표현식을 그대로 쓰는 것 권장
```

---

---

# ③ HAVING — 집계 결과 필터링 ⭐️

```sql
-- WHERE 는 집계 전 / HAVING 은 집계 후
SELECT country, COUNT(*) AS cnt
FROM Transactions
GROUP BY country
HAVING COUNT(*) >= 10;   -- 10건 이상인 국가만
```

```txt
WHERE vs HAVING:
  WHERE   집계 전 필터 (개별 행)
  HAVING  집계 후 필터 (그룹 결과)

  ❌ WHERE COUNT(*) >= 10  → 에러 (집계함수 사용 불가)
  ✅ HAVING COUNT(*) >= 10 → 정상
```

---

---

# ④ 실행 순서

```txt
FROM → WHERE → GROUP BY → HAVING → SELECT → ORDER BY

1. FROM      테이블 읽기
2. WHERE     행 필터링 (집계 전)
3. GROUP BY  그룹으로 묶기
4. HAVING    그룹 필터링 (집계 후)
5. SELECT    컬럼 선택
6. ORDER BY  정렬
```

---

---

# ⑤ 실전 패턴 — 월별 + 조건부 집계

```sql
-- PostgreSQL
SELECT
    TO_CHAR(trans_date, 'YYYY-MM')                            AS month,
    country,
    COUNT(*)                                                   AS trans_count,
    COUNT(*) FILTER (WHERE state = 'approved')                AS approved_count,
    SUM(amount)                                               AS trans_total,
    SUM(amount) FILTER (WHERE state = 'approved')             AS approved_total
FROM Transactions
GROUP BY TO_CHAR(trans_date, 'YYYY-MM'), country;

-- MySQL
SELECT
    DATE_FORMAT(trans_date, '%Y-%m')                          AS month,
    country,
    COUNT(*)                                                   AS trans_count,
    SUM(CASE WHEN state = 'approved' THEN 1 ELSE 0 END)       AS approved_count,
    SUM(amount)                                               AS trans_total,
    SUM(CASE WHEN state = 'approved' THEN amount ELSE 0 END)  AS approved_total
FROM Transactions
GROUP BY DATE_FORMAT(trans_date, '%Y-%m'), country;
```

---
---
# GROUP BY vs Window 함수 — 언제 뭘 쓰나 ⭐️

```txt
같은 결과를 두 가지 방식으로 낼 수 있을 때
더 자연스러운 방법을 선택하는 기준
```

## 문제: 상품별 주문 총량 (100 이상만)

```sql
-- 방법 1: Window 함수 + CTE
WITH sum_po AS (
    SELECT
        p.product_name,
        SUM(o.unit) OVER(PARTITION BY p.product_name) AS sum_unit
    FROM Products p
    JOIN Orders o ON p.product_id = o.product_id
       AND o.order_date >= '2020-02-01'
       AND o.order_date < '2020-03-01'
)
SELECT DISTINCT product_name, sum_unit AS unit
FROM sum_po
WHERE sum_unit >= 100;

-- 방법 2: GROUP BY + HAVING (더 자연스러움) ⭐️
SELECT
    p.product_name,
    SUM(o.unit) AS unit
FROM Products p
JOIN Orders o ON p.product_id = o.product_id
WHERE o.order_date >= '2020-02-01'
  AND o.order_date < '2020-03-01'
GROUP BY p.product_name
HAVING SUM(o.unit) >= 100;
```

```txt
비교:
  Window 방식:
    SUM() OVER(PARTITION BY) → 각 행에 그룹 합계 붙이기
    DISTINCT 필요 (중복 제거)
    코드 복잡 / CTE 필요

  GROUP BY 방식:
    SUM() + GROUP BY → 그룹별 집계 → 행 하나씩
    HAVING 으로 필터
    코드 간결 / 더 직관적

선택 기준 ⭐️:
  "각 행을 유지하면서 그룹 정보도 함께 보고 싶다"
    → Window 함수 (원본 행 유지)

  "그룹별 집계 결과만 필요하다"
    → GROUP BY (행 수가 줄어듦)

  상품별 총량 → GROUP BY 가 더 자연스러움
  개인 판매량 + 전체 합계 비교 → Window 함수
```

---
---
# GROUP BY 기준은 출력값이 아니라 식별자 ⭐️

## 흔히 하는 실수

```sql
-- ❌ 이름으로 그룹핑 — 동명이인 있으면 잘못된 결과
SELECT
    u.name,
    COALESCE(SUM(r.distance), 0) AS travelled_distance
FROM Users u
LEFT JOIN Rides r ON u.id = r.user_id
GROUP BY u.name          -- ← 이름이 같은 두 사람이 하나로 합쳐짐
ORDER BY travelled_distance DESC, u.name ASC;
```

## 올바른 코드

```sql
-- ✅ id 로 그룹핑 — 각 사용자를 정확히 구분
SELECT
    u.name,
    COALESCE(SUM(r.distance), 0) AS travelled_distance
FROM Users u
LEFT JOIN Rides r ON u.id = r.user_id
GROUP BY u.id, u.name    -- ← id 로 사람을 구분, name 은 출력용
ORDER BY travelled_distance DESC, u.name ASC;
```

```txt
왜 GROUP BY u.id, u.name 인가:
  u.id    = 사용자를 유일하게 식별 (그룹 기준)
  u.name  = 화면에 보여줄 값 → SELECT 에 있으면 GROUP BY 에도 포함

핵심 사고방식 ⭐️:
  ❌ "출력에 name 이 필요함 → name 으로 묶어야 함"
  ✅ "화면엔 name 보여줄게, 계산 기준은 id 로"

  출력 컬럼 ≠ 그룹 기준
  name = 보여줄 값
  id   = 사용자를 구분하는 값 (기본키)

동명이인 예시:
  id=1, name='홍길동'  거리 100
  id=2, name='홍길동'  거리 200

  GROUP BY name   → 홍길동 한 명으로 합산 → 300 (잘못됨!)
  GROUP BY id     → 홍길동 두 명 각각 100 / 200 (정확)

적용 범칙:
  JOIN 후 GROUP BY 할 때 → 항상 PK(기본키) 기준으로
  이름 / 제목 / 카테고리 단독으로 GROUP BY → 중복 주의
```
