---
aliases:
  - SQL 함수
  - COALESCE
  - ROUND
  - NOW
tags:
  - SQL
related:
  - "[[00_SQL_HomePage]]"
  - "[[SQL_SELECT]]"
  - "[[SQL_Aggregate]]"
  - "[[PG_Specific]]"
  - "[[SQL_Date_Functions]]"
---
# SQL_Functions_Basic — 기본 함수

```txt
자주 쓰는 내장 함수 모음
문자열 / 숫자 / 날짜 / NULL 처리
```

---

---

# 문자열 함수 ⭐️

## 대소문자

```sql
UPPER('hello')       -- 'HELLO'
LOWER('HELLO')       -- 'hello'
```

## 길이

```sql
LENGTH('hello')      -- 5 (바이트 단위 / 한글은 3)
CHAR_LENGTH('hello') -- 5 (문자 수 기준 / MySQL 에서 한글 안전)
```

## 자르기 — LEFT / RIGHT / SUBSTRING ⭐️

```sql
LEFT('abcdef', 3)          -- 'abc'  왼쪽에서 3글자
RIGHT('abcdef', 3)         -- 'def'  오른쪽에서 3글자

SUBSTRING('abcdef', 2)     -- 'bcdef'  2번째부터 끝까지
SUBSTRING('abcdef', 2, 3)  -- 'bcd'   2번째부터 3글자
--                  ↑ ↑
--                시작  길이
```

```txt
SUBSTRING(문자열, 시작위치, 길이):
  시작위치는 1부터 (0 아님)
  길이 생략하면 끝까지
  PostgreSQL / MySQL 둘 다 동일
```


## SUBSTR — 문자열 자르기 ⭐️

```sql
-- SUBSTR = SUBSTRING 의 별칭 (PostgreSQL / MySQL 둘 다 동일하게 사용 가능)
SUBSTR('abcdef', 2)        -- 'bcdef'   2번째부터 끝까지
SUBSTR('abcdef', 2, 3)     -- 'bcd'    2번째부터 3글자
```

```txt
SUBSTRING vs SUBSTR:
  동일한 기능 / 이름만 다름
  SUBSTRING → SQL 표준
  SUBSTR    → 오라클 스타일 / 둘 다 쓸 수 있음
```

## 합치기 — CONCAT vs || ⭐️

```sql
-- MySQL
CONCAT('Hello', ' ', 'World')        -- 'Hello World'
CONCAT(UPPER(LEFT(name,1)), LOWER(SUBSTRING(name,2)))

-- PostgreSQL
'Hello' || ' ' || 'World'           -- 'Hello World'
UPPER(LEFT(name,1)) || LOWER(SUBSTRING(name,2))

-- PostgreSQL 에서 CONCAT 도 사용 가능
CONCAT('Hello', ' ', 'World')       -- 'Hello World'
```

```txt
||  PostgreSQL 문자열 연결 연산자
    NULL 이 하나라도 있으면 결과 NULL

CONCAT()  NULL 을 빈 문자열로 처리
    → NULL 포함 시 CONCAT 이 더 안전

PostgreSQL 에서:
  || 와 CONCAT() 둘 다 사용 가능
  실무에서는 || 를 더 많이 씀

MySQL 에서:
  || 는 OR 연산자 (문자열 연결 아님)
  → 반드시 CONCAT() 사용
```

## 실전 예제 — 이름 첫 글자 대문자 ⭐️

```sql
-- MySQL
SELECT
    user_id,
    CONCAT(
        UPPER(LEFT(name, 1)),
        LOWER(SUBSTRING(name, 2))
    ) AS name
FROM Users
ORDER BY user_id;

-- PostgreSQL
SELECT
    user_id,
    UPPER(LEFT(name, 1)) || LOWER(SUBSTRING(name, 2)) AS name
FROM Users
ORDER BY user_id;
```

```txt
패턴 분석:
  LEFT(name, 1)         첫 글자 추출
  UPPER(...)            대문자로
  SUBSTRING(name, 2)    2번째 글자부터 끝까지
  LOWER(...)            소문자로
  합치기                CONCAT (MySQL) / || (PostgreSQL)

결과: 'alice' → 'Alice'
```

## 공백 제거 TRIM

```sql
-- 공백 제거 (모든 DB 공통)
TRIM('  hello  ')        -- 'hello'   양쪽 제거
LTRIM('  hello  ')       -- 'hello  ' 왼쪽 제거
RTRIM('  hello  ')       -- '  hello' 오른쪽 제거


-- TRIM 방향 지정 (표준 문법)
SELECT TRIM(LEADING  'x' FROM 'xxSQLxx');       -- 결과: 'SQLxx'  (왼쪽만)
SELECT TRIM(TRAILING 'x' FROM 'xxSQLxx');       -- 결과: 'xxSQL'  (오른쪽만)
SELECT TRIM(BOTH     'x' FROM 'xxSQLxx');       -- 결과: 'SQL'    (양쪽)
```

