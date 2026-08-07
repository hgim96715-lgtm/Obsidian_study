---
aliases:
  - Swagger
  - OpenAPI
  - 타입 생성
tags:
  - NextJS
  - NestJS
related:
  - "[[00_NestJS_Ecosystem_HomePage]]"
  - "[[00_JS_Ecosystem_HomePage]]"
  - "[[NestJS_Swagger]]"
  - "[[TS_Utility_Types]]"
---
# OpenAPI_Codegen — API 타입 자동 생성 파이프라인

>[!info]
>NestJS Swagger 스펙 → `dump-openapi.ts`로 `openapi.json` 추출 → `openapi-typescript`로 TypeScript 타입 생성 → 프론트에서 사용. 
>백엔드가 바뀌면 한 명령어(`gen:api`)로 프론트 타입이 자동 업데이트된다. 
>NestJS Swagger 설정 → [[NestJS_Swagger]]

---

# 왜 이 파이프라인이 필요한가 ⭐️⭐️⭐️⭐️

```txt
수동 방식 (문제):
  NestJS에서 API를 만들면
  → 프론트에서 직접 타입을 손으로 작성
  → 백엔드가 바뀌면 → 프론트 타입도 수동으로 수정
  → 실수하면 타입은 통과하는데 런타임에 에러

자동 방식 (해결):
  NestJS가 API 스펙을 자동 생성 (Swagger)
  → 스펙을 파일로 저장 (openapi.json)
  → 생성기가 파일을 읽어서 타입 자동 생성 (api.d.ts)
  → 프론트에서 그 타입을 import해서 사용

  백엔드 DTO 바뀜 → gen:api 한 번 실행 → 프론트 타입 자동 업데이트
```

---

# 전체 파이프라인 — 4단계 ⭐️⭐️⭐️⭐️

```txt
[1단계] NestJS 앱을 잠깐 실행해서 Swagger 스펙을 파일로 저장
           dump-openapi.ts 실행
                ↓
        openapi.json 생성  (apps/web/openapi/openapi.json)

[2단계] openapi.json → TypeScript 타입 파일 생성
           openapi-typescript 실행
                ↓
        api.d.ts 생성  (apps/web/src/types/api.d.ts)

[3단계] 프론트에서 생성된 타입을 import해서 사용
           import type { components } from '@/types/api';
           type ApiPost = components['schemas']['Post'];

[4단계] 생성 타입이 안 맞으면 Omit & 확장으로 조정
           export type ApiComment = Omit<Schemas['CommentResponseDto'], 'author'> & {
             author: ApiAuthor;
           };
```

---

# 1단계 — dump-openapi.ts ⭐️⭐️⭐️⭐️

```txt
문제:
  openapi.json을 만들려면 NestJS 서버의 Swagger 스펙이 필요
  하지만 서버를 항상 실행 중일 수는 없음
  (CI/CD, 처음 세팅 등에서 서버 없이 타입을 생성해야 함)

해결:
  dump-openapi.ts = "NestJS 앱을 잠깐 켜고, 스펙을 파일로 저장하고, 앱을 끄는" 스크립트
```

```typescript
// apps/api/src/scripts/dump-openapi.ts
/**
 * Nest AppModule을 띄워 Swagger document → apps/web/openapi/openapi.json
 *
 *   pnpm --filter api dump:openapi
 *   (루트) pnpm gen:api
 *
 * apps/api/.env DATABASE_URL 필요 (Prisma $connect).
 */
import 'dotenv/config';
import { mkdir, writeFile } from 'node:fs/promises';
import path from 'node:path';
import { NestFactory } from '@nestjs/core';
import { DocumentBuilder, SwaggerModule } from '@nestjs/swagger';
import { AppModule } from '../app.module';

async function main() {
  // ① NestJS 앱 생성 (HTTP 서버는 열지 않음, 스펙 추출용)
  const app = await NestFactory.create(AppModule, {
    logger: ['error', 'warn'],  // 불필요한 로그 최소화
  });

  // ② Swagger 스펙 설정 (main.ts의 것과 동일하게)
  const config = new DocumentBuilder()
    .setTitle('Music Community API')
    .setVersion('0.0.1')
    .addBearerAuth({ ... }, 'access-token')
    .build();

  // ③ Swagger 문서(JSON 객체) 생성
  const document = SwaggerModule.createDocument(app, config);

  // ④ openapi.json 파일로 저장
  const outDir  = path.resolve(process.cwd(), '../web/openapi');
  const outFile = path.join(outDir, 'openapi.json');
  await mkdir(outDir, { recursive: true });
  await writeFile(outFile, `${JSON.stringify(document, null, 2)}\n`, 'utf8');

  console.log(`wrote ${outFile}`);

  // ⑤ 앱 종료 (HTTP 서버 열지 않았으므로 그냥 닫음)
  await app.close();
}

main().catch((err) => {
  console.error(err);
  process.exit(1);
});
```

