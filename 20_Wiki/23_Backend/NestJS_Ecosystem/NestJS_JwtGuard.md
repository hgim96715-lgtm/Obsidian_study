---
aliases: [Auth, Guard, JWT, JwtAuthGuard, request.user, RolesGuard]
tags: [NestJS]
related:
  - "[[00_NestJS_Ecosystem_HomePage]]"
  - "[[Auth_Concept]]"
  - "[[NestJS_Bcrypt]]"
  - "[[NodeJS_HTTP_Request]]"
  - "[[NodeJS_Passport]]"
---
# NestJS_JwtGuard — JWT 인증 파이프라인

> [!info] 
> Guard 인프라(CanActivate · Reflector · SetMetadata)는 NestJS가 제공하고, 
> 실제 검증 로직(JwtAuthGuard · @Roles · @Public · @UserId 등)은 직접 만든다. 이 둘을 구분하는 것이 이 노트의 핵심.

---

# NestJS가 제공하는 것 vs 직접 만드는 것 ⭐️⭐️⭐️⭐️

```txt
헷갈림의 근본 원인:
  @Roles · @Public · @AllowWithdrawing · JwtAuthGuard
  → 전부 "직접 만드는 것"인데 예제마다 당연하게 나와서 내장인 것처럼 보임

  내장처럼 보이는 이유:
  NestJS가 "조립 도구"(SetMetadata, Reflector, CanActivate 등)를 제공하고
  개발자가 그 도구로 만든 패턴이 업계 관례로 굳어져서
  어디서나 같은 이름, 같은 모양으로 나오는 것
```

|                            | 제공 주체         | 역할                                    |
| -------------------------- | ------------- | ------------------------------------- |
| `CanActivate`              | NestJS        | Guard가 구현해야 하는 인터페이스                  |
| `ExecutionContext`         | NestJS        | 현재 요청의 handler·class·request에 접근하는 객체 |
| `SetMetadata(key, value)`  | NestJS        | 클래스/메서드에 "라벨"을 붙이는 함수                 |
| `Reflector`                | NestJS        | 붙인 라벨을 다시 읽는 헬퍼                       |
| `APP_GUARD`                | NestJS        | 전역 Guard 등록 토큰                        |
| `@UseGuards()`             | NestJS        | 특정 Guard를 라우트에 적용하는 데코레이터             |
| `createParamDecorator()`   | NestJS        | 커스텀 파라미터 데코레이터를 만드는 함수                |
| `JwtModule` · `JwtService` | `@nestjs/jwt` | JWT 발급(signAsync) · 검증(verifyAsync)   |
| **`JwtAuthGuard`**         | **직접 만듦**     | Bearer 토큰 검증, request.user 채우기        |
| **`@Public()`**            | **직접 만듦**     | 인증 건너뛰기 라벨 (SetMetadata 래퍼)           |
| **`@Roles()`**             | **직접 만듦**     | role 체크 라벨 (SetMetadata 래퍼)           |
| **`RolesGuard`**           | **직접 만듦**     | @Roles 라벨을 읽어서 role 검사                |
| **`@UserId()`**            | **직접 만듦**     | request.user.sub를 꺼내는 파라미터 데코레이터      |
| **`@AllowWithdrawing()`**  | **직접 만듦**     | 특정 상태 예외 라벨 (SetMetadata 래퍼)          |

---

# Guard — 요청을 가로막는 문지기 ⭐️⭐️⭐️

```typescript
// Guard = CanActivate 인터페이스를 구현하는 클래스
// canActivate가 true → 통과 / false (또는 throw) → 차단

@Injectable()
export class SomeGuard implements CanActivate {
  canActivate(context: ExecutionContext): boolean | Promise<boolean> {
    const request = context.switchToHttp().getRequest();
    // 여기서 request를 보고 통과시킬지 막을지 결정
    return true; // 또는 false, 또는 throw new UnauthorizedException()
  }
}
```

```txt
ExecutionContext — 현재 요청에 대한 모든 정보의 창구:
  context.switchToHttp().getRequest()  → Express Request 객체 (토큰, body 등)
  context.getHandler()                 → 지금 실행될 라우트 메서드 자체
  context.getClass()                   → 지금 실행될 컨트롤러 클래스 자체

getHandler() · getClass()는 Reflector로 메타데이터를 읽을 때 씀 (아래 참고)
```

---

