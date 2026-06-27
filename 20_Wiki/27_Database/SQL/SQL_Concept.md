---
aliases:
  - SQL 개념
  - DB란
  - RDBMS
  - 데이터베이스 기초
tags:
  - SQL
related:
  - "[[00_SQL_HomePage]]"
  - "[[SQL_SELECT]]"
  - "[[SQL_DDL]]"
  - "[[SQL_Setup]]"
---

# SQL_Concept — DB & SQL 기초 개념

---

---

# ① DB (Database) 란 ⭐️

```txt
Database = 데이터를 체계적으로 저장하고 관리하는 시스템

DB 가 없으면:
  파일(.txt / .csv) 에 저장
  → 여러 사람이 동시에 접근 불가
  → 검색 느림 / 데이터 중복 / 일관성 없음

DB 가 있으면:
  동시 접근 허용 (트랜잭션)
  빠른 검색 (인덱스)
  데이터 무결성 보장
  백업 / 복구 지원
```

---

---

# ② RDBMS ⭐️

```txt
RDBMS = Relational DataBase Management System
관계형 데이터베이스 관리 시스템

관계형 = 데이터를 테이블(행 + 열) 형태로 저장
테이블끼리 관계(JOIN)로 연결

대표적인 RDBMS:
  PostgreSQL  오픈소스 / 기능 강력 / 데이터 엔지니어 많이 씀
  MySQL       오픈소스 / 웹 서비스에 많이 씀
  SQLite      파일 기반 / 가벼움 / 모바일/임베디드
  Oracle      기업용 / 유료
  MSSQL       Microsoft / 윈도우 환경

RDBMS vs NoSQL:
  RDBMS   테이블 구조 / 스키마 고정 / SQL 사용
  NoSQL   문서/키-값/그래프 / 스키마 유연 / 대용량 수평 확장
  MongoDB / Redis / Cassandra 등
```

---

---

# ③ SQL 이란 ⭐️

```txt
SQL = Structured Query Language
구조화된 질의 언어

DB 에게 "이런 데이터 줘" / "이렇게 바꿔" / "이거 지워" 명령하는 언어

종류:
  DQL (Data Query Language)
    SELECT  데이터 조회

  DML (Data Manipulation Language)
    INSERT  데이터 삽입
    UPDATE  데이터 수정
    DELETE  데이터 삭제

  DDL (Data Definition Language)
    CREATE  테이블 생성
    ALTER   테이블 수정
    DROP    테이블 삭제

  DCL (Data Control Language)
    GRANT   권한 부여
    REVOKE  권한 회수
```

---

---

# ④ 테이블 · 행 · 컬럼 ⭐️

```txt
테이블 (Table):
  데이터를 저장하는 단위
  스프레드시트의 시트 하나와 유사

행 (Row / Record / Tuple):
  테이블의 가로 한 줄
  실제 데이터 하나
  예: 홍길동 / 30세 / 서울 → 한 명의 정보

컬럼 (Column / Field / Attribute):
  테이블의 세로 한 줄
  데이터의 속성/항목
  예: name / age / city

예시:
  users 테이블

  id │ name   │ age │ city
  ───┼────────┼─────┼──────
   1 │ 홍길동 │  30 │ 서울    ← 행(Row)
   2 │ 김영희 │  25 │ 부산
   3 │ 이철수 │  35 │ 인천
   ↑
  컬럼(Column)
```

---

---

# ⑤ 기본 용어 정리

```txt
스키마 (Schema):
  DB 의 구조 설계도
  테이블 목록 / 컬럼 이름과 타입 / 제약조건 정의

기본키 (Primary Key / PK):
  각 행을 고유하게 식별하는 컬럼
  중복 불가 / NULL 불가
  보통 id 컬럼

외래키 (Foreign Key / FK):
  다른 테이블의 PK 를 참조하는 컬럼
  테이블 간 관계 표현

인덱스 (Index):
  검색 속도를 높이기 위한 자료구조
  책의 색인과 유사
  PK 는 자동으로 인덱스 생성

쿼리 (Query):
  DB 에 보내는 SQL 명령문

트랜잭션 (Transaction):
  여러 SQL 을 하나의 작업 단위로 묶은 것
  전부 성공 또는 전부 실패 (원자성)
```

---

---

# ⑥ SQL 이 중요한 이유

```txt
백엔드 개발:
  NestJS + TypeORM → 결국 SQL 로 변환됨
  복잡한 쿼리 / 성능 최적화 → SQL 직접 작성 필요
  DB 설계 → DDL 로 테이블 구조 정의

데이터 처리:
  원천 DB → SQL 변환 → 데이터 웨어하우스
  ETL (Extract Transform Load)
  Airflow / Spark 에서 SQL 쿼리 실행
  PostgreSQL / BigQuery / Redshift 모두 SQL

공통:
  어느 방향이든 DB 를 다루는 사람이라면 SQL 은 필수
  ORM 써도 내부는 SQL → SQL 알아야 디버깅 가능
  데이터 검증 / 집계 / 분석 → SQL 이 가장 직접적
```