```txt
이 스크립트가 하는 것 순서:
  ① NestJS 앱을 메모리에서 실행 (HTTP 포트는 열지 않음)
  ② Swagger 설정으로 문서 JSON 객체 생성
  ③ JSON을 openapi.json 파일로 저장
  ④ 앱 종료

  실제 서버(listen)를 열지 않기 때문에:
  DB 연결은 하지만 HTTP 요청을 받지는 않음
  빠르게 실행하고 종료 (보통 5~10초)
  → CI/CD나 개발 세팅 시 서버 없이 스펙 파일 생성 가능
```

---

# 2단계 — openapi.json ⭐️⭐️⭐️⭐️

```txt
openapi.json = NestJS가 생성한 API 스펙 파일
  모든 엔드포인트 목록
  각 엔드포인트의 요청/응답 형태
  DTO 스키마 정보 (@ApiProperty가 있는 필드들)

apps/
├── api/
│   └── src/scripts/dump-openapi.ts  ← 이 스크립트가 생성
└── web/
    └── openapi/
        └── openapi.json             ← 생성된 파일 (커밋 포함 또는 gitignore)
```

```json
// openapi.json 일부 예시
{
  "openapi": "3.0.0",
  "components": {
    "schemas": {
      "PostResponseDto": {
        "type": "object",
        "properties": {
          "id":        { "type": "string" },
          "title":     { "type": "string" },
          "content":   { "type": "string", "nullable": true },
          "createdAt": { "type": "string" }
        }
      },
      "CommentResponseDto": { ... }
    }
  },
  "paths": {
    "/posts": {
      "get": { "summary": "게시글 목록", ... }
    }
  }
}
```

```txt
openapi.json을 만들려면 @ApiProperty가 있어야 함:
  DTO에서 @ApiProperty를 빠뜨린 필드 → 스펙에서 누락 → api.d.ts에서도 누락
  → NestJS_Swagger 에서 @ApiProperty를 모든 필드에 달아야 완전한 스펙 생성
```

---

# 3단계 — openapi-typescript로 타입 생성 ⭐️⭐️⭐️⭐️

```bash
# 설치
pnpm add -D openapi-typescript

# openapi.json → api.d.ts 생성
npx openapi-typescript apps/web/openapi/openapi.json -o apps/web/src/types/api.d.ts
```

```txt
openapi-typescript가 하는 것:
  openapi.json (JSON 스펙)을 읽어서
  TypeScript 타입 파일(.d.ts)을 자동으로 생성

.d.ts 파일이란:
  TypeScript 타입 선언만 담은 파일
  실제 JavaScript 코드(함수, 클래스)는 없음
  타입 정보만 — 번들에 포함 안 됨

생성되는 파일 위치:
  apps/web/src/types/api.d.ts
```

---

# 4단계 — api.d.ts (생성된 타입 파일) ⭐️⭐️⭐️⭐️

```typescript
// apps/web/src/types/api.d.ts (자동 생성 — 직접 수정 금지)
export interface components {
  schemas: {
    PostResponseDto: {
      id:        string;
      title:     string;
      content:   string | null;
      createdAt: string;
      author:    components['schemas']['UserSummaryDto'];
    };
    CommentResponseDto: {
      id:       string;
      body:     string | null;
      parentId: string | null;
      deletedAt: string | null;
      author:   components['schemas']['UserSummaryDto'];  // ← 자동 생성 타입
    };
    UserSummaryDto: {
      id:       string;
      nickname: string;
    };
  };
}
```

```txt
자동 생성 파일 규칙:
  직접 수정하면 안 됨 — gen:api 실행 시 덮어씌워짐
  git에 커밋할 수도 있고, gitignore할 수도 있음
    커밋: 생성 없이 바로 사용 가능 (CI 빠름)
    gitignore: 항상 최신 백엔드에서 생성해야 함
```

