---
aliases:
  - Seed
  - 테스트 데이터
  - Prisma
tags:
  - NestJS
related:
  - "[[00_NestJS_Ecosystem_HomePage]]"
  - "[[NestJS_Prisma]]"
  - "[[NestJS_Env_Config]]"
---
# NestJS_Seed — 시드 (테스트 데이터)

> [!info] 
> 시드(Seed) = 개발/테스트 환경에 가짜 데이터를 넣는 스크립트. 
> 페이지네이션, 검색, UI 레이아웃 등을 실제 데이터처럼 테스트할 때 사용한다. 
> 운영 DB에는 절대 실행하지 않는다.

---

# 시드란? ⭐️⭐️⭐️

```txt
개발 중 이런 상황이 생김:
  "멤버가 1명뿐인데 페이지네이션이 제대로 되는지 어떻게 테스트하지?"
  "검색 기능을 만들었는데 데이터가 없어서 결과가 없음"
  "무한 스크롤 구현했는데 데이터가 5개밖에 없어서 스크롤이 안 됨"

시드 스크립트로 해결:
  npx tsx scripts/seed-members.ts ROOM_ID=xxx COUNT=50
  → DB에 가짜 멤버 50명 즉시 생성
  → 실제처럼 테스트 가능

삭제도 쉽게:
  @seed.local 같은 식별 가능한 이메일 패턴으로 생성
  → cleanup 스크립트로 한 번에 삭제
```

---

# package.json 스크립트 등록 ⭐️⭐️⭐️

```json
{
  "scripts": {
    "prisma:migrate":    "prisma migrate dev",
    "seed:members":      "npx tsx scripts/seed-members.ts",
    "seed:cleanup":      "npx tsx scripts/cleanup-seed-users.ts"
  }
}
```

```bash
# 실행 방법
cd apps/api

# ROOM_ID 없으면 활성 방 목록만 출력
npm run seed:members

# 방 지정
ROOM_ID=019f7dd4-... npm run seed:members

# 방 + 수량 + 접두사 지정
ROOM_ID=019f7dd4-... COUNT=50 PREFIX=bot npm run seed:members

# DRY_RUN — 실제 DB 변경 없이 어떻게 실행되는지 미리 확인
DRY_RUN=1 ROOM_ID=019f7dd4-... npm run seed:cleanup
```

---

# 시드 스크립트 구조 ⭐️⭐️⭐️⭐️

```typescript
// scripts/seed-members.ts
import 'dotenv/config';   // .env 로드 (DATABASE_URL 등)
import { PrismaPg } from '@prisma/adapter-pg';
import { PrismaClient } from '../src/generated/prisma/client';

// 1. 환경변수로 파라미터 받기
const roomId = process.env.ROOM_ID?.trim();
const count  = Math.min(Math.max(Number(process.env.COUNT ?? 20), 1), 200);
const prefix = (process.env.PREFIX ?? 'seed').trim() || 'seed';

// 2. Prisma 직접 연결 (NestJS DI 없이)
const prisma = new PrismaClient({
  adapter: new PrismaPg({ connectionString: process.env.DATABASE_URL }),
});

async function seed() {
  // 3. 생성할 데이터 만들기
  const stamp = Date.now().toString(36);   // 타임스탬프 기반 짧은 고유 문자열
  const users = Array.from({ length: count }, (_, i) => {
    const n = String(i + 1).padStart(3, '0');  // 001, 002, ...
    return {
      email:        `${prefix}_${stamp}_${n}@seed.local`,  // 식별용 도메인
      nickname:     `${prefix}_${n}_${stamp.slice(-4)}`,
      passwordHash: null,  // 시드 유저는 로그인 불필요
    };
  });

  // 4. 트랜잭션으로 원자적 삽입
  const result = await prisma.$transaction(async (tx) => {
    for (const u of users) {
      const user = await tx.user.create({ data: u });
      await tx.member.create({
        data: { roomId, userId: user.id, role: 'member' },
      });
    }
    // 5. 집계 필드 동기화
    const total = await tx.member.count({ where: { roomId } });
    await tx.room.update({
      where: { id: roomId },
      data:  { memberCount: total },
    });
    return total;
  });

  console.log(`✅ ${count}명 추가 · 총 ${result}명`);
}

// 6. main + 에러 처리 + 연결 해제
async function main() {
  if (!process.env.DATABASE_URL) {
    console.error('DATABASE_URL 없음');
    process.exitCode = 1;
    return;
  }
  if (!roomId) {
    // ROOM_ID 없으면 목록 출력 (안내용)
    await listItems();
    return;
  }
  await seed();
}

main()
  .catch((e) => { console.error(e); process.exitCode = 1; })
  .finally(async () => { await prisma.$disconnect(); });
```

---

# 식별 가능한 이메일 패턴 ⭐️⭐️⭐️⭐️

```typescript
email: `${prefix}_${stamp}_${n}@seed.local`
// 예: seed_lf2k8x_001@seed.local
//     bot_lf2k8x_002@seed.local
```

