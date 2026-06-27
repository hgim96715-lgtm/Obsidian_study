---
aliases:
  - UNION ALL
  - 다중 JOIN
  - 체인 JOIN
  - 고급 JOIN
  - UNION
  - Unpivot
  - 와이드, 롱
tags:
  - SQL
related:
  - "[[00_SQL_HomePage]]"
  - "[[SQL_JOIN]]"
  - "[[SQL_Window_Functions]]"
  - "[[SQL_CTE]]"
---
# SQL_JOIN_Advanced — 고급 JOIN & UNION

# 한 줄 요약

```
JOIN   테이블을 가로로 합침 (컬럼 추가)
UNION  쿼리 결과를 세로로 합침 (행 추가)
```

---

---

# JOIN vs UNION ⭐️

```
JOIN  — 가로로 합침 (컬럼 늘어남)

  users          orders
  id | name      user_id | amount
  1  | 철수      1       | 5000

  JOIN 결과:
  id | name | amount
  1  | 철수  | 5000

UNION — 세로로 합침 (행 늘어남)

  korea_users    japan_users
  name | city    name | city
  철수  | 서울   太郎  | 도쿄

  UNION 결과:
  name | city
  철수  | 서울
  太郎  | 도쿄
```

---

---

# UNION vs UNION ALL ⭐️

```sql
-- UNION — 중복 제거 (느림)
SELECT name FROM employees_korea
UNION
SELECT name FROM employees_japan;

-- UNION ALL — 중복 허용 (빠름) ← 기본 사용 권장
SELECT name FROM employees_korea
UNION ALL
SELECT name FROM employees_japan;
```

```
UNION:
  내부적으로 DISTINCT 실행 → 느림
  완전히 같은 행 제거

UNION ALL:
  중복 제거 없이 그냥 붙임 → 빠름
  카테고리가 겹치지 않으면 항상 UNION ALL

언제 UNION (중복 제거) 써야 하나:
  두 결과에 같은 행이 있고 한 번만 보여줘야 할 때
  예: 두 기간의 활성 유저 목록 합치기
```

---

---

# 사용 규칙 ⭐️

```sql
-- ✅ 컬럼 수가 같아야 함
SELECT id, name FROM users
UNION ALL
SELECT id, name FROM admins;

-- ❌ 컬럼 수 다르면 에러
SELECT id, name FROM users
UNION ALL
SELECT id, name, email FROM admins;

-- 컬럼 수 맞출 때 NULL 채우기
SELECT id, name, email, NULL AS phone FROM users
UNION ALL
SELECT id, company, NULL AS email, phone FROM companies;
```

```
규칙:
  SELECT 컬럼 수가 같아야 함
  각 컬럼 타입이 호환되어야 함
  결과 컬럼명 = 첫 번째 SELECT 의 이름 기준
  ORDER BY 는 전체 마지막에 한 번만
```

---

---

# ORDER BY + LIMIT 은 괄호로 ⭐️

```sql
-- ❌ 괄호 없으면 LIMIT 이 전체에 적용됨
SELECT name FROM A ORDER BY name LIMIT 1
UNION ALL
SELECT name FROM B ORDER BY name LIMIT 1;

-- ✅ 각 쿼리를 괄호로 감싸야 각각 적용
(SELECT name FROM A ORDER BY name LIMIT 1)
UNION ALL
(SELECT name FROM B ORDER BY name LIMIT 1);
```

---

---

# 실전 패턴

## 와이드 → 롱 변환 (Unpivot) — 같은 테이블을 여러 번 ⭐️⭐️⭐️

```
지금까지 본 UNION ALL 은 전부 "서로 다른 테이블" 을 세로로 합치는 용도였음
근데 "테이블 하나" 를 컬럼만 다르게 골라서 여러 번 조회한 뒤 합치는 것도 가능함
→ 가로로 퍼진(wide) 여러 컬럼을 세로로 긴(long) 형태로 펴는 패턴 — 흔히 "Unpivot" 이라 부름
```

```sql
-- Products 테이블 (와이드 형태 — 한 행에 store별 가격이 옆으로 나열됨)
-- product_id | store1 | store2 | store3
-- 1          | 9.99   | NULL   | 8.99

SELECT
    product_id,
    'store1' AS store,
    store1   AS price
FROM Products
WHERE store1 IS NOT NULL

UNION ALL

SELECT
    product_id,
    'store2' AS store,
    store2   AS price
FROM Products
WHERE store2 IS NOT NULL

UNION ALL

SELECT
    product_id,
    'store3' AS store,
    store3   AS price
FROM Products
WHERE store3 IS NOT NULL;

-- 결과 (롱 형태 — 한 행 = product 하나 + store 하나 + price 하나)
-- product_id | store  | price
-- 1          | store1 | 9.99
-- 1          | store3 | 8.99
```

## 'store1' AS store — 왜 따옴표가 있는 리터럴을 쓰는가 ⭐️

