---
aliases:
  - NestJS Prisma
  - Prisma
  - Prisma ORM
tags:
  - NestJS
related:
  - "[[00_NestJS_Ecosystem_HomePage]]"
  - "[[NestJS_Module]]"
  - "[[NestJS_Pagination_r]]"
  - "[[NestJS_Prisma_Monorepo]]"
  - "[[JS_Operators]]"
  - "[[NestJS_DTO]]"
  - "[[PG_Types]]"
  - "[[NestJS_Migration]]"
  - "[[NestJS_Idempotency]]"
---
# NestJS_Prisma — Prisma ORM

> [!info] 
> Prisma는 타입 안전한 쿼리 + 마이그레이션 ORM
> 반복 루프는 schema 수정 → migrate dev → Client 사용이고, 나머지(Model 문법, Relations, CRUD, where...)는 이 루프 안의 디테일이다.

---

## 흐름도

```mermaid-beautiful
flowchart TB
    S([schema.prisma]) --> M[migrate dev]
    M --> DB[(DB)]
    M --> G[generate]

    G --> P[PrismaClient]
    G --> T["Prisma 타입<br/>WhereInput · CreateInput · UpdateInput"]

    P --> Q["this.prisma.모델.메서드()"]
    T -.->|where / data| Q
    Q --> DB

    Q --> R{"읽기 — 무엇이 필요?"}
    R -->|"지금 모델 컬럼만<br/>민감 필드 제외"| SEL["select<br/>id · email · nickname"]
    R -->|"연결된 테이블 행<br/>목록·상세"| INC["include<br/>user → posts[]"]
    R -->|"연결된 것 개수만<br/>행은 안 가져옴"| CNT["_count<br/>posts: 12"]

    S -->|수정| M
```

### 읽기 shape — 언제 뭘 쓰나 ⭐️⭐️⭐️

```txt
한 가지만 골라서 생각하면 됨 — "지금 조회하는 모델" vs "relations로 연결된 다른 테이블"

① 지금 모델(User)의 컬럼만 필요 → select
   · passwordHash 같은 건 빼고 싶을 때 (화이트리스트)
   · 아무 옵션 없으면 scalar는 전부, 관계는 안 옴

② 연결된 테이블의 실제 행이 필요 → include
   · User + 그 User가 쓴 Recommendation 목록
   · include 안에서 where / orderBy / select도 가능

③ 연결된 것이 몇 개인지만 필요 → _count
   · "추천 12개"처럼 숫자만 — 행을 통째로 안 가져와서 가벼움
   · include 안에 넣거나, select 안에 넣어도 됨

⚠️ 최상위에서 select + include 동시 ❌
   select + _count ✅ — 관리자 유저 목록이 이 패턴
```

---

# TypeORM vs Prisma

| |TypeORM|Prisma|
|---|---|---|
|정의 방식|Entity 클래스|`schema.prisma` 파일 하나|
|흐름|코드 먼저 → DB 반영|스키마 먼저 → Client 자동 생성|
|타입 안전성|보통|강력(자동완성)|

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
② 한 줄이 하는 일: 변경분만 SQL 마이그레이션 생성 → 개발 DB에 적용 → Client 코드 재생성
```

> 모노레포(pnpm workspace)에서는 명령 앞에 위치만 맞추면 동일하게 동작 — [[NestJS_Prisma_Monorepo]] 참고

## migrate dev / migrate deploy / generate

|명령|언제|동작|
|---|---|---|
|`migrate dev --name x`|로컬 개발|마이그레이션 생성 + DB 적용 + Client 재생성|
|`migrate deploy`|배포/CI|기존 마이그레이션만 순서대로 적용 (새로 생성 안 함)|
|`generate`|Client만 다시|DB 변경 없이 타입만 재생성|

```txt
migrate reset / resolve / diff / --create-only / seed / 커스텀 SQL 추가 / 에러 처리
→ [[NestJS_Migration]] 참고
```

## Prisma 6 → 7 주요 변경 ⭐️⭐️

|항목|6 (옛 튜토리얼)|7 (현재)|
|---|---|---|
|DB url 위치|`schema.prisma` 안|`prisma.config.ts`|
|generator provider|`prisma-client-js`|`prisma-client` + `output` 필수|
|import 경로|`@prisma/client`|`output` 지정 경로 기준|
|DB 연결|`new PrismaClient()`|adapter 필요 (`@prisma/adapter-pg` 등)|

```txt
인터넷 튜토리얼 대부분 6 기준 — "따라했는데 안 됨"의 흔한 원인이 이 표의 차이들
이런 식으로 라이브러리 메이저 버전이 바뀌며 import 경로/타입 위치가 통째로 바뀌는 건
Prisma만의 특징이 아님 — 다른 라이브러리에서의 비슷한 사례는 [[React_Charts]] 참고
```

## 타입이 안 보일 때 체크리스트 ⭐️⭐️

```txt
① schema에 모델/필드 정확히 들어갔는지
② migrate dev (또는 generate) 실행
③ 서버 재시작 ⭐️ — Node는 require한 모듈을 메모리에 캐싱해서, 파일이 새로 생겨도
   이미 떠 있는 프로세스는 재시작 전까지 옛 버전을 그대로 씀 (가장 자주 빠뜨리는 단계)
