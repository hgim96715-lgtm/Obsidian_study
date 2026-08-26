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
# NestJS_Seed — Seed · CLI 패턴

>[!info]
>Seed = 개발·스테이징 환경에서 DB에 초기 데이터를 넣는 작업. 
>단순 bulk insert라면 Prisma seed + `createMany`로 충분. **비즈니스 로직(규칙·카운트·FK 검증)을 그대로 타야 한다면** `NestFactory.createApplicationContext`로 DI 컨테이너만 띄워서 기존 `@Injectable()` 서비스를 직접 호출한다. 
>운영 환경 실수 방지를 위해 env 가드 필수.

---

# Seed란 ⭐️⭐️⭐️⭐️

```txt
Seed = DB에 초기 데이터를 심는 스크립트

DB를 처음 만들거나 초기화했을 때:
  로그인할 계정이 없음
  테스트할 게시글이 없음
  화면이 텅 비어있어서 개발이 어려움
  → Seed를 실행하면 필요한 데이터가 자동으로 들어감

비유:
  화분(DB)에 흙(schema)을 채웠으면
  씨앗(seed)을 심어야 식물(데이터)이 자람

언제 실행하는가:
  새 개발 환경을 세팅할 때         (clone 후 초기화)
  migrate reset으로 DB를 날렸을 때
  스테이징 환경에 데모 데이터가 필요할 때

운영(production)에는 쓰지 않는다:
  실제 서비스 DB에 가짜 데이터가 들어가면 안 됨
  → 환경변수 가드로 실수 방지
```

---

# 두 가지 방법 — 언제 뭘 쓰는가 ⭐️⭐️⭐️⭐️

```txt
방법 1 — Prisma seed (prisma/seed.ts):
  NestJS와 무관하게 PrismaClient를 직접 써서 DB에 넣음
  비즈니스 로직을 타지 않음 → 빠르고 단순
  → 관리자 계정, 카테고리, 코드 테이블 같은 필수 초기값

방법 2 — NestJS CLI (src/cli/*.ts):
  DI 컨테이너를 띄워서 기존 서비스를 호출
  비즈니스 로직(유효성 검사, 카운트 집계, FK 검증)이 그대로 실행됨
  → 실제 사용자 행동을 시뮬레이션하는 데모 데이터
  → 규칙이 복잡한 도메인 데이터 (후기, 티켓, 활동 집계 등)

Prisma seed를 쓰면 안 되는 경우:
  서비스 레이어에서 검증하는 "하루 1회 제한" 같은 규칙
  countIncrement 같은 집계가 함께 돌아야 하는 경우
  JWT 발급·인증 플로우를 거쳐야 하는 경우
```

---

# 파일 구조 ⭐️⭐️⭐️⭐️

```txt
방법 1 (Prisma seed):
  prisma/
    seed.ts             PrismaClient 직접 사용
  package.json          "prisma.seed" 항목에 등록

방법 2 (NestJS CLI):
  apps/api/src/cli/
    demo-seed.ts        메인 시드 스크립트 (nest build 대상)
    demo-purge.ts       데모 데이터 일괄 삭제
  apps/api/package.json "scripts"에 등록

  disposable/demo-seed/ 오픈 전 통째 삭제할 "쓰레기통"
    personas.json       닉네임 풀·후기 템플릿 (런타임에 fs로 읽음)
    config.ts           수치·이메일 규칙 (문서용)

분리 이유:
  cli/        → nest build 대상. Prisma·DI와 같은 컴파일 파이프에 포함
  disposable/ → 코드가 아닌 데이터 파일만. 오픈 전 디렉터리째 삭제 가능

선택 기준:
  Prisma seed → 대량 bulk insert, 비즈니스 로직 없이 그냥 행을 넣으면 됨
  NestJS CLI  → register·login·도메인 액션을 순서대로 거쳐야 함
```

---

# 방법 1 — Prisma seed (단순 데이터) ⭐️⭐️⭐️

