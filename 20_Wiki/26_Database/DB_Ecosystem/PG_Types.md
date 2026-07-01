---
aliases:
  - PostgreSQL
  - Database
  - Types
  - SQL
  - TIMESTAMPTZ
  - JSONB
  - UUID
  - TEXT
  - INTEGER
tags:
  - SQL
  - PostgreSQL
related:
  - "[[00_DB_HomePage]]"
  - "[[NestJS_Prisma]]"
  - "[[NestJS_PostgreSQL]]"
---
# PG_Types — PostgreSQL 타입

> [!info] 
> PostgreSQL의 주요 타입 — timestamp/timestamptz · JSONB · uuid · ARRAY · ENUM. "어떤 타입을 왜 쓰는가"와 Prisma 매핑, 각 타입의 함정까지 정리

---

# 타입 한눈에 — Prisma 매핑

| PostgreSQL 타입    | Prisma 어노테이션                  | JS/TS 타입           | 비고                       |
| ---------------- | ----------------------------- | ------------------ | ------------------------ |
| `TIMESTAMPTZ(n)` | `DateTime @db.Timestamptz(n)` | `Date`             | UTC instant — 권장         |
| `TIMESTAMP(n)`   | `DateTime` (기본, 어노테이션 없음)     | `Date`             | timezone-naive — 쓰지 말 것  |
| `JSONB`          | `Json`                        | `Prisma.JsonValue` | 읽기/쓰기 타입이 다름             |
| `UUID`           | `String @db.Uuid`             | `string`           | 안 붙이면 `TEXT`(36자)로 저장    |
| `TEXT[]`         | `String[]`                    | `string[]`         | PostgreSQL 전용, SQLite 불가 |
| `INTEGER[]`      | `Int[]`                       | `number[]`         | 동일                       |
| `enum`           | `enum` 블록                     | enum 타입            | 값 추가 시 `migrate dev` 필수  |

---

# timestamp vs timestamptz ⭐️⭐️⭐️⭐️

## 두 타입 비교

|
|`TIMESTAMP`|`TIMESTAMPTZ`|
|---|---|---|
|정식 명칭|`timestamp without time zone`|`timestamp with time zone`|
|저장 방식|날짜+시간 문자열처럼 — TZ 정보 없음|UTC epoch(ms)로 변환해서 저장|
|INSERT 시|넣은 값 그대로 저장|세션 TZ → UTC로 변환 후 저장|
|SELECT 시|저장된 값 그대로 반환|UTC → 세션 TZ로 변환 후 반환|
|세션 TZ 영향|받음 — 환경마다 해석이 달라짐|받지 않음 — 어디서나 같은 instant|
|Prisma 어노테이션|`DateTime` (기본, 생략 시)|`DateTime @db.Timestamptz(3)`|

```txt
핵심: timestamptz는 "항상 UTC로 저장"이 아니라
  "넣을 때 UTC로 변환 → 꺼낼 때 세션 TZ로 역변환"
  → 어느 환경(KST 서버, UTC 서버, DataGrip)에서 넣어도 항상 같은 물리적 순간을 가리킴

TIMESTAMP의 위험:
  KST 서버에서 "2025-07-01 10:00:00" 저장
  UTC 서버에서 읽으면 "2025-07-01 10:00:00"로 읽음 — TZ 변환 없이 그대로
  → 실제로는 KST 10:00이었는데 UTC 10:00으로 오해하게 됨 (9시간 오차)
```

## precision (n)

```txt
TIMESTAMP(3) / TIMESTAMPTZ(3):
  소수점 이하 자릿수 = 밀리초 단위
  TIMESTAMP(3)  → .000 (밀리초)
  TIMESTAMP(6)  → .000000 (마이크로초) — 생략 시 PostgreSQL 기본값
  실용적으로 3이면 충분, 6은 거의 필요 없음
```

## AT TIME ZONE — 조회 시 TZ 변환

```sql
-- timestamptz 컬럼을 KST 기준으로 표시
SELECT created_at AT TIME ZONE 'Asia/Seoul' FROM "User";

-- KST 기준 날짜별 집계
SELECT
  DATE(created_at AT TIME ZONE 'Asia/Seoul') AS date_kst,
  COUNT(*) AS cnt
FROM "User"
GROUP BY date_kst;
```

