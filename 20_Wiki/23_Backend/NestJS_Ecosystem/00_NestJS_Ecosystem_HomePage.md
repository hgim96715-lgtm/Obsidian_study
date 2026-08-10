---
aliases: [00_NestJS_Ecosystem_HomePage — NestJS · NodeJS, NestJS Ecosystem HomePage]
tags: [HomePage]
related:
  - "[[00_DB_HomePage]]"
  - "[[00_Deployment_HomePage]]"
  - "[[00_DevOps_HomePage]]"
  - "[[00_JS_Ecosystem_HomePage]]"
  - "[[00_Project_HomePage]]"
cssclasses:
  - max
  - table-max
  - table-wrap
---
# 00_NestJS_Ecosystem_HomePage — NestJS · NodeJS

>[!info]
> NestJS는 NodeJS 위에 얹힌 프레임워크.
> 이 홈은 **만드는 순서**로 정리한다. 주제만 찾을 때는 아래 [[#빠른 찾기]]로.
> JS/TS 범용 문법은 → [[00_JS_Ecosystem_HomePage]]

```mermaid-beautiful
flowchart LR
  M1["1 모노레포"] --> M2["2 개념·main"]
  M2 --> M3["3 Env·CORS·Swagger"]
  M3 --> M4["4 Prisma"]
  M4 --> M5["5 CRUD·DTO"]
  M5 --> M6["6 인증·JWT·Roles"]
  M6 --> M7["7 이후 심화"]
```

```txt
큰 틀: 번호 순으로 쌓는다. 한 번에 전부 보지 말고 단계마다 노트만 연다.
```

---

# 빠른 찾기

| 찾을 때 | 섹션 |
| --- | --- |
| 개념 · main.ts · Module/DI | [[#1. 기초 · 부트스트랩]] |
| 모노레포 | [[#0. 모노레포]] |
| Env · CORS · Swagger | [[#2. 설정 · HTTP 입구 · 문서]] |
| Prisma · 마이그레이션 | [[#3. 데이터베이스 · Prisma]] |
| Controller · DTO · Pipe | [[#4. CRUD · 요청 레이어]] |
| Auth · JWT · Roles · Optional | [[#5. 인증 · JWT · Roles]] |
| WebSocket · 이메일 · 배포 등 | [[#6. 이후에 붙이는 것]] |

---

## 0. 모노레포

| | 노트 |
| --- | --- |
| **모노레포** | [[Monorepo_PNPM]] |

```txt
Monorepo_PNPM  pnpm workspace · apps/api · apps/web · filter 스크립트
```

---

## 1. 기초 · 부트스트랩

`main.ts` · Module/Controller/Service · DI · CLI — **맨 앞에서** 본다.

|                  | 노트                          |
| ---------------- | --------------------------- |
| **개념 · 지도**      | [[NestJS_Concept]]          |
| **모듈**           | [[NestJS_Module]]           |
| **Service · DI** | [[NestJS_Service_Provider]] |

```txt
NestJS_Concept           Nest란 · 요청 흐름 · DI · 설치 · CLI · main.ts 지도
                         → ConfigService·EnvKeys는 NestJS_Env_Config로
NestJS_Module            imports/exports · @Global · forwardRef
NestJS_Service_Provider  @Injectable · DI · useFactory/useClass
```

---

## 2. 설정 · HTTP 입구 · 문서

`main.ts`에 CORS · ValidationPipe · Swagger · Config를 얹는 구간.

| | 노트 |
| --- | --- |
| **환경변수** | [[NestJS_Env_Config]] |
| **CORS** | [[NestJS_CORS]] |
| **HTTP 개념** | [[HTTP_Concept]] |
| **Swagger** | [[NestJS_Swagger]] · [[NestJS_Versioning]] |
| **타입 생성** | [[OpenAPI_Codegen]] |

```txt
NestJS_Env_Config  EnvKeys · Joi · ConfigModule · getOrThrow
NestJS_CORS        origin · credentials · FRONTEND_URL
HTTP_Concept       요청/응답 · Authorization · Cookie · HTTPS
NestJS_Swagger     DocumentBuilder · @ApiTags · @ApiBearerAuth · addBearerAuth
OpenAPI_Codegen    /api-json → 프론트 타입
```

---

## 3. 데이터베이스 · Prisma

| | 노트 |
| --- | --- |
| **Prisma** | [[NestJS_Prisma]] · [[NestJS_Prisma_Patterns]] |
| **마이그레이션** | [[NestJS_Migration]] |
| **트랜잭션** | [[NestJS_Transaction]] · [[PG_Transaction]] |
| **PostgreSQL** | [[NestJS_PostgreSQL]] · [[PG_Patterns]] |
| **통계** | [[NestJS_StatsBucket]] |
| **DB 전체** | [[00_DB_HomePage]] |

```txt
NestJS_Prisma           CRUD · where · select/include · PrismaService
NestJS_Prisma_Patterns  조건부 where · 관계 필터 · 토글 · 커서
NestJS_Migration        migrate dev/deploy/reset · seed 루프
```

---

## 4. CRUD · 요청 레이어

리소스 단위: Module ↔ Controller ↔ Service ↔ DTO.

| | 노트 |
| --- | --- |
| **Controller** | [[NestJS_Controller]] |
| **DTO** | [[NestJS_DTO]] |
| **Pipe** | [[NestJS_Pipe]] |

```txt
NestJS_Controller  @Get/@Post… · @Body/@Param · @Req/@Res · CRUD · Guard 적용 위치
NestJS_DTO         class-validator · PartialType · @ApiProperty
NestJS_Pipe        ValidationPipe · ParseUUIDPipe
→ @Req = Request 읽기(영향 적음) · @Res = Response 직접(기본 비추천, passthrough)
→ 프론트 타입 [[NextJS_Types]] · 문서 [[NestJS_Swagger]]
```

---

## 5. 인증 · JWT · Roles

개념 → 발급(login) → Guard(검증·Public·UserId) → Roles·Optional까지 **한 축**.

| | 노트 |
| --- | --- |
| **개념** | [[Auth_Concept]] |
| **발급 · AuthModule** | [[NestJS_Auth]] |
| **Guard · 데코레이터** | [[NestJS_JwtGuard]] |

```txt
Auth_Concept     인증 vs 인가 · Session vs Token · JWT · OAuth
NestJS_Auth      JwtModule · register/login · bcrypt · 토큰 발급
NestJS_JwtGuard  JwtAuthGuard · @Public · @UserId · APP_GUARD
                 @Roles · RolesGuard · OptionalJwt · @OptionalUserId
                 단일 role vs roles[] · APP_GUARD vs @UseGuards
```

---

## 6. 이후에 붙이는 것

필수 스택(0~5) 위에 올리는 기능. 필요할 때만.

### 실시간 · WebSocket

| | 노트 |
| --- | --- |
| **NestJS** | [[NestJS_WebSocket]] |
| **Next.js** | [[NextJS_WebSocket]] (→ JS_Ecosystem 폴더) |
| **패턴** | [[WebSocket_Patterns]] |

```txt
Gateway · 룸 · REST+WS 브로드캐스트 · socket.io-client
```

### 패턴 · 기법

| | 노트 |
| --- | --- |
| **이메일** | [[NestJS_Email]] |
| **스케줄링** | [[NestJS_Scheduling]] |
| **스로틀링** | [[NestJS_Throttle]] |
| **로깅** | [[NestJS_Logger]] |
| **페이지네이션** | [[NestJS_Pagination]] |
| **멱등성** | [[NestJS_Idempotency]] |
| **시드** | [[NestJS_Seed]] |

### 배포

| | 노트 |
| --- | --- |
| **배포** | [[00_Deployment_HomePage]] |

---

```txt
폴더 합친 이유:
  NestJS와 NodeJS가 얽히는 지점은 인증(Passport)과 HTTP 요청 레이어
  분류는 접두사(NestJS_ / NodeJS_)가 이미 함
홈 축을 만드는 순서로 둔 이유:
  main.ts·Module이 주제별 분류에선 맨 아래로 밀려 학습 순서와 어긋남
```
