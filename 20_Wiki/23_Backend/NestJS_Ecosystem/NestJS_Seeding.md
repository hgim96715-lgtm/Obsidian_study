---
aliases:
  - Database Seeding
  - 시딩
  - 샘플 데이터
  - Postman Runner
tags:
  - NestJS
related:
  - "[[00_NestJS_Ecosystem_HomePage]]"
---
# NestJS_Seeding — DB 시딩

```txt
Seeding = 개발/테스트용 초기 데이터를 DB 에 미리 넣는 것

방법 1: Prisma seed 스크립트 — 관리자 계정 / 필수 초기 데이터
방법 2: Postman Runner    — 랜덤 샘플 데이터 대량 생성
```

---

---

# Seeding 이 필요한 이유

```txt
개발 시작할 때 DB 가 비어있으면:
  로그인 테스트 → 계정이 없음
  목록 조회 테스트 → 데이터가 없음
  관계 테스트 → 연결할 데이터가 없음

Seeding 으로 해결:
  관리자 계정 → 앱 시작 시 자동 생성 (Prisma seed)
  샘플 영화 / 장르 / 감독 → Postman Runner 로 대량 생성
```

---

---

# 방법 1 — Prisma seed 스크립트 ⭐️

```txt
prisma/seed.ts 파일에 초기 데이터 삽입 로직 작성
pnpm seed 명령어 하나로 실행

언제:
  관리자 계정처럼 필수 초기 데이터
  앱 배포 후 최초 1회 실행
  DB 초기화 후 다시 세팅할 때
```

## package.json 설정

```json
{
  "scripts": {
    "seed": "tsx prisma/seed.ts"
  },
  "prisma": {
    "seed": "tsx prisma/seed.ts"
  }
}
```

```txt
tsx vs ts-node:

ts-node + CommonJS 방식의 문제:
  Prisma 7 은 TypeScript ESM 규칙으로
  generated/prisma/client.ts 안에 .js 확장자로 import 를 씀
    import * as $Class from "./internal/class.js"
  실제 디스크에는 class.ts 만 있고 class.js 는 없음
  ts-node + CommonJS 로 돌리면 진짜 class.js 파일을 찾다가 에러

  Cannot find module './internal/class.js'
    ← generated/prisma/client.ts
    ← prisma/seed.ts

tsx 권장 이유:
  ESM + CJS 모두 처리 가능
  .js → .ts 경로 자동 해석
  Prisma 7 generated 파일과 호환

tsconfig-paths 가 필요한 경우:
  seed 에서 @/* 절대 경로 alias 사용 시
  "seed": "tsx --import tsconfig-paths/register prisma/seed.ts"

  지금처럼 ../src/config/env.keys 상대 경로로 쓰면
  tsconfig-paths 없이도 됨
```

## prisma/seed.ts 작성 ⭐

```typescript
import 'dotenv/config';
// ↑ .env 파일 로드
// seed 는 NestJS 앱 밖에서 실행 → ConfigModule / Joi 안 탐
// dotenv/config 로 직접 로드해야 process.env 에서 읽힘

import { PrismaClient, Role } from '../generated/prisma/client';
import { PrismaPg } from '@prisma/adapter-pg';
import { EnvKeys } from '../src/config/env.keys';
import * as bcrypt from 'bcrypt';

// Prisma 7 + driver adapter
// new PrismaClient() 만 쓰면 에러 → adapter 필수
const prisma = new PrismaClient({
  adapter: new PrismaPg({
    connectionString: process.env[EnvKeys.DATABASE_URL],
    //                             ↑ EnvKeys 상수 활용 (하드코딩 방지)
  }),
});

async function main() {
  const email    = process.env[EnvKeys.ADMIN_EMAIL];
  const password = process.env[EnvKeys.ADMIN_PASSWORD];

  if (!email || !password) {
    throw new Error(
      `${EnvKeys.ADMIN_EMAIL} / ${EnvKeys.ADMIN_PASSWORD} 가 .env 에 필요합니다.`,
    );
  }
// [[NestJS_Bcrypt]]
  const saltRounds   = 10;
  const passwordHash = await bcrypt.hash(password, saltRounds);

  await prisma.user.upsert({
    where:  { email },
    create: { email, passwordHash, role: Role.ADMIN },
    update: { passwordHash, role: Role.ADMIN },
  });
  //  ↑ 이미 있으면 update / 없으면 create [[NestJS_Prisma]]

  console.log(`Admin seed 완료: ${email}`);
}

main()
  .catch((e) => {
    console.error(e);
    process.exit(1); // 에러 시 프로세스 종료 (종료 코드 1 = 실패)
  })
  .finally(() => prisma.$disconnect());
  //                  ↑ 실행 후 반드시 연결 해제
```