```txt
AT TIME ZONE 'Asia/Seoul':
  UTC로 저장된 값을 KST로 변환해서 반환
  → 한국 서비스에서 "오늘 가입자 수" 같은 집계에 필수

DataGrip에서 UTC처럼 보이는 이유:
  DataGrip/JVM이 UTC 기준으로 timestamptz를 표시하기 때문
  → 저장 자체는 올바름, 표시 설정(File → Settings → Database → Timezone)을 바꾸면 됨
  ([[NestJS_PostgreSQL]] 참고)
```

## Prisma 스키마 적용

```prisma
model User {
  createdAt    DateTime  @default(now()) @db.Timestamptz(3)
  updatedAt    DateTime  @updatedAt      @db.Timestamptz(3)
  lastActiveAt DateTime?                 @db.Timestamptz(3)
}
```

```txt
기존 TIMESTAMP → TIMESTAMPTZ 마이그레이션:
  schema.prisma 수정만으로는 DB 타입이 안 바뀜 → prisma migrate deploy 필수

  ALTER TABLE "User"
    ALTER COLUMN "createdAt" TYPE TIMESTAMPTZ(3)
    USING "createdAt" AT TIME ZONE 'UTC';
  
  USING ... AT TIME ZONE 'UTC': 기존 데이터가 UTC로 들어갔다고 가정하고 변환
  ([[NestJS_Prisma]] DateTime 필드 섹션 참고)
```

---

# JSONB ⭐️⭐️⭐️

## JSON vs JSONB

|
|`JSON`|`JSONB`|
|---|---|---|
|저장 방식|텍스트 그대로 (exact copy)|binary로 분해해서 저장|
|저장 시|빠름 (그냥 저장)|느림 (파싱 후 저장)|
|조회 시|느림 (매번 파싱)|빠름 (이미 파싱됨)|
|공백/키 순서|유지됨|유지 안 됨|
|중복 키|마지막 값 유지|중복 키 없음|
|인덱스|불가|GIN 인덱스 가능 ✅|
|실무 선택|거의 안 씀|항상 JSONB|

```txt
Prisma의 Json 필드 → PostgreSQL에서 자동으로 JSONB로 생성됨
→ 직접 타입을 지정할 필요 없음
```

## JSONB 조회 연산자

|연산자|의미|반환|
|---|---|---|
|`->`|키로 값 꺼내기|JSON 타입|
|`->>`|키로 값 꺼내기|TEXT 타입|
|`#>`|경로로 중첩값 꺼내기|JSON 타입|
|`#>>`|경로로 중첩값 꺼내기|TEXT 타입|
|`@>`|왼쪽이 오른쪽을 포함하는가|BOOLEAN|
|`?`|해당 키가 존재하는가|BOOLEAN|

```sql
-- data 컬럼에서 name 키 꺼내기
SELECT data -> 'name'   FROM items;   -- JSON: "Alice"  (따옴표 포함)
SELECT data ->> 'name'  FROM items;   -- TEXT:  Alice   (따옴표 없음)

-- 중첩 경로
SELECT data #>> '{address, city}' FROM items;   -- TEXT: Seoul

-- 특정 값을 포함하는 행 필터
SELECT * FROM items WHERE data @> '{"status": "active"}';

-- 키 존재 여부
SELECT * FROM items WHERE data ? 'email';
```

```txt
-> vs ->>:
  ->  는 JSON 타입 반환 → 비교할 때 CAST 필요 또는 다시 ->> 사용
  ->> 는 TEXT 반환     → WHERE data ->> 'age' = '30' (숫자도 문자열로 비교)
  → 값 비교 시 ->> 를 주로 씀

Prisma에서 JSONB 필터:
  Prisma에서 JSONB 내부 조건 검색은 string_contains / path 연산자로 제한됨
  복잡한 @> 조건이 필요하면 $queryRaw로 직접 SQL 작성
```

## GIN 인덱스 — JSONB 검색 속도

```sql
-- JSONB 전체에 GIN 인덱스
CREATE INDEX idx_items_data ON items USING GIN (data);

-- 특정 키에만 인덱스 (더 작음)
CREATE INDEX idx_items_status ON items USING GIN ((data -> 'status'));
```

```txt
GIN(Generalized Inverted Index):
  JSONB 내부의 키/값 전체를 인덱싱
  @>, ?, ?| 연산자가 인덱스를 탐
  데이터가 많고 JSONB 내부로 자주 검색한다면 GIN 추가 검토

Prisma에서 GIN 인덱스 추가:
  schema.prisma 자체 문법으로는 GIN 인덱스 지정 불가
  → prisma migrate dev로 빈 마이그레이션 생성 후 SQL 직접 작성
```

