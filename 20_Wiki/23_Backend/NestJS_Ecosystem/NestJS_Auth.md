---
aliases:
  - 인증
  - OAuth
  - Passport
  - Social
tags:
  - NestJS
related:
  - "[[00_NestJS_Ecosystem_HomePage]]"
  - "[[Auth_Concept]]"
  - "[[Monorepo_PNPM]]"
  - "[[NestJS_JwtGuard]]"
  - "[[NestJS_Module]]"
  - "[[NextJS_TokenStorage]]"
  - "[[Monorepo_PNPM]]"
  - "[[TS_ImportType]]"
---
# NestJS_Auth — NestJS 인증 구현

> [!info] 
> NestJS에서 인증은 Passport 미들웨어로 처리한다. 
> 전략(Strategy)을 등록하고, Guard로 요청을 막고, 성공 시 JWT를 발급하는 구조다.

---
# 흐름도

```mermaid
flowchart TD
    W1["상황 · 이메일+비밀번호 로그인"]
    W2["상황 · 소셜 OAuth 로그인"]

    Q{"언제 · 어떤 로그인?"}

    W1 --> Q
    W2 --> Q

    Q -->|로컬| A
    Q -->|소셜| B

    A["LocalAuthGuard"]
    A1["LocalStrategy → bcrypt 검증"]
    A --> A1

    B["소셜 OAuth"]
    B1["① 시작 · session.next"]
    B2["Provider"]
    B3["② 콜백 · Strategy → Service"]
    B --> B1 --> B2 --> B3

    A1 --> T["issueTokens → JWT"]
    B3 --> T
    T --> F["프론트 토큰 저장"]

    F --> C
    C["이후 API · Bearer JWT"]
    C1["JwtAuthGuard → JwtStrategy"]
    C --> C1
```

> **infra** — `express-session`(OAuth state·next만) · `passport.initialize` · `session: false`  
> **레이어** — Strategy=프로필만 · Controller=HTTP·redirect · Service=병합·JWT  
> **Apple** — Strategy 없음 · POST callback에서 `id_token` 직접 검증

---

# 패키지 구조 ⭐️⭐️⭐️⭐️

```bash
# 공통 필수
pnpm add @nestjs/passport @nestjs/jwt passport passport-jwt
pnpm add -D @types/passport @types/passport-jwt

# 로컬 (ID/비밀번호)
pnpm add passport-local bcrypt
pnpm add -D @types/passport-local @types/bcrypt

# 소셜 로그인
pnpm add passport-google-oauth20
pnpm add -D @types/passport-google-oauth20

pnpm add passport-kakao
pnpm add passport-naver-v2

# Apple (전용 passport 전략 없음 — JWT 직접 검증)
pnpm add apple-signin-auth

# OAuth callback 중 세션 임시 저장 (선택)
pnpm add express-session
pnpm add -D @types/express-session
```

```txt
모노레포(apps/api 구조)에서는 --filter 플래그로 설치 대상 지정:
  pnpm --filter api add @nestjs/passport ...
  → package.json의 "name": "api"인 앱에만 설치

단일 프로젝트면 그냥 pnpm add로 충분
모노레포 설정 → [[Monorepo_PNPM]] 참고
```

|패키지|역할|
|---|---|
|`passport`|Node.js 인증 미들웨어 — 전략 패턴으로 다양한 인증 방식 지원|
|`@nestjs/passport`|NestJS에서 Passport를 DI로 사용하게 해주는 래퍼|
|`@nestjs/jwt`|JWT 생성/검증 모듈|
|`passport-jwt`|JWT를 Passport 전략으로 검증|
|`passport-local`|이메일/비밀번호 전략|
|`passport-google-oauth20`|Google OAuth 2.0 전략|
|`bcrypt`|비밀번호 해시|
|`express-session`|OAuth callback 사이 state 임시 보관 (선택적)|

---

# 모듈 구조 ⭐️⭐️⭐️

