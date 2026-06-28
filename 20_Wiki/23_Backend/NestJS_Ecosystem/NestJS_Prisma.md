---
aliases: [NestJS Prisma, Prisma, Prisma ORM]
tags:
  - NestJS
related:
  - "[[00_NestJS_Ecosystem_HomePage]]"
  - "[[NestJS_Module]]"
  - "[[NestJS_Pagination]]"
  - "[[NestJS_Prisma_Monorepo]]"
  - "[[SQL_Window_Functions]]"
---
# NestJS_Prisma — Prisma ORM

# 한 줄 요약

```txt
Prisma = 타입 안전한 쿼리 + 마이그레이션 ORM
반복 루프: schema 수정 → migrate dev → Client 사용 ⭐️
나머지(Model 문법, Relations, CRUD, where...)는 이 루프 안의 디테일
```

---

## 흐름도

```mermaid
graph TD
    A["schema.prisma 모델/관계 정의"]
    B["migrate dev 실행"]
    C["마이그레이션 SQL 생성 + DB 반영"]
    D["Prisma Client 타입 재생성"]
    E["this.prisma.모델.메서드() 호출"]
    F["타입 안전한 SQL 실행"]
    G{"스키마 또다시 변경?"}

    A --> B --> C --> D --> E --> F --> G
    G -->|예| A
    G -->|아니오| E
```

---

---

# TypeORM vs Prisma

| |TypeORM|Prisma|
|---|---|---|
|정의 방식|Entity 클래스|`schema.prisma` 파일 하나|
|흐름|코드 먼저 → DB 반영|스키마 먼저 → Client 자동 생성|
|타입 안전성|보통|강력(자동완성)|

---

---

# 워크플로우 — 반복 루프 ⭐️⭐️

```txt
① schema.prisma 수정
② npx prisma migrate dev --name 설명용_이름
③ (보통 자동) npx prisma generate
④ 서버 재시작
⑤ this.prisma.모델명.메서드() — 타입 자동완성 즉시 적용
```

```txt
② 한 줄이 하는 일: 변경분만 SQL 마이그레이션 생성 → 개발 DB 에 적용 → Client 코드 재생성
```

> 모노레포(pnpm workspace)에서는 명령 앞에 위치만 맞추면 동일하게 동작 — [[NestJS_Prisma_Monorepo]] 참고

## migrate dev / migrate deploy / generate

|명령|언제|동작|
|---|---|---|
|`migrate dev --name x`|로컬 개발|마이그레이션 생성 + DB 적용 + Client 재생성|
|`migrate deploy`|배포/CI|기존 마이그레이션만 순서대로 적용 (새로 생성 안 함)|
|`generate`|Client만 다시|DB 변경 없이 타입만 재생성|

## Prisma 6 → 7 주요 변경 ⭐️⭐️

|항목|6 (옛 튜토리얼)|7 (현재)|
|---|---|---|
|DB url 위치|`schema.prisma` 안|`prisma.config.ts`|
|generator provider|`prisma-client-js`|`prisma-client` + `output` 필수|
|import 경로|`@prisma/client`|`output` 지정 경로 기준|
|DB 연결|`new PrismaClient()`|adapter 필요 (`@prisma/adapter-pg` 등)|

```txt
인터넷 튜토리얼 대부분 6 기준 — "따라했는데 안 됨" 의 흔한 원인이 이 표의 차이들
```

## 타입이 안 보일 때 체크리스트 ⭐️⭐️

```txt
① schema 에 모델/필드 정확히 들어갔는지
② migrate dev (또는 generate) 실행
③ 서버 재시작 ⭐️ — Node 는 require 한 모듈을 메모리에 캐싱해서, 파일이 새로 생겨도
   이미 떠 있는 프로세스는 재시작 전까지 옛 버전을 그대로 씀 (가장 자주 빠뜨리는 단계)
④ (그래도 안 되면) 에디터 TS 서버 재시작
```

---

---

# 설치 & 초기화

```bash
pnpm add @prisma/client
pnpm add -D prisma
pnpm add @prisma/adapter-pg pg   # Prisma 7+, PostgreSQL 사용 시 필수

npx prisma init   # prisma/schema.prisma + .env 생성
```

> 모노레포라면 명령 위치(cd vs --filter)는 [[NestJS_Prisma_Monorepo]] 참고 — 결과는 동일

## prisma.config.ts (Prisma 7)

