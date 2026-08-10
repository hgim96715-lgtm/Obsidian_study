---
aliases:
  - Auth
  - Guard
  - JWT
  - JwtAuthGuard
  - request.user
  - RolesGuard
tags:
  - NestJS
related:
  - "[[00_NestJS_Ecosystem_HomePage]]"
  - "[[Auth_Concept]]"
  - "[[HTTP_Concept]]"
  - "[[NestJS_Auth]]"
  - "[[TS_ImportType]]"
---
# NestJS_JwtGuard — JWT Guard · 인증 파이프라인

>[!info]
>Guard = 컨트롤러 핸들러 실행 전에 "이 요청이 허용되는가"를 결정하는 레이어. 
>`CanActivate` 인터페이스를 구현해서 만든다.
> `JwtAuthGuard`·`@Public()`·`@UserId()`는 **NestJS가 제공하는 조립 도구로 직접 만드는 것** — 내장이 아니다. 
> JWT 개념 → [[Auth_Concept]], HTTP 헤더·Authorization → [[HTTP_Concept]], JWT 발급 로직 → [[NestJS_Auth]]

---

# NestJS가 제공하는 것 vs 직접 만드는 것 ⭐️⭐️⭐️⭐️

```txt
헷갈림의 근본 원인:
  @Roles · @Public · @UserId · JwtAuthGuard
  → 전부 "직접 만드는 것"인데 예제마다 당연하게 나와서 내장인 것처럼 보임

NestJS가 제공하는 조립 도구:
  CanActivate      → Guard를 만드는 인터페이스
  ExecutionContext  → 현재 요청 정보에 접근하는 객체
  SetMetadata      → 메타데이터를 붙이는 함수
  Reflector        → 붙인 메타데이터를 읽는 도구
  createParamDecorator → 파라미터 데코레이터 만드는 함수
  APP_GUARD        → 전역 Guard 등록 토큰

개발자가 그 도구로 만드는 것:
  JwtAuthGuard     → CanActivate + JwtService로 직접 구현
  @Public()        → SetMetadata로 직접 만든 데코레이터
  @UserId()        → createParamDecorator로 직접 만든 데코레이터
  @Roles()         → SetMetadata로 직접 만든 데코레이터

"업계 관례"로 굳어진 것:
  이름과 모양이 어디서나 같아서 내장처럼 보임
  실제로는 NestJS 공식 문서의 패턴을 따라 만드는 것
```

---

# JWT 인증 흐름 요약 ⭐️⭐️⭐️⭐️

```txt
1. 클라이언트가 Authorization 헤더에 토큰을 담아 요청
   Authorization: Bearer eyJhbGci...

2. JwtAuthGuard가 가로챔
   헤더에서 토큰 추출 → jwtService.verify() → payload 꺼냄
   request.user = { sub: 'userId', iat: ..., exp: ... }

3. @UserId() 데코레이터
   request.user.sub를 컨트롤러 파라미터에 주입

4. 컨트롤러
   @UserId() userId: string 으로 바로 사용

JWT 개념(토큰이란, 서명, Base64) → [[Auth_Concept]]
Authorization 헤더 구조 → [[HTTP_Concept]]
토큰 발급(login, issueTokens) → [[NestJS_Auth]]
```

---

# Guard란 — CanActivate ⭐️⭐️⭐️⭐️

```txt
Guard = CanActivate 인터페이스를 구현한 클래스
  canActivate() 메서드 하나만 구현하면 됨
  true  → 요청 통과
  false → 403 Forbidden 자동 반환
  throw → 해당 예외로 응답

요청 처리 순서에서의 위치:
  요청 → 미들웨어 → Guard → Interceptor → Pipe → Controller → Service
                  ↑
              여기서 인증·인가 처리

Guard가 Middleware보다 나중에 실행되는 이유:
  미들웨어는 Express 레이어 (라우팅 전)
  Guard는 NestJS 레이어 (라우팅 후, DI 컨테이너 활용 가능)

Guard가 Pipe보다 먼저 실행되는 이유:
  인증 안 된 요청에 데이터 변환·검증 비용을 쓸 필요 없음
  DB 조회가 필요한 Pipe가 있다면 더욱 중요
```