④ (그래도 안 되면) 에디터 TS 서버 재시작
```

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

```txt
이 파일만 process.env 직접 사용 — CLI(migrate/generate)는 NestJS 부팅 없이 실행되어 ConfigService를 못 씀
(NestJS 런타임/Prisma CLI/Docker Compose가 .env를 각자 다르게 읽는 전체 그림은 [[NestJS_Env_Config]] 참고)
```

## DATABASE_URL 구조 ⭐️

```bash
postgresql://user:password@host:port/dbname?schema=public&sslmode=disable
```

|구간|의미|
|---|---|
|`postgresql://`|DB 종류|
|`user:password`|접속 계정|
|`host:port`|접속 주소|
|`dbname`|사용할 DB|
|`?schema=`|기본값 `public`, 보통 그대로 사용|
|`sslmode=`|로컬(Docker)→`disable` / 클라우드(Neon 등)→`require`|

```txt
⚠️ prisma init으로 자동 생성된 URL은 Prisma 클라우드 주소 — 직접 쓰는 DB 주소로 반드시 교체
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

```txt
moduleFormat 누락 시 흔한 에러: "exports is not defined in ES module scope"
→ Prisma 7 기본 출력(ESM)과 NestJS 빌드(CJS) 형식 불일치가 원인
```

## generator json — Json 필드에 직접 타입 붙이려면 (선택) ⭐️⭐️⭐️

```prisma
generator json {
  provider = "prisma-json-types-generator"
}
```

```bash
pnpm add -D prisma-json-types-generator
```

```txt
이 generator 블록을 schema.prisma에 추가하고 패키지를 설치해야, 필드 뒤에 붙이는
/// [TypeName] 주석이 실제로 동작함 — generator 블록 없이 주석만 적어두면 그냥 무시되는 일반 텍스트일 뿐

추가한 뒤에는 prisma generate(또는 migrate dev)를 다시 돌려야 적용됨
(generator를 새로 추가/변경한 경우도 위 "타입이 안 보일 때 체크리스트"와 같은 순서로 확인할 것)

실제 /// [TypeName] 사용법과 PrismaJson 네임스페이스 작성법은
"Prisma namespace 타입"의 "더 나은 대안 — Json 필드에 직접 타입 붙이기" 섹션 참고
```

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
PrismaService는 보통 src/prisma/에 둠 — schema.prisma(루트의 prisma/)와는 다른 위치이니 혼동 주의
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
@Global() 자체의 동작 원리(왜 한 번만 import해도 되는지, isGlobal 옵션과의 관계)는 [[NestJS_Module]] 참고
여기서는 PrismaService에 적용한 예시만 — PrismaService는 거의 모든 기능 모듈이 필요로 하는 성격이라
@Global()의 대표적인 적용 후보로 꼽힘 (반대로 일부 모듈만 쓰는 서비스는 명시적 import가 더 명확함)
```

> 참조 [[NestJS_Module]]

---

# Model — 테이블 정의

```prisma
model User {
  id        Int      @id @default(autoincrement())
  email     String   @unique
  name      String?                       // ? = nullable
  createdAt DateTime @default(now()) @db.Timestamptz(3)
  updatedAt DateTime @updatedAt @db.Timestamptz(3)
  role      Role     @default(USER)
  posts     Post[]                        // 가상 필드, DB 컬럼 아님
}
```

|어노테이션|의미|
|---|---|
|`@id`|Primary Key|
|`@default(autoincrement())`|숫자 ID 자동 증가|
|`@default(uuid())`|UUID 자동 생성, 예측 어려움|
|`@unique`|UNIQUE 제약|
|`?`|nullable (없으면 `NOT NULL`)|
|`@updatedAt`|수정 시 자동 갱신|
|`@db.VarChar(n)`|`VARCHAR(n)` 명시|
|`@db.Uuid`|PostgreSQL 네이티브 `uuid` 타입으로 저장 (안 붙이면 `TEXT`) — 아래 참고|
|`@db.Timestamptz(n)`|PostgreSQL 네이티브 `timestamptz(n)` 타입으로 저장 (안 붙이면 `timestamp`) — 아래 참고|