# SetMetadata + Reflector — 공통 메커니즘 ⭐️⭐️⭐️⭐️

```txt
@Public · @Roles · @AllowWithdrawing — 이 셋이 전부 같은 방식으로 작동함
먼저 이 메커니즘을 이해하면, 새 데코레이터가 나와도 "아, 이 패턴이구나"가 됨
```

```txt
메커니즘 두 단계:
  1. SetMetadata(key, value)  →  클래스/메서드에 "라벨"을 저장해둠
  2. Reflector                →  Guard 실행 시 그 라벨을 다시 읽어옴

  라벨을 붙인다고 해서 실행 흐름이 바로 바뀌는 게 아님
  Guard가 Reflector로 "이 라우트에 어떤 라벨이 붙었나?"를 확인하고 분기하는 것
```

```typescript
// 라벨 붙이기 — SetMetadata 래퍼 커스텀 데코레이터
export const IS_PUBLIC_KEY = 'isPublic';
export const Public = () => SetMetadata(IS_PUBLIC_KEY, true);
//                                       ↑ key          ↑ value

export const ROLES_KEY = 'roles';
export const Roles = (...roles: Role[]) => SetMetadata(ROLES_KEY, roles);
```

```typescript
// 라벨 읽기 — Guard 안에서 Reflector 사용
const isPublic = this.reflector.getAllAndOverride<boolean>(IS_PUBLIC_KEY, [
  context.getHandler(),  // 메서드에 붙은 라벨 먼저 확인
  context.getClass(),    // 없으면 클래스에 붙은 라벨 확인
]);
```

```txt
getAllAndOverride:
  이름 때문에 "합친다(All)"로 오해하기 쉬운데
  "여러 곳을 확인(All)하되, 더 구체적인 쪽(메서드)이 있으면 그것만 쓰고 나머지는 버린다(Override)"
  → 메서드에 @Roles('admin')이 있으면 클래스 레벨 @Roles는 무시

  여러 값을 합치고 싶으면 getAllAndMerge 사용
```

---

# 파이프라인 조립

## 1단계 — JwtAuthGuard (인증 · Bearer 토큰 검증)

```typescript
// jwt-auth.guard.ts
@Injectable()
export class JwtAuthGuard implements CanActivate {
  constructor(
    private jwtService: JwtService,
    private configService: ConfigService,
    private reflector: Reflector,
  ) {}

  async canActivate(context: ExecutionContext): Promise<boolean> {
    // @Public() 라벨이 있으면 토큰 없이 통과
    const isPublic = this.reflector.getAllAndOverride<boolean>(IS_PUBLIC_KEY, [
      context.getHandler(),
      context.getClass(),
    ]);
    if (isPublic) return true;

    // Bearer 토큰 추출
    const request = context.switchToHttp().getRequest();
    const token = this.extractToken(request);
    if (!token) throw new UnauthorizedException();

    // 서명·만료 검증 → 통과하면 payload 반환
    try {
      const payload = await this.jwtService.verifyAsync<JwtPayload>(token, {
        secret: this.configService.getOrThrow('JWT_SECRET'),
      });
      request.user = payload;  // ← 내가 직접 채우는 부분
    } catch {
      throw new UnauthorizedException();
    }
    return true;
  }

  private extractToken(request: Request): string | undefined {
    const [type, token] = request.headers.authorization?.split(' ') ?? [];
    return type === 'Bearer' && token ? token : undefined;
  }
}
```

```txt
request.user = payload 를 직접 써야 하는 이유:
  request.user는 Express 기본 타입에 없는 속성
  Passport를 쓰면 Passport가 자동으로 채워주지만
  Passport 없이 직접 구현하면 내가 한 줄 써야 함
  → 이름은 관례(Passport에서 온 convention)를 따른 것, NestJS 강제 사항 아님
```

## 2단계 — request.user 타입 확장

```typescript
// src/types/express.d.ts
import type { JwtPayload } from '../auth/jwt-payload';

declare global {
  namespace Express {
    interface Request {
      user?: JwtPayload;
    }
  }
}

export {};
```

```txt
이 파일이 필요한 이유:
  TypeScript는 request.user가 뭔지 모름 → 타입 에러 또는 any
  Express의 Request 인터페이스에 Declaration Merging으로 user 필드를 추가

  이 파일 = "타입만 알려주는 것"
  실제 값을 넣는 건 JwtAuthGuard의 request.user = payload

tsconfig의 include에 이 파일 경로가 포함돼야 적용됨
```