```typescript
// prisma/seed.ts
import { PrismaClient } from '../generated/prisma/client';

const prisma = new PrismaClient();

async function main() {
  // upsert — 여러 번 실행해도 중복 없음 (멱등)
  await prisma.user.upsert({
    where:  { email: 'admin@example.com' },
    update: {},
    create: {
      email:    'admin@example.com',
      nickname: '관리자',
      role:     'admin',
      password: await bcrypt.hash('Admin1234!', 10),
    },
  });
  console.log('✅ Seed 완료');
}

main()
  .catch(e => { console.error(e); process.exit(1); })
  .finally(() => prisma.$disconnect());
```

```json
// package.json
{
  "prisma": {
    "seed": "ts-node --compiler-options {\"module\":\"CommonJS\"} prisma/seed.ts"
  }
}
```

```bash
pnpm prisma db seed
pnpm prisma migrate reset   # reset 시 자동으로 seed 실행
```

---

# 방법 2 — NestJS CLI (서비스 레이어 활용) ⭐️⭐️⭐️⭐️

## 핵심 개념 — ApplicationContext

```typescript
import { NestFactory } from '@nestjs/core';
import { AppModule }   from '../app.module';

async function bootstrap() {
  // HTTP 서버 없이 DI 컨테이너만 띄움
  const app = await NestFactory.createApplicationContext(AppModule);

  // 기존 서비스를 꺼내서 직접 호출
  const authService   = app.get(AuthService);
  const reviewService = app.get(ReviewPostService);

  await authService.register({ email, password, nickname });
  await reviewService.create(userId, dto);

  await app.close();  // 반드시 종료
}
bootstrap();
```

```txt
createApplicationContext(AppModule):
  HTTP 서버(@nestjs/platform-express)는 안 올라감
  DI 컨테이너·ConfigService·Prisma·서비스만 올라옴
  → app.get(서비스) 로 원하는 서비스를 꺼낼 수 있음

컨트롤러를 직접 부르지 않는 이유:
  파이프·JWT Guard·@UserId() 데코레이터는 컨트롤러 책임
  CLI는 userId를 이미 알고 있으니 서비스를 직접 호출하면 됨
  컨트롤러 우회 없이 짧고 명확하게

app.close():
  안 하면 프로세스가 종료 안 될 수 있음
  finally 블록에서 반드시 호출
```

## 왜 nest build 후 실행인가

```txt
CLI 실행 방법 비교:

  tsx / esbuild로 AppModule 직접 실행:
    emitDecoratorMetadata 없음 → ConfigService injection 실패

  ts-node로 src/ import:
    Prisma 7 generated client의 .js 상대 import 해석 실패

  nest build 후 dist/*.js 실행: ✅
    tsc + Nest와 동일 경로·DI 정상 동작
```

```json
// package.json
{
  "scripts": {
    "seed:demo": "nest build && node -r dotenv/config dist/cli/demo-seed.js",
    "seed:purge": "nest build && node -r dotenv/config dist/cli/demo-purge.js"
  }
}
```

```txt
-r dotenv/config:
  dist/ 실행 전에 apps/api/.env 를 로드
  ENABLE_DEMO_SEED, DEMO_SEED_PASSWORD 등을 process.env에 주입
```

## 파일 구조

```txt
apps/api/src/cli/
  demo-seed.ts          메인 시드 스크립트 (build 대상)
  demo-purge.ts         demo 데이터 일괄 삭제

disposable/demo-seed/
  config.ts             수치·이메일 규칙 (문서용)
  personas.json         닉네임 풀·후기 템플릿
  seed-day.ts           deprecated stub

분리 이유:
  cli/     → nest build 대상, DI·Prisma와 같은 컴파일 파이프
  disposable/ → 오픈 전 통째로 삭제할 "쓰레기통", JSON·설정만
```

## env 가드 — 운영 실수 방지

```typescript
// apps/api/src/cli/demo-seed.ts
function assertEnv() {
  if (process.env.NODE_ENV === 'production') {
    console.error('❌ production에서 demo seed 금지');
    process.exit(1);
  }
  if (process.env.ENABLE_DEMO_SEED !== '1') {
    console.error('❌ ENABLE_DEMO_SEED=1 필요');
    process.exit(1);
  }
  if (!process.env.DEMO_SEED_PASSWORD) {
    console.error('❌ DEMO_SEED_PASSWORD 필요');
    process.exit(1);
  }
}
```

