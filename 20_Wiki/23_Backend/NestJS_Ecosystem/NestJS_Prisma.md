---
aliases:
  - NestJS Prisma
  - Prisma
  - Prisma ORM
  - Model
  - $queryRaw · $executeRaw
tags:
  - NestJS
related:
  - "[[00_NestJS_Ecosystem_HomePage]]"
  - "[[NestJS_Migration]]"
  - "[[NestJS_Prisma_Patterns]]"
  - "[[NestJS_Service_Provider]]"
  - "[[PG_DDL]]"
  - "[[NestJS_PostgreSQL]]"
---
# NestJS_Prisma — Prisma ORM

>[!info]
>Prisma는 타입 안전한 쿼리 + 마이그레이션 ORM이다. 
>schema.prisma에 모델을 정의하면 `migrate dev` 한 번으로 DB 반영 + Client 자동 생성까지 된다.
> `Prisma.RoomWhereInput` 같은 namespace 타입으로 조건을 타입 안전하게 조립하고, 중첩 where로 관계 테이블 필드까지 필터링할 수 있다. 
> 설치·워크플로우·migrate → [[NestJS_Migration]], 패턴 → [[NestJS_Prisma_Patterns]], Prisma가 생성하는 DDL 개념 → [[PG_DDL]]

---

# 읽기 shape — 언제 뭘 쓰나 ⭐️⭐️⭐️

```txt
"지금 조회하는 모델" vs "relations로 연결된 다른 테이블" 한 가지만 기준으로 생각하면 됨

① 지금 모델의 컬럼만 필요 → select
   민감 필드를 빼고 싶을 때 (화이트리스트)
   아무 옵션 없으면 scalar는 전부, 관계는 안 옴

② 연결된 테이블의 실제 행이 필요 → include
   User + 그 User가 작성한 Post 목록
   include 안에서 where / orderBy / select도 가능

③ 연결된 것이 몇 개인지만 필요 → _count
   "댓글 12개"처럼 숫자만 — 행을 통째로 안 가져와서 가벼움
   include 안에 넣거나 select 안에 넣어도 됨

⚠️ 최상위에서 select + include 동시 ❌
   select + _count ✅
```

---

# TypeORM vs Prisma

| |TypeORM|Prisma|
|---|---|---|
|정의 방식|Entity 클래스|`schema.prisma` 파일 하나|
|흐름|코드 먼저 → DB 반영|스키마 먼저 → Client 자동 생성|
|타입 안전성|보통|강력 (자동완성)|

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

```txt
PrismaService는 보통 src/prisma/에 둠
schema.prisma(루트의 prisma/)와는 다른 위치이니 혼동 주의
```

## @Global() — 매 모듈마다 import 안 해도 되게 ⭐️⭐️⭐️

```typescript
import { Global, Module } from '@nestjs/common';

@Global()
@Module({ providers: [PrismaService], exports: [PrismaService] })
export class PrismaModule {}
```

```txt
@Global() 동작 원리(왜 한 번만 import해도 되는지) → [[NestJS_Module]]
PrismaService는 거의 모든 기능 모듈이 필요로 하는 성격이라 @Global()의 대표적인 적용 후보
반대로 일부 모듈만 쓰는 서비스는 명시적 import가 더 명확함
```

---
# schema.prisma 기본 구조 (Prisma 7) ⭐️⭐️⭐️⭐️

```prisma
// prisma/schema.prisma
generator client {
  provider     = "prisma-client"
  output       = "../src/generated/prisma"   // 클라이언트 생성 위치
  moduleFormat = "cjs"                        // NestJS는 CommonJS
}

datasource db {
  provider = "postgresql"
  // Prisma 7: url은 여기에 안 넣음 → prisma.config.ts에서 관리
}
```

```txt
moduleFormat = "cjs":
  NestJS는 CommonJS(require 방식)로 빌드됨
  이 옵션 없으면 → "exports is not defined in ES module scope" 에러

output = "../src/generated/prisma":
  Prisma Client가 생성되는 위치
  기본값은 node_modules/.prisma/client
  커스텀 경로로 지정하면 모노레포에서 위치 명확히 관리 가능

Prisma 7 변경사항 — datasource url:
  이전(Prisma 5/6): datasource db { url = env("DATABASE_URL") }
  이후(Prisma 7):   url을 schema.prisma에서 제거
                    prisma.config.ts에서 dotenv로 읽어서 관리
```
---

# Model — 테이블 정의

```prisma
model User {
  id        Int      @id @default(autoincrement())
  email     String   @unique
  name      String?
  createdAt DateTime @default(now()) @db.Timestamptz(3)
  updatedAt DateTime @updatedAt      @db.Timestamptz(3)
  role      Role     @default(USER)
  posts     Post[]   // 가상 필드, DB 컬럼 아님
}
```

| 어노테이션                       | 의미                                                           |
| --------------------------- | ------------------------------------------------------------ |
| `@id`                       | Primary Key                                                  |
| `@default(autoincrement())` | 숫자 ID 자동 증가                                                  |
| `@default(uuid())`          | UUID 자동 생성                                                   |
| `@unique`                   | UNIQUE 제약                                                    |
| `?`                         | nullable (없으면 `NOT NULL`)                                    |
| `@updatedAt`                | 수정 시 자동 갱신                                                   |
| `@map("컬럼명")`               | 코드 필드명 ↔ DB 컬럼명 매핑                                           |
| `@@map("테이블명")`             | 코드 모델명 ↔ DB 테이블명 매핑                                          |
| `@db.VarChar(n)`            | `VARCHAR(n)` 명시                                              |
| `@db.Uuid`                  | PostgreSQL 네이티브 `uuid` 타입으로 저장 (안 붙이면 `TEXT`)                |
| `@db.Timestamptz(n)`        | PostgreSQL 네이티브 `timestamptz(n)` 타입으로 저장 (안 붙이면 `timestamp`) |

## @map · @@map — 이름 매핑 ⭐️⭐️⭐️⭐️

```prisma
model User {
  id        String   @id @default(cuid())
  email     String   @unique
  createdAt DateTime @default(now()) @map("created_at")
  //                                  ↑ 코드: createdAt / DB: created_at

  @@map("users")
  //    ↑ 코드: User 모델 / DB: users 테이블
}
```

