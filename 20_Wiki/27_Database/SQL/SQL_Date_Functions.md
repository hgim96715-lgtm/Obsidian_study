---
aliases:
  - 날짜 함수
  - DATE_FORMAT
  - TO_CHAR
  - Date Functions
tags:
  - SQL
related:
  - "[[00_SQL_HomePage]]"
  - "[[SQL_GROUP_BY]]"
  - "[[SQL_WHERE]]"
  - "[[SQL_Functions_Basic]]"
  - "[[SQL_Pattern_날짜범위]]"
  - "[[00_SQL_HomePage]]"
---
# SQL_Date_Functions — 날짜 함수

```txt
날짜 데이터를 쪼개고 / 더하고 / 포맷 변환
월별 집계 / 날짜 범위 필터 / 날짜 차이 계산
```

---

---

# 현재 날짜 / 시간

```sql
-- PostgreSQL
NOW()           -- 날짜 + 시간 (2024-01-27 14:30:00)
CURRENT_DATE    -- 날짜만     (2024-01-27)
CURRENT_TIME    -- 시간만     (14:30:00)

-- MySQL
NOW()           -- 날짜 + 시간
CURDATE()       -- 날짜만
CURTIME()       -- 시간만
```

---

---

# 날짜 → 문자열 포맷 변환 ⭐️

## PostgreSQL — TO_CHAR

```sql
TO_CHAR(trans_date, 'YYYY-MM')               -- '2024-01'
TO_CHAR(trans_date, 'YYYY-MM-DD')            -- '2024-01-27'
TO_CHAR(trans_date, 'YYYY-MM-DD HH24:MI:SS') -- '2024-01-27 14:30:00'
TO_CHAR(trans_date, 'YYYY년 MM월')           -- '2024년 01월'
```

## MySQL — DATE_FORMAT

```sql
DATE_FORMAT(trans_date, '%Y-%m')              -- '2024-01'
DATE_FORMAT(trans_date, '%Y-%m-%d')           -- '2024-01-27'
DATE_FORMAT(trans_date, '%Y-%m-%d %H:%i:%s')  -- '2024-01-27 14:30:00'
```

```txt
포맷 코드 비교:
  PostgreSQL   MySQL   의미
  YYYY         %Y      4자리 연도
  MM           %m      2자리 월
  DD           %d      2자리 일
  HH24         %H      24시간
  MI           %i      분
  SS           %s      초
```

---

---

# 날짜 추출 — EXTRACT / YEAR() ⭐️

```sql
-- PostgreSQL
EXTRACT(YEAR  FROM trans_date)    -- 2024
EXTRACT(MONTH FROM trans_date)    -- 1
EXTRACT(DAY   FROM trans_date)    -- 27
EXTRACT(DOW   FROM trans_date)    -- 요일 (0:일 ~ 6:토)
EXTRACT(HOUR  FROM trans_date)    -- 시간
EXTRACT(EPOCH FROM trans_date)    -- Unix 타임스탬프 (초)

-- MySQL
YEAR(trans_date)      -- 2024
MONTH(trans_date)     -- 1
DAY(trans_date)       -- 27
HOUR(trans_date)      -- 시간
DAYOFWEEK(trans_date) -- 요일 (1:일 ~ 7:토)
```

```txt
GROUP BY 에서 연도·월 추출:
  EXTRACT(YEAR FROM date)   → 숫자 (2024)
  TO_CHAR(date, 'YYYY-MM')  → 문자열 '2024-01'

  월별 집계 시 TO_CHAR / DATE_FORMAT 이 더 편함
  → 연도 + 월을 하나의 문자열로 GROUP BY 가능
```

---

---

# DATE_TRUNC — 날짜 자르기 (PostgreSQL) ⭐️

```sql
DATE_TRUNC('month', trans_date)          -- 2024-01-01 00:00:00
DATE_TRUNC('month', trans_date)::DATE    -- 2024-01-01
DATE_TRUNC('year',  trans_date)::DATE    -- 2024-01-01
DATE_TRUNC('week',  trans_date)::DATE    -- 해당 주 월요일
DATE_TRUNC('day',   trans_date)          -- 2024-01-27 00:00:00

-- GROUP BY 에서 활용
GROUP BY DATE_TRUNC('month', trans_date)
```

```txt
DATE_TRUNC vs TO_CHAR:
  DATE_TRUNC  날짜 타입 유지 → 날짜 정렬·연산 가능
  TO_CHAR     문자열로 변환 → 출력 목적

단위 종류:
  'year' / 'month' / 'week' / 'day' / 'hour' / 'minute'
```

---

---

# 날짜 연산 ⭐️