## 빈자리 채우기 — LPAD / RPAD ⭐️

```sql
LPAD('42', 5, '0')      -- '00042'   왼쪽을 0으로 채워 5자리
RPAD('42', 5, '0')      -- '42000'   오른쪽을 0으로 채워 5자리

LPAD('hi', 5, '*')      -- '***hi'   왼쪽을 * 로 채우기
RPAD('hi', 5, '-')      -- 'hi---'   오른쪽을 - 로 채우기
```

```txt
LPAD(문자열, 목표길이, 채울문자):
  문자열이 목표길이보다 짧으면 왼쪽에 채울문자 추가
  이미 목표길이 이상이면 그대로 반환

주로 쓰는 곳:
  상품 코드  LPAD(product_id, 8, '0') → '00000042'
  사원 번호  LPAD(emp_no, 6, '0')     → '001234'
  자릿수를 일정하게 맞춰야 할 때
```


## 대체 REPLACE

```sql
-- 문법: REPLACE(문자열, 찾을문자, 바꿀문자)
REPLACE('hello world', 'world', 'SQL')   -- 'hello SQL'

-- 삭제 (PostgreSQL · SQL Server: 빈 문자열 '' 명시 필수)
SELECT REPLACE('010-1234-5678', '-', '');    -- 결과: '01012345678'
```


## SPLIT — 문자열 쪼개기

```sql
-- PostgreSQL
STRING_TO_ARRAY('a,b,c', ',')           -- '{a,b,c}'  배열로 반환
SPLIT_PART('a,b,c', ',', 2)             -- 'b'  N번째 조각만 반환
--                           ↑ 1부터 시작

-- MySQL
SUBSTRING_INDEX('a,b,c', ',', 2)        -- 'a,b'   앞에서 N개
SUBSTRING_INDEX('a,b,c', ',', -1)       -- 'c'     뒤에서 1개
```

```txt
SPLIT_PART(문자열, 구분자, N):
  PostgreSQL 전용
  N번째 조각 반환 (1부터 시작)
  SPLIT_PART('2024-06-05', '-', 2) → '06'

SUBSTRING_INDEX(문자열, 구분자, N):
  MySQL 전용
  양수 N → 왼쪽에서 N개 구분자까지
  음수 N → 오른쪽에서 N개 구분자까지
```

## STRING_AGG — 여러 행을 하나로 합치기 ⭐️


```sql
-- PostgreSQL
SELECT STRING_AGG(name, ', ')           -- 'Alice, Bob, Charlie'
FROM users;

SELECT STRING_AGG(name, ', ' ORDER BY name)  -- 정렬 후 합치기
FROM users;

-- MySQL
SELECT GROUP_CONCAT(name SEPARATOR ', ')    -- 'Alice, Bob, Charlie'
FROM users;

SELECT GROUP_CONCAT(name ORDER BY name SEPARATOR ', ')
FROM users;
```

```txt
STRING_AGG(컬럼, 구분자):     PostgreSQL
GROUP_CONCAT(컬럼 SEPARATOR): MySQL

활용:
  태그 목록 합치기   'action, drama, thriller'
  이름 목록 합치기   '홍길동, 김철수, 이영희'
  GROUP BY 와 함께 그룹별 값 목록 만들기
```

## CHR / ASCII — 문자 ↔ 코드 변환

```sql
-- 코드 → 문자
CHR(65)          -- 'A'   (PostgreSQL)
CHAR(65)         -- 'A'   (MySQL)

-- 문자 → 코드
ASCII('A')       -- 65    (PostgreSQL / MySQL 공통)
ASCII('a')       -- 97
```

```txt
주로 쓰는 곳:
  특수문자를 안전하게 삽입할 때
    CHR(10)  → 줄바꿈 (\n)
    CHR(9)   → 탭 (\t)

  문자 비교 / 암호화 로직
  A-Z: 65-90 / a-z: 97-122 / 0-9: 48-57

PostgreSQL: CHR()
MySQL:      CHAR()
둘 다:      ASCII()
```

---

---

# 숫자 함수

## 반올림 / 버림

```sql
-- ROUND(숫자, 자릿수)
ROUND(3.456, 2)        -- 3.46    소수점 2자리 반올림
ROUND(123.4567, -1)    -- 120     1의 자리에서 반올림 (음수 = 정수 자리)

-- TRUNC(숫자, 자릿수) — 반올림 없이 잘라냄
TRUNC(123.4567, 2)     -- 123.45  소수점 2자리 이하 버림
TRUNC(123.4567, -1)    -- 120     1의 자리 이하 버림
```