```txt
@map("컬럼명"):
  코드(Prisma)에서 쓰는 필드명 ↔ 실제 DB 컬럼명을 다르게 설정
  코드: createdAt (camelCase — TypeScript 관례)
  DB:  created_at (snake_case — DB 관례)

@@map("테이블명"):
  코드(Prisma)에서 쓰는 모델명 ↔ 실제 DB 테이블명을 다르게 설정
  코드: User (PascalCase — TypeScript 관례)
  DB:  users (snake_case, 소문자 복수 — DB 관례)

언제 쓰는가:
  DB 컨벤션(snake_case)과 코드 컨벤션(camelCase)을 동시에 지키고 싶을 때
  기존 DB 테이블 이름이 정해져 있고 코드에서 다른 이름을 쓰고 싶을 때

쓰지 않는다면:
  Prisma는 기본적으로 model User → "User" 테이블 (PascalCase 그대로)
  필드 createdAt → "createdAt" 컬럼 (camelCase 그대로)
  → DataGrip에서 테이블이 "User", 컬럼이 "createdAt"으로 저장됨
```

```prisma
// 실전 예시 — 전체 모델에 적용
model Post {
  id        String   @id @default(cuid())
  title     String
  content   String?
  authorId  String   @map("author_id")
  createdAt DateTime @default(now()) @map("created_at")
  updatedAt DateTime @updatedAt      @map("updated_at")
  deletedAt DateTime?                @map("deleted_at")

  author    User     @relation(fields: [authorId], references: [id])

  @@map("posts")
}
// DB 테이블: posts
// DB 컬럼:  id, title, content, author_id, created_at, updated_at, deleted_at
// 코드:    post.authorId, post.createdAt (camelCase 그대로 사용)
```


## @db.Uuid — UUID 컬럼을 네이티브 타입으로 ⭐️⭐️

```prisma
model Post {
  id       String @id @default(uuid()) @db.Uuid
  authorId String @db.Uuid   // 이 PK를 참조하는 FK도 같은 타입으로 맞춰야 함
}
```

```txt
@db.Uuid 없이 String @default(uuid())만 쓰면:
  Postgres 컬럼이 TEXT로 생성됨 — UUID 값(36자 문자열)을 텍스트로 저장
  → 저장 공간을 더 쓰고, 인덱스/비교 연산도 텍스트 비교라 더 느림

@db.Uuid를 붙이면:
  Postgres 네이티브 uuid 타입(고정 16바이트)으로 컬럼 생성
  → 저장 공간 절약, 인덱스/비교 더 빠름, DB가 UUID 형식인지도 검증
  → Prisma Client 쪽 TS 타입은 여전히 string — @db.Uuid는 DB 컬럼 타입만 바꿈

⚠️ PK가 @db.Uuid면, 그 PK를 참조하는 FK 컬럼도 반드시 @db.Uuid로 맞춰야 함
   (PK는 네이티브 uuid, FK는 String/TEXT로 두면 타입 불일치로 관계 생성 시 에러)

⚠️ 네이티브 uuid 타입엔 contains/startsWith 같은 문자열 연산자 불가
   UUID로 부분 검색이 필요한 특수한 경우라면 @db.Uuid를 빼고 String으로 두는 게 나음

TEXT vs 네이티브 UUID 비교 · v4/v7 차이 → [[PG_Types]] "UUID" 섹션
```

## uuid() 버전 — v4(기본) vs v7

```prisma
id String @id @default(uuid())    // v4 — 완전 무작위
id String @id @default(uuid(7))   // v7 — 생성 시각 순으로 정렬됨
```

```txt
v7은 값 앞부분에 타임스탬프가 들어가 있어서 생성 순서대로 정렬됨
INSERT가 많은 테이블의 PK라면 v7이 인덱스 단편화를 줄여 더 유리
추측하기 어려운 정도는 v4와 동일
```

## @db.Timestamptz(3) — DateTime을 timezone-aware 타입으로 ⭐️⭐️⭐️⭐️

```prisma
model User {
  createdAt    DateTime  @default(now()) @db.Timestamptz(3)
  updatedAt    DateTime  @updatedAt      @db.Timestamptz(3)
  lastActiveAt DateTime?                 @db.Timestamptz(3)
}
```

```txt
@db.Timestamptz(3) 없이 DateTime만 쓰면:
  PostgreSQL 컬럼이 timestamp(3) — "timezone 없음" 타입으로 생성됨
  세션 TZ 설정에 따라 해석이 달라질 수 있음 (환경마다 값이 다르게 읽힐 위험)

@db.Timestamptz(3)을 붙이면:
  PostgreSQL의 timestamptz(3) — UTC instant로 저장
  어떤 TZ 환경에서 넣어도 항상 같은 순간(UTC)을 가리킴
  Prisma Client 쪽 TS 타입은 여전히 Date — DB 컬럼 타입만 바뀜

precision (3):
  소수점 이하 자릿수 = 밀리초 단위 (.000)
  생략 시 PostgreSQL 기본값 6 (마이크로초) — 실용적으로 3이면 충분

기존 timestamp → timestamptz 마이그레이션이 필요한 경우:
  schema.prisma만 수정해서는 DB 타입이 안 바뀜 → migrate deploy 필수
  -- 마이그레이션 SQL 예시 (기존 데이터가 UTC로 들어갔다고 가정)
  ALTER TABLE "User"
    ALTER COLUMN "createdAt" TYPE TIMESTAMPTZ(3)
    USING "createdAt" AT TIME ZONE 'UTC';

timestamp vs timestamptz 차이의 PostgreSQL 원리 → [[NestJS_PostgreSQL]]
```

## @@unique — 복합 유니크 ⭐️⭐️⭐️⭐️

```prisma
model PostLike {
  userId Int
  postId Int
  @@unique([userId, postId])
  // 한 사람이 같은 게시글에 좋아요 두 번 못 누름
}
```

```txt
@unique  vs  @@unique:
  @unique    컬럼 하나가 테이블 전체에서 유일 (한 필드에 바로 붙임)
  @@unique   컬럼 조합이 유일 (두 개 이상 컬럼을 묶어서 유일성 보장)

  userId가 여러 행에 있어도 되고, postId가 여러 행에 있어도 됨
  둘의 "조합"만 중복이 안 되면 됨
```

## 복합 unique 키 이름 규칙

