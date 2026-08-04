---
aliases:
  - NestJS 설치
  - Concept
  - DI
tags:
  - NestJS
related:
  - "[[00_NestJS_Ecosystem_HomePage]]"
  - "[[Monorepo_PNPM]]"
  - "[[NestJS_Env_Config]]"
  - "[[NestJS_Module]]"
  - "[[NestJS_Controller]]"
  - "[[NestJS_Service_Provider]]"
---
# NestJS_Concept — NestJS 핵심 개념
 
>[!info]
>NestJS = Express 위에 TypeScript·OOP 구조를 얹은 프레임워크. 
>Module·Controller·Service 계층 구조와 의존성 주입(DI)으로 큰 앱을 체계적으로 조립한다. 
>요청은 Middleware → Guard → Pipe → Controller → Service 순서로 처리된다. 
>설치·CLI 명령어도 이 파일에 정리.

---

# NestJS란 — 왜 Express 대신 쓰는가 ⭐️⭐️⭐️⭐️

```txt
Express:
  HTTP 요청·응답만 제공하는 최소한의 프레임워크
  라우트, 미들웨어, 에러 처리를 직접 구성
  파일 구조, 코드 패턴에 아무런 제약 없음
  → 작은 앱엔 편리, 큰 앱엔 코드가 제각각

NestJS:
  Express를 기반으로 체계적인 구조를 강제
  TypeScript 기본 지원
  Module·Controller·Service로 역할을 분리
  의존성 주입(DI) 컨테이너 내장
  Guard·Pipe·Interceptor 등 요청 처리 레이어 제공
  → 팀이 커져도 일관된 코드 구조 유지
```

```txt
비유:
  Express  = 빈 땅 (직접 집을 설계하고 지어야 함)
  NestJS   = 설계도가 있는 아파트 (방의 용도가 정해져 있음)

  큰 앱을 만들수록 NestJS의 강제 구조가 장점이 됨
  "이 코드는 어디에 있어야 하는가"가 명확해짐
```

---

# 핵심 구성 요소 4가지 ⭐️⭐️⭐️⭐️

```txt
Module      기능 단위를 묶는 상자 (DmsModule, RoomsModule)
Controller  HTTP 요청을 받고 응답을 반환 (@Get, @Post ...)
Service     비즈니스 로직 + DB 쿼리 (실제 일을 하는 곳)
Provider    DI 컨테이너에 등록되는 것 (Service, Guard, Repository 등)
```

```typescript
// 전형적인 구조 — 기능 하나가 이 세 파일로 구성됨
posts.module.ts      // PostsModule — Controller·Service 묶기
posts.controller.ts  // PostsController — 요청 받기
posts.service.ts     // PostsService — DB 쿼리·로직
```

## 각 레이어의 역할

```typescript
// Controller — 요청 받기 + 응답 반환만
@Controller('posts')
export class PostsController {
  constructor(private readonly postsService: PostsService) {}  // DI

  @Get()
  findAll(@Query() query: ListPostsQueryDto) {
    return this.postsService.findAll(query);  // Service에 위임
  }
}

// Service — 실제 일을 하는 곳
@Injectable()
export class PostsService {
  constructor(private readonly prisma: PrismaService) {}  // DI

  async findAll(query: ListPostsQueryDto) {
    return this.prisma.post.findMany({ where: { ... } });  // DB 쿼리
  }
}

// Module — 둘을 묶고 DI 컨테이너에 등록
@Module({
  controllers: [PostsController],
  providers:   [PostsService],
})
export class PostsModule {}
```

---

# 요청 처리 순서 (Pipeline) ⭐️⭐️⭐️⭐️

