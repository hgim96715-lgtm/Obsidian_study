---
aliases:
  - Transaction
  - PG
  - NestJS
  - BEGIN
  - COMMIT
  - ROLLBACK
tags:
  - PostgreSQL
related:
  - "[[00_DB_HomePage]]"
  - "[[DB_Transaction]]"
  - "[[NestJS_Transaction]]"
---
# PG_Transaction — PostgreSQL 트랜잭션

> [!info] 
> PostgreSQL에서 트랜잭션은 `BEGIN`으로 시작해 `COMMIT` 또는 `ROLLBACK`으로 끝난다.
>  격리 수준이 동시 요청 간 간섭을 얼마나 허용할지를 결정하고, DEADLOCK은 PostgreSQL이 자동으로 감지해 한쪽을 강제 롤백한다. 
>  Prisma에서의 사용법 → [[NestJS_Transaction]], ACID 개념 → [[DB_Transaction]]

---

# 트랜잭션 기본 — BEGIN / COMMIT / ROLLBACK ⭐️⭐️⭐️

```sql
BEGIN;

UPDATE accounts SET balance = balance - 1000 WHERE id = 1;
UPDATE accounts SET balance = balance + 1000 WHERE id = 2;

COMMIT;   -- 성공 시 확정
-- 또는
ROLLBACK; -- 실패 시 전부 되돌림
```

```txt
BEGIN 이전: 각 쿼리가 자동으로 COMMIT되는 오토커밋 모드
BEGIN 이후: 명시적으로 COMMIT 또는 ROLLBACK 할 때까지 변경이 확정되지 않음

트랜잭션 블록 안에서 에러가 발생하면:
  이후 쿼리를 실행해도 "ERROR: current transaction is aborted" 메시지와 함께 전부 거부됨
  → 반드시 ROLLBACK 후 다시 시작해야 함
```

## 에러 후 상태 처리

```sql
BEGIN;
UPDATE accounts SET balance = balance - 1000 WHERE id = 1;

-- 여기서 에러 발생 (예: 잔액 부족 CHECK 제약 위반)

-- 이 상태에서 다른 쿼리를 실행하면 전부 오류
SELECT * FROM accounts;  -- ERROR: current transaction is aborted

-- ROLLBACK 후 다시 시작해야 함
ROLLBACK;
BEGIN;
-- ...
```

```txt
DataGrip 등에서 트랜잭션이 꼬인 것 같을 때:
  ROLLBACK; 한 번 실행하면 깨끗하게 초기화됨
```

---

# SAVEPOINT — 부분 롤백 ⭐️⭐️⭐️

```sql
BEGIN;

INSERT INTO orders (user_id, product_id) VALUES (1, 10);

SAVEPOINT after_order;  -- 이 시점을 저장

INSERT INTO inventory_log (product_id, change) VALUES (10, -1);
-- 여기서 에러 발생

ROLLBACK TO SAVEPOINT after_order;  -- after_order 이후만 되돌림
                                     -- orders INSERT는 살아있음

-- 다른 방식으로 재시도
INSERT INTO inventory_log (product_id, change, note) VALUES (10, -1, 'manual');

RELEASE SAVEPOINT after_order;  -- SAVEPOINT 해제 (COMMIT은 아님)

COMMIT;  -- 최종 확정
```

```txt
SAVEPOINT 명령 3가지:
  SAVEPOINT 이름              — 현재 시점 저장
  ROLLBACK TO SAVEPOINT 이름  — 그 시점으로 되돌림 (SAVEPOINT는 유지됨)
  RELEASE SAVEPOINT 이름      — SAVEPOINT 해제 (변경은 유지, COMMIT은 아님)

ROLLBACK TO SAVEPOINT vs ROLLBACK:
  ROLLBACK TO SAVEPOINT  → 일부만 되돌림, 트랜잭션은 유지됨
  ROLLBACK               → 트랜잭션 전체를 처음으로 되돌림

언제 쓰는가:
  트랜잭션 안에서 "여기까지는 확실히 쓰고, 이 부분은 실패해도 괜찮아"라는 분기가 있을 때
  예: 주문 생성은 확정, 쿠폰 적용은 실패해도 주문은 유지하고 싶을 때
```