```typescript
// Guard의 기본 구조
import { CanActivate, ExecutionContext, Injectable } from '@nestjs/common';

@Injectable()   // ← DI 컨테이너에 등록 (JwtService 주입 받으려면 필수)
export class MyGuard implements CanActivate {
  canActivate(context: ExecutionContext): boolean {
    // 여기서 요청을 검사하고
    // true(통과) 또는 false(거부) 또는 throw(예외) 반환
    return true;
  }
}
```

---

# ExecutionContext — 현재 요청 정보의 창구 ⭐️⭐️⭐️⭐️

```typescript
canActivate(context: ExecutionContext): boolean {
  // HTTP 요청 객체 (Express Request)
  const request = context.switchToHttp().getRequest();
  //    ↑ request.headers, request.user, request.body, request.cookies 등

  // 현재 실행될 라우트 메서드 자체
  const handler = context.getHandler();
  // ex) PostsController의 create 메서드 함수

  // 현재 실행될 컨트롤러 클래스 자체
  const cls = context.getClass();
  // ex) PostsController 클래스

  // Reflector가 메타데이터를 읽을 때 이 두 가지를 씀
  // getHandler() → 메서드에 붙은 @Public() 확인
  // getClass()   → 클래스에 붙은 @Public() 확인
}
```

```txt
switchToHttp():
  NestJS는 HTTP뿐 아니라 WebSocket, gRPC 등도 지원
  switchToHttp()로 "이건 HTTP 요청"이라고 명시
  → getRequest()로 Express Request 객체 반환

getRequest()로 접근할 수 있는 것:
  request.headers.authorization  → JWT 토큰
  request.body                   → POST/PATCH body
  request.params                 → URL 파라미터
  request.query                  → 쿼리스트링
  request.cookies                → 쿠키
  request.user                   → Guard가 저장한 payload
```

---
# 메타데이터 · SetMetadata · Reflector ⭐️⭐️⭐️⭐️

## 메타데이터란

```txt
메타데이터 = "이 함수·클래스에 대한 부가 정보"
  코드 실행에 직접 영향을 주지 않지만
  다른 곳(Guard 등)에서 읽어서 동작을 결정하는 데 씀

  SetMetadata('isPublic', true)
  = "이 메서드/클래스에 isPublic=true라는 꼬리표를 붙인다"

  Guard가 나중에 Reflector로 그 꼬리표를 읽어서
  "isPublic이 true구나, 그럼 토큰 검증 건너뛰자" 결정

비유:
  SetMetadata = 파일에 스티커 붙이기 ("공개", "관리자 전용")
  Reflector   = 스티커를 읽는 사람 (Guard)
  → Guard가 스티커를 보고 통과/거부 결정

@Public()·@Roles()·@AllowWithdrawing 전부 SetMetadata로 만들어짐
→ "내장처럼 보이지만 직접 만드는 것"의 핵심 도구
```

## SetMetadata — 메타데이터 붙이기

```typescript
// SetMetadata('키', 값) 형태로 메서드·클래스에 붙임
@Get()
@Public()           // 실제로는: SetMetadata('isPublic', true)
findAll() { ... }

// TypeScript Reflect API를 NestJS가 래핑한 것
// Reflect.defineMetadata('isPublic', true, findAll_메서드)
```

## Reflector.getAllAndOverride — 메타데이터 읽기

```typescript
const isPublic = this.reflector.getAllAndOverride<boolean>(IS_PUBLIC_KEY, [
  context.getHandler(),  // 메서드(findAll)에 붙은 메타데이터
  context.getClass(),    // 클래스(PostsController)에 붙은 메타데이터
]);
```

```txt
getAllAndOverride(key, [handler, class]):
  handler(메서드)를 먼저 확인
  → 있으면 즉시 반환
  → 없으면 class(컨트롤러)를 확인
  → 둘 다 없으면 undefined 반환 → falsy → 토큰 검증 진행

왜 handler와 class 두 곳을 확인하는가:
  @Public()을 메서드에 붙이면 → 그 메서드만 공개
  @Public()을 클래스에 붙이면 → 컨트롤러 전체 공개
```