```txt
auth/
  auth.module.ts        AuthModule — 전략들 등록
  auth.controller.ts    /auth/login, /auth/google, /auth/google/callback 등
  auth.service.ts       사용자 검증, JWT 발급
  strategies/
    local.strategy.ts   이메일+비밀번호 검증
    jwt.strategy.ts     JWT 검증 (모든 인증 요청에 사용)
    google.strategy.ts  Google OAuth
    kakao.strategy.ts   Kakao OAuth
    naver.strategy.ts   Naver OAuth
    apple.strategy.ts   Apple Sign In
  guards/
    local-auth.guard.ts
    jwt-auth.guard.ts
    google-auth.guard.ts
```

---

# express-session — OAuth 플로우용 임시 세션 ⭐️⭐️⭐️⭐️

```txt
⚠️ 여기서 쓰는 세션은 "세션 인증"이 아님

  우리 서비스 인증 방식: JWT (영구 로그인 상태 유지)
  express-session의 역할: OAuth 리다이렉트 중 state를 잠깐 들고 있는 임시 저장소

OAuth 플로우에서 왜 세션이 필요한가:
  ① GET /auth/google       → 구글 로그인 페이지로 redirect (요청 A)
  ② GET /auth/google/callback → 구글이 code와 함께 돌아옴 (요청 B)

  요청 A와 요청 B는 별개의 HTTP 요청
  Passport가 CSRF 방지를 위해 state 값을 A에서 생성하고 B에서 검증함
  이 state를 A와 B 사이에 어딘가 저장해야 함 → express-session

  즉, 세션을 "로그인 유지"에 쓰는 게 아니라
  "OAuth 콜백이 돌아오는 짧은 순간에만 state를 기억"하는 데 씀
  → maxAge: 10 * 60 * 1000 (10분)으로 짧게 설정하는 이유
```

## main.ts 설정

```typescript
import session  from 'express-session';
import passport from 'passport';

async function bootstrap() {
  const app = await NestFactory.create(AppModule);
  const configService = app.get(ConfigService);

  app.use(
    session({
      secret:            configService.getOrThrow<string>(EnvKeys.SESSION_SECRET),
      resave:            false,
      saveUninitialized: false,
      cookie: {
        httpOnly: true,
        sameSite: 'lax',
        secure:   process.env.NODE_ENV === 'production',
        maxAge:   10 * 60 * 1000,  // 10분 — OAuth 플로우용
      },
    }),
  );

  // passport.authenticate('google', ...) 를 쓰려면 반드시 필요
  app.use(passport.initialize());

  // passport.session() 은 넣지 않음 ← 우리 흐름에서 불필요 (아래 설명)
}
```

## 세 가지 미들웨어 — 무엇이 필요하고 왜 ⭐️⭐️⭐️⭐️

```txt
express-session   → 필요함
  Google OAuth의 state 값을 콜백까지 넘기려면 세션이 필요함
  /auth/google 요청(state 생성)과 /auth/google/callback(state 검증) 사이의 임시 저장소

passport.initialize()  → 필요함
  passport.authenticate('google', ...) 를 쓰려면 반드시 있어야 함

passport.session()  → 인증 유지 방식에 따라 선택
  Passport가 req.user를 세션에 저장·복원할 때 쓰는 미들웨어

  필요한 경우: 세션 기반 로그인 유지 (전통적 웹앱 방식)
    로그인 성공 → serializeUser(user → session에 id 저장)
    이후 요청  → deserializeUser(session id → DB에서 user 복원 → req.user)
    → 로그아웃할 때 session destroy로 즉시 무효화 가능

  불필요한 경우: JWT + 프론트 redirect 방식
    OAuth 완료 후 JWT 발급하고 바로 프론트로 redirect
    이후 요청은 Authorization: Bearer {JWT} 헤더로 인증
    → req.user를 세션에 저장·복원할 이유가 없음
    → callback route에서 session: false 로 명시

  passport.session() 없이 소셜 로그인만 하려면:
    callback route에 { session: false } 반드시 추가
    없으면 Passport가 serializeUser를 찾다가 에러
```

## 콜백 라우트에서 session: false ⭐️⭐️⭐️

```typescript
// callback 라우트 — session: false 로 명시
// passport.session()을 안 썼으니 세션에 user를 저장·복원하려 하면 안 됨
@Get('google/callback')
googleCallback(@Req() req: Request, @Res() res: Response, @Next() next: NextFunction) {
  passport.authenticate('google', { session: false }, async (err, user) => {
    if (err || !user) return res.redirect('/auth/error');
    const tokens = this.authService.issueTokens(user);
    res.cookie('accessToken', tokens.accessToken, { httpOnly: true });
    res.redirect(this.config.get('FRONTEND_URL') + '/auth/callback');
  })(req, res, next);
}
```

