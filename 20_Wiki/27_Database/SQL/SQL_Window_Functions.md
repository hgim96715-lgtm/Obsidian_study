---
aliases:
  - 윈도우 함수
  - OVER
  - PARTITION BY
  - ROW_NUMBER
  - LAG
  - RANK
  - DENSE_RANK
  - LEAD
tags:
  - SQL
related:
  - "[[00_SQL_HomePage]]"
  - "[[SQL_Subquery]]"
  - "[[SQL_CASE_WHEN]]"
  - "[[SQL_Functions_Basic]]"
  - "[[SQL_GROUP_BY]]"
---
# SQL_Window_Functions — 윈도우 함수

# 한 줄 요약

```txt
GROUP BY 처럼 행을 압축하지 않고
원본 행을 유지한 채 계산 결과를 옆에 붙여주는 함수
```

---

---

# 윈도우 함수란 — 왜 쓰는가 ⭐️⭐️

## GROUP BY 의 한계

```sql
-- GROUP BY: 부서별 1줄만 남음 → 원본의 다른 컬럼(사원명 등)은 사라짐
SELECT 부서, AVG(급여) FROM 사원 GROUP BY 부서;
```

```txt
GROUP BY 는 "압축" 임:
  같은 부서 행들을 하나로 합치는 순간, 그 행들 각각이 갖고 있던
  사원명 / 입사일 같은 다른 컬럼 정보는 같이 사라짐
  → "각 사원 정보는 그대로 두고, 부서 평균만 옆에 알고 싶다" 는 요청에는 안 맞음
```

## 윈도우 함수의 해결 방식

```sql
-- PARTITION BY: 원본 행 유지 + 부서평균 컬럼만 추가
SELECT 사원명, 부서, 급여,
       AVG(급여) OVER (PARTITION BY 부서) AS 부서평균
FROM 사원;
```

```txt
GROUP BY     → 행을 합쳐서 요약 (행 수 줄어듦, 다른 컬럼 소실)
PARTITION BY → 원본 유지 + 계산값만 추가 (행 수 그대로, 다른 컬럼 살아있음)
```

## 실전 — 막혔던 지점 (Employee Primary Department) ⭐️⭐️

```txt
문제: 직원별 "대표 부서" 한 줄만 뽑기
  부서가 1개뿐이면 그 부서가 정답
  부서가 여러 개면 primary_flag = 'Y' 인 부서가 정답

Employee 테이블:
  employee_id  department_id  primary_flag
  1            1              Y
  2            1              Y
  2            2              N
  3            1              N
  3            2              N
  3            3              Y
```

```txt
처음 막혔던 고민: "GROUP BY 로 employee_id, department_id, primary_flag
                  3개를 다 묶어야 하나?"

→ 이게 안 되는 이유: GROUP BY (3개 컬럼) 으로 묶어봐도
  "이 employee_id 가 총 몇 개의 부서에 속해 있는지" 라는 정보 자체가
  그 묶음 안에는 들어있지 않음 (그건 employee_id 만으로 다시 세야 하는 별개의 집계임)
  → GROUP BY 만으로 풀려면 부서 개수를 먼저 따로 집계한 뒤
    원본 테이블과 다시 JOIN 해야 함 (아래 비교 참고)
```

```sql
-- GROUP BY + JOIN 방식 — 부서 개수를 따로 집계해서 다시 합쳐야 함
WITH dept_count AS (
    SELECT employee_id, COUNT(*) AS cnt
    FROM Employee
    GROUP BY employee_id
)
SELECT e.employee_id, e.department_id
FROM Employee e
JOIN dept_count d ON e.employee_id = d.employee_id
WHERE e.primary_flag = 'Y' OR d.cnt = 1;

-- 윈도우 함수 방식 — JOIN 없이 한 번의 SELECT 안에서 끝남 ⭐️
WITH cnt AS (
    SELECT employee_id, department_id, primary_flag,
           COUNT(department_id) OVER (PARTITION BY employee_id) AS department_id_cnt
           --                         ↑ employee_id 별로 "부서가 몇 개인지" 를
           --                           원본 행(department_id, primary_flag) 그대로 둔 채 옆에 붙임
    FROM Employee
)
SELECT employee_id, department_id
FROM cnt
WHERE primary_flag = 'Y' OR department_id_cnt = 1;
```