```
보통 AS 는 "컬럼 이름" 뒤에 붙여서 그 컬럼을 다른 이름으로 부르는 용도로 씀
  (예: store1 AS price → store1 "컬럼" 의 값을 price 라는 이름으로 내보냄)

근데 'store1' 처럼 따옴표로 감싸면 컬럼이 아니라 "문자열 그 자체"(리터럴)임
→ store1 컬럼의 "값" 을 가져오는 게 아니라
  이 부분 쿼리에서 나오는 모든 행에 'store1' 이라는 글자를 그대로 박아 넣는 것
  (해당 SELECT 가 몇 행을 반환하든, store 컬럼 값은 전부 똑같이 'store1')
```

```
왜 이게 필요한가:
  store1 AS price 만 하면 9.99 라는 "값" 만 넘어가고
  그 값이 원래 "store1 컬럼" 이었다는 정보(컬럼 이름 자체)는 사라짐
  → 나중에 결과를 보면 "이 가격이 어느 store 것인지" 구분할 방법이 없어짐
  → 그래서 그 정보를 'store1' 문자열 리터럴로 직접 다시 만들어서 같이 내려보내는 것
```

## 같은 테이블을 3번 조회하는 이유 — 컬럼을 "내리는" 원리 ⭐️

```
한 행(product_id=1) 안에 store1/store2/store3 가격이 나란히 있는 게 "와이드(wide)" 형태
이걸 "한 컬럼(store)당 한 행" 으로 바꾸고 싶으면(롱/long 형태)

→ 컬럼을 하나씩 따로 뽑아서(각각 다른 SELECT), 그 결과들을 UNION ALL 로 쌓아 올리는 것
→ 원래 1행이 (store1/2/3 값이 있는 만큼) 최대 3행으로 "펴짐"

  SELECT 1 — store1 컬럼만 뽑아서 store='store1' 인 행으로 변환
  SELECT 2 — store2 컬럼만 뽑아서 store='store2' 인 행으로 변환
  SELECT 3 — store3 컬럼만 뽑아서 store='store3' 인 행으로 변환
  → 셋을 UNION ALL 로 이어붙이면 "컬럼이 행으로 내려간" 최종 결과가 됨
```

```
WHERE store1 IS NOT NULL 이 필요한 이유:
  특정 상품이 store1 에서 안 팔리면(store1 컬럼 값이 NULL)
  그 행까지 그대로 변환하면 (product_id, 'store1', NULL) 같은 의미 없는 행이 생김
  → 실제로 가격이 있는 store 데이터만 결과에 남도록 필터링하는 것
```

```
이 패턴의 이름 — Unpivot:
  와이드(컬럼이 여러 개로 퍼짐) → 롱(한 컬럼당 한 행)으로 바꾸는 것 = Unpivot
  반대로 롱 → 와이드로 모으는 건 Pivot (보통 CASE WHEN + GROUP BY 로 구현)

지금까지의 UNION ALL 예시와 차이점:
  employees_korea / employees_japan 합치기  → "다른 테이블" 을 그대로 합침
  이 패턴(store1/2/3)                      → "같은 테이블" 을 컬럼만 다르게 골라 여러 번 합침
  → UNION ALL 자체는 같은 문법, "무엇을 합치는가" 만 다름
```

## 고정 카테고리 집계 ⭐️

```sql
-- 카테고리별 COUNT — 빈 카테고리도 0으로 출력
SELECT 'Low Salary'     AS category, COUNT(*) AS accounts_count
FROM Accounts WHERE income < 20000

UNION ALL

SELECT 'Average Salary' AS category, COUNT(*) AS accounts_count
FROM Accounts WHERE income BETWEEN 20000 AND 50000

UNION ALL

SELECT 'High Salary'    AS category, COUNT(*) AS accounts_count
FROM Accounts WHERE income > 50000;
```

```bash
COUNT(*) 는 행이 0개여도 0 반환
→ 빈 카테고리도 자동으로 0 으로 나옴
→ GROUP BY 만 쓰면 빈 그룹은 결과에서 사라짐

이 패턴도 위 Unpivot 과 같은 원리: 'Low Salary' 같은 문자열 리터럴로
"이 묶음의 결과가 무슨 카테고리인지" 를 직접 라벨링하는 것

# 더 복잡한 경우 → [[SQL_Pattern_전체목록_LEFT_JOIN]] 참고
```

## 두 가지 1등 하나로 합치기 ⭐️

```sql
-- "사용자별 평가 개수 1등" 이름
-- + "2월 영화별 평균 평점 1등" 제목
-- → results 컬럼 하나로 출력

(
  SELECT u.name AS results
  FROM Users u
  JOIN MovieRating mr ON u.user_id = mr.user_id
  GROUP BY u.name
  ORDER BY COUNT(*) DESC, u.name ASC   -- 동점 시 이름 오름차순
  LIMIT 1
)
UNION ALL
(
  SELECT m.title AS results
  FROM Movies m
  JOIN MovieRating mr ON m.movie_id = mr.movie_id
  WHERE mr.created_at >= '2020-02-01'
    AND mr.created_at <  '2020-03-01'
  GROUP BY m.title
  ORDER BY AVG(mr.rating) DESC, m.title ASC
  LIMIT 1
);
```