```typescript
// @@unique([userId, postId]) → Prisma가 자동으로 userId_postId 이름 생성
await prisma.postLike.findUnique({
  where: { userId_postId: { userId: 1, postId: 42 } },
  //       ↑ 필드명_필드명 형태로 묶임
});
```

```txt
복합 unique 이름 커스텀:
  @@unique([userId, postId], name: "user_post_like")
  → where: { user_post_like: { userId, postId } }

자동 생성 이름(필드명_필드명)이 길거나 읽기 어려울 때 사용
```

## 중복 요청 방어 — P2002 에러 처리 ⭐️⭐️⭐️⭐️

```typescript
async createLike(userId: number, postId: number) {
  try {
    return await this.prisma.postLike.create({ data: { userId, postId } });
  } catch (e) {
    if (
      e instanceof Prisma.PrismaClientKnownRequestError &&
      e.code === 'P2002'
    ) {
      throw new ConflictException('이미 좋아요를 눌렀습니다.');
    }
    throw e;
  }
}
```

```txt
P2002가 발생하는 시점:
  DB가 INSERT/UPDATE를 실행하는 순간 unique 제약 위반 감지
  → "이미 있는지 먼저 조회"하는 방어 코드 없이도 DB가 보장
  → "조회 → 없으면 INSERT" 패턴보다 안전
    (조회와 INSERT 사이에 다른 요청이 끼어들 수 있는 race condition 방지)

e.meta.target → P2002면 ['userId', 'postId'] 처럼 어떤 컬럼이 충돌인지 알려줌
더 복잡한 중복 방어 전략(멱등키, 낙관적/비관적 잠금) → [[NestJS_Idempotency]]
```

## @@id — 복합 PK

```prisma
model MovieLike {
  movieId Int
  userId  Int
  @@id([movieId, userId])
}
```

```txt
컬럼 하나로는 안 유일하지만 합치면 유일한 경우에 사용
조회: findUnique({ where: { movieId_userId: { movieId, userId } } })
```

## @@index — 인덱스

```prisma
@@index([area])              // 단일
@@index([area, feeType])     // 복합 — 순서 중요 (앞쪽 컬럼 단독 검색도 효과 있음)
```

```txt
"이 컬럼으로 자주 찾는다"는 힌트
WHERE/ORDER BY에 자주 쓰는 컬럼에 추가 — 조회는 빨라지지만 쓰기는 약간 느려짐
@id / @unique는 자동으로 인덱스 생성됨, 그 외 컬럼은 수동 추가
```

## @@index vs @@unique

|구분|`@@index`|`@@unique`|
|---|---|---|
|목적|조회 속도만 빠르게|중복 자체를 DB가 막음 (+ 조회도 빠름)|
|중복 행|허용|불허 (P2002 에러)|
|판단 기준|"이 컬럼으로 자주 검색/정렬하는가"|"이 조합이 두 번 있으면 안 되는가"|

```txt
중복이 "버그"면 @@unique, 중복이 "정상"인데 빠르게 찾고 싶으면 @@index
⚠️ @@unique는 자동으로 인덱스 역할도 겸함 — 같은 컬럼 조합에 @@index를 따로 또 만들 필요 없음
```

---

# Scalar Types

> PostgreSQL 타입 상세(timestamp · JSONB · UUID · ARRAY · ENUM 원리) → [[PG_Types]]

|Prisma|PostgreSQL|설명|
|---|---|---|
|`String`|TEXT|문자열|
|`Int` / `BigInt`|INTEGER / BIGINT|정수|
|`Float` / `Decimal`|REAL / NUMERIC|부동소수 / 정밀 소수 (금액)|
|`Boolean`|BOOLEAN|true/false|
|`DateTime`|`TIMESTAMP` / `TIMESTAMPTZ`|날짜+시간 — `@db.Timestamptz(3)` 명시 권장|
|`Json`|JSONB|JSON|

```prisma
enum Role { USER  ADMIN  MODERATOR }
```

```txt
⚠️ Enum도 Prisma Client 생성 경로에서 import
   다른 곳(예: 예전 TypeORM entity 파일)에서 가져오면 타입 불일치
```

---

# Relations — 관계

```prisma
// One to Many
model User  { posts Post[] }
model Post  { authorId Int; author User @relation(fields: [authorId], references: [id]) }

// One to One — FK에 @unique 추가
model Profile { userId Int @unique; user User @relation(fields: [userId], references: [id]) }

// Many to Many — Prisma가 중간 테이블 자동 생성
model Post { tags Tag[] }
model Tag  { posts Post[] }
```

```txt
fields:     내가 들고 있는 FK 컬럼
references: 상대 테이블 PK
```

|onDelete|동작|언제 쓰는가|
|---|---|---|
|`Cascade`|부모 삭제 → 자식도 삭제|게시글 삭제 시 댓글도 함께 삭제|
|`SetNull`|부모 삭제 → FK를 NULL로|작성자 탈퇴 시 게시글을 익명으로 유지|
|`Restrict`|참조 중이면 삭제 불가 (에러)|자식이 있는 부모는 삭제 못 하게 막음|
|`NoAction`|참조 중이면 삭제 불가 (지연 체크)|`Restrict`과 비슷, PostgreSQL에서는 사실상 동일|
|`SetDefault`|부모 삭제 → FK를 기본값으로|기본값이 설정된 경우에만 유효|

```prisma
// Cascade — 부모 삭제 시 자식도 삭제
model Post {
  comments Comment[]
}
model Comment {
  post   Post   @relation(fields: [postId], references: [id], onDelete: Cascade)
  postId String
}
// Post 삭제 → 해당 Post의 모든 Comment 자동 삭제

// Restrict — 자식이 있으면 부모 삭제 불가
model Comment {
  parent   Comment?  @relation("CommentReplies", fields: [parentId], references: [id], onDelete: Restrict)
  parentId String?
  replies  Comment[] @relation("CommentReplies")
}
// 답글이 있는 댓글은 삭제 불가 → 먼저 답글을 삭제해야 함

// SetNull — 부모 삭제 시 FK를 NULL로
model Post {
  author   User?   @relation(fields: [authorId], references: [id], onDelete: SetNull)
  authorId String? // NULL 허용이어야 함
}
// 유저 탈퇴 시 Post.authorId = NULL (게시글은 남김)
```