---

# Isolation Level — 격리 수준 ⭐️⭐️⭐️⭐️

```txt
동시에 여러 트랜잭션이 실행될 때 서로 얼마나 간섭하는가를 결정하는 수준
격리 수준이 낮을수록 성능은 좋고, 높을수록 데이터 일관성이 보장됨
```

## 이상 현상(Anomaly) — 격리 수준이 낮으면 발생할 수 있는 문제

|이상 현상|설명|예시|
|---|---|---|
|**Dirty Read**|아직 COMMIT 안 된 다른 트랜잭션의 데이터를 읽음|A가 balance를 수정 중인데 B가 수정된 값을 미리 읽음. A가 ROLLBACK하면 B는 없던 값을 읽은 셈|
|**Non-Repeatable Read**|같은 행을 트랜잭션 안에서 두 번 읽었을 때 값이 다름|B가 읽는 사이에 A가 COMMIT해서 값이 바뀜|
|**Phantom Read**|같은 조건의 범위 쿼리를 두 번 실행했을 때 행 개수가 다름|B가 COUNT하는 사이에 A가 INSERT + COMMIT해서 행이 추가됨|

## 격리 수준 비교

|격리 수준|Dirty Read|Non-Repeatable Read|Phantom Read|PostgreSQL 기본|
|---|---|---|---|---|
|`READ UNCOMMITTED`|발생 가능|발생 가능|발생 가능|❌ (PostgreSQL은 READ COMMITTED처럼 동작)|
|`READ COMMITTED`|방지|발생 가능|발생 가능|✅ **기본값**|
|`REPEATABLE READ`|방지|방지|발생 가능|❌|
|`SERIALIZABLE`|방지|방지|방지|❌|

```txt
PostgreSQL의 READ UNCOMMITTED:
  SQL 표준에는 있지만 PostgreSQL은 이를 지원하지 않음
  READ UNCOMMITTED를 설정해도 READ COMMITTED처럼 동작함
  → Dirty Read는 어떤 격리 수준에서도 PostgreSQL에서는 발생하지 않음

PostgreSQL REPEATABLE READ:
  SQL 표준에서는 Phantom Read가 가능하다고 명시하지만
  PostgreSQL의 MVCC 구현 덕분에 실제로는 Phantom Read도 방지됨
```

## 격리 수준 설정

```sql
-- 트랜잭션 단위로 설정 (BEGIN 직후)
BEGIN;
SET TRANSACTION ISOLATION LEVEL REPEATABLE READ;
-- 이 트랜잭션 안에서만 적용

-- 세션 단위로 설정 (이후 모든 트랜잭션에 적용)
SET SESSION CHARACTERISTICS AS TRANSACTION ISOLATION LEVEL SERIALIZABLE;
```

```sql
-- 현재 격리 수준 확인
SHOW transaction_isolation;
```

## 언제 격리 수준을 올리는가

```txt
READ COMMITTED (기본):
  대부분의 CRUD API에서 충분
  쿼리가 실행되는 순간의 최신 COMMIT된 데이터를 읽음

REPEATABLE READ:
  트랜잭션 안에서 같은 행을 여러 번 읽는데 일관된 값이 필요할 때
  예: 리포트 생성 트랜잭션 — 계산 중에 다른 트랜잭션이 값을 바꿔도 처음 읽은 값으로 일관되게 처리

SERIALIZABLE:
  "이 트랜잭션들을 하나씩 순서대로 실행한 것과 결과가 동일해야 한다"는 최강 보장
  예: 금융 이체, 재고 정합성이 완벽히 필요한 경우
  성능 트레이드오프가 크고, 충돌 시 P2034 에러 → 재시도 로직 필요
```

