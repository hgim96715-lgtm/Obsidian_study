---
aliases:
  - 인증
  - OAuth
  - Passport
  - Social
  - bcrypt
tags:
  - NestJS
related:
  - "[[00_NestJS_Ecosystem_HomePage]]"
  - "[[Auth_Concept]]"
  - "[[NestJS_JwtGuard]]"
  - "[[NestJS_Controller]]"
  - "[[NestJS_Env_Config]]"
  - "[[TS_ImportType]]"
---
# NestJS_Auth — JWT 인증 구현

>[!info]
>NestJS에서 JWT 인증 구현. `JwtModule`로 토큰 설정, `AuthService`에서 로그인·토큰 발급, `AuthController`에서 엔드포인트 제공.
> Passport 없이 `@nestjs/jwt` 직접 사용 — Passport는 OAuth 등 다양한 전략이 필요할 때 쓰며 이 패턴은 JWT만 있는 경우 더 간결하다. 
> 토큰 검증·Guard → [[NestJS_JwtGuard]], 개념 → [[Auth_Concept]]

---

# 설치 ⭐️⭐️

```bash
pnpm add @nestjs/jwt @nestjs/passport passport passport-jwt
pnpm add -D @types/passport-jwt
```

---

# AuthModule 설정 ⭐️⭐️⭐️⭐️

```typescript
// auth.module.ts
import { Module }    from '@nestjs/common';
import { JwtModule } from '@nestjs/jwt';
import { ConfigModule, ConfigService } from '@nestjs/config';

@Module({
  imports: [
    JwtModule.registerAsync({
      imports:    [ConfigModule],
      inject:     [ConfigService],
      useFactory: (config: ConfigService) => ({
        secret:      config.getOrThrow('JWT_SECRET'),
        signOptions: { expiresIn: '15m' },  // Access Token 만료 15분
      }),
    }),
  ],
  providers:   [AuthService],
  controllers: [AuthController],
  exports:     [JwtModule, AuthService],  // Guard에서 사용하기 위해 export
})
export class AuthModule {}
```

```txt
JwtModule.registerAsync:
  환경변수(ConfigService)를 주입받아서 동적으로 설정
  JWT_SECRET — 서명 비밀키 (절대 코드에 직접 쓰지 않음)
  expiresIn  — Access Token 만료시간 ('15m', '1h', '7d' 형태)

exports: [JwtModule]:
  다른 모듈(JwtAuthGuard)에서 JwtService를 쓸 수 있게 공개
```

---

# AuthModule 설정 ⭐️⭐️⭐️⭐️

```typescript
// auth.module.ts
import { Module }    from '@nestjs/common';
import { JwtModule } from '@nestjs/jwt';
import { ConfigModule, ConfigService } from '@nestjs/config';

@Module({
  imports: [
    JwtModule.registerAsync({
      imports:    [ConfigModule],
      inject:     [ConfigService],
      useFactory: (config: ConfigService) => ({
        secret:      config.getOrThrow('JWT_SECRET'),
        signOptions: { expiresIn: '15m' },  // Access Token 만료 15분
      }),
    }),
  ],
  providers:   [AuthService],
  controllers: [AuthController],
  exports:     [JwtModule, AuthService],  // Guard에서 사용하기 위해 export
})
export class AuthModule {}
```

```txt
JwtModule.registerAsync:
  환경변수(ConfigService)를 주입받아서 동적으로 설정
  JWT_SECRET — 서명 비밀키 (절대 코드에 직접 쓰지 않음)
  expiresIn  — Access Token 만료시간 ('15m', '1h', '7d' 형태)

exports: [JwtModule]:
  다른 모듈(JwtAuthGuard)에서 JwtService를 쓸 수 있게 공개
```

---

# AuthService — 토큰 발급 ⭐️⭐️⭐️⭐️