## @db.Uuid — UUID 컬럼을 네이티브 타입으로 ⭐️⭐️

```prisma
model Like {
  id               String @id @default(uuid()) @db.Uuid
  recommendationId String @db.Uuid   // 이 PK를 참조하는 FK도 같은 네이티브 타입으로 맞춰야 함
}
```

```txt
@db.Uuid 없이 String @default(uuid())만 쓰면:
  Postgres 컬럼이 TEXT로 생성됨 — UUID 값(36자 문자열)을 그냥 텍스트로 저장
  → 저장 공간을 더 쓰고, 인덱스/비교 연산도 텍스트 비교라 더 느림

@db.Uuid를 붙이면:
  Postgres의 네이티브 uuid 타입(고정 16바이트)으로 컬럼이 생성됨
  → 저장 공간 절약, 인덱스/비교 더 빠름, DB가 "진짜 UUID 형식인지"도 같이 검증
  → Prisma Client 쪽에서 보이는 TS 타입은 여전히 string — @db.Uuid는 DB 컬럼 타입만 바꿈
  (TEXT vs 네이티브 UUID 비교 · v4/v7 차이 → [[PG_Types]] "UUID" 섹션 참고)
```

```txt
⚠️ PK가 @db.Uuid면, 그 PK를 참조하는 FK 컬럼도 반드시 같이 @db.Uuid로 맞춰야 함
   (PK는 네이티브 uuid, FK는 그냥 String/TEXT로 두면 타입이 안 맞아 관계 생성 시 에러)

⚠️ 네이티브 uuid 타입엔 contains/startsWith 같은 문자열 연산자를 못 씀
   (Postgres의 uuid 타입 자체가 LIKE류 연산자를 지원 안 해서 "operator does not exist" 에러)
   → UUID 값으로 부분 검색을 해야 하는 특수한 경우라면 @db.Uuid를 빼고 그냥 String으로 두는 게 나음
```

## uuid() 버전 — v4(기본) vs v7

```prisma
id String @id @default(uuid())     // v4 — 완전 무작위
id String @id @default(uuid(7))    // v7 — 생성 시각 순으로 정렬됨
```

```txt
v7은 값 앞부분에 타임스탬프가 들어가 있어서 생성 순서대로 정렬됨
INSERT가 많은 테이블의 PK라면 v7이 인덱스 단편화를 줄여 더 유리 — 추측하기 어려운 정도는 v4와 동일
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
```

```txt
timestamp vs timestamptz 차이의 PostgreSQL 원리 → [[NestJS_PostgreSQL]]
AT TIME ZONE 연산자 · PostgreSQL 타입 전체(JSONB · UUID · ARRAY · ENUM) → [[PG_Types]]

기존 timestamp → timestamptz 마이그레이션이 필요한 경우:
  schema.prisma만 수정해서는 DB 타입이 바뀌지 않음 → prisma migrate deploy 필수

  -- 마이그레이션 SQL 예시 (기존 데이터가 UTC로 들어갔다고 가정)
  ALTER TABLE "User"
    ALTER COLUMN "createdAt" TYPE TIMESTAMPTZ(3)
    USING "createdAt" AT TIME ZONE 'UTC';
```

## @@unique — 복합 유니크 ⭐️⭐️⭐️⭐️

```prisma
model Director {
  name String
  dob  DateTime
  @@unique([name, dob])
  // 이름+생년월일 조합이 같은 행은 DB가 물리적으로 거부
}
```

```txt
@unique  vs  @@unique:
  @unique    컬럼 하나가 테이블 전체에서 유일 (한 필드 선언에 바로 붙임)
  @@unique   컬럼 조합이 유일 (두 개 이상의 컬럼을 묶어서 유일성 보장)

  예: userId + postId 조합 → 한 사람이 같은 글에 좋아요 두 번 못 누름
      userId가 여러 행에 있어도 되고, postId가 여러 행에 있어도 됨
      둘의 "조합"만 중복이 안 되면 됨
```

## 조회 — 복합 unique 키 이름 규칙

```typescript
// @@unique([name, dob]) → Prisma가 자동으로 name_dob 라는 복합 키 이름 생성
await prisma.director.findUnique({
  where: {
    name_dob: { name: '봉준호', dob: new Date('1969-09-14') },
    //  ↑ 필드명_필드명 형태로 묶임 ⚠️ 헷갈리기 쉬운 포인트
  },
});

// @@unique([userId, postId]) → userId_postId
await prisma.postLike.findUnique({
  where: { userId_postId: { userId: 1, postId: 42 } },
});
```

