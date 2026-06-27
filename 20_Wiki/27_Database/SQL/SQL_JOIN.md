---
aliases:
  - JOIN
  - INNER JOIN
  - LEFT JOIN
  - RIGHT JOIN
  - FULL JOIN
  - CROSS JOIN
  - Anti-Join
tags:
  - SQL
related:
  - "[[00_SQL_HomePage]]"
  - "[[SQL_Subquery]]"
  - "[[SQL_CTE]]"
---
# SQL_JOIN — 테이블 연결

# 한 줄 요약

```txt
JOIN = 두 테이블을 연결해서 하나로 조회
INNER  = 양쪽 모두 있는 것만
LEFT   = 왼쪽 전부 + 오른쪽 있으면 붙이기
RIGHT  = 오른쪽 전부 + 왼쪽 있으면 붙이기
FULL   = 양쪽 전부
```

---

---

# 왜 JOIN 이 필요한가 ⭐️

```txt
관계형 DB 는 데이터를 여러 테이블로 "쪼개서" 저장함
  예: 직원 정보는 employees, 부서 정보는 departments — 따로 보관

  쪼개는 이유: 부서 이름이 바뀌면 departments 테이블 한 줄만 고치면 됨
              (모든 직원 행에 부서 이름을 복사해서 저장했다면 전부 다 고쳐야 함)

→ 평소엔 따로 보관하다가, "직원 이름 + 부서 이름을 같이 보고 싶다" 처럼
  여러 테이블의 정보를 한 번에 합쳐서 봐야 할 때 JOIN 을 씀
```

---

---

# JOIN 종류 ⭐️

```sql
-- INNER JOIN: 양쪽 테이블에 모두 있는 행만
SELECT * FROM employees e
INNER JOIN departments d ON e.dept_id = d.id;

-- LEFT JOIN: 왼쪽(employees) 전부 + 오른쪽 매칭
SELECT * FROM employees e
LEFT JOIN departments d ON e.dept_id = d.id;
-- 부서 없는 직원도 포함 (dept 컬럼은 NULL)

-- RIGHT JOIN: 오른쪽(departments) 전부 + 왼쪽 매칭
SELECT * FROM employees e
RIGHT JOIN departments d ON e.dept_id = d.id;
-- 직원 없는 부서도 포함

-- FULL JOIN: 양쪽 전부 (PostgreSQL / 표준 SQL)
SELECT * FROM employees e
FULL JOIN departments d ON e.dept_id = d.id;
```

|JOIN|포함하는 행|
|---|---|
|INNER|양쪽 모두 있는 행만|
|LEFT|왼쪽 전부 + 오른쪽 매칭 (없으면 NULL)|
|RIGHT|오른쪽 전부 + 왼쪽 매칭 (없으면 NULL)|
|FULL|양쪽 전부 (없으면 NULL)|

```txt
헷갈리면 이렇게 판단: "어느 쪽을 무조건 다 보고 싶은가?"
  양쪽 다 있는 것만 보고 싶다           → INNER
  왼쪽은 무조건 다 보고, 오른쪽은 있으면만 → LEFT
  오른쪽은 무조건 다 보고, 왼쪽은 있으면만 → RIGHT
  양쪽 다 무조건 다 보고 싶다           → FULL

실무에서는 RIGHT JOIN 보다 LEFT JOIN 을 더 많이 씀
  → 테이블 순서를 바꿔서 LEFT JOIN 으로 똑같이 표현 가능하기 때문
    (A RIGHT JOIN B  ==  B LEFT JOIN A)
```

---

---

# ON vs WHERE ⭐️

```sql
-- ON: JOIN 조건 (어떻게 연결할지)
-- WHERE: 결과 필터 (JOIN 후 어떤 것을 볼지)

SELECT e.name, d.name
FROM employees e
LEFT JOIN departments d
    ON e.dept_id = d.id       -- JOIN 조건
WHERE d.location = 'Seoul';    -- 결과 필터
```

```txt
LEFT JOIN 에서 WHERE 와 ON+AND 차이 ⭐️:

문제:
  LEFT JOIN 후 WHERE 에 오른쪽 테이블 조건 넣으면
  → NULL 행 제거됨 → INNER JOIN 처럼 동작

  ❌ WHERE 에 조건 (판매 없는 제품 제외됨)
  SELECT p.product_id
  FROM Prices p
  LEFT JOIN UnitsSold u ON p.product_id = u.product_id
  WHERE u.purchase_date BETWEEN p.start_date AND p.end_date
  -- u가 NULL 인 행 → WHERE 에서 제거됨

  ✅ ON 에 AND 로 조건 (판매 없는 제품 NULL 로 유지)
  SELECT p.product_id
  FROM Prices p
  LEFT JOIN UnitsSold u
      ON p.product_id = u.product_id
     AND u.purchase_date BETWEEN p.start_date AND p.end_date
  -- 날짜 범위 안 맞으면 u 컬럼 전체 NULL / 제품 자체는 유지
```