```typescript
// auth.service.ts
import { Injectable, UnauthorizedException } from '@nestjs/common';
import { JwtService } from '@nestjs/jwt';
import * as bcrypt   from 'bcrypt';

@Injectable()
export class AuthService {
  constructor(
    private readonly jwtService:  JwtService,
    private readonly usersService: UsersService,
  ) {}

  // 로그인 — 이메일+비밀번호 검증 후 토큰 발급
  async login(email: string, password: string) {
    // ① 사용자 조회
    const user = await this.usersService.findByEmail(email);
    if (!user) throw new UnauthorizedException('이메일 또는 비밀번호가 틀렸습니다.');

    // ② 비밀번호 검증 (bcrypt — 해시 비교)
    const isMatch = await bcrypt.compare(password, user.hashedPassword);
    if (!isMatch) throw new UnauthorizedException('이메일 또는 비밀번호가 틀렸습니다.');

    // ③ 토큰 발급
    return this.issueTokens(user.id);
  }

  // 회원가입
  async register(email: string, password: string, nickname: string) {
    // 이메일 중복 확인
    const exists = await this.usersService.findByEmail(email);
    if (exists) throw new ConflictException('이미 사용 중인 이메일입니다.');

    // 비밀번호 해시
    const hashedPassword = await bcrypt.hash(password, 10);

    const user = await this.usersService.create({ email, hashedPassword, nickname });
    return this.issueTokens(user.id);
  }

  // Access Token + Refresh Token 발급
  issueTokens(userId: string) {
    const payload = { sub: userId };  // sub = JWT 표준 Subject

    const accessToken  = this.jwtService.sign(payload, { expiresIn: '15m' });
    const refreshToken = this.jwtService.sign(payload, { expiresIn: '7d' });

    return { accessToken, refreshToken };
  }

  // register / login 성공 후 공통 응답 헬퍼
  private async buildAuthResponse(user: { id: string; email: string }) {
    const payload: JwtPayload = { sub: user.id };
    const accessToken = await this.jwtService.signAsync(payload);
    return {
      accessToken,
      user: { id: user.id, email: user.email },
    };
  }

  // Refresh Token으로 새 Access Token 발급
  async refresh(refreshToken: string) {
    try {
      const payload = this.jwtService.verify(refreshToken);
      return { accessToken: this.jwtService.sign({ sub: payload.sub }, { expiresIn: '15m' }) };
    } catch {
      throw new UnauthorizedException('유효하지 않은 토큰입니다.');
    }
  }
}
```

```typescript
// JwtPayload 타입 (별도 파일 또는 auth.service.ts 상단) 보통 auth/jwt-payload.ts
export type JwtPayload {
  sub: string;   // userId
}
```

```txt
buildAuthResponse를 헬퍼로 분리하는 이유:
  register / login 둘 다 성공하면 동일한 응답을 돌려줌
  → 공통 로직을 private 메서드로 추출해서 중복 제거

sign vs signAsync:
  sign()      → 동기 (즉시 반환)
  signAsync() → 비동기 (Promise 반환)
  → async 메서드 안에서 await와 함께 쓸 때는 signAsync

private:
  AuthController에서 직접 호출하지 않고 서비스 내부에서만 사용
  → private으로 캡슐화
```

---

# AuthController — 엔드포인트 ⭐️⭐️⭐️⭐️

```typescript
// auth.controller.ts
import { Controller, Post, Body, HttpCode, HttpStatus } from '@nestjs/common';
import { ApiTags, ApiOperation } from '@nestjs/swagger';

@ApiTags('auth')
@Controller('auth')
export class AuthController {
  constructor(private readonly authService: AuthService) {}

  // POST /auth/login
  @Post('login')
  @HttpCode(HttpStatus.OK)  // 로그인은 200 (생성이 아니라 조회)
  @ApiOperation({ summary: '로그인' })
  async login(@Body() dto: LoginDto) {
    return this.authService.login(dto.email, dto.password);
  }

  // POST /auth/register
  @Post('register')
  @HttpCode(HttpStatus.CREATED)
  @ApiOperation({ summary: '회원가입' })
  async register(@Body() dto: RegisterDto) {
    return this.authService.register(dto.email, dto.password, dto.nickname);
  }

  // POST /auth/refresh
  @Post('refresh')
  @HttpCode(HttpStatus.OK)
  @ApiOperation({ summary: 'Access Token 재발급' })
  async refresh(@Body() dto: RefreshDto) {
    return this.authService.refresh(dto.refreshToken);
  }

  // POST /auth/logout
  @Post('logout')
  @HttpCode(HttpStatus.NO_CONTENT)
  @ApiOperation({ summary: '로그아웃' })
  async logout(@UserId() userId: string, @Body() dto: LogoutDto) {
    // Refresh Token을 DB에서 삭제 (저장하고 있는 경우)
    await this.authService.logout(userId);
  }
}
```