```txt
복합 unique 이름 커스텀 (name 옵션):
  @@unique([userId, postId], name: "user_post_like")
  → where: { user_post_like: { userId, postId } }

  자동 생성 이름(필드명_필드명)이 길거나 읽기 어려울 때 사용
```

## 중복 요청 방어 — P2002 에러 처리 ⭐️⭐️⭐️⭐️

```typescript
// @@unique / @unique 위반 시 Prisma가 던지는 에러: P2002
async createLike(userId: number, postId: number) {
  try {
    return await this.prisma.postLike.create({
      data: { userId, postId },
    });
  } catch (e) {
    if (
      e instanceof Prisma.PrismaClientKnownRequestError &&
      e.code === 'P2002'
    ) {
      // 어떤 필드 조합이 충돌했는지
      // e.meta.target → ['userId', 'postId']
      throw new ConflictException('이미 좋아요를 눌렀습니다.');
    }
    throw e;
  }
}
```

```txt
P2002가 발생하는 시점:
  DB가 INSERT/UPDATE를 실행하는 순간 unique 제약 위반 감지
  → 코드에서 "이미 있는지 먼저 조회"하는 방어 코드 없이도 DB가 보장
  → "조회 → 없으면 INSERT" 패턴보다 안전
    (조회와 INSERT 사이에 다른 요청이 끼어들 수 있는 race condition 방지)

버튼 두 번 클릭, 네트워크 재시도 등 중복 요청 방어의 가장 기본 패턴
→ 더 복잡한 중복 방어 전략(멱등키, 낙관적/비관적 잠금) → [[NestJS_Idempotency]]
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
같은 조합으로 INSERT 시도 시 DB가 직접 거부(Prisma 에러 P2002) — 코드의 사전 체크 없이도 안전망 역할
조회: findUnique({ where: { movieId_userId: { movieId, userId } } })
```

## @@index — 인덱스 (자주 찾는다는 힌트) ⭐️

```prisma
@@index([area])              // 단일
@@index([area, feeType])     // 복합 — 순서 중요(앞쪽 컬럼 단독 검색도 효과 있음)
```

```txt
"이 컬럼으로 자주 찾는다"는 힌트
WHERE/ORDER BY에 자주 쓰는 컬럼에 추가 — 조회는 빨라지지만 쓰기는 약간 느려짐(트레이드오프)
@id/@unique는 자동으로 인덱스 생성됨, 그 외 컬럼은 수동 추가
```

## @@index vs @@unique — 언제 뭘 쓰나 ⭐️⭐️⭐️

|구분|`@@index`|`@@unique`|
|---|---|---|
|목적|조회 속도만 빠르게|중복 자체를 DB가 막음 (+ 조회도 빠름)|
|중복 행|허용 (같은 값 여러 행 가능)|불허 (같은 조합 INSERT 시 에러 `P2002`)|
|판단 기준|"이 컬럼으로 자주 검색/정렬하는가"|"이 조합이 두 번 있으면 안 되는가(비즈니스 규칙)"|
|예시|`postId`로 그 글의 댓글 목록 조회 — 한 글에 여러 행이 당연함|`(userId, postId)` — 한 사람이 같은 글에 좋아요 두 번 못 누름|

```txt
판단 한 줄: 중복이 "버그"면 @@unique, 중복이 "정상"인데 그냥 빠르게 찾고 싶으면 @@index

⚠️ @@unique는 자동으로 인덱스 역할도 겸함 — 같은 컬럼 조합에 @@index를 따로 또 만들 필요 없음
   (반대로 @@index는 중복을 막지 못함 — "빠르게 찾기"만 하고, 데이터 무결성 보장은 없음)
```

---

# Scalar Types

> PostgreSQL 타입 상세(timestamp · JSONB · UUID · ARRAY · ENUM 원리) → [[PG_Types]]

|Prisma|PostgreSQL|설명|
|---|---|---|
|`String`|TEXT|문자열|
|`Int` / `BigInt`|INTEGER / BIGINT|정수|
|`Float` / `Decimal`|REAL / NUMERIC|부동소수 / 정밀 소수(금액)|
|`Boolean`|BOOLEAN|true/false|
|`DateTime`|`TIMESTAMP` / `TIMESTAMPTZ`|날짜+시간 — `@db.Timestamptz(3)` 명시 권장 (위 참고)|
|`Json`|JSONB|JSON|

```prisma
enum Role { USER  ADMIN  MODERATOR }
```

```txt
⚠️ Enum도 Prisma Client 생성 경로에서 import — 다른 곳(예: 예전 TypeORM entity 파일)에서 가져오면 타입 불일치
```

