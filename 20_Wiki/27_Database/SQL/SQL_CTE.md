---
aliases:
  - CTE
  - WITH
  - 공통 테이블 표현식
  - WITH RECURSIVE
  - 재귀CTE
  - VIEW
  - 임시테이블
tags:
  - SQL
related:
  - "[[00_SQL_HomePage]]"
  - "[[SQL_Window_Functions]]"
  - "[[SQL_Subquery]]"
  - "[[SQL_JOIN]]"
---
# SQL_CTE — CTE (공통 테이블 표현식)

# 한 줄 요약

```txt
WITH 절로 임시 테이블(이름 있는 서브쿼리)을 만드는 것
복잡한 쿼리를 단계별로 쪼개서 읽기 쉽게 만들기
```

---

---

# CTE 기본 문법

```sql
WITH cte이름 AS (
    SELECT ...
    FROM ...
)
SELECT *
FROM cte이름;
```

```sql
-- 평균 이상인 사람만 조회
WITH avg_salary AS (
    SELECT AVG(salary) AS avg_sal
    FROM employees
)
SELECT name, salary
FROM employees, avg_salary    -- ⚠️ 아래에서 이 부분 설명
WHERE salary > avg_sal;
```

## ⚠️ FROM employees, avg_salary — 콤마로 이어붙인 이유

```txt
FROM a, b 는 콤마로 두 테이블을 나열하는 옛 스타일의 JOIN 표기법
(명시적 CROSS JOIN 과 같은 뜻 — 모든 조합을 다 만듦)

근데 avg_salary CTE 는 결과가 "딱 1행" (평균값 하나) 뿐이라서
employees 의 모든 행과 그 1행을 조합해도 → 그냥 모든 행에 평균값 하나가 똑같이 붙는 것뿐
(1행과의 CROSS JOIN = 단순히 그 값을 모든 행에 끼워넣는 것과 같음)

→ 결과 자체는 맞지만, 요즘은 이렇게 콤마로 쓰는 것보다
  명시적으로 CROSS JOIN 이라고 쓰거나, 애초에 스칼라 서브쿼리로 충분한 경우가 많음
```

```sql
-- 사실 이 예시는 CTE 가 꼭 필요하진 않음 — 스칼라 서브쿼리로도 충분
SELECT name, salary
FROM employees
WHERE salary > (SELECT AVG(salary) FROM employees);
-- (자세한 스칼라 서브쿼리는 [[SQL_SubQuery]] 참고)

-- CROSS JOIN 으로 명시하면 (콤마 스타일 대신)
SELECT name, salary
FROM employees
CROSS JOIN avg_salary
WHERE salary > avg_sal;
```

```txt
→ CTE 의 진짜 가치는 이렇게 "값 하나" 를 쓸 때가 아니라
  여러 단계로 나누거나, 윈도우 함수 결과를 다시 필터해야 할 때 드러남 (아래에서 계속)
```

---

---

# CTE vs 서브쿼리 ⭐️

```sql
-- 서브쿼리 방식 (중첩될수록 읽기 어려움)
SELECT name
FROM employees
WHERE salary > (SELECT AVG(salary) FROM employees);

-- CTE 방식 (이름이 붙어서 읽기 쉬움)
WITH avg_sal AS (
    SELECT AVG(salary) AS avg_val FROM employees
)
SELECT name
FROM employees
WHERE salary > (SELECT avg_val FROM avg_sal);
```

```txt
CTE 를 쓰는 이유:
  복잡한 쿼리를 단계별로 나눔 → 읽기 쉬움 (각 단계에 "이름" 이 붙어서 의도가 드러남)
  같은 서브쿼리를 여러 번 쓸 때 한 번만 정의
  윈도우 함수 결과를 WHERE 로 필터할 때 필수 (바로 아래)
```

---

---

# CTE vs 서브쿼리 vs VIEW vs 임시 테이블 — 헷갈리는 것들 구분 ⭐️⭐️⭐️

```txt
넷 다 "쿼리 결과를 테이블처럼 다룬다" 는 점은 비슷한데
"언제까지 살아있는지" 와 "데이터를 실제로 저장하는지" 가 전부 다름
```

|구분|살아있는 기간|데이터를 실제로 저장?|여러 쿼리에서 재사용?|
|---|---|---|---|
|서브쿼리|그 한 줄 안에서만|❌ (그때그때 계산)|❌|
|CTE (`WITH`)|그 쿼리 한 번 실행되는 동안만|❌ (그때그때 계산)|❌ (같은 쿼리 안에서만 여러 번 참조 가능)|
|VIEW|영구적으로 DB 에 정의가 저장됨|❌ (조회할 때마다 다시 계산, 보통)|✅ (다른 쿼리/다른 세션에서도 계속 사용)|
|임시 테이블|보통 그 세션(접속) 이 끝날 때까지|✅ (진짜로 데이터를 디스크/메모리에 저장)|✅ (같은 세션 안의 여러 쿼리에서 재사용)|