```typescript
// 세 가지 Reflector 메서드 비교
this.reflector.get<boolean>(IS_PUBLIC_KEY, context.getHandler())
// handler 하나만 확인

this.reflector.getAllAndOverride<boolean>(IS_PUBLIC_KEY, [handler, class])
// handler 먼저, 없으면 class → 가장 많이 씀

this.reflector.getAllAndMerge<string[]>(ROLES_KEY, [handler, class])
// 둘 다 배열로 합침 — @Roles에서 활용
// @Roles('user') + @Roles('admin') → ['user', 'admin']
```

---

# JwtAuthGuard — 완전한 구현 ⭐️⭐️⭐️⭐️

```typescript
// auth/jwt-auth.guard.ts
import {
  CanActivate,
  ExecutionContext,
  Injectable,
  UnauthorizedException,
} from '@nestjs/common';
import { Reflector }  from '@nestjs/core';
import { JwtService } from '@nestjs/jwt';
import { IS_PUBLIC_KEY } from './decorators/public.decorator';

@Injectable()
export class JwtAuthGuard implements CanActivate {
  constructor(
    private readonly jwtService: JwtService,   // JWT 검증
    private readonly reflector:  Reflector,    // 메타데이터 읽기
  ) {}

  canActivate(context: ExecutionContext): boolean {
    // ① @Public() 확인 — 공개 라우트면 토큰 없이 통과
    const isPublic = this.reflector.getAllAndOverride<boolean>(IS_PUBLIC_KEY, [
      context.getHandler(),  // 메서드 레벨 메타데이터
      context.getClass(),    // 클래스 레벨 메타데이터
    ]);
    if (isPublic) return true;

    // ② Authorization 헤더에서 Bearer 토큰 추출
    const request = context.switchToHttp().getRequest();
    const token   = this.extractBearerToken(request);
    if (!token) throw new UnauthorizedException('토큰이 없습니다.');

    // ③ 토큰 검증 후 payload를 request.user에 저장
    try {
      const payload = this.jwtService.verify(token);
      request.user  = payload;
      // request.user = { sub: 'user-uuid', iat: 1700000000, exp: 1700003600 }
    } catch {
      throw new UnauthorizedException('유효하지 않은 토큰입니다.');
    }

    return true;
  }

  private extractBearerToken(request: any): string | undefined {
    const [type, token] = request.headers.authorization?.split(' ') ?? [];
    return type === 'Bearer' && token ? token : undefined;
    // "Bearer eyJ..." → type='Bearer', token='eyJ...'
    // type이 Bearer가 아니면 undefined
  }
}
```

---

# 전역 적용 — APP_GUARD ⭐️⭐️⭐️

```typescript
// app.module.ts
import { APP_GUARD } from '@nestjs/core';

@Module({
  imports:   [AuthModule, ...],   // AuthModule이 JwtModule을 export해야 함
  providers: [
    {
      provide:  APP_GUARD,
      useClass: JwtAuthGuard,
    },
  ],
})
export class AppModule {}
```

```txt
APP_GUARD:
  NestJS가 제공하는 Provider 토큰
  이 토큰으로 등록한 Guard는 모든 라우트에 자동 적용

전역 등록 후의 규칙:
  토큰 없이 요청 → 자동 401
  @Public() 붙은 라우트 → 통과
  @Public() 없는 라우트 → 토큰 필수

AuthModule 설정 필요:
  JwtAuthGuard가 JwtService를 주입받으려면
  JwtModule이 AuthModule에서 export 되어야 함
  → [[NestJS_Auth]] AuthModule 설정 참고
```

---

# 파이프라인 전체 조립 ⭐️⭐️⭐️⭐️

```txt
Guard를 만들어서 실제로 사용하기까지:

① Guard 클래스 작성
   jwt-auth.guard.ts
   CanActivate 구현, @Injectable() 붙이기

② 데코레이터 작성
   public.decorator.ts (@Public)
   user-id.decorator.ts (@UserId)

③ AuthModule에서 JwtModule export
   다른 모듈에서 JwtService를 쓸 수 있게

④ AppModule에서 APP_GUARD 등록
   전역 적용

⑤ 컨트롤러에서 사용
   @Public(), @UserId() 사용
```