```typescript
import "dotenv/config";
import { defineConfig, env } from "prisma/config";

export default defineConfig({
  schema: "prisma/schema.prisma",
  datasource: { url: env("DATABASE_URL") },
});
```

```text
이 파일만 process.env 직접 사용 — CLI(migrate/generate)는 NestJS 부팅 없이 실행되어 ConfigService 를 못 씀
(NestJS 런타임/Prisma CLI/Docker Compose 가 .env 를 각자 다르게 읽는 전체 그림은 [[NestJS_Env_Config]] 참고)
```

## DATABASE_URL 구조 ⭐️

```bash
postgresql://user:password@host:port/dbname?schema=public&sslmode=disable
```

| 구간              | 의미                                            |
| --------------- | --------------------------------------------- |
| `postgresql://` | DB 종류                                         |
| `user:password` | 접속 계정                                         |
| `host:port`     | 접속 주소                                         |
| `dbname`        | 사용할 DB                                        |
| `?schema=`      | 기본값 `public`, 보통 그대로 사용                       |
| `sslmode=`      | 로컬(Docker)→`disable` / 클라우드(Neon 등)→`require` |


```text
⚠️ prisma init 으로 자동 생성된 URL 은 Prisma 클라우드 주소 — 직접 쓰는 DB 주소로 반드시 교체
```

## schema.prisma 기본 구성 (Prisma 7)

```prisma
// prisma/schema.prisma
datasource db {
  provider = "postgresql"
}

generator client {
  provider     = "prisma-client"
  output       = "../src/generated/prisma"
  moduleFormat = "cjs"        // NestJS(CJS 빌드)와 형식 맞추기 위해 필요
}
```

```text
moduleFormat 누락 시 흔한 에러: "exports is not defined in ES module scope"
→ Prisma 7 기본 출력(ESM)과 NestJS 빌드(CJS) 형식 불일치가 원인
```

---

---

# NestJS 연동

```typescript
// prisma.service.ts
import { Injectable, OnModuleDestroy, OnModuleInit } from '@nestjs/common';
import { PrismaClient } from '../generated/prisma/client';
import { PrismaPg } from '@prisma/adapter-pg';
import { ConfigService } from '@nestjs/config';

@Injectable()
export class PrismaService extends PrismaClient implements OnModuleInit, OnModuleDestroy {
  constructor(configService: ConfigService) {
    super({ adapter: new PrismaPg({ connectionString: configService.getOrThrow('DATABASE_URL') }) });
  }
  async onModuleInit() { await this.$connect(); }
  async onModuleDestroy() { await this.$disconnect(); }
}
```

```typescript
// prisma.module.ts
@Module({ providers: [PrismaService], exports: [PrismaService] })
export class PrismaModule {}
```

```text
PrismaService 는 보통 src/prisma/ 에 둠 — schema.prisma(루트의 prisma/) 와는 다른 위치이니 혼동 주의
```

## @Global() — 매 모듈마다 import 안 해도 되게 ⭐️⭐️⭐️

```typescript
// prisma.module.ts
import { Global, Module } from '@nestjs/common';

@Global()
@Module({ providers: [PrismaService], exports: [PrismaService] })
export class PrismaModule {}
```

```txt
@Global() 자체의 동작 원리(왜 한 번만 import 해도 되는지, isGlobal 옵션과의 관계)는 [[NestJS_Module]] 참고
여기서는 PrismaService 에 적용한 예시만 — PrismaService 는 거의 모든 기능 모듈이 필요로 하는 성격이라
@Global() 의 대표적인 적용 후보로 꼽힘 (반대로 일부 모듈만 쓰는 서비스는 명시적 import 가 더 명확함)
```

>참조 [[NestJS_Module]]

---

---

# Model — 테이블 정의

```prisma
model User {
  id        Int      @id @default(autoincrement())
  email     String   @unique
  name      String?                       // ? = nullable
  createdAt DateTime @default(now())
  updatedAt DateTime @updatedAt
  role      Role     @default(USER)
  posts     Post[]                        // 가상 필드, DB 컬럼 아님
}
```

| 어노테이션                       | 의미                                                    |
| --------------------------- | ----------------------------------------------------- |
| `@id`                       | Primary Key                                           |
| `@default(autoincrement())` | 숫자 ID 자동 증가                                           |
| `@default(uuid())`          | UUID 자동 생성, 예측 어려움                                    |
| `@unique`                   | UNIQUE 제약                                             |
| `?`                         | nullable (없으면 `NOT NULL`)                             |
| `@updatedAt`                | 수정 시 자동 갱신                                            |
| `@db.VarChar(n)`            | `VARCHAR(n)` 명시                                       |
| `@db.Uuid`                  | PostgreSQL 네이티브 `uuid` 타입으로 저장 (안 붙이면 `TEXT`) — 아래 참고 |

