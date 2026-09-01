---
aliases:
  - Database
  - PostgreSQL
  - MySQL
  - Redis
  - SQL
tags:
  - HomePage
related:
  - "[[00_NestJS_Ecosystem_HomePage]]"
  - "[[NestJS_PostgreSQL]]"
  - "[[00_DevOps_Ecosystem_HomePage]]"
cssclasses:
  - max
  - table-max
  - table-wrap
---
# 00_DB_HomePage — 데이터베이스

>[!info]
>주 DB: PostgreSQL. DDL(테이블 정의)·DML(데이터 조작)·Aggregate(집계)는 SQL의 핵심.
> 트랜잭션·격리·패턴은 PG 전용 노트에. 
> Prisma ORM·마이그레이션은 NestJS 폴더 → [[00_NestJS_Ecosystem_HomePage]] 🗄️ 섹션.

---

## 빠른 찾기

| 찾는 것 | 노트 |
|---|---|
| CREATE · ALTER · DROP · 제약조건 | [[PG_DDL]] |
| SELECT · INSERT · UPDATE · DELETE · JOIN · 서브쿼리 | [[PG_DML]] |
| btrim · lower · ILIKE · concat_ws · regexp_replace | [[PG_StringFunctions]] |
| COUNT · SUM · GROUP BY · 윈도우 함수 | [[PG_Aggregate]] |
| NULL UNIQUE 함정 · ON CONFLICT · 인덱스 | [[PG_Patterns]] |
| 테이블·오브젝트 존재 여부 확인 (`to_regclass`) | [[PG_Patterns]] |
| Expand-Contract · Backfill · CONCURRENTLY | [[DB_MigrationPattern]] |
| 트랜잭션 · 격리 수준 · DEADLOCK | [[PG_Transaction]] |
| ACID · BASE · CAP 이론 | [[DB_Transaction]] |
| 캐싱 · TTL · Pub/Sub | [[Redis_Patterns]] |
| Prisma 쿼리 · migrate | → [[NestJS_Prisma]] |

---

## 🐘 PostgreSQL — SQL

| 노트               | 내용                                                                  |
| ---------------- | ------------------------------------------------------------------- |
| [[PG_DDL]]       | CREATE TABLE · ALTER · DROP · 제약조건 · 인덱스 DDL                        |
| [[PG_DML]]       | SELECT · INSERT · UPDATE · DELETE · JOIN · 서브쿼리 · ORDER BY RANDOM() |
| [[PG_StringFunctions]] | btrim · ltrim · rtrim · lower · upper · ILIKE · concat_ws · regexp_replace |
| [[PG_Aggregate]] | COUNT · SUM · AVG · GROUP BY · HAVING · 윈도우 함수                      |

```txt
PG_DDL (Data Definition Language) — 구조 정의:
  CREATE TABLE — 컬럼 타입 · NOT NULL · DEFAULT · PRIMARY KEY
  ALTER TABLE  — 컬럼 추가/삭제/타입 변경
  DROP TABLE   — 테이블 삭제
  제약조건      — UNIQUE · CHECK · FOREIGN KEY · ON DELETE CASCADE
  인덱스 DDL   — CREATE INDEX · UNIQUE INDEX · Partial Index

PG_DML (Data Manipulation Language) — 데이터 조작:
  SELECT · WHERE · ORDER BY(RANDOM 포함) · LIMIT · OFFSET
  INSERT · UPDATE · DELETE · JOIN · 서브쿼리
  SELECT      — WHERE · ORDER BY · LIMIT · OFFSET
  JOIN        — INNER · LEFT · RIGHT · FULL OUTER
  INSERT      — 단건 · 다건 · ON CONFLICT
  UPDATE      — SET · WHERE · FROM
  DELETE      — WHERE · RETURNING
  서브쿼리     — IN · EXISTS · 스칼라 서브쿼리

PG_Aggregate — 집계:
  COUNT · SUM · AVG · MIN · MAX
  GROUP BY · HAVING
  윈도우 함수 — ROW_NUMBER · RANK · LAG · LEAD · OVER(PARTITION BY)
  DISTINCT · FILTER

PG_StringFunctions — 문자열 함수:
  trim 계열  — btrim(양쪽) · ltrim(왼쪽) · rtrim(오른쪽)
  변환       — lower · upper · initcap
  길이       — length · octet_length
  이어붙이기  — concat · concat_ws(구분자 포함) · || 연산자
  부분 추출  — substring · left · right
  교체       — replace · regexp_replace
  검색       — position · strpos
  패턴 검색  — LIKE(대소문자 구분) · ILIKE(무시, PostgreSQL 전용)
```