```typescript
// 파일 구조
src/
├── auth/
│   ├── auth.module.ts          ← JwtModule 설정 + export
│   ├── auth.service.ts         ← login, issueTokens
│   ├── auth.controller.ts      ← /auth/login, /auth/register
│   ├── jwt-auth.guard.ts       ← JwtAuthGuard 클래스
│   └── decorators/
│       ├── public.decorator.ts    ← @Public()
│       ├── user-id.decorator.ts   ← @UserId()
│       └── roles.decorator.ts     ← @Roles()
└── app.module.ts               ← APP_GUARD 전역 등록
```

---

# @Public() 만드는 법 ⭐️⭐️⭐️⭐️

```typescript
// auth/decorators/public.decorator.ts
import { SetMetadata } from '@nestjs/common';  // NestJS 제공

// IS_PUBLIC_KEY: 메타데이터를 읽을 때 사용할 키
export const IS_PUBLIC_KEY = 'isPublic';

// @Public() = SetMetadata('isPublic', true)의 단축 데코레이터
export const Public = () => SetMetadata(IS_PUBLIC_KEY, true);
```


---

# @UserId() 만드는 법 ⭐️⭐️⭐️⭐️

```typescript
// auth/decorators/user-id.decorator.ts
import { createParamDecorator, ExecutionContext } from '@nestjs/common';

export const UserId = createParamDecorator(
  // _data: @UserId('something') 처럼 인자 전달 시 사용 (여기선 사용 안 함)
  (_data: unknown, ctx: ExecutionContext): string => {
    const request = ctx.switchToHttp().getRequest();
    return request.user.sub;
    // JwtAuthGuard가 request.user = { sub: 'userId', ... } 저장했음
  },
);
```

```typescript
// 사용
@Post()
create(
  @UserId()    userId: string,  // request.user.sub 값이 주입됨
  @Body() dto: CreatePostDto,
) {
  return this.postsService.create(userId, dto);
}
```

---

# OptionalJwtAuthGuard + @OptionalUserId() ⭐️⭐️⭐️⭐️

```txt
"로그인 안 해도 되지만, 로그인했다면 userId를 알고 싶다"

예시:
  게시글 피드 — 비로그인도 볼 수 있지만
  로그인했으면 친구 피드 / 좋아요 여부 / 북마크 여부를 함께 표시
  → 토큰 있으면 검증, 없으면 그냥 통과
  → 토큰이 있지만 유효하지 않아도 통과 (게스트로 취급)
```

## OptionalJwtAuthGuard — 실제 구현

```typescript
// auth/optional-jwt-auth.guard.ts
import { CanActivate, ExecutionContext, Injectable } from '@nestjs/common';
import { ConfigService }  from '@nestjs/config';
import { JwtService }     from '@nestjs/jwt';
import { Request }        from 'express';
import { EnvKeys }        from 'src/config/env.keys';
import type { JwtPayload } from './jwt-payload';
import { AuthService }    from './auth.service';

@Injectable()
export class OptionalJwtAuthGuard implements CanActivate {
  constructor(
    private readonly configService: ConfigService,
    private readonly jwtService:    JwtService,
    private readonly authService:   AuthService,
  ) {}

  private extractBearerToken(request: Request): string | undefined {
    const [type, token] = request.headers.authorization?.split(' ') ?? [];
    return type === 'Bearer' && token ? token : undefined;
  }

  async canActivate(context: ExecutionContext): Promise<boolean> {
    const request = context.switchToHttp().getRequest<Request>();
    const token   = this.extractBearerToken(request);

    if (!token) return true;  // 토큰 없음 = 게스트 → 통과

    try {
      const payload = await this.jwtService.verifyAsync<JwtPayload>(token, {
        secret: this.configService.getOrThrow(EnvKeys.API_JWT_SECRET),
      });
      request.user = payload;

      // 로그인 확인 시 마지막 활동 시각 갱신 (부수 효과 — 실패해도 무시)
      this.authService.touchLastActiveAtFromGuard(payload.sub, payload.role);
    } catch {
      // 잘못된 토큰 = 게스트로 취급 → 그냥 통과
      // request.user는 undefined 상태로 유지
    }

    return true;
  }
}
```