## @db.Uuid — UUID 컬럼을 네이티브 타입으로 ⭐️⭐️

```prisma
model Like {
  id               String @id @default(uuid()) @db.Uuid
  recommendationId String @db.Uuid   // 이 PK 를 참조하는 FK 도 같은 네이티브 타입으로 맞춰야 함
}
```

```txt
@db.Uuid 없이 String @default(uuid()) 만 쓰면:
  Postgres 컬럼이 TEXT 로 생성됨 — UUID 값(36자 문자열)을 그냥 텍스트로 저장
  → 저장 공간을 더 쓰고, 인덱스/비교 연산도 텍스트 비교라 더 느림

@db.Uuid 를 붙이면:
  Postgres 의 네이티브 uuid 타입(고정 16바이트)으로 컬럼이 생성됨
  → 저장 공간 절약, 인덱스/비교 더 빠름, DB 가 "진짜 UUID 형식인지" 도 같이 검증
  → Prisma Client 쪼에서 보이는 TS 타입은 여전히 string — @db.Uuid 는 DB 컬럼 타입만 바꿈
```

```txt
⚠️ PK 가 @db.Uuid 면, 그 PK 를 참조하는 FK 컬럼도 반드시 같이 @db.Uuid 로 맞춰야 함
   (PK 는 네이티브 uuid, FK 는 그냥 String/TEXT 로 두면 타입이 안 맞아 관계 생성 시 에러)

⚠️ 네이티브 uuid 타입엔 contains/startsWith 같은 문자열 연산자를 못 씀
   (Postgres 의 uuid 타입 자체가 LIKE 류 연산자를 지원 안 해서 "operator does not exist" 에러)
   → UUID 값으로 부분 검색을 해야 하는 특수한 경우라면 @db.Uuid 를 빼고 그냥 String 으로 두는 게 나음
```

## uuid() 버전 — v4(기본) vs v7

```prisma
id String @id @default(uuid())     // v4 — 완전 무작위
id String @id @default(uuid(7))    // v7 — 생성 시각 순으로 정렬됨
```

```txt
v7 은 값 앞부분에 타임스탬프가 들어가 있어서 생성 순서대로 정렬됨
INSERT 가 많은 테이블의 PK 라면 v7 이 인덱스 단편화를 줄여 더 유리 — 추측하기 어려운 정도는 v4 와 동일
```

## @@unique — 복합 유니크

```prisma
model Director {
  name String
  dob  DateTime
  @@unique([name, dob])
}
// findUnique({ where: { name_dob: { name, dob } } })   ⚠️ 필드명_필드명으로 묶임
```

## @@id — 복합 PK ⭐️

```prisma
model MovieLike {
  movieId Int
  userId  Int
  @@id([movieId, userId])   // 같은 쌍 중복 방지
}
```

```txt
컬럼 하나로는 안 유일하지만 합치면 유일한 경우에 사용
같은 조합으로 INSERT 시도 시 DB 가 직접 거부(Prisma 에러 P2002) — 코드의 사전 체크 없이도 안전망 역할
조회: findUnique({ where: { movieId_userId: { movieId, userId } } })
```

## @@index — 인덱스 (자주 찾는다는 힌트) ⭐️

```prisma
@@index([area])              // 단일
@@index([area, feeType])     // 복합 — 순서 중요(앞쪽 컬럼 단독 검색도 효과 있음)
```

```txt
“이 컬럼으로 자주 찾는다”는 힌트
WHERE/ORDER BY 에 자주 쓰는 컬럼에 추가 — 조회는 빨라지지만 쓰기는 약간 느려짐(트레이드오프)
@id/@unique 는 자동으로 인덱스 생성됨, 그 외 컬럼은 수동 추가
```

## @@index vs @@unique — 언제 뭘 쓰나 ⭐️⭐️⭐️