```txt
선택 기준:
  이번 쿼리 한 번만 단계를 나누고 싶다       → CTE (가장 가벼움, 가장 흔하게 씀)
  같은 정의를 앞으로도 계속, 여러 쿼리에서 쓰고 싶다 → VIEW
  계산이 무겁고 같은 세션에서 여러 번 재사용해야 한다 → 임시 테이블 (실제로 저장해두니 다시 계산 안 해도 됨)

CTE 가 "임시 테이블" 이라고 불리지만 진짜 테이블처럼 디스크에 저장되는 건 아님
→ 그 쿼리가 실행되는 동안만 잠깐 존재하는, 이름 붙은 서브쿼리에 더 가까움
```

---

---

# 윈도우 함수 결과 필터 — CTE 필수 ⭐️

```sql
-- ❌ 윈도우 함수 결과를 바로 WHERE 로 필터 불가
SELECT person_name,
       SUM(weight) OVER (ORDER BY turn) AS weight_sum
FROM Queue
WHERE weight_sum <= 1000;   -- 에러! weight_sum 은 아직 계산 전

-- ✅ CTE 로 먼저 계산 → 그 다음 WHERE 필터
WITH total_weight AS (
    SELECT
        person_name,
        SUM(weight) OVER (ORDER BY turn) AS weight_sum
    FROM Queue
)
SELECT person_name
FROM total_weight
WHERE weight_sum <= 1000       -- CTE 결과에서 필터
ORDER BY weight_sum DESC
LIMIT 1;
```

```txt
실행 순서 이해:
  1. CTE 안의 쿼리 먼저 실행 → 임시 결과 생성
  2. 외부 쿼리에서 CTE 를 테이블처럼 사용

  윈도우 함수(OVER) 는 SELECT 단계에서 계산됨
  WHERE 는 SELECT 보다 먼저 실행되는 단계
  → 따라서 WHERE 에서 같은 줄의 윈도우 함수 결과를 바로 못 씀
  → CTE 로 먼저 "윈도우 함수까지 다 계산된 결과" 를 만들고
    바깥 쿼리에서 그 결과를 보통 컬럼처럼 WHERE 로 필터
```

---

---

# 여러 CTE 연결 ⭐️

```sql
-- CTE 여러 개를 , 로 연결
WITH
step1 AS (
    SELECT ... FROM raw_data
),
step2 AS (
    SELECT ... FROM step1        -- 앞의 CTE 를 참조
    WHERE ...
),
step3 AS (
    SELECT ... FROM step2
    JOIN other_table ON ...
)
SELECT * FROM step3;
```

```txt
복잡한 쿼리를 단계별로 분리:
  step1  원본 데이터 정제
  step2  필터링
  step3  집계 / JOIN
  → 각 단계를 독립적으로 이해 가능, 중간 단계만 따로 떼서 실행해보며 디버깅도 쉬움
  → 뒤 CTE 가 앞 CTE 를 참조하는 건 되지만, 반대(앞이 뒤를 참조)는 안 됨 (위에서 아래로만 흐름)
```

---

---

# 재귀 CTE — WITH RECURSIVE ⭐️⭐️⭐️

```txt
지금까지 본 CTE 는 "한 번 계산하고 끝" 이었음
근데 "직원의 모든 상위 관리자" 나 "카테고리의 모든 하위 카테고리" 처럼
계층(트리) 구조를 끝까지 따라 올라가거나 내려가야 할 때는
CTE 가 "자기 자신을 다시 참조" 하면서 반복적으로 실행돼야 함 → 이게 재귀 CTE
```

```sql
-- 특정 직원(id=5)의 모든 상위 관리자 체인 찾기
WITH RECURSIVE manager_chain AS (
    -- ① anchor(기준점) — 재귀의 시작이 되는 한 행, 딱 한 번만 실행됨
    SELECT employee_id, manager_id, name, 1 AS depth
    FROM Employees
    WHERE employee_id = 5

    UNION ALL

    -- ② 재귀 부분 — manager_chain(자기 자신)을 다시 참조
    SELECT e.employee_id, e.manager_id, e.name, mc.depth + 1
    FROM Employees e
    JOIN manager_chain mc ON e.employee_id = mc.manager_id
)
SELECT * FROM manager_chain;
```

```txt
동작 순서:
  1. anchor 부분 실행 → employee_id=5 인 행 하나로 시작
  2. 재귀 부분 실행 → 방금 찾은 행의 manager_id 를 가진 직원을 또 찾음
  3. 그 결과를 다시 manager_chain 에 추가하고, 또 그 manager_id 로 한 단계 더 올라감
  4. 더 이상 매칭되는 행이 없으면(맨 위 관리자까지 도달) 자동으로 멈춤

WITH RECURSIVE 인 이유:
  그냥 WITH 로는 CTE 가 자기 자신(manager_chain)을 정의 중간에 참조할 수 없음
  RECURSIVE 키워드가 있어야 "이 CTE 는 자기 자신을 다시 참조하는 게 허용된다" 고 DB 에 알려줌

구조는 항상 2부분:
  anchor 부분    (한 번만 실행되는 "시작점", UNION ALL 위쪽)
  재귀 부분      (CTE 자기 이름을 참조, UNION ALL 아래쪽 — 결과가 없을 때까지 반복)

흔한 사용처:
  조직도(상위 관리자 체인) / 카테고리 트리(상위·하위 카테고리) / 댓글-답글 구조(부모-자식)
```

