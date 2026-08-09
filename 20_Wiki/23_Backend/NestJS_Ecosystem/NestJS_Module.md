---
aliases:
  - "@Module"
  - 모듈
  - Dynamic Module
  - forRootAsync
  - Module
  - forwardRef
tags:
  - NestJS
related:
  - "[[00_NestJS_Ecosystem_HomePage]]"
  - "[[NestJS_Env_Config]]"
  - "[[NestJS_Service_Provider]]"
  - "[[NestJS_Concept]]"
---
# NestJS_Module — 모듈 시스템

>[!info]
>Module = NestJS 앱을 기능 단위로 나누는 상자.
> `@Module()`로 providers(서비스)·controllers·imports·exports를 선언. 
> `app.module.ts`가 모든 기능 모듈을 조립하는 루트 모듈
> DI 개념 → [[NestJS_Concept]], Service/Provider → [[NestJS_Service_Provider]]

---

# Module이란 ⭐️⭐️⭐️⭐️

```txt
NestJS 앱 = 여러 Module의 조합
Module = 관련 기능을 하나로 묶은 상자

  PostsModule    → 게시글 관련 (PostsController, PostsService)
  UsersModule    → 사용자 관련 (UsersController, UsersService)
  AuthModule     → 인증 관련  (AuthController, AuthService, Guard)
  AppModule      → 모든 모듈을 조립하는 루트

  Module 없이 전부 한 파일에 넣으면:
  → 파일이 거대해지고 어디서 무엇을 쓰는지 파악 불가
  → 테스트·수정이 어려워짐
```

---

# @Module — 4가지 속성 ⭐️⭐️⭐️⭐️

```typescript
@Module({
  imports:     [],   // 다른 모듈을 가져옴
  controllers: [],   // 이 모듈의 컨트롤러 (라우트 처리)
  providers:   [],   // 이 모듈의 서비스 (DI 컨테이너에 등록)
  exports:     [],   // 외부에 공개할 providers
})
export class PostsModule {}
```

```txt
controllers:
  이 모듈의 HTTP 엔드포인트 담당 클래스
  PostsController → /posts CRUD 처리
  NestJS가 자동으로 라우트 등록

providers:
  이 모듈 안에서 DI로 주입받아 쓸 클래스들
  PostsService, PrismaService 등
  NestJS DI 컨테이너에 등록 → new 없이 주입 가능

imports:
  다른 모듈을 가져와서 그 모듈의 exports를 사용
  PrismaModule을 import → PrismaService 주입 가능

exports:
  이 모듈의 providers 중 다른 모듈에게 공개할 것
  exports에 없으면 다른 모듈에서 주입 불가
```

## providers vs imports — 가장 헷갈리는 부분 ⭐️⭐️⭐️⭐️

```typescript
// PostsModule이 PrismaService를 쓰고 싶을 때

// ❌ providers에 직접 넣으면 (틀림)
@Module({
  providers: [PostsService, PrismaService],
  // → PrismaService가 모듈마다 새로 생성됨
  // → DB 연결이 공유 안 되고 중복 생성
})

// ✅ PrismaModule을 imports (올바름)
@Module({
  imports:   [PrismaModule],   // PrismaModule의 exports를 가져옴
  providers: [PostsService],   // 이 모듈 전용 서비스만
})
```

```txt
규칙:
  이 모듈에서 직접 만드는 서비스 → providers
  다른 모듈의 서비스를 빌려 쓸 때 → imports

  비유:
  providers = "이 팀에서 직접 고용한 직원"
  imports   = "다른 팀에서 파견 온 직원" (그 팀이 exports로 공개한)
```

## exports가 필요한 이유 ⭐️⭐️⭐️⭐️

```typescript
// PrismaModule
@Module({
  providers: [PrismaService],
  exports:   [PrismaService],  // ← 이게 있어야 다른 모듈에서 쓸 수 있음
})
export class PrismaModule {}

// AuthModule — PrismaService를 쓰고 싶음
@Module({
  imports:   [PrismaModule],   // PrismaModule을 가져옴
  providers: [AuthService],    // AuthService 안에서 PrismaService 주입받음
})
export class AuthModule {}
```