---

# MVCC — 격리 수준의 동작 원리 ⭐️⭐️⭐️

```txt
PostgreSQL은 MVCC(Multi-Version Concurrency Control)로 격리를 구현한다

핵심 아이디어:
  데이터를 수정하면 기존 행을 지우지 않고 새 버전을 추가
  각 트랜잭션은 자신이 시작된 시점의 "스냅샷"을 가짐
  → 읽기 트랜잭션이 쓰기 트랜잭션을 블로킹하지 않음 (읽기-쓰기 비충돌)

READ COMMITTED 동작:
  각 쿼리 실행 시점의 최신 커밋된 스냅샷을 봄
  쿼리가 실행될 때마다 스냅샷 갱신 → Non-Repeatable Read 가능

REPEATABLE READ 동작:
  트랜잭션 시작 시점의 스냅샷을 고정
  트랜잭션이 끝날 때까지 같은 스냅샷으로 읽음 → Non-Repeatable Read 방지

불필요해진 오래된 버전의 행은 VACUUM이 정리함
```

---

# DEADLOCK — 교착 상태 ⭐️⭐️⭐️⭐️

```txt
DEADLOCK: 두 트랜잭션이 서로가 잠근 행을 기다리는 상태

발생 예시:
  트랜잭션 A: accounts(id=1) 잠금 → accounts(id=2) 기다림
  트랜잭션 B: accounts(id=2) 잠금 → accounts(id=1) 기다림
  → 서로가 서로를 기다려서 영원히 진행 불가
```

```sql
-- DEADLOCK 발생 시나리오
-- 트랜잭션 A
BEGIN;
UPDATE accounts SET balance = balance - 100 WHERE id = 1;  -- id=1 잠금
-- (여기서 일시 정지)

-- 트랜잭션 B
BEGIN;
UPDATE accounts SET balance = balance - 100 WHERE id = 2;  -- id=2 잠금
UPDATE accounts SET balance = balance + 100 WHERE id = 1;  -- id=1 기다림... (A가 점유 중)

-- 트랜잭션 A 재개
UPDATE accounts SET balance = balance + 100 WHERE id = 2;  -- id=2 기다림... (B가 점유 중)
-- → DEADLOCK!
```

## PostgreSQL의 자동 감지

```txt
PostgreSQL은 DEADLOCK을 자동으로 감지한다:
  잠금 대기 그래프를 주기적으로 검사 (기본 1초)
  사이클이 감지되면 그 중 한 트랜잭션을 "희생자(victim)"로 선택
  희생자 트랜잭션에 에러를 던지고 롤백
  나머지 트랜잭션은 정상 진행

에러 메시지:
  ERROR: deadlock detected
  DETAIL: Process N waits for ShareLock on transaction M; blocked by process K.
```

## DEADLOCK 예방 — 항상 같은 순서로 잠금 ⭐️⭐️⭐️

```sql
-- ❌ 트랜잭션마다 다른 순서로 잠금 → DEADLOCK 위험
-- A: id=1 먼저 → id=2
-- B: id=2 먼저 → id=1

-- ✅ 항상 id 오름차순으로 처리 → DEADLOCK 방지
BEGIN;
-- 항상 id가 작은 것부터 잠금
UPDATE accounts SET balance = balance - 100 WHERE id = 1;  -- 작은 id 먼저
UPDATE accounts SET balance = balance + 100 WHERE id = 2;  -- 큰 id 나중에
COMMIT;
```

```txt
DEADLOCK 예방 원칙:
  ① 항상 같은 순서로 잠금 (가장 효과적)
     예: 여러 행을 수정할 때 PK 오름차순으로 정렬해서 처리
  ② 트랜잭션을 짧게 유지
     잠금을 점유하는 시간이 짧을수록 다른 트랜잭션과 겹칠 확률이 낮아짐
  ③ 필요한 행만 잠금
     과도한 범위 잠금은 DEADLOCK 확률을 높임
```