```txt
클라이언트 요청
      │
      ▼
 1. Middleware      Express 미들웨어 — 로깅, 바디 파싱, CORS 등
      │             app.use() 로 등록한 것들
      ▼
 2. Guard           인증·인가 — "이 요청이 허용되는가?"
      │             JwtAuthGuard: 토큰 검증
      │             RolesGuard: 역할 확인
      ▼
 3. Interceptor      (요청 전처리) — 캐싱, 요청 변환, 로깅
      │
      ▼
 4. Pipe            데이터 변환·검증 — ParseUUIDPipe, ValidationPipe
      │             형식이 잘못되면 여기서 400 반환
      ▼
 5. Controller      라우트 핸들러 실행 — @Get, @Post 메서드
      │             요청 데이터 꺼내서(@Param, @Body) Service 호출
      ▼
 6. Service         비즈니스 로직 + DB 쿼리
      │
      ▼
 7. Interceptor      (응답 후처리) — 응답 변환, 캐시 저장
      │
      ▼
 8. ExceptionFilter  예외 발생 시 처리 — 에러 응답 형식 통일
      │
      ▼
 클라이언트 응답
```

## 각 레이어가 하는 일

```txt
Middleware (1)
  Express 수준의 처리
  모든 요청에 공통 적용 (로깅, 바디 파싱)
  NestJS의 Guard/Pipe보다 먼저 실행

Guard (2)
  "이 요청을 통과시킬 것인가" 결정
  true → 다음 레이어로
  false 또는 throw → 요청 차단 (401, 403)
  실행 시점: Pipe·Controller보다 앞 → 인증 안 된 요청이 DB에 닿지 않음

Pipe (4)
  Guard 통과 후 실행
  데이터 변환: string → number (ParseIntPipe)
  데이터 검증: DTO class-validator (ValidationPipe)
  실패 시 자동 400 BadRequest

Controller (5)
  Guard·Pipe를 통과한 깨끗한 데이터가 도달
  데이터를 꺼내서 Service에 전달하는 것만 담당

ExceptionFilter (8)
  어느 레이어에서든 throw된 예외를 잡아서 응답 형식 변환
  throw new NotFoundException() → { statusCode: 404, message: ... }
```

---

# 의존성 주입 (DI) ⭐️⭐️⭐️⭐️

## 의존성(Dependency)이란

```txt
"A가 B를 필요로 할 때, A는 B에 의존한다"

PostsController는 PostsService가 없으면 아무것도 못 함
→ PostsController의 의존성 = PostsService

PostsService는 PrismaService가 없으면 DB에 접근 못 함
→ PostsService의 의존성 = PrismaService
```

## 주입(Injection)이 없으면

```typescript
// 의존성을 직접 만드는 방법 (DI 없이)
class PostsController {
  private postsService: PostsService;

  constructor() {
    // PostsController가 PostsService를 "직접 생성"
    const prisma  = new PrismaService();
    this.postsService = new PostsService(prisma);
    //                   ↑ PostsService의 내부 구현까지 알아야 함
    //                   ↑ PostsService가 바뀌면 여기도 수정
  }
}
```

```txt
문제:
  PostsController가 PostsService의 내부 생성 방법까지 알아야 함
  PrismaService처럼 공통으로 쓰는 것이 여러 곳에서 각각 new PrismaService()
  → 인스턴스가 여러 개 생성 → 비효율
  → DB 연결이 여러 개 → 심각한 문제

  테스트할 때:
  PostsController를 테스트하려는데 실제 PostsService + 실제 PrismaService + 실제 DB가 필요
  → 가짜(mock) 객체를 쓰기 어려움
```

## 주입(Injection)이란 — 외부에서 만들어 넘겨주기

```typescript
// DI 사용
class PostsController {
  constructor(
    private readonly postsService: PostsService  // "나에게 주입해줘"
  ) {}
  // PostsController는 PostsService를 어떻게 만드는지 모름
  // 그냥 "나는 PostsService가 필요해"라고 선언만 함
}
```

```txt
NestJS가 하는 일:
  PostsController를 만들려면 PostsService가 필요하다는 걸 앎
  PostsService를 만들려면 PrismaService가 필요하다는 걸 앎
  PrismaService를 먼저 만들고 → PostsService에 넣고 → PostsController에 넣음

  개발자가 할 일:
  "나는 이게 필요해" 를 constructor 타입으로 선언
  NestJS가 알아서 만들어서 넣어줌 = 주입(Injection)
```

