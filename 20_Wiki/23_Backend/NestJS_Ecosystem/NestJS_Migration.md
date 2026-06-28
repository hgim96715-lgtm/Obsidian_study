---
aliases: [마이그레이션, Migration, typeorm migration]
tags:
  - NestJS
related:
  - "[[00_NestJS_Ecosystem_HomePage]]"
  - "[[NestJS_Transaction]]"
  - "[[NestJS_TypeORM_Entity]]"
  - "[[NestJS_TypeORM]]"
---
# NestJS_Migration — 마이그레이션

```txt
Migration = DB 스키마 변경사항을 버전 관리하는 방법
synchronize: true 대신 운영 환경에서 사용
up() 으로 적용 / down() 으로 롤백
```

---

---

# synchronize vs Migration ⭐️

```txt
synchronize: true
  Entity 변경 → DB 자동 반영
  개발 환경에서 편리
  ⚠️ 운영에서 위험 → 컬럼 삭제 시 데이터 손실
  ⚠️ 언제 어떻게 바뀌었는지 추적 불가

Migration
  스키마 변경을 스크립트로 직접 작성
  up() 적용 / down() 되돌리기
  변경 히스토리 버전 관리 가능
  개발/스테이징/운영 환경 스키마 동일하게 유지 가능
```

```typescript
// app.module.ts — 환경에 따라 synchronize 자동 전환 ⭐️
synchronize: configService.get<string>(envVariableKeys.env) === 'prod' ? false : true,
//           prod 환경에서는 false → Migration 으로 관리
//           dev 환경에서는 true  → 개발 편의
```

---

---

# 폴더 구조

```txt
src/
└── database/
    ├── data-source.ts       ← TypeORM CLI 용 DataSource 설정
    └── migrations/
        ├── 1780382470929-init.ts
        └── 1780383028247-test.ts
```

---

---

# 설치

```bash
# dotenv — .env 파일 로드
pnpm i dotenv

# typeorm CLI — migration 명령어 실행
pnpm i -g typeorm
```

---

---

# data-source.ts 설정 ⭐️

```txt
app.module.ts 의 TypeORM 설정과 별도로
TypeORM CLI 전용 DataSource 파일을 따로 만들어야 함
→ migration:generate / migration:run 이 이 파일을 참조
```

```typescript
// src/database/data-source.ts
import * as dotenv from 'dotenv';
import { DataSource } from 'typeorm';

dotenv.config();   // .env 파일 로드

export default new DataSource({
  type:     process.env.DB_TYPE as 'postgres',
  host:     process.env.DB_HOST,
  port:     Number(process.env.DB_PORT || 5432),
  username: process.env.DB_USER,
  password: process.env.DB_PASSWORD,
  database: process.env.DB_DATABASE,
  logging:  false,
  synchronize: false,   // CLI 용이므로 반드시 false
  entities:   [__dirname + '/../**/*.entity{.ts,.js}'],
  migrations: [__dirname + '/../database/migrations/*.{ts,js}'],
});
```

```txt
⚠️ CLI 명령어는 dist/ 의 js 파일을 참조
   → 명령어 실행 전 반드시 pnpm build 필요
   → -d dist/database/data-source.js 경로 사용
```

---

---

# Migration 파일 구조 — up / down

```typescript
// src/database/migrations/1780383028247-test.ts
import { MigrationInterface, QueryRunner } from 'typeorm';

export class Test1780383028247 implements MigrationInterface {

  // up(): 마이그레이션 적용
  async up(queryRunner: QueryRunner): Promise<void> {
    await queryRunner.query(`CREATE TABLE "test"(id SERIAL)`);
  }

  // down(): 마이그레이션 되돌리기 (up 의 반대)
  async down(queryRunner: QueryRunner): Promise<void> {
    await queryRunner.query(`DROP TABLE "test"`);
  }
}
```

```txt
파일명 규칙:
  타임스탬프-이름.ts
  1780383028247-test.ts

  타임스탬프로 실행 순서 보장
  migrations 테이블에 timestamp / name 으로 기록됨
```

---

---

# Migration CLI 명령어 ⭐️

## 빈 파일 생성 — 직접 작성할 때

```bash
typeorm migration:create ./src/database/migrations/이름
# → 1780381775278-이름.ts 생성 (up/down 비어있음)
# 직접 SQL 작성
```

## 자동 생성 — Entity 기반 ⭐️

```bash
# 1. 먼저 빌드 (dist/ 생성)
pnpm build

# 2. Entity 와 현재 DB 스키마 비교해서 자동 생성
typeorm migration:generate src/database/migrations/init -d dist/database/data-source.js
# → 1780382470929-init.ts 생성
```

```txt
⚠️ DataGrip 에서 테이블이 이미 있으면 생성 안 됨
   No changes in database schema were found 에러 발생
   → DB 에서 테이블 DROP 후 다시 실행

⚠️ -d 뒤는 dist/ 의 js 파일
   src/ 의 ts 파일 경로 쓰면 안 됨
   → 반드시 pnpm build 먼저
```

## 실행 — up() 적용

```bash
pnpm build
typeorm migration:run -d ./dist/database/data-source.js
```