```
핵심:
  괄호로 각 SELECT 감싸기
    → 괄호 없으면 ORDER BY / LIMIT 이 전체에 적용됨

  컬럼 별칭 통일 (AS results)
    → UNION ALL 결과 컬럼명 = 첫 번째 쿼리 별칭

  ORDER BY + LIMIT 1 = 1등 뽑기
    COUNT(*) DESC  많은 것부터
    LIMIT 1        첫 번째 = 1등

  동점 처리
    ORDER BY COUNT(*) DESC, u.name ASC
    → 개수 같으면 이름 알파벳 순으로 결정

  날짜 범위
    >= '2020-02-01' AND < '2020-03-01'
    → BETWEEN 보다 안전 (시간 단위 누락 방지)
```

## 여러 테이블 합치기

```sql
-- 한국 + 미국 사용자 전체 목록
SELECT 'KR' AS country, user_id, name FROM users_kr
UNION ALL
SELECT 'US' AS country, user_id, name FROM users_us
ORDER BY name;
```

## 두 테이블 중 "정확히 한쪽에만" 있는 행 — UNION 활용 ⭐️⭐️

```
이번엔 UNION ALL 이 아니라 UNION (중복 제거 버전) 을 일부러 쓰는 경우
"Anti-Join 두 번 + UNION" 으로 두 테이블의 대칭차집합(symmetric difference) 을 구하는 패턴
```

```sql
-- Employees 에만 있는 id
SELECT e.employee_id
FROM Employees e
LEFT JOIN Salaries s ON e.employee_id = s.employee_id
WHERE s.employee_id IS NULL

UNION

-- Salaries 에만 있는 id
SELECT s.employee_id
FROM Salaries s
LEFT JOIN Employees e ON s.employee_id = e.employee_id
WHERE e.employee_id IS NULL

ORDER BY employee_id;
```

```bash
UNION ALL 이 아니라 UNION 을 쓰는 이유:
  이론적으로 "Employees 에만 있는 것" 과 "Salaries 에만 있는 것" 은 서로 겹칠 수 없음
  (한쪽에만 있다는 조건 자체가 상호 배타적)
  → 그래서 사실 UNION ALL 을 써도 결과는 같지만
    "혹시 모를 중복까지 안전하게 걸러낸다" 는 관례로 UNION 을 쓰기도 함
  → 데이터가 매우 크고 중복 가능성이 100% 없다고 확신하면 UNION ALL 로 바꿔도 무방 (더 빠름)

# 각 SELECT 가 하는 일 → [[SQL_JOIN]] 의 "Anti-Join" 섹션 참고
  (LEFT JOIN 후 WHERE 오른쪽.id IS NULL = "매칭 안 된 행만 찾기")
  이 패턴을 양방향으로 한 번씩 적용해서 UNION 으로 합친 것
```

## OR 조건 → UNION ALL ⭐️

```sql
-- ❌ OR JOIN (인덱스 못 탐 → 느림)
SELECT * FROM A
JOIN B ON A.id = B.a_id OR A.id = B.b_id;

-- ✅ UNION ALL 로 분리 (각각 인덱스 사용)
SELECT * FROM A JOIN B ON A.id = B.a_id
UNION ALL
SELECT * FROM A JOIN B ON A.id = B.b_id;
```

---

---

# 다중 JOIN — 체인 JOIN

```sql
-- 3개 이상 테이블 연결
SELECT u.name, m.title, mr.rating
FROM Users u
JOIN MovieRating mr ON u.user_id  = mr.user_id
JOIN Movies m       ON mr.movie_id = m.movie_id;
-- Users → MovieRating(연결) → Movies(연결)
```

---

---

# 한눈에

|패턴|코드|
|---|---|
|결과 세로 합치기 (중복 포함)|`UNION ALL`|
|결과 세로 합치기 (중복 제거)|`UNION`|
|각 쿼리에 ORDER BY + LIMIT|`(SELECT ... LIMIT 1) UNION ALL (SELECT ... LIMIT 1)`|
|1등 뽑기|`ORDER BY 기준 DESC LIMIT 1`|
|동점 처리|`ORDER BY 기준 DESC, 이름 ASC`|
|빈 카테고리 포함 집계|`UNION ALL + COUNT(*)`|
|OR JOIN 대체|`UNION ALL 로 분리`|
|와이드 컬럼 → 롱 행 변환 (Unpivot) ⭐️|`'라벨' AS col` 을 컬럼마다 만들어 `UNION ALL`|
|두 테이블 중 한쪽에만 있는 것|양방향 Anti-Join + `UNION` ([[SQL_JOIN]] 참고)|

```
JOIN vs UNION:
  JOIN  → 가로 합침 (컬럼 추가)
  UNION → 세로 합침 (행 추가)

UNION ALL 이 UNION 보다 빠름 → 기본 사용
괄호로 감싸야 각 SELECT 에 ORDER BY / LIMIT 적용

UNION ALL 은 "다른 테이블" 합칠 때도, "같은 테이블의 다른 컬럼" 합칠 때(Unpivot)도 둘 다 사용
'문자열' AS 컬럼명 → 그 묶음 전체에 라벨을 붙이는 리터럴 (실제 컬럼 값이 아님)
```