---

# Relations — 관계

```prisma
// One to Many
model User { posts Post[] }
model Post { authorId Int;
author User @relation(fields: [authorId], references: [id]) }

// One to One — FK에 @unique 추가
model Profile {
userId Int @unique;
user User @relation(fields: [userId], references: [id]) }

// Many to Many — Prisma가 중간 테이블 자동 생성
model Post { tags Tag[] }
model Tag  { posts Post[] }
```

```txt
fields: 내가 들고 있는 FK 컬럼 / references: 상대 테이블 PK
```

|onDelete|동작|
|---|---|
|`Cascade`|부모 삭제 → 자식도 삭제|
|`SetNull`|부모 삭제 → FK를 NULL로|
|`Restrict` (기본)|참조 중이면 삭제 불가|

## 관계 이름 — `@relation("이름")` ⭐️⭐️⭐️

|구분|역할|
|---|---|
|`@relation(fields, references)`|FK 들고 있는 쪽|
|`@relation("이름")`만|반대편(back-relation) 표시|
|`"문자열"`|DB와 무관, 양쪽을 짝짓는 Prisma 전용 키 — 양쪽 동일해야 함|

```txt
필요 조건: 같은 모델(자기 자신 포함)을 한 모델에서 2번 이상 참조할 때만
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
안 붙이면 → Ambiguous relation detected 에러
같은 패턴: self-relation(teacher/students 둘 다 User), 한 모델 안 관계 2개(author/pinnedBy 둘 다 User)
```

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
// mode: 대소문자 무시 (PostgreSQL 한정, MySQL은 기본값이 무시라 불필요)
{ title: { contains: 'art', mode: 'insensitive' } }

// AND(기본) / OR / NOT
where: { isVisible: true, genre: 'drama' }                              // AND
where: { OR: [{ name: { contains: '김' } }, { email: { contains: 'gmail' } }] }
where: { name, dob, NOT: { id } }   // findFirst에서만 가능
```

---

# select / omit / include ⭐️⭐️

|키워드|용도|
|---|---|
|`select`|가져올 필드만 지정|
|`omit`|뺄 필드만 지정 (나머지 전부)|
|`include`|관계(연결된 테이블) 함께 조회|

```typescript
this.prisma.user.findMany({ select: { id: true, email: true } });
this.prisma.user.findMany({ omit: { password: true } });     // password만 빼고 전부

this.prisma.user.findUnique({ where: { id }, include: { posts: true } });  // 관계 통째로
```

## 언제 select, 언제 omit, 언제 아무것도 안 쓰나 ⭐️⭐️⭐️⭐️

```txt
아무 옵션도 안 쓰면: 그 모델의 scalar 필드는 전부 가져옴, 관계는 안 가져옴(기본 동작)
  → 민감한 필드(password 등)가 없는 모델이고, 다 써도 상관없으면 이게 가장 간단함
```

|선택|동작 방식|언제 더 안전/편한가|
|---|---|---|
|`select`|화이트리스트 — 명시한 필드만 결과에 포함|모델에 비밀번호 등 민감 필드가 있을 때 — 나중에 모델에 새 필드가 추가돼도, select에 직접 추가하지 않는 한 자동으로 노출되지 않음|
|`omit`|블랙리스트 — 명시한 필드만 빼고 나머지 전부 포함|필드가 많고 뺄 게 1~2개뿐일 때 — 짧지만, 나중에 모델에 새로 추가된 민감 필드를 깜빡하고 omit에 안 넣으면 그대로 노출됨|

```txt
판단 기준: "이 모델에 민감한 필드(password, refreshToken 등)가 있는가?"
  있다면 → select(화이트리스트)가 더 안전함 — 새 필드가 추가돼도 기본적으로 안 보이는 쪽으로 안전하게 닫혀있음
  없다면 → omit이 더 짧고 편함, 또는 그냥 select/omit 없이 다 가져와도 무방

이 "화이트리스트가 기본적으로 더 안전하다"는 발상은 [[NestJS_DTO]]의
whitelist: true(ValidationPipe)와 같은 원리 — 명시한 것만 통과시키고, 나머지는 자동으로 막힘
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

```txt
include 안의 관계 필드도 findMany와 같은 옵션(where/orderBy/take/skip/select)을 그대로 받음
→ "관계 = 또 하나의 작은 조회"로 생각하면 됨
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

```txt
⚠️ include를 깊게/넓게 쓸수록 한 번에 가져오는 데이터가 커짐 — 화면에 필요한 만큼만
```

---

# select/include 객체를 재사용 가능한 상수로 빼기 ⭐️⭐️⭐️⭐️

```typescript
const adminUserSelect = {
  id: true,
  email: true,
  nickname: true,
  role: true,
  createdAt: true,
} as const;
```

```txt
이렇게 별도 상수로 빼는 이유는 두 가지가 섞여 있음 — 따로 구분해서 보면 헷갈리지 않음:

① 재사용(코드 조직) — as const와 무관한, 그냥 "변수로 빼서 여러 쿼리에 같이 쓰기" 목적
   findMany/findUnique 등 여러 곳에서 같은 필드 목록을 또 타이핑하지 않고 import해서 재사용
   필드를 추가/삭제할 때 한 곳만 고치면 모든 쿼리에 한꺼번에 반영됨

② as const 자체 — 타입 추론 정밀도를 위한 것, 재사용 여부와는 별개의 이유
   (바로 아래에서 자세히)
```

## as const가 왜 필요한가 — true가 boolean으로 "넓혀지는" 문제 ⭐️⭐️⭐️⭐️

```typescript
const withoutConst = { id: true, email: true };
// TS가 추론하는 타입: { id: boolean; email: boolean }
//                          ↑ 'true'라는 구체적인 값이 아니라 "boolean이라는 종류"로 넓혀짐(widening)

const withConst = { id: true, email: true } as const;
// TS가 추론하는 타입: { readonly id: true; readonly email: true }
//                          ↑ 'true'라는 구체적인 값 그대로 유지됨
```

```txt
Prisma의 select 타입은 "이 필드가 true인 경우에만 결과 타입에 그 필드를 포함시킨다"는
조건부 타입(conditional type)으로 동작함 — 그런데 이게 정확히 동작하려면 그 필드 값이
"true라는 구체적인 값"이어야 함, 그냥 "boolean(true일 수도 false일 수도 있는 어떤 값)"으로는
"이 필드를 포함할지 말지가 컴파일 타임에 확정 안 됨"으로 취급되어 정확한 추론이 안 됨

as const 없이 넘기면: Prisma가 "이 select 객체에 어떤 필드들이 진짜 포함되는지"를
정확히 못 좁혀서, 결과 타입이 너무 넓어지거나(모든 필드가 optional처럼 되거나) 타입 에러가 날 수 있음

as const를 붙이면: id/email 같은 각 속성이 boolean이 아니라 정확히 true(리터럴 타입)로 고정됨
→ Prisma가 "이 select 객체는 정확히 id와 email만 true다"라고 컴파일 타임에 확정적으로 알 수 있어서,
  반환 타입을 { id: number; email: string }처럼 정확하게 좁혀서 추론해줌

→ 결론: as const는 "재사용을 위해서"가 아니라 "Prisma가 정확한 반환 타입을 추론하게 하려고" 붙이는 것
  (재사용은 변수로 빼는 것 자체로 이미 가능 — as const가 없어도 변수로는 빼짐, 다만 타입 추론이 부정확해짐)
  (리터럴 타입을 유지하는 as const의 일반적인 동작은 [[NestJS_DTO]]의
   "허용 값을 별도 상수 파일로 분리하기"에서 다룬 것과 같은 원리)
```

## 상황별로 다른 select/include 모양 — 세 가지 비교 ⭐️⭐️⭐️⭐️

```typescript
// ① 단순 select — scalar 필드만 고를 때
const adminUserSelect = {
  id: true, email: true, nickname: true, role: true, createdAt: true,
} as const;

// ② include 안에 select — 관계를 가져오되, 그 관계의 일부 필드만
const savedCardInclude = {
  recommendation: {
    select: { id: true, title: true, artist: true, createdAt: true },
  },
} as const;

// ③ include인데 실제 데이터가 아니라 개수만 — _count
const adminUserInclude = {
  _count: {
    select: { recommendations: true, reactions: true, savedCards: true },
  },
} as const;
```

|패턴|언제 쓰나|
|---|---|
|① 단순 select|관계 없이 그 모델 자체의 필드 일부만 필요할 때|
|② include 안에 select|관계 데이터를 가져오되, 그 관계 모델의 전체가 아니라 일부 필드만 필요할 때 (위 "include 안에서 필터·정렬·select 같이 쓰기" 참고)|
|③ include + `_count`만|관계의 실제 데이터는 필요 없고, "몇 개 있는지"만 필요할 때 (위 "`_count`" 참고)|

## `_count`는 select 안에서도 쓸 수 있음 — 재사용 트릭 ⭐️⭐️⭐️⭐️

```typescript
this.prisma.user.findMany({
  select: {
    ...adminUserSelect,             // ① 단순 select 필드들을 그대로 펼치고
    _count: adminUserInclude._count, // ③ include용으로 만들어둔 _count 설정을 재사용
  },
});
```

```txt
_count는 사실 select와 include 둘 다에서 쓸 수 있음 — "관계의 개수만 보고 싶다"는
같은 기능이 어느 쪽 옵션을 쓰든 동일하게 제공됨