---

---

# ON + AND — JOIN 에 여러 조건 ⭐️

```sql
-- ON 에 여러 조건을 AND 로 추가
FROM Prices p
LEFT JOIN UnitsSold u
    ON p.product_id = u.product_id
   AND u.purchase_date BETWEEN p.start_date AND p.end_date
--  ↑ JOIN 할 때 날짜 범위도 같이 체크
```

```txt
ON + AND 언제 쓰나:
  LEFT JOIN 에서 조건을 걸되
  조건 불만족 시 오른쪽 테이블을 NULL 로 남기고 싶을 때

  ON p.id = u.id
  AND u.날짜 BETWEEN p.시작 AND p.끝
  → 날짜 범위 밖 = u 컬럼 전체 NULL (행은 유지)
  → WHERE 에 넣으면 그 행 자체가 사라짐 ← 차이!
```

---

---

# Anti-Join — LEFT JOIN + WHERE IS NULL ⭐️⭐️⭐️

```txt
"한쪽 테이블에는 있는데, 다른 쪽엔 없는 행" 을 찾는 패턴 — Anti-Join 이라고 부름
예: "부서가 배정 안 된 직원" / "한 번도 주문 안 한 고객" 처럼
    "매칭이 안 되는 쪽" 자체를 찾는 게 목적일 때
```

```sql
-- 부서가 없는 직원 찾기
SELECT e.name
FROM employees e
LEFT JOIN departments d ON e.dept_id = d.id
WHERE d.id IS NULL;
```

```txt
동작 원리:
  ① LEFT JOIN 으로 employees 는 전부, departments 는 매칭되면만 붙임
     → 매칭 안 된 직원의 d.id 자리는 NULL 로 채워짐
  ② WHERE d.id IS NULL 로 "매칭이 안 된 행만" 골라냄
     → 결과적으로 "departments 에 짝이 없는 employees" 만 남음
```

## ⚠️ 바로 위 "ON vs WHERE" 경고와 모순처럼 보이지만 아님 ⭐️

```txt
위에서 "LEFT JOIN 후 WHERE 에 오른쪽 테이블 조건 넣으면 NULL 행이 사라진다" 고 경고했는데
여기서는 정확히 그 "사라지는 NULL 행" 을 찾는 게 목적임
→ 둘은 반대 상황일 뿐, 같은 원리(WHERE 의 오른쪽 조건 = NULL 행 제거) 위에서 설명됨

판단 기준:
  매칭 안 된 행을 "없애고" 싶다  → WHERE 에 오른쪽 조건 (그래서 NULL 행이 사라짐, 의도한 결과)
  매칭 안 된 행을 "찾고" 싶다    → WHERE 절에 IS NULL 로 정확히 그 NULL 행만 남김 (Anti-Join)
  매칭 조건을 더 정밀하게 걸되 행 자체는 유지하고 싶다 → ON + AND (위 섹션)
```

## ⚠️ 기준 컬럼 선택 — IS NULL 은 "아무 컬럼" 이나 검사하면 안 됨 ⭐️⭐️⭐️

```txt
Anti-Join 에서 가장 많이 하는 실수: WHERE 오른쪽.아무컬럼 IS NULL 처럼
"오른쪽 테이블의 임의의 컬럼" 을 검사하는 것 — 이러면 틀린 결과가 나올 수 있음

올바른 기준: 오른쪽 테이블의 PK(기본키) — 또는 "절대 NULL 일 수 없는" 컬럼만 검사해야 함
```

```sql
-- 실수 사례 — 직원-관리자 자기 자신과 JOIN(Self Join)
-- "급여 3만 미만이면서, 관리자가 있었는데 그 관리자가 지금 Employees 에 없는 사람" 찾기

-- ❌ 틀림
SELECT e1.employee_id
FROM Employees e1
LEFT JOIN Employees e2 ON e1.manager_id = e2.employee_id
WHERE e1.salary < 30000
  AND e1.manager_id IS NOT NULL
  AND e2.manager_id IS NULL;     -- ← 검사 대상이 잘못됨

-- ✅ 올바름
SELECT e1.employee_id
FROM Employees e1
LEFT JOIN Employees e2 ON e1.manager_id = e2.employee_id
WHERE e1.salary < 30000
  AND e1.manager_id IS NOT NULL
  AND e2.employee_id IS NULL;    -- ← e2 의 PK 를 검사
```