```txt
기본값:
  onDelete를 명시 안 하면 → Restrict (참조 중이면 삭제 불가)

Restrict vs Cascade 선택 기준:
  부모 삭제 시 자식도 함께 사라져야 함  → Cascade
  자식이 있으면 부모를 못 지우게 막아야 함 → Restrict
  자식은 남기되 연결만 끊음             → SetNull

self-relation에서 Restrict (댓글-답글):
  답글이 달린 댓글을 삭제하려면 → 먼저 답글을 모두 삭제해야 함
  → 소프트 삭제(deletedAt)를 함께 고려하는 경우가 많음
```

## 관계 이름 — `@relation("이름")` ⭐️⭐️⭐️

```txt
필요 조건: 같은 모델을 한 모델에서 2번 이상 참조할 때만

안 붙이면 → Ambiguous relation detected 에러
같은 패턴: self-relation, 한 모델 안 관계 2개 (author/pinnedBy 둘 다 User 참조)
```

```prisma
// Friendship이 User를 두 번 참조
model Friendship {
  requester User @relation("FriendshipRequester", fields: [requesterId], references: [id])
  addressee User @relation("FriendshipAddressee", fields: [addresseeId], references: [id])
}
model User {
  sentRequests     Friendship[] @relation("FriendshipRequester")
  receivedRequests Friendship[] @relation("FriendshipAddressee")
}
```

```txt
"문자열" — DB와 무관, 양쪽을 짝짓는 Prisma 전용 키 — 양쪽 동일해야 함
```

---

# findUnique vs findFirst vs findMany ⭐️

```txt
조건이 @id / @unique 컬럼 딱 하나  → findUnique
조건 자유롭고 결과 1건             → findFirst (NOT 포함 가능)
여러 건                            → findMany
```

| |조건|결과|
|---|---|---|
|`findUnique`|반드시 unique 컬럼만|단건 또는 `null`|
|`findFirst`|자유 (`NOT` 포함)|첫 행 또는 `null`|
|`findMany`|자유|배열|

```typescript
this.prisma.user.findUnique({ where: { id } });
this.prisma.user.findFirst({ where: { name, NOT: { id } } });  // 자기 자신 제외 중복 체크
this.prisma.post.findMany({ where: { isVisible: true } });
```

## OrThrow 변형 — findUniqueOrThrow · findFirstOrThrow ⭐️⭐️⭐️⭐️

```typescript
// findUnique   → User | null  (null 체크 필요)
// findUniqueOrThrow → User   (없으면 throw, null 체크 불필요)

const me = await this.prisma.user.findUniqueOrThrow({
  where:  { id: userId },
  select: { role: true },
});
// me.role 바로 사용 가능 — null 가능성 없음
```

```txt
OrThrow가 없는 버전과의 차이:

  findUnique       → 결과 타입: User | null
  findUniqueOrThrow → 결과 타입: User
    없으면 Prisma.PrismaClientKnownRequestError (code: P2025) 던짐

  null 체크 없이 바로 쓸 수 있다는 게 핵심
    findUnique:       const user = await ...; if (!user) throw new NotFoundException();
    findUniqueOrThrow: const user = await ...;  // 이게 끝, null 체크 불필요
```

```txt
언제 OrThrow를 쓰는가 — "없으면 버그"인 상황:

  ✅ 쓰기 좋은 경우:
    JWT 토큰에서 온 userId로 유저 조회
    → 토큰이 유효하다면 유저가 반드시 존재해야 함
    → 없으면 데이터 불일치 (서버 오류) → throw가 맞음

    이미 존재를 검증한 뒤의 상세 조회
    → 이전 단계에서 존재 확인이 끝났으므로 없으면 서버 버그

    외래키(FK)가 보장하는 관계 조회
    → DB 제약이 있으므로 정상 흐름에서 없을 수 없음

  ❌ 쓰면 안 되는 경우:
    사용자 입력 ID로 조회 (없을 수 있는 정상 케이스)
    → 없으면 NotFoundException을 직접 던지는 게 더 명확한 에러 메시지 가능

    목록에서 특정 조건으로 찾기 (없을 수 있음이 정상)
    → findFirst + null 체크
```

```typescript
// ✅ OrThrow가 적합한 패턴 — userId는 JWT에서 왔으므로 반드시 존재
private async resolveStatus(userId: string, otherRole: UserRole): Promise<DmStatus> {
  const me = await this.prisma.user.findUniqueOrThrow({
    where:  { id: userId },
    select: { role: true },
  });
  if (me.role === UserRole.admin || otherRole === UserRole.admin) {
    return DmStatus.open;
  }
  // ...
}

// ❌ 사용자 입력 ID → 없으면 명확한 메시지와 함께 404
async getPost(postId: string) {
  const post = await this.prisma.post.findUnique({ where: { id: postId } });
  if (!post) throw new NotFoundException('게시글을 찾을 수 없습니다.');
  return post;
}
```

```txt
OrThrow가 던지는 에러:
  Prisma.PrismaClientKnownRequestError (code: P2025)
  NestJS 기본 동작: 에러 필터가 잡지 못하면 500 InternalServerError로 올라감

  → 사용자에게 노출되면 안 되는 내부 오류 (서버 버그 신호)로 다루는 게 적절
  → 사용자 입력 ID 조회처럼 "없으면 404"가 명확히 필요한 경우는 findUnique + 수동 throw
```

---

# CRUD 기본

```typescript
// CREATE
await this.prisma.user.create({ data: { email, passwordHash } });

// READ
const user = await this.prisma.user.findUnique({ where: { id } });
const list = await this.prisma.post.findMany({
  where: { isVisible: true },
  orderBy: { createdAt: 'desc' },
  take: 10,
});

// UPDATE
await this.prisma.user.update({ where: { id }, data: { name } });

// UPSERT — 있으면 update, 없으면 create
await this.prisma.user.upsert({
  where:  { email },
  create: { email, passwordHash },
  update: { passwordHash },
});

// DELETE
await this.prisma.user.delete({ where: { id } });
await this.prisma.post.deleteMany({ where: { isVisible: false } });
```

---

# where — 조건 연산자

```typescript
where: {
  views:     { gt: 100, gte: 100, lt: 100, lte: 100 },
  email:     { contains: 'gmail' },        // LIKE '%gmail%'
  name:      { startsWith: 'A' },
  role:      { in: [Role.ADMIN, Role.USER] },
  deletedAt: null,                         // IS NULL
}
```