---

# jwtService.sign · verify — 토큰 생성·검증 ⭐️⭐️⭐️⭐️

## jwtService.sign() — 토큰 생성

```typescript
// payload를 담아서 JWT 토큰 문자열 생성
const token = this.jwtService.sign(payload, options?);
```

```typescript
// 기본 사용 — JwtModule에 설정한 secret과 expiresIn 사용
const accessToken = this.jwtService.sign({ sub: userId });
// JwtModule: secret='my-secret', expiresIn='15m' → 15분짜리 토큰 생성

// options으로 개별 설정 — JwtModule 설정 덮어쓰기
const accessToken  = this.jwtService.sign(
  { sub: userId },
  { expiresIn: '15m' }   // 이 토큰만 15분
);
const refreshToken = this.jwtService.sign(
  { sub: userId },
  { expiresIn: '7d' }    // 이 토큰만 7일
);
```

```txt
sign()이 하는 것:
  { sub: userId }를 Payload에 담고
  비밀키(JWT_SECRET)로 서명해서
  Header.Payload.Signature 형태의 JWT 문자열 반환

  sign() 결과:
  "eyJhbGciOiJIUzI1NiIsInR5cCI6IkpXVCJ9.eyJzdWIiOiJ1c2VyLTEyMyJ9.서명값"

sub:
  Subject의 약자 — JWT 표준 클레임
  "이 토큰의 주체(사용자)"를 나타냄
  userId를 sub에 넣는 것이 관례
```

## jwtService.verify() — 토큰 검증·디코딩

```typescript
// 토큰이 유효하면 payload 반환, 유효하지 않으면 예외 throw
const payload = this.jwtService.verify(token);
```

```typescript
// 실전 사용 — try/catch 필수
async refresh(refreshToken: string) {
  try {
    const payload = this.jwtService.verify(refreshToken);
    //              ↑ 유효하면 { sub: 'user-uuid', iat: 1234, exp: 5678 } 반환
    //              ↑ 유효하지 않으면 예외 throw

    return {
      accessToken: this.jwtService.sign(
        { sub: payload.sub },  // payload에서 userId 꺼내서 새 토큰 생성
        { expiresIn: '15m' },
      ),
    };
  } catch {
    throw new UnauthorizedException('유효하지 않은 토큰입니다.');
  }
}
```

```txt
verify()가 하는 것:
  ① 서명 검증 — 토큰이 이 서버가 발급한 것인지 확인 (비밀키로)
  ② 만료 확인 — exp가 현재 시각보다 이전이면 만료된 토큰
  ③ 성공 시   → payload 객체 반환 { sub, iat, exp, ... }
  ④ 실패 시   → 예외 throw (서명 불일치, 만료, 형식 오류 등)

try/catch가 반드시 필요한 이유:
  토큰이 만료됐거나, 조작됐거나, 형식이 잘못됐으면 예외 발생
  catch 없이 호출하면 서버 500 에러
  → UnauthorizedException으로 변환해서 401로 응답

verify vs decode:
  verify(token)  → 서명 검증 O + payload 반환 (정상 흐름에서 사용)
  decode(token)  → 서명 검증 X + payload 반환 (검증 없이 내용만 볼 때)
  → 반드시 verify 사용, decode는 디버깅 용도로만
```

---

# 비밀번호 해시 — bcrypt ⭐️⭐️⭐️⭐️

```bash
pnpm --filter api add bcrypt
pnpm --filter api add -D @types/bcrypt
```

## bcrypt란

```txt
bcrypt = 비밀번호를 안전하게 저장하기 위한 해시 함수

평문 비밀번호를 DB에 저장하면:
  DB가 유출되면 → 모든 사용자 비밀번호 노출
  → 절대 평문 저장 금지

bcrypt 특징:
  단방향 해시 — 해시에서 원문 복원 불가
  salt(랜덤값)를 섞어서 해시 — 같은 비밀번호도 해시가 매번 다름
  의도적으로 느림 — 브루트포스 공격 방어
```

