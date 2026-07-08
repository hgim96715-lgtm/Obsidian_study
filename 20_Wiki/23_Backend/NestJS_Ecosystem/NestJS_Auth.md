---
aliases:
  - 인증
  - Passport
  - OAuth
  - Social
tags:
  - NestJS
related:
  - "[[00_NestJS_Ecosystem_HomePage]]"
  - "[[Auth_Concept]]"
  - "[[NestJS_JwtGuard]]"
  - "[[NestJS_Module]]"
  - "[[Web_Cookie]]"
  - "[[NextJS_TokenStorage]]"
---

# NestJS_Auth — NestJS 인증 구현

> [!info] 
> NestJS에서 인증은 Passport 미들웨어로 처리한다. 
> 전략(Strategy)을 등록하고, Guard로 요청을 막고, 성공 시 JWT를 발급하는 구조다.

---

# 패키지 구조 ⭐️⭐️⭐️⭐️

```bash
# 공통 필수
pnpm --filter api add @nestjs/passport @nestjs/jwt passport passport-jwt
pnpm --filter api add -D @types/passport @types/passport-jwt

# 로컬 (ID/비밀번호)
pnpm --filter api add passport-local bcrypt
pnpm --filter api add -D @types/passport-local @types/bcrypt

# 소셜 로그인
pnpm --filter api add passport-google-oauth20
pnpm --filter api add -D @types/passport-google-oauth20

pnpm --filter api add passport-kakao
pnpm --filter api add passport-naver-v2

# Apple (전용 passport 전략 없음 — JWT 직접 검증)
pnpm --filter api add apple-signin-auth

# OAuth callback 중 세션 임시 저장 (선택)
pnpm --filter api add express-session
pnpm --filter api add -D @types/express-session
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

# AuthModule 설정 ⭐️⭐️⭐️

```typescript
// auth.module.ts
@Module({
  imports: [
    PassportModule,
    JwtModule.registerAsync({
      imports:    [ConfigModule],
      useFactory: (config: ConfigService) => ({
        secret:      config.get('JWT_SECRET'),
        signOptions: { expiresIn: '1h' },
      }),
      inject: [ConfigService],
    }),
    UserModule,  // UserService 주입을 위해
  ],
  providers: [
    AuthService,
    LocalStrategy,
    JwtStrategy,
    GoogleStrategy,
    KakaoStrategy,
    NaverStrategy,
    AppleStrategy,
  ],
  controllers: [AuthController],
})
export class AuthModule {}
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

# 소셜 로그인 공통 구조 ⭐️⭐️⭐️⭐️

```txt
모든 소셜 전략은 구조가 동일:
  ① callbackURL로 code가 들어옴
  ② Passport가 code → 제공자 API → 사용자 프로필 교환 (자동)
  ③ validate()에서 DB upsert → User 반환
  ④ authService.issueTokens(user)로 JWT 발급
  ⑤ 프론트로 토큰을 쿠키나 쿼리스트링으로 전달

다른 점은 callbackURL, clientID/Secret, scope뿐
```

## Google ⭐️⭐️⭐️

```typescript
// strategies/google.strategy.ts
@Injectable()
export class GoogleStrategy extends PassportStrategy(GooglePassportStrategy, 'google') {
  constructor(
    config:      ConfigService,
    private auth: AuthService,
  ) {
    super({
      clientID:     config.get('GOOGLE_CLIENT_ID'),
      clientSecret: config.get('GOOGLE_CLIENT_SECRET'),
      callbackURL:  config.get('GOOGLE_CALLBACK_URL'),
      scope:        ['email', 'profile'],
    });
  }

  async validate(
    _accessToken:  string,
    _refreshToken: string,
    profile:       Profile,
  ) {
    const email = profile.emails?.[0]?.value;
    const name  = profile.displayName;
    const image = profile.photos?.[0]?.value;

    return this.auth.findOrCreateSocialUser({
      provider:   'google',
      providerId: profile.id,
      email,
      name,
      image,
    });
  }
}
```

## Kakao ⭐️⭐️⭐️

```typescript
// strategies/kakao.strategy.ts
@Injectable()
export class KakaoStrategy extends PassportStrategy(KakaoPassportStrategy, 'kakao') {
  constructor(config: ConfigService, private auth: AuthService) {
    super({
      clientID:    config.get('KAKAO_CLIENT_ID'),
      callbackURL: config.get('KAKAO_CALLBACK_URL'),
      // Kakao는 clientSecret 선택사항
    });
  }

  async validate(_accessToken: string, _refreshToken: string, profile: any) {
    // ⚠️ Kakao는 이메일 경로가 다름
    const email = profile._json?.kakao_account?.email;
    const name  = profile.displayName;
    const image = profile._json?.properties?.profile_image;

    return this.auth.findOrCreateSocialUser({
      provider:   'kakao',
      providerId: String(profile.id),
      email,
      name,
      image,
    });
  }
}
```

## Naver ⭐️⭐️⭐️

