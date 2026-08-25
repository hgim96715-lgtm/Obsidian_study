---
aliases:
  - 00_NestJS_Ecosystem_HomePage — NestJS · NodeJS
  - NestJS Ecosystem HomePage
tags:
  - HomePage
related:
  - "[[00_DB_HomePage]]"
  - "[[00_Deployment_HomePage]]"
  - "[[00_DevOps_Ecosystem_HomePage]]"
  - "[[00_JS_Ecosystem_HomePage]]"
  - "[[00_Project_HomePage]]"
cssclasses:
  - max
  - table-max
  - table-wrap
---
# 00_NestJS_Ecosystem_HomePage — NestJS · NodeJS

>[!info]
>NestJS = Express 위에 TypeScript·OOP 구조를 얹은 프레임워크.
>Module·Controller·Service 계층과 DI로 앱을 조립한다.
>JS/TS 범용 문법은 → [[00_JS_Ecosystem_HomePage]]

---

# 빠른 찾기

| 찾을 때 | 섹션 |
|---|---|
| 기초 개념 · main.ts | [[#📦 기초 개념]] |
| 인증 / JWT / Guard | [[#🔐 인증 · JWT]] |
| HTTP · Controller | [[#🌐 HTTP 요청 · 응답]] |
| 데이터베이스 / Prisma | [[#🗄️ 데이터베이스]] |
| 모듈 / DI | [[#🧩 모듈 · DI]] |
| DTO / 유효성 검사 | [[#📋 요청 · 응답 처리]] |
| 환경변수 / 설정 / 보안 | [[#🚀 설정 · 보안 · 배포]] |
| 이메일 / 스케줄링 등 | [[#⚙️ 패턴 · 기법]] |
| WebSocket | [[#📡 실시간 · WebSocket]] |
| API 문서화 · 버전 | [[#📄 API 문서화]] |

---

# NestJS 요청 처리 흐름

```txt
클라이언트 요청
  → Middleware          (로깅·인증 전처리)
  → Guard               (인증·인가 — JwtAuthGuard · RolesGuard)
  → Interceptor (before) (요청 변환·로깅)
  → Pipe                (유효성 검사·타입 변환 — ValidationPipe)
  → Controller          (@Get·@Post — 라우팅·파라미터 추출)
  → Service             (비즈니스 로직)
  → Repository/Prisma   (DB 접근)
  ← Interceptor (after) (응답 변환·에러 처리)
  ← ExceptionFilter     (전역 에러 포맷)
```

```mermaid-beautiful
flowchart LR
  A[개념·DI] --> B[Controller]
  B --> C[Module·DI]
  C --> D[데이터베이스]
  D --> E[인증·Guard]
  E --> F[패턴·기법]
  F --> G[배포]
```

---

## 📦 기초 개념

| 노트 | 내용 |
|---|---|
| [[NestJS_Concept]] | NestJS란 · Module/Controller/Service · 요청 처리 순서 · DI · 설치 · CLI · main.ts 전역 설정 · NestExpressApplication |
| [[HTTP_Concept]] | HTTP 요청/응답 · REST CRUD 매핑 · 멱등성 · 헤더(Authorization·Content-Type) · curl |

```txt
NestJS_Concept  NestJS = Express 래퍼 · 데코레이터 기반 OOP · Module/Controller/Service 3계층
                DI(의존성 주입) 개념 · CLI(nest g) · main.ts(ValidationPipe·CORS·prefix 전역 설정)
                NestExpressApplication vs NestApplication
HTTP_Concept    HTTP 메서드(GET·POST·PUT·PATCH·DELETE) · 멱등성 · 상태코드
                REST CRUD 매핑 · Authorization 헤더 · Content-Type · curl 테스트
```

---

## 🔐 인증 · JWT

| | 노트 |
|---|---|
| **구현** | [[NestJS_Auth]] · [[NestJS_JwtGuard]] |
| **개념** | [[Auth_Concept]] |

```txt
Auth_Concept    인증 vs 인가 · Session vs Token · JWT 구조(header.payload.sig) · Access+Refresh Token · OAuth
NestJS_Auth     JwtModule · AuthService · login/register · bcrypt 설치 · buildAuthResponse 헬퍼
NestJS_JwtGuard 메타데이터·Reflector · Guard · @Public · @UserId · @OptionalUserId
                OptionalJwtAuthGuard(ConfigService+JwtService+AuthService) · @Roles 단일 role
                APP_GUARD(전역) vs @UseGuards(로컬) 언제 뭘 쓰는가
```

---

## 🌐 HTTP 요청 · 응답

| | 노트 |
|---|---|
| **NestJS** | [[NestJS_Controller]] · [[NestJS_CORS]] |

```txt
NestJS_Controller  @Get/@Post/@Patch/@Delete · @Param/@Query/@Body · @Req(express에서 import)
                   @Res(passthrough) · @HttpCode · @Header · CRUD 패턴
                   외부 시스템 cron 엔드포인트 — x-cron-secret + @Public() + @UseGuards(CronSecretGuard)
NestJS_CORS        app.enableCors() · frontendOrigin 삼항 패턴 · credentials
```

---

## 🗄️ 데이터베이스

|                | 노트                                                |
| -------------- | ------------------------------------------------- |
| **연결**         | [[NestJS_PostgreSQL]]                             |
| **Prisma**     | [[NestJS_Prisma]]                                 |
| **마이그레이션**     | [[NestJS_Migration]]                              |
| **트랜잭션**       | [[NestJS_Transaction]]                            |
| **PostgreSQL** | [[PG_Transaction]] · [[PG_DML]] · [[PG_Patterns]] |
| **통계**         | [[NestJS_StatsBucket]]                            |
| **DB 전체**      | [[00_DB_HomePage]]                                |

```txt
NestJS_PostgreSQL  DB 연결 방법 선택 · DATABASE_URL vs POSTGRES_* 관계
                   Docker Compose(env_file·healthcheck·5444포트) · Dockerfile(corepack·prisma generate 더미URL)
                   PrismaService($connect/$disconnect) · PrismaModule(@Global) · DataGrip
                   migrate → generate 워크플로우 · 용어 정리표
NestJS_Prisma      schema.prisma 기본구조(Prisma 7: moduleFormat cjs, output) · @map/@@map
                   findMany/findUnique · where · select/include · $queryRaw(SELECT 1 헬스체크)
                   PrismaExceptionFilter(P2002) · 방법1(try/catch) vs 방법2(ExceptionFilter)
NestJS_Migration   migrate dev/deploy/reset · seed · Railway 배포 시 migrate deploy 적용 순서
NestJS_Transaction $transaction 패턴
```

---

## 🧩 모듈 · DI

| | 노트 |
|---|---|
| **NestJS** | [[NestJS_Module]] · [[NestJS_Service_Provider]] |

```txt
NestJS_Module           providers/imports/exports 역할 · @Global · forwardRef(순환 참조)
                        SharedModule 패턴 · forRootAsync(환경변수 동적 설정)
NestJS_Service_Provider Service · @Injectable · DI · useValue/useFactory/useClass
```

---

## 📋 요청 · 응답 처리

| | 노트 |
|---|---|
| **DTO · 유효성** | [[NestJS_DTO]] · [[NestJS_Pipe]] |

```txt
NestJS_DTO   class-validator · @IsIn vs @IsEnum · @ApiProperty(format·enum) · Response DTO
             @IsOptional 위치 · @ValidateIf 조건부 · PartialType/OmitType
             strictPropertyInitialization: false 이유 → [[TS_TsConfig]]
NestJS_Pipe  ValidationPipe(transform·whitelist·forbidNonWhitelisted) · ParseUUIDPipe
```

---

## 🚀 설정 · 보안 · 배포

| | 노트 |
|---|---|
| **환경변수** | [[NestJS_Env_Config]] |
| **CORS** | [[NestJS_CORS]] |
| **모노레포** | [[Monorepo_PNPM]] |
| **배포** | [[00_Deployment_HomePage]] |

```txt
NestJS_Env_Config  환경변수란 · .env 파일 · ConfigModule(isGlobal) · ConfigService.getOrThrow
                   EnvKeys 상수(오타 방지) · Joi validationSchema · validationOptions convert:true
                   forRootAsync(환경변수로 JwtModule 설정) · main.ts에서 app.get(ConfigService)
Monorepo_PNPM      초기 설정 순서 · pnpm-workspace.yaml · allowBuilds · ERR_PNPM_UNEXPECTED_STORE
                   "type":"commonjs" · store-dir=.pnpm-store · Next.js 포트(-p 3051) · shared 패키지
배포 상세 → [[00_Deployment_HomePage]] (Railway · Neon · GitHub Actions · Docker)
```

---

## ⚙️ 패턴 · 기법

| |노트|
|---|---|
|**이메일**|[[NestJS_Email]]|
|**스케줄링**|[[NestJS_Scheduling]]|
|**스로틀링**|[[NestJS_Throttle]]|
|**로깅**|[[NestJS_Logger]]|
|**페이지네이션**|[[NestJS_Pagination]]|
|**시드**|[[NestJS_Seed]]|
|**통계 집계**|[[NestJS_StatsBucket]]|
|**캐시 테이블**|[[NestJS_CacheTable]]|
|**AI 연동**|[[NestJS_AiProvider]]|

```txt
NestJS_Email       Resend · Nodemailer · SMTP 설정 · MailService 패턴
NestJS_Scheduling  @Cron · @Interval · @Timeout · 타임존 · SchedulerRegistry
NestJS_Throttle    스로틀링 · 서비스 레벨 force 패턴
NestJS_Seed        테스트 데이터 생성 · $transaction · cleanup
NestJS_StatsBucket 통계 버킷 · 사전집계 vs 실시간 · upsert increment · 차트 연동
NestJS_CacheTable  외부 API(TMDB 등) 캐시 테이블 패턴 · MoviePool · ORDER BY RANDOM() · N+1 해소 · rate limit
멱등성 개념 → [[HTTP_Concept]] REST CRUD 섹션
```

---

## 📡 실시간 · WebSocket

| | 노트 |
|---|---|
| **NestJS** | [[NestJS_WebSocket]] |
| **Next.js** | [[NextJS_WebSocket]] |
| **패턴** | [[WebSocket_Patterns]] |

```txt
NestJS_WebSocket   Gateway · 룸 · 인증 · REST+WS 브로드캐스트
NextJS_WebSocket   socket.io-client · 싱글턴 · on/off 구독
WebSocket_Patterns 서버+클라이언트 코드 나란히 — 연결·룸·브로드캐스트·재연결
```

---

## 📄 API 문서화

| | 노트 |
|---|---|
| **NestJS** | [[NestJS_Swagger]] · [[NestJS_Versioning]] |
| **타입 생성** | [[OpenAPI_Codegen]] |

```txt
NestJS_Swagger    @ApiTags · @ApiProperty(format·enum) · @ApiCreatedResponse · @ApiOkResponse
                  Response DTO · addBearerAuth · Bearer Auth 설정
NestJS_Versioning URI·Header·MediaType 방식 · @Version · VERSION_NEUTRAL · defaultVersion
                  header: 'version' = 클라이언트가 보내야 하는 헤더 키 이름
OpenAPI_Codegen   NestJS dump-openapi → openapi-typescript → 프론트 타입 자동 생성
```

---

```txt
폴더 합친 이유:
  NestJS와 NodeJS가 실제로 얽히는 지점에서
  분류는 접두사(NestJS_ / NodeJS_)가 이미 하고 있음
```