## hash · compare ⭐️⭐️⭐️⭐️

```typescript
import * as bcrypt from 'bcrypt';

// 비밀번호 저장 시 — 해시 생성
const hashedPassword = await bcrypt.hash(plainPassword, 10);
//                                                       ↑ salt rounds

// 비밀번호 검증 시 — 비교
const isMatch = await bcrypt.compare(plainPassword, hashedPassword);
// true: 비밀번호 일치 / false: 불일치
```

```txt
salt rounds(10)의 의미:
  10 = 2^10 = 1024번 반복 계산
  숫자가 클수록 느리지만 더 안전
  10~12가 일반적 (너무 크면 로그인이 느려짐)

  10라운드 → 약 100ms  ← 일반적인 선택
  12라운드 → 약 400ms

compare()가 느린 이유 (의도적):
  빠르면 초당 수만 번 시도 가능 (브루트포스 공격)
  느리면 초당 수십 번만 가능 → 공격 비용 증가

salt가 같은 비밀번호를 다른 해시로 만드는 이유:
  '$2b$10$랜덤솔트값...' 형태로 salt가 해시 안에 포함됨
  compare()가 해시에서 salt를 추출해서 동일하게 적용 후 비교
  → 같은 비밀번호라도 해시가 달라서 레인보우 테이블 공격 방어
```

## 조건부 해싱 — PATCH에서 비밀번호 변경 시만 ⭐️⭐️⭐️

```typescript
// 비밀번호 변경이 포함된 PATCH
async updateUser(userId: string, dto: UpdateUserDto) {
  const updateData: Prisma.UserUpdateInput = {
    nickname: dto.nickname,
  };

  // 비밀번호가 있을 때만 해시 — 없으면 해시 안 함
  if (dto.password) {
    updateData.hashedPassword = await bcrypt.hash(dto.password, 10);
  }

  return this.prisma.user.update({
    where: { id: userId },
    data:  updateData,
  });
}
```

```txt
왜 조건부 해싱이 필요한가:
  PATCH는 일부 필드만 수정
  비밀번호 없이 닉네임만 변경할 때
  → dto.password가 undefined → 해시 안 함
  → 조건 없이 항상 해시하면 undefined를 해시해서 덮어씌움 → 버그
```

## 응답에서 비밀번호 제거 ⭐️⭐️⭐️

```typescript
// Prisma select로 처음부터 제외 (권장)
async findUser(userId: string) {
  return this.prisma.user.findUnique({
    where:  { id: userId },
    select: {
      id:        true,
      email:     true,
      nickname:  true,
      createdAt: true,
      // hashedPassword 없음 → DB에서 아예 안 가져옴
    },
  });
}

// 또는 구조분해로 제거
const { hashedPassword, ...safeUser } = await this.prisma.user.findUniqueOrThrow({
  where: { id: userId },
});
return safeUser;  // hashedPassword 없는 객체
```

```txt
왜 응답에서 반드시 제거해야 하는가:
  hashedPassword를 API 응답에 포함하면
  → 클라이언트에 해시값 노출
  → 공격자가 오프라인 브루트포스 가능

  Prisma select로 처음부터 안 가져오는 게 가장 안전
  구조분해는 DB에서는 가져오고 JS에서 제거 (약간 비효율)
```

---

# DTO ⭐️⭐️⭐️

```typescript
// login.dto.ts
export class LoginDto {
  @ApiProperty({ description: '이메일' })
  @IsEmail()
  email: string;

  @ApiProperty({ description: '비밀번호' })
  @IsString()
  @MinLength(8)
  password: string;
}

// register.dto.ts
export class RegisterDto {
  @ApiProperty()
  @IsEmail()
  email: string;

  @ApiProperty()
  @IsString()
  @MinLength(8)
  password: string;

  @ApiProperty()
  @IsString()
  @MinLength(2)
  @MaxLength(20)
  nickname: string;
}

// refresh.dto.ts
export class RefreshDto {
  @ApiProperty()
  @IsString()
  @IsNotEmpty()
  refreshToken: string;
}
```

---

# 환경변수