```txt
COUNT(department_id) OVER (PARTITION BY employee_id) 한 줄이 하는 일:

  PARTITION BY employee_id  → employee_id 가 같은 행끼리 하나의 "창(window)" 으로 묶음
  COUNT(department_id)      → 그 창 안의 행 개수를 셈 (= 이 직원의 부서 개수)
  → 그 결과를 "행을 합치지 않고" 모든 원본 행에 그대로 붙여줌

결과 (department_id_cnt 컬럼이 어떻게 채워지는지):
  employee_id  department_id  primary_flag  department_id_cnt
  1            1              Y             1
  2            1              Y             2   ← employee 2 는 부서 2개
  2            2              N             2
  3            1              N             3   ← employee 3 는 부서 3개
  3            2              N             3
  3            3              Y             3

WHERE primary_flag='Y' OR department_id_cnt=1
  → employee 1: 부서 1개 → department_id_cnt=1 조건으로 통과
  → employee 2: 부서 2개라 1번 조건은 안 되지만, primary_flag='Y' 인 행이 통과
  → employee 3: 부서 3개, primary_flag='Y' 인 행(department_id=3)만 통과

핵심 통찰:
  "집계값(부서 개수)은 필요한데, 원본 행의 다른 컬럼(primary_flag)도 그대로 봐야 한다"
  → 이게 바로 GROUP BY 대신 PARTITION BY 를 쓰는 전형적인 신호임
```

## 기본 문법

```sql
함수명() OVER (
    PARTITION BY 컬럼   -- 그룹 나누기 (선택)
    ORDER BY 컬럼       -- 정렬 기준   (필수 여부는 함수마다 다름)
    ROWS BETWEEN 시작 AND 끝  -- 범위 지정 (선택)
)
```

---

---

# 순위 함수 — ROW_NUMBER / RANK / DENSE_RANK ⭐️

```sql
ROW_NUMBER() OVER (ORDER BY 급여 DESC)
RANK()       OVER (ORDER BY 급여 DESC)
DENSE_RANK() OVER (ORDER BY 급여 DESC)
```

|함수|동점 처리|출력 예시|
|---|---|---|
|`ROW_NUMBER()`|무조건 고유 번호|1, 2, 3, 4|
|`RANK()`|같은 등수, 번호 건너뜀|1, 2, 2, **4**|
|`DENSE_RANK()`|같은 등수, 번호 연속|1, 2, 2, **3**|

```txt
ORDER BY 필수 — 없으면 에러
  ROW_NUMBER() OVER ()                   ← ❌ 에러
  ROW_NUMBER() OVER (ORDER BY 급여 DESC) ← ✅
```

## Top-N 뽑기 패턴 ⭐️

```sql
-- ⚠️ 윈도우 함수는 WHERE 에 바로 못 씀 → CTE 로 감싸야 함
-- (방금 위 employee 예제의 department_id_cnt 와 같은 이유)
WITH ranked AS (
    SELECT 사원명, 급여,
        DENSE_RANK() OVER (ORDER BY 급여 DESC) AS rnk
    FROM 사원
)
SELECT * FROM ranked WHERE rnk <= 3;

-- 부서별 Top-N
WITH ranked AS (
    SELECT 부서, 사원명, 급여,
        DENSE_RANK() OVER (PARTITION BY 부서 ORDER BY 급여 DESC) AS rnk
    FROM 사원
)
SELECT * FROM ranked WHERE rnk <= 3;
```

## 언제 뭘 쓰나

|상황|함수|
|---|---|
|딱 N명만 (동점 무시)|`ROW_NUMBER()`|
|동점자 같은 등수, 번호 건너뜀|`RANK()`|
|동점자 같은 등수, 번호 연속|`DENSE_RANK()`|
|순위 번호 필요 없이 개수|`COUNT(*) + GROUP BY + HAVING`|

---

---

# LAG / LEAD — 이전/다음 행 ⭐️