|연산자|SQL|
|---|---|
|`gt`/`gte`/`lt`/`lte`|`>` `>=` `<` `<=`|
|`contains`/`startsWith`/`endsWith`|`LIKE`|
|`in`/`notIn`|`IN (...)`|

```typescript
// mode: 대소문자 무시 (PostgreSQL 한정)
{ title: { contains: 'art', mode: 'insensitive' } }

// AND (기본) / OR / NOT
where: { isVisible: true, genre: 'drama' }      // AND
where: { OR: [{ name: { contains: '김' } }, { email: { contains: 'gmail' } }] }
where: { name, dob, NOT: { id } }               // findFirst에서만 가능
```

```txt
관계 기반 필터 (some / every / none / is / isNot) → [[NestJS_Prisma_Patterns]]
```

---

# select / omit / include ⭐️⭐️

|키워드|용도|
|---|---|
|`select`|가져올 필드만 지정 (화이트리스트)|
|`omit`|뺄 필드만 지정 (나머지 전부)|
|`include`|관계(연결된 테이블) 함께 조회|

```typescript
this.prisma.user.findMany({ select: { id: true, email: true } });
this.prisma.user.findMany({ omit: { passwordHash: true } }); // passwordHash만 빼고 전부

this.prisma.user.findUnique({ where: { id }, include: { posts: true } });
```

## 언제 select, 언제 omit, 언제 아무것도 안 쓰나 ⭐️⭐️⭐️⭐️

```txt
아무 옵션도 안 쓰면: scalar 필드는 전부, 관계는 안 가져옴 (기본 동작)
  → 민감한 필드가 없는 모델이고 다 써도 상관없으면 가장 간단함
```

|선택|동작 방식|언제 더 안전/편한가|
|---|---|---|
|`select`|화이트리스트 — 명시한 필드만 결과에 포함|민감 필드(password 등)가 있을 때 — 새 필드가 추가돼도 select에 직접 추가하지 않는 한 자동으로 노출되지 않음|
|`omit`|블랙리스트 — 명시한 필드만 빼고 나머지 전부|필드가 많고 뺄 게 1~2개뿐일 때 — 나중에 새로 추가된 민감 필드를 깜빡하고 omit에 안 넣으면 그대로 노출됨|

```txt
판단 기준: "이 모델에 민감한 필드(password, refreshToken 등)가 있는가?"
  있다면 → select (화이트리스트)가 더 안전
  없다면 → omit이 더 짧고 편함, 또는 그냥 다 가져와도 무방

이 발상은 [[NestJS_DTO]] whitelist: true(ValidationPipe)와 같은 원리
명시한 것만 통과, 나머지는 자동으로 막힘
```

## include에 뭘 넣는가 — 판단 기준 ⭐️⭐️⭐️⭐️

```txt
판단 기준 한 문장: "이 관계 테이블의 행이 응답에 실제로 필요한가?"

필요하다  → include에 넣는다
필요 없다 → 넣지 않는다 (where 안의 관계 필터와 다른 개념)
```

```typescript
// 채팅방 목록 — 방장 프로필 · 내 lastReadAt · 마지막 메시지 시각이 필요한 경우
const rooms = await this.prisma.room.findMany({
  where: {
    status: 'active',
    members: { some: { userId } },  // 필터: "내가 멤버인 방만" — members 데이터 안 가져옴
  },
  include: {
    owner: {                         // "방장 닉네임/이미지를 화면에 보여야 한다"
      select: { id: true, nickname: true, image: true },
    },
    members: {                       // "내 lastReadAt을 읽어야 한다"
      where:  { userId },            //   나에 해당하는 행만
      select: { lastReadAt: true },
      take:   1,
    },
    messages: {                      // "마지막 메시지 시각을 보여야 한다"
      where:   { deletedAt: null },
      orderBy: { createdAt: 'desc' },
      take:    1,
      select:  { createdAt: true },
    },
  },
});
```

```txt
⭐️ 같은 관계(members)가 where와 include 양쪽에 동시에 나오는 이유:

  where.members.some  →  "내가 멤버인 방만 가져와" (Room 필터링)
    members 데이터를 결과에 포함하지 않음 — 방의 존재 여부만 확인

  include.members     →  "그 방에서 내 멤버 행도 같이 줘" (데이터 로딩)
    lastReadAt이 응답에 필요하기 때문에 실제 데이터를 가져옴

  같은 관계를 쓰지만 역할이 완전히 다름:
    최상위 where 안 → 어떤 Room을 반환할지 결정 (존재 여부만 확인)
    include 안      → 반환하는 Room에서 관계 데이터를 함께 로드 (실제 데이터)
```

```txt
include 넣을지 말지 체크 순서:
  ① 응답 객체에 이 관계의 데이터가 직접 들어가야 하는가?
       예: 채팅방 카드에 방장 닉네임을 표시 → owner include
       예: 내가 어디까지 읽었는지 표시 → members include (내 것만 where로 제한)
       예: 마지막 메시지 시각 표시 → messages include (take: 1)
  ② 개수만 필요한가? → _count (실제 행 불필요, 아래 참고)
  ③ 필터링에만 쓰는가? → where 안에만 (include 불필요)
```

## include 안에서 필터·정렬·select 같이 쓰기 ⭐️⭐️⭐️

```typescript
this.prisma.user.findUnique({
  where: { id },
  include: {
    posts: {
      where:   { isVisible: true },
      orderBy: { createdAt: 'desc' },
      take:    5,
      select:  { id: true, title: true },
    },
  },
});
```

```txt
include 안의 관계 필드도 findMany와 같은 옵션(where/orderBy/take/skip/select)을 받음
→ "관계 = 또 하나의 작은 조회"로 생각하면 됨
```

## _count — 개수만 필요할 때

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

```txt
⚠️ include를 깊게/넓게 쓸수록 한 번에 가져오는 데이터가 커짐 — 화면에 필요한 만큼만
⚠️ select + include 동시 사용 불가
```

## select/include 객체를 재사용 가능한 상수로 빼기 ⭐️⭐️⭐️⭐️

```typescript
const userSelect = {
  id: true,
  email: true,
  nickname: true,
  role: true,
  createdAt: true,
} as const;
```

