---
aliases:
  - CASE WHEN
  - 조건부 집계
  - FILTER
  - 부호전환패턴
tags:
  - SQL
related:
  - "[[00_SQL_HomePage]]"
  - "[[SQL_Aggregate]]"
  - "[[SQL_GROUP_BY]]"
  - "[[PG_Specific]]"
  - "[[SQL_Window_Functions]]"
---
# SQL_CASE_WHEN — 조건 분기

```txt
SQL 의 IF-ELSE
조건에 따라 다른 값 반환 / 조건부 집계에 핵심
```

---

---


# 기본 문법

```sql
CASE
    WHEN 조건1 THEN 값1
    WHEN 조건2 THEN 값2
    ELSE 기본값          -- 생략 시 NULL 반환
END
```

```sql
-- 상태 코드 → 한글로 변환
SELECT
    id,
    CASE
        WHEN state = 'approved' THEN '승인'
        WHEN state = 'declined' THEN '거부'
        ELSE '기타'
    END AS state_kr
FROM Transactions;
```

```txt
동작 순서:
  위에서 아래로 조건 순서대로 평가
  처음으로 True 인 WHEN 에서 멈추고 THEN 값 반환
  모두 False 면 ELSE 반환 (없으면 NULL)

ELSE 생략 시:
  어떤 조건도 안 맞으면 NULL 반환
  SUM / AVG 에서 NULL 은 무시됨 → 의도와 다를 수 있음
  → 명시적으로 ELSE NULL 또는 ELSE 0 쓰는 게 더 명확
```

---

---

# 어디서 쓰나 ⭐️

```txt
CASE WHEN 은 값을 반환하는 표현식
값이 들어갈 수 있는 곳이면 어디든 사용 가능
```

## SELECT — 컬럼 값 변환

```sql
SELECT
    id,
    CASE
        WHEN score >= 90 THEN 'A'
        WHEN score >= 80 THEN 'B'
        WHEN score >= 70 THEN 'C'
        ELSE 'F'
    END AS grade
FROM Students;
```

## ORDER BY — 조건부 정렬

```sql
-- VIP 회원을 먼저, 나머지는 이름 순
SELECT name, grade
FROM Members
ORDER BY
    CASE WHEN grade = 'VIP' THEN 0 ELSE 1 END,
    name;
```

## GROUP BY — 그룹 기준으로 사용

```sql
-- 점수를 등급 그룹으로 묶기
SELECT
    CASE
        WHEN score >= 90 THEN 'A'
        WHEN score >= 80 THEN 'B'
        ELSE 'C'
    END AS grade,
    COUNT(*) AS cnt
FROM Students
GROUP BY
    CASE
        WHEN score >= 90 THEN 'A'
        WHEN score >= 80 THEN 'B'
        ELSE 'C'
    END;
```

## SUM / COUNT 안 — 조건부 집계 ⭐️

```sql
-- 가장 자주 쓰는 패턴
SUM(CASE WHEN state = 'approved' THEN 1     ELSE 0 END) AS approved_count
SUM(CASE WHEN state = 'approved' THEN amount ELSE 0 END) AS approved_total
```

---

---

# 조건부 집계 ⭐️

```txt
일반 집계: 전체에 대해 COUNT / SUM
조건부 집계: 조건을 만족하는 행에 대해서만 COUNT / SUM

핵심 공식:
  건수 세기  → SUM(CASE WHEN 조건 THEN 1     ELSE 0 END)
  금액 합산  → SUM(CASE WHEN 조건 THEN 컬럼  ELSE 0 END)
```

```sql
SELECT
    COUNT(*)                                                    AS total_count,
    SUM(CASE WHEN state = 'approved' THEN 1     ELSE 0 END)    AS approved_count,
    SUM(amount)                                                 AS total_amount,
    SUM(CASE WHEN state = 'approved' THEN amount ELSE 0 END)   AS approved_amount
FROM Transactions;
```

## 왜 COUNT 대신 SUM(CASE WHEN 1 ELSE 0) 인가

```sql
-- COUNT 로 조건부 세기
COUNT(CASE WHEN state = 'approved' THEN 1 END)
-- ELSE 없으면 NULL 반환 → COUNT 는 NULL 제외 → 조건 충족 행만 셈

-- SUM 으로 조건부 세기 (더 직관적)
SUM(CASE WHEN state = 'approved' THEN 1 ELSE 0 END)
-- 조건 맞으면 1, 아니면 0 → SUM = 조건 충족 행 수
```