```sql
LAG(컬럼, N) OVER (
    PARTITION BY 그룹컬럼  -- 선택: 그룹 내에서 이전 값
    ORDER BY 정렬컬럼       -- 필수: 어떤 순서로 앞을 볼지
)

LAG(컬럼)           -- 바로 윗줄 / 없으면 NULL
LAG(컬럼, 7)        -- 7줄 위 (7일 전)
LAG(컬럼, 7, 0)     -- 7줄 위 / 없으면 0 (3번째 인자 = 기본값)
LEAD(컬럼)          -- 바로 아랫줄
```

```txt
ORDER BY 가 핵심:
  LAG 는 "정렬 기준으로 N행 앞" 을 가져옴
  ORDER BY 없으면 "앞" 이 어딘지 정의 안 됨 → 에러

그룹 내 이전 값:
  LAG(amount) OVER (PARTITION BY user_id ORDER BY visited_on)
  → user_id 별로 각자의 이전 날짜 값 (지역/사용자 바뀌면 초기화)
```

```sql
-- 전일 대비 증감
SELECT 날짜, 매출,
    LAG(매출) OVER (ORDER BY 날짜) AS 전일매출,
    매출 - LAG(매출) OVER (ORDER BY 날짜) AS 증감
FROM 매출;
```

## LAG 로 연속성 판단 패턴 ⭐️

```txt
고민: "7일이 연속인지 어떻게 확인하지?"

❌ COUNT 로는 안 됨 — "2번 이상 등장" ≠ "연속해서 등장"
  HAVING COUNT(*) >= 2

✅ 방법 1 — 바로 이전 행과 간격 비교
✅ 방법 2 — N행 앞의 값 존재 여부로 "N+1개 연속" 확인
```

```sql
-- 방법 1: 간격으로 연속 여부 직접 확인
SELECT 연도,
    연도 - LAG(연도) OVER (PARTITION BY 선수 ORDER BY 연도) AS 간격
FROM 기록;
-- 간격 = 1 → 날짜/연도 연속
-- 간격 = 4 → 올림픽처럼 4년 주기 연속 출전

-- 방법 2: 6행 앞 날짜 존재 여부 (7일치 완성 여부)
SELECT
    visited_on,
    LAG(visited_on, 6) OVER (ORDER BY visited_on) AS start_date
    -- NULL 이면 아직 6행 앞이 없음 → 7일치 미완성
    -- 값 있으면 6행 앞이 존재 → 7일치(현재 포함) 완성
```

```txt
LAG(컬럼, N) 에서 N 의 의미:
  N=1 (기본) → 바로 이전 1행
  N=6        → 6행 앞 (ROWS BETWEEN 6 PRECEDING 의 첫 행과 같은 지점)
  → "현재 행에서 6행 앞이 존재하면 = 7일치 데이터가 다 쌓였다" 는 뜻
```

## 문제 키워드 → 함수

|키워드|함수|
|---|---|
|이전 / 전날 / 전월|`LAG`|
|다음 / 익일|`LEAD`|
|연속 / 연속 등장|`LAG` 간격 비교 or `ROW_NUMBER` 패턴 (아래 섹션)|
|변화량 / 증감률|`LAG` + 뺄셈|

---

---

# ROWS BETWEEN — 프레임 범위 ⭐️

## 키워드 의미

```txt
PRECEDING   = 현재 행보다 앞 (위쪽)
FOLLOWING   = 현재 행보다 뒤 (아래쪽)
CURRENT ROW = 현재 행
UNBOUNDED   = 끝까지 (위 또는 아래 끝)
숫자        = 몇 행 앞/뒤
```

## 자주 쓰는 패턴 ⭐️

```sql
-- 패턴 1: 처음부터 현재까지 (누적합)
ROWS BETWEEN UNBOUNDED PRECEDING AND CURRENT ROW

-- 패턴 2: 현재부터 끝까지 (역방향 누적)
ROWS BETWEEN CURRENT ROW AND UNBOUNDED FOLLOWING

-- 패턴 3: 전체 파티션 (모든 행 같은 값)
ROWS BETWEEN UNBOUNDED PRECEDING AND UNBOUNDED FOLLOWING

-- 패턴 4: N일 이동합계 (N-1 PRECEDING)
ROWS BETWEEN 6 PRECEDING AND CURRENT ROW   -- 7일 (오늘 포함)
ROWS BETWEEN 2 PRECEDING AND CURRENT ROW   -- 3일

-- 패턴 5: 앞뒤 포함 (중심 이동평균)
ROWS BETWEEN 1 PRECEDING AND 1 FOLLOWING   -- 전일 + 오늘 + 다음날
```