```typescript
// strategies/naver.strategy.ts
@Injectable()
export class NaverStrategy extends PassportStrategy(NaverPassportStrategy, 'naver') {
  constructor(config: ConfigService, private auth: AuthService) {
    super({
      clientID:     config.get('NAVER_CLIENT_ID'),
      clientSecret: config.get('NAVER_CLIENT_SECRET'),
      callbackURL:  config.get('NAVER_CALLBACK_URL'),
    });
  }

  async validate(_accessToken: string, _refreshToken: string, profile: any) {
    const email = profile.emails?.[0]?.value;
    const name  = profile.displayName;
    const image = profile.photos?.[0]?.value;

    return this.auth.findOrCreateSocialUser({
      provider:   'naver',
      providerId: profile.id,
      email,
      name,
      image,
    });
  }
}
```

## Apple — JWT 직접 검증 방식 ⭐️⭐️⭐️

```txt
Apple Sign In은 다른 제공자와 다름:
  passport-apple 같은 표준 전략이 없거나 불안정
  대신 id_token(JWT)을 Apple의 공개키로 직접 검증

  Apple 특이점:
  ① 이메일 숨기기 선택 시 릴레이 이메일 제공 (xxx@privaterelay.appleid.com)
  ② 첫 로그인 시에만 name 정보가 옴 → DB에 저장해야 함, 이후 요청엔 없음
  ③ client_secret이 고정값이 아니라 ES256 JWT로 직접 생성해야 함
```

```typescript
// auth.controller.ts — Apple은 POST로 callback이 옴 (다른 제공자는 GET)
@Post('apple/callback')
async appleCallback(@Body() body: { id_token: string; user?: string }) {
  const appleUser = await appleSignin.verifyIdToken(body.id_token, {
    audience: this.config.get('APPLE_CLIENT_ID'),
    nonce:    'nonce',  // 선택적
  });

  // 첫 로그인 시에만 user 필드(이름)가 옴
  const userData = body.user ? JSON.parse(body.user) : null;
  const name = userData?.name
    ? `${userData.name.firstName} ${userData.name.lastName}`
    : undefined;

  const user = await this.authService.findOrCreateSocialUser({
    provider:   'apple',
    providerId: appleUser.sub,
    email:      appleUser.email,
    name,
  });

  return this.authService.issueTokens(user);
}
```

---

# findOrCreateSocialUser — 공통 upsert ⭐️⭐️⭐️⭐️

```typescript
// auth.service.ts
async findOrCreateSocialUser(dto: {
  provider:   string;
  providerId: string;
  email?:     string;
  name?:      string;
  image?:     string;
}): Promise<User> {
  // 소셜 계정으로 기존 사용자 찾기
  const existing = await this.prisma.socialAccount.findUnique({
    where: {
      provider_providerId: {
        provider:   dto.provider,
        providerId: dto.providerId,
      },
    },
    include: { user: true },
  });

  if (existing) return existing.user;

  // 없으면 사용자 + 소셜 계정 생성
  const user = await this.prisma.user.create({
    data: {
      email:   dto.email,
      name:    dto.name,
      image:   dto.image,
      socialAccounts: {
        create: {
          provider:   dto.provider,
          providerId: dto.providerId,
        },
      },
    },
  });

  return user;
}
```

## Prisma 스키마

```prisma
model User {
  id             Int             @id @default(autoincrement())
  email          String?         @unique
  name           String?
  image          String?
  passwordHash   String?         // 로컬 로그인 시에만 있음
  socialAccounts SocialAccount[]
  createdAt      DateTime        @default(now())
}

model SocialAccount {
  id         Int    @id @default(autoincrement())
  provider   String              // 'google' | 'kakao' | 'naver' | 'apple'
  providerId String              // 제공자가 준 고유 ID
  user       User   @relation(fields: [userId], references: [id])
  userId     Int

  @@unique([provider, providerId])  // 같은 제공자의 같은 ID는 하나만
}
```

---

# 컨트롤러 — 소셜 로그인 라우트 ⭐️⭐️⭐️

```typescript
// auth.controller.ts
@Controller('auth')
export class AuthController {
  // --- 로컬 ---
  @Post('register')
  register(@Body() dto: RegisterDto) {
    return this.authService.register(dto.email, dto.password);
  }

  @Post('login')
  @UseGuards(LocalAuthGuard)
  login(@Req() req: Request) {
    return this.authService.issueTokens(req.user as User);
  }

  // --- Google ---
  @Get('google')
  @UseGuards(GoogleAuthGuard)
  googleLogin() {}  // Guard가 리다이렉트 처리

  @Get('google/callback')
  @UseGuards(GoogleAuthGuard)
  googleCallback(@Req() req: Request, @Res() res: Response) {
    const tokens = this.authService.issueTokens(req.user as User);
    // 프론트로 토큰 전달 (쿠키 또는 쿼리스트링)
    res.cookie('accessToken', tokens.accessToken, { httpOnly: true });
    res.redirect(this.config.get('FRONTEND_URL') + '/auth/callback');
  }

  // Kakao, Naver도 동일한 패턴 — 경로와 Guard만 다름
  @Get('kakao')
  @UseGuards(KakaoAuthGuard)
  kakaoLogin() {}

  @Get('kakao/callback')
  @UseGuards(KakaoAuthGuard)
  kakaoCallback(@Req() req: Request, @Res() res: Response) {
    const tokens = this.authService.issueTokens(req.user as User);
    res.cookie('accessToken', tokens.accessToken, { httpOnly: true });
    res.redirect(this.config.get('FRONTEND_URL') + '/auth/callback');
  }
}
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