```txt
exports가 없으면:
  PrismaModule을 imports해도 PrismaService를 주입받을 수 없음
  → "Nest can't resolve dependencies of AuthService" 에러

exports = "우리 모듈의 이 서비스는 외부에서 써도 됩니다"
없으면 = 모듈 외부에서 접근 불가 (캡슐화)
```

---

# app.module.ts — 루트 모듈 ⭐️⭐️⭐️⭐️

```typescript
@Module({
  imports: [
    // 전역 설정
    ConfigModule.forRoot({ isGlobal: true }),

    // 전역 DB
    PrismaModule,

    // 인증
    AuthModule,

    // 기능 모듈들
    PostsModule,
    UsersModule,
    RoomsModule,
    NotificationsModule,
  ],
  // 루트 모듈은 보통 controllers, providers 비어있음
  // 기능은 각 모듈에 위임
})
export class AppModule {}
```

```txt
app.module.ts에 넣어야 하는 것:
  ✅ 전역 모듈 (ConfigModule, PrismaModule)
  ✅ 기능 모듈들 (PostsModule, UsersModule, ...)
  ❌ 개별 서비스·컨트롤러 → 각 기능 모듈에

app.module.ts가 너무 커진다면:
  기능들이 모듈로 분리가 안 된 것
  각 기능을 PostsModule, UsersModule 등으로 분리해야 함
```

---

# 기능 모듈 패턴 ⭐️⭐️⭐️⭐️

```typescript
// posts/posts.module.ts
@Module({
  imports:     [PrismaModule],
  controllers: [PostsController],
  providers:   [PostsService],
  exports:     [PostsService],   // 다른 모듈에서 PostsService 필요할 때
})
export class PostsModule {}
```

```bash
# CLI로 생성
nest generate module posts
nest generate controller posts
nest generate service posts

# 한 번에 (resource = module + controller + service + dto)
nest generate resource posts
```

---

# @Global() — 전역 모듈 ⭐️⭐️⭐️

```typescript
@Global()   // AppModule에 한 번 import하면 전체 앱에서 사용 가능
@Module({
  providers: [PrismaService],
  exports:   [PrismaService],
})
export class PrismaModule {}
```

```txt
@Global() 없으면:
  PrismaService를 쓰는 PostsModule, UsersModule, AuthModule ...
  전부 imports: [PrismaModule] 해야 함

@Global() 있으면:
  AppModule에서 한 번만 import → 나머지 모듈에서 imports 불필요

남용 금지:
  @Global()은 DB, Config처럼 진짜 전역인 것만
  남용하면 의존 관계가 불명확해짐
```

---

# forRootAsync — 동적 모듈 (환경변수 사용) ⭐️⭐️⭐️

```typescript
// ❌ 정적 설정 — 환경변수 사용 불가
JwtModule.register({ secret: 'hard-coded' })

// ✅ 동적 설정 — ConfigService로 환경변수에서 읽음
JwtModule.registerAsync({
  imports:    [ConfigModule],
  inject:     [ConfigService],
  useFactory: (config: ConfigService) => ({
    secret:      config.getOrThrow('JWT_SECRET'),
    signOptions: { expiresIn: '15m' },
  }),
})
```

```txt
registerAsync / forRootAsync를 쓰는 이유:
  환경변수는 런타임에 주입됨
  모듈 정의 시점에는 ConfigService가 아직 준비 안 됨
  → useFactory로 ConfigService를 주입받아서 그때 읽음

라이브러리마다 메서드 이름이 다름:
  JwtModule    → register / registerAsync
  TypeOrmModule → forRoot / forRootAsync
  동작 원리는 동일
```

---

# forwardRef — 순환 참조 해결 ⭐️⭐️⭐️