|구분|`@@index`|`@@unique`|
|---|---|---|
|목적|조회 속도만 빠르게|중복 자체를 DB 가 막음 (+ 조회도 빠름)|
|중복 행|허용 (같은 값 여러 행 가능)|불허 (같은 조합 INSERT 시 에러 `P2002`)|
|판단 기준|"이 컬럼으로 자주 검색/정렬하는가"|"이 조합이 두 번 있으면 안 되는가(비즈니스 규칙)"|
|예시|`postId` 로 그 글의 댓글 목록 조회 — 한 글에 여러 행이 당연함|`(userId, postId)` — 한 사람이 같은 글에 좋아요 두 번 못 누름|

```txt
판단 한 줄: 중복이 "버그" 면 @@unique, 중복이 "정상" 인데 그냥 빠르게 찾고 싶으면 @@index

⚠️ @@unique 는 자동으로 인덱스 역할도 겸함 — 같은 컬럼 조합에 @@index 를 따로 또 만들 필요 없음
   (반대로 @@index 는 중복을 막지 못함 — "빠르게 찾기" 만 하고, 데이터 무결성 보장은 없음)
```

---

---

# Scalar Types

|Prisma|PostgreSQL|설명|
|---|---|---|
|`String`|TEXT|문자열|
|`Int` / `BigInt`|INTEGER / BIGINT|정수|
|`Float` / `Decimal`|REAL / NUMERIC|부동소수 / 정밀 소수(금액)|
|`Boolean`|BOOLEAN|true/false|
|`DateTime`|TIMESTAMP|날짜+시간|
|`Json`|JSONB|JSON|

```prisma
enum Role { USER  ADMIN  MODERATOR }
```

```text
⚠️ Enum 도 Prisma Client 생성 경로에서 import — 다른 곳(예: 예전 TypeORM entity 파일)에서 가져오면 타입 불일치
```

---

---

# Relations — 관계

```prisma
// One to Many
model User { posts Post[] }
model Post { authorId Int; 
author User @relation(fields: [authorId], references: [id]) }

// One to One — FK 에 @unique 추가
model Profile { 
userId Int @unique; 
user User @relation(fields: [userId], references: [id]) }

// Many to Many — Prisma 가 중간 테이블 자동 생성
model Post { tags Tag[] }
model Tag  { posts Post[] }
```

```txt
fields: 내가 들고 있는 FK 컬럼 / references: 상대 테이블 PK
```

|onDelete|동작|
|---|---|
|`Cascade`|부모 삭제 → 자식도 삭제|
|`SetNull`|부모 삭제 → FK 를 NULL 로|
|`Restrict` (기본)|참조 중이면 삭제 불가|

## 관계 이름 — `@relation("이름")` ⭐️⭐️⭐️

|구분|역할|
|---|---|
|`@relation(fields, references)`|FK 들고 있는 쪽|
|`@relation("이름")`만|반대편(back-relation) 표시|
|`"문자열"`|DB 와 무관, 양쪽을 짝짓는 Prisma 전용 키 — 양쪽 동일해야 함|

```text
필요 조건: 같은 모델(자기 자신 포함)을 한 모델에서 2번 이상 참조할 때만
```

```prisma
// Friendship 이 User 를 두 번 참조
model Friendship {
  requester User @relation("FriendshipRequester", fields: [requesterId], references: [id])
  addressee User @relation("FriendshipAddressee", fields: [addresseeId], references: [id])
}
model User {
  sentRequests     Friendship[] @relation("FriendshipRequester")
  receivedRequests Friendship[] @relation("FriendshipAddressee")
}
```

```text
안 붙이면 → Ambiguous relation detected 에러
같은 패턴: self-relation(teacher/students 둘 다 User), 한 모델 안 관계 2개(author/pinnedBy 둘 다 User)
```

---

---

# findUnique vs findFirst vs findMany ⭐️

```txt
조건이 @id/@unique 컬럼 딱 하나   → findUnique
조건 자유롭고 결과 1건           → findFirst (NOT 포함 가능)
여러 건                         → findMany
```

| |조건|결과|
|---|---|---|
|`findUnique`|**반드시** unique 컬럼만|단건 또는 `null`|
|`findFirst`|자유 (`NOT` 포함)|첫 행 또는 `null`|
|`findMany`|자유|배열|

```typescript
this.prisma.user.findUnique({ where: { id } });
this.prisma.user.findFirst({ where: { name, NOT: { id } } });  // 자기 자신 제외 중복 체크
this.prisma.movie.findMany({ where: { isVisible: true } });
```

---

---

# CRUD 기본

