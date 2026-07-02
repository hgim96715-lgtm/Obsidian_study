---
aliases:
  - .env
  - 환경변수
  - ConfigModule
  - Joi
tags:
  - NestJS
related:
  - "[[00_NestJS_Ecosystem_HomePage]]"
  - "[[Monorepo_PNPM]]"
  - "[[NestJS_Concept]]"
  - "[[NextJS_Env_Config]]"
  - "[[NestJS_Prisma]]"
  - "[[NestJS_CORS]]"
  - "[[NestJS_Module]]"
---
# NestJS_Env_Config — 환경변수

> [!info] 
> 0단계 .env 만들기 → 1단계 process.env 직접 → 2단계 ConfigModule → 3단계 Joi 검증(선택) → 4단계 키 상수화(선택)
> 3·4단계는 필수가 아님 — 프로젝트 규모에 따라 어디서 멈춰도 된다.

---
# 흐름도

```mermaid-beautiful
flowchart LR
    S0["0 .env · .env.example"] --> S1["1 process.env<br/>직접 · --env-file"]
    S1 --> S2["2 ConfigModule<br/>ConfigService"]
    S2 -.->|선택| S3["3 Joi 검증"]
    S2 -.->|선택| S4["4 env.keys 상수화"]
```

```txt
실선: 단계가 올라갈수록 편의·안전 ↑ — 2단계만으로도 Nest 실무에서 충분한 경우 많음
점선 3·4: 필수값 검증 · 키 오타 방지 — 프로젝트 커지면 추가
```

---

# 0단계 — .env / .env.example 만들기 ⭐️⭐️⭐️

```txt
어떤 프로젝트든 가장 먼저 하는 일 — 프레임워크/라이브러리 선택과 무관
```

```properties
# .env
DATABASE_URL=postgresql://postgres:password@localhost:5432/mydb
PORT=3000
NODE_ENV=development
FRONTEND_URL=http://localhost:3031
```

```txt
규칙: KEY=VALUE (= 앞뒤 공백 없음), 대문자+언더스코어 관례, 주석은 #
반드시 .gitignore에 추가: .env / .env.local / .env.production
```

```properties
# .env.example — 팀 공유용, 값은 비우거나 예시만, Git에 포함
DATABASE_URL=
PORT=3000
NODE_ENV=development
FRONTEND_URL=
```

```txt
FRONTEND_URL처럼 "프론트엔드 주소를 백엔드가 알아야 하는" 변수는 흔한 패턴
  예: FRONTEND_URL, CLIENT_URL, WEB_ORIGIN, CORS_ORIGIN 등 — 보통 CORS 허용 출처 지정에 씀
  ([[NestJS_CORS]] 참고)
```

## .env 파일 종류 ⭐️⭐️⭐️

|파일|용도|Git 포함|
|---|---|---|
|`.env`|모든 환경 공통 기본값|보통 제외|
|`.env.local`|내 컴퓨터에서만 쓰는 로컬 오버라이드, 최우선|항상 제외|
|`.env.development` / `.env.production`|환경별 전용 값|민감값 없으면 포함 가능|
|`.env.example`|키 목록 + 예시값 (실제 값 없음)|항상 포함|

```txt
⚠️ 이 멀티파일 구조는 dotenv 생태계의 일반적인 관례일 뿐, NestJS가 자동으로 다 읽어주는 게 아님
  NestJS ConfigModule 기본값: 루트의 .env 파일 "하나만" 읽음
  여러 파일을 환경별로 읽고 싶다면 envFilePath를 직접 배열로 지정해야 함 (2단계 참고)

  Next.js는 .env/.env.local/.env.development/.env.production을 프레임워크가 자동으로 우선순위 처리
  → NestJS와 동작 방식이 다름 ([[NextJS_Env_Config]] 참고)
```

---

# 1단계 — process.env 직접 사용 (NestJS 도구 없이) ⭐️⭐️

```typescript
const port = process.env.PORT || 3000;
```

```txt
가장 단순한 방법 — ConfigModule 자체가 필요 없음 (Node.js라면 어디서나 동작)
작은 스크립트나 토이 프로젝트는 이 단계로 충분
```

## ⚠️ .env를 누군가 직접 로드해줘야 함 ⭐️⭐️⭐️

```txt
process.env.PORT라고 코드에 적는 것과, .env 파일의 PORT=3000이 실제로 채워지는 것은 별개
Node.js는 .env 파일을 알아서 읽어주지 않음 — 둘을 연결해주는 무언가가 반드시 있어야 함
ConfigModule을 안 쓰는 이 1단계에서는 --env-file 플래그가 그 역할을 함
```

```json
{
  "scripts": {
    "start:dev": "nest start --watch --env-file .env"
  }
}
```

```txt
--env-file .env: Node.js 20.6+ 내장 옵션 (NestJS CLI v11+ 전달 가능)
  실행 시점에 .env를 읽어 process.env에 채워 넣음

⚠️ 2단계(ConfigModule)로 넘어가면 이 플래그는 필요 없어짐
   ConfigModule이 내부적으로 dotenv로 .env를 직접 읽어주기 때문
   --env-file과 ConfigModule은 "둘 중 하나"의 관계

한계:
  값이 항상 string | undefined — 숫자 필요하면 매번 parseInt() 직접 변환
  필수 환경변수가 빠져도 서버가 그냥 시작됨 → 나중에 런타임 에러로 처음 발견
  키 오타나도 그냥 undefined — 컴파일 타임에 못 잡음
```

---

# 2단계 — ConfigModule (NestJS 권장, Joi 없이도 동작) ⭐️⭐️⭐️

```bash
pnpm add @nestjs/config
```

```typescript
// app.module.ts
import { ConfigModule } from '@nestjs/config';

@Module({
  imports: [
    ConfigModule.forRoot({ isGlobal: true }),  // 이것만으로도 동작 — Joi는 선택사항
  ],
})
export class AppModule {}
```

```typescript
@Injectable()
export class SomeService {
  constructor(private configService: ConfigService) {}

  someMethod() {
    const port = this.configService.get<number>('PORT');
  }
}
```

|옵션|설명|
|---|---|
|`isGlobal: true`|AppModule에 1번만 등록하면 모든 모듈에서 ConfigService 바로 사용 — 거의 항상 true|
|`isGlobal: false` (기본)|ConfigService가 필요한 모듈마다 매번 imports에 ConfigModule 추가해야 함|

```txt
1단계보다 나은 점: get<T>()로 타입 지정 가능, dotenv 로드를 직접 안 해도 됨
한계: validationSchema를 안 쓰면 1단계와 마찬가지로 필수값 검증이 없음
```

## envFilePath — 어떤 .env 파일을 읽을지 직접 지정 ⭐️⭐️

```typescript
ConfigModule.forRoot({
  isGlobal: true,
  envFilePath: '.env',  // 문자열 하나 또는 배열로 지정
});
```

```txt
envFilePath를 아예 안 써도 기본값이 이미 프로젝트 루트의 .env
→ envFilePath: '.env'라고 명시적으로 써도 동작은 안 쓴 것과 동일

그런데도 명시적으로 쓰는 이유:
  ① "이 프로젝트는 .env를 여기서 읽는다"는 걸 코드만 보고 바로 알 수 있게 — 문서화 목적
  ② 모노레포 등에서 nest start 실행 위치(cwd)가 매번 같다고 확신하기 어려울 때
  ③ 나중에 경로를 바꾸게 되더라도, 이 한 줄만 고치면 됨
```

## 여러 .env 파일을 환경별로 읽고 싶다면

```typescript
ConfigModule.forRoot({
  isGlobal: true,
  envFilePath: [`.env.${process.env.NODE_ENV}`, '.env'],
  // 앞쪽이 우선 — .env.development에 있으면 그 값, 없으면 .env의 값으로 fallback
});
```

|값 형태|의미|
|---|---|
|문자열 `'.env'`|그 파일 한 개만 읽음|
|배열 `['.env.dev', '.env']`|앞에서부터 순서대로 — 같은 키면 먼저 나온 파일의 값이 우선|

---

# 3단계 — Joi로 필수값 검증 (선택) ⭐️⭐️⭐️

```txt
"배포 전에 필수 환경변수 빠진 걸 미리 잡고 싶다"는 요구가 생기면 추가하는 단계
class-validator로도 같은 걸 할 수 있음 — Joi가 유일한 방법은 아님
```

```bash
pnpm add joi
```

```typescript
// config/env.validation.ts
import * as Joi from 'joi';

export const envValidationSchema = Joi.object({
  DATABASE_URL: Joi.string().uri().required(),
  PORT:         Joi.number().port().default(3000),
  NODE_ENV:     Joi.string().valid('development', 'production', 'test').default('development'),
});
```

```typescript
ConfigModule.forRoot({
  isGlobal:          true,
  envFilePath:       '.env',
  validationSchema:  envValidationSchema,
  validationOptions: { convert: true },  // .env 값은 항상 string → 선언된 타입으로 자동 변환
});
```

|Joi 메서드|역할|
|---|---|
|`.required()`|없으면 서버 시작 실패|
|`.optional()`|없어도 됨|
|`.default(값)`|없으면 그 값 사용|
|`.valid(...)`|허용값 제한|
|`.uri()` / `.email()` / `.port()`|형식 검증|

```txt
validationSchema 없을 때 vs 있을 때:
  없음: DATABASE_URL 빠져도 서버 시작됨 → DB 연결 시도하다 런타임 에러 (원인 파악 어려움)
  있음: 서버 시작 즉시 검사 → 없으면 명확한 에러 메시지로 시작 자체가 실패 (배포 전에 발견)

validationOptions: { convert: true }가 거의 항상 필요한 이유:
  PORT=5432는 .env에서 문자열 '5432'로 들어옴 → Joi.number() 검증 시 타입 불일치로 막힘
  convert: true가 있어야 '5432' → 5432(number)로 자동 변환 후 검증 통과
```

## 대안 — class-validator (Joi 안 쓰고 싶다면)

```txt
NestJS ConfigModule은 validationSchema(Joi) 대신 validate 함수도 받을 수 있음
이미 프로젝트에서 class-validator를 쓰고 있다면(DTO 검증 등) 새 라이브러리 추가 없이 통일 가능
둘 중 어느 쪽이 정답은 아니고, 이미 쓰는 도구에 맞춰 선택하는 문제
```

---

# 4단계 — 키를 상수로 관리 (선택, 키가 많아지면) ⭐️

```typescript
// config/env.keys.ts
export const EnvKeys = {
  DATABASE_URL:  'DATABASE_URL',
  PORT:          'PORT',
  NODE_ENV:      'NODE_ENV',
  FRONTEND_URL:  'FRONTEND_URL',
} as const;
```

```typescript
configService.get(EnvKeys.NODE_ENV)  // 오타 나면 TS 에러로 바로 발견
configService.get('NODE_ENV')        // 오타 나도 그냥 undefined 반환, 발견 늦음
```

```txt
as const: 객체를 읽기 전용으로 만들고, 값이 string이 아닌 리터럴 타입으로 추론되게 함
키 이름을 바꿀 때도 env.keys.ts 한 곳만 고치면 전체 반영 (문자열 직접 쓰면 전체 검색/치환 필요)

Joi 스키마에서도 재사용 가능:
  Joi.object({ [EnvKeys.DATABASE_URL]: Joi.string().uri().required() })
```

---

# ConfigService — get vs getOrThrow ⭐️

```typescript
const host  = this.configService.get<string>('POSTGRES_USER');        // 없으면 undefined
const dbUrl = this.configService.getOrThrow<string>('DATABASE_URL');  // 없으면 에러 발생
```

```txt
validationSchema로 이미 필수값을 검증했다면 런타임에 undefined를 받을 일은 거의 없음
그래도 getOrThrow를 권장하는 이유:
  "이 값은 반드시 있어야 한다"는 의도를 코드에서 바로 드러냄
```

|방식|값 없을 때|타입|
|---|---|---|
|`process.env.X`|`undefined`|항상 `string \| undefined`, 직접 변환 필요|
|`configService.get<T>(X)`|`undefined`|제네릭으로 타입 지정, `convert: true`면 자동 변환|
|`configService.getOrThrow<T>(X)`|에러 발생|위와 동일, 누락을 코드 레벨에서도 명시|

---

# 같은 .env를 읽는 여러 주체 ⭐️⭐️⭐️

```txt
NestJS + Prisma 조합에서는 같은 .env 파일 하나를 실행 컨텍스트마다 완전히 다른 방식으로 읽음
"왜 이 값은 ConfigService로 검증되는데 저건 안 되지?" 가 헷갈리는 이유가 이 차이를 모를 때임
```

|읽는 주체|언제 실행|로딩 방식|Joi 검증 적용?|
|---|---|---|---|
|NestJS 런타임 (main.ts 부팅)|`pnpm start:dev` 등 앱 실행 시|ConfigModule (내부적으로 dotenv)|✅ validationSchema 등록했다면|
|Prisma CLI (`migrate`, `generate`)|`prisma migrate dev` 등 CLI 실행 시|`prisma.config.ts` 안의 `dotenv/config` (Nest 부팅 없음)|❌ ConfigModule과 완전히 무관|
|Docker Compose (`env_file:`)|`docker compose up` 으로 컨테이너 시|compose가 직접 파일을 읽어 환경변수로 주입|❌ 마찬가지로 무관|

```txt
→ 같은 DATABASE_URL이라는 키라도 "누가 지금 이걸 읽고 있는가"에 따라
  검증을 거치는지, 심지어 같은 .env 파일을 보는지조차 다를 수 있음

Prisma CLI가 ConfigModule/Joi를 안 타는 이유:
  prisma migrate/generate는 NestJS 앱을 부팅하지 않고 독립적으로 실행되는 별개 프로세스
  → ConfigModule이 등록될 기회 자체가 없음
  → 그래서 prisma.config.ts가 dotenv로 직접 .env를 읽음
  ([[NestJS_Prisma]] 참고)
```

---

# 자주 하는 실수

|실수|원인|해결|
|---|---|---|
|다른 모듈에서 ConfigService 못 씀|`isGlobal: false`|`isGlobal: true` 설정|
|숫자 값이 string으로 옴|`.env`는 항상 string|`validationOptions: { convert: true }`|
|서버 시작은 되는데 런타임 에러|validationSchema 없음|3단계 검증 추가|
|키 오타로 undefined 반환|문자열 직접 사용|EnvKeys 상수로 교체 (4단계)|
|`.env`가 Git에 올라감|.gitignore 누락|`.env`, `.env.local` 추가|
|환경별 값이 안 갈림|.env 파일 하나만 사용|`envFilePath: ['.env.${NODE_ENV}', '.env']`|

---

# 어느 단계까지 필요한가

|상황|권장|
|---|---|
|로컬 스크립트, 토이 프로젝트|1단계 (process.env)|
|일반적인 NestJS 프로젝트 시작|2단계 (ConfigModule, Joi 없이)|
|배포 전 필수 환경변수 누락을 미리 잡고 싶음|3단계 (+ Joi 또는 class-validator)|
|환경변수가 많고 여러 서비스에서 재사용|4단계 (+ env.keys.ts)|

```txt
처음부터 0~4단계를 다 갖출 필요는 없음 — 프로젝트 초반엔 2단계만으로도 충분히 동작함
```

---

# 한눈에

```txt
0단계: .env/.env.local/.env.example
  모든 프로젝트의 시작, 프레임워크 무관
  .gitignore에 .env / .env.local 추가 필수

1단계: process.env 직접
  가장 단순하지만, .env를 실제로 로드해줄 --env-file 플래그가 따로 필요

2단계: ConfigModule(isGlobal: true)
  내부적으로 dotenv가 로드까지 알아서 함, --env-file 불필요
  envFilePath 명시는 문서화·모노레포 안정성 목적

3단계: Joi(또는 class-validator) 검증 — 선택
  필수값을 배포 전에 잡고 싶을 때
  convert: true 없으면 숫자/불리언 타입 검증 실패

4단계: EnvKeys 상수화 — 선택
  키가 많아지고 오타 방지가 필요할 때
  as const로 리터럴 타입 고정

get() → 없으면 undefined / getOrThrow() → 없으면 에러 (의도 표현에 권장)

같은 .env를 읽는 주체가 다르다:
  NestJS 런타임 → ConfigModule (Joi 검증 O)
  Prisma CLI    → prisma.config.ts dotenv (Joi 검증 X)
  Docker Compose → compose 직접 주입 (Joi 검증 X)

Next.js 환경변수(NEXT_PUBLIC_ 접두사 규칙) → [[NextJS_Env_Config]]
```