```txt
실행 후 DB 에 migrations 테이블 자동 생성
  timestamp  실행한 마이그레이션 타임스탬프
  name       마이그레이션 클래스 이름

migrations 테이블로 어떤 마이그레이션이 실행됐는지 추적
```

## 되돌리기 — down() 실행

```bash
typeorm migration:revert -d ./dist/database/data-source.js
# → 마지막 실행한 마이그레이션의 down() 실행
# → 순서대로 하나씩 되돌림
```

### 상태 확인

```bash
typeorm migration:show -d ./dist/database/data-source.js
```

---

---

# 전체 흐름 ⭐️

```txt
1. Migration 파일 생성
     빈 파일:   typeorm migration:create ./src/database/migrations/이름
     자동 생성: pnpm build
               typeorm migration:generate src/database/migrations/이름 -d dist/database/data-source.js

2. up() / down() 작성 (자동 생성이면 이미 작성됨)

3. 빌드
     pnpm build

4. 실행
     typeorm migration:run -d ./dist/database/data-source.js

5. 되돌리기 (필요시)
     typeorm migration:revert -d ./dist/database/data-source.js
```

---

---

# ⚠️ 트러블슈팅

### No changes in database schema were found

```txt
원인:
  DB 에 이미 테이블이 존재
  → Entity 와 DB 스키마가 동일하다고 판단 → 생성 안 함

해결:
  DataGrip 에서 테이블 전체 DROP 후 재실행
  또는 typeorm migration:create 로 빈 파일 만들어서 직접 작성
```

### 중복 마이그레이션 충돌

```txt
원인:
  migrations 폴더에 같은 내용의 파일이 2개 이상 있음
  두 번째 마이그레이션이 첫 번째가 만든 테이블을 또 만들려고 해서 충돌

해결:
  1. DB 초기화
     typeorm query "DROP SCHEMA public CASCADE; CREATE SCHEMA public;" \
       -d ./dist/database/data-source.js

  2. dist 삭제 후 재빌드 (중복 js 파일 제거)
     rm -rf dist
     pnpm build

  3. dist/database/migrations 에 파일 1개만 남아있는지 확인
     ls -la dist/database/migrations

  4. 재실행
     typeorm migration:run -d ./dist/database/data-source.js
```

### -d 경로 오류

```txt
원인:
  src/ 의 .ts 파일 경로를 넣음
  TypeORM CLI 는 컴파일된 .js 파일을 읽어야 함

해결:
  pnpm build → dist/ 생성 후
  -d dist/database/data-source.js 경로 사용
```

---

---

# package.json scripts 등록

```json
{
  "scripts": {
    "migration:create":   "typeorm migration:create ./src/database/migrations/",
    "migration:generate": "pnpm build && typeorm migration:generate src/database/migrations/init -d dist/database/data-source.js",
    "migration:run":      "pnpm build && typeorm migration:run -d ./dist/database/data-source.js",
    "migration:revert":   "pnpm build && typeorm migration:revert -d ./dist/database/data-source.js",
    "migration:show":     "typeorm migration:show -d ./dist/database/data-source.js"
  }
}
```

---

---

# 주요 작업 패턴

### 테이블 생성

```typescript
async up(queryRunner: QueryRunner) {
  await queryRunner.query(`
    CREATE TABLE movie (
      id    SERIAL PRIMARY KEY,
      title VARCHAR(100) NOT NULL
    )
  `);
}
async down(queryRunner: QueryRunner) {
  await queryRunner.query(`DROP TABLE movie`);
}
```

### 컬럼 추가

```typescript
async up(queryRunner: QueryRunner) {
  await queryRunner.query(`ALTER TABLE movie ADD COLUMN rating FLOAT`);
}
async down(queryRunner: QueryRunner) {
  await queryRunner.query(`ALTER TABLE movie DROP COLUMN rating`);
}
```

### 컬럼 이름 변경

```typescript
async up(queryRunner: QueryRunner) {
  await queryRunner.query(`ALTER TABLE movie RENAME COLUMN genre TO category`);
}
async down(queryRunner: QueryRunner) {
  await queryRunner.query(`ALTER TABLE movie RENAME COLUMN category TO genre`);
}
```

### FK 추가

```typescript
async up(queryRunner: QueryRunner) {
  await queryRunner.query(`
    ALTER TABLE movie
    ADD COLUMN director_id INTEGER,
    ADD CONSTRAINT FK_movie_director
      FOREIGN KEY (director_id) REFERENCES director(id)
      ON DELETE SET NULL
  `);
}
async down(queryRunner: QueryRunner) {
  await queryRunner.query(`ALTER TABLE movie DROP CONSTRAINT FK_movie_director`);
  await queryRunner.query(`ALTER TABLE movie DROP COLUMN director_id`);
}
```

---

---

# 명령어 한눈에

|명령어|역할|
|---|---|
|`migration:create`|빈 파일 생성 (직접 작성)|
|`migration:generate`|Entity 기반 자동 생성|
|`migration:run`|up() 실행|
|`migration:revert`|down() 실행 (마지막 것부터)|
|`migration:show`|실행 상태 확인|