## IoC — 제어 역전


```txt
Inversion of Control (제어의 역전)

일반적인 방식:
  내가 필요한 객체를 내가 만듦 (내가 제어)
  new PostsService(new PrismaService())

IoC (DI):
  내가 필요한 객체를 NestJS가 만들어서 줌 (제어가 역전됨)
  constructor(private readonly postsService: PostsService)

"제어가 역전"된다는 것:
  객체 생성의 책임이 나 → NestJS (DI 컨테이너)로 넘어감
```

## NestJS DI 동작 방식


```typescript
// 1. @Injectable() — "나는 DI 컨테이너에 등록 가능해"
@Injectable()
export class PrismaService { ... }

@Injectable()
export class PostsService {
  constructor(private readonly prisma: PrismaService) {}
  // "나는 PrismaService가 필요해"
}

// 2. providers 등록 — "이 모듈에서 이것들을 DI 컨테이너에 넣어"
@Module({
  providers: [PrismaService, PostsService],
})
export class PostsModule {}

// 3. NestJS가 자동으로:
//   PrismaService 인스턴스 생성
//   → PostsService(prismaInstance) 생성
//   → PostsController(postsServiceInstance) 생성
```

## 싱글턴 — 하나의 인스턴스를 공유

```txt
NestJS DI는 기본적으로 싱글턴
같은 모듈 범위 안에서 동일한 클래스는 인스턴스 하나만 생성

PostsController와 CommentsController가 둘 다 PrismaService를 주입받아도
PrismaService 인스턴스는 하나 — DB 연결이 하나만 유지됨

new PrismaService()를 직접 하면:
  각자 인스턴스가 생겨서 DB 연결이 여러 개 → 연결 풀 낭비
```

## DI의 실제 장점

```typescript
// 테스트할 때 — 가짜(Mock) Service로 교체 가능
const module = await Test.createTestingModule({
  providers: [
    PostsController,
    {
      provide:  PostsService,
      useValue: {              // 실제 PostsService 대신 가짜 객체 주입
        create: jest.fn().mockResolvedValue({ id: '123', title: 'test' }),
      },
    },
  ],
}).compile();

// PostsController를 실제 DB 없이 테스트 가능
// PostsService의 동작을 원하는 대로 제어 가능
```


```txt
DI가 주는 것:
  ① 느슨한 결합 — Controller가 Service 구현 방법을 몰라도 됨
  ② 싱글턴 관리 — 같은 Service 인스턴스를 여러 곳에서 공유
  ③ 테스트 용이 — 가짜 객체로 쉽게 교체
  ④ 코드 변경 최소화 — Service 내부가 바뀌어도 주입받는 쪽 수정 불필요
```
---

# 전체 아키텍처 — 파일 구조

```txt
src/
├── main.ts                  앱 시작점 — 포트, 전역 설정
├── app.module.ts            루트 모듈 — 기능 모듈들 조립
├── prisma/
│   └── prisma.service.ts    DB 연결 — @Global()로 전역 제공
├── auth/
│   ├── auth.module.ts
│   ├── auth.controller.ts   /auth/login, /auth/register
│   ├── auth.service.ts
│   ├── jwt-auth.guard.ts    전역 Guard
│   └── decorators/          @UserId(), @Public()
├── posts/
│   ├── posts.module.ts
│   ├── posts.controller.ts  /posts CRUD
│   ├── posts.service.ts
│   └── dto/
│       ├── create-post.dto.ts
│       ├── update-post.dto.ts
│       └── list-posts-query.dto.ts
└── rooms/ ...               다른 기능도 같은 패턴
```

# NestJS 설치 및 프로젝트 생성 ⭐️⭐️⭐️

## 새 프로젝트 시작

```bash
# NestJS CLI 전역 설치
npm install -g @nestjs/cli

# 새 프로젝트 생성
nest new project-name
# 패키지 매니저 선택: pnpm 권장
```

