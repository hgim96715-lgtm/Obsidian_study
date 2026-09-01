---
aliases: [서브쿼리, CTE(WITH), DELETE, INSERT, JOIN, SELECT, UPDATE]
tags: [SQL, PostgreSQL]
related:
  - "[[00_DB_HomePage]]"
  - "[[NestJS_CacheTable]]"
  - "[[NestJS_PostgreSQL]]"
  - "[[NestJS_Prisma]]"
  - "[[PG_Aggregate]]"
  - "[[PG_DDL]]"
  - "[[PG_StringFunctions]]"
  - "[[PG_Types]]"
---
# PG_DML — PostgreSQL 데이터 조작 언어

>[!info]
>DML(Data Manipulation Language) = 데이터를 조회·삽입·수정·삭제하는 SQL.
> `SELECT`·`INSERT`·`UPDATE`·`DELETE`. 
> DDL이 테이블 구조(그릇)를 만든다면 DML은 그 안의 데이터를 다룬다 → [[PG_DDL]]. 
> 집계(GROUP BY·윈도우 함수) → [[PG_Aggregate]]

---

# DML이란 ⭐️⭐️⭐️⭐️

```txt
DDL (Data Definition Language) — 구조 정의
  CREATE TABLE, ALTER TABLE, DROP TABLE

DML (Data Manipulation Language) — 데이터 다루기
  SELECT  → 조회
  INSERT  → 삽입
  UPDATE  → 수정
  DELETE  → 삭제

비유:
  DDL = 창고를 만들고 선반을 설치
  DML = 창고에 물건을 넣고 꺼내고 바꾸고 버리는 것
```

---

# SELECT — 조회 ⭐️⭐️⭐️⭐️

## 기본 구조

```sql
SELECT 컬럼들
FROM 테이블
WHERE 조건
ORDER BY 정렬
LIMIT 개수 OFFSET 건너뛸 수;
```

```sql
-- 전체 컬럼
SELECT * FROM posts;

-- 특정 컬럼만
SELECT id, title, created_at FROM posts;

-- 별칭(alias)
SELECT
  u.id,
  u.nickname  AS name,     -- name이라는 별칭
  p.title     AS post_title
FROM users u                -- users를 u로 줄임
JOIN posts p ON p.author_id = u.id;
```

## WHERE — 조건 필터링 ⭐️⭐️⭐️⭐️

```sql
-- 기본 비교
WHERE status = 'active'
WHERE view_count > 100
WHERE created_at >= '2024-01-01'

-- 여러 조건
WHERE status = 'active' AND author_id = 'uuid'   -- 둘 다
WHERE status = 'draft'  OR  status = 'active'    -- 하나라도

-- NULL 체크 (= NULL은 안 됨)
WHERE deleted_at IS NULL      -- 삭제 안 된 것만
WHERE deleted_at IS NOT NULL  -- 삭제된 것만

-- 범위
WHERE created_at BETWEEN '2024-01-01' AND '2024-12-31'
WHERE id IN ('uuid1', 'uuid2', 'uuid3')
WHERE id NOT IN ('uuid1', 'uuid2')

-- 문자열 패턴
WHERE title LIKE '%검색어%'         -- 대소문자 구분
WHERE title ILIKE '%검색어%'        -- 대소문자 무시 (PostgreSQL 전용)
WHERE title ILIKE '검색어%'         -- 시작하는 것
```

## ORDER BY · LIMIT · OFFSET ⭐️⭐️⭐️⭐️

```sql
-- 정렬
ORDER BY created_at DESC          -- 최신순
ORDER BY created_at ASC           -- 오래된 순
ORDER BY created_at DESC, id DESC -- 같은 시각이면 id로 정렬 (안정적 정렬)

-- 개수 제한
LIMIT 20                          -- 20개만
LIMIT 20 OFFSET 40                -- 41번째부터 20개 (페이지 3)

-- OFFSET 페이지네이션의 한계:
-- 데이터가 많을수록 느려짐 → 커서 페이지네이션 권장 → [[NestJS_Pagination]]
```

