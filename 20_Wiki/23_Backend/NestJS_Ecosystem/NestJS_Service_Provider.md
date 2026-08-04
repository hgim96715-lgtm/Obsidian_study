---
aliases:
  - Dependency Injection
  - DI
  - IoC
  - Provider
  - Service
tags:
  - NestJS
related:
  - "[[00_NestJS_Ecosystem_HomePage]]"
  - "[[NestJS_Controller]]"
  - "[[NestJS_Module]]"
  - "[[NestJS_Prisma]]"
  - "[[NestJS_Concept]]"
---
# NestJS_Service_Provider — 서비스 · 프로바이더

> [!info] 
> Service = 비즈니스 로직을 담당하는 클래스.
>  `@Injectable()`로 DI 컨테이너에 등록하면 다른 클래스의 `constructor`에서 주입받아 쓸 수 있다. 
>  Provider = DI 컨테이너에 등록되는 모든 것의 통칭 — Service가 가장 흔하고, Guard·Factory·Value도 Provider가 될 수 있다.
>   DI 개념 → [[NestJS_Concept]], 모듈 구성 → [[NestJS_Module]]

---

# Service란 — 왜 필요한가 ⭐️⭐️⭐️⭐️

```txt
Controller가 모든 것을 하면 어떻게 되는가:
  @Post()
  async create(@Body() dto: CreatePostDto) {
    const user = await this.prisma.user.findUnique(...);  // DB 쿼리
    if (!user.isActive) throw new ForbiddenException();   // 비즈니스 규칙
    const post = await this.prisma.post.create(...);      // DB 쿼리
    await this.emailService.send(...);                    // 이메일 발송
    return post;
  }
  → 컨트롤러가 DB, 비즈니스 규칙, 외부 API를 직접 알아야 함
  → 코드가 길어지고 재사용 불가, 테스트 어려움
```

```txt
Service로 분리하면:
  Controller  → 요청 받고 응답 반환만 (얇게)
  Service     → 실제 일을 담당 (DB 쿼리, 로직, 외부 호출)

  장점:
  - 같은 로직을 여러 Controller에서 재사용 가능
  - Service만 따로 테스트 가능
  - 코드 역할이 명확히 분리됨
```

---

# @Injectable() — DI 등록 표시 ⭐️⭐️⭐️⭐️

```typescript
@Injectable()   // ← 이게 없으면 DI 컨테이너가 이 클래스를 모름
export class PostsService {
  constructor(private readonly prisma: PrismaService) {}  // 주입받기

  async create(userId: string, dto: CreatePostDto) {
    return this.prisma.post.create({
      data: { ...dto, authorId: userId },
    });
  }
}
```

```txt
@Injectable() 이 하는 일:
  "나는 DI 컨테이너에 등록될 수 있어"라는 표시
  이것이 없으면 다른 클래스가 constructor에서 주입받을 수 없음

  @Injectable() 없이 주입 시도 → NestJS 에러:
  "Nest can't resolve dependencies of PostsService"
```

---

# Provider란 — Service보다 넓은 개념 ⭐️⭐️⭐️

```txt
Provider = DI 컨테이너에 등록되어 주입 가능한 모든 것

가장 흔한 Provider = Service (@Injectable() 붙인 클래스)
그 외에도:
  Guard       (@Injectable() + CanActivate)
  Pipe        (@Injectable() + PipeTransform)
  Interceptor (@Injectable() + NestInterceptor)
  Factory     (값을 만들어내는 함수)
  Value       (상수, 환경변수 등)

모두 Module의 providers 배열에 등록해서 사용
```

---

# Module에 등록 ⭐️⭐️⭐️⭐️

```typescript
@Module({
  providers: [PostsService],   // 이 모듈에서 PostsService를 쓸 수 있게 등록
  controllers: [PostsController],
})
export class PostsModule {}
```

```txt
providers 배열에 등록해야 하는 이유:
  NestJS는 어떤 Service가 있는지 자동으로 알지 못함
  providers에 명시해야 DI 컨테이너가 인식하고 주입할 준비를 함
  → 등록 안 하면 "Nest can't resolve dependencies" 에러

  @Global() 로 등록된 것(PrismaService 등)은 예외
  → 전역으로 이미 등록되어 어디서든 주입 가능
```

---

# Service 기본 패턴 ⭐️⭐️⭐️⭐️

```typescript
@Injectable()
export class PostsService {
  // 다른 Service나 PrismaService를 주입받아 사용
  constructor(
    private readonly prisma:        PrismaService,
    private readonly emailService:  EmailService,
  ) {}

  // 조회
  async findAll(query: ListPostsQueryDto) {
    return this.prisma.post.findMany({ where: { ... } });
  }

  // 단건 조회 — 없으면 404
  async findOne(id: string) {
    const post = await this.prisma.post.findUnique({ where: { id } });
    if (!post) throw new NotFoundException('게시글을 찾을 수 없습니다.');
    return post;
  }

  // 생성
  async create(userId: string, dto: CreatePostDto) {
    return this.prisma.post.create({
      data: { ...dto, authorId: userId },
    });
  }

  // 수정 — 본인만 가능
  async update(id: string, userId: string, dto: UpdatePostDto) {
    const post = await this.findOne(id);  // 같은 Service 내 메서드 호출 가능
    if (post.authorId !== userId) throw new ForbiddenException();
    return this.prisma.post.update({ where: { id }, data: dto });
  }

  // 삭제
  async remove(id: string, userId: string) {
    const post = await this.findOne(id);
    if (post.authorId !== userId) throw new ForbiddenException();
    await this.prisma.post.delete({ where: { id } });
  }
}
```