```bash
# .env
JWT_SECRET=your-super-secret-key-here  # 절대 코드에 직접 쓰지 않음
# 길고 랜덤한 문자열로 설정
# 예: openssl rand -base64 32 으로 생성
```

```txt
JWT_SECRET 주의사항:
  짧거나 예측 가능한 문자열 → 브루트포스로 서명 위조 가능
  최소 32자 이상 랜덤 문자열 사용
  절대 git에 커밋하면 안 됨 → .env는 .gitignore에 포함
  운영 환경에서는 환경변수 또는 시크릿 관리 서비스 사용
```

---

# 흐름 요약

```txt
회원가입:
  POST /auth/register (email, password, nickname)
  → 이메일 중복 확인
  → bcrypt.hash(password)
  → DB에 저장
  → Access Token + Refresh Token 발급 → 응답

로그인:
  POST /auth/login (email, password)
  → 이메일로 사용자 조회
  → bcrypt.compare(password, hashedPassword)
  → Access Token + Refresh Token 발급 → 응답

API 호출:
  Authorization: Bearer {accessToken} 헤더
  → JwtAuthGuard가 검증 → UserId 추출
  → [[NestJS_JwtGuard]] 참고

토큰 만료:
  POST /auth/refresh (refreshToken)
  → refreshToken 검증
  → 새 Access Token 발급 → 응답

로그아웃:
  POST /auth/logout
  → DB의 Refresh Token 삭제
  → 이후 refresh 불가 → 사실상 로그아웃
```
```
```

---

# AuthController — 엔드포인트 ⭐️⭐️⭐️⭐️

```typescript
// auth.controller.ts
import { Controller, Post, Body, HttpCode, HttpStatus } from '@nestjs/common';
import { ApiTags, ApiOperation } from '@nestjs/swagger';

@ApiTags('auth')
@Controller('auth')
export class AuthController {
  constructor(private readonly authService: AuthService) {}

  // POST /auth/login
  @Post('login')
  @HttpCode(HttpStatus.OK)  // 로그인은 200 (생성이 아니라 조회)
  @ApiOperation({ summary: '로그인' })
  async login(@Body() dto: LoginDto) {
    return this.authService.login(dto.email, dto.password);
  }

  // POST /auth/register
  @Post('register')
  @HttpCode(HttpStatus.CREATED)
  @ApiOperation({ summary: '회원가입' })
  async register(@Body() dto: RegisterDto) {
    return this.authService.register(dto.email, dto.password, dto.nickname);
  }

  // POST /auth/refresh
  @Post('refresh')
  @HttpCode(HttpStatus.OK)
  @ApiOperation({ summary: 'Access Token 재발급' })
  async refresh(@Body() dto: RefreshDto) {
    return this.authService.refresh(dto.refreshToken);
  }

  // POST /auth/logout
  @Post('logout')
  @HttpCode(HttpStatus.NO_CONTENT)
  @ApiOperation({ summary: '로그아웃' })
  async logout(@UserId() userId: string, @Body() dto: LogoutDto) {
    // Refresh Token을 DB에서 삭제 (저장하고 있는 경우)
    await this.authService.logout(userId);
  }
}
```

---

# jwtService.sign · verify — 토큰 생성·검증 ⭐️⭐️⭐️⭐️

## jwtService.sign() — 토큰 생성

```typescript
// payload를 담아서 JWT 토큰 문자열 생성
const token = this.jwtService.sign(payload, options?);
```

```typescript
// 기본 사용 — JwtModule에 설정한 secret과 expiresIn 사용
const accessToken = this.jwtService.sign({ sub: userId });
// JwtModule: secret='my-secret', expiresIn='15m' → 15분짜리 토큰 생성

// options으로 개별 설정 — JwtModule 설정 덮어쓰기
const accessToken  = this.jwtService.sign(
  { sub: userId },
  { expiresIn: '15m' }   // 이 토큰만 15분
);
const refreshToken = this.jwtService.sign(
  { sub: userId },
  { expiresIn: '7d' }    // 이 토큰만 7일
);
```