```txt
session: false 가 없으면:
  Passport가 serializeUser를 찾으려 함 → 등록 안 됐으면 에러
  → callback route에 session: false 명시로 "세션에 저장 안 해도 됨" 선언

언제 passport.session()이 필요한가:
  세션 기반 인증(쿠키 세션으로 로그인 유지)을 구현할 때
  serializeUser(req.user → 세션 저장) + deserializeUser(세션 → req.user 복원) 세트로 등록
  → 우리는 JWT + redirect 방식이라 해당 없음
```

## 옵션 설명

|옵션|값|설명|
|---|---|---|
|`secret`|랜덤 문자열|세션 쿠키 서명에 쓰는 비밀키 — 반드시 .env에서 관리|
|`resave`|`false`|변경 없으면 세션 재저장 안 함 (성능)|
|`saveUninitialized`|`false`|빈 세션 저장 안 함 (메모리 절약)|
|`cookie.httpOnly`|`true`|JS로 세션 쿠키 접근 불가 (XSS 방어)|
|`cookie.sameSite`|`'lax'`|외부에서 직접 세션 쿠키 전송 차단 (CSRF 기본 방어)|
|`cookie.secure`|운영에서만 `true`|HTTPS에서만 전송 — 로컬 개발 시 `false`|
|`cookie.maxAge`|`10 * 60 * 1000`|10분 — OAuth 플로우가 끝날 때까지만 유효하면 충분|

```txt
SESSION_SECRET .env 예시:
  SESSION_SECRET=your-random-secret-string-here

  강한 랜덤 문자열 생성:
  node -e "console.log(require('crypto').randomBytes(32).toString('hex'))"
```

---

# AuthModule 설정 ⭐️⭐️⭐️

```typescript
@Module({
  imports: [
    // session: false → JWT 방식 (기본값 false로 설정하면 개별 route마다 명시 불필요)
    // session: true  → 세션 기반 로그인 유지 방식 (passport.session() 함께 사용)
    PassportModule.register({ session: false }),

    JwtModule.registerAsync({
      imports:    [ConfigModule],
      inject:     [ConfigService],
      useFactory: (configService: ConfigService) => ({
        secret:      configService.getOrThrow<string>(EnvKeys.JWT_SECRET),
        signOptions: { expiresIn: '7d' },
      }),
    }),

    UserModule,  // UserService 주입을 위해
  ],
  providers: [
    AuthService,

    // Guards — exports에 포함해서 다른 모듈에서도 @UseGuards로 사용 가능하게
    JwtAuthGuard,
    RolesGuard,

    // 전략들 — 사용하는 것만 추가 (안 쓰는 전략은 제거해도 됨)
    JwtStrategy,
    LocalStrategy,   // 로컬 ID/비밀번호를 쓴다면
    GoogleStrategy,  // 소셜 전략은 쓰는 것만
    // KakaoStrategy,
    // NaverStrategy,
    // AppleStrategy,
  ],
  exports: [
    JwtModule,      // 다른 모듈에서 JwtService를 직접 쓰고 싶을 때
    JwtAuthGuard,   // 다른 모듈 컨트롤러에서 @UseGuards(JwtAuthGuard) 사용 가능하게
    RolesGuard,     // @UseGuards(RolesGuard) 를 다른 모듈에서도
    AuthService,    // 다른 모듈에서 토큰 발급·검증 등 AuthService 메서드를 쓸 때
  ],
  controllers: [AuthController],
})
export class AuthModule {}
```

```txt
PassportModule.register({ session: false/true }):
  false  JWT 방식 — callback route마다 { session: false }를 개별 명시 안 해도 됨
  true   세션 방식 — serializeUser/deserializeUser + passport.session() 함께 사용

providers:
  전략(Strategy)은 사용하는 것만 등록 — 미사용 전략은 지워도 됨
  Guard는 providers에 등록해야 DI가 가능하고 exports에도 올려야 다른 모듈에서 씀

exports:
  JwtModule    다른 모듈에서 JwtService.sign() 등을 직접 쓸 때 필요
  JwtAuthGuard 다른 모듈 컨트롤러에 @UseGuards(JwtAuthGuard) 붙이려면 필요
  AuthService  토큰 발급·검증 로직을 다른 모듈(예: UserModule)에서도 쓸 때

AppModule에 AuthModule 한 번만 등록하면
exports에 있는 것들은 imports한 모든 모듈에서 자동으로 주입 가능
```