```txt
순환 참조 = A 모듈이 B를 import, B 모듈이 A를 import
  → NestJS가 어느 것을 먼저 만들어야 할지 모름 → 에러

예시:
  AuthModule → UsersModule (회원 조회 필요)
  UsersModule → AuthModule (토큰 검증 필요)
  → 서로 참조 → 순환
```

```typescript
// ❌ 순환 참조 에러 발생
// auth.module.ts
@Module({ imports: [UsersModule] })
export class AuthModule {}

// users.module.ts
@Module({ imports: [AuthModule] })
export class UsersModule {}
```

```typescript
// ✅ forwardRef로 해결
// auth.module.ts
import { forwardRef } from '@nestjs/common';

@Module({
  imports: [forwardRef(() => UsersModule)],  // 나중에 해석
})
export class AuthModule {}

// users.module.ts
@Module({
  imports: [forwardRef(() => AuthModule)],   // 나중에 해석
})
export class UsersModule {}
```

```typescript
// 서비스에서 forwardRef로 주입받을 때
@Injectable()
export class AuthService {
  constructor(
    @Inject(forwardRef(() => UsersService))
    private usersService: UsersService,
  ) {}
}
```

```txt
forwardRef(() => Module):
  "이 모듈은 나중에 해석해라"
  → NestJS가 순환을 피해서 지연 참조로 해결

  최선은 구조를 바꿔 순환을 없애는 것
  forwardRef는 어쩔 수 없을 때만 (순환이 불가피한 경우)
  남용하면 코드 구조가 복잡해짐
```

---

# 공통 모듈 분리 — SharedModule ⭐️⭐️⭐️

```txt
여러 기능 모듈에서 공통으로 쓰는 것들을 하나의 모듈로 묶는 패턴

예시:
  PostsModule, UsersModule, RoomsModule 전부
  → PrismaService 필요
  → EmailService 필요
  → 공통 유틸 필요

방법 1 — 각 모듈에서 직접 import (반복):
  PostsModule:  imports: [PrismaModule, EmailModule]
  UsersModule:  imports: [PrismaModule, EmailModule]
  RoomsModule:  imports: [PrismaModule, EmailModule]
  → 추가할 때마다 전부 수정해야 함

방법 2 — SharedModule로 묶기:
  SharedModule: imports + exports [PrismaModule, EmailModule]
  PostsModule:  imports: [SharedModule]  ← 한 줄로 해결
```

```typescript
// shared/shared.module.ts
@Module({
  imports: [
    PrismaModule,
    EmailModule,
  ],
  exports: [
    PrismaModule,   // PrismaModule의 exports를 재수출
    EmailModule,
  ],
})
export class SharedModule {}
```

```typescript
// posts/posts.module.ts
@Module({
  imports:     [SharedModule],     // PrismaService, EmailService 전부 사용 가능
  controllers: [PostsController],
  providers:   [PostsService],
})
export class PostsModule {}
```

```txt
SharedModule의 exports:
  PrismaModule 자체를 exports → PrismaModule의 exports(PrismaService)가 전달됨
  모듈을 re-export 하는 것
  → SharedModule을 import한 모든 곳에서 PrismaService 주입 가능

주의:
  SharedModule이 너무 커지면 오히려 관리가 어려워짐
  정말 공통으로 쓰이는 것만 포함
  @Global() 대안: 진짜 전역이어야 하면 @Global() 사용
```

| 에러                                            | 원인                                | 해결                                    |
| --------------------------------------------- | --------------------------------- | ------------------------------------- |
| `Nest can't resolve dependencies of XService` | 주입받으려는 서비스가 providers/imports에 없음 | 해당 서비스 모듈을 imports에 추가                |
| `XService is not exported`                    | 그 모듈이 exports 안 함                 | 해당 모듈의 exports에 XService 추가           |
| `Circular dependency`                         | A 모듈이 B를, B 모듈이 A를 import         | `forwardRef()` 사용 (위 섹션 참고) 또는 구조 재설계 |