→ 위 코드는 adminUserInclude를 "원래는 include용으로 만들어둔 객체"였지만,
  그 안의 _count 부분만 따로 꺼내서(adminUserInclude._count) select 객체 안에 끼워넣은 것
  (스프레드(...)로 객체를 펼치는 동작 자체는 [[JS_Operators]] 참고)

이렇게 하면 _count의 select 모양(어떤 관계를 카운트할지)을 한 곳에서만 정의해두고,
"이 모델을 select로 조회할 때"와 "include로 조회할 때" 양쪽에서 똑같이 재사용할 수 있음
```

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

# Prisma namespace 타입

```txt
prisma generate를 실행하면, schema.prisma에 정의한 모델마다 다음 타입들이 자동으로 만들어짐:
  {모델명}WhereInput    그 모델로 조건(where)을 걸 때 쓸 수 있는 타입
  {모델명}CreateInput   그 모델을 create할 때 data에 넣을 수 있는 타입
  {모델명}UpdateInput   그 모델을 update할 때 data에 넣을 수 있는 타입 (전 필드 optional)

모델을 하나 추가하면 이 세 타입도 자동으로 같이 생김 — 직접 선언할 필요 없이
Prisma 네임스페이스 안에서 바로 꺼내 씀 (User 모델이면 Prisma.UserWhereInput 등)
```

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

## Json 필드 — JsonValue vs InputJsonValue ⭐️⭐️⭐️⭐️

> PostgreSQL JSONB 타입 원리(JSON vs JSONB · GIN 인덱스 · 연산자) → [[PG_Types]] "JSONB" 섹션

```prisma
model Post {
  customization Json
}
```

```txt
schema에 Json으로 선언한 필드는, Prisma Client에서 "읽을 때"와 "쓸 때"의 타입이 다름:
  Prisma.JsonValue       읽을 때(조회 결과)의 타입
  Prisma.InputJsonValue  쓸 때(create/update의 data)의 타입

둘 다 string | number | boolean | null | JsonObject | JsonArray 형태의 재귀적 유니온 타입 —
TS가 자동으로 좁혀주지 않아서, 중첩된 값을 다루려면 단언이나 타입가드가 필요한 경우가 많음
(as 자체의 위험성은 [[TS_TypeAssertion]] 참고)
```

```typescript
customization: dto.customization as Prisma.InputJsonValue,
```

```txt
DTO의 customization 필드가 보통 구체적인 인터페이스나 Record<string, any> 같은 타입으로
선언돼 있는데, 그 타입을 Prisma.InputJsonValue에 그대로 넘기면 TS가 거부하는 경우가 흔함

가장 자주 걸리는 원인 — 옵셔널 속성(?):
  interface Customization { theme?: string; }
  theme?: string는 TS 내부적으로 theme: string | undefined로 다뤄짐
  근데 undefined는 유효한 JSON 값이 아님(JSON에는 undefined가 없음, null만 있음)
  → "옵셔널 속성이 있는 객체 타입"은 구조적으로 InputJsonValue와 안 맞는다고 TS가 판단함
  → 실제 런타임 값은 멀쩡한 JSON인데도 타입 레벨에서는 막혀서, 단언(as)으로 우회하게 됨
  (이건 TS 자체의 잘 알려진 한계 — Prisma만의 결함이 아니라 옵셔널 속성과 JSON 타입의 구조적 차이에서 옴)
```

### 더 나은 대안 — Json 필드에 직접 타입 붙이기 ⭐️⭐️⭐️

```txt
매번 as로 우회하는 대신, schema.prisma의 Json 필드 위에 주석으로 실제 타입을 지정해두는 방법도 있음
(prisma-json-types-generator 같은 별도 생성기 설치 + schema.prisma에 generator 블록 추가가 먼저 필요 —
 "schema.prisma 기본 구성"의 "generator json" 섹션 참고)
```

```prisma
customization Json /// [CustomizationType]
```

```txt
⚠️ 이 /// 주석은 필드 "위 줄"이 아니라 필드와 "같은 줄, 뒤쪽"에 붙여야 PrismaJson이 인식함
   (Prisma의 일반적인 /// 문서 주석은 보통 위에 붙이는 관례라 헷갈리기 쉬운데,
    prisma-json-types-generator는 같은 줄 트레일링 위치를 요구함)