```typescript
// CREATE
await this.prisma.user.create({ data: { email, passwordHash } });

// READ
const user = await this.prisma.user.findUnique({ where: { id } });
const list = await this.prisma.movie.findMany({ where: { isVisible: true }, orderBy: { createdAt: 'desc' }, take: 10 });

// UPDATE
await this.prisma.user.update({ where: { id }, data: { name } });

// UPSERT — 있으면 update, 없으면 create
await this.prisma.user.upsert({ where: { email }, create: { email, passwordHash }, update: { passwordHash } });

// DELETE
await this.prisma.user.delete({ where: { id } });
await this.prisma.movie.deleteMany({ where: { isVisible: false } });
```

---

---

# where — 조건 연산자

```typescript
where: {
  views: { gt: 100, gte: 100, lt: 100, lte: 100 },
  email: { contains: 'gmail' },        // LIKE '%gmail%'
  name:  { startsWith: 'A' },
  role:  { in: [Role.ADMIN, Role.USER] },
  deletedAt: null,                     // IS NULL
}
```

|연산자|SQL|
|---|---|
|`gt`/`gte`/`lt`/`lte`|`>` `>=` `<` `<=`|
|`contains`/`startsWith`/`endsWith`|`LIKE`|
|`in`/`notIn`|`IN (...)`|

```typescript
// mode: 대소문자 무시 (PostgreSQL 한정, MySQL 은 기본값이 무시라 불필요)
{ title: { contains: 'art', mode: 'insensitive' } }

// AND(기본) / OR / NOT
where: { isVisible: true, genre: 'drama' }                              // AND
where: { OR: [{ name: { contains: '김' } }, { email: { contains: 'gmail' } }] }
where: { name, dob, NOT: { id } }   // findFirst 에서만 가능
```

---

---

# select / omit / include ⭐️⭐️

|키워드|용도|
|---|---|
|`select`|가져올 필드만 지정|
|`omit`|뺄 필드만 지정 (나머지 전부)|
|`include`|관계(연결된 테이블) 함께 조회|

```typescript
this.prisma.user.findMany({ select: { id: true, email: true } });
this.prisma.user.findMany({ omit: { password: true } });     // password 만 빼고 전부

this.prisma.user.findUnique({ where: { id }, include: { posts: true } });  // 관계 통째로
```

## include 안에서 필터·정렬·select 같이 쓰기 ⭐️⭐️⭐️

```typescript
this.prisma.user.findUnique({
  where: { id },
  include: {
    posts: {
      where: { isVisible: true },          // 관계 안에서도 조건 필터링
      orderBy: { createdAt: 'desc' },      // 정렬
      take: 5,                             // 페이지네이션
      select: { id: true, title: true },   // 관계 쪽 필드도 일부만
    },
  },
});
```

```text
include 안의 관계 필드도 findMany 와 같은 옵션(where/orderBy/take/skip/select)을 그대로 받음
→ "관계 = 또 하나의 작은 조회" 로 생각하면 됨
```

## `_count` — 개수만 필요할 때 ⭐️

```typescript
this.prisma.user.findUnique({
  where: { id },
  include: { _count: { select: { posts: true } } },
});
// → { ...user, _count: { posts: 12 } }
```

## 중첩 include — 관계의 관계까지

```typescript
include: { posts: { include: { tags: true } } }
```

|규칙|내용|
|---|---|
|`select` + `include`|동시 사용 불가|
|include 안 옵션|`where`/`orderBy`/`take`/`skip`/`select`/`include` 전부 가능|
|관계 개수만|`_count: { select: { 관계명: true } }`|

```text
⚠️ include 를 깊게/넓게 쓸수록 한 번에 가져오는 데이터가 커짐 — 화면에 필요한 만큼만
```

---

---

# orderBy / take / skip

```typescript
this.prisma.post.findMany({
  orderBy: [{ views: 'desc' }, { createdAt: 'desc' }],   // 동률이면 다음 기준
  take: 10,    // LIMIT
  skip: (page - 1) * 10,   // OFFSET
});
```

> 커서 기반 페이지네이션은 [[NestJS_Pagination]] 참고

---

---

# count / aggregate / groupBy

```typescript
const count = await this.prisma.user.count({ where: { role: Role.USER } });

const stats = await this.prisma.post.aggregate({
  _count: { id: true }, _avg: { views: true }, _max: { views: true },
});

const rows = await this.prisma.exhibition.groupBy({
  by: ['area'], _count: { _all: true },
});
```