---

## 🐘 PostgreSQL — 고급

| 노트 | 내용 |
|---|---|
| [[PG_Patterns]] | NULL UNIQUE · ON CONFLICT · RETURNING · 인덱스 전략 · EXPLAIN |
| [[PG_Transaction]] | BEGIN/COMMIT · 격리 수준 · DEADLOCK · SAVEPOINT · FOR UPDATE |

```txt
PG_Patterns:
  UNIQUE + NULL 함정 — NULL끼리 중복 허용되는 이유 · 센티넬 값(-1)으로 해결
  ON CONFLICT — upsert (없으면 INSERT, 있으면 UPDATE · DO NOTHING)
  RETURNING   — INSERT/UPDATE/DELETE 결과 즉시 반환 (추가 SELECT 불필요)
  인덱스 전략  — 복합 인덱스 순서 · Partial Index · 언제 만드는가
  EXPLAIN ANALYZE — Seq Scan vs Index Scan · 실행 계획 읽는 법
  to_regclass — 테이블·오브젝트 존재 여부 확인 (존재하면 이름 반환 · 없으면 NULL)

PG_Transaction:
  격리 수준   — READ COMMITTED(기본) · REPEATABLE READ · SERIALIZABLE
  DEADLOCK   — 자동 감지 · 예방 원칙 (항상 같은 순서로 잠금)
  MVCC       — 읽기-쓰기 비충돌 동작 원리
  SAVEPOINT  — 부분 롤백 패턴
  FOR UPDATE — 배타적 잠금 · SKIP LOCKED (작업 큐 패턴)
```

---

## 📚 DB 이론

| 노트 | 내용 |
|---|---|
| [[DB_Transaction]] | ACID · BASE · CAP 이론 · 트랜잭션 개념 · 격리 수준 이론 |
| [[DB_MigrationPattern]] | Expand-Contract · Backfill UPDATE · CONCURRENTLY · 무중단 스키마 변경 |

```txt
DB_Transaction:
  ACID — 원자성(Atomicity) · 일관성(Consistency) · 격리성(Isolation) · 영속성(Durability)
  BASE — 분산 DB에서 완화된 일관성 (Basically Available · Soft state · Eventually consistent)
  CAP  — Consistency · Availability · Partition tolerance 중 2가지만 보장
  격리 수준 이론 — Dirty Read · Non-Repeatable Read · Phantom Read

→ PostgreSQL에서의 실제 적용 → [[PG_Transaction]]

DB_MigrationPattern — 무중단 마이그레이션:
  Expand-Contract — ① 새 구조 추가(nullable) ② Backfill 데이터 이전 ③ 기존 구조 제거
  Backfill UPDATE — UPDATE ... FROM · btrim 공백 필터 · 배치 처리(LIMIT + sleep)
  CONCURRENTLY    — CREATE INDEX CONCURRENTLY · 락 없이 인덱스 생성 (트랜잭션 밖에서)
  NOT NULL 추가   — nullable 추가 → Backfill → NOT NULL 설정 순서
```

---

## 🔴 Redis

| 노트 | 내용 |
|---|---|
| [[Redis_Patterns]] | String · Hash · List · Set · Sorted Set · TTL · 캐싱 · Pub/Sub |

---

## 🔗 Prisma ORM · 마이그레이션

```txt
Prisma는 NestJS 폴더에서 관리 (NestJS와 함께 쓰는 ORM이므로)

Prisma 쿼리 레퍼런스     → [[NestJS_Prisma]]
where 조립 · 토글 패턴   → [[NestJS_Prisma_Patterns]]
migrate 명령어           → [[NestJS_Migration]]
$transaction            → [[NestJS_Transaction]]
```