---

# Express 타입 확장 — express.d.ts ⭐️⭐️⭐️⭐️

```txt
Express 기본 Request 타입에는 session, user가 없음
express-session을 설치해도 타입이 자동으로 붙지 않음
→ 직접 declare module / declare global로 타입을 확장해야 함
```

## 설치

```bash
pnpm add -D @types/express-session
```

## apps/api/src/types/express.d.ts

```typescript
import type { JwtPayload } from '../auth/jwt-payload';
import type { Session }    from 'express-session';

// express-session의 SessionData에 커스텀 필드 추가
declare module 'express-session' {
  interface SessionData {
    oauthNext?: string;  // OAuth 완료 후 리다이렉트할 URL
  }
}

// Express의 Request에 user, session 추가
declare global {
  namespace Express {
    interface Request {
      user?:   JwtPayload;  // JwtStrategy.validate()가 담아주는 값
      session: Session;     // express-session이 담아주는 값
    }
  }
}

export {}; // 이 파일을 모듈로 만들어야 declare global이 동작함
```

```txt
왜 export {}가 필요한가:
  import/export 문이 없으면 TypeScript가 이 파일을 "스크립트"로 처리
  declare global은 "모듈" 안에서만 전역 확장이 가능
  → export {} 한 줄로 "나는 모듈이다"를 선언 → declare global 동작

declare module 'express-session':
  express-session 라이브러리의 SessionData 인터페이스를 확장
  oauthNext를 추가하면 req.session.oauthNext = '...' 처럼 타입 안전하게 사용 가능

declare global { namespace Express { interface Request } }:
  Express.Request 인터페이스를 전역으로 확장
  @Req() req: Request 에서 req.user, req.session에 타입 자동완성이 됨

이 두 방식(declare module, declare global)의 차이:
  declare module '라이브러리명'  외부 라이브러리의 타입을 확장
  declare global { ... }        전역 타입(window, Express 등)을 확장
```

## tsconfig에서 포함 확인

```json
// apps/api/tsconfig.json
{
  "include": ["src/**/*"]  // src/types/express.d.ts 포함되는지 확인
}
```

```txt
include 범위에 .d.ts 파일이 들어가지 않으면 확장이 적용 안 됨
```

## JwtPayload 타입

```typescript
// apps/api/src/auth/jwt-payload.ts
export interface JwtPayload {
  sub:   number;  // user.id
  email: string;
  // 필요한 필드 추가
}
```

```txt
req.user의 타입을 JwtPayload로 잡아두면:
  controller에서 req.user.sub, req.user.email 에 자동완성 동작
  JwtStrategy.validate()의 반환 타입과 맞춰야 함
```

---

# 로컬 전략 — 이메일 + 비밀번호 ⭐️⭐️⭐️⭐️

## Strategy

```typescript
// strategies/local.strategy.ts
@Injectable()
export class LocalStrategy extends PassportStrategy(Strategy, 'local') {
  constructor(private authService: AuthService) {
    super({ usernameField: 'email' }); // 기본값은 'username' — 'email'로 변경
  }

  async validate(email: string, password: string): Promise<User> {
    const user = await this.authService.validateUser(email, password);
    if (!user) throw new UnauthorizedException('이메일 또는 비밀번호가 올바르지 않습니다.');
    return user; // req.user에 저장됨
  }
}
```

## AuthService