## ORDER BY RANDOM() — 랜덤 정렬 ⭐️⭐️⭐️

```sql
-- 전체에서 랜덤으로 5개 선택
SELECT * FROM movies
ORDER BY RANDOM()
LIMIT 5;

-- 조건 필터 후 랜덤 선택
SELECT * FROM resource_pool
WHERE is_active = true
  AND tags && ARRAY['horror']   -- 태그 필터
ORDER BY RANDOM()
LIMIT 10;

-- 이미 선택된 항목 제외 후 랜덤
SELECT * FROM resource_pool
WHERE id != ALL(ARRAY['id1', 'id2'])  -- 제외 목록
ORDER BY RANDOM()
LIMIT 1;
```

```txt
RANDOM():
  PostgreSQL 내장 함수 — 0.0 이상 1.0 미만의 무작위 실수 반환
  ORDER BY RANDOM() = 행마다 무작위 값을 붙여서 정렬
  → 매번 다른 순서 보장

LIMIT과 함께 쓰면:
  무작위 순서 중 앞에서 N개 → 공정한 랜덤 샘플링

성능 주의:
  테이블 전체를 정렬한 뒤 자름 → 대용량에서 느릴 수 있음
  수만 건 이하면 실용적으로 충분
  수십만 건 이상이면 → TABLESAMPLE 또는 랜덤 ID 방식 고려

뽑기·추천 기능에서:
  외부 API는 순서 고정 → RANDOM() 불가
  DB 캐시 테이블(Pool)에 저장해두면 RANDOM() 가능
  → [[NestJS_CacheTable]] 외부 API 캐시 테이블 패턴
```

---

# JOIN — 테이블 연결 ⭐️⭐️⭐️⭐️

```txt
JOIN = 두 테이블을 연결해서 함께 조회
FK(외래 키) 관계를 기준으로 연결

posts.author_id → users.id 라면:
  JOIN users ON users.id = posts.author_id
```

## INNER JOIN

```sql
-- 두 테이블 모두에 일치하는 행만 반환
SELECT
  p.title,
  u.nickname AS author
FROM posts p
INNER JOIN users u ON u.id = p.author_id;
-- author_id가 NULL이거나 해당 user가 없는 posts는 결과에서 제외
```

## LEFT JOIN

```sql
-- 왼쪽 테이블(posts)은 전부, 오른쪽(users)은 일치하면 붙임
SELECT
  p.title,
  u.nickname AS author   -- user가 없으면 NULL
FROM posts p
LEFT JOIN users u ON u.id = p.author_id;
-- author_id가 NULL이어도 post는 반드시 포함됨
```

```txt
INNER JOIN vs LEFT JOIN:
  INNER JOIN → 양쪽 다 있는 것만 (교집합)
  LEFT JOIN  → 왼쪽은 전부, 오른쪽은 있으면 붙임

  게시글 목록을 가져오는데 작성자 정보도 같이:
  작성자가 탈퇴해서 없어도 게시글은 보여야 한다 → LEFT JOIN
  작성자 없는 게시글은 보여줄 필요 없다 → INNER JOIN
```

## 여러 테이블 JOIN

```sql
SELECT
  p.title,
  u.nickname                AS author,
  COUNT(c.id)               AS comment_count,
  COUNT(DISTINCT l.user_id) AS like_count
FROM posts p
LEFT JOIN users    u ON u.id = p.author_id
LEFT JOIN comments c ON c.post_id = p.id AND c.deleted_at IS NULL
LEFT JOIN likes    l ON l.post_id = p.id
WHERE p.deleted_at IS NULL
GROUP BY p.id, u.nickname
ORDER BY p.created_at DESC
LIMIT 20;
```

---

# 서브쿼리 ⭐️⭐️⭐️