```sql
-- PostgreSQL
NOW() + INTERVAL '1 day'          -- 1일 후
NOW() - INTERVAL '1 month'        -- 1달 전
NOW() + INTERVAL '1 year'         -- 1년 후
NOW() - INTERVAL '7 days'         -- 7일 전
(NOW() - INTERVAL '7 days')::DATE -- DATE 타입으로 캐스팅

-- MySQL
DATE_ADD(trans_date, INTERVAL 1 DAY)
DATE_ADD(trans_date, INTERVAL 1 MONTH)
DATE_SUB(trans_date, INTERVAL 1 MONTH)
DATE_SUB(NOW(),      INTERVAL 7 DAY)
```

---

---

# 날짜 차이 계산 ⭐️

```sql
-- PostgreSQL
'2024-12-31'::DATE - '2024-01-01'::DATE    -- 365  (일 수)
AGE('2024-12-31', '2024-01-01')            -- 11 mons 30 days (사람이 읽기 좋은 형태)
EXTRACT(DAY FROM ('2024-12-31'::DATE - '2024-01-01'::DATE))  -- 365

-- MySQL
DATEDIFF('2024-12-31', '2024-01-01')       -- 365  (일 수)
TIMESTAMPDIFF(MONTH, '2024-01-01', '2024-12-31')  -- 11  (월 차이)
TIMESTAMPDIFF(YEAR,  '2000-01-01', '2024-01-01')  -- 24  (년 차이)
```

```txt
PostgreSQL:
  날짜 - 날짜 = 정수 (일 수)
  AGE(끝날짜, 시작날짜) = interval 타입

MySQL:
  DATEDIFF(끝날짜, 시작날짜) = 일 수
  TIMESTAMPDIFF(단위, 시작, 끝) = 단위별 차이
    단위: SECOND / MINUTE / HOUR / DAY / MONTH / YEAR
```

---

---

# 날짜 범위 필터 ⭐️

```sql
-- 특정 월 데이터
WHERE TO_CHAR(trans_date, 'YYYY-MM') = '2024-01'       -- PostgreSQL
WHERE DATE_FORMAT(trans_date, '%Y-%m') = '2024-01'     -- MySQL

-- 최근 7일
WHERE trans_date >= NOW() - INTERVAL '7 days'           -- PostgreSQL
WHERE trans_date >= DATE_SUB(NOW(), INTERVAL 7 DAY)    -- MySQL

-- 특정 기간
WHERE trans_date BETWEEN '2024-01-01' AND '2024-01-31'

-- 이번 달
WHERE DATE_TRUNC('month', trans_date) = DATE_TRUNC('month', NOW())  -- PostgreSQL
WHERE MONTH(trans_date) = MONTH(NOW()) AND YEAR(trans_date) = YEAR(NOW()) -- MySQL
```

---

---

# 기타 유용한 함수

```sql
-- PostgreSQL
TO_DATE('2024-01-27', 'YYYY-MM-DD')   -- 문자열 → DATE 변환

-- MySQL
STR_TO_DATE('2024-01-27', '%Y-%m-%d') -- 문자열 → DATE 변환
LAST_DAY('2024-02-01')                 -- 해당 월의 마지막 날 (2024-02-29)
DAYNAME(trans_date)                    -- 요일 이름 ('Monday')
```

---

---

# 함수 한눈에

|목적|PostgreSQL|MySQL|
|---|---|---|
|포맷 변환|`TO_CHAR(date, 'YYYY-MM')`|`DATE_FORMAT(date, '%Y-%m')`|
|연도 추출|`EXTRACT(YEAR FROM date)`|`YEAR(date)`|
|월 추출|`EXTRACT(MONTH FROM date)`|`MONTH(date)`|
|날짜 더하기|`date + INTERVAL '1 day'`|`DATE_ADD(date, INTERVAL 1 DAY)`|
|날짜 빼기|`date - INTERVAL '1 day'`|`DATE_SUB(date, INTERVAL 1 DAY)`|
|날짜 자르기|`DATE_TRUNC('month', date)`|`DATE_FORMAT(date, '%Y-%m-01')`|
|날짜 차이|`date1 - date2` (일 수)|`DATEDIFF(date1, date2)`|
|단위별 차이|`EXTRACT(DAY FROM date1-date2)`|`TIMESTAMPDIFF(DAY, date1, date2)`|
|경과 시간|`AGE(date1, date2)`|`TIMESTAMPDIFF(MONTH, ...)`|
|문자열→날짜|`TO_DATE('str', 'YYYY-MM-DD')`|`STR_TO_DATE('str', '%Y-%m-%d')`|
|마지막 날|-|`LAST_DAY(date)`|