```txt
둘 다 동일한 결과
SUM(CASE WHEN ... THEN 1 ELSE 0 END) 가 더 명시적이라 많이 쓰임
COUNT(CASE WHEN ... THEN 1 END) 도 표준 SQL 패턴
```

---

---

# 부호 전환 패턴 ⭐️

```txt
Buy(매수) 는 돈이 나가고 Sell(매도) 는 돈이 들어오는 계산
→ CASE WHEN 안에서 - 부호를 붙이면 SUM 이 자동으로 뺄셈
```

```sql
-- 주식별 순수익 계산
SELECT
    stock_name,
    SUM(
        CASE
            WHEN operation = 'Buy'  THEN -price   -- 매수: 돈 나감 (-)
            WHEN operation = 'Sell' THEN  price   -- 매도: 돈 들어옴 (+)
        END
    ) AS capital_gain_loss
FROM Stocks
GROUP BY stock_name;
```

```txt
핵심 인사이트:
  CASE WHEN 안에서 값을 -(음수)로 반환하면
  SUM 이 자동으로 뺄셈 역할

다른 접근 방법들과 비교:

  ① SUM(sell) - SUM(buy)
    → Buy / Sell 행을 따로 집계 후 빼야 함
    → 서브쿼리 또는 셀프조인 필요 → 복잡

  ② SUM(price) OVER (PARTITION BY stock_name)
    → Buy/Sell 부호 구분 없이 전체 합산만 됨 ❌
    → 결과를 1행으로 줄이지 않고 행을 유지
    → 이 문제엔 부적합

  ③ CASE WHEN 부호 전환 ✅
    → 한 번의 GROUP BY 로 해결
    → 가장 간결하고 직관적
```

## 언제 GROUP BY vs 윈도우 함수

```txt
결과를 그룹별 1행으로 줄이고 싶다  → GROUP BY + 집계 함수
행을 유지하면서 그룹 계산을 붙이고 싶다  → 윈도우 함수

주식별 최종 손익 = 1행으로 줄여야 함  → GROUP BY ✅
```

---

---

# AVG 에서 주의 ⚠️

```sql
-- ❌ ELSE 0 → 0 이 평균에 포함되어 값이 낮아짐
AVG(CASE WHEN state = 'approved' THEN amount ELSE 0 END)

-- ✅ ELSE NULL 또는 ELSE 생략 → NULL 은 AVG 에서 자동 제외
AVG(CASE WHEN state = 'approved' THEN amount END)
AVG(CASE WHEN state = 'approved' THEN amount ELSE NULL END)
```

```txt
SUM 에서는  ELSE 0    (0 을 더해도 합계에 영향 없음)
AVG 에서는  ELSE NULL (0 을 포함하면 평균이 낮아짐 ⚠️)

예시:
  approved 금액: 100, 200, 300  → 실제 평균 200
  ELSE 0 로 10행 중 3행만 approved:
    (100 + 200 + 300 + 0×7) / 10 = 60  ← 잘못된 값
  ELSE NULL:
    (100 + 200 + 300) / 3 = 200  ← 올바른 값
```

---

---

# PostgreSQL — FILTER(WHERE) ⭐️

```sql
-- CASE WHEN 방식 (모든 DB)
SUM(CASE WHEN state = 'approved' THEN 1      ELSE 0 END) AS approved_count
SUM(CASE WHEN state = 'approved' THEN amount ELSE 0 END) AS approved_total

-- FILTER 방식 (PostgreSQL 전용 / 더 간결)
COUNT(*) FILTER (WHERE state = 'approved')               AS approved_count
SUM(amount) FILTER (WHERE state = 'approved')            AS approved_total
```

```sql
-- FILTER 로 빼기도 가능
SUM(timestamp) FILTER (WHERE activity_type = 'end')
-
SUM(timestamp) FILTER (WHERE activity_type = 'start') AS process_time
```

|방식|사용 DB|특징|
|---|---|---|
|`SUM(CASE WHEN ...)`|모든 DB|표준 SQL / 범용|
|`FILTER(WHERE)`|PostgreSQL 전용|간결 / 가독성 좋음|

