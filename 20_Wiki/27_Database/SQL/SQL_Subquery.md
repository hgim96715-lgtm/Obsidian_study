---
aliases:
  - 서브쿼리
  - 스칼라 서브쿼리
  - 인라인 뷰
  - EXISTS
  - 상관 서브쿼리
  - correlated subquery
tags:
  - SQL
related:
  - "[[00_SQL_HomePage]]"
  - "[[SQL_JOIN]]"
  - "[[SQL_JOIN_Advanced]]"
  - "[[SQL_CTE]]"
---
# SQL_SubQuery — 서브쿼리

# 한 줄 요약

```
서브쿼리 = 쿼리 안에 쿼리
위치에 따라 스칼라 / 인라인 뷰 / WHERE 서브쿼리로 구분
```

---

---

# 왜 서브쿼리가 필요한가 ⭐️

```
"전체 평균보다 급여가 높은 직원" 같은 질문에 답하려면
"전체 평균" 이라는 계산 결과가 먼저 있어야, 그걸 기준으로 다시 비교할 수 있음

→ 한 쿼리로 "계산" 과 "그 계산 결과를 이용한 조회" 를 동시에 하기 어려운 경우
  쿼리 안에 또 다른 쿼리(서브쿼리)를 넣어서
  "먼저 이걸 계산하고, 그 결과로 메인 쿼리를 완성" 하는 방식으로 해결
```

---

---

# 서브쿼리 위치 3가지 ⭐️

```
SELECT 절 → 스칼라 서브쿼리  (행마다 단일 값 반환)
FROM 절   → 인라인 뷰        (가상 테이블)
WHERE 절  → 조건 서브쿼리    (필터링)
```

---

---

# 스칼라 서브쿼리 — SELECT / WHERE 안에서 ⭐️

```
단일 값 하나를 반환하는 서브쿼리
SELECT 절 또는 WHERE 절 안에서 사용
```

```sql
-- 전체 유저 수를 스칼라 서브쿼리로 계산
SELECT
    r.contest_id,
    COUNT(*) AS 참가자수,
    (SELECT COUNT(*) FROM Users) AS 전체유저수
FROM Register r
GROUP BY r.contest_id;
```

## 비율 계산 패턴 ⭐️

```sql
-- PostgreSQL: ::numeric 캐스팅 필요
SELECT
    r.contest_id,
    ROUND(
        COUNT(*)::numeric / (SELECT COUNT(*) FROM Users) * 100,
        2
    ) AS percentage
FROM Register r
GROUP BY r.contest_id
ORDER BY percentage DESC, r.contest_id ASC;

-- MySQL: 캐스팅 없이 바로 나눗셈 가능
SELECT
    r.contest_id,
    ROUND(
        COUNT(*) / (SELECT COUNT(*) FROM Users) * 100,
        2
    ) AS percentage
FROM Register r
GROUP BY r.contest_id
ORDER BY percentage DESC, r.contest_id ASC;
```

```
PostgreSQL vs MySQL 나눗셈 차이 ⭐️:
  PostgreSQL: 정수 / 정수 = 정수 (소수점 버림)
    COUNT(*) = 3, Users = 10 → 3/10 = 0  ← 0 이 나옴!
    → ::numeric 또는 ::float 캐스팅 필수 → [[PG_Specific]] 참고
  MySQL: 정수 / 정수 = 소수 자동 처리
    3/10 = 0.3  (캐스팅 불필요)

해결 패턴 (PostgreSQL):
  COUNT(*)::numeric / 전체수  ← 피제수를 numeric 으로
  COUNT(*) * 1.0 / 전체수     ← 1.0 곱해서 float 유도
```

---

---

# 상관 서브쿼리 vs 비상관 서브쿼리 ⭐️⭐️

```
이 구분을 모르면 "왜 어떤 서브쿼리는 바깥 테이블 컬럼을 쓸 수 있는지" 헷갈릴 수 있음
```

```sql
-- 비상관(non-correlated) — 바깥 쿼리와 무관하게 "혼자서도" 완성된 서브쿼리
(SELECT COUNT(*) FROM Users)
-- → Users 가 몇 명인지는 바깥 쿼리가 뭘 묻든 항상 똑같은 값 하나
-- → 딱 한 번만 계산되고, 모든 바깥 행에 같은 값이 재사용됨

-- 상관(correlated) — 바깥 쿼리의 컬럼(c.id)을 안에서 직접 참조함
SELECT c.name
FROM customers c
WHERE EXISTS (
    SELECT 1
    FROM orders o
    WHERE o.customer_id = c.id   -- ← 바깥의 c.id 를 그대로 씀
);
```