```txt
왜 이렇게 작성하나:

import 'dotenv/config':
  seed 는 NestJS 앱 밖에서 실행됨
  pnpm seed → ts-node prisma/seed.ts → ConfigModule 안 탐
  Joi 유효성 검사도 안 돌아감
  → dotenv/config 로 직접 .env 로드 필수

Prisma 7 + PrismaPg adapter:
  Prisma 7 부터 new PrismaClient() 만 쓰면 에러
  PrismaService 와 동일하게 adapter 필수

  PrismaService:
    const adapter = new PrismaPg({ connectionString: configService.getOrThrow(...) });
    super({ adapter });

  seed.ts (NestJS 밖):
    new PrismaClient({ adapter: new PrismaPg({ connectionString: process.env[...] }) })
    ConfigService 대신 process.env 직접 사용

EnvKeys 상수 사용:
  process.env['DATABASE_URL'] 하드코딩 대신
  process.env[EnvKeys.DATABASE_URL] 으로 타입 안전하게 참조
  → 오탈자 방지 / 자동완성 지원
  src/config/env.keys.ts 를 seed 에서도 import 가능

upsert (create + update):
  이미 있으면 update / 없으면 create
  반복 실행해도 중복 에러 없음 (멱등성 보장)

main() 즉시 실행:
  async function 은 선언만으로 실행 안 됨
  main() 호출해야 실행됨

.catch((e) => { console.error(e); process.exit(1) }):
  에러 시 종료 코드 1 (실패) 로 종료
  CI/CD 에서 실패 감지 가능

.finally(() => prisma.$disconnect()):
  성공 / 실패 상관없이 항상 실행
  없으면 스크립트가 대기 상태로 걸릴 수 있음
```

## 실행

```bash
pnpm seed
# → ts-node -r tsconfig-paths/register prisma/seed.ts 실행
# → .env 의 ADMIN_EMAIL / ADMIN_PASSWORD 로 관리자 계정 생성
```

---

---

# 방법 2 — Postman Runner (샘플 대량 생성) ⭐️

```txt
Prisma seed 와 다른 목적:
  Prisma seed   → 필수 초기 데이터 (관리자 등)
  Postman Runner → 개발 중 테스트용 랜덤 샘플 대량 생성
                   영화 / 장르 / 감독 등 다양한 데이터
```

## Postman 랜덤 변수 ⭐️

```txt
{{$randomFullName}}       랜덤 이름 (John Doe)
{{$randomCountry}}        랜덤 나라 (Korea)
{{$randomNoun}}           랜덤 명사 (table / river)
{{$randomAdjective}}      랜덤 형용사 (beautiful / fast)
{{$randomAlphaNumeric}}   랜덤 영문숫자 (a3B9)
{{$randomLoremSentence}}  랜덤 문장
```

```json
// POST /director Body
{
  "name":        "{{$randomFullName}}",
  "dob":         "1999-07-30T00:00:00.000Z",
  "nationality": "{{$randomCountry}}"
}

// POST /genre Body
{
  "name": "{{$randomNoun}}"
}

// POST /movie Body
{
  "title":      "{{$randomNoun}} {{$randomAdjective}} {{$randomAlphaNumeric}}",
  "detail":     "{{$randomLoremSentence}}",
  "directorId": {{director}},
  "genreIds":   {{genres}}
}
```