```txt
변수로 빼는 이유: 여러 쿼리에서 같은 필드 목록을 재사용, 한 곳만 수정하면 모든 쿼리에 반영

as const가 왜 따로 필요한가:
  const withoutConst = { id: true, email: true };
  // TS 추론: { id: boolean; email: boolean } — 'true'가 boolean으로 넓혀짐(widening)

  const withConst = { id: true, email: true } as const;
  // TS 추론: { readonly id: true; readonly email: true } — 'true' 리터럴 타입 유지

  Prisma의 select 타입은 각 필드 값이 정확히 "true라는 리터럴"이어야 반환 타입을 좁힐 수 있음
  as const 없이 넘기면 boolean으로 넓혀져서 Prisma가 반환 타입을 정확히 추론 못 함
  → as const는 "재사용 때문에"가 아니라 "Prisma가 정확한 반환 타입을 추론하게 하려고" 붙이는 것
```

## 상황별 select/include 모양 3가지 ⭐️⭐️⭐️⭐️

```typescript
// ① 단순 select — scalar 필드만 고를 때
const userSelect = {
  id: true, email: true, nickname: true, role: true, createdAt: true,
} as const;

// ② include 안에 select — 관계를 가져오되 일부 필드만
const savedCardInclude = {
  item: {
    select: { id: true, title: true, createdAt: true },
  },
} as const;

// ③ include + _count — 개수만, 실제 데이터는 안 가져올 때
const userCountInclude = {
  _count: {
    select: { posts: true, comments: true },
  },
} as const;
```

## _count는 select 안에서도 쓸 수 있음 — 재사용 트릭

```typescript
this.prisma.user.findMany({
  select: {
    ...userSelect,              // ① scalar 필드
    _count: userCountInclude._count,  // ③ _count 부분만 꺼내서 재사용
  },
});
```

```txt
_count는 select와 include 둘 다에서 쓸 수 있음
include용으로 만들어둔 객체의 _count 부분만 따로 꺼내서 select에 끼워넣는 것도 가능
→ 한 곳에서만 정의해두고 양쪽에서 재사용
```

---

# orderBy / take / skip

```typescript
this.prisma.post.findMany({
  orderBy: [{ views: 'desc' }, { createdAt: 'desc' }],  // 동률이면 다음 기준
  take: 10,                  // LIMIT
  skip: (page - 1) * 10,    // OFFSET
});
```

> 커서 기반 페이지네이션 → [[NestJS_Pagination]]

---

# count / aggregate / groupBy

```typescript
const count = await this.prisma.user.count({ where: { role: Role.USER } });

const stats = await this.prisma.post.aggregate({
  _count: { id: true },
  _avg:   { views: true },
  _max:   { views: true },
});

const rows = await this.prisma.post.groupBy({
  by: ['category'],
  _count: { _all: true },
});
```

| |역할|
|---|---|
|`aggregate`|전체(또는 where 조건)를 한 번에 집계|
|`groupBy`|컬럼 값 기준으로 그룹별 집계 (SQL `GROUP BY`)|

---

# Prisma namespace 타입

```txt
prisma generate를 실행하면, schema.prisma에 정의한 모델마다 자동으로 타입 생성:
  {모델명}WhereInput         where 조건에 쓸 수 있는 타입
  {모델명}WhereUniqueInput   unique 조건 (findUnique의 where)
  {모델명}CreateInput        create data 타입
  {모델명}UpdateInput        update data 타입 (전 필드 optional)
  {모델명}OrderByWithRelationInput  orderBy 타입
  {모델명}Select             select 옵션 타입
  {모델명}Include            include 옵션 타입
```

```typescript
import { Prisma } from '../generated/prisma/client';

Prisma.RoomWhereInput        // where: { ... } 에 들어가는 타입
Prisma.RoomCreateInput       // create: { data: ... } 타입
Prisma.RoomUpdateInput       // update: { data: ... } 타입 (전 필드 optional)
Prisma.RoomWhereUniqueInput  // findUnique의 where (unique 컬럼만)
```

## WhereInput — 조건부 where 조립 ⭐️⭐️⭐️⭐️

```txt
서비스에서 where 조건을 단계별로 조립할 때 타입 안전하게 쓰는 방법
```

```typescript
// 조건에 따라 where를 쌓아가는 패턴
async findRooms(q?: string, status?: RoomStatus) {
  const where: Prisma.RoomWhereInput = {};  // 처음엔 비어있음

  if (status) {
    where.status = status;           // 조건 추가
  }
  if (q) {
    where.OR = [                     // OR 조건 추가
      { name:  { contains: q, mode: 'insensitive' } },
      { owner: { nickname: { contains: q, mode: 'insensitive' } } },
    ];
  }

  return this.prisma.room.findMany({ where });
}
```

```txt
Prisma.RoomWhereInput vs let where: object = {}:
  object  = 어떤 객체든 가능, 타입 체크 없음
            where.typo = 'abc' 처럼 잘못된 필드를 써도 에러 안 남

  Prisma.RoomWhereInput = Room의 where 조건에 맞는 필드만 허용
            where.typo = 'abc' → TS 에러 즉시 잡힘
            where.OR, where.name, where.status 만 쓸 수 있음

  → 실제 코드에서는 Prisma.RoomWhereInput 을 쓰는 것이 더 안전
    조건부 where 조립 패턴 → [[NestJS_Prisma_Patterns]]
```

---

# 중첩 where — 관계 테이블 필드로 필터링 ⭐️⭐️⭐️⭐️

```txt
단순 where: 이 테이블의 컬럼 조건
중첩 where: 관계로 연결된 다른 테이블의 컬럼 조건
```

```typescript
// 단순: Room의 컬럼으로 필터
where: { status: 'active' }

// 중첩: Room에 연결된 owner(User)의 컬럼으로 필터
where: { owner: { nickname: { contains: q } } }
// "nickname에 q가 포함된 owner를 가진 방"
```

```txt
{ owner: { nickname: { contains: q } } } 읽는 법:

  owner          →  Room과 @relation으로 연결된 User
  nickname       →  그 User의 nickname 컬럼
  contains: q    →  q가 포함된 것만

  SQL로 표현하면:
    JOIN users ON rooms.owner_id = users.id
    WHERE users.nickname LIKE '%q%'

  Prisma가 중첩 객체 구조를 보고 자동으로 JOIN을 생성
  → 직접 JOIN을 작성할 필요 없음
```

## 검색 — 여러 필드를 OR로 ⭐️⭐️⭐️⭐️