```

```typescript
// types.ts
declare global {
  namespace PrismaJson {
    type CustomizationType = { theme?: string; layout?: string };
  }
}
```

```txt
이렇게 해두면 Prisma Client가 만드는 타입 자체가 Prisma.JsonValue 대신 CustomizationType이 되어서,
조회/쓰기 양쪽에서 더 정확한 자동완성을 받고 as 단언도 줄어듦
→ 지금처럼 as로 한두 곳만 우회하는 정도면 충분하고, Json 필드가 많아지고 자주 다루게 되면 검토할 만함
```

### JSON의 null — Prisma.JsonNull vs Prisma.DbNull ⭐️⭐️

```typescript
await this.prisma.post.update({
  where: { id },
  data: { customization: Prisma.JsonNull }, // 컬럼에 "JSON 값으로서의 null"을 저장
});
```

```txt
Json 컬럼에서는 두 가지 "없음"이 구분됨:
  Prisma.DbNull   SQL의 NULL — 컬럼 자체에 값이 없음
  Prisma.JsonNull JSON의 null — 컬럼에 값은 있는데 그 값이 JSON null임

그냥 null을 넘기면 Prisma가 이 둘 중 어느 의도인지 헷갈릴 수 있어서,
Json 필드를 명시적으로 "비워달라"고 할 때는 이 전용 상수를 쓰는 게 정확함
```

---

# $transaction — 트랜잭션

|방식|특징|
|---|---|
|인터랙티브 (권장)|콜백 안에서 조건 분기, 실패 시 throw로 자동 롤백|
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
error.meta → P2002면 { target: ['email'] } 처럼 어떤 컬럼이 중복인지 알려줌
```

---

# 자주 만나는 에러

|증상|원인|해결|
|---|---|---|
|복합 unique 에러|`findUnique({ where: { name, dob } })` 처럼 따로 넘김|`findUnique({ where: { name_dob: { name, dob } } })`|
|TypeORM 문법 사용|`find({ relations: {...} })`|`findMany({ include: {...} })`|
|타입 자동완성 안 바뀜|`migrate dev`/`generate` 누락 또는 서버 재시작 안 함|위 체크리스트 참고|
|`Ambiguous relation detected`|같은 모델을 두 번 참조하는데 관계 이름 없음|충돌하는 두 필드에 `@relation("이름")`, 양쪽 동일하게|

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

# 한눈에

```txt
반복 루프: ① schema 수정 → ② migrate dev → ③ 사용
타입 안 보이면: migrate dev/generate 했는지 → 서버 재시작했는지 순서로 확인 ⭐️

같은 모델을 2번 이상 참조하면 @relation("이름") 필수, 양쪽 동일 문자열

DateTime 필드는 반드시 @db.Timestamptz(3) 명시:
  안 붙이면 timezone-naive TIMESTAMP → 세션 TZ에 따라 해석이 달라질 수 있음
  붙이면 UTC instant로 저장 — 어느 환경에서도 같은 순간을 가리킴
  PostgreSQL 타입 원리 → [[NestJS_PostgreSQL]]

조회: unique 컬럼 하나 → findUnique / 자유조건+1건 → findFirst / 여러 건 → findMany
select(필요한 것만, 화이트리스트 — 민감 필드 있으면 더 안전) / omit(뺄 것만, 블랙리스트) /
  include(관계, 내부에서 where·orderBy·take·select 다 가능)
  → select+include 동시 사용 불가, 개수만 필요하면 _count(select/include 둘 다에서 가능)

select/include 객체를 상수로 뺄 때 as const 필수 — true가 boolean으로 넓혀지면
Prisma가 정확한 반환 타입을 추론 못 함 (재사용은 변수로 빼는 것 자체로 가능, as const는 별개 이유)

모델마다 {모델명}WhereInput/CreateInput/UpdateInput이 generate 시 자동으로 같이 생김

Json 필드는 읽기(JsonValue)/쓰기(InputJsonValue) 타입이 다름 — 옵셔널 속성이 있는 DTO 타입은
구조적으로 안 맞아서 as Prisma.InputJsonValue 단언이 자주 필요함 (근본 해결은 prisma-json-types-generator)
JSON null을 명시적으로 쓰려면 Prisma.JsonNull (SQL NULL은 Prisma.DbNull)

에러는 Prisma.PrismaClientKnownRequestError + error.code 분기 (P2002 중복 / P2003 FK / P2025 없음)

Prisma 6 튜토리얼대로 안 되면 버전 차이부터 의심 (prisma.config.ts / adapter / output)
모노레포 명령 실행 디테일 → [[NestJS_Prisma_Monorepo]]
```