```sql
-- IN 서브쿼리 — 친구의 게시글만
SELECT * FROM posts
WHERE author_id IN (
  SELECT friend_id FROM friendships
  WHERE user_id = 'my-uuid' AND status = 'accepted'
);

-- EXISTS 서브쿼리 — 댓글 달린 게시글만
SELECT * FROM posts p
WHERE EXISTS (
  SELECT 1 FROM comments c
  WHERE c.post_id = p.id
);

-- 스칼라 서브쿼리 — 단일 값 반환
SELECT
  p.*,
  (SELECT COUNT(*) FROM comments WHERE post_id = p.id) AS comment_count
FROM posts p;
```

---

# INSERT ⭐️⭐️⭐️⭐️

```sql
-- 단건 삽입
INSERT INTO posts (title, content, author_id)
VALUES ('제목', '내용', 'uuid');

-- 여러 건 삽입
INSERT INTO posts (title, content, author_id) VALUES
  ('제목1', '내용1', 'uuid1'),
  ('제목2', '내용2', 'uuid2'),
  ('제목3', '내용3', 'uuid3');

-- 삽입 후 결과 반환 (RETURNING)
INSERT INTO posts (title, author_id)
VALUES ('제목', 'uuid')
RETURNING id, created_at;  -- 생성된 id와 created_at 즉시 반환
```

## ON CONFLICT — Upsert ⭐️⭐️⭐️⭐️

```sql
-- 충돌 시 업데이트 (Upsert)
INSERT INTO stats (date, hour, count)
VALUES ('2024-01-15', -1, 100)
ON CONFLICT (date, hour)
DO UPDATE SET
  count      = EXCLUDED.count,    -- EXCLUDED = 삽입하려던 새 값
  updated_at = NOW();

-- 충돌 시 무시
INSERT INTO room_members (room_id, user_id)
VALUES ('room-uuid', 'user-uuid')
ON CONFLICT (room_id, user_id)
DO NOTHING;
```

```txt
EXCLUDED:
  ON CONFLICT DO UPDATE에서 쓰는 특수 테이블
  "삽입하려고 했던 새 값"을 가리킴
  EXCLUDED.count = 삽입하려던 count 값
```

---

# UPDATE ⭐️⭐️⭐️⭐️

## 기본 구조


```sql
UPDATE 테이블명
SET    컬럼 = 값
WHERE  조건;
```

```txt
UPDATE 다음에 오는 것:
  테이블명 — 어느 테이블의 행을 수정할지
  (FROM이나 JOIN이 아닌, 수정 대상 테이블)

SET 다음에 오는 것:
  컬럼 = 새값  형태로 변경할 컬럼과 값 나열
  여러 컬럼은 쉼표로 구분

WHERE 다음에 오는 것:
  어떤 행을 수정할지 조건
  없으면 모든 행이 수정됨 → 반드시 확인
```

## 테이블명 따옴표 — "users" vs users

```sql
-- Prisma의 @@map("users")로 생성된 테이블
UPDATE "users"
SET role = 'admin'
WHERE email = 'test@example.com';

-- 소문자 단순 이름이면 따옴표 없어도 됨
UPDATE posts
SET view_count = view_count + 1
WHERE id = 'uuid';
```

```txt
PostgreSQL 테이블명 따옴표 규칙:
  따옴표 없음 → PostgreSQL이 자동으로 소문자로 변환
    users, USERS, Users 전부 → users로 해석

  따옴표 있음("") → 대소문자 그대로 유지
    "Users" → Users (대문자 U)
    "users" → users (소문자)

Prisma의 @@map("users")로 만든 테이블:
  테이블명이 소문자 "users"로 생성됨
  → UPDATE "users" 또는 UPDATE users 둘 다 작동

Prisma가 @@map 없이 model User로 만든 테이블:
  테이블명이 "User" (대문자 U)로 생성됨
  → UPDATE "User" (따옴표 필요)
  → UPDATE User → user로 해석 → 테이블 없음 에러
```

## 예시들