```
비상관 서브쿼리:
  바깥 쿼리 없이도 "혼자서" 실행 가능 (Users 수는 그 자체로 의미가 있는 질문)
  → 한 번만 계산해서 결과를 재사용 (보통 더 빠름)

상관 서브쿼리:
  바깥 쿼리의 "현재 행" 값(c.id)을 안에서 참조해야 의미가 생김
  → 개념적으로 "바깥 쿼리의 각 행마다 다시 실행되는 것" 처럼 동작함
  → EXISTS / NOT EXISTS 는 거의 항상 상관 서브쿼리로 씀 (아래 EXISTS 섹션 참고)

구분하는 방법:
  서브쿼리 안에 바깥 테이블의 별칭(c.id 같은 것)이 보이면 → 상관
  서브쿼리만 따로 떼서 실행해도 말이 되면(Users 수처럼) → 비상관
```

---

---

# 인라인 뷰 — FROM 안에서

```sql
-- FROM 절에 서브쿼리를 넣어 가상 테이블로 사용
SELECT dept, avg_salary
FROM (
    SELECT dept, AVG(salary) AS avg_salary
    FROM employees
    GROUP BY dept
) AS dept_avg
WHERE avg_salary > 5000;
```

```bash
인라인 뷰:
  FROM 절 안에 서브쿼리
  반드시 별칭(AS) 필요
  # CTE(WITH) 로 대체 가능 → 더 가독성 좋음 → [[SQL_CTE]] 참고
```

## 실전 — UNION ALL + GROUP BY + HAVING — 인라인 뷰의 대표 활용 ⭐️⭐️⭐️

```sql
-- 두 테이블 중 "정확히 한쪽에만" 존재하는 employee_id 찾기
SELECT employee_id
FROM (
    SELECT employee_id FROM Employees
    UNION ALL
    SELECT employee_id FROM Salaries
) AS employee_ids
GROUP BY employee_id
HAVING COUNT(*) = 1
ORDER BY employee_id;
```

```bash
한 줄씩 — 왜 이렇게 동작하는가:

  ① FROM 절 안의 (UNION ALL ...) 부분이 인라인 뷰
     Employees 의 모든 employee_id 와 Salaries 의 모든 employee_id 를
     "구분 없이 그냥 쌓아올린" 하나의 가상 테이블을 만듦
     (UNION ALL 이라 중복 제거 없이, 양쪽에 있으면 2번 쌓임)

  ② GROUP BY employee_id
     같은 employee_id 끼리 묶음

  ③ HAVING COUNT(*) = 1
     그 employee_id 가 "쌓인 횟수가 정확히 1번" 인 것만 남김
     → 만약 그 id 가 Employees, Salaries 양쪽에 다 있었다면 2번 쌓였을 것 (COUNT=2)
     → 1번만 쌓였다는 건 "둘 중 한쪽에만 있었다" 는 뜻

# → [[SQL_JOIN]] 의 Anti-Join(LEFT JOIN + WHERE IS NULL) 으로 푼 것과 정확히 같은 결과
  "어느 쪽에만 있는지" 까지는 안 알려주지만(둘을 구분 안 함), 코드가 훨씬 짧음
  → 아래 "종합 비교" 섹션에서 두 방법을 나란히 비교
```

---

---

# WHERE 서브쿼리

```sql
-- 평균보다 높은 급여 조회
SELECT name, salary
FROM employees
WHERE salary > (SELECT AVG(salary) FROM employees);

-- IN 서브쿼리
SELECT name
FROM employees
WHERE dept_id IN (SELECT id FROM departments WHERE location = 'Seoul');
```

```
이 두 예시는 비상관 서브쿼리 (바깥 쿼리 없이도 각자 완성된 질문)
```

---

---

# EXISTS ⭐️

```sql
-- 주문이 있는 고객만 조회
SELECT c.name
FROM customers c
WHERE EXISTS (
    SELECT 1
    FROM orders o
    WHERE o.customer_id = c.id
);
```

```
EXISTS:
  서브쿼리가 행을 하나라도 반환하면 TRUE
  반환 값은 중요하지 않음 → SELECT 1 관례 (실제 값을 안 쓰고 "존재하는지" 만 보기 때문)
  IN 보다 큰 데이터셋에서 성능 좋음 (보통 첫 매칭에서 바로 멈추는 식으로 최적화됨)

NOT EXISTS:
  서브쿼리 결과가 없으면 TRUE
```

## NOT EXISTS vs Anti-Join(LEFT JOIN + IS NULL) — 같은 목적, 다른 방법 ⭐️