```bash
MySQL 쓰면 → SUM(CASE WHEN)
PostgreSQL 쓰면 → FILTER(WHERE) 권장
# [[PG_Specific]] 참고 
```

---

---

# 실전 패턴

## 월별 조건부 집계

```sql
-- PostgreSQL
SELECT
    TO_CHAR(trans_date, 'YYYY-MM')                          AS month,
    country,
    COUNT(*)                                                 AS trans_count,
    COUNT(*)      FILTER (WHERE state = 'approved')         AS approved_count,
    SUM(amount)                                             AS trans_total,
    SUM(amount)   FILTER (WHERE state = 'approved')         AS approved_total
FROM Transactions
GROUP BY TO_CHAR(trans_date, 'YYYY-MM'), country;

-- MySQL
SELECT
    DATE_FORMAT(trans_date, '%Y-%m')                                       AS month,
    country,
    COUNT(*)                                                                AS trans_count,
    SUM(CASE WHEN state = 'approved' THEN 1      ELSE 0 END)               AS approved_count,
    SUM(amount)                                                             AS trans_total,
    SUM(CASE WHEN state = 'approved' THEN amount ELSE 0 END)               AS approved_total
FROM Transactions
GROUP BY DATE_FORMAT(trans_date, '%Y-%m'), country;
```

## 비율 계산

```sql
-- 승인율 = 승인건수 / 전체건수 * 100
SELECT
    ROUND(
        COUNT(*) FILTER (WHERE state = 'approved') * 100.0
        / COUNT(*),
        2
    ) AS approval_rate
FROM Transactions;
```

```txt
정수 / 정수 = 정수 (소수점 버림) ⚠️
→ * 100.0 또는 CAST(... AS FLOAT) 로 실수 변환 필요

PostgreSQL: * 100.0 또는 ::float
MySQL:      * 100.0 또는 CAST(... AS DECIMAL)
```

## 등급 분류 후 통계

```sql
-- 점수 등급별 인원 수 + 평균
SELECT
    CASE
        WHEN score >= 90 THEN 'A'
        WHEN score >= 80 THEN 'B'
        WHEN score >= 70 THEN 'C'
        ELSE 'F'
    END                 AS grade,
    COUNT(*)            AS cnt,
    AVG(score)          AS avg_score   -- ELSE 0 쓰지 않아도 됨 (그룹 자체가 조건)
FROM Students
GROUP BY
    CASE
        WHEN score >= 90 THEN 'A'
        WHEN score >= 80 THEN 'B'
        WHEN score >= 70 THEN 'C'
        ELSE 'F'
    END;
```

---

---

# 한눈에 정리

## 목적별 공식

|목적|공식|
|---|---|
|건수 세기|`SUM(CASE WHEN 조건 THEN 1 ELSE 0 END)`|
|금액 합산|`SUM(CASE WHEN 조건 THEN 컬럼 ELSE 0 END)`|
|조건 평균|`AVG(CASE WHEN 조건 THEN 컬럼 END)` — ELSE 생략|
|부호 전환|`SUM(CASE WHEN Buy THEN -price WHEN Sell THEN price END)`|
|PostgreSQL|`COUNT(*) FILTER (WHERE 조건)`|

## ELSE 선택 기준

|집계 함수|ELSE|이유|
|---|---|---|
|`SUM`|`ELSE 0`|0 더해도 합계 불변|
|`COUNT` (SUM 방식)|`ELSE 0`|동일|
|`AVG`|`ELSE NULL` 또는 생략|0 포함 시 평균 낮아짐 ⚠️|

## 자주 하는 실수

```sql
-- ❌ AVG 에 ELSE 0
AVG(CASE WHEN state = 'approved' THEN amount ELSE 0 END)
-- → 0 이 평균에 포함되어 낮아짐

-- ❌ 비율 계산에서 정수 나누기
COUNT(*) FILTER (WHERE state = 'approved') / COUNT(*)
-- → 정수/정수 = 0 또는 1 (소수점 버림)

-- ❌ WHEN 순서 잘못
CASE
    WHEN score >= 70 THEN 'C'   -- 90점도 여기서 걸림
    WHEN score >= 80 THEN 'B'
    WHEN score >= 90 THEN 'A'   -- 절대 도달 안 함
END
-- → 범위 조건은 큰 값부터 작성
```