```sql
-- 역할 변경 (DataGrip이나 터미널에서 직접 실행)
UPDATE "users"
SET role = 'admin'
WHERE email = 'test@example.com';

-- 단일 컬럼 수정
UPDATE posts
SET view_count = view_count + 1
WHERE id = 'uuid';

-- 여러 컬럼 동시 수정
UPDATE posts
SET
  title      = '새 제목',
  updated_at = NOW()
WHERE id = 'uuid' AND author_id = 'user-uuid';

-- 업데이트 후 결과 반환
UPDATE posts
SET view_count = view_count + 1
WHERE id = 'uuid'
RETURNING view_count;  -- 업데이트된 값 즉시 반환

-- 다른 테이블 값으로 업데이트
UPDATE posts
SET author_name = u.nickname
FROM users u
WHERE posts.author_id = u.id;
```

```txt
⚠️ WHERE 없이 UPDATE하면 전체 행이 변경됨
  UPDATE "users" SET role = 'admin'   ← WHERE 없음 → 모든 유저가 admin!
  → WHERE를 항상 확인하고 실행
  → DataGrip에서 실행 전 눈으로 한 번 더 확인
```
---

# DELETE ⭐️⭐️⭐️⭐️

```sql
-- 특정 행 삭제
DELETE FROM posts
WHERE id = 'uuid';

-- 조건부 삭제
DELETE FROM sessions
WHERE expires_at < NOW();

-- 삭제 후 반환
DELETE FROM notifications
WHERE user_id = 'uuid' AND is_read = true
RETURNING id;

-- 관련 데이터 함께 삭제 (서브쿼리)
DELETE FROM comments
WHERE post_id IN (
  SELECT id FROM posts WHERE author_id = 'user-uuid'
);
```

```txt
DELETE vs TRUNCATE:
  DELETE FROM posts WHERE ...  → 조건부 삭제, 느림, ROLLBACK 가능
  TRUNCATE posts               → 전체 삭제, 매우 빠름, ROLLBACK 불가 (주의)

소프트 삭제 패턴 (deleted_at):
  실제로 행을 지우지 않고 deleted_at 타임스탬프를 기록
  복구 가능, 감사 로그 유지
  UPDATE posts SET deleted_at = NOW() WHERE id = 'uuid'
  조회 시: WHERE deleted_at IS NULL
```

## FK RESTRICT 에러 — 참조된 행 삭제 시 ⭐️⭐️⭐️⭐️

```txt
에러:
  ERROR: update or delete on table "users" violates RESTRICT setting of
  foreign key constraint "tickets_user_id_fkey" on table "tickets"
  Detail: Key (id)=(uuid) is referenced from table "tickets".

원인:
  tickets.user_id → users.id 외래키가 RESTRICT로 설정되어 있음
  tickets에서 해당 user_id를 참조하는 행이 존재하는 상태에서
  users 행을 삭제하려고 했기 때문
  → PostgreSQL이 참조 무결성 위반으로 차단

RESTRICT vs CASCADE:
  RESTRICT  → 참조하는 자식 행이 있으면 삭제 차단 (기본값)
  CASCADE   → 부모 삭제 시 자식 행도 자동 삭제
  SET NULL  → 부모 삭제 시 자식의 FK 컬럼을 NULL로 변경 (nullable일 때)
```

```sql
-- ❌ 이렇게 하면 RESTRICT 에러
DELETE FROM users WHERE email = 'test@example.com';
```

### 즉각적 해결 — 자식 먼저 삭제 후 부모 삭제

```sql
-- 1단계: 참조하는 자식 레코드 먼저 삭제
DELETE FROM tickets
WHERE user_id = '01a03b5d-cba9-774b-8893-ef77205550bf';

-- 2단계: 부모 레코드 삭제
DELETE FROM users
WHERE email = 'test@example.com';
```

```sql
-- user_id를 모르면 서브쿼리로
DELETE FROM tickets
WHERE user_id = (SELECT id FROM users WHERE email = 'test@example.com');

DELETE FROM users
WHERE email = 'test@example.com';
```

### 트랜잭션으로 안전하게 (권장) ⭐️⭐️⭐️