```sql
-- NOT EXISTS 버전 — 주문 없는 고객
SELECT c.name
FROM customers c
WHERE NOT EXISTS (
    SELECT 1 FROM orders o WHERE o.customer_id = c.id
);

-- Anti-Join 버전 — 같은 결과 ([[SQL_JOIN]] 의 Anti-Join 참고)
SELECT c.name
FROM customers c
LEFT JOIN orders o ON o.customer_id = c.id
WHERE o.customer_id IS NULL;
```

```
둘 다 "주문이 없는 고객" 을 찾는 같은 목적
NOT EXISTS  → 상관 서브쿼리 — "이 고객(c.id) 에 대한 주문이 하나라도 있나?" 를 직접 물어봄
Anti-Join   → JOIN 으로 일단 다 붙여보고, 안 붙은(NULL) 것만 거름

선택 기준 (대부분 둘 다 결과는 같고, DB 옵티마이저가 비슷하게 최적화하는 경우가 많음):
  "없다는 것" 자체를 묻는 의도가 분명할 때           → NOT EXISTS (읽었을 때 의도가 더 명확)
  이미 LEFT JOIN 으로 다른 컬럼도 같이 가져와야 할 때  → Anti-Join (조인 결과를 그대로 활용)
```

---

---

# 종합 — 같은 문제, 4가지 해법 비교 ⭐️⭐️⭐️

```
"두 테이블 중 정확히 한쪽에만 있는 employee_id 찾기" 를
네 가지 완전히 다른 접근으로 풀 수 있음 — 결과는 같지만 사고방식이 다름
```

|방법|핵심 아이디어|노트|
|---|---|---|
|Anti-Join × 2 + UNION|양쪽 방향으로 "매칭 안 된 행" 찾고 합치기|[[SQL_JOIN]] 의 Anti-Join|
|UNION ALL + GROUP BY + HAVING COUNT=1 (인라인 뷰)|일단 다 쌓고, 정확히 1번만 나온 것 찾기|이 노트의 "인라인 뷰" 실전 예제|
|같은 로직 + CTE|위와 완전히 같은 로직, `FROM(...)` 대신 `WITH ... AS(...)` 로 이름만 붙임|[[SQL_CTE]] 의 "패턴 4"|
|NOT EXISTS|"이 id 가 반대편 테이블에 있나?" 를 직접 물어보기 (양방향으로 한 번씩)|이 노트의 EXISTS 섹션|

```typescript
// NOT EXISTS 로 풀면 이런 형태가 됨 (양방향 각각)
SELECT employee_id FROM Employees e
WHERE NOT EXISTS (SELECT 1 FROM Salaries s WHERE s.employee_id = e.employee_id)
UNION
SELECT employee_id FROM Salaries s
WHERE NOT EXISTS (SELECT 1 FROM Employees e WHERE e.employee_id = s.employee_id);
```

```
어느 게 "정답" 인 건 아님 — 같은 문제를 보는 4가지 다른 사고방식:
  Anti-Join 식       "조인해보고 안 붙은 거 찾기" 라는 시각적/관계 중심 사고
  GROUP BY+HAVING 식  "다 모아놓고 개수로 판단" 이라는 집계 중심 사고 — 코드가 가장 짧음
  CTE 식             위와 로직은 동일, "이름을 먼저 붙여서" 본문을 더 깔끔하게 읽히게 함
  NOT EXISTS 식       "존재 여부를 직접 질문" 하는 논리 중심 사고 — 의도가 가장 명확하게 읽힘

처음에 어떤 방법이 안 떠올라도 괜찮음 — 이렇게 "같은 문제, 여러 해법" 을 비교해보면서
  점점 어떤 상황에 어떤 사고방식이 더 빨리 떠오르는지 감이 생김
```

---

---

# 한눈에

|위치|이름|용도|
|---|---|---|
|SELECT 절|스칼라 서브쿼리|행마다 단일 값 계산|
|FROM 절|인라인 뷰|가상 테이블|
|WHERE 절|조건 서브쿼리|동적 필터링|

```
비율 계산 시:
  PostgreSQL → COUNT(*)::numeric / 전체수 * 100
  MySQL      → COUNT(*) / 전체수 * 100

상관 vs 비상관:
  서브쿼리 안에 바깥 테이블 컬럼이 보이면 → 상관 (EXISTS 가 대표적)
  서브쿼리만 떼어내도 의미가 통하면      → 비상관 (스칼라 서브쿼리 다수)

"두 테이블 중 한쪽에만 있는 것" 찾는 3가지 방법 → 위 "종합 비교" 섹션 참고
```