## 3단계 — @UserId() 커스텀 파라미터 데코레이터

```typescript
// user-id.decorator.ts
export const UserId = createParamDecorator(
  (_data: unknown, ctx: ExecutionContext): string => {
    const request = ctx.switchToHttp().getRequest<Request>();
    const userId = request.user?.sub;
    if (!userId) throw new UnauthorizedException('로그인이 필요합니다.');
    return userId;
  },
);
```

```typescript
// 사용 — @Req() 받아서 req.user.sub 꺼내는 반복을 없앰
@Get('me')
getMe(@UserId() userId: string) {
  return this.usersService.findOne(userId);
}
```

## 4단계 — 인가 패턴들

```txt
인증(누구야?) 은 JwtAuthGuard가 끝냄
인가(해도 돼?) 는 아래 패턴들이 담당 — 전부 SetMetadata+Reflector 메커니즘

패턴이 3개지만 동작 방식은 완전히 같음:
  라벨을 붙이는 커스텀 데코레이터 → Guard가 Reflector로 라벨을 읽어서 분기
```

### @Public() — 인증 자체를 건너뜀

```typescript
// public.decorator.ts
export const IS_PUBLIC_KEY = 'isPublic';
export const Public = () => SetMetadata(IS_PUBLIC_KEY, true);
```

```typescript
// 읽는 곳: JwtAuthGuard 안 (위 1단계 코드 참고)
// 라벨이 있으면 토큰 없이 통과

@Public()
@Post('login')
login(@Body() dto: LoginDto) { ... }  // 로그인은 토큰 없이 접근 가능
```

### @Roles() + RolesGuard — role 기반 인가

```typescript
// roles.decorator.ts
export const ROLES_KEY = 'roles';
export type Role = 'user' | 'admin';
export const Roles = (...roles: Role[]) => SetMetadata(ROLES_KEY, roles);

// roles.guard.ts
@Injectable()
export class RolesGuard implements CanActivate {
  constructor(private readonly reflector: Reflector) {}

  canActivate(context: ExecutionContext): boolean {
    const requiredRoles = this.reflector.getAllAndOverride<Role[]>(ROLES_KEY, [
      context.getHandler(),
      context.getClass(),
    ]);
    if (!requiredRoles?.length) return true;  // @Roles() 없는 라우트는 role 제한 없음

    const { user } = context.switchToHttp().getRequest<Request>();
    if (!user || !requiredRoles.includes(user.role)) {
      throw new ForbiddenException('권한이 없습니다.');
    }
    return true;
  }
}
```

```typescript
@Roles('admin')
@Delete(':id')
remove(@Param('id') id: string) { ... }  // admin만 접근 가능
```

```txt
⚠️ ROLES_KEY 오타 주의:
  ROLES_KEY를 다른 파일에서 import할 때 오타(ROLLES_KEY 등)가 나면
  Reflector가 메타데이터를 못 찾아서 requiredRoles가 항상 undefined
  → !requiredRoles?.length 가 항상 true → 모든 role이 통과되는 조용한 버그
  → export하는 쪽과 import하는 쪽이 같은 상수를 참조하는지 확인
```

### @AllowWithdrawing() — 사용자 상태 기반 예외

```typescript
// allow-withdrawing.decorator.ts — 직접 만드는 것, 내장 아님
export const ALLOW_WITHDRAWING_KEY = 'allowWithdrawing';
export const AllowWithdrawing = () => SetMetadata(ALLOW_WITHDRAWING_KEY, true);

// withdrawing.guard.ts
@Injectable()
export class WithdrawingGuard implements CanActivate {
  constructor(private readonly reflector: Reflector) {}

  canActivate(context: ExecutionContext): boolean {
    const allow = this.reflector.getAllAndOverride<boolean>(
      ALLOW_WITHDRAWING_KEY,
      [context.getHandler(), context.getClass()],
    );
    if (allow) return true;  // @AllowWithdrawing()이 있으면 통과

    const { user } = context.switchToHttp().getRequest<Request>();
    if (user?.status === 'withdrawing') {
      throw new ForbiddenException('탈퇴 처리 중인 계정입니다.');
    }
    return true;
  }
}
```