```txt
단순 버전(JwtService만)과 실제 버전의 차이:

  ConfigService 주입:
    verifyAsync에 secret을 명시적으로 전달
    JwtModule 설정과 별개로 직접 지정 (더 명시적)

  verifyAsync vs verify:
    verify()      → 동기 — canActivate가 boolean 반환
    verifyAsync() → 비동기 — canActivate가 Promise<boolean> 반환
    async 메서드 안에서 await 쓰려면 verifyAsync 필요

  AuthService (touchLastActiveAtFromGuard):
    로그인이 확인됐을 때 마지막 활동 시각을 DB에 기록
    await 없이 호출 → fire-and-forget (실패해도 요청 영향 없음)
    컨트롤러마다 따로 안 써도 Guard 통과 시 자동 처리

  catch에서 throw 안 하는 이유:
    토큰 만료·조작·형식 오류 → 게스트로 그냥 통과
    "잘못된 토큰 = 401"이 아니라 "잘못된 토큰 = 게스트 취급"
    서비스에서 viewerId가 undefined면 게스트 응답 반환
```

## @OptionalUserId() 데코레이터

```typescript
// auth/decorators/optional-user-id.decorator.ts
export const OptionalUserId = createParamDecorator(
  (_data: unknown, ctx: ExecutionContext): string | undefined => {
    return ctx.switchToHttp().getRequest().user?.sub;
    // 로그인 → payload.sub (userId)
    // 비로그인 → request.user undefined → undefined 반환
  },
);
```

## 사용

```typescript
@Get()
@UseGuards(OptionalJwtAuthGuard)       // APP_GUARD 대신 이 라우트에만 적용
findAll(
  @Query()          query:     ListPostsQueryDto,
  @OptionalUserId() viewerId?: string, // 비로그인이면 undefined
) {
  return this.postsService.findAll(query, viewerId);
}
```

```txt
서비스에서 viewerId로 분기:
  if (!viewerId) → 전체 공개 피드
  if (viewerId)  → 친구 피드 / 개인화 피드
```

## AuthModule에 등록하는 방법

```typescript
// auth/auth.module.ts
@Module({
  providers: [
    AuthService,
    OptionalJwtAuthGuard,         // providers에 등록 (APP_GUARD 아님)
    {
      provide:  APP_GUARD,
      useClass: JwtAuthGuard,     // JwtAuthGuard만 전역 APP_GUARD
    },
  ],
  exports: [
    AuthService,
    OptionalJwtAuthGuard,         // 다른 모듈에서 @UseGuards로 쓸 수 있게 export
    JwtModule,
  ],
})
export class AuthModule {}
```

```txt
JwtAuthGuard vs OptionalJwtAuthGuard 등록 방식 차이:

  JwtAuthGuard:
    APP_GUARD로 등록 → 전체 앱의 모든 라우트에 자동 적용
    → 따로 @UseGuards 안 써도 됨

  OptionalJwtAuthGuard:
    providers에 일반 클래스로 등록 → APP_GUARD 아님
    → 필요한 라우트에만 @UseGuards(OptionalJwtAuthGuard) 직접 붙여야 함
    exports에도 추가해야 다른 모듈(PostsModule 등)에서 @UseGuards로 사용 가능

왜 OptionalJwtAuthGuard를 APP_GUARD로 안 쓰는가:
  APP_GUARD로 등록하면 전체 라우트에 적용됨
  하지만 OptionalJwtAuthGuard는 "일부 라우트"에만 게스트 허용이 목적
  → 전역으로 걸면 모든 라우트가 게스트 허용 → 의도와 다름
  → 라우트마다 명시적으로 @UseGuards(OptionalJwtAuthGuard) 붙이는 게 맞음
```

---

# @Roles() — 역할 기반 접근 제어 ⭐️⭐️⭐

## 단일 role vs roles[] — 헷갈리는 이유 ⭐️⭐️⭐️⭐️

