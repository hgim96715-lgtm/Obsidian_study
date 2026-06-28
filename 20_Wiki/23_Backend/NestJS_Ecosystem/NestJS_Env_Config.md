---
aliases: [.env, 환경변수, ConfigModule, Joi]
tags:
  - NestJS
related:
  - "[[00_NestJS_Ecosystem_HomePage]]"
  - "[[Monorepo_PNPM]]"
  - "[[NestJS_Concept]]"
  - "[[NextJS_Env_Config]]"
---
# NestJS_Env_Config — 환경변수

# 한 줄 요약

```txt
환경변수 = 코드 밖에서 주입하는 설정값 (DB 비밀번호 / API 키 / 포트 / 환경 구분)
이 노트 순서 = 실제로 점진적으로 늘려가는 순서:
  0단계 .env 파일 만들기 → 1단계 process.env 직접 → 2단계 ConfigModule → 3단계 Joi 검증(선택) → 4단계 키 상수화(선택)
3, 4단계는 필수가 아님 — 프로젝트 규모에 따라 어디서 멈춰도 됨
```

---

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
반드시 .gitignore 에 추가: .env / .env.local / .env.production
```

```properties
# .env.example — 팀 공유용, 값은 비우거나 예시만, Git 에 포함
DATABASE_URL=
PORT=3000
NODE_ENV=development
FRONTEND_URL=
```

```txt
FRONTEND_URL 처럼 "프론트엔드 주소를 백엔드가 알아야 하는" 변수는 흔한 패턴(이름은 프로젝트마다 다름)
  예: FRONTEND_URL, CLIENT_URL, WEB_ORIGIN, CORS_ORIGIN 등 — 보통 CORS 허용 출처 지정에 씀 ([[NestJS_CORS]] 참고)
```

## .env 파일 종류 — .env.local 이 뭔지 ⭐️⭐️⭐️

|파일|용도|Git 포함|
|---|---|---|
|`.env`|모든 환경 공통 기본값|보통 제외(프로젝트에 따라 예시값만 넣고 포함하기도 함)|
|`.env.local`|내 컴퓨터에서만 쓰는 로컬 오버라이드, 최우선 적용|항상 제외|
|`.env.development` / `.env.production`|환경별 전용 값|제외 또는 포함(민감값 없으면 포함 가능)|
|`.env.example`|키 목록 + 예시값(실제 값 없음)|항상 포함|

```txt
⚠️ 이 멀티파일 구조는 dotenv 생태계의 일반적인 관례일 뿐, NestJS 가 자동으로 다 읽어주는 게 아님
  NestJS ConfigModule 기본값: 루트의 .env 파일 "하나만" 읽음
  여러 파일을 환경별로 읽고 싶다면 envFilePath 를 직접 배열로 지정해야 함 (아래 2단계 참고)
  (Next.js 는 .env/.env.local/.env.development/.env.production 을 프레임워크가 자동으로 우선순위 처리해줌
   — NestJS 와 동작 방식이 다르다는 것만 기억하면 됨)
```

---

---

# 1단계 — process.env 직접 사용 (NestJS 도구 없이) ⭐️⭐️

```typescript
const port = process.env.PORT || 3000;
```

```txt
가장 단순한 방법 — ConfigModule 자체가 필요 없음 (Node.js 라면 어디서나 동작)
작은 스크립트나 토이 프로젝트는 이 단계로 충분
```

## ⚠️ 이 단계에서는 .env 를 누군가 직접 로드해줘야 함 ⭐️⭐️⭐️

```txt
process.env.PORT 라고 코드에 적는 것과, .env 파일의 PORT=3000 이 실제로 그 자리에 채워지는 것은 별개임
Node.js 는 .env 파일을 "알아서" 읽어주지 않음 — 둘을 연결해주는 무언가가 반드시 있어야 함
ConfigModule 을 안 쓰는 이 1단계에서는 그 역할을 보통 --env-file 플래그가 함
```

```json
{
  "scripts": {
    "start:dev": "nest start --watch --env-file .env"
  }
}
```

```txt
--env-file .env: Node.js 20.6+ 내장 옵션을 nest start 가 그대로 전달받아 사용 (NestJS CLI v11+)
  실행 시점에 .env 를 읽어 process.env 에 채워 넣음 — 이게 있어야 process.env.PORT 가 실제 값을 가짐
  (스크립트 작성 예시 전체는 [[NestJS_Concept]] 의 "PORT — 포트 충돌 처리" 참고)