```dotenv
# apps/api/.env  (local / staging — prod 금지)
ENABLE_DEMO_SEED=1
DEMO_SEED_PASSWORD=change-me-demo-seed
```

```txt
왜 두 단계 가드인가:
  NODE_ENV=production 체크 → 배포 환경 자체를 막음
  ENABLE_DEMO_SEED=1 체크  → 스테이징에서도 명시적으로 켜야 실행

DEMO_SEED_PASSWORD:
  seed로 만드는 계정의 비밀번호를 환경변수로 분리
  코드에 비밀번호 하드코딩 방지
  팀원마다 다른 값 사용 가능
```

## 전체 CLI 스크립트 구조

```typescript
// src/cli/demo-seed.ts
import { NestFactory }    from '@nestjs/core';
import { AppModule }      from '../app.module';
import { AuthService }    from '../auth/auth.service';
import { ReviewService }  from '../reviews/review.service';

async function main() {
  assertEnv();  // 가드 먼저

  const app = await NestFactory.createApplicationContext(AppModule);

  try {
    const auth   = app.get(AuthService);
    const review = app.get(ReviewService);

    const PASSWORD = process.env.DEMO_SEED_PASSWORD!;

    // 신규 유저 등록
    for (let seq = 0; seq < NEW_PER_DAY; seq++) {
      const email = demoEmail(seq);   // demo+{date}-{seq}@demo.example.invalid
      const { user } = await auth.register({ email, password: PASSWORD, nickname: pickNickname() });

      await runActivity(app, user.id);
      await sleep(STAGGER_MS);  // 전광판 시간대 분산
    }

    // 기존 유저 재방문
    const returning = await getReturningUsers(app, TOTAL - NEW_PER_DAY);
    for (const user of returning) {
      await auth.login({ email: user.email, password: PASSWORD });
      await runActivity(app, user.id);
      await sleep(STAGGER_MS);
    }

    console.log('✅ Demo seed 완료');
  } finally {
    await app.close();
  }
}

main().catch(e => { console.error(e); process.exit(1); });
```

```typescript
// 이메일 규칙 — 날짜 + 순번으로 고유성 보장
function demoEmail(seq: number): string {
  const date = kstDateKey(new Date());  // "2026-08-20"
  return `demo+${date}-${seq}@demo.example.invalid`;
  // 같은 날 재실행: seq만 증가 → 의도적으로 새 계정 생성
  // .invalid 도메인 → 실제 이메일 없는 가짜 계정임을 명시
}
```

---

# 멱등성 — 여러 번 실행해도 안전하게 ⭐️⭐️⭐️

```typescript
// Prisma seed 방식 — upsert
await prisma.user.upsert({
  where:  { email: 'admin@example.com' },
  update: {},          // 있으면 아무것도 안 바꿈
  create: { ... },     // 없으면 생성
});

// CLI 방식 — 이메일에 날짜+seq 포함
// demo+2026-08-20-0@demo.example.invalid → 같은 날 재실행해도 새 계정
// → 의도적으로 멱등하지 않음 (하루치 활동을 추가로 쌓는 것이 목적)
```

---

# purge — demo 데이터 일괄 삭제

```typescript
// src/cli/demo-purge.ts
async function purge() {
  assertEnv();

  const app = await NestFactory.createApplicationContext(AppModule);
  try {
    const prisma = app.get(PrismaService);

    // 자식 먼저 삭제 → 부모 (외래키 순서)
    await prisma.review.deleteMany({
      where: { author: { email: { contains: '@demo.example.invalid' } } },
    });
    await prisma.user.deleteMany({
      where: { email: { contains: '@demo.example.invalid' } },
    });

    console.log('✅ Demo 데이터 삭제 완료');
  } finally {
    await app.close();
  }
}
```

```txt
.invalid 도메인을 쓰는 이유:
  실제 존재하지 않는 도메인 (.invalid는 RFC로 보장)
  WHERE email LIKE '%@demo.example.invalid' 로 정확히 필터
  실제 유저 데이터와 확실히 구분
```
---

# 스크립트 종료 패턴 — main().catch · process.exitCode ⭐️⭐️⭐️⭐️

```txt
NestJS CLI 스크립트(seed, test-user 등)는 비동기 함수로 작성됨
최상위(top-level)에서 async/await를 쓰려면 async 함수로 감싸야 함
→ 그 함수를 호출하면서 에러를 잡는 패턴이 main().catch(...)
```