## DEADLOCK vs 일반 Lock Wait

```txt
Lock Wait (DEADLOCK 아님):
  트랜잭션 A가 행을 잠근 상태
  트랜잭션 B가 같은 행에 접근 → A가 끝날 때까지 기다림
  A가 COMMIT/ROLLBACK → B가 잠금 획득하고 진행
  → 정상적인 대기, 언젠가는 해소됨

DEADLOCK:
  서로가 서로를 기다려서 절대로 해소되지 않는 상태
  → PostgreSQL이 강제로 한쪽을 롤백해야 해소됨

Lock Wait Timeout 설정:
  lock_timeout = '5s'  → 5초 이상 기다리면 에러 (DEADLOCK이 아닌 일반 대기에도 적용)
  deadlock_timeout = '1s'  → DEADLOCK 감지 주기 (기본값)
```

---

# 잠금 모드 — FOR UPDATE / FOR SHARE ⭐️⭐️⭐️

```sql
-- FOR UPDATE — 배타적 잠금 (읽고 수정할 예정)
SELECT * FROM products WHERE id = 1 FOR UPDATE;
-- 다른 트랜잭션의 UPDATE, DELETE, FOR UPDATE 전부 블로킹

-- FOR SHARE — 공유 잠금 (읽기만, 다른 읽기는 허용)
SELECT * FROM products WHERE id = 1 FOR SHARE;
-- 다른 트랜잭션의 FOR SHARE는 허용, UPDATE/DELETE는 블로킹
```

|모드|다른 트랜잭션의 FOR SHARE|다른 트랜잭션의 FOR UPDATE|다른 트랜잭션의 UPDATE/DELETE|
|---|---|---|---|
|`FOR SHARE`|✅ 허용|❌ 블로킹|❌ 블로킹|
|`FOR UPDATE`|❌ 블로킹|❌ 블로킹|❌ 블로킹|

```sql
-- NOWAIT — 기다리지 않고 즉시 에러
SELECT * FROM products WHERE id = 1 FOR UPDATE NOWAIT;
-- 이미 잠겨있으면 대기 없이 ERROR: could not obtain lock on row

-- SKIP LOCKED — 잠긴 행은 건너뜀
SELECT * FROM tasks WHERE status = 'pending' FOR UPDATE SKIP LOCKED LIMIT 1;
-- 잠긴 행을 건너뛰고 잠글 수 있는 첫 번째 행을 가져옴
-- → 작업 큐 구현에 자주 쓰임 (여러 워커가 같은 큐를 처리할 때)
```

```txt
SKIP LOCKED 활용 — 작업 큐 패턴:
  여러 서버 인스턴스가 같은 DB의 tasks 테이블에서 작업을 가져갈 때
  FOR UPDATE SKIP LOCKED → 다른 인스턴스가 처리 중인 행은 건너뛰고
  자신이 처리할 수 있는 첫 번째 작업을 원자적으로 가져감
  → 중복 처리 없이 분산 처리 가능
```

---

# 자주 만나는 에러

|에러|원인|해결|
|---|---|---|
|`ERROR: current transaction is aborted`|트랜잭션 블록 안에서 에러 후 계속 쿼리|`ROLLBACK;` 후 재시작|
|`ERROR: deadlock detected`|서로 다른 순서로 같은 행들을 잠금|잠금 순서 통일 (PK 오름차순 등)|
|`ERROR: could not obtain lock on row`|`NOWAIT` 옵션인데 이미 잠긴 행|재시도 또는 NOWAIT 제거|
|`P2034` (Prisma)|Serializable 충돌|catch에서 재시도 처리|

```txt
트랜잭션 관련 PostgreSQL 설정값 확인:
  SHOW lock_timeout;         -- 잠금 대기 최대 시간
  SHOW deadlock_timeout;     -- DEADLOCK 감지 주기
  SHOW idle_in_transaction_session_timeout;  -- 트랜잭션 유휴 최대 시간
```