```txt
sign()이 하는 것:
  { sub: userId }를 Payload에 담고
  비밀키(JWT_SECRET)로 서명해서
  Header.Payload.Signature 형태의 JWT 문자열 반환

  sign() 결과:
  "eyJhbGciOiJIUzI1NiIsInR5cCI6IkpXVCJ9.eyJzdWIiOiJ1c2VyLTEyMyJ9.서명값"

sub:
  Subject의 약자 — JWT 표준 클레임
  "이 토큰의 주체(사용자)"를 나타냄
  userId를 sub에 넣는 것이 관례
```

## jwtService.verify() — 토큰 검증·디코딩

```typescript
// 토큰이 유효하면 payload 반환, 유효하지 않으면 예외 throw
const payload = this.jwtService.verify(token);
```

```typescript
// 실전 사용 — try/catch 필수
async refresh(refreshToken: string) {
  try {
    const payload = this.jwtService.verify(refreshToken);
    //              ↑ 유효하면 { sub: 'user-uuid', iat: 1234, exp: 5678 } 반환
    //              ↑ 유효하지 않으면 예외 throw

    return {
      accessToken: this.jwtService.sign(
        { sub: payload.sub },  // payload에서 userId 꺼내서 새 토큰 생성
        { expiresIn: '15m' },
      ),
    };
  } catch {
    throw new UnauthorizedException('유효하지 않은 토큰입니다.');
  }
}
```

```txt
verify()가 하는 것:
  ① 서명 검증 — 토큰이 이 서버가 발급한 것인지 확인 (비밀키로)
  ② 만료 확인 — exp가 현재 시각보다 이전이면 만료된 토큰
  ③ 성공 시   → payload 객체 반환 { sub, iat, exp, ... }
  ④ 실패 시   → 예외 throw (서명 불일치, 만료, 형식 오류 등)

try/catch가 반드시 필요한 이유:
  토큰이 만료됐거나, 조작됐거나, 형식이 잘못됐으면 예외 발생
  catch 없이 호출하면 서버 500 에러
  → UnauthorizedException으로 변환해서 401로 응답

verify vs decode:
  verify(token)  → 서명 검증 O + payload 반환 (정상 흐름에서 사용)
  decode(token)  → 서명 검증 X + payload 반환 (검증 없이 내용만 볼 때)
  → 반드시 verify 사용, decode는 디버깅 용도로만
```

---

# 비밀번호 해시 — bcrypt ⭐️⭐️⭐️⭐

```bash
pnpm --filter api add bcrypt
pnpm --filter api add -D @types/bcrypt
```

## bcrypt란

```txt
bcrypt = 비밀번호를 안전하게 저장하기 위한 해시 함수

평문 비밀번호를 DB에 저장하면:
  DB가 유출되면 → 모든 사용자 비밀번호 노출
  → 절대 평문 저장 금지

bcrypt 특징:
  단방향 해시 — 해시에서 원문 복원 불가
  salt(랜덤값)를 섞어서 해시 — 같은 비밀번호도 해시가 매번 다름
  의도적으로 느림 — 브루트포스 공격 방어
```

## hash · compare ⭐️⭐️⭐️⭐️

```typescript
import * as bcrypt from 'bcrypt';

// 비밀번호 저장 시 — 해시 생성
const hashedPassword = await bcrypt.hash(plainPassword, 10);
//                                                       ↑ salt rounds

// 비밀번호 검증 시 — 비교
const isMatch = await bcrypt.compare(plainPassword, hashedPassword);
// true: 비밀번호 일치 / false: 불일치
```

```txt
salt rounds(10)의 의미:
  10 = 2^10 = 1024번 반복 계산
  숫자가 클수록 느리지만 더 안전
  10~12가 일반적 (너무 크면 로그인이 느려짐)

  10라운드 → 약 100ms  ← 일반적인 선택
  12라운드 → 약 400ms

compare()가 느린 이유 (의도적):
  빠르면 초당 수만 번 시도 가능 (브루트포스 공격)
  느리면 초당 수십 번만 가능 → 공격 비용 증가

salt가 같은 비밀번호를 다른 해시로 만드는 이유:
  '$2b$10$랜덤솔트값...' 형태로 salt가 해시 안에 포함됨
  compare()가 해시에서 salt를 추출해서 동일하게 적용 후 비교
  → 같은 비밀번호라도 해시가 달라서 레인보우 테이블 공격 방어
```