```txt
Nest 공식 문서 예시와 실제 프로젝트가 달라 보이는데, 둘 다 맞음
"유저가 역할을 어떻게 갖는지"가 다른 것

Nest 문서 방식:                      이 프로젝트 방식:
user.roles = ['admin', 'editor']      user.role = 'admin'
역할을 배열로 여러 개                  역할이 하나 (enum)

문서 방식 검사:
  required.some(role => user?.roles?.includes(role))
  → 필요 역할 중 하나라도 유저 배열에 있으면 통과

이 프로젝트 검사:
  requiredRoles.includes(userRole)
  → 유저의 단일 role이 필요 목록에 있으면 통과

⚠️ 문서 예시를 그대로 복붙하면:
  user.roles가 없어서 항상 실패 — 조심
  JwtPayload가 role: 'user' | 'admin' 단일이므로
  단일 검사가 맞음
```

## 데코레이터

```typescript
// auth/decorators/roles.decorator.ts
import { SetMetadata } from '@nestjs/common';
export const ROLES_KEY = 'roles';
export type Role = 'user' | 'admin';
export const Roles = (...roles: Role[]) => SetMetadata(ROLES_KEY, roles);
```

## RolesGuard — 단일 role 방식

```typescript
// auth/roles.guard.ts
import {
  CanActivate, ExecutionContext, ForbiddenException, Injectable,
} from '@nestjs/common';
import { Reflector }        from '@nestjs/core';
import { Role, ROLES_KEY }  from './decorators/roles.decorator';
import type { Request }     from 'express';

@Injectable()
export class RolesGuard implements CanActivate {
  constructor(private readonly reflector: Reflector) {}

  canActivate(context: ExecutionContext): boolean {
    const requiredRoles = this.reflector.getAllAndOverride<Role[]>(ROLES_KEY, [
      context.getHandler(),
      context.getClass(),
    ]);

    // @Roles() 없는 라우트는 통과
    if (!requiredRoles?.length) return true;

    // request.user.role — 단일 role
    const request  = context.switchToHttp().getRequest<Request>();
    const userRole = request.user?.role;

    if (!userRole || !requiredRoles.includes(userRole)) {
      throw new ForbiddenException('권한이 없습니다.');
    }
    return true;
  }
}
```

```txt
ForbiddenException vs return false:
  return false → 403 Forbidden (NestJS 기본 메시지)
  ForbiddenException → "권한이 없습니다." 직접 지정 가능 → 권장

나중에 다중 역할로 바꾸려면:
  user.role → user.roles 배열로 변경
  requiredRoles.includes(userRole)
  → required.some(role => user.roles.includes(role))
```

## 사용

```typescript
@Delete('admin/:id')
@Roles('admin')
adminDelete(@Param('id', ParseUUIDPipe) id: string) { ... }
```

---
# APP_GUARD vs @UseGuards — 언제 뭘 쓰는가 ⭐️⭐️⭐️⭐️

```txt
핵심:
  APP_GUARD    = 전역으로 켜둠 (모든 라우트에 기본 적용)
  @UseGuards() = 이 라우트에만 켜기

  둘 다 "Guard를 걸어라"인데 거는 범위가 다름
```

## 한눈에 비교

|방식|적용 범위|언제|
|---|---|---|
|`APP_GUARD + JwtAuthGuard`|전체 앱 (기본 로그인 필수)|기본이 "로그인 필수"인 앱. 공개 라우트만 `@Public()`으로 예외|
|`@UseGuards(JwtAuthGuard)`|특정 컨트롤러·메서드|JWT를 전역으로 안 걸었을 때 보호할 라우트에 개별 추가|
|`@UseGuards(JwtAuthGuard, RolesGuard)`|특정 컨트롤러·메서드|전역 JWT가 없을 때 "로그인 + 역할" 한 곳에 명시|
|`@UseGuards(RolesGuard)` + `@Roles('admin')`|특정 컨트롤러·메서드|이미 APP_GUARD로 JWT가 전역이면 역할만 추가|
|`APP_GUARD + RolesGuard`|전체 앱|`@Roles()` 있는 라우트만 검사, 없으면 자동 통과|