```txt
그림으로 이해:
  데이터: [1, 2, 3, 4, 5]  현재 행 = 3

  UNBOUNDED PRECEDING ~ CURRENT ROW  → [1,2,3] ← 누적
  CURRENT ROW ~ UNBOUNDED FOLLOWING  → [3,4,5] ← 역방향
  UNBOUNDED PRECEDING ~ UNBOUNDED FOLLOWING → [1,2,3,4,5] ← 전체
  2 PRECEDING ~ CURRENT ROW          → [1,2,3] ← 이동 3일
  1 PRECEDING ~ 1 FOLLOWING          → [2,3,4] ← 중심
```

## 패턴별 용도

|패턴|용도|
|---|---|
|`UNBOUNDED PRECEDING ~ CURRENT ROW`|누적합 / 누적 카운트 ⭐️|
|`N PRECEDING ~ CURRENT ROW`|이동 N일 합계/평균 ⭐️|
|`UNBOUNDED PRECEDING ~ UNBOUNDED FOLLOWING`|LAST_VALUE 전체 파티션|
|`1 PRECEDING ~ 1 FOLLOWING`|중심 이동평균 (전후 포함)|

---

---

# 집계 윈도우 함수 — SUM / AVG OVER ⭐️

```txt
ORDER BY 없음 → 파티션 전체 합산 (모든 행에 같은 값)
ORDER BY 있음 → 현재 행까지 누적
```

```sql
SELECT 사원명, 급여,
    SUM(급여) OVER ()                       AS 전체합계,
    AVG(급여) OVER (PARTITION BY 부서)      AS 부서평균,
    SUM(급여) OVER (ORDER BY 입사일)        AS 누적합
FROM 사원;
```

## 이동 평균

```sql
-- 최근 3일 이동 평균 (오늘 포함)
AVG(매출) OVER (ORDER BY 날짜 ROWS BETWEEN 2 PRECEDING AND CURRENT ROW)

-- 공식: N = 원하는 개수 - 1
-- 3일 이동평균: ROWS BETWEEN 2 PRECEDING AND CURRENT ROW
-- 7일 이동평균: ROWS BETWEEN 6 PRECEDING AND CURRENT ROW
```

## 누적합 주의 — RANGE vs ROWS ⚠️

```sql
-- ❌ ORDER BY 만 쓰면 기본 프레임이 RANGE → 같은 값이 한꺼번에 더해져 점프 발생
SUM(급여) OVER (ORDER BY 급여)

-- ✅ ROWS 명시 → 정확히 1행씩 누적
SUM(급여) OVER (ORDER BY 급여 ROWS BETWEEN UNBOUNDED PRECEDING AND CURRENT ROW)
```

## 실전 — 누적합 + 조건 필터 ⭐️

```sql
-- 문제: 버스 무게 제한 1000kg 을 넘지 않는 마지막 탑승자 찾기
-- (Queue 테이블: person_name, weight, turn)

WITH total_weight AS (
    SELECT
        person_name,
        SUM(weight) OVER (ORDER BY turn) AS weight_sum
        --              ↑ turn 순서대로 누적 합산
    FROM Queue
)
SELECT person_name
FROM total_weight
WHERE weight_sum <= 1000       -- 제한 이하인 행만
ORDER BY weight_sum DESC       -- 가장 큰 누적합 = 마지막 탑승자
LIMIT 1;
```