```txt
@seed.local 도메인을 쓰는 이유:
  실제 이메일 주소가 아님 → 실수로 이메일이 발송되는 것 방지
  cleanup 스크립트에서 @seed.local 로 끝나는 계정만 삭제 가능
  운영 데이터와 명확하게 구분됨

Date.now().toString(36):
  현재 타임스탬프를 36진수(0-9a-z)로 변환 → 짧은 고유 문자열
  예: 1718000000000 → 'lf2k8x'
  같은 시간에 여러 번 실행해도 PREFIX나 COUNT로 구분

padStart(3, '0'):
  1 → '001', 10 → '010' — 정렬 시 일관된 순서 유지
```

---

# DRY_RUN 패턴 ⭐️⭐️⭐️

```typescript
const dryRun = process.env.DRY_RUN === '1';

async function cleanup() {
  const targets = await prisma.user.findMany({
    where: { email: { endsWith: '@seed.local' } },
    select: { id: true, email: true },
  });

  console.log(`삭제 대상: ${targets.length}명`);
  targets.forEach((u) => console.log(`  ${u.email}`));

  if (dryRun) {
    console.log('\nDRY_RUN=1 — 실제 삭제 안 함');
    return;
  }

  await prisma.user.deleteMany({
    where: { email: { endsWith: '@seed.local' } },
  });
  console.log(`✅ ${targets.length}명 삭제 완료`);
}
```

```bash
# 삭제 전 확인
DRY_RUN=1 npm run seed:cleanup

# 실제 삭제
npm run seed:cleanup
```

```txt
DRY_RUN 패턴:
  실제 DB를 바꾸기 전에 "무엇을 삭제할 것인지" 미리 확인
  실수로 중요한 데이터를 삭제하는 것 방지

  process.env.DRY_RUN === '1':
    '1' 이외의 값(undefined, '0', 'false')은 모두 false → 실제 실행
```

---

# cleanup 스크립트 패턴 ⭐️⭐️⭐️

```typescript
// scripts/cleanup-seed-users.ts
async function cleanup() {
  // 1. 대상 조회
  const users = await prisma.user.findMany({
    where: { email: { endsWith: '@seed.local' } },
    select: { id: true, email: true, _count: { select: { memberships: true } } },
  });
  console.log(`대상 ${users.length}명`);

  if (dryRun || users.length === 0) return;

  // 2. 관계된 데이터 먼저 삭제 (FK 제약)
  const userIds = users.map((u) => u.id);

  await prisma.$transaction([
    prisma.member.deleteMany({ where: { userId: { in: userIds } } }),
    prisma.user.deleteMany({   where: { id:     { in: userIds } } }),
  ]);

  // 3. 집계 필드 동기화
  // memberCount를 쓰는 경우 cleanup 후 다시 count하여 업데이트 필요
}
```

```txt
FK 제약 순서:
  User를 먼저 삭제하면 → Member(userId FK) 에러
  Member 먼저 삭제 → User 삭제 (참조하는 쪽부터)

$transaction([...]):
  배열로 여러 쿼리를 하나의 트랜잭션으로 실행
  하나라도 실패하면 전체 롤백
```

---

# Prisma 직접 연결 (스크립트용) ⭐️⭐️

```typescript
// NestJS DI 없이 스크립트에서 Prisma 직접 사용
import 'dotenv/config';   // .env 파일 로드 (반드시 최상단)
import { PrismaPg } from '@prisma/adapter-pg';
import { PrismaClient } from '../src/generated/prisma/client';

const prisma = new PrismaClient({
  adapter: new PrismaPg({
    connectionString: process.env.DATABASE_URL,
  }),
});

// 마지막에 반드시 연결 해제
.finally(async () => {
  await prisma.$disconnect();
})
```

```txt
import 'dotenv/config':
  반드시 최상단에 — 다른 import보다 먼저 실행돼야 .env가 로드됨
  없으면 DATABASE_URL 등 환경변수가 undefined

PrismaPg adapter:
  PostgreSQL 연결 방식 — 프로젝트 설정에 따라 다름
  일반 Prisma 연결이면 adapter 없이 new PrismaClient()로도 됨
```

---

# 한눈에

```txt
시드 = 개발용 가짜 데이터 생성 스크립트 (운영 DB에 절대 실행 금지)

식별 패턴:
  이메일: xxx@seed.local  → cleanup으로 한 번에 삭제
  닉네임: prefix_001_stamp

환경변수 파라미터:
  ROOM_ID=uuid  대상 지정
  COUNT=50      생성 수 (Math.min/max로 범위 제한)
  PREFIX=bot    식별 접두사
  DRY_RUN=1     실제 변경 없이 미리 보기

스크립트 구조:
  import 'dotenv/config'  ← 최상단
  환경변수 파싱
  PrismaClient 직접 생성
  main() → .catch → .finally(disconnect)

트랜잭션:
  $transaction(async tx => {...})  여러 테이블 원자적 삽입
  삭제 시 FK 순서: 참조하는 테이블 먼저

DRY_RUN:
  DRY_RUN=1 → 조회만, 실제 변경 없음
  DRY_RUN 없음 → 실제 실행
```