## 프로젝트 패턴 — APP_GUARD로 전역 JWT

```typescript
// app.module.ts
providers: [
  { provide: APP_GUARD, useClass: JwtAuthGuard },  // 전체 앱 로그인 필수
  { provide: APP_GUARD, useClass: RolesGuard },    // @Roles() 있는 곳만 역할 검사
]
```

```txt
이렇게 하면:
  모든 라우트 → JwtAuthGuard 통과해야 함 (로그인 필수)
  @Roles('admin') 붙은 라우트 → RolesGuard도 통과해야 함
  @Public() 붙은 라우트 → JwtAuthGuard 건너뜀 (비로그인 가능)
  OptionalJwtAuthGuard → 개별 @UseGuards로 따로 붙임
```

## 헷갈리는 경우별 답 ⭐️⭐️⭐️⭐️

```txt
Q: 비로그인도 볼 수 있는 라우트는?
A: @Public() 붙이면 됨
   → JwtAuthGuard가 @Public() 확인하고 건너뜀

Q: 비로그인은 기본 응답, 로그인하면 개인화 응답을 주려면?
A: @UseGuards(OptionalJwtAuthGuard) + @OptionalUserId()
   → APP_GUARD가 아니라 이 라우트에만 Optional Guard 적용

Q: 어드민만 접근 가능한 라우트는?
A: @Roles('admin') 붙이면 됨 (APP_GUARD에 RolesGuard 전역 등록된 경우)
   → 로그인 여부는 JwtAuthGuard가, 역할은 RolesGuard가 각각 처리

Q: APP_GUARD + RolesGuard를 등록했는데 @Roles() 없는 일반 라우트는?
A: RolesGuard가 @Roles() 없으면 자동 통과 (내부에서 requiredRoles 없으면 return true)
   → 전역으로 걸어도 역할 없는 라우트는 영향 없음
```
---

# 실전 컨트롤러 패턴 ⭐️⭐️⭐️⭐️

```typescript
@ApiTags('posts')
@ApiBearerAuth('access-token')
@Controller('posts')
export class PostsController {

  // 비로그인 가능 — @Public()
  @Get()
  @Public()
  findAll(@Query() query: ListPostsQueryDto) { ... }

  // 비로그인 가능 + 로그인하면 개인화 — OptionalJwtAuthGuard
  @Get(':id')
  @UseGuards(OptionalJwtAuthGuard)
  findOne(
    @Param('id', ParseUUIDPipe) id: string,
    @OptionalUserId()           viewerId?: string,
  ) { ... }

  // 로그인 필수 — 기본값 (APP_GUARD가 처리)
  @Post()
  @HttpCode(HttpStatus.CREATED)
  create(
    @UserId()    userId: string,
    @Body() dto: CreatePostDto,
  ) { ... }

  // 로그인 + 본인만 — 서비스에서 userId로 소유권 검증
  @Patch(':id')
  update(
    @UserId()                   userId: string,
    @Param('id', ParseUUIDPipe) id:     string,
    @Body() dto:                UpdatePostDto,
  ) { ... }

  // 어드민만 — @Roles
  @Delete('admin/:id')
  @Roles('admin')
  adminDelete(@Param('id', ParseUUIDPipe) id: string) { ... }
}
```

---

# 자주 만나는 에러

| 에러                              | 원인                    | 해결                                     |
| ------------------------------- | --------------------- | -------------------------------------- |
| `401 Unauthorized`              | 토큰 없음 또는 만료           | 로그인 후 유효한 Access Token 첨부              |
| `403 Forbidden`                 | 토큰 유효하지만 권한 없음        | @Roles 확인, 또는 서비스에서 소유권 검증             |
| `request.user` undefined        | Guard 없이 @UserId() 사용 | APP_GUARD 등록 확인 / AuthModule export 확인 |
| @Public()이 안 먹힘                 | IS_PUBLIC_KEY 불일치     | decorator의 key와 guard의 key가 같은지 확인     |
| `Nest can't resolve JwtService` | JwtModule이 export 안 됨 | AuthModule에서 `exports: [JwtModule]` 추가 |