```txt
ROUND vs TRUNC:
  ROUND  반올림 포함
  TRUNC  그냥 잘라냄 (버림)

  ROUND(123.4567, -1) = 120  (1의 자리 반올림 → 4 이므로 120)
  TRUNC(123.4567, -1) = 120  (1의 자리 이하 무조건 버림)

자릿수 음수:
  -1 → 1의 자리
  -2 → 10의 자리
  -3 → 100의 자리
```

## 올림 / 내림 / 절댓값/부호 판별

```sql
CEIL(3.1)      -- 4      올림
FLOOR(3.9)     -- 3      내림
ABS(-5)        -- 5      절댓값
SIGN(-25)      -- -1     양수: 1 / 0: 0 / 음수: -1
```

## 나머지 / 최솟값 / 최댓값

```sql
MOD(10, 3)              -- 1     나머지 (10 % 3)

LEAST(10, 5, 8)         -- 5     여러 값 중 최솟값
GREATEST(10, 5, 8)      -- 10    여러 값 중 최댓값
```

```txt
LEAST / GREATEST:
  인자 여러 개 비교 (컬럼도 가능)
  LEAST(price, discount_price)  → 더 낮은 가격 선택
```

## ⚠️ 정수 나눗셈 함정 (PostgreSQL) ⭐️

```sql
SELECT 7 / 2;      -- 3   (소수점 버림)
SELECT 7.0 / 2;    -- 3.5 (정상)
SELECT 7 / 2.0;    -- 3.5 (정상)
```

```bash
PostgreSQL 에서 정수 / 정수 = 정수
소수점 이하 그냥 버림 (파이썬 // 연산자와 동일)
→ 소수 결과가 필요하면 한쪽을 실수로 캐스팅

CAST(7 AS FLOAT) / 2     -- 3.5
7::float / 2             -- 3.5 (PostgreSQL 캐스팅)

# → [[PG_Specific]] 참고
```

---
---

# NULL 처리 함수 ⭐

## COALESCE

```txt
COALESCE(값1, 값2, 값3, ...)
  왼쪽부터 순서대로 보고 처음으로 NULL 이 아닌 값 반환
  전부 NULL 이면 → NULL 반환

Oracle · PostgreSQL · SQL Server 모두 지원
```

```sql
-- 값이 NULL 이면 대체값 반환 
COALESCE(NULL, 'default')       -- 'default'
COALESCE(NULL, NULL, 'third')   -- 'third'  첫 번째 non-NULL
SELECT COALESCE(NULL, NULL);             -- NULL (전부 NULL 이면 NULL
```

## IFNULL — MySQL 전용

```txt
IFNULL(값, NULL일때대체값)
```

```sql
-- MySQL 전용
IFNULL(NULL, 'default')         -- 'default'
```

## NULLIF — 같으면 NULL 로 (0 나눗셈 방지)

```txt
NULLIF(값1, 값2)
  값1 = 값2  →  NULL 반환
  값1 ≠ 값2  →  값1 그대로 반환

주 용도: 분모가 0 일 때 Division by Zero 에러 방지
```

```sql
-- 두 값이 같으면 NULL 반환
NULLIF(5, 5)    -- NULL
NULLIF(5, 3)    -- 5
```

---

---

# 날짜 함수

```bash
# → [[SQL_Date_Functions]] 참고
NOW() / CURRENT_DATE / EXTRACT / TO_CHAR / DATE_TRUNC / INTERVAL / DATEDIFF
```

---
---
# MySQL vs PostgreSQL 문자열 차이 ⭐️

|기능|MySQL|PostgreSQL|
|---|---|---|
|문자열 연결|`CONCAT(a, b)`|`a \| b` 또는 `CONCAT(a, b)`|
|`\|` 의미|OR 연산자 ⚠️|문자열 연결|
|대소문자 무시 검색|`LIKE` (기본 대소문자 무시)|`ILIKE`|
|정규식|`REGEXP_LIKE(col, 'pat')`|`col ~ 'pat'`|
|NULL 대체|`IFNULL(col, 'default')`|`COALESCE(col, 'default')`|
|날짜 포맷|`DATE_FORMAT(col, '%Y-%m')`|`TO_CHAR(col, 'YYYY-MM')`|
|문자열 집계|`GROUP_CONCAT(col)`|`STRING_AGG(col, ',')`|

```txt
⚠️ MySQL 에서 || 는 OR 연산자
   'hello' || 'world' = 0 (FALSE 로 처리)
   → MySQL 에서 문자열 연결은 반드시 CONCAT() 사용
```