## 기본 구조

```typescript
async function main() {
  // ... 실제 작업
}

main().catch((error) => {
  console.error('[script-name] 실패:', error);
  process.exitCode = 1;
});
```

```txt
왜 이렇게 쓰는가:
  async function main()은 Promise를 반환
  → main()을 그냥 호출하면 에러가 나도 무시됨 (unhandled rejection)
  → .catch()로 에러를 반드시 잡아야 함

  .catch() 없이 main() 만 호출하면:
    UnhandledPromiseRejectionWarning 경고 출력
    Node.js v16+ 에서는 프로세스 자체가 종료될 수 있음 (비결정적)
    CI/CD 파이프라인에서 성공으로 처리될 수 있음 (위험)
```

## process.exitCode vs process.exit()

```typescript
// 방법 A: process.exitCode 설정
process.exitCode = 1;
// → 현재 실행 중인 코드가 끝날 때까지 기다렸다가 종료
// → 이벤트 루프가 자연스럽게 빠져나가면서 종료

// 방법 B: process.exit(1)
process.exit(1);
// → 즉시 강제 종료
// → finally 블록, 열린 DB 연결, 파일 핸들 등이 정리 안 될 수 있음
```

```txt
exitCode vs exit() 차이:

  process.exitCode = 1:
    종료 코드를 예약만 해두고 즉시 종료 안 함
    현재 실행 중인 코드가 끝나고 이벤트 루프가 비워지면 자연 종료
    finally 블록이 실행됨 → app.close(), DB 연결 정리 가능
    → 안전한 종료 (권장)

  process.exit(1):
    지금 당장 즉시 종료
    finally 블록이 실행되지 않을 수 있음
    → 긴급 종료나 정말 강제로 끝내야 할 때만 사용

종료 코드:
  0 = 성공 (정상 종료)
  1 = 실패 (에러 발생)
  → CI/CD, shell script에서 0이면 성공, 0이 아니면 실패로 판단
     스크립트 실패 시 exitCode = 1을 안 하면 CI가 성공으로 처리함
```

## try/finally — app.close() 보장

```typescript
async function main() {
  const app = await NestFactory.createApplicationContext(AppModule, {
    logger: false,
  });

  try {
    const prisma = app.get(PrismaService);
    // ... 실제 작업
  } finally {
    await app.close();  // 에러가 나도 반드시 실행됨
  }
}

main().catch((error) => {
  console.error('[test-user] 실패:', error);
  process.exitCode = 1;
});
```

```txt
try/finally를 쓰는 이유:
  try 블록에서 에러가 throw되면 → catch로 가기 전에 finally가 먼저 실행
  → app.close()가 반드시 호출됨 → DB 연결, 열린 핸들 정리
  → 정리 없이 종료되면 프로세스가 hang(멈춤) 상태로 남을 수 있음

app.close()가 없으면:
  PrismaService의 DB 연결이 열린 채로 남음
  Node.js 이벤트 루프가 비워지지 않아 스크립트가 종료 안 됨
  → 터미널이 멈춰 보이는 현상
```

## 전체 패턴 요약

```typescript
async function main() {
  const app = await NestFactory.createApplicationContext(AppModule, {
    logger: false,          // 시작 로그 숨김
  });

  try {
    const service = app.get(SomeService);
    await service.doWork();
    console.log('[script] 완료');
  } finally {
    await app.close();      // 성공이든 실패든 반드시 정리
  }
}

main().catch((error) => {
  console.error('[script] 실패:', error);
  process.exitCode = 1;     // CI/CD가 실패로 인식하도록
});

// main()이 성공하면 → exitCode 기본값 0 → 성공
// main()이 실패하면 → catch에서 exitCode = 1 → 실패
```

```txt
이 패턴이 표준인 이유:
  에러가 나도 DB 연결 등 리소스 정리 보장 (finally)
  CI/CD 파이프라인이 성공/실패를 올바르게 판단 (exitCode)
  unhandled rejection 없이 깔끔하게 에러 처리 (.catch)
  NestJS seed, 마이그레이션 스크립트, CLI 툴 전부 이 패턴 사용
```
