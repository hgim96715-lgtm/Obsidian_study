---
aliases:
  - WHERE절
  - 필터링
  - BETWEEN
  - IN
  - LIKE
  - IS NULL
  - 조건절
  - 비교연산자
  - ILIKE
tags:
  - SQL
related:
  - "[[00_SQL_HomePage]]"
  - "[[SQL_Pattern_날짜범위]]"
  - "[[SQL_GROUP_BY]]"
  - "[[PG_Specific]]"
  - "[[MySQL_Specific]]"
---
# SQL_WHERE — 조건 필터링

# 한 줄 요약

```
WHERE = 조건에 맞는 행만 뽑아내기
엑셀의 필터 기능과 동일
```

---

---

# 비교 연산자 ⭐️

|연산자|의미|예시|
|---|---|---|
|`=`|같다|`age = 20`|
|`>` `<`|크다 / 작다|`price > 1000`|
|`>=` `<=`|크거나 같다 / 작거나 같다|`score <= 50`|
|`<>`|다르다 (표준 SQL) ⭐️|`city <> 'Seoul'`|
|`!=`|다르다 (대부분 지원)|`city != 'Seoul'`|

---

---

# AND / OR / NOT ⭐️

## 연산자 우선순위 ⭐️

```
1순위: ( )
2순위: 비교 연산자 (= > < >= <=)
3순위: NOT
4순위: AND
5순위: OR   ← AND 보다 나중에 계산됨

A OR B AND C  →  A OR (B AND C)
→ 섞어 쓸 땐 반드시 괄호 ()
```

```sql
-- 괄호 없을 때 (의도와 다를 수 있음)
WHERE city = 'Seoul' OR city = 'Busan' AND age > 20
-- → city = 'Seoul' OR (city = 'Busan' AND age > 20)

-- 명확하게 괄호로 묶기
WHERE (city = 'Seoul' OR city = 'Busan') AND age > 20
```

## 드 모르간의 법칙

```
NOT (A OR B)  →  NOT A AND NOT B
NOT (A AND B) →  NOT A OR  NOT B
```

```sql
-- NULL 포함 행 제거 (두 방법 동일)
WHERE NOT (species IS NULL OR body_mass_g IS NULL)
WHERE species IS NOT NULL AND body_mass_g IS NOT NULL
```

---

---

# BETWEEN — 범위 조건 ⭐️

```
양 끝값 모두 포함 (Inclusive)
A BETWEEN B AND C  =  B <= A <= C
```

```sql
WHERE age BETWEEN 10 AND 20        -- 10, 20 포함
WHERE age NOT BETWEEN 10 AND 20

WHERE price BETWEEN 100 AND 300    -- 숫자
WHERE created_at BETWEEN '2026-01-01' AND '2026-12-31'  -- 날짜
```

## 날짜 사용 시 주의 ⚠️

```sql
-- ❌ DATETIME 타입이면 마지막 시간 누락 위험
WHERE created_at BETWEEN '2026-03-04' AND '2026-03-05'

-- ✅ >= AND < 권장
WHERE created_at >= '2026-03-04' AND created_at < '2026-03-06'
```

## 값 BETWEEN 컬럼1 AND 컬럼2 ⭐️

```sql
-- 일반 (컬럼이 기준)
WHERE price BETWEEN 100 AND 300
-- = price >= 100 AND price <= 300

-- 값이 기준 (반대처럼 보임)
WHERE 200 BETWEEN col1 AND col2
-- = col1 <= 200 AND col2 >= 200
-- "200 이 col1 ~ col2 사이에 있는가?"
```

---

---

# IN — 집합 조건 ⭐️

```sql
-- OR 여러 번 대신 IN 으로 간결하게
WHERE city IN ('Seoul', 'Busan', 'Daegu')
-- = WHERE city = 'Seoul' OR city = 'Busan' OR city = 'Daegu'

WHERE city NOT IN ('Seoul', 'Busan')
```

## NOT IN + NULL 함정 ⚠️

```sql
-- NOT IN 목록에 NULL 있으면 결과 0건
WHERE id NOT IN (1, 2, NULL)
-- NULL 과의 비교 = UNKNOWN → 모든 행 탈락

-- 해결: NULL 명시 제거
WHERE id NOT IN (SELECT id FROM t WHERE id IS NOT NULL)
```

---

---

# LIKE / ILIKE — 패턴 매칭 ⭐️

```
JS 의 startsWith / endsWith / includes 에 해당하는 SQL 문법

name.startsWith('M')  →  name LIKE 'M%'
name.endsWith('수')   →  name LIKE '%수'
name.includes('길')   →  name LIKE '%길%'
```

