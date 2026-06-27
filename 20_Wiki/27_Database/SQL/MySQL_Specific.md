---
aliases:
  - MySQL 전용
  - REGEXP_LIKE
  - GROUP_CONCAT
  - (^| ) 패턴
tags:
  - SQL
related:
  - "[[00_SQL_HomePage]]"
  - "[[SQL_WHERE]]"
  - "[[PG_Specific]]"
---

# MySQL_Specific — MySQL 전용 기능

---

---

# REGEXP_LIKE — 정규식 패턴 매칭 ⭐️

```sql
-- REGEXP_LIKE(컬럼, 패턴, 플래그)
SELECT user_id, name, mail
FROM Users
WHERE REGEXP_LIKE(
  mail,
  '^[A-Za-z][A-Za-z0-9._-]*@leetcode[.]com$',
  'c'   -- 'c' = case-sensitive (대소문자 구분)
);
```

```txt
플래그:
  'c'  case-sensitive  대소문자 구분
  'i'  case-insensitive 대소문자 무시

왜 'c' 를 명시하나:
  MySQL collation 설정에 따라 기본이 대소문자 무시일 수 있음
  소문자 도메인이 정확히 맞아야 하는 경우 'c' 명시 필수
  → 'c' 없으면 LeetCode.com / LEETCODE.COM 도 통과될 수 있음
```

## (^| ) 패턴 — 단어 시작 찾기 ⭐️

```sql
-- DIAB1 로 시작하거나, 공백 바로 뒤에 DIAB1 이 오는 경우
WHERE REGEXP_LIKE(conditions, '(^| )DIAB1', 'c')

-- 예시:
-- 'DIAB100 HEART2'  → 맨 앞 DIAB1 → 매칭 ✅
-- 'HEART2 DIAB100'  → 공백 뒤 DIAB1 → 매칭 ✅
-- 'ADIAB100'        → 앞에 문자 있음 → 매칭 안 됨 ❌
```

```txt
(^| ) 의미:
  ^ 또는 공백( ) 중 하나
  → 단어의 시작 위치에 패턴이 오는지 확인

왜 LIKE '%DIAB1%' 로 안 되나:
  'ADIAB100' 처럼 앞에 다른 문자 붙어도 매칭됨
  → 정확히 단어 시작 위치만 찾으려면 정규식 필요
```

---

---

#  IFNULL — NULL 대체값

```sql
-- IFNULL(값, NULL 일 때 대체값)
SELECT IFNULL(salary, 0) FROM employees;
-- salary 가 NULL 이면 0 반환

-- PostgreSQL 의 COALESCE 와 동일
SELECT COALESCE(salary, 0) FROM employees;
```

---

---

#  DATE_FORMAT — 날짜 포맷

```sql
-- DATE_FORMAT(날짜, 형식)
SELECT DATE_FORMAT(created_at, '%Y-%m') AS month FROM orders;
-- '2024-01'

SELECT DATE_FORMAT(trans_date, '%Y-%m') AS month
FROM Transactions
GROUP BY DATE_FORMAT(trans_date, '%Y-%m');
```

```txt
주요 형식:
  %Y  4자리 연도 (2024)
  %m  2자리 월 (01~12)
  %d  2자리 일 (01~31)
  %H  시간 (00~23)
  %i  분
  %s  초

PostgreSQL: TO_CHAR(날짜, 'YYYY-MM')
MySQL:      DATE_FORMAT(날짜, '%Y-%m')
```

---

---

#  GROUP_CONCAT — 문자열 집계

```sql
SELECT
  sell_date,
  GROUP_CONCAT(DISTINCT product ORDER BY product SEPARATOR ',') AS products
FROM Activities
GROUP BY sell_date;
```

```txt
GROUP_CONCAT(DISTINCT 컬럼 ORDER BY 정렬 SEPARATOR '구분자')
PostgreSQL 의 STRING_AGG 와 동일
```

---

---

#  SUM(조건) 단축 패턴

```sql
-- MySQL 에서 boolean 이 0/1 로 처리됨
SELECT SUM(rating < 3) AS low_rating_count
FROM Reviews;
-- rating < 3 이 참이면 1, 거짓이면 0 → SUM 으로 카운트

-- PostgreSQL 에서는 FILTER 사용
SELECT COUNT(*) FILTER (WHERE rating < 3)
FROM Reviews;
```