```typescript
if (q) {
  where.OR = [
    { name:  { contains: q, mode: 'insensitive' } },  // 방 이름에 q 포함
    { owner: { nickname: { contains: q, mode: 'insensitive' } } },  // 방장 닉네임에 q 포함
  ];
}
```

```txt
mode: 'insensitive':
  대소문자를 무시하고 검색 (PostgreSQL 전용)
  'Hello'로 검색하면 'hello', 'HELLO', 'Hello' 전부 매칭

  mode: 'default'  → 대소문자 구분 (기본값)
  mode: 'insensitive' → 대소문자 무시

OR 배열:
  배열 안의 조건 중 하나라도 일치하면 해당 행 포함
  [ { name: ... }, { owner: { nickname: ... } } ]
  = "방 이름에 q가 있거나, 방장 닉네임에 q가 있는 방"
```

## 중첩 깊이 ⭐️⭐️⭐️

```typescript
// 2단계 중첩
where: {
  owner: {         // Room → User
    nickname: { contains: q }
  }
}

// 3단계 중첩 — Room → Member → User
where: {
  members: {
    some: {
      user: {      // Member → User
        status: 'active'
      }
    }
  }
}
```

```txt
중첩 구조가 schema.prisma의 @relation 관계를 따름
relation이 정의된 필드만 중첩해서 조건을 걸 수 있음

일반 where(scalar 필드)와 관계 where를 같이 쓸 수 있음:
  where: {
    status: 'active',           // scalar 조건
    owner: {                    // 관계 조건 (중첩)
      nickname: { contains: q }
    },
    members: { some: { userId } } // 관계 필터
  }
```

---

# Prisma namespace 타입 — 자주 쓰는 타입 전체

```typescript
// 타입 선언 예시
const where:   Prisma.RoomWhereInput         = {};  // where 조건
const data:    Prisma.RoomCreateInput        = {};  // create 데이터
const update:  Prisma.RoomUpdateInput        = {};  // update 데이터 (전 optional)
const orderBy: Prisma.RoomOrderByWithRelationInput = {};  // 정렬

// 중첩 타입
const ownerWhere: Prisma.UserWhereInput = { nickname: { contains: q } };
```

```txt
모델명 자리에 schema.prisma에 정의한 model 이름을 그대로 씀:
  model Room { ... } → Prisma.RoomWhereInput
  model User { ... } → Prisma.UserWhereInput
  model Post { ... } → Prisma.PostCreateInput
```

## Json 필드 — JsonValue vs InputJsonValue ⭐️⭐️⭐️⭐️

> PostgreSQL JSONB 타입 원리(JSON vs JSONB · GIN 인덱스 · 연산자) → [[PG_Types]] "JSONB" 섹션

```prisma
model Post {
  metadata Json
}
```

```txt
schema에 Json으로 선언한 필드는 읽을 때와 쓸 때의 타입이 다름:
  Prisma.JsonValue       읽을 때 (조회 결과)
  Prisma.InputJsonValue  쓸 때 (create/update의 data)

둘 다 string | number | boolean | null | JsonObject | JsonArray 형태의 재귀적 유니온 타입
TS가 자동으로 좁혀주지 않아서 중첩된 값을 다루려면 단언이나 타입가드가 필요한 경우가 많음
```

```typescript
metadata: dto.metadata as Prisma.InputJsonValue,
```

```txt
DTO의 metadata 필드가 구체적인 인터페이스나 Record<string, any>로 선언돼 있어도
그 타입을 Prisma.InputJsonValue에 그대로 넘기면 TS가 거부하는 경우가 흔함

가장 자주 걸리는 원인 — 옵셔널 속성(?):
  interface Metadata { theme?: string }
  theme?: string는 TS 내부적으로 theme: string | undefined로 다뤄짐
  undefined는 유효한 JSON 값이 아님 (JSON에는 undefined가 없음, null만 있음)
  → 옵셔널 속성이 있는 객체 타입은 구조적으로 InputJsonValue와 안 맞다고 TS가 판단함
  → 실제 런타임 값은 멀쩡한 JSON인데 타입 레벨에서 막혀서 as로 우회하게 됨
```

### 더 나은 대안 — Json 필드에 직접 타입 붙이기 ⭐️⭐️⭐️

```txt
prisma-json-types-generator 설치 + schema.prisma에 generator json 블록 추가 → [[NestJS_Migration]] 참고
```

```prisma
metadata Json /// [MetadataType]
// ⚠️ /// 주석은 필드와 "같은 줄, 뒤쪽"에 붙여야 함 (위 줄 아님)
```

```typescript
// types.ts
declare global {
  namespace PrismaJson {
    type MetadataType = { theme?: string; layout?: string };
  }
}
```

```txt
이렇게 해두면 Prisma Client 타입 자체가 Prisma.JsonValue 대신 MetadataType이 되어서
조회/쓰기 양쪽에서 더 정확한 자동완성, as 단언도 줄어듦
→ as로 한두 곳만 우회하는 정도면 충분, Json 필드가 많아지면 검토할 만함
```

## JSON의 null — Prisma.JsonNull vs Prisma.DbNull

```typescript
await this.prisma.post.update({
  where: { id },
  data: { metadata: Prisma.JsonNull },  // 컬럼에 "JSON 값으로서의 null" 저장
});
```

```txt
Json 컬럼에서는 두 가지 "없음"이 구분됨:
  Prisma.DbNull   SQL의 NULL — 컬럼 자체에 값이 없음
  Prisma.JsonNull JSON의 null — 컬럼에 값은 있는데 그 값이 JSON null임

그냥 null을 넘기면 Prisma가 어느 의도인지 헷갈릴 수 있어서
Json 필드를 명시적으로 비워달라고 할 때는 이 전용 상수를 씀
```

---
# $queryRaw · $executeRaw — Raw SQL ⭐️⭐️⭐️

```typescript
// $queryRaw — 결과값이 필요한 쿼리 (SELECT)
const result = await this.prisma.$queryRaw`SELECT 1`;
//                                        ↑ 템플릿 리터럴 문법

// $executeRaw — 결과가 없는 쿼리 (UPDATE, DELETE 등)
const count = await this.prisma.$executeRaw`UPDATE "users" SET active = true`;
// → 영향받은 행 수(number)를 반환
```