```txt
일반화한 규칙:
  Anti-Join 에서 WHERE 오른쪽.컬럼 IS NULL 을 쓸 때
  그 "컬럼" 은 항상 오른쪽 테이블의 PK(또는 NOT NULL 제약이 있는 컬럼) 여야 함
  → 그 외의 일반 컬럼은 "매칭은 됐지만 그 컬럼 값 자체가 원래 NULL" 인 경우와
    "매칭이 안 됨" 인 경우를 구분 못 해서 잘못된 결과를 낼 수 있음

  이 노트의 다른 Anti-Join 예시들이 전부 d.id IS NULL / o.customer_id IS NULL 처럼
  "오른쪽 테이블의 PK" 를 쓰고 있던 것도 우연이 아니라 바로 이 이유 때문임
```

```txt
참고 — Self Join 이라는 것도 같이 기억해두면 좋음:
  위 예시는 Employees 테이블을 "자기 자신과" JOIN 한 것 (Self Join)
  같은 테이블이라서 e1 / e2 처럼 별칭(alias)을 다르게 줘야만 구분이 가능함
  관리자-직원처럼 "같은 테이블 안에서 행끼리 관계가 있는" 데이터에서 흔히 씀
```

## 실전 — 두 테이블에서 "한쪽에만 있는" employee_id 찾기 ⭐️

```sql
-- Employees 에는 있는데 Salaries 에는 없는 사람
SELECT e.employee_id
FROM Employees e
LEFT JOIN Salaries s
    ON e.employee_id = s.employee_id
WHERE s.employee_id IS NULL

UNION

-- Salaries 에는 있는데 Employees 에는 없는 사람
SELECT s.employee_id
FROM Salaries s
LEFT JOIN Employees e
    ON s.employee_id = e.employee_id
WHERE e.employee_id IS NULL

ORDER BY employee_id;
```

```bash
이 쿼리가 하는 일:
  앞쪽 SELECT  — Anti-Join 으로 "Employees 에만 있는" id 찾기
  뒤쪽 SELECT  — 반대 방향 Anti-Join 으로 "Salaries 에만 있는" id 찾기
  UNION       — 둘을 세로로 합침 (UNION ALL 이 아니라 UNION 인 이유는
                이론상 두 결과가 겹칠 수 없어서 중복 제거 자체는 의미 없지만,
                관례적으로 "혹시 모를 중복까지 안전하게" UNION 을 쓰기도 함)

→ "두 테이블 중 정확히 한쪽에만 있는 행" = 집합론의 대칭차집합(symmetric difference) 과 같은 개념
  같은 문제를 서브쿼리(UNION ALL + GROUP BY + HAVING) 로 푸는 방법은 # [[SQL_SubQuery]] 참고
  → 한 노트에 두 가지 방법을 비교해서 "언제 어느 쪽이 더 읽기 쉬운지" 정리해둠
```

---

---

# COALESCE — NULL 이면 기본값

```sql
COALESCE(값, 기본값)
-- 값이 NULL 이면 기본값 반환

-- LEFT JOIN 결과에서 판매 없는 제품 → 0 으로
COALESCE(SUM(p.price * u.units) / SUM(u.units), 0)
```