```typescript
// auth.service.ts
@Injectable()
export class AuthService {
  constructor(
    private userService: UserService,
    private jwtService: JwtService,
  ) {}

  // 비밀번호 검증
  async validateUser(email: string, password: string): Promise<User | null> {
    const user = await this.userService.findByEmail(email);
    if (!user) return null;

    const isMatch = await bcrypt.compare(password, user.passwordHash);
    if (!isMatch) return null;

    return user;
  }

  // JWT 발급
  issueTokens(user: User) {
    const payload = { sub: user.id, email: user.email };
    return {
      accessToken:  this.jwtService.sign(payload, { expiresIn: '1h' }),
      refreshToken: this.jwtService.sign(payload, { expiresIn: '30d' }),
    };
  }

  // 회원가입
  async register(email: string, password: string) {
    const existing = await this.userService.findByEmail(email);
    if (existing) throw new ConflictException('이미 사용 중인 이메일입니다.');

    const hash = await bcrypt.hash(password, 10);
    const user = await this.userService.create({ email, passwordHash: hash });
    return this.issueTokens(user);
  }
}
```

## Guard + Controller

```typescript
// guards/local-auth.guard.ts
@Injectable()
export class LocalAuthGuard extends AuthGuard('local') {}
```

```typescript
// auth.controller.ts
@Post('login')
@UseGuards(LocalAuthGuard) // validate()가 실행되고 req.user에 결과가 담김
login(@Req() req: Request) {
  return this.authService.issueTokens(req.user as User);
}

@Post('register')
register(@Body() dto: RegisterDto) {
  return this.authService.register(dto.email, dto.password);
}
```

---

# JWT 전략 — 모든 인증 요청 검증 ⭐️⭐️⭐️⭐️

```typescript
// strategies/jwt.strategy.ts
@Injectable()
export class JwtStrategy extends PassportStrategy(Strategy, 'jwt') {
  constructor(config: ConfigService) {
    super({
      jwtFromRequest: ExtractJwt.fromAuthHeaderAsBearerToken(),
      secretOrKey:    config.get('JWT_SECRET'),
    });
  }

  async validate(payload: { sub: number; email: string }) {
    return { id: payload.sub, email: payload.email }; // req.user에 저장됨
  }
}
```

```typescript
// guards/jwt-auth.guard.ts
@Injectable()
export class JwtAuthGuard extends AuthGuard('jwt') {}

// 사용 — 인증이 필요한 모든 엔드포인트에
@UseGuards(JwtAuthGuard)
@Get('me')
getMe(@Req() req: Request) {
  return req.user; // JwtStrategy.validate()가 반환한 값
}
```

---

# 소셜 로그인 레이어 구조 ⭐️⭐️⭐️⭐️

```txt
Strategy / Controller / Service 각자 하는 일이 다름
섞으면 테스트도 어렵고 다른 제공자 추가할 때 중복이 생김
```

|레이어|책임|하지 않는 것|
|---|---|---|
|**Strategy**|Provider 프로필 → 앱 형태로 정규화 (`validate()`)|DB 접근 · JWT · redirect|
|**Controller**|HTTP (`req`/`res`) · session next 저장 · `res.redirect()`|try/catch 병합 로직 · URL 규칙|
|**Service**|계정 연결 · User 생성 · JWT 발급 · redirect URL 조립|`passport.authenticate()` 직접 호출|

---

# 소셜 전략 — 프로필 정규화만 ⭐️⭐️⭐️⭐️

```txt
validate()의 역할: Provider가 준 프로필을 앱이 쓰는 형태로 변환만 함
DB 조회 · 계정 연결 · JWT 발급은 하지 않음 → Service 책임
```

```typescript
// types/oauth-profile.ts — 제공자 무관 공통 타입
export type OAuthProfile = {
  providerAccountId: string;
  email:             string | null;
  emailVerified:     boolean;
  displayName:       string | null;
};
```

```typescript
// strategies/google.strategy.ts
@Injectable()
export class GoogleStrategy extends PassportStrategy(Strategy, 'google') {
  constructor(configService: ConfigService) {
    const port = configService.getOrThrow<string>(EnvKeys.PORT);
    super({
      clientID:     configService.getOrThrow<string>(EnvKeys.GOOGLE_CLIENT_ID),
      clientSecret: configService.getOrThrow<string>(EnvKeys.GOOGLE_CLIENT_SECRET),
      callbackURL:  `http://localhost:${port}/auth/oauth/google/callback`,
      scope:        ['email', 'profile'],
    });
  }

  // 반환값이 req.user에 담김 — DB 접근 없이 프로필 정규화만
  validate(
    _accessToken:  string,
    _refreshToken: string,
    profile: Profile,
  ): OAuthProfile {
    return {
      providerAccountId: profile.id,
      email:             profile.emails?.[0]?.value ?? null,
      emailVerified:     profile.emails?.[0]?.verified ?? false,
      displayName:       profile.displayName ?? null,
    };
  }
}
```

```txt
제공자별 validate() 차이점:

Kakao — 이메일 경로가 다름:
  email:         profile._json?.kakao_account?.email ?? null
  emailVerified: profile._json?.kakao_account?.is_email_verified ?? false

Naver — Google과 동일한 경로:
  email:         profile.emails?.[0]?.value ?? null
  emailVerified: 항상 true (Naver는 인증된 이메일만 제공)

Apple — Strategy 없음, Controller에서 id_token JWT 직접 검증 (아래 참고)
```

---

# 소셜 OAuth 컨트롤러 — 두 단계 패턴 ⭐️⭐️⭐️⭐️

```txt
소셜 로그인은 두 개의 라우트로 구성됨:
  ① 시작 라우트: session에 next 저장 → passport.authenticate()로 제공자 페이지 이동
  ② 콜백 라우트: @UseGuards(AuthGuard) → validate() 실행 → req.user → Service

시작 라우트에 @UseGuards(AuthGuard) 쓰면 안 되는 이유:
  시작 라우트는 제공자 로그인 페이지로 "보내기만" 하는 역할
  session.oauthNext 저장 같은 커스텀 로직이 필요
  → passport.authenticate()(req, res) 직접 호출로 제어권을 가져옴
```

```typescript
// oauth.controller.ts
const DEFAULT_NEXT = '/dashboard';

// next 파라미터 보안 검증 — open redirect 방지
function sanitizeRedirectPath(path: string | undefined): string {
  // 절대 경로가 아니거나 //로 시작하면(프로토콜 상대 URL) 기본값으로
  if (!path || !path.startsWith('/') || path.startsWith('//')) {
    return DEFAULT_NEXT;
  }
  return path;
}

@Controller('auth/oauth')
export class OAuthController {
  constructor(
    private readonly authService:   AuthService,
    private readonly configService: ConfigService,
  ) {}

  // ① 시작 — next를 session에 저장하고 제공자 로그인 페이지로 이동
  @Get('google')
  googleStart(
    @Query('next') next: string | undefined,
    @Req() req: Request,
    @Res() res: Response,
  ) {
    req.session.oauthNext = sanitizeRedirectPath(next);
    return passport.authenticate('google')(req, res);
    //     ↑ @UseGuards 아니라 직접 호출 — 커스텀 처리 후 인증 시작
  }

  // ② 콜백 — AuthGuard가 Strategy.validate() 실행 → req.user에 OAuthProfile 담김
  @Get('google/callback')
  @UseGuards(AuthGuard('google'))
  async googleCallback(@Req() req: Request, @Res() res: Response) {
    const next = sanitizeRedirectPath(req.session.oauthNext);
    delete req.session.oauthNext;  // 사용 후 즉시 삭제

    const redirectUrl = await this.authService.handleOAuthCallback(
      req.user as OAuthProfile,
      next,
    );
    return res.redirect(redirectUrl);
  }
}
```

```txt
sanitizeRedirectPath — open redirect 방지:
  next 파라미터에 'https://evil.com'이 들어오면
  우리 서버가 그쪽으로 리다이렉트하는 취약점 발생
  → '/'로 시작하는 상대 경로만 허용, 나머지는 기본값

session.oauthNext:
  ① 시작 라우트와 ② 콜백 라우트는 별개의 HTTP 요청
  next 값을 둘 사이에 전달하려면 express-session이 필요
  → express.d.ts에 oauthNext?: string 선언 필요
```

---

# 소셜 OAuth 서비스 — 계정 연결 · JWT 발급 ⭐️⭐️⭐️⭐️

```typescript
@Injectable()
export class AuthService {