```typescript
@Controller('me')
export class MeController {
  @AllowWithdrawing()
  @Get()
  getMe() {}           // 탈퇴 유예 중에도 내 정보 조회 가능

  @AllowWithdrawing()
  @Post('withdraw/cancel')
  cancelWithdraw() {}  // 탈퇴 취소도 허용

  @Get('profile')
  getProfile() {}      // @AllowWithdrawing() 없음 → 탈퇴 유예 중이면 403
}
```

```txt
세 가지 패턴 비교:

  @Public()          → JwtAuthGuard가 읽음  → 인증 자체를 건너뜀
  @Roles()           → RolesGuard가 읽음    → role 기반 인가
  @AllowWithdrawing() → WithdrawingGuard가 읽음 → 상태 기반 예외 허용

  동작 방식은 완전히 동일: SetMetadata로 라벨 → Guard가 Reflector로 읽어서 분기
  "어떤 라벨을", "어떤 Guard가 읽는지"만 다름
```

---

# 전역 적용 — APP_GUARD ⭐️⭐️⭐️

```typescript
// auth.module.ts
@Module({
  providers: [
    { provide: APP_GUARD, useClass: JwtAuthGuard },    // 모든 라우트에 인증
    { provide: APP_GUARD, useClass: WithdrawingGuard }, // 탈퇴 유예 체크
    { provide: APP_GUARD, useClass: RolesGuard },       // role 체크
  ],
})
export class AuthModule {}
```

```txt
APP_GUARD로 등록하면 @UseGuards()를 매 컨트롤러마다 안 붙여도 자동 적용

Guard 실행 순서: 등록 순서대로
  JwtAuthGuard   → request.user 채움 (인증)
  WithdrawingGuard → request.user 읽어서 탈퇴 상태 체크 (JwtAuthGuard 다음이어야 함)
  RolesGuard     → request.user 읽어서 role 체크 (JwtAuthGuard 다음이어야 함)

  JwtAuthGuard가 반드시 먼저 → 뒤따르는 Guard들이 request.user를 읽을 수 있음

전략: "기본적으로 전부 막고, 공개 라우트만 @Public()으로 예외 처리"
```

---

# JwtModule 설치 및 설정 ⭐️

```bash
pnpm add @nestjs/jwt --filter api
```

```typescript
// auth.module.ts
@Module({
  imports: [
    JwtModule.register({
      secret: process.env.JWT_SECRET,
      signOptions: { expiresIn: '15m' },
    }),
  ],
  providers: [AuthService, JwtAuthGuard],
})
export class AuthModule {}
```

```typescript
// JwtPayload — signAsync로 담는 것 = verifyAsync로 꺼내는 것
export type JwtPayload = {
  sub:  string;           // 표준 클레임 — userId를 담는 관례
  role: 'user' | 'admin'; // 커스텀 클레임 — 이 서비스가 직접 정의
};
```

```txt
JWT 구조: header.payload.signature

표준 클레임(RFC 7519에서 미리 정해둔 것):
  sub  누구에 대한 토큰인지 (보통 userId)
  exp  만료 시각 (signOptions.expiresIn이 자동 생성)
  iat  발급 시각

커스텀 클레임:
  role 같은 서비스 전용 필드 — 표준에 없고, 내가 직접 추가한 것

payload는 Base64 디코딩하면 그냥 JSON이 보임 — 민감한 정보(비밀번호 등) 넣지 않음
서명(signature)이 위변조를 막는 것이지, payload 자체가 암호화된 건 아님
```

---

# 자주 만나는 에러

| 증상                 | 원인                                                  | 해결                                      |
| ------------------ | --------------------------------------------------- | --------------------------------------- |
| 모든 라우트가 401        | JwtAuthGuard가 전역 등록됐는데 @Public() 없음                 | 로그인 라우트에 @Public() 추가                   |
| role 체크가 항상 통과     | ROLES_KEY 오타로 Reflector가 라벨을 못 찾음                   | 상수 import 경로와 철자 확인                     |
| request.user 타입 에러 | express.d.ts가 tsconfig include에 없음                  | tsconfig include 경로 확인                  |
| Guard 실행 순서 문제     | WithdrawingGuard나 RolesGuard가 JwtAuthGuard보다 먼저 등록됨 | APP_GUARD 등록 순서 확인                      |
| 401 대신 500이 나옴     | Guard 안에서 throw 대신 return false를 사용                 | 명시적으로 throw new UnauthorizedException() |