---

# 5단계 — 생성 타입 사용 + Omit으로 조정 ⭐️⭐️⭐️⭐️

## 기본 사용

```typescript
// apps/web/src/types/apiTypes.ts
import type { components } from '@/types/api';

// 편의를 위해 짧은 이름으로 alias
type Schemas = components['schemas'];

// 그대로 사용
export type ApiPost        = Schemas['PostResponseDto'];
export type ApiUser        = Schemas['UserSummaryDto'];
```

## 생성 타입이 안 맞을 때 — Omit & 확장

```typescript
// CommentResponseDto의 author 타입이 UserSummaryDto인데
// 프론트에서는 ApiAuthor라는 다른 타입을 써야 할 때

export type ApiComment = Omit<
  Schemas['CommentResponseDto'],
  'author' | 'parentId' | 'deletedAt'  // ← 교체할 필드들을 제거
> & {
  parentId:  string | null;   // ← 원하는 타입으로 다시 추가
  deletedAt: string | null;
  author:    ApiAuthor;        // ← 다른 타입으로 교체
};
```

```txt
Omit & 확장 패턴이 필요한 경우:
  생성된 타입에서 일부 필드의 타입이 원하는 것과 다를 때
  nullable 처리가 안 됐을 때 (string | null 이어야 하는데 string으로 나올 때)
  중첩 타입을 다른 타입으로 교체하고 싶을 때

  Omit<원본, '제거할 키'> & { 제거한 키: 원하는 타입 }
  = 원본에서 특정 필드를 제거하고 원하는 타입으로 다시 붙임
```

## 실전 — apiTypes.ts 전체 패턴

```typescript
// apps/web/src/types/apiTypes.ts
import type { components } from '@/types/api';

type Schemas = components['schemas'];

// 1. 그대로 쓸 수 있는 타입
export type ApiPost    = Schemas['PostResponseDto'];
export type ApiAuthor  = Schemas['UserSummaryDto'];

// 2. 수정이 필요한 타입 — Omit & 확장
export type ApiComment = Omit<
  Schemas['CommentResponseDto'],
  'author' | 'parentId' | 'deletedAt'
> & {
  parentId:  string | null;
  deletedAt: string | null;
  author:    ApiAuthor;
};

export type ApiPublicUser = Omit<
  Schemas['PublicUserDto'],
  'image' | 'bio'
> & {
  image: string | null;
  bio:   string | null;
};

// 3. 기존에 수동으로 만들었던 타입 — alias로 교체
export type WithdrawResult = {
  success: boolean;
  message: string;
};
```

---

# package.json scripts 설정 ⭐️⭐️⭐️

```json
// apps/api/package.json
{
  "scripts": {
    "dump:openapi": "ts-node src/scripts/dump-openapi.ts"
  }
}

// apps/web/package.json
{
  "scripts": {
    "gen:types": "openapi-typescript ../openapi/openapi.json -o src/types/api.d.ts"
  }
}

// 루트 package.json
{
  "scripts": {
    "gen:api": "pnpm --filter api dump:openapi && pnpm --filter web gen:types"
    //          ↑ api 스크립트 실행                    ↑ web 스크립트 실행
    //          (dump:openapi)                         (gen:types)
  }
}
```

```txt
pnpm gen:api 실행 순서:
  ① pnpm --filter api dump:openapi
     apps/api의 dump-openapi.ts 실행
     → apps/web/openapi/openapi.json 생성

  ② pnpm --filter web gen:types
     openapi.json → apps/web/src/types/api.d.ts 생성

  한 명령어로 전체 파이프라인 실행
```

---

# 언제 gen:api를 실행하는가

```txt
실행해야 하는 경우:
  NestJS DTO가 추가되거나 변경될 때
  @ApiProperty 데코레이터가 추가/수정될 때
  새 API 엔드포인트가 생겼을 때
  처음 프로젝트를 클론해서 세팅할 때

실행 안 해도 되는 경우:
  백엔드 변경 없이 프론트만 작업할 때
  이미 최신 api.d.ts가 있을 때 (커밋된 경우)

자동화:
  CI에서 백엔드 배포 후 gen:api 실행 → api.d.ts 커밋
  또는 백엔드 변경 PR에서 자동으로 gen:api 실행
```