---
aliases:
  - PostgreSQL
  - 패턴
  - 주의사항
  - UNIQUE+NULL
tags:
  - PostgreSQL
  - SQL
related:
  - "[[00_DB_HomePage]]"
  - "[[DB_Transaction]]"
  - "[[NestJS_Prisma_Patterns]]"
---
# PG_Patterns — PostgreSQL 패턴 · 주의사항

>[!info]
>PostgreSQL을 쓰다 마주치는 특유의 동작과 패턴 모음. 
>SQL 표준과 다르게 동작하는 것, Prisma와 연계 시 주의사항. 
>트랜잭션 격리·DEADLOCK → [[PG_Transaction]], Prisma 쿼리 패턴 → [[NestJS_Prisma_Patterns]]

---

# UNIQUE + NULL — 예상과 다른 동작 ⭐️⭐️⭐️⭐️

```txt
PostgreSQL UNIQUE 제약:
  NULL을 "알 수 없는 값"으로 취급 (SQL 표준)
  NULL ≠ NULL → NULL끼리는 중복이 아님
  → UNIQUE 컬럼에 NULL이 여러 개 들어가도 에러가 발생하지 않음
```

```sql
CREATE TABLE stats (
  date    DATE    NOT NULL,
  hour    INTEGER,          -- NULL = 일 합계 의도
  count   INTEGER NOT NULL,
  UNIQUE (date, hour)       -- hour가 NULL이면 UNIQUE 효과 없음!
);

-- 아래 두 행 모두 삽입 가능 (에러 없음)
INSERT INTO stats (date, hour, count) VALUES ('2024-01-15', NULL, 100);
INSERT INTO stats (date, hour, count) VALUES ('2024-01-15', NULL, 200);
-- date + hour(NULL) 조합이 "동일하지 않음"으로 처리됨
```

```txt
왜 이런가:
  SQL 표준: NULL은 "모름/없음" → NULL과 NULL이 같은지 알 수 없음
  UNIQUE 제약은 "같은 값이 두 번 들어가면 안 된다"
  NULL은 값이 없으므로 → 중복 체크 대상이 아님
```

## 해결 — 센티넬 값 (Sentinel Value) ⭐️⭐️⭐️⭐️

```txt
NULL 대신 의미 있는 특수 값을 사용
  hour = 0~23: 시간대별 데이터
  hour = -1:   일 합계 (sentinel — 유효한 hour가 아니므로 명확)

  UNIQUE (date, hour) → -1끼리 중복 에러 정상 발생
```

```sql
-- ✅ 센티넬 값으로 변경
CREATE TABLE stats (
  date    DATE        NOT NULL,
  hour    INTEGER     NOT NULL DEFAULT -1,  -- NULL 대신 -1
  count   INTEGER     NOT NULL,
  UNIQUE (date, hour)  -- -1끼리는 정상적으로 UNIQUE 동작
);
```

```typescript
// Prisma schema
model HourlyStat {
  id     String   @id @default(cuid())
  date   DateTime @db.Date
  hour   Int      @default(-1)   // -1 = 일 합계, 0~23 = 시간대별
  count  Int

  @@unique([date, hour])         // UNIQUE가 의도대로 동작
}

// 일 합계 upsert
await prisma.hourlyStat.upsert({
  where: { date_hour: { date, hour: -1 } },
  create: { date, hour: -1, count },
  update: { count },
});

// 쿼리에서 구분
WHERE hour = -1    // 일 합계만
WHERE hour >= 0    // 시간대별만
```

## PostgreSQL 15+ 대안 — NULLS NOT DISTINCT ⭐️⭐️

```sql
-- PostgreSQL 15 이상에서 NULL도 중복으로 처리
CREATE UNIQUE INDEX ON stats (date, hour) NULLS NOT DISTINCT;
-- → NULL끼리도 동일하게 취급 → 중복 에러 발생
```

```txt
단점:
  PostgreSQL 15 미만에서 사용 불가
  Prisma에서 NULLS NOT DISTINCT를 스키마로 직접 표현 불가
  → 센티넬 값(-1) 방식이 더 범용적
```

---

# ON CONFLICT — Upsert ⭐️⭐️⭐️⭐️

```sql
-- 없으면 INSERT, 있으면 UPDATE (Upsert)
INSERT INTO stats (date, hour, count)
VALUES ('2024-01-15', -1, 100)
ON CONFLICT (date, hour)
DO UPDATE SET
  count = EXCLUDED.count,           -- EXCLUDED = 삽입하려던 새 값
  updated_at = NOW();

-- 충돌 시 아무것도 안 함
INSERT INTO stats (date, hour, count)
VALUES ('2024-01-15', -1, 100)
ON CONFLICT (date, hour)
DO NOTHING;
```

