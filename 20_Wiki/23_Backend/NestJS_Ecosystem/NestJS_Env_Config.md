---
aliases: [.env, 환경변수, ConfigModule, EnvKeys, Joi]
tags: [NestJS]
related:
  - "[[00_NestJS_Ecosystem_HomePage]]"
  - "[[Monorepo_PNPM]]"
  - "[[NestJS_Concept]]"
---
# NestJS_Env_Config — 환경변수 · ConfigModule

>[!info]
>환경변수 = 코드에 직접 쓰면 안 되는 값(DB URL, JWT 시크릿, API 키 등)을 외부에서 주입하는 방법. 
>`@nestjs/config`의 `ConfigModule`로 `.env`를 읽고, `ConfigService`로 꺼낸다. 
>`EnvKeys` 상수로 오타 방지. 
>`getOrThrow`로 누락된 환경변수를 즉시 발견. 
>`Joi` 스키마로 앱 시작 시 타입·형식·기본값까지 검증한다.

---

# 환경변수란 — 왜 필요한가 ⭐️⭐️⭐️⭐️

```txt
문제 1 — 보안:
  const db = new PrismaClient({
    datasourceUrl: 'postgresql://admin:password123@db.railway.app/mydb'
  });
  → 비밀번호가 코드에 있으면 git에 올라가서 공개됨

문제 2 — 환경별 다른 값:
  로컬 개발:  DATABASE_URL = postgresql://localhost:5432/mydb_dev
  운영 서버:  DATABASE_URL = postgresql://prod-server/mydb_prod
  → 코드가 같아야 하는데 값이 다름

환경변수가 해결:
  값을 코드 밖에 저장 (로컬은 .env 파일, 운영은 서버 설정)
  코드는 "DATABASE_URL이라는 변수에서 읽어라"만 알면 됨
  → 코드 수정 없이 환경마다 다른 값 사용 가능
  → 비밀값이 git에 올라가지 않음
```

---

# .env 파일 ⭐️⭐️⭐️⭐️

```bash
# apps/api/.env
# key=value 형태 (따옴표 불필요, 공백 없음)

# 서버
PORT=3030
NODE_ENV=development

# DB
DATABASE_URL=postgresql://user:password@localhost:5432/mydb

# 인증
JWT_SECRET=my-super-secret-key-minimum-32-chars
SESSION_SECRET=session-secret-key

# 프론트엔드
FRONTEND_URL=http://localhost:3031

# 메일
MAIL_PROVIDER=gmail
MAIL_FROM=noreply@myapp.com
MAIL_USER=my@gmail.com
MAIL_PASS=app-specific-password
SUPPORT_EMAIL=support@myapp.com
```

```txt
.gitignore에 반드시 추가:
  .env
  .env.local
  .env.production
  → 이 파일들은 절대 git에 올리면 안 됨

.env.example은 올려도 됨 (값 없이 키만):
  # .env.example (이 파일은 git에 올림)
  PORT=
  DATABASE_URL=
  JWT_SECRET=
  → 팀원들이 "어떤 환경변수가 필요한지" 파악하는 용도
```

---

# 설치 ⭐️⭐️

```bash
pnpm --filter api add @nestjs/config
# dotenv가 내부적으로 포함됨 — 별도 설치 불필요
```

---

# ConfigModule 설정 ⭐️⭐️⭐️⭐️

```typescript
// app.module.ts
import { ConfigModule } from '@nestjs/config';

@Module({
  imports: [
    ConfigModule.forRoot({
      isGlobal:   true,    // 모든 모듈에서 ConfigService 주입 가능
      envFilePath: '.env', // .env 파일 위치 (기본값, 생략 가능)
    }),
    ...
  ],
})
export class AppModule {}
```

```txt
isGlobal: true:
  ConfigModule을 전역으로 등록
  → 모든 모듈에서 imports 없이 ConfigService 주입 가능
  → false면 ConfigService가 필요한 모든 모듈에
    imports: [ConfigModule]을 추가해야 함
  → 거의 항상 true로 설정

ConfigModule이 하는 것:
  ① .env 파일을 읽어서 파싱
  ② process.env에 값을 주입
  ③ ConfigService를 통해 값을 꺼낼 수 있게 등록
```

---

# EnvKeys — 타입 안전한 키 관리 ⭐️⭐️⭐️⭐️

```typescript
// config/env.keys.ts
export const EnvKeys = {
  // 서버
  PORT:             'PORT',
  NODE_ENV:         'NODE_ENV',

  // DB
  DATABASE_URL:     'DATABASE_URL',

  // 인증
  JWT_SECRET:       'JWT_SECRET',
  SESSION_SECRET:   'SESSION_SECRET',

  // 프론트엔드
  FRONTEND_URL:     'FRONTEND_URL',

  // 메일
  MAIL_PROVIDER:    'MAIL_PROVIDER',
  MAIL_FROM:        'MAIL_FROM',
  MAIL_USER:        'MAIL_USER',
  MAIL_PASS:        'MAIL_PASS',
  SUPPORT_EMAIL:    'SUPPORT_EMAIL',
} as const;
```

```txt
왜 EnvKeys를 쓰는가:

  ❌ 문자열 직접 사용
  configService.getOrThrow('JTW_SECRET')  // 오타 → 런타임에 undefined
  configService.getOrThrow('jwt_secret')  // 대소문자 틀림 → 못 찾음

  ✅ EnvKeys 상수 사용
  configService.getOrThrow(EnvKeys.JWT_SECRET)
  → 오타 시 TypeScript 에러 + IDE 자동완성

as const:
  객체 값이 string이 아닌 리터럴 타입으로 추론됨
  EnvKeys.JWT_SECRET → 타입이 string이 아닌 'JWT_SECRET'
  → TypeScript가 더 정확하게 타입 검사 가능
```

---

# ConfigService — 환경변수 읽기 ⭐️⭐️⭐️⭐️

```typescript
import { Injectable } from '@nestjs/common';
import { ConfigService } from '@nestjs/config';
import { EnvKeys }       from '../config/env.keys';

@Injectable()
export class AuthService {
  constructor(private readonly configService: ConfigService) {}

  someMethod() {
    // get — 없으면 undefined 반환
    const port = this.configService.get<number>(EnvKeys.PORT);
    // port가 undefined여도 앱은 계속 실행 (나중에 에러 날 수 있음)

    // getOrThrow — 없으면 즉시 에러 (권장)
    const secret = this.configService.getOrThrow<string>(EnvKeys.JWT_SECRET);
    // JWT_SECRET이 .env에 없으면 앱 시작 시 바로 크래시
  }
}
```

```txt
get<T>(key):
  값이 있으면 반환, 없으면 undefined
  → 없어도 앱이 실행됨 → 나중에 예상치 못한 에러 발생 가능

getOrThrow<T>(key):
  값이 있으면 반환, 없으면 즉시 에러 throw
  → 앱 시작 시 누락된 환경변수를 바로 발견
  → 운영 배포 시 환경변수 빠뜨리면 즉시 크래시 → 빠른 발견
  → 권장 방법

<string> 제네릭:
  TypeScript 타입 힌트 — 값을 실제로 변환하지는 않음
  환경변수는 항상 문자열로 읽힘
  숫자가 필요하면 직접 변환:
    const port = parseInt(configService.getOrThrow(EnvKeys.PORT), 10);
    또는
    configService.get<number>(EnvKeys.PORT) ?? 3030
```

---

# forRootAsync — 환경변수로 모듈 설정 ⭐️⭐️⭐️⭐️

```typescript
// JwtModule을 환경변수로 설정
JwtModule.registerAsync({
  imports:    [ConfigModule],
  inject:     [ConfigService],
  useFactory: (config: ConfigService) => ({
    secret:      config.getOrThrow<string>(EnvKeys.JWT_SECRET),
    signOptions: { expiresIn: '15m' },
  }),
})

// TypeOrmModule, MailerModule 등도 동일한 패턴
SomeModule.forRootAsync({
  imports:    [ConfigModule],
  inject:     [ConfigService],
  useFactory: (config: ConfigService) => ({
    // 환경변수에서 읽어서 설정
  }),
})
```

```txt
왜 forRootAsync가 필요한가:
  forRoot({ secret: '...' })는 코드에 직접 값을 넣음 → 안 됨
  환경변수는 런타임에 주입됨
  모듈 정의 시점에는 ConfigService가 아직 준비 안 됨
  → forRootAsync + useFactory로 ConfigService를 주입받아서 그때 읽음
```

---

# main.ts에서 ConfigService 사용 ⭐️⭐️⭐️

```typescript
// main.ts — DI가 안 되는 곳에서 ConfigService 쓰기
async function bootstrap() {
  const app = await NestFactory.create<NestExpressApplication>(AppModule);
  const configService = app.get(ConfigService);
  //                         ↑ DI 컨테이너에서 직접 꺼냄

  const port        = configService.get<number>(EnvKeys.PORT) ?? 3030;  // 사용안하면 생략 가능  
  const frontendUrl = configService.get<string>(EnvKeys.FRONTEND_URL);
  const frontendOrigin = frontendUrl ? new URL(frontendUrl).origin : undefined;

  app.enableCors({
    origin: frontendOrigin
      ? ['http://localhost:3031', frontendOrigin]
      : undefined,
    credentials: true,
  });

  app.use(
    session({
      secret: configService.getOrThrow<string>(EnvKeys.SESSION_SECRET),
      resave: false,
      saveUninitialized: false,
      cookie: { httpOnly: true, sameSite: 'lax' },
    }),
  );

  await app.listen(port);
}
```

```txt
main.ts는 constructor DI가 안 됨
app.get(ConfigService)로 컨테이너에서 직접 꺼내서 사용

ConfigModule.forRoot({ isGlobal: true })가 먼저 실행돼야
app.get(ConfigService)가 가능하므로
AppModule imports 맨 위에 ConfigModule이 있어야 함
```

---

# process.env vs ConfigService ⭐️⭐️⭐️

```typescript
// process.env — Node.js 기본 방식
const secret = process.env.JWT_SECRET;  // string | undefined

// ConfigService — NestJS 방식 (권장)
const secret = configService.getOrThrow<string>(EnvKeys.JWT_SECRET);
```

```txt
process.env를 직접 쓰면 되는 경우:
  main.ts 이전 (NestJS가 초기화되기 전)
  ConfigModule이 없는 스크립트 (dump-openapi.ts 등)

ConfigService를 써야 하는 경우:
  NestJS 서비스·모듈 안 (DI 가능한 곳)
  타입 힌트 + getOrThrow로 안전하게 읽고 싶을 때

NestJS에서 ConfigModule.forRoot()를 실행하면:
  .env 파일을 읽어서 process.env에 주입
  → ConfigService도 결국 process.env에서 읽음
  → 둘 다 같은 값을 참조
```

---

# 운영 환경 배포 ⭐️⭐️⭐️

```txt
로컬 개발:
  .env 파일에서 읽음
  dotenv가 파일 파싱 → process.env에 주입

Railway · Vercel 등 배포:
  .env 파일을 서버에 올리지 않음 (gitignore)
  대시보드에서 직접 환경변수 설정
  서버가 시작될 때 OS가 process.env에 주입

  Railway: 프로젝트 → Variables 탭
  Vercel:  프로젝트 → Settings → Environment Variables
```

---

# Joi — 환경변수 유효성 검증 ⭐️⭐️⭐️⭐️

## Joi란

```txt
Joi = JavaScript/TypeScript 객체 스키마 검증 라이브러리
  "이 값은 숫자여야 한다"
  "이 값은 필수이고 최소 32자 이상이어야 한다"
  "이 값이 없으면 기본값 3030을 써라"

  → 앱이 시작될 때 환경변수가 올바른 형태인지 검사
  → 잘못된 환경변수로 앱이 이상하게 동작하는 것을 미리 방지
```

```bash
pnpm --filter api add joi
```

## Joi 스키마 주요 타입과 메서드

```typescript
import * as Joi from 'joi';

// 타입
Joi.string()    // 문자열
Joi.number()    // 숫자
Joi.boolean()   // boolean
Joi.array()     // 배열

// 자주 쓰는 메서드
.required()     // 필수 (없으면 에러)
.optional()     // 선택 (없어도 됨)
.default(값)    // 없으면 이 기본값 사용
.min(n)         // 최솟값 (숫자) 또는 최소 길이 (문자열)
.max(n)         // 최댓값 또는 최대 길이
.valid('a','b') // 허용되는 값 목록 (enum처럼)
.uri()          // URL 형식
.email()        // 이메일 형식
```

## env.validation.ts 작성 ⭐️⭐️⭐️⭐️

```typescript
// config/env.validation.ts
import * as Joi from 'joi';
import { EnvKeys } from './env.keys';

export const envValidationSchema = Joi.object({
  // 서버
  [EnvKeys.PORT]:     Joi.number().default(3030),
  [EnvKeys.NODE_ENV]: Joi.string().valid('development', 'production', 'test').default('development'),

  // DB — 필수
  [EnvKeys.DATABASE_URL]: Joi.string().required(),

  // PostgreSQL (Docker용 — 선택)
  [EnvKeys.POSTGRES_PORT]:     Joi.number().port().optional(),
  [EnvKeys.POSTGRES_USER]:     Joi.string().optional(),
  [EnvKeys.POSTGRES_PASSWORD]: Joi.string().optional(),
  [EnvKeys.POSTGRES_DB]:       Joi.string().optional(),

  // 인증 — 필수 + 최소 길이
  [EnvKeys.JWT_SECRET]:     Joi.string().min(32).required(),
  [EnvKeys.SESSION_SECRET]: Joi.string().min(32).required(),

  // 프론트엔드 — 선택 (없으면 CORS 열림)
  [EnvKeys.FRONTEND_URL]: Joi.string().uri().optional(),

  // 메일 — 선택
  [EnvKeys.MAIL_PROVIDER]: Joi.string().valid('gmail', 'icloud').optional(),
  [EnvKeys.MAIL_USER]:     Joi.string().email().optional(),
  [EnvKeys.MAIL_PASS]:     Joi.string().optional(),
  [EnvKeys.MAIL_FROM]:     Joi.string().email().optional(),
  [EnvKeys.SUPPORT_EMAIL]: Joi.string().email().optional(),
});
```

```txt
[EnvKeys.PORT] 문법:
  대괄호 표기법 = 동적 키 이름
  EnvKeys.PORT = 'PORT' 이므로
  { [EnvKeys.PORT]: ... } = { PORT: ... }
  → EnvKeys 상수를 Joi 스키마 키로 재사용 (오타 방지)
```

## ConfigModule에 연결 ⭐️⭐️⭐️⭐️

```typescript
// app.module.ts
import { ConfigModule } from '@nestjs/config';
import { envValidationSchema } from './config/env.validation';

@Module({
  imports: [
    ConfigModule.forRoot({
      isGlobal:         true,
      envFilePath: '.env', // .env 파일 위치 (기본값, 생략 가능)
      validationSchema: envValidationSchema,  // Joi 스키마로 env 검증
      validationOptions: {
        convert: true,   // 문자열 → 스키마 타입으로 자동 변환
      },
    }),
  ],
})
export class AppModule {}
```

```txt
validationOptions.convert: true:
  환경변수는 항상 문자열로 읽힘
  ("3030"은 숫자가 아니라 문자열 "3030")

  convert: true 없으면:
  PORT=3030 → process.env.PORT = "3030" (string)
  Joi.number()로 검증하면 타입 불일치 경고
  configService.get<number>(EnvKeys.PORT) → "3030" (여전히 문자열)

  convert: true 있으면:
  PORT=3030 → Joi가 "3030"을 3030 (number)로 변환
  configService.get<number>(EnvKeys.PORT) → 3030 (실제 number)

  default()도 같이 작동:
  PORT 없음 → Joi가 default(3030)으로 채워줌 → 3030 (number)
```

```txt
validationSchema가 있으면:
  앱이 시작될 때 .env 파일의 값을 스키마로 검사
  → 필수 값이 없으면 즉시 에러 + 앱 시작 실패
  → 타입이 맞지 않으면 즉시 에러

  에러 예시:
  ValidationError: "DATABASE_URL" is required
  ValidationError: "JWT_SECRET" length must be at least 32 characters long

  → 운영 배포 시 환경변수를 빠뜨리면 바로 발견 (getOrThrow와 같은 목적)

default() 동작:
  .default(3030) → .env에 PORT가 없어도 3030으로 설정
  → ConfigService.get(EnvKeys.PORT) = 3030
```

## Joi vs getOrThrow — 둘 다 쓰는 이유

```txt
getOrThrow(key):
  값을 읽을 때마다 개별 검사
  "이 키가 없으면 에러"만 가능

Joi validationSchema:
  앱 시작 시 한 번에 전체 검사
  타입 검사 (number, string 등)
  형식 검사 (URL, email, min 길이 등)
  기본값 설정

  예시: JWT_SECRET이 있는데 10자짜리면
  → getOrThrow는 통과 (값이 있으니까)
  → Joi .min(32)는 에러 (너무 짧음)

함께 쓰면:
  Joi → 앱 시작 시 구조·형식·기본값 검증
  getOrThrow → 코드에서 읽을 때 타입 안전하게 접근
```

---

| 에러                           | 원인                     | 해결                                                       |
| ---------------------------- | ---------------------- | -------------------------------------------------------- |
| `ConfigService is undefined` | ConfigModule 미등록       | AppModule에 `ConfigModule.forRoot({ isGlobal: true })` 추가 |
| `getOrThrow` 에러로 앱 시작 실패     | .env에 필수 환경변수 없음       | .env에 해당 키·값 추가                                          |
| 로컬은 되는데 배포 시 에러              | 배포 환경에 환경변수 미설정        | Railway/Vercel 대시보드에서 환경변수 추가                            |
| `process.env.X`가 undefined   | ConfigModule 초기화 전에 접근 | bootstrap() 안에서 app.get(ConfigService) 사용                |