## Prisma에서의 Json 타입 함정

```txt
Prisma Json 필드는 읽기/쓰기 타입이 다름:
  읽을 때: Prisma.JsonValue
  쓸 때:   Prisma.InputJsonValue

옵셔널 속성 있는 DTO 타입은 InputJsonValue와 구조가 안 맞아서
  dto.field as Prisma.InputJsonValue  로 단언이 자주 필요함
  (자세한 내용은 [[NestJS_Prisma]] "Json 필드" 섹션 참고)
```

---

# UUID ⭐️⭐️⭐️

## TEXT vs 네이티브 UUID

|
|`String` (TEXT)|`String @db.Uuid` (UUID)|
|---|---|---|
|PostgreSQL 타입|`TEXT`|`UUID` (16바이트 고정)|
|저장 크기|36바이트 (하이픈 포함 문자열)|16바이트|
|인덱스/비교|문자열 비교 (느림)|바이너리 비교 (빠름)|
|DB 형식 검증|없음|UUID 형식 아니면 에러|
|문자열 연산|contains · startsWith 가능|불가 (`operator does not exist`)|
|Prisma TS 타입|`string`|`string` (동일)|

```txt
@db.Uuid 붙이는 기준:
  UUID 값으로 부분 검색(contains)이 필요 없다면 → @db.Uuid 붙이기 (더 효율적)
  UUID 값 자체를 LIKE 검색해야 한다면 → @db.Uuid 빼고 TEXT로 (특수 케이스)
```

## uuid() 버전 — v4 vs v7

```prisma
id String @id @default(uuid())    // v4 — 완전 무작위
id String @id @default(uuid(7))   // v7 — 앞부분에 타임스탬프 포함
```

```txt
v4: 완전 랜덤 → 예측 불가, INSERT 시 인덱스 단편화 발생 가능
v7: 생성 시각 순으로 정렬 가능 → INSERT가 많은 테이블의 PK로 유리 (단편화 ↓)
    예측 어려운 정도는 v4와 동일

FK도 같이 @db.Uuid 맞춰야 함:
  PK가 @db.Uuid면 그 PK를 참조하는 FK 컬럼도 반드시 @db.Uuid
  타입이 안 맞으면 관계 생성 시 에러

([[NestJS_Prisma]] "@db.Uuid" 섹션 참고)
```

---

# ARRAY ⭐️⭐️

## 기본 문법

```sql
-- 배열 컬럼 정의
CREATE TABLE posts (
  id      SERIAL PRIMARY KEY,
  tags    TEXT[],          -- 문자열 배열
  scores  INTEGER[]        -- 정수 배열
);

-- 삽입
INSERT INTO posts (tags, scores) VALUES (ARRAY['nestjs', 'postgresql'], ARRAY[90, 85]);

-- 조회
SELECT tags[1] FROM posts;   -- 1-indexed! (0이 아님)
```

```txt
⚠️ PostgreSQL 배열 인덱스는 1부터 시작 (대부분의 언어는 0부터)
```

## 주요 연산자와 함수

|연산자/함수|의미|예시|
|---|---|---|
|`@>`|배열이 오른쪽 배열을 포함|`tags @> ARRAY['nestjs']`|
|`&&`|두 배열이 겹치는 요소가 있는가|`tags && ARRAY['nestjs', 'react']`|
|`= ANY(array)`|값이 배열 안에 있는가|`'nestjs' = ANY(tags)`|
|`array_length(arr, dim)`|배열 길이|`array_length(tags, 1)`|
|`unnest(arr)`|배열을 행으로 펼치기|`SELECT unnest(tags)`|
|`array_agg(col)`|여러 행을 배열로 모으기|`SELECT array_agg(tag)`|

```sql
-- 'nestjs' 태그를 가진 게시글 조회
SELECT * FROM posts WHERE 'nestjs' = ANY(tags);
-- 또는
SELECT * FROM posts WHERE tags @> ARRAY['nestjs'];

-- 태그 행으로 펼치기 (태그별 집계 등에 활용)
SELECT id, unnest(tags) AS tag FROM posts;
```

## Prisma에서 배열 타입

```prisma
model Post {
  tags   String[]    // TEXT[] 로 생성
  scores Int[]       // INTEGER[] 로 생성
}
```