  async handleOAuthCallback(profile: OAuthProfile, next: string): Promise<string> {
    try {
      // 1. email_verified 게이트
      if (!profile.emailVerified) {
        return this.buildErrorRedirect('email_not_verified', next);
      }
      if (!profile.email) {
        return this.buildErrorRedirect('email_missing', next);
      }

      // 2. 계정 연결 · 생성
      const user = await this.findOrCreateOAuthUser(profile);

      // 3. JWT 발급 + 4. 성공 redirect URL 조립
      const { accessToken } = this.issueTokens(user);
      return this.buildSuccessRedirect(accessToken, next);

    } catch {
      return this.buildErrorRedirect('account_link_failed', next);
    }
  }

  private async findOrCreateOAuthUser(profile: OAuthProfile): Promise<User> {
    // OAuthAccount로 기존 연결 찾기
    const existing = await this.prisma.oAuthAccount.findUnique({
      where: {
        provider_providerAccountId: {
          provider:          'google',
          providerAccountId: profile.providerAccountId,
        },
      },
      include: { user: true },
    });
    if (existing) return existing.user;

    // email로 기존 User 찾기 → 계정 병합
    const existingUser = await this.prisma.user.findUnique({
      where: { email: profile.email! },
    });
    if (existingUser) {
      await this.prisma.oAuthAccount.create({
        data: {
          provider:          'google',
          providerAccountId: profile.providerAccountId,
          userId:            existingUser.id,
        },
      });
      return existingUser;
    }

    // 새 User + OAuthAccount 생성
    return this.prisma.user.create({
      data: {
        email:       profile.email!,
        displayName: profile.displayName,
        oAuthAccounts: {
          create: {
            provider:          'google',
            providerAccountId: profile.providerAccountId,
          },
        },
      },
    });
  }

  private buildSuccessRedirect(accessToken: string, next: string): string {
    const url = new URL('/auth/oauth/callback', this.configService.getOrThrow(EnvKeys.FRONTEND_URL));
    url.searchParams.set('accessToken', accessToken);
    url.searchParams.set('next', next);
    return url.toString();
  }

  private buildErrorRedirect(oauthError: string, next: string): string {
    const url = new URL('/login', this.configService.getOrThrow(EnvKeys.FRONTEND_URL));
    url.searchParams.set('next', next);
    url.searchParams.set('oauthError', oauthError);
    return url.toString();
  }
}
```

## oauthError 코드

|코드|의미|
|---|---|
|`email_not_verified`|Provider가 인증된 이메일 미제공|
|`email_missing`|OAuth 응답에 이메일 없음|
|`provider_error`|Provider / 콜백 일반 실패|
|`account_link_failed`|DB 연결·병합 실패|

## Apple — id_token 직접 검증 ⭐️⭐️⭐️

```txt
Apple은 Strategy 없음 — 다른 제공자와 동작 방식이 다름:
  ① callback이 GET이 아니라 POST로 옴
  ② access_token으로 사용자 API를 호출하지 않음
  ③ id_token(JWT)을 Apple 공개키로 직접 검증해서 사용자 정보 추출
  ④ 첫 로그인 시에만 name 정보가 옴 → DB 저장 필수

  특이점: 이메일 숨기기 선택 시 릴레이 이메일(@privaterelay.appleid.com) 제공
```

```typescript
@Post('apple/callback')
async appleCallback(@Body() body: { id_token: string; user?: string }) {
  const payload = await appleSignin.verifyIdToken(body.id_token, {
    audience: this.configService.getOrThrow(EnvKeys.APPLE_CLIENT_ID),
  });

  // 첫 로그인 시에만 user 필드(이름)가 옴
  const userData = body.user ? JSON.parse(body.user) : null;
  const displayName = userData?.name
    ? `${userData.name.firstName} ${userData.name.lastName}`
    : null;

  const profile: OAuthProfile = {
    providerAccountId: payload.sub,
    email:             payload.email ?? null,
    emailVerified:     payload.email_verified === 'true',
    displayName,
  };

  const next = sanitizeRedirectPath(undefined);
  const redirectUrl = await this.authService.handleOAuthCallback(profile, next);
  return res.redirect(redirectUrl);
}
```

## Prisma 스키마

```prisma
enum OAuthProvider {
  google
  naver
  kakao
  apple
}