## 조건부 해싱 — PATCH에서 비밀번호 변경 시만 ⭐️⭐️⭐️

```typescript
// 비밀번호 변경이 포함된 PATCH
async updateUser(userId: string, dto: UpdateUserDto) {
  const updateData: Prisma.UserUpdateInput = {
    nickname: dto.nickname,
  };

  // 비밀번호가 있을 때만 해시 — 없으면 해시 안 함
  if (dto.password) {
    updateData.hashedPassword = await bcrypt.hash(dto.password, 10);
  }

  return this.prisma.user.update({
    where: { id: userId },
    data:  updateData,
  });
}
```

```txt
왜 조건부 해싱이 필요한가:
  PATCH는 일부 필드만 수정
  비밀번호 없이 닉네임만 변경할 때
  → dto.password가 undefined → 해시 안 함
  → 조건 없이 항상 해시하면 undefined를 해시해서 덮어씌움 → 버그
```

## 응답에서 비밀번호 제거 ⭐️⭐️⭐️

```typescript
// Prisma select로 처음부터 제외 (권장)
async findUser(userId: string) {
  return this.prisma.user.findUnique({
    where:  { id: userId },
    select: {
      id:        true,
      email:     true,
      nickname:  true,
      createdAt: true,
      // hashedPassword 없음 → DB에서 아예 안 가져옴
    },
  });
}

// 또는 구조분해로 제거
const { hashedPassword, ...safeUser } = await this.prisma.user.findUniqueOrThrow({
  where: { id: userId },
});
return safeUser;  // hashedPassword 없는 객체
```

```txt
왜 응답에서 반드시 제거해야 하는가:
  hashedPassword를 API 응답에 포함하면
  → 클라이언트에 해시값 노출
  → 공격자가 오프라인 브루트포스 가능

  Prisma select로 처음부터 안 가져오는 게 가장 안전
  구조분해는 DB에서는 가져오고 JS에서 제거 (약간 비효율)
```

---

# DTO ⭐️⭐️⭐️

```typescript
// login.dto.ts
export class LoginDto {
  @ApiProperty({ description: '이메일' })
  @IsEmail()
  email: string;

  @ApiProperty({ description: '비밀번호' })
  @IsString()
  @MinLength(8)
  password: string;
}

// register.dto.ts
export class RegisterDto {
  @ApiProperty()
  @IsEmail()
  email: string;

  @ApiProperty()
  @IsString()
  @MinLength(8)
  password: string;

  @ApiProperty()
  @IsString()
  @MinLength(2)
  @MaxLength(20)
  nickname: string;
}

// refresh.dto.ts
export class RefreshDto {
  @ApiProperty()
  @IsString()
  @IsNotEmpty()
  refreshToken: string;
}
```

---

# 환경변수

```bash
# .env
JWT_SECRET=your-super-secret-key-here  # 절대 코드에 직접 쓰지 않음
# 길고 랜덤한 문자열로 설정
# 예: openssl rand -base64 32 으로 생성
```

```txt
JWT_SECRET 주의사항:
  짧거나 예측 가능한 문자열 → 브루트포스로 서명 위조 가능
  최소 32자 이상 랜덤 문자열 사용
  절대 git에 커밋하면 안 됨 → .env는 .gitignore에 포함
  운영 환경에서는 환경변수 또는 시크릿 관리 서비스 사용
```

---

# 흐름 요약

```txt
회원가입:
  POST /auth/register (email, password, nickname)
  → 이메일 중복 확인
  → bcrypt.hash(password)
  → DB에 저장
  → Access Token + Refresh Token 발급 → 응답

로그인:
  POST /auth/login (email, password)
  → 이메일로 사용자 조회
  → bcrypt.compare(password, hashedPassword)
  → Access Token + Refresh Token 발급 → 응답

API 호출:
  Authorization: Bearer {accessToken} 헤더
  → JwtAuthGuard가 검증 → UserId 추출
  → [[NestJS_JwtGuard]] 참고

토큰 만료:
  POST /auth/refresh (refreshToken)
  → refreshToken 검증
  → 새 Access Token 발급 → 응답

로그아웃:
  POST /auth/logout
  → DB의 Refresh Token 삭제
  → 이후 refresh 불가 → 사실상 로그아웃
```