→ 자세한 내용: [[SQL_Functions_Basic#COALESCE — NULL 이면 기본값 ⭐️]]

---

---

# 가중평균 패턴 ⭐️

```txt
가중평균 = SUM(가격 × 수량) / SUM(수량)

단순 AVG 와 차이:
  100원 × 3개 / 200원 × 1개
  AVG(price)  = (100+200)/2 = 150  ← 틀림
  가중평균     = (300+200)/4 = 125  ← 맞음
```

## MySQL 방식

```sql
SELECT
    p.product_id,
    ROUND(
        COALESCE(SUM(p.price * u.units) / SUM(u.units), 0),
        2
    ) AS average_price
FROM Prices p
LEFT JOIN UnitsSold u
    ON p.product_id = u.product_id
   AND u.purchase_date BETWEEN p.start_date AND p.end_date
GROUP BY p.product_id;
```

## PostgreSQL 방식

```sql
SELECT
    p.product_id,
    COALESCE(
        ROUND(
            SUM(p.price * u.units)::numeric / SUM(u.units),
            2
        ),
        0
    ) AS average_price
FROM Prices p
LEFT JOIN UnitsSold u
    ON p.product_id = u.product_id
   AND u.purchase_date BETWEEN p.start_date AND p.end_date
GROUP BY p.product_id;
```

```txt
두 방식 차이:
  MySQL:      COALESCE(ROUND(SUM/SUM, 2), 0)
  PostgreSQL: ::numeric 추가 → integer 나눗셈 방지
              COALESCE(ROUND(SUM::numeric/SUM, 2), 0)
```

---

---

# CROSS JOIN — 모든 조합 만들기 ⭐️

```txt
CROSS JOIN = 두 테이블의 모든 행 조합
카테시안 곱 (Cartesian Product)

학생 4명 × 과목 3개 = 12개 행 전부 생성
→ 시험을 안 본 과목도 행으로 만들어줌
```

```sql
-- 모든 학생 × 모든 과목 조합
SELECT s.student_id, s.student_name, sub.subject_name
FROM Students s
CROSS JOIN Subjects sub
ORDER BY s.student_id, sub.subject_name;

-- 결과: 학생 4명 × 과목 3개 = 12행
-- Bob × Physics 행도 생성됨 (시험 안 봤어도)
```

## CROSS JOIN + LEFT JOIN 패턴 ⭐️

```txt
문제:
  학생별 과목별 시험 응시 횟수 구하기
  시험 안 본 과목도 0 으로 표시해야 함

핵심:
  CROSS JOIN 으로 모든 조합 먼저 만들기
  LEFT JOIN 으로 실제 데이터 연결
  COUNT(e.subject_name) 으로 없으면 0
```

```sql
SELECT
    s.student_id,
    s.student_name,
    sub.subject_name,
    COUNT(e.subject_name) AS attended_exams  -- 없으면 자동 0
FROM Students s
CROSS JOIN Subjects sub                      -- 모든 학생 × 과목 조합 생성
LEFT JOIN Examinations e
    ON s.student_id = e.student_id
   AND sub.subject_name = e.subject_name     -- ON 에 조건! (WHERE 에 넣으면 안 됨)
GROUP BY s.student_id, s.student_name, sub.subject_name
ORDER BY s.student_id, sub.subject_name;
```

```txt
왜 CROSS JOIN 이 필요한가:
  Students + Examinations 만 JOIN 하면
  Bob 이 Physics 를 안 봤으면 행 자체가 없음
  → 0 을 표시할 방법이 없음

  CROSS JOIN 으로 모든 조합 먼저 만들면
  Bob × Physics 행이 생성됨
  LEFT JOIN 으로 연결 시 e.subject_name = NULL
  COUNT(e.subject_name) = 0 → 정상 출력

ON 에 AND 조건 (WHERE 아님) ⭐️:
  AND sub.subject_name = e.subject_name 을 WHERE 에 넣으면
  → NULL 행 제거 → 0 이 사라짐
  → 반드시 ON 절 안에!
```

---

---

# SUM vs COUNT

→ [[SQL_Aggregate#SUM vs COUNT — 언제 뭘 쓰나 ⭐️]] 참고

---

---

# 한눈에

| 상황                         | 문법                                                    |
| -------------------------- | ----------------------------------------------------- |
| 양쪽 모두 있는 행                 | `INNER JOIN ... ON`                                   |
| 왼쪽 전부 + 오른쪽 선택             | `LEFT JOIN ... ON`                                    |
| LEFT JOIN + 추가 조건 (행 유지)   | `ON a.id = b.id AND b.날짜 BETWEEN ...`                 |
| 매칭 안 된 행 찾기 (Anti-Join) ⭐️ | `LEFT JOIN ... WHERE b.id IS NULL`                    |
| 두 테이블 중 한쪽에만 있는 것          | Anti-Join 양방향 + `UNION` (또는 [[SQL_Subquery]] 의 다른 방법) |
| 모든 조합 만들기                  | `CROSS JOIN` ⭐️                                       |
| 전체목록 + 없으면 0               | `CROSS JOIN` + `LEFT JOIN` + `COUNT(컬럼)` ⭐️           |
| JOIN 결과 NULL → 기본값         | `COALESCE(컬럼, 0)`                                     |
| 가중평균                       | `SUM(price*units) / SUM(units)`                       |

```txt
WHERE 에 오른쪽 테이블 조건을 넣었을 때 일어나는 일은 항상 같음 — "NULL 행이 사라짐"
  그 사라짐이 "의도치 않은 버그" 면      → ON + AND 로 옮길 것
  그 사라짐(정확히는 "NULL 인 것만 남김") 이 "의도한 목적" 이면 → Anti-Join (IS NULL)

Anti-Join 의 IS NULL 은 항상 "오른쪽 테이블의 PK" 에 걸 것 ⭐️⭐️
  일반 컬럼에 걸면 "매칭은 됐지만 그 컬럼이 원래 NULL인 경우" 와
  "매칭 자체가 안 된 경우" 를 구분 못 해서 결과가 틀어질 수 있음
```