| |역할|
|---|---|
|`aggregate`|전체(또는 where 조건)를 한 번에 집계|
|`groupBy`|컬럼 값 기준으로 그룹별 집계 (SQL `GROUP BY`)|

---

---

# Prisma namespace 타입

```typescript
import { Prisma } from '../generated/prisma/client';

Prisma.PostCreateInput   // create data 타입
Prisma.PostUpdateInput   // update data 타입 (전 필드 optional)
Prisma.PostWhereInput    // where 조건 타입
```

```typescript
// 조건별로 다른 필드 조합할 때 활용
function buildUpdateInput(existing: Post | null, incoming: Dto): Prisma.PostUpdateInput {
  return existing ? { title: incoming.title } : { title: incoming.title, isVisible: true };
}
```

---

---

# $transaction — 트랜잭션

|방식|특징|
|---|---|
|인터랙티브 (권장)|콜백 안에서 조건 분기, 실패 시 throw 로 자동 롤백|
|배치|배열로 단순 묶음만, 분기 불가|

```typescript
// 인터랙티브 — 콜백 안에서는 this.prisma 대신 매개변수 prisma 사용
return this.prisma.$transaction(async (prisma) => {
  const movie = await prisma.movie.findUnique({ where: { id } });
  if (!movie) throw new NotFoundException('없음');
  return prisma.movie.update({ where: { id }, data: dto });
});

// 배치
await this.prisma.$transaction([
  this.prisma.movie.update({ where: { id }, data: dto }),
  this.prisma.movieLog.create({ data: { movieId: id } }),
]);
```

---

---

# 에러 처리 — Prisma 에러 코드 ⭐️

```typescript
try {
  await this.prisma.user.create({ data: { email } });
} catch (error) {
  if (error instanceof Prisma.PrismaClientKnownRequestError && error.code === 'P2002') {
    throw new ConflictException('이미 사용 중인 이메일입니다.');
  }
  throw error;
}
```

|코드|의미|보통 던지는 예외|
|---|---|---|
|`P2002`|Unique 제약 위반|`ConflictException`|
|`P2003`|FK 제약 위반|`BadRequestException`|
|`P2025`|대상 없음|`NotFoundException`|

```txt
error.meta → P2002 면 { target: ['email'] } 처럼 어떤 컬럼이 중복인지 알려줌
```

---

---

# 자주 만나는 에러

|증상|원인|해결|
|---|---|---|
|복합 unique 에러|`findUnique({ where: { name, dob } })` 처럼 따로 넘김|`findUnique({ where: { name_dob: { name, dob } } })`|
|TypeORM 문법 사용|`find({ relations: {...} })`|`findMany({ include: {...} })`|
|타입 자동완성 안 바뀜|`migrate dev`/`generate` 누락 또는 서버 재시작 안 함|위 체크리스트 참고|
|`Ambiguous relation detected`|같은 모델을 두 번 참조하는데 관계 이름 없음|충돌하는 두 필드에 `@relation("이름")`, 양쪽 동일하게|

---

---

# TypeORM ↔ Prisma 메서드 대조

|TypeORM|Prisma|용도|
|---|---|---|
|`findOne({ where })`|`findUnique({ where })`|단건(unique)|
|`find({ where })`|`findMany({ where })`|목록|
|`save(entity)`|`create({ data })`|생성|
|`update({ id }, dto)`|`update({ where, data })`|수정|
|`save()`(있으면 update)|`upsert({ where, create, update })`|upsert|
|`delete({ id })`|`delete({ where })`|삭제|

---

---

# 한눈에

```txt
반복 루프: ① schema 수정 → ② migrate dev → ③ 사용
타입 안 보이면: migrate dev/generate 했는지 → 서버 재시작했는지 순서로 확인 ⭐️

같은 모델을 2번 이상 참조하면 @relation("이름") 필수, 양쪽 동일 문자열

조회: unique 컬럼 하나 → findUnique / 자유조건+1건 → findFirst / 여러 건 → findMany
select(필요한 것만) / omit(뺄 것만) / include(관계, 내부에서 where·orderBy·take·select 다 가능)
  → select+include 동시 사용 불가, 개수만 필요하면 _count

에러는 Prisma.PrismaClientKnownRequestError + error.code 분기 (P2002 중복 / P2003 FK / P2025 없음)

Prisma 6 튜토리얼대로 안 되면 버전 차이부터 의심 (prisma.config.ts / adapter / output)
모노레포 명령 실행 디테일 →  [[NestJS_Prisma_Monorepo]]
```