```sql
WHERE name LIKE 'M%'        -- M으로 시작 (startsWith)
WHERE name LIKE '%수'        -- 수로 끝남  (endsWith)
WHERE name LIKE '%길%'       -- 길 포함    (includes)
WHERE name LIKE '김_수'      -- 김X수 (중간 딱 1글자)
WHERE name NOT LIKE '%테스트%'
```

```
%  모든 문자 (0글자 이상)
_  딱 한 글자

LIKE   대소문자 구분 (모든 DB)
ILIKE  대소문자 무시 (PostgreSQL 전용)
```

## LIKE 를 CASE WHEN 안에서 ⭐️

```sql
-- 홀수 employee_id 이면서 이름이 M 으로 시작하지 않으면 salary
-- 그 외는 0
SELECT
  employee_id,
  CASE
    WHEN employee_id % 2 = 1 AND name NOT LIKE 'M%' THEN salary
    ELSE 0
  END AS bonus
FROM Employees
ORDER BY employee_id;
```

```sql
-- 같은 결과 — 조건을 쪼개서 쓰는 방법
SELECT
  employee_id,
  CASE
    WHEN employee_id % 2 = 0 THEN 0        -- 짝수면 0
    WHEN name LIKE 'M%'      THEN 0        -- M 으로 시작하면 0
    ELSE salary                            -- 나머지는 salary
  END AS bonus
FROM Employees
ORDER BY employee_id;
```

```
두 방법 비교:
  첫 번째 (AND 조합):
    한 WHEN 에 조건 두 개 묶기
    "홀수이면서 M 으로 안 시작하면" → 직관적

  두 번째 (조건 분리):
    WHEN 을 여러 개로 나누기
    조건 하나씩 처리 → 디버깅 쉬움
    같은 결과 / 취향 차이

주의 — JS 방식은 SQL 에서 안 됨:
  name.startsWith('M')   ❌ JS 메서드
  name LIKE 'M%'         ✅ SQL 방식
```

## ESCAPE — 진짜 % · _ 검색

```sql
-- '%50%' 문자열 자체를 찾고 싶을 때
WHERE review LIKE '%50\%%' ESCAPE '\'
--                  ↑ \% = 진짜 % (와일드카드 아님)

-- PostgreSQL
WHERE page_location ILIKE '%\_%' ESCAPE '\'
```

## WHERE 에 LIKE 쓸 때 주의 ⚠️

```sql
-- ❌ WHERE 로 필터링하면 분모가 줄어듦 (비율 계산 시 문제)
SELECT COUNT(*) FROM payments WHERE credit ILIKE '%gift%'

-- ✅ 전체 유지하고 CASE WHEN 으로 조건부 집계
SELECT
  SUM(CASE WHEN credit ILIKE '%gift%' THEN 1 ELSE 0 END) AS gift_count,
  COUNT(*) AS total_count
FROM payments
```

---

---

# IS NULL ⭐️

```
NULL = "알 수 없음"
= / <> 로 비교 불가
```

```sql
-- ❌ 절대 안 됨
WHERE age = NULL      -- 항상 FALSE
WHERE age <> NULL     -- 항상 FALSE

-- ✅ IS NULL / IS NOT NULL
WHERE age IS NULL
WHERE age IS NOT NULL
```

---

---

# 정규식 패턴 매칭 ⭐️

```sql
-- PostgreSQL — ~ 연산자
WHERE mail ~ '^[A-Za-z][A-Za-z0-9._-]*@leetcode[.]com$'
WHERE name ~* '^(A|B)'    -- ~* = 대소문자 무시

-- MySQL — REGEXP_LIKE
WHERE REGEXP_LIKE(mail, '^[A-Za-z][A-Za-z0-9._-]*@leetcode[.]com$', 'c')
-- 'c' = case-sensitive
```

---

---

# WHERE 절 제한사항

```
실행 순서: FROM → WHERE → SELECT
WHERE 가 SELECT 보다 먼저 실행됨

별칭(Alias) 사용 불가:
  ❌ WHERE total > 1000      (total 이 아직 없음)
  ✅ WHERE price * qty > 1000

집계 함수 사용 불가:
  ❌ WHERE AVG(salary) > 5000
  ✅ HAVING AVG(salary) > 5000
```

---

---

# 실수 체크리스트

|실수|해결|
|---|---|
|NULL 찾을 때 `=` 사용|`IS NULL` 사용|
|AND·OR 혼용 시 괄호 없음|괄호 `()` 필수|
|BETWEEN 끝값 제외 착각|양 끝값 모두 포함|
|`값 BETWEEN COL1 AND COL2` 오해|`COL1 <= 값 AND 값 <= COL2`|
|NOT IN 에 NULL 포함|`IS NOT NULL` 로 제거|
|JS 메서드로 문자열 검색 시도|`LIKE 'M%'` 사용|
|WHERE 에 별칭 사용|원본 컬럼으로 계산|
|WHERE 에 집계 함수|HAVING 사용|