```txt
Service에서 예외를 던지는 이유:
  Controller는 "요청 받기 + 서비스 호출 + 응답"만 담당
  "없으면 404", "권한 없으면 403" 같은 판단은 Service의 비즈니스 로직
  → Service에서 NestJS 예외를 던지면 NestJS가 자동으로 에러 응답 생성

자주 쓰는 예외:
  NotFoundException(404)        리소스 없음
  ForbiddenException(403)       권한 없음
  UnauthorizedException(401)    인증 안 됨
  BadRequestException(400)      요청 형식 잘못됨
  ConflictException(409)        중복·충돌
```

---

# 다른 모듈의 Service 주입받기 ⭐️⭐️⭐️

```typescript
// DmsService에서 RoomsService가 필요한 경우

// 1. RoomsModule에서 exports 추가
@Module({
  providers:   [RoomsService],
  exports:     [RoomsService],   // ← 이 줄이 있어야 다른 모듈에서 사용 가능
})
export class RoomsModule {}

// 2. DmsModule에서 RoomsModule을 imports에 추가
@Module({
  imports:   [RoomsModule],      // ← RoomsService를 쓰려면 RoomsModule import
  providers: [DmsService],
})
export class DmsModule {}

// 3. DmsService에서 주입받아 사용
@Injectable()
export class DmsService {
  constructor(private readonly roomsService: RoomsService) {}
}
```

```txt
exports가 없으면:
  RoomsModule 안에서만 RoomsService를 쓸 수 있음
  DmsModule에서 주입하려 하면 에러 발생

순환 참조 (두 모듈이 서로 import):
  forwardRef 또는 공통 모듈 분리로 해결
  → [[NestJS_Module]] forwardRef 섹션
```

---

# 커스텀 Provider ⭐️⭐️⭐️

```txt
@Injectable() 클래스 대신 값, 함수로 Provider를 만드는 방법
providers 배열에 { provide, useXxx } 형태로 등록
```

## useValue — 상수값 제공

```typescript
// 설정값, 환경변수, 목 객체 등을 Provider로 등록
@Module({
  providers: [
    {
      provide:   'API_KEY',         // 주입할 때 사용할 토큰(이름)
      useValue:  process.env.API_KEY,  // 주입될 값
    },
  ],
})

// 주입받는 곳
constructor(@Inject('API_KEY') private readonly apiKey: string) {}
```

## useFactory — 함수로 생성, 비동기 가능

```typescript
// 다른 Provider에 의존하거나 비동기 초기화가 필요할 때
@Module({
  providers: [
    {
      provide:    'REDIS_CLIENT',
      useFactory: async (configService: ConfigService) => {
        const client = createClient({ url: configService.get('REDIS_URL') });
        await client.connect();
        return client;
      },
      inject: [ConfigService],   // useFactory에 주입할 것들
    },
  ],
})
```

## useClass — 환경에 따라 다른 구현체

```typescript
// 개발 환경과 운영 환경에서 다른 클래스 사용
@Module({
  providers: [
    {
      provide:   EmailService,
      useClass:  process.env.NODE_ENV === 'production'
        ? SendGridEmailService   // 운영: 실제 이메일 발송
        : FakeEmailService,      // 개발: 로컬에서 출력만
    },
  ],
})
```

```txt
커스텀 Provider를 쓰는 경우:
  useValue  → 환경변수, 설정값, 테스트 목(mock) 객체
  useFactory → 비동기 초기화 (Redis 연결, DB 설정 등)
  useClass  → 인터페이스 기반 교체 가능 구현
```

---

# Service 설계 원칙 ⭐️⭐️⭐️

```txt
하나의 Service = 하나의 도메인 책임

PostsService    → 게시글 관련 모든 것
UsersService    → 유저 관련 모든 것
EmailService    → 이메일 발송 (도메인과 독립)
PrismaService   → DB 연결 관리 (전역)

Service가 너무 커지면:
  PostsService → PostsQueryService + PostsCommandService 로 분리
  (조회와 수정을 분리하는 CQRS 패턴)
```

```typescript
// 한 Service에서 다른 Service를 주입하는 건 자연스러움
@Injectable()
export class DmsService {
  constructor(
    private readonly prisma:        PrismaService,   // DB
    private readonly usersService:  UsersService,    // 유저 확인
    private readonly gateway:       SharedGateway,   // WebSocket emit
  ) {}

  async createDmRoom(userId: string, targetId: string) {
    await this.usersService.assertExists(targetId);  // UsersService 활용
    const room = await this.prisma.dmRoom.create(...);
    this.gateway.emitDmRoomCreated(userId, room);    // Gateway 활용
    return room;
  }
}
```

---

# 자주 만나는 에러

| 에러                                | 원인                   | 해결                                                |
| --------------------------------- | -------------------- | ------------------------------------------------- |
| `Nest can't resolve dependencies` | providers 배열에 등록 안 됨 | 해당 Service를 Module의 providers에 추가                 |
| 다른 모듈 Service를 못 씀                | exports 누락           | 해당 Service를 Module의 exports에 추가                   |
| 순환 참조 에러                          | 두 모듈이 서로 import      | forwardRef 또는 SharedModule 분리 → [[NestJS_Module]] |
| `@Inject('TOKEN')`이 undefined     | provide 토큰 이름 불일치    | provide와 @Inject의 문자열이 정확히 같은지 확인                 |