## ⚠️ 무한 루프 주의

```txt
실수로 종료 조건이 없는 데이터(순환 참조 등)면 무한히 반복될 수 있음
대부분의 DB 는 안전장치로 재귀 깊이 제한(예: PostgreSQL 기본 100단계 경고)이 있지만
  큰 데이터에서는 직접 깊이 제한을 둘 수도 있음 (위 예시의 depth 컬럼처럼)

WITH RECURSIVE manager_chain AS (
  ...
  UNION ALL
  SELECT ...
  FROM Employees e JOIN manager_chain mc ON ...
  WHERE mc.depth < 10   -- 안전하게 최대 10단계까지만
)
```

---

---

# 실전 패턴

## 패턴 1 — 누적합 + 조건 필터

```sql
-- 버스 무게 제한 초과 전 마지막 탑승자
WITH total_weight AS (
    SELECT
        person_name,
        SUM(weight) OVER (ORDER BY turn) AS weight_sum
    FROM Queue
)
SELECT person_name
FROM total_weight
WHERE weight_sum <= 1000
ORDER BY weight_sum DESC
LIMIT 1;
```

## 패턴 2 — 전체 대비 비율

```sql
WITH total AS (
    SELECT COUNT(*) AS total_cnt FROM orders
)
SELECT
    status,
    COUNT(*) AS cnt,
    COUNT(*) * 100.0 / total_cnt AS ratio
FROM orders, total
GROUP BY status, total_cnt;
```

## 패턴 3 — 그룹별 최신 행

```sql
WITH ranked AS (
    SELECT *,
           ROW_NUMBER() OVER (PARTITION BY user_id ORDER BY created_at DESC) AS rn
    FROM orders
)
SELECT *
FROM ranked
WHERE rn = 1;   -- 각 user_id 의 가장 최근 주문만
```

## 패턴 4 — 같은 문제를 CTE 로: 두 테이블 중 한쪽에만 있는 행 ⭐️

```sql
-- [[SQL_SubQuery]] 에서 인라인 뷰(FROM 절 서브쿼리)로 풀었던 것과 완전히 같은 쿼리
-- FROM (...) AS alias 대신 WITH ... AS (...) 로 "이름을 먼저 붙여둔" 것뿐
WITH employee_ids AS (
    SELECT employee_id FROM Employees
    UNION ALL
    SELECT employee_id FROM Salaries
)
SELECT employee_id
FROM employee_ids
GROUP BY employee_id
HAVING COUNT(*) = 1
ORDER BY employee_id;
```

```bash
인라인 뷰 버전과 비교:
  FROM (SELECT ... UNION ALL ...) AS employee_ids   ← 서브쿼리를 "그 자리에서" 바로 씀
  WITH employee_ids AS (SELECT ... UNION ALL ...)   ← 먼저 이름 붙여서 선언, 본문은 그 이름만 사용

  로직은 토씨 하나 안 다르게 똑같음 — 차이는 오직 "어디서 정의하고 어디서 쓰는지" 뿐
  → CTE 는 본문(SELECT employee_id FROM employee_ids ...) 이 훨씬 깔끔하게 읽힘
  # → [[SQL_JOIN]] 의 Anti-Join+UNION 버전, NOT EXISTS 버전과 함께
    "같은 문제, 4가지 해법" 으로 #[[SQL_SubQuery]] 의 종합 비교 섹션 참고
```

---

---

# 한눈에

| |CTE|서브쿼리|VIEW|임시 테이블|
|---|---|---|---|---|
|재사용 범위|같은 쿼리 안에서만|그 자리 한 번만|여러 쿼리/세션|같은 세션의 여러 쿼리|
|가독성|✅ 단계별 명확|❌ 중첩되면 복잡|✅ (이름으로 호출)|✅|
|윈도우 함수 필터|✅ 가능 (거의 필수)|✅ 인라인 뷰로 가능|✅|✅|
|재귀(자기 참조)|✅ `WITH RECURSIVE`|❌|❌ (재귀 VIEW 는 일반적이지 않음)|△ (직접 반복문으로 채워야)|
|실제 데이터 저장|❌|❌|❌ (보통)|✅|

```txt
CTE 핵심 3가지만 기억:
  ① 서브쿼리에 "이름" 을 붙여서 단계별로 읽기 좋게 만든 것
  ② 윈도우 함수 결과를 WHERE 로 필터하려면 거의 항상 CTE(또는 인라인 뷰) 필요
  ③ 자기 자신을 참조해야(계층 구조) 할 땐 WITH RECURSIVE
```