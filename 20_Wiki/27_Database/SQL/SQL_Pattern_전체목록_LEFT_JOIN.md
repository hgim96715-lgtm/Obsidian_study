---
aliases:
  - 전체목록
  - LEFT JOIN
  - UNION ALL
  - 0개 카테고리
tags:
  - SQL
related:
  - "[[00_SQL_HomePage]]"
  - "[[SQL_CTE]]"
  - "[[SQL_JOIN]]"
  - "[[SQL_Aggregate]]"
  - "[[SQL_CASE_WHEN]]"
  - "[[SQL_JOIN_Advanced]]"
---


# SQL_Pattern_전체목록_LEFT_JOIN — 빈 그룹도 포함

# 한 줄 요약

```
GROUP BY 만 하면 데이터 없는 카테고리가 결과에서 사라짐
→ 빈 카테고리도 0 으로 출력해야 할 때 두 가지 방법
  방법 1: UNION ALL — 카테고리 수 적고 고정일 때 (더 간단)
  방법 2: CTE + LEFT JOIN — 카테고리가 많거나 동적일 때
```

---

---

# 문제 상황 ⭐️

```sql
-- 급여 카테고리별 직원 수 구하기
-- Low Salary / Average Salary / High Salary 세 줄 모두 나와야 함

-- ❌ GROUP BY 만 쓰면
WITH account_cate AS (
  SELECT
    CASE
      WHEN income < 20000  THEN 'Low Salary'
      WHEN income > 50000  THEN 'High Salary'
      ELSE 'Average Salary'
    END AS category
  FROM Accounts
)
SELECT category, COUNT(*) AS accounts_count
FROM account_cate
GROUP BY category;
```

```
문제:
  High Salary 에 해당하는 직원이 없으면
  → High Salary 행 자체가 결과에 없음
  → 항상 세 줄이 나와야 하는 요구사항 위반

GROUP BY 는 데이터 있는 그룹만 집계
→ 없는 그룹은 아예 안 나옴
```

---

---

# 방법 1 — UNION ALL ⭐️ (더 간단)

```sql
SELECT 'Low Salary'     AS category, COUNT(*) AS accounts_count
FROM Accounts
WHERE income < 20000

UNION ALL

SELECT 'Average Salary' AS category, COUNT(*) AS accounts_count
FROM Accounts
WHERE income BETWEEN 20000 AND 50000

UNION ALL

SELECT 'High Salary'    AS category, COUNT(*) AS accounts_count
FROM Accounts
WHERE income > 50000;
```

```
핵심:
  COUNT(*) 는 조건에 맞는 행이 0개여도 0 을 반환
  → WHERE 에 해당하는 데이터가 없어도 해당 SELECT 가 실행됨
  → 세 줄이 무조건 나옴

UNION ALL 이 좋은 경우:
  카테고리 수가 적고 고정 (3~5개)
  카테고리 경계가 명확한 범위 조건
  코드가 훨씬 짧고 직관적

⚠️ UNION vs UNION ALL:
  UNION      중복 제거 (느림)
  UNION ALL  중복 허용 (빠름) ← 카테고리가 겹치지 않으므로 UNION ALL 사용
```

---

---

# 방법 2 — CTE + LEFT JOIN

```sql
-- 1. 고정 카테고리 3개를 기준 테이블로 만들기
WITH categories AS (
  SELECT 'Low Salary'     AS category
  UNION ALL
  SELECT 'Average Salary'
  UNION ALL
  SELECT 'High Salary'
),

-- 2. 각 계정의 카테고리 구하기
account_cate AS (
  SELECT
    CASE
      WHEN income < 20000  THEN 'Low Salary'
      WHEN income > 50000  THEN 'High Salary'
      ELSE 'Average Salary'
    END AS category
  FROM Accounts
),

-- 3. 카테고리별 개수 집계
account_count AS (
  SELECT category, COUNT(*) AS accounts_count
  FROM account_cate
  GROUP BY category
)

-- 4. 기준 테이블에 LEFT JOIN → 없는 카테고리는 NULL → COALESCE 로 0
SELECT
  c.category,
  COALESCE(ac.accounts_count, 0) AS accounts_count
FROM categories c
LEFT JOIN account_count ac
  ON c.category = ac.category;
```

```
핵심:
  categories  → 있어야 할 카테고리 전부 (기준 테이블)
  LEFT JOIN   → 기준 테이블 기준 / 없으면 NULL
  COALESCE    → NULL → 0 으로 변환

CTE + LEFT JOIN 이 좋은 경우:
  카테고리가 많거나 DB 에서 동적으로 가져와야 할 때
  집계 로직이 복잡해서 단계별로 나눠야 할 때
  다른 테이블의 값을 기준 목록으로 쓸 때
```

---

---

# 두 방법 비교

| |UNION ALL|CTE + LEFT JOIN|
|---|---|---|
|코드 길이|짧음|김|
|직관성|높음|낮음|
|카테고리 수|고정 소수|많거나 동적|
|빈 카테고리 처리|COUNT(*) = 0 자동|COALESCE 필요|
|적합한 경우|급여 구간 / 연령대 등|지역별 / 부서별 등|

---

---

# 자주 하는 실수 ⭐️

## COALESCE 를 엉뚱하게 쓴 경우

```sql
-- ❌ category 는 문자열이라 COALESCE(category, 0) 은 count 와 무관
SELECT category, COALESCE(category, 0) AS accounts_count
FROM account_cate
GROUP BY category;

-- ✅ COUNT 결과에 COALESCE 적용
SELECT c.category, COALESCE(ac.accounts_count, 0) AS accounts_count
FROM categories c
LEFT JOIN account_count ac ON c.category = ac.category;
```

## 조건 경계 주의 ⭐️

```
문제: "less than 20000" → income < 20000
      "20000 이하"       → income <= 20000

경계가 20000 일 때:
  income < 20000   20000 미만 (20000 은 Average)
  income <= 20000  20000 이하 (20000 도 Low)

문제 조건 정확히 읽기
BETWEEN 20000 AND 50000 = 20000 이상 50000 이하 (양 끝 포함)
```

---

---

# 한눈에

```
빈 카테고리도 출력해야 할 때:

방법 1 (UNION ALL):
  카테고리가 고정 소수개 → 이게 더 간단
  COUNT(*) 은 0개여도 0 반환 → 자동으로 해결

방법 2 (CTE + LEFT JOIN):
  categories  고정 목록 (UNION ALL 로 생성)
  LEFT JOIN   기준 목록 기준 조인
  COALESCE    NULL → 0

GROUP BY 만 쓰면 빈 그룹 사라짐 → 두 방법 중 하나 선택
```