

```txt
데이터를 다루는 언어
PostgreSQL 실무 + MySQL 실무
```

---

---

## Level 0. 개념 잡기

```txt
SQL 이 뭔지, DB 가 뭔지
```

| 노트              | 핵심 개념                                                                            |
| --------------- | -------------------------------------------------------------------------------- |
| [[SQL_Concept]] | DB 란 / RDBMS / SQL 종류(DQL·DML·DDL) / 테이블·행·컬럼 / PK·FK·인덱스 / 기본 용어                |
| [[SQL_Setup]]   | PostgreSQL 설치(Docker·직접) / psql / DataGrip ⭐️ / Node.js(pg) / NestJS(TypeORM) 연결 |

---

---

## Level 1. 기본 조회

```txt
데이터를 꺼내는 법
```

| 노트                       | 핵심 개념                                                                                             |
| ------------------------ | ------------------------------------------------------------------------------------------------- |
| [[SQL_SELECT]] ⭐         | SELECT / FROM / WHERE / ORDER BY / LIMIT / DISTINCT                                               |
| [[SQL_WHERE]] ⭐          | AND·OR 우선순위 / BETWEEN 끝값포함 / IN·NOT IN NULL함정 / LIKE·ESCAPE / IS NULL                             |
| [[SQL_Functions_Basic]]  | UPPER / LOWER / LEFT / SUBSTRING / CONCAT vs \|\| ⭐️ / COALESCE / NULLIF / MySQL vs PostgreSQL 차이 |
| [[SQL_Date_Functions]] ⭐ | EXTRACT / DATE_TRUNC / INTERVAL / TO_CHAR / 날짜 연산                                                 |


---

## Level 2. 집계 & 그룹

```txt
데이터를 요약하는 법
  윈도우 함수 → 행별로 계산 / 전체 데이터 유지
  GROUP BY + 조건부 집계 → 그룹별 단일 결과
```

| 노트                  | 핵심 개념                                                                                                   |
| ------------------- | ------------------------------------------------------------------------------------------------------- |
| [[SQL_Aggregate]] ⭐ | COUNT / SUM / AVG / MAX / MIN / NULL 투명인간 / NULL 전파 / 중첩 집계 불가 / 조건부 집계 / 비율 계산/STRING_AGG·GROUP_CONCAT |
| [[SQL_GROUP_BY]] ⭐  | GROUP BY / HAVING / GROUP BY 기준=PK(id) ⭐️ / GROUP BY vs Window 함수️                                      |
| [[SQL_Subquery]] ⭐  | 서브쿼리 / 인라인 뷰 / 스칼라 서브쿼리 / EXISTS                                                                        |

---
---

## Level 3. JOIN

```txt
여러 테이블 연결하는 법
```

| 노트                    | 핵심 개념                                                                                                     |
| --------------------- | --------------------------------------------------------------------------------------------------------- |
| [[SQL_JOIN]] ⭐        | INNER / LEFT / RIGHT / FULL / ON vs WHERE ⭐️ / ON+AND 조건 / CROSS JOIN / COALESCE / 가중평균 패턴 / SUM vs COUNT |
| [[SQL_JOIN_Advanced]] | JOIN vs UNION / UNION vs UNION ALL / 괄호+ORDER BY+LIMIT / 1등 뽑기 / 동점 처리 / OR JOIN → UNION ALL              |


---

---

## Level 4. 데이터 조작

```txt
데이터를 넣고 바꾸고 지우는 법
```

| 노트                 | 핵심 개념                                                                                 |
| ------------------ | ------------------------------------------------------------------------------------- |
| [[SQL_DDL]]        | CREATE TABLE / ALTER TABLE / DROP / TRUNCATE / RESTART IDENTITY CASCADE / PG vs MySQL |
| [[SQL_DML]] ⭐      | INSERT·UPDATE·DELETE 문법 / RETURNING ⭐️ / UPSERT·ON CONFLICT                           |
| [[SQL_Data_Types]] | VARCHAR / INT / BOOLEAN / DATE / TIMESTAMP / JSON                                     |

---

---

## Level 5. 고급 분석

```txt
실무 분석 쿼리
  윈도우 함수 → 행별로 계산 / 전체 데이터 유지
```

| 노트                         | 핵심 개념                                                                 |
| -------------------------- | --------------------------------------------------------------------- |
| [[SQL_Window_Functions]] ⭐ | ROW_NUMBER / RANK / LAG / LEAD / SUM OVER / ROWS BETWEEN / 7일 이동합계 패턴 |
| [[SQL_CTE]] ⭐              | WITH / 재귀 CTE / 복잡한 쿼리 단계별 분리                                         |
| [[SQL_CASE_WHEN]] ⭐        | IF-ELSE / 조건부 집계 / 부호 전환 패턴(Buy→-price) ⭐️ / 피벗 / 비율 계산               |


---

---

## Level 6. PostgreSQL 실무

```txt
PostgreSQL 전용 기능
```

| 노트              | 핵심 개념                                                                  |
| --------------- | ---------------------------------------------------------------------- |
| [[PG_Specific]] | DISTINCT ON / ::캐스팅 / ILIKE / 정규식(~) / 튜플 비교 ⭐️ / "큰따옴표" 식별자/(^\| ) 패턴 |
| [[PG_Index]]    | 인덱스 / EXPLAIN / 쿼리 최적화                                                 |
| [[PG_JSON]]     | JSON / JSONB / ->> / @> 연산자                                            |


---

---

## Level 6. MySQL 실무

```txt
MySQL 전용 기능
```

| 노트                 | 핵심 개념                                                              |
| ------------------ | ------------------------------------------------------------------ |
| [[MySQL_Specific]] | REGEXP_LIKE·플래그 ⭐️ / IFNULL / DATE_FORMAT / GROUP_CONCAT/(^\| ) 패턴 |
| [[MySQL_Engine]]   | InnoDB / 트랜잭션 / AUTO_INCREMENT                                     |

---

---

## 실전 패턴

| 노트                             | 설명                                                             |
| ------------------------------ | -------------------------------------------------------------- |
| [[SQL_Pattern_전체목록_LEFT_JOIN]] | 빈 카테고리도 0으로 출력 / UNION ALL (간단) vs CTE+LEFT JOIN / COALESCE ⭐️ |
| [[SQL_Pattern_최신행]]            | 그룹별 최신 행 1건 (DISTINCT ON / ROW_NUMBER)                         |
| [[SQL_Pattern_비율계산]]           | 전체 대비 비율 / WHERE 금지 / CASE WHEN                                |
| [[SQL_Pattern_날짜범위]]           | = AND < 월별 패턴 / INTERVAL / ::DATE 캐스팅 / ON 에 날짜 조건             |