```txt
$queryRaw`SQL`:
  Prisma 모델 없이 SQL을 직접 실행
  결과를 배열로 반환
  SELECT 등 데이터를 읽을 때

$executeRaw`SQL`:
  결과값 없이 실행
  영향받은 행 수(number) 반환
  UPDATE, DELETE, INSERT 직접 실행할 때

템플릿 리터럴 (백틱) 문법 사용 이유:
  SQL Injection 방지 — 변수를 ${}에 넣으면 자동으로 파라미터 바인딩
  await this.prisma.$queryRaw`SELECT * FROM users WHERE id = ${userId}`
  → userId가 문자열로 삽입되는 게 아니라 파라미터로 전달 → 안전
```

## 헬스체크에서 SELECT 1 ⭐️⭐️⭐️

```typescript
@Controller('health')
export class HealthController {
  constructor(private readonly prisma: PrismaService) {}

  @Get()
  async healthCheck() {
    await this.prisma.$queryRaw`SELECT 1`;
    // 에러 없으면 DB 연결 정상
    return { ok: true };
  }
}
```

```txt
SELECT 1:
  가장 단순한 SQL 쿼리 — "숫자 1을 반환해라"
  DB가 연결돼 있고 쿼리를 처리할 수 있으면 에러 없이 반환
  실제 데이터를 읽지 않아서 부하가 거의 없음
  → "DB가 살아있는지" 확인하는 표준 패턴

에러가 발생하면:
  DB 연결 실패, DB 서버 다운 등
  → throw → 500 에러 → 헬스체크 실패로 감지
```

---
# 에러 처리 ⭐️⭐️⭐️⭐️

## 방법 1 — try/catch로 개별 처리

```typescript
try {
  await this.prisma.user.create({ data: { email } });
} catch (error) {
  if (
    error instanceof Prisma.PrismaClientKnownRequestError &&
    error.code === 'P2002'
  ) {
    throw new ConflictException('이미 사용 중인 이메일입니다.');
  }
  throw error;
}
```

```txt
각 서비스에서 필요한 곳마다 직접 처리
세밀한 제어 가능 (코드마다 다른 메시지 등)
반복이 많아질 수 있음
```

## 방법 2 — PrismaExceptionFilter (전역 처리) ⭐️⭐️⭐️⭐️

```typescript
// src/common/filters/prisma-exception.filter.ts
import {
  ArgumentsHost,
  Catch,
  ExceptionFilter,
} from '@nestjs/common';
import { Prisma } from '../../generated/prisma/client';

@Catch(Prisma.PrismaClientKnownRequestError)
export class PrismaExceptionFilter implements ExceptionFilter {
  catch(
    exception: Prisma.PrismaClientKnownRequestError,
    host: ArgumentsHost,
  ) {
    const res = host.switchToHttp().getResponse();

    if (exception.code === 'P2002') {
      return res.status(409).json({
        statusCode: 409,
        message: '이미 사용 중입니다.',
      });
    }
    
     if (exception.code === 'P2025') {
      return res.status(404).json({
        statusCode: 404,
        message: '리소스를 찾을 수 없습니다',
      });
    }

    // 처리 안 된 Prisma 에러는 500
    return res.status(500).json({
      statusCode: 500,
      message: 'Database error',
    });
  }
}
```

```typescript
// main.ts에 전역 등록
app.useGlobalFilters(new PrismaExceptionFilter());
```

```txt
@Catch(Prisma.PrismaClientKnownRequestError):
  PrismaClientKnownRequestError가 throw되면 이 Filter가 가로챔
  try/catch를 서비스마다 작성하지 않아도 됨

ExceptionFilter:
  NestJS가 제공하는 예외 처리 인터페이스
  catch(exception, host) 하나만 구현

ArgumentsHost:
  현재 실행 컨텍스트에 대한 정보
  host.switchToHttp().getResponse() → Express Response 객체 직접 접근
  → res.status(409).json(...)으로 응답 직접 제어

방법 1 vs 방법 2:
  try/catch (방법 1) → 에러 메시지를 상황마다 다르게 할 때
  ExceptionFilter (방법 2) → P2002 같은 공통 에러를 한 곳에서 처리
  → 보통 두 가지를 혼용
  ExceptionFilter로 공통 처리 + 특수한 경우만 try/catch
```

## Prisma 에러 코드

|코드|의미|보통 던지는 응답|
|---|---|---|
|`P2002`|Unique 제약 위반 (중복)|`409 Conflict`|
|`P2003`|FK 제약 위반|`400 Bad Request`|
|`P2025`|대상 레코드 없음|`404 Not Found`|


```typescript
// exception.code와 meta 활용
if (exception.code === 'P2002') {
  // exception.meta.target → ['email'] 처럼 어떤 컬럼이 충돌인지
  const field = (exception.meta?.target as string[])?.[0];
  return res.status(409).json({
    statusCode: 409,
    message: `${field ?? '값'}이(가) 이미 사용 중입니다.`,
  });
}
```
---

# TypeORM ↔ Prisma 메서드 대조

|TypeORM|Prisma|용도|
|---|---|---|
|`findOne({ where })`|`findUnique({ where })`|단건 (unique)|
|`find({ where })`|`findMany({ where })`|목록|
|`save(entity)`|`create({ data })`|생성|
|`update({ id }, dto)`|`update({ where, data })`|수정|
|`save()` (있으면 update)|`upsert({ where, create, update })`|upsert|
|`delete({ id })`|`delete({ where })`|삭제|
|`find({ relations: {...} })`|`findMany({ include: {...} })`|관계 조회|

---

# 자주 만나는 에러

| 증상                                          | 원인                                              | 해결                                         |
| ------------------------------------------- | ----------------------------------------------- | ------------------------------------------ |
| 복합 unique 에러                                | `findUnique({ where: { name, dob } })` 처럼 따로 넘김 | `where: { name_dob: { name, dob } }` 로 묶어서 |
| TypeORM 문법 사용                               | `find({ relations: {...} })`                    | `findMany({ include: {...} })`             |
| 타입 자동완성 안 바뀜                                | `migrate dev`/`generate` 누락 또는 서버 재시작 안 함       | [[NestJS_Migration]] 체크리스트 참고              |
| `Ambiguous relation detected`               | 같은 모델을 두 번 참조하는데 관계 이름 없음                       | 충돌하는 두 필드에 `@relation("이름")`, 양쪽 동일하게      |
| `exports is not defined in ES module scope` | `moduleFormat = "cjs"` 누락                       | schema.prisma generator 블록에 추가             |