```typescript
// 생성
await this.prisma.post.create({
  data: { tags: ['nestjs', 'postgresql'] }
});

// 배열 안에 특정 값 포함 여부 필터
await this.prisma.post.findMany({
  where: { tags: { has: 'nestjs' } }           // 하나 포함
});

await this.prisma.post.findMany({
  where: { tags: { hasEvery: ['nestjs', 'postgresql'] } }  // 전부 포함
});

await this.prisma.post.findMany({
  where: { tags: { hasSome: ['nestjs', 'react'] } }        // 하나라도 포함
});
```

```txt
⚠️ PostgreSQL 전용 기능 — SQLite, MySQL에서는 배열 타입 없음
   다른 DB로 마이그레이션할 가능성이 있다면 JSONB[]보다 별도 관계 테이블이 더 이식성 높음
```

---

# ENUM ⭐️⭐️

## PostgreSQL ENUM 타입

```sql
-- DB 레벨에서 타입 생성
CREATE TYPE user_role AS ENUM ('USER', 'ADMIN', 'MODERATOR');

-- 테이블에서 사용
CREATE TABLE users (
  id   SERIAL PRIMARY KEY,
  role user_role NOT NULL DEFAULT 'USER'
);
```

```txt
ENUM 장점:
  유효하지 않은 값 INSERT 시 DB가 직접 거부
  저장 공간이 TEXT보다 작음
  ENUM 값 목록을 DB 레벨에서 문서화 효과

ENUM 단점 (주의):
  값 추가 → ALTER TYPE ... ADD VALUE — 트랜잭션 안에서 못 씀 (PostgreSQL 제약)
  값 삭제/순서 변경 → 사실상 불가 (타입 재생성 필요)
  → 자주 바뀔 것 같은 값 목록이라면 ENUM 대신 VARCHAR + CHECK 제약 고려
```

## Prisma에서 ENUM

```prisma
enum UserRole {
  USER
  ADMIN
  MODERATOR
}

model User {
  role UserRole @default(USER)
}
```

```typescript
import { UserRole } from '../generated/prisma/client';
// ⚠️ @prisma/client가 아니라 output으로 지정한 경로에서 import

await this.prisma.user.findMany({
  where: { role: UserRole.ADMIN }
});
```

```txt
Prisma enum 주의사항:
  Prisma Client 생성 경로에서 import해야 함
  다른 파일(예: 이전 TypeORM entity)에서 가져온 enum과 섞으면 타입 불일치 에러

값 추가 시 흐름:
  ① schema.prisma의 enum 블록에 새 값 추가
  ② prisma migrate dev --name add_enum_value
  ③ 생성된 마이그레이션 SQL 확인:
     ALTER TYPE "UserRole" ADD VALUE 'NEW_VALUE';
  ④ 서버 재시작
```

---

# 한눈에

```txt
timestamp vs timestamptz:
  TIMESTAMP  → TZ 정보 없음, 세션에 따라 해석 달라짐 → 쓰지 말 것
  TIMESTAMPTZ → UTC instant 저장 → 항상 일관성 → 채택
  Prisma: @db.Timestamptz(3) 반드시 명시
  KST 집계: AT TIME ZONE 'Asia/Seoul' ([[NestJS_PostgreSQL]] 참고)

JSONB:
  JSON보다 조회 빠름, GIN 인덱스 가능 → 항상 JSONB
  연산자: -> (JSON 반환) · ->> (TEXT 반환) · @> (포함 여부) · ? (키 존재)
  Prisma: Json 필드가 자동으로 JSONB 매핑
  읽기(JsonValue) / 쓰기(InputJsonValue) 타입 다름 ([[NestJS_Prisma]] 참고)

UUID:
  @db.Uuid → 네이티브 16바이트, 더 효율적
  안 붙이면 TEXT(36자) 저장 → 인덱스/비교 느림
  v4: 랜덤 / v7: 시각 순 정렬 (INSERT 많은 PK에 유리)
  FK도 반드시 같이 @db.Uuid 맞추기

ARRAY:
  TEXT[] · INT[] — PostgreSQL 전용
  인덱스 1부터 시작 ⚠️
  Prisma: has / hasEvery / hasSome 필터
  이식성 필요하면 별도 관계 테이블 검토

ENUM:
  유효 값 DB 레벨 검증, 저장 공간 효율
  값 삭제/순서 변경 어려움 → 자주 바뀌면 VARCHAR + CHECK
  Prisma: enum 블록 → migrate dev → Prisma Client 경로에서 import
```