## 생성된 ID 환경변수에 누적 ⭐️

```javascript
// POST /director → Tests 탭 스크립트
pm.test('Status code is 201', function () {
  pm.response.to.have.status(201);

  const body = pm.response.json();

  let directors = pm.environment.get('directorIds');

  if (!directors) {
    directors = body.id;
  } else {
    directors = directors + ',' + body.id;
  }

  pm.environment.set('directorIds', directors);
  // → "1,2,3,4,5" 형태로 누적
});
```

```javascript
// POST /genre → Tests 탭 스크립트
pm.test('Status code is 201', function () {
  pm.response.to.have.status(201);

  const body = pm.response.json();

  let genres = pm.environment.get('genreIds');

  if (!genres) {
    genres = body.id;
  } else {
    genres = genres + ',' + body.id;
  }

  pm.environment.set('genreIds', genres);
});
```

```txt
왜 누적하나:
  영화 생성 시 directorId / genreIds 가 필요
  Director / Genre 가 생성될 때마다 ID 를 쌓아둠
  나중에 영화 생성 시 여기서 랜덤으로 꺼내 씀
```

## lodash — 랜덤 선택 ⭐️

```javascript
// POST /movie → Pre-request Script 탭
const _ = require('lodash');
// Postman 에서 require('lodash') 바로 사용 가능

// directorIds → 랜덤으로 1개 선택
const directorIds      = pm.environment.get('directorIds').split(',');
const randomDirectorId = _.sample(directorIds);
pm.environment.set('director', randomDirectorId);

// genreIds → 랜덤으로 3개 선택 (중복 없이)
const genreIds      = pm.environment.get('genreIds').split(',');
const pickedGenreIds = [];
let failCount = 0;

while (pickedGenreIds.length < 3 && failCount < 20) {
  const randomGenreId = _.sample(genreIds);

  if (pickedGenreIds.includes(randomGenreId)) {
    failCount++;
    continue;
  }

  pickedGenreIds.push(randomGenreId);
}

pm.environment.set('genres', pickedGenreIds);
```

```txt
lodash 자주 쓰는 함수:
  _.sample(array)     배열에서 랜덤 1개 선택
  _.shuffle(array)    배열 랜덤 섞기
  _.uniq(array)       중복 제거
  _.chunk(array, n)   n 개씩 나누기
  _.range(1, 10)      1~9 배열 생성
```

## Postman Runner 실행

```txt
1. Postman 상단 Runner 탭 클릭
2. seed 폴더 선택
3. Iterations = 반복 횟수 (10 → 10번 실행)
4. Run 클릭

Iterations: 10 이면:
  Director 10개 / Genre 10개 / Movie 10개 생성
  매 실행마다 랜덤 데이터 → 다양한 샘플
```

## Seeding 폴더 구조

```txt
📁 seed (Postman 컬렉션 폴더)
  POST /auth/login     → accessToken 환경변수 저장
  POST /director       → directorIds 누적 (5회)
  POST /genre          → genreIds 누적 (5회)
  POST /movie          → Pre-request: 랜덤 director/genre 선택
```

---

---

# 두 방법 비교

```txt
Prisma seed (pnpm seed):
  목적    필수 초기 데이터 (관리자 계정 등)
  시점    앱 배포 / DB 초기화 후
  특징    upsert 로 멱등성 보장 / .env 값 사용
  실행    pnpm seed 또는 npx prisma db seed

Postman Runner:
  목적    개발용 랜덤 샘플 대량 생성
  시점    개발 중 테스트 데이터 필요할 때
  특징    랜덤 변수 / lodash 활용 / 반복 실행
  실행    Postman Runner 에서 수동 실행
```