```txt
이 패턴의 핵심:
  SUM() OVER (ORDER BY turn) → turn 순서대로 weight 를 1명씩 누적

  예시:
    turn=1  weight=100  weight_sum=100
    turn=2  weight=200  weight_sum=300
    turn=3  weight=300  weight_sum=600
    turn=4  weight=500  weight_sum=1100  ← 1000 초과

  WHERE weight_sum <= 1000 → turn=1,2,3 만 남음
  ORDER BY weight_sum DESC → turn=3 (600) 이 첫 번째 → 마지막 탑승자

CTE 와 함께 쓰는 이유:
  윈도우 함수 결과(weight_sum)를 WHERE 로 바로 필터 불가
  → CTE 로 먼저 계산 → 그 결과를 바깥 WHERE 로 필터
  → [[SQL_CTE]] 참고
```

---

---

# ROW_NUMBER 응용 패턴 ⭐️

## 그룹별 최신 행 1건 추출

```sql
-- 모든 DB: ROW_NUMBER
WITH ranked AS (
    SELECT *,
           ROW_NUMBER() OVER (PARTITION BY product_id ORDER BY change_date DESC) AS rn
    FROM Products
    WHERE change_date <= '2019-08-16'
)
SELECT * FROM ranked WHERE rn = 1;

-- PostgreSQL 전용: DISTINCT ON (더 간결)
SELECT DISTINCT ON (product_id)
       product_id, new_price
FROM Products
WHERE change_date <= '2019-08-16'
ORDER BY product_id, change_date DESC;
--       ↑ DISTINCT ON 컬럼이 ORDER BY 첫 번째여야 함
```

## Gaps and Islands — 연속 구간 찾기 ⭐️

```txt
연속된 값의 구간을 그룹으로 묶는 패턴
"값 - ROW_NUMBER()" 가 같은 행끼리 = 같은 연속 구간
```

```sql
-- 패턴 1: 값 자체가 1씩 증가 (연도, 날짜)
WITH numbered AS (
    SELECT year,
           year - ROW_NUMBER() OVER (ORDER BY year) AS grp
    FROM awards
)
SELECT MIN(year) AS start, MAX(year) AS end, COUNT(*) AS cnt
FROM numbered
GROUP BY grp;

-- 패턴 2: 같은 값이 연속 등장 (숫자/문자)
WITH row_id AS (
    SELECT num,
           ROW_NUMBER() OVER (ORDER BY id)
           - ROW_NUMBER() OVER (PARTITION BY num ORDER BY id) AS grp
    FROM Logs
)
SELECT num FROM row_id GROUP BY num, grp HAVING COUNT(*) >= 3;

-- 패턴 3: 조건 필터 후 연속 id 그룹
WITH filtered AS (
    SELECT id, visit_date, people,
           id - ROW_NUMBER() OVER (ORDER BY id) AS grp
    FROM Stadium
    WHERE people >= 100   -- 조건 먼저 필터링
),
counted AS (
    SELECT *, COUNT(*) OVER (PARTITION BY grp) AS cnt
    FROM filtered
)
SELECT id, visit_date, people
FROM counted WHERE cnt >= 3 ORDER BY visit_date;
```

```txt
패턴 선택:
  값이 1씩 증가 (연도/날짜) → 패턴 1: year - rn
  같은 값 반복 등장          → 패턴 2: 전체rn - 값별rn
  조건 필터 후 연속 id        → 패턴 3: id - rn + COUNT OVER
```

---

---

# WHERE vs HAVING 날짜 범위 ⭐️

```sql
-- "기간 내에만 존재하는 것" 검증
HAVING MIN(sale_date) >= '2019-01-01'
   AND MAX(sale_date) <= '2019-03-31'

-- WHERE 로 하면 안 되는 이유:
-- WHERE BETWEEN → 기간 내 판매가 "있기만" 하면 통과 (기간 밖 판매가 같이 있어도 통과)
-- HAVING MIN/MAX → 그 상품의 모든 판매가 기간 안에 있다는 걸 보장
```

---

---

# 실전 — 7일 이동합계 & 이동평균 패턴 ⭐️

## 문제 상황

```txt
식당 고객 방문 기록에서
최근 7일 동안의 총 결제 금액과 7일 평균 결제 금액 구하기

주의: 하루에 여러 고객이 방문할 수 있음
→ 날짜별 합계를 먼저 만든 후 이동합계 적용
```

## CTE + 윈도우 함수 조합 ⭐️

