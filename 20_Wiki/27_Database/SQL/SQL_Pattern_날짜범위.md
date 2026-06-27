---
aliases:
  - 날짜범위
  - date range
  - 월별 필터
  - ::DATE
  - BETWEEN
  - JOIN 에서 날짜
tags:
  - SQL
related:
  - "[[00_SQL_HomePage]]"
  - "[[SQL_WHERE]]"
  - "[[SQL_Date_Functions]]"
---

# SQL_Pattern_날짜범위 — 날짜 범위 필터

---

---

# ① 기본 원칙 ⭐️

```sql
-- ❌ BETWEEN (DATETIME 타입 시 마지막 시간 누락)
WHERE order_date BETWEEN '2020-02-01' AND '2020-02-29'
-- 2020-02-29 00:00:00 까지만 포함
-- 2020-02-29 10:30:00 같은 데이터 누락!

-- ✅ >= AND < 권장
WHERE order_date >= '2020-02-01'
  AND order_date < '2020-03-01'
-- 2020-02-01 00:00:00 이상
-- 2020-03-01 00:00:00 미만 = 2월 전체 완벽 포함
```

```txt
규칙:
  시작 >= 해당 월 1일
  종료 <  다음 달 1일

  왜 다음 달 첫날 미만인가:
    말일 계산 실수 방지 (2월 28일? 29일?)
    DATETIME / TIMESTAMP 타입도 완벽 처리
    어느 DB든 일관되게 동작
```

---

---

# ② 월별 필터 패턴

```sql
-- 2020년 2월
WHERE order_date >= '2020-02-01' AND order_date < '2020-03-01'

-- 2020년 전체
WHERE order_date >= '2020-01-01' AND order_date < '2021-01-01'

-- 특정 분기 (Q1)
WHERE order_date >= '2020-01-01' AND order_date < '2020-04-01'

-- 최근 7일 (PostgreSQL)
WHERE order_date >= CURRENT_DATE - INTERVAL '7 days'
  AND order_date < CURRENT_DATE + INTERVAL '1 day'

-- 최근 30일
WHERE order_date >= NOW() - INTERVAL '30 days'
```

---

---

# ③ JOIN 에서 날짜 조건 ⭐️

```sql
-- WHERE 대신 ON 에 날짜 조건
SELECT p.product_name, SUM(o.unit) AS unit
FROM Products p
JOIN Orders o
    ON p.product_id = o.product_id
   AND o.order_date >= '2020-02-01'      -- ON 에 포함
   AND o.order_date < '2020-03-01'
GROUP BY p.product_name
HAVING SUM(o.unit) >= 100;
```

```txt
ON vs WHERE 날짜 조건:
  LEFT JOIN 시 ON 에 넣으면 → 조건 맞는 것만 연결 / 없으면 NULL
  LEFT JOIN 시 WHERE 에 넣으면 → NULL 행도 제거 → INNER JOIN 효과

  날짜 범위 + LEFT JOIN → ON 에 넣기
```

---

---

# ④ ::DATE 캐스팅 (PostgreSQL) ⭐️

```sql
-- TIMESTAMP 를 DATE 로 비교
WHERE created_at::DATE = '2020-02-15'
-- 시간 무시하고 날짜만 비교

-- >= AND < 와 동일 효과
WHERE created_at >= '2020-02-15'
  AND created_at < '2020-02-16'

-- INTERVAL 활용
WHERE created_at >= NOW() - INTERVAL '7 days'
WHERE created_at >= DATE_TRUNC('month', NOW())  -- 이번 달 1일부터
```

---

---

# 한눈에

|기간|패턴|
|---|---|
|특정 월|`>= '2020-02-01' AND < '2020-03-01'`|
|특정 연도|`>= '2020-01-01' AND < '2021-01-01'`|
|최근 N일|`>= NOW() - INTERVAL 'N days'`|
|날짜만 비교|`created_at::DATE = '2020-02-15'`|
|이번 달|`>= DATE_TRUNC('month', NOW())`|