```sql
BEGIN;

-- 자식 테이블들 먼저 삭제 (FK가 걸린 테이블 순서대로)
DELETE FROM tickets
WHERE user_id = (SELECT id FROM users WHERE email = 'test@example.com');

-- 다른 자식 테이블이 더 있으면 계속
-- DELETE FROM orders WHERE user_id = (...);
-- DELETE FROM reviews WHERE user_id = (...);

-- 마지막에 부모 삭제
DELETE FROM users
WHERE email = 'test@example.com';

COMMIT;
-- 중간에 문제가 생기면 ROLLBACK; 으로 전부 취소
```

```txt
트랜잭션으로 묶는 이유:
  자식 삭제 후 부모 삭제 사이에 오류가 생기면
  자식만 삭제된 채로 남아버릴 수 있음 (데이터 불일치)
  → BEGIN/COMMIT으로 묶으면 전부 성공하거나 전부 취소됨
```

### 근본적 해결 — CASCADE 설정 (DDL 변경)

```sql
-- 기존 FK 제약 제거 후 CASCADE로 재생성
ALTER TABLE tickets
  DROP CONSTRAINT tickets_user_id_fkey;

ALTER TABLE tickets
  ADD CONSTRAINT tickets_user_id_fkey
  FOREIGN KEY (user_id)
  REFERENCES users(id)
  ON DELETE CASCADE;   -- 부모 삭제 시 자식도 자동 삭제
```

```txt
CASCADE 적합한 경우:
  부모가 사라지면 자식도 의미 없는 데이터 (ex. 유저 탈퇴 시 티켓도 삭제)

RESTRICT(기본값) 적합한 경우:
  자식이 남아있어야 하는 데이터 (ex. 주문이 있는 유저는 실수로 삭제 방지)

Prisma schema에서:
  @relation(fields: [userId], references: [id], onDelete: Cascade)
  @relation(fields: [userId], references: [id], onDelete: Restrict) ← 기본값
```

### 참조 관계 확인

```sql
-- 어떤 테이블이 users를 참조하는지 확인
SELECT
  tc.table_name       AS 자식테이블,
  kcu.column_name     AS FK컬럼,
  rc.delete_rule      AS 삭제규칙
FROM information_schema.table_constraints tc
JOIN information_schema.key_column_usage kcu
  ON tc.constraint_name = kcu.constraint_name
JOIN information_schema.referential_constraints rc
  ON tc.constraint_name = rc.constraint_name
JOIN information_schema.table_constraints tc2
  ON rc.unique_constraint_name = tc2.constraint_name
WHERE tc.constraint_type = 'FOREIGN KEY'
  AND tc2.table_name = 'users';   -- 부모 테이블명
```

---

# 자주 쓰는 패턴

## 최신 N개 페이지네이션

```sql
-- Offset 방식 (단순하지만 느려질 수 있음)
SELECT * FROM posts
WHERE deleted_at IS NULL
ORDER BY created_at DESC, id DESC
LIMIT 20 OFFSET 0;

-- Cursor 방식 (더 빠름)
SELECT * FROM posts
WHERE
  deleted_at IS NULL
  AND (
    created_at < '2024-01-15T09:30:00Z'
    OR (created_at = '2024-01-15T09:30:00Z' AND id < 'last-uuid')
  )
ORDER BY created_at DESC, id DESC
LIMIT 20;
```

## 소프트 삭제 + 조회

```sql
-- 삭제 안 된 것만 조회
SELECT * FROM posts
WHERE deleted_at IS NULL
ORDER BY created_at DESC;

-- 삭제 포함 전체 조회 (관리자용)
SELECT
  *,
  CASE WHEN deleted_at IS NOT NULL THEN '삭제됨' ELSE '활성' END AS status
FROM posts
ORDER BY created_at DESC;
```

## EXISTS vs COUNT

```sql
-- 존재 여부 확인 — EXISTS가 더 빠름
SELECT EXISTS(
  SELECT 1 FROM likes
  WHERE post_id = 'uuid' AND user_id = 'user-uuid'
) AS is_liked;

-- COUNT는 전체를 세므로 단순 존재 여부엔 과함
SELECT COUNT(*) > 0 FROM likes  -- ← 비효율
```