```typescript
// Prisma에서 Upsert
await prisma.hourlyStat.upsert({
  where: { date_hour: { date, hour } },
  create: { date, hour, count },
  update: { count },
});

// createMany + skipDuplicates (DO NOTHING 효과)
await prisma.hourlyStat.createMany({
  data: rows,
  skipDuplicates: true,  // ON CONFLICT DO NOTHING
});
```

---

# RETURNING — INSERT/UPDATE 결과 즉시 반환 ⭐️⭐️⭐️

```sql
-- INSERT 후 생성된 행 반환
INSERT INTO posts (title, user_id)
VALUES ('제목', 1)
RETURNING id, created_at;

-- UPDATE 후 변경된 행 반환
UPDATE posts
SET view_count = view_count + 1
WHERE id = 42
RETURNING view_count;

-- DELETE 후 삭제된 행 반환
DELETE FROM sessions
WHERE expires_at < NOW()
RETURNING id, user_id;
```

```txt
RETURNING의 장점:
  INSERT → id 알기 위해 추가 SELECT 불필요
  UPDATE → 업데이트 후 값 확인을 위한 추가 SELECT 불필요
  → 쿼리 1번으로 결과까지 가져옴

Prisma는 내부적으로 RETURNING을 자동으로 사용
create(), update(), delete() 모두 결과 행을 반환
```

---

# 인덱스 — 언제 필요한가 ⭐️⭐️⭐️⭐️

```sql
-- WHERE, ORDER BY, JOIN에서 자주 쓰는 컬럼에 인덱스
CREATE INDEX idx_posts_user_id ON posts (user_id);
CREATE INDEX idx_posts_created_at ON posts (created_at DESC);

-- 복합 인덱스 — 순서 중요
CREATE INDEX idx_stats_date_hour ON stats (date, hour);
-- → WHERE date = ? AND hour = ? 에 유효
-- → WHERE date = ?               에도 유효 (앞 컬럼 기준)
-- → WHERE hour = ?               에는 비효율 (앞 컬럼 없음)

-- 조건부 인덱스 (Partial Index)
CREATE INDEX idx_active_users ON users (created_at)
WHERE deleted_at IS NULL;
-- deleted_at IS NULL인 행에만 인덱스 → 크기 작고 효율 높음
```

```txt
인덱스가 필요한 신호:
  EXPLAIN ANALYZE에서 Seq Scan이 나오고 테이블이 큰 경우
  WHERE 절에 자주 쓰이는 컬럼
  JOIN의 ON 절 컬럼
  ORDER BY에 자주 쓰이는 컬럼

Prisma schema에서 인덱스:
  @@index([userId])
  @@index([createdAt(sort: Desc)])
```

---

# EXPLAIN ANALYZE — 쿼리 실행 계획 ⭐️⭐️⭐️

```sql
-- 쿼리가 어떻게 실행되는지 확인
EXPLAIN ANALYZE
SELECT * FROM posts WHERE user_id = 1 ORDER BY created_at DESC LIMIT 20;
```

```txt
주요 실행 계획:
  Seq Scan       → 테이블 전체 순회 (인덱스 없음 또는 비효율)
  Index Scan     → 인덱스 사용 (효율적)
  Bitmap Scan    → 인덱스 + 실제 행 조회 (중간)
  Hash Join      → 해시 테이블로 JOIN
  Nested Loop    → 중첩 루프 JOIN (작은 테이블에 효율적)

EXPLAIN vs EXPLAIN ANALYZE:
  EXPLAIN         → 실행 계획만 (실제 실행 안 함)
  EXPLAIN ANALYZE → 실제 실행 + 계획 + 실제 소요 시간
  → EXPLAIN ANALYZE는 실제로 쿼리가 실행되므로 SELECT에만 사용 권장
     (UPDATE/DELETE에 사용 시 실제 데이터가 변경됨)
```
---

# to_regclass — 오브젝트 존재 여부 확인 ⭐️⭐️⭐️

```txt
PostgreSQL 시스템 카탈로그 함수
  존재하면 → 오브젝트 이름(regclass) 반환
  없으면   → NULL 반환 (에러 없이 조용히 반환 — 이게 핵심)

확인 가능한 오브젝트: 테이블 · 인덱스 · 시퀀스 · 뷰
```

```sql
-- 테이블 존재 여부 확인 — migration 적용 후 검증 등에 사용
SELECT to_regclass('public.admin_daily_reports');
--  결과: 'public.admin_daily_reports'  → 존재함
--  결과: NULL                          → 없음 (migration 미적용)

-- 스키마 없이도 가능 (search_path 기준)
SELECT to_regclass('admin_daily_reports');
```

```txt
언제 쓰나:
  prisma migrate deploy 후 테이블 실제 생성 여부 확인
  Railway Shell · DataGrip에서 migration 적용 확인
  조건부 DDL 실행 전 존재 여부 체크

비교:
  to_regclass  → NULL 반환 (에러 없음) — 존재 체크용
  ::regclass   → 없으면 에러 발생      — 존재 확신할 때
```