model User {
  id           String    @id @default(uuid(7)) @db.Uuid
  email        String    @unique
  nickname     String    @unique
  passwordHash String?                          // 로컬 로그인 시에만 있음
  role         UserRole  @default(user)
  bio          String?
  lastActiveAt DateTime? @db.Timestamptz(3)
  createdAt    DateTime  @default(now()) @db.Timestamptz(3)
  updatedAt    DateTime  @updatedAt @db.Timestamptz(3)

  oAuthAccounts OAuthAccount[]
}

model OAuthAccount {
  id                String      @id @default(uuid(7)) @db.Uuid
  provider          OAuthProvider
  providerAccountId String
  userId            String      @db.Uuid
  user              User        @relation(fields: [userId], references: [id], onDelete: Cascade)
  createdAt         DateTime    @default(now()) @db.Timestamptz(3)

  @@unique([provider, providerAccountId])
  @@index([provider])
}
```

```txt
uuid(7) — UUID v7 (시간 기반 정렬 가능)
  uuid(4)는 완전 랜덤 → DB 인덱스 단편화 심함
  uuid(7)은 시간 순서 포함 → 삽입 순서로 자연 정렬 가능 → 인덱스 효율적

OAuthProvider enum:
  문자열 대신 enum으로 → 잘못된 provider 값이 DB에 들어가는 것을 DB 레벨에서 차단
  Prisma에서 OAuthProvider.google 처럼 타입 안전하게 사용 가능
  새 제공자 추가 시 enum에 추가 + migrate 필요

onDelete: Cascade:
  User 삭제 시 연결된 OAuthAccount도 자동 삭제
  고아 레코드(user 없는 oauthAccount) 방지

@@index([provider]):
  "이 제공자의 계정들" 조회가 빠르게
  @@unique([provider, providerAccountId])가 있으면 사실 provider 단독 인덱스는 없어도 되지만
  제공자별 통계, 관리 쿼리에서 유용

Service 코드에서 provider 타입:
  provider: OAuthProvider  (← string 아님)
  import { OAuthProvider } from '../generated/prisma';
```

---

# .env 정리 ⭐️⭐️⭐️

```properties
# JWT
JWT_SECRET=your-secret-key-here
JWT_EXPIRES_IN=1h

# Google
GOOGLE_CLIENT_ID=xxx.apps.googleusercontent.com
GOOGLE_CLIENT_SECRET=GOCSPX-xxx
GOOGLE_CALLBACK_URL=http://localhost:3000/auth/google/callback

# Kakao
KAKAO_CLIENT_ID=xxx
KAKAO_CALLBACK_URL=http://localhost:3000/auth/kakao/callback

# Naver
NAVER_CLIENT_ID=xxx
NAVER_CLIENT_SECRET=xxx
NAVER_CALLBACK_URL=http://localhost:3000/auth/naver/callback

# Apple
APPLE_CLIENT_ID=com.yourapp.bundle
APPLE_TEAM_ID=XXXXXXXXXX
APPLE_KEY_ID=XXXXXXXXXX
APPLE_PRIVATE_KEY="-----BEGIN PRIVATE KEY-----\n..."
```

---

# 한눈에

```txt
패키지 구조:
  @nestjs/passport + passport            NestJS Passport 통합
  @nestjs/jwt + passport-jwt             JWT 발급/검증
  passport-local + bcrypt               ID/비밀번호
  passport-google-oauth20               Google OAuth
  passport-kakao                        Kakao OAuth
  passport-naver-v2                     Naver OAuth
  apple-signin-auth                     Apple (JWT 직접 검증)

공통 흐름:
  전략(Strategy) 등록 → Guard로 요청 막기 → validate() 실행 → req.user 세팅 → JWT 발급

소셜 전략 공통:
  validate()에서 findOrCreateSocialUser() → DB upsert → User 반환
  issueTokens()로 우리 JWT 발급 → 프론트로 전달 (쿠키 or 쿼리스트링)

Apple 특이점:
  POST callback (다른 건 GET)
  id_token JWT 직접 검증
  첫 로그인에만 이름 옴 → DB 저장 필수

DB 스키마:
  User + SocialAccount 분리
  @@unique([provider, providerId])로 중복 방지

개념(OAuth 흐름, JWT vs 세션) → [[Auth_Concept]]
Guard 구현 상세 → [[NestJS_Guard]]
토큰 저장 위치 선택 → [[NextJS_TokenStorage]]
```