```sql
-- 1단계: 날짜별 합계 (CTE)
WITH cu_visit AS (
    SELECT
        visited_on,
        SUM(amount) AS amount       -- 같은 날짜 방문자 합산
    FROM Customer
    GROUP BY visited_on
),

-- 2단계: 7일 이동합계 + 시작일 확인
move_visit AS (
    SELECT
        visited_on,
        SUM(amount) OVER (
            ORDER BY visited_on
            ROWS BETWEEN 6 PRECEDING AND CURRENT ROW   -- 이전 6일 + 오늘 = 7일
        ) AS amount,
        LAG(visited_on, 6) OVER (ORDER BY visited_on) AS start_date
        --  ↑ 6일 전 날짜 (7일치가 존재하는지 확인용)
    FROM cu_visit
)

-- 3단계: 7일치 데이터가 완성된 행만 필터
SELECT
    visited_on,
    amount,
    ROUND(amount / 7.0, 2) AS average_amount
    --                ↑ 7.0 으로 소수 계산 안정적으로
FROM move_visit
WHERE start_date IS NOT NULL;
--    ↑ LAG = NULL 이면 아직 7일치 미만 → 제외
```

## 핵심 포인트 ⭐️

```txt
1. 날짜별 GROUP BY 먼저:
   하루에 여러 방문자 → SUM(amount) GROUP BY visited_on → 날짜 하나에 값 하나

2. ROWS BETWEEN 6 PRECEDING AND CURRENT ROW:
   현재 행 + 이전 6행 = 총 7행

3. LAG(visited_on, 6) 로 시작일 검증:
   7일치가 쌓이기 전(처음 6행)은 LAG = NULL → WHERE 로 제외

4. amount / 7.0:
   7(정수)로 나누면 정수 나눗셈 → 소수점 버림 / 7.0(실수)이면 소수점 유지

요약: CTE 1(날짜별 합계) → CTE 2(이동합계 + LAG 시작일 체크) → 최종(완성된 행만 + 평균)
```

---

---

# 초보자 실수 모음

|실수|해결|
|---|---|
|순위 함수에 `ORDER BY` 없이 `OVER()`|`OVER (ORDER BY 컬럼)` 필수|
|윈도우 함수를 `WHERE` 절에 직접 사용|CTE 감싸고 바깥에서 필터링|
|집계는 필요한데 GROUP BY 하면 다른 컬럼이 사라짐|`PARTITION BY` 로 원본 행 유지하며 집계 (맨 위 employee 예제 참고)|
|`LAST_VALUE()` 가 자기 자신만 출력|`ROWS BETWEEN UNBOUNDED PRECEDING AND UNBOUNDED FOLLOWING`|
|누적합이 점프함|`ROWS BETWEEN UNBOUNDED PRECEDING AND CURRENT ROW` 명시 (RANGE 기본값 주의)|
|`COUNT` 로 연속성 판단|`LAG` + 간격 비교 또는 `ROW_NUMBER` 패턴|
|`WHERE` 날짜 범위 → 일부 상품 사라짐|`HAVING MIN(날짜) >= 시작 AND MAX(날짜) <= 종료`|

---

---

# 한눈에

```txt
가장 먼저 떠올려야 할 질문:
  "집계값이 필요한데, 원본 행의 다른 컬럼도 그대로 봐야 하나?"
  → Yes → PARTITION BY (윈도우 함수)
  → No (그냥 그룹별 요약만 필요)  → GROUP BY

함수 선택:
  순위 / Top-N              → ROW_NUMBER / RANK / DENSE_RANK
  이전·다음 행 / 연속성 판단   → LAG / LEAD
  누적합 / 이동평균           → SUM·AVG OVER + ROWS BETWEEN
  그룹별 개수를 원본 행에 유지  → COUNT(컬럼) OVER (PARTITION BY ...)

공통 주의사항:
  윈도우 함수는 WHERE 에 직접 못 씀 → CTE 로 감싸고 바깥에서 필터
  ORDER BY 없는 누적합은 RANGE 기본값 때문에 점프할 수 있음 → ROWS 명시
  COUNT 로는 "연속" 판단 불가 → LAG 간격 비교 또는 ROW_NUMBER 패턴
```