## 모노레포에서 NestJS API 추가 (pnpm workspace)

```bash
# 모노레포 루트에서
mkdir apps/api
cd apps/api
nest new . --skip-git
# 또는 직접 package.json 만들고 패키지 설치
```

## 주요 패키지 설치

```bash
# 핵심
pnpm add @nestjs/common @nestjs/core @nestjs/platform-express reflect-metadata rxjs

# 설정·환경변수
pnpm add @nestjs/config

# JWT 인증
pnpm add @nestjs/jwt @nestjs/passport passport passport-jwt
pnpm add -D @types/passport-jwt

# 유효성 검사
pnpm add class-validator class-transformer

# Swagger 문서화
pnpm add @nestjs/swagger

# WebSocket
pnpm add @nestjs/websockets @nestjs/platform-socket.io socket.io

# 스케줄링
pnpm add @nestjs/schedule

# Prisma → [[NestJS_Migration]] 참고
pnpm add prisma @prisma/client
```

## 자주 쓰는 nest CLI 명령어

```bash
nest g module posts          # posts.module.ts 생성
nest g controller posts      # posts.controller.ts 생성
nest g service posts         # posts.service.ts 생성
nest g resource posts        # module + controller + service + dto 한 번에 생성
```

```txt
nest g resource posts 가 만들어 주는 것:
  posts/
  ├── posts.module.ts
  ├── posts.controller.ts
  ├── posts.service.ts
  └── dto/
      ├── create-post.dto.ts
      └── update-post.dto.ts

AppModule에 PostsModule을 자동으로 imports에 추가해줌
```

---

# 각 개념의 상세 노트

```txt
전체 흐름 · DI · 설치 · 요청 순서   → 이 파일 (NestJS_Concept)
Module 구성 · DI · forwardRef        → [[NestJS_Module]]
Service · Provider · DI 상세         → [[NestJS_Service_Provider]]
Controller · 라우트 · @Param/@Body   → [[NestJS_Controller]]
DTO · class-validator · PartialType  → [[NestJS_DTO]]
Pipe · ParseUUIDPipe · ValidationPipe → [[NestJS_Pipe]]
Guard · JWT 검증 · @Public · @Roles  → [[NestJS_JwtGuard]]
Prisma · findMany · where · select   → [[NestJS_Prisma]]
Prisma 패턴 · 토글 · 페어키          → [[NestJS_Prisma_Patterns]]
마이그레이션 · migrate 명령어         → [[NestJS_Migration]]
트랜잭션 · $transaction               → [[NestJS_Transaction]]
페이지네이션 · cursor · take+1        → [[NestJS_Pagination]]
WebSocket · Gateway                   → [[NestJS_WebSocket]]
CORS 설정                             → [[NestJS_CORS]]
```

---

# NestJS 시작 시 자주 하는 설정

```typescript
// main.ts — 앱 전역 설정
async function bootstrap() {
  const app = await NestFactory.create(AppModule);

  // 전역 파이프 — 모든 @Body, @Query 자동 검증·변환
  app.useGlobalPipes(new ValidationPipe({
    whitelist: true,
    transform: true,
    transformOptions: { enableImplicitConversion: true },
  }));

  // 전역 Guard — 모든 라우트 JWT 인증 (공개 라우트엔 @Public())
  app.useGlobalGuards(new JwtAuthGuard(jwtService, configService, reflector));

  // CORS 설정
  app.enableCors({ origin: [...], credentials: true });

  // Swagger 설정
  const config = new DocumentBuilder().setTitle('API').setVersion('1.0').build();
  SwaggerModule.setup('api', app, SwaggerModule.createDocument(app, config));

  await app.listen(3000);
}
```

```txt
전역 설정 vs 개별 설정:
  useGlobalPipes  → 모든 라우트에 적용 (컨트롤러마다 @UsePipes 안 해도 됨)
  useGlobalGuards → 모든 라우트에 인증 적용 (@Public()으로 예외 처리)
  enableCors      → 프론트엔드 도메인 허용 → [[NestJS_CORS]]
```