⚠️ 2단계(ConfigModule) 로 넘어가면 이 플래그는 필요 없어짐 — ConfigModule 이 내부적으로
   dotenv 로 .env 를 직접 읽어서 채워주기 때문 (--env-file 과 ConfigModule 은 같은 일을 하는
   "둘 중 하나" 의 관계 — 같이 쓸 필요는 없음)
```

```txt
한계 (이 단계 전체):
  값이 항상 string | undefined — 숫자 필요하면 매번 직접 parseInt() 등으로 변환
  필수 환경변수가 빠져도 서버가 그냥 시작됨 → 나중에 DB 연결 시도하다 런타임 에러로 처음 발견
  키 오타 나도 그냥 undefined — 컴파일 타임에 못 잡음
```

---

---

# 2단계 — ConfigModule (NestJS 권장 기본, Joi 없이도 동작) ⭐️⭐️⭐️

```bash
pnpm add @nestjs/config
```

```typescript
// app.module.ts
import { ConfigModule } from '@nestjs/config';

@Module({
  imports: [
    ConfigModule.forRoot({ isGlobal: true }),   // 이것만으로도 동작 — Joi 는 선택사항
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
|`isGlobal: true`|`AppModule` 에 1번만 등록하면 모든 모듈에서 `ConfigService` 바로 사용 — 거의 항상 true|
|`isGlobal: false`(기본)|`ConfigService` 가 필요한 모듈마다 매번 `imports` 에 `ConfigModule` 추가해야 함|

```txt
이 단계가 1단계보다 나은 점: get<T>() 로 타입을 지정할 수 있고, dotenv 로드를 직접 안 해도 됨
이 단계의 한계: validationSchema 를 안 쓰면 1단계와 마찬가지로 필수값 검증이 없음
```

## envFilePath — 어떤 .env 파일을 읽을지 직접 지정 ⭐️⭐️

```typescript
ConfigModule.forRoot({
  isGlobal: true,
  envFilePath: '.env',   // 문자열 하나만 줄 수도 있음
});
```

```txt
⚠️ envFilePath 를 아예 안 써도 기본값이 이미 프로젝트 루트의 .env 임
   → envFilePath: '.env' 라고 명시적으로 써도 동작은 안 쓴 것과 동일 (의미상 변화 없음)

그런데도 명시적으로 쓰는 이유:
  ① "이 프로젝트는 .env 를 여기서 읽는다" 는 걸 코드만 보고 바로 알 수 있게 — 문서화 목적
  ② 모노레포 등에서 nest start 를 실행하는 위치(cwd)가 매번 같다고 확신하기 어려울 때,
     상대경로를 명시해두면 "어디서 실행해도 이 경로를 본다" 는 의도를 코드로 고정함
  ③ 나중에 경로를 바꾸게 되더라도, 기본값에 의존하지 않고 이 한 줄만 고치면 됨
```

## 여러 .env 파일을 환경별로 읽고 싶다면 — 배열로

```typescript
ConfigModule.forRoot({
  isGlobal: true,
  envFilePath: [`.env.${process.env.NODE_ENV}`, '.env'],
  // 배열의 앞쪽이 우선 — .env.development 에 있으면 그 값, 없으면 .env 의 값으로 fallback
});
```

|값 형태|의미|
|---|---|
|문자열 하나 (`'.env'`)|그 파일 한 개만 읽음 (기본값과 동일한 경로면 사실상 명시적 표기 목적)|
|배열 (`['.env.dev', '.env']`)|앞에서부터 순서대로 — 같은 키가 여러 파일에 있으면 먼저 나온 파일의 값이 우선|

---

---

# 3단계 — Joi 로 필수값 검증 (선택사항) ⭐️⭐️⭐️

```txt
Joi 는 필수가 아님 — "배포 전에 필수 환경변수 빠진 걸 미리 잡고 싶다" 는 요구가 생기면 추가하는 단계
class-validator 로도 같은 걸 할 수 있음(아래 대안 참고) — Joi 가 유일한 방법은 아님
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
  isGlobal: true,
  envFilePath: '.env',                    // 명시 안 해도 기본값과 동일 — 위 "envFilePath" 섹션 참고
  validationSchema: envValidationSchema,
  validationOptions: { convert: true },   // .env 값은 항상 string → 선언된 타입으로 자동 변환
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
  없음: .env 에 DATABASE_URL 빠져도 서버 시작됨 → DB 연결 시도하다 런타임 에러 (원인 파악 어려움)
  있음: 서버 시작 즉시 검사 → 없으면 명확한 에러 메시지로 시작 자체가 실패 (배포 전에 미리 발견)

validationOptions: { convert: true } 가 거의 항상 필요한 이유:
  PORT=5432 는 .env 에서 문자열 '5432' 로 들어옴 → Joi.number() 검증 시 타입 불일치로 막힘
  convert: true 가 있어야 '5432' → 5432(number) 로 자동 변환된 후 검증 통과
```

## 대안 — class-validator 로 검증 (Joi 안 쓰고 싶다면)

```txt
NestJS ConfigModule 은 validationSchema(Joi) 대신 validate 함수도 받을 수 있음
이미 프로젝트에서 class-validator 를 쓰고 있다면(DTO 검증 등) 새 라이브러리 추가 없이 같은 도구로 통일 가능
→ 둘 중 어느 쪽이 "정답" 은 아니고, 이미 쓰는 도구에 맞춰 선택하는 문제
```

---

---

# 4단계 — 키를 상수로 관리 (선택사항, 키가 많아지면) ⭐️

```txt
문자열 'DATABASE_URL' 을 여기저기 직접 타이핑하면 오타가 나도 그냥 undefined 로 조용히 실패함
→ 상수 객체로 한 번 감싸면 오타가 TS 컴파일 에러로 바로 드러남
```

```typescript
// config/env.keys.ts
export const EnvKeys = {
  DATABASE_URL: 'DATABASE_URL',
  PORT:         'PORT',
  NODE_ENV:     'NODE_ENV',
} as const;
```

```typescript
configService.get(EnvKeys.NODE_ENV)     // 오타 나면 TS 에러로 바로 발견
configService.get('NODE_ENV')           // 오타 나도 그냥 undefined 반환, 발견 늦음
```

```txt
as const: 객체를 읽기 전용으로 만들고, 값이 string 이 아닌 리터럴 타입으로 추론되게 함
키 이름을 바꿀 때도 env.keys.ts 한 곳만 고치면 전체에 자동 반영됨 (문자열 직접 쓰면 전체 검색/치환 필요)

Joi 스키마에서도 그대로 재사용 가능:
  Joi.object({ [EnvKeys.DATABASE_URL]: Joi.string().uri().required() })
```

---

---

# 어느 단계까지 필요한가 ⭐️⭐️

|상황|권장 단계|
|---|---|
|로컬 스크립트, 토이 프로젝트|1단계 (process.env)|
|일반적인 NestJS 프로젝트 시작|2단계 (ConfigModule, Joi 없이)|
|배포 전 필수 환경변수 누락을 미리 잡고 싶음|3단계 (+ Joi 또는 class-validator)|
|환경변수가 많고 여러 서비스에서 재사용|4단계 (+ env.keys.ts)|

```txt
"처음부터 0~4단계를 다 갖출 필요는 없음" — 프로젝트 초반엔 2단계만으로도 충분히 동작함
```

---

---

# ConfigService 사용 — get vs getOrThrow ⭐️

```typescript
const host  = this.configService.get<string>('POSTGRES_USER');       // 없으면 undefined
const dbUrl = this.configService.getOrThrow<string>('DATABASE_URL'); // 없으면 에러 발생
```

```txt
validationSchema 로 이미 필수값을 검증했다면, 런타임에 get() 으로 undefined 를 받을 일은 거의 없음
그래도 getOrThrow 를 권장하는 이유: "이 값은 반드시 있어야 한다" 는 의도를 코드에서 바로 드러냄
```

|방식|값 없을 때|타입|
|---|---|---|
|`process.env.X`|`undefined`|항상 `string \| undefined`, 직접 변환 필요|
|`configService.get<T>(X)`|`undefined`|제네릭으로 타입 지정, `convert: true` 면 자동 변환|
|`configService.getOrThrow<T>(X)`|에러 발생|위와 동일, 누락을 코드 레벨에서도 명시|

---

---

# 자주 하는 실수

|실수|원인|해결|
|---|---|---|
|다른 모듈에서 `ConfigService` 못 씀|`isGlobal: false`|`isGlobal: true` 설정|
|숫자 값이 string 으로 옴|`.env` 는 항상 string|`validationOptions: { convert: true }`|
|서버 시작은 되는데 런타임 에러|`validationSchema`(또는 `validate`) 없음|3단계 검증 추가, 또는 최소 필수값만 수동 체크|
|키 오타로 `undefined` 반환|문자열을 직접 사용|`EnvKeys` 상수로 교체 (4단계)|
|`.env` 가 Git 에 올라감|`.gitignore` 누락|`.gitignore` 에 `.env`, `.env.local` 추가|
|환경별 값이 안 갈림(개발/운영 혼용)|`.env` 파일 하나만 사용, `envFilePath` 미지정|`envFilePath: ['.env.${NODE_ENV}', '.env']` 로 분리|

---

---

# 같은 .env 를 읽는 여러 주체 — 헷갈리지 않기 ⭐️⭐️⭐️

```txt
NestJS + Prisma 조합에서는 같은 .env 파일 하나를, 실행 컨텍스트마다 완전히 다른 방식으로 읽음
"왜 이 값은 ConfigService 로 검증되는데 저건 안 되지?" 가 헷갈리는 이유가 보통 이 차이를 모를 때임
```

|읽는 주체|언제 실행되나|로딩 방식|Joi 검증 적용?|
|---|---|---|---|
|NestJS 런타임 (main.ts 부팅)|`pnpm start:dev` 등 앱 자체를 실행할 때|`@nestjs/config` 의 `ConfigModule` (내부적으로 dotenv)|✅ `validationSchema` 등록했다면 적용됨|
|Prisma CLI (`migrate`, `generate`)|`prisma migrate dev` 등 CLI 명령 실행 시|`prisma.config.ts` 안의 `dotenv/config` import (Nest 부팅 자체가 없음)|❌ Nest 의 ConfigModule 과 완전히 무관 — Joi 검증 안 거침|
|Docker Compose (`env_file:`)|`docker compose up` 으로 컨테이너 띄울 때|compose 가 직접 그 파일을 읽어 컨테이너 환경변수로 주입|❌ 마찬가지로 무관|

```txt
→ 같은 DATABASE_URL 이라는 키라도, "누가 지금 이걸 읽고 있는가" 에 따라
  검증을 거치는지/거치지 않는지, 심지어 같은 .env 파일을 보는지조차 다를 수 있음

Prisma CLI 가 NestJS 의 ConfigModule/Joi 를 안 타는 이유:
  prisma migrate/generate 는 NestJS 앱을 부팅하지 않고 독립적으로 실행되는 별개의 프로세스
  → ConfigModule 이 등록될 기회 자체가 없음, 그래서 prisma.config.ts 가 dotenv 로 직접 .env 를 읽음
  (자세한 prisma.config.ts 설정은 [[NestJS_Prisma]] 참고)
```

---

---

# 한눈에

```txt
0단계 .env/.env.example 만들기 — 모든 프로젝트의 시작, 프레임워크 무관
1단계 process.env 직접 — 가장 단순하지만, .env 를 실제로 로드해줄 --env-file 플래그가 따로 필요
2단계 ConfigModule(isGlobal:true) — 내부적으로 dotenv 가 로드까지 알아서 함, --env-file 불필요
3단계 Joi(또는 class-validator) 검증 — 선택, 필수값을 배포 전에 잡고 싶을 때
4단계 EnvKeys 상수화 — 선택, 키가 많아지고 오타 방지가 필요할 때

.env.local = 내 컴퓨터 전용 오버라이드, NestJS는 자동 cascading 없음 → envFilePath 직접 지정해야 함
get() 은 조용히 undefined, getOrThrow() 는 즉시 에러 — 의도 표현용으로 getOrThrow 권장

모노레포에서 .env 위치/공유 방식 → [[Monorepo_PNPM]]
Next.js(프론트) 쪽 환경변수는 NEXT_PUBLIC_ 접두사 규칙이 핵심이라 별도 노트 → [[NextJS_Env_Config]]
같은 .env 라도 NestJS 런타임/Prisma CLI/Docker Compose 가 각자 다른 방식으로 읽음 — 위 표 참고
```