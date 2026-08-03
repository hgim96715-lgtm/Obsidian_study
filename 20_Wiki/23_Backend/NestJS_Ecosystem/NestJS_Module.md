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
  - "[[NestJS_Prisma]]"
  - "[[NestJS_Service_Provider]]"
  - "[[NestJS_Controller]]"
  - "[[NestJS_WebSocket]]"
---
# NestJS_Module — 모듈

> [!info] 
> 모듈 = 관련된 Controller·Service를 하나로 묶는 상자. 
> AppModule은 그 상자들을 담는 큰 상자. 
> 새 기능을 만들 때 재작성하는 게 아니라, 새 상자(모듈)를 만들고 AppModule에 추가한다.

---

# 모듈이란 — 왜 있는가 ⭐️⭐️⭐️⭐️

```txt
모듈이 없으면:
  모든 Controller, Service가 AppModule 하나에 등록
  기능이 늘수록 한 파일이 수백 줄 → 유지보수 불가

모듈로 기능 단위로 나누면:
  각 기능(DM, 채팅방, 인증 등)이 자신의 모듈 안에 담김
  관련 코드끼리 같은 폴더/파일에 모여 있음
  다른 모듈의 기능이 필요할 때만 명시적으로 선언(imports)
```

```txt
상자 비유:
  DmsModule     ← DM 관련 코드가 담긴 상자
  RoomsModule   ← 채팅방 관련 코드가 담긴 상자
  AuthModule    ← 인증 관련 코드가 담긴 상자

  AppModule     ← 위 상자들을 전부 담는 큰 상자 (앱 전체의 시작점)
```

---

# @Module — 4가지 필드 ⭐️⭐️⭐️⭐️

```typescript
@Module({
  imports:     [],  // 다른 모듈을 가져와서 그 모듈의 서비스를 쓸 수 있게 함
  controllers: [],  // 이 모듈의 컨트롤러 (HTTP 요청을 받는 클래스)
  providers:   [],  // 이 모듈의 서비스 (비즈니스 로직 클래스)
  exports:     [],  // 이 모듈의 서비스를 다른 모듈에서도 쓸 수 있게 공개
})
```

|필드|역할|언제 추가하는가|
|---|---|---|
|`imports`|다른 모듈 가져오기|다른 모듈의 서비스가 필요할 때|
|`controllers`|컨트롤러 등록|이 모듈에서 HTTP 요청을 처리할 때|
|`providers`|서비스 등록|이 모듈에서 비즈니스 로직을 만들 때|
|`exports`|서비스 공개|다른 모듈에서 이 모듈의 서비스를 써야 할 때|

```txt
⚠️ 가장 흔한 실수 — exports 누락:
  DmsModule에서 RoomsService를 쓰고 싶음
  RoomsModule에 imports: [DmsModule]을 추가했는데 에러 발생
  → DmsModule에 exports: [DmsService]가 없으면
    "Nest can't resolve dependencies" 에러

  공식: 서비스를 외부에서 쓰려면 반드시 그 모듈의 exports에 추가
```

---

# AppModule — 루트 모듈 ⭐️⭐️⭐️⭐️

```typescript
// app.module.ts
@Module({
  imports: [
    ConfigModule.forRoot({ isGlobal: true }),
    PrismaModule,
    AuthModule,
    DmsModule,
    RoomsModule,
  ],
})
export class AppModule {}
```

```txt
AppModule의 역할:
  앱 전체의 시작점 — NestJS가 AppModule을 기반으로 앱을 시작
  기능 모듈들을 imports에 모아서 조립하는 곳

AppModule에서 하지 않는 것:
  비즈니스 로직 → 기능 모듈(RoomsModule, DmsModule 등)에
  Controller, Service 직접 등록 → 기능 모듈에

  AppModule = 큰 상자, 기능 모듈 = 작은 상자
  큰 상자는 작은 상자들을 담기만 함

재작성하는 게 아님:
  새 기능을 만들 때 AppModule을 재작성하지 않음
  새 기능 모듈을 만들고, AppModule의 imports에 한 줄 추가
```

---

# 새 기능 만들 때 워크플로우 ⭐️⭐️⭐️⭐️

```txt
예시: DM(다이렉트 메시지) 기능을 새로 만드는 경우
```

## 1단계 — 기능 모듈 생성

```bash
nest g resource dms
# → dms/dms.module.ts
# → dms/dms.controller.ts
# → dms/dms.service.ts
# → dms/dto/ 폴더
```

```txt
nest g resource 가 자동으로:
  모듈 / 컨트롤러 / 서비스 파일 생성
  AppModule의 imports에 DmsModule 자동 추가 ← 이 단계가 자동으로 됨
```

## 2단계 — 기본 모듈 (이미 생성됨)

```typescript
// dms/dms.module.ts
@Module({
  controllers: [DmsController],
  providers:   [DmsService],
})
export class DmsModule {}
```

## 3단계 — 다른 모듈의 서비스가 필요할 때

```typescript
// DmsService 안에서 PrismaService를 쓰고 싶다면
// PrismaModule이 @Global()이면 → 아무것도 안 해도 됨
// PrismaModule이 일반 모듈이면 → imports에 추가

@Module({
  imports:     [PrismaModule],  // PrismaService를 쓰려면 PrismaModule을 가져와야 함
  controllers: [DmsController],
  providers:   [DmsService],
})
export class DmsModule {}
```

## 4단계 — DmsService를 다른 모듈에서 써야 할 때

```typescript
@Module({
  imports:     [PrismaModule],
  controllers: [DmsController],
  providers:   [DmsService],
  exports:     [DmsService],  // 이 줄 추가 → 다른 모듈에서 DmsService 주입 가능
})
export class DmsModule {}
```

---

# imports/exports 의사결정 ⭐️⭐️⭐️⭐️

```txt
질문 1: 이 모듈에서 다른 모듈의 서비스를 써야 하는가?
  → 예: imports에 그 모듈 추가
  → 아니오: 그냥 둠

질문 2: 이 모듈의 서비스를 다른 모듈이 써야 하는가?
  → 예: exports에 그 서비스 추가
  → 아니오: 그냥 둠 (이 모듈 안에서만 사용)
```

```typescript
// 시나리오: DmsService가 RoomsService를 필요로 함
//           RoomsService가 DmsService를 필요로 함

// RoomsModule — DmsService를 쓸 수 있도록 exports
@Module({
  providers: [RoomsService],
  exports:   [RoomsService],  // ← 공개
})
export class RoomsModule {}

// DmsModule — RoomsModule을 가져와서 RoomsService 사용
@Module({
  imports:     [RoomsModule],  // ← 가져오기
  providers:   [DmsService],
  exports:     [DmsService],
})
export class DmsModule {}

// DmsService 안에서 주입받기
@Injectable()
export class DmsService {
  constructor(private readonly roomsService: RoomsService) {}
}
```

---

# @Global() — 모든 모듈에서 쓰는 서비스 ⭐️⭐️⭐️

```typescript
@Global()
@Module({
  providers: [PrismaService],
  exports:   [PrismaService],
})
export class PrismaModule {}
```

```txt
@Global()이 없으면:
  PrismaService를 쓰는 모든 모듈마다 imports: [PrismaModule] 추가 필요
  → DmsModule, RoomsModule, AuthModule... 전부

@Global()이 있으면:
  AppModule에 PrismaModule 한 번만 import
  이후 어떤 모듈이든 imports 없이 PrismaService 주입 가능

⚠️ @Global()이어도 AppModule에는 한 번 import 해야 함
   "import가 전혀 필요 없다"가 아니라 "AppModule에 한 번만 하면 됨"

언제 @Global()을 쓰는가:
  거의 모든 모듈이 필요로 하는 서비스 → PrismaService, ConfigService, JwtService
  일부 모듈만 쓴다면 → 명시적 imports가 더 명확
```

```txt
ConfigModule.forRoot({ isGlobal: true })의 isGlobal:
  내부적으로 @Global()과 같은 효과
  라이브러리들이 isGlobal 옵션으로 @Global()을 켜고 끌 수 있게 미리 만들어둔 것
```

---

# forwardRef — 순환 참조 해결 ⭐️⭐️⭐️⭐️

```txt
순환 참조(Circular Dependency):
  DmsModule   → RoomsModule을 import
  RoomsModule → DmsModule을 import
  → 서로가 서로를 기다리는 상황
  → NestJS가 어느 것도 먼저 초기화하지 못함
  → 에러: "A circular dependency between modules has been detected."
```

```typescript
// DmsModule
@Module({
  imports: [AuthModule, forwardRef(() => RoomsModule)],
  controllers: [DmsController],
  providers:   [DmsService],
  exports:     [DmsService],
})
export class DmsModule {}

// RoomsModule — 반대편도 forwardRef 필요
@Module({
  imports:   [forwardRef(() => DmsModule)],
  providers: [RoomsService, RoomsGateway],
  exports:   [RoomsService],
})
export class RoomsModule {}
```

```txt
forwardRef(() => RoomsModule) 동작:
  일반 import는 즉시 참조 → 초기화 순서 결정 → 순환이면 막힘
  forwardRef는 함수로 감싸서 참조를 나중으로 미룸 (지연 참조)
  NestJS가 두 모듈을 각각 먼저 만들고, 이후에 연결

⚠️ 양쪽 모두 forwardRef 필요 — 한쪽만 하면 여전히 에러
```

## 더 좋은 해결책 — 공통 모듈 분리 ⭐️⭐️⭐️⭐️

```txt
forwardRef는 임시방편 — 순환 참조 자체가 설계 문제의 신호

순환 참조가 생기는 이유:
  두 모듈이 서로의 기능을 필요로 함
  → 그 공통 기능을 제3의 모듈로 분리하면 해결

패턴:
  ❌ A ↔ B (순환)
  ✅ A → Shared ← B (공통 분리)

  A와 B는 서로를 몰라도 됨
  둘 다 Shared만 import
  Shared는 누구도 import 안 함 → 순환 없음
```

```typescript
// ❌ 순환 참조 — forwardRef 필요
@Module({ imports: [forwardRef(() => FeatureBModule)], ... })
export class FeatureAModule {}

@Module({ imports: [forwardRef(() => FeatureAModule)], ... })
export class FeatureBModule {}

// ✅ 공통 분리 — forwardRef 없음
@Module({
  providers: [SharedGateway],
  exports:   [SharedGateway],  // 기능 모듈들이 주입해서 사용
})
export class SharedModule {}

@Module({ imports: [SharedModule], ... })  // Shared만 import
export class FeatureAModule {}

@Module({ imports: [SharedModule], ... })  // Shared만 import
export class FeatureBModule {}
```

```txt
언제 어떤 것을 Shared로 빼는가:
  두 모듈이 서로를 필요로 하게 된 "공통 기능"이 무엇인지 찾기
  그 기능만 별도 모듈로 분리

WebSocket 예시:
  ModuleA와 ModuleB 둘 다 실시간 emit이 필요
  → SharedGateway(연결·인증·emit)를 SharedModule로 분리
  → 기능별 이벤트(@SubscribeMessage)는 각 모듈에 유지
  → [[NestJS_WebSocket]] Gateway 책임 분리 패턴 참고
```

---

# Dynamic Module — forRoot / forRootAsync ⭐️⭐️⭐️

```txt
ConfigModule.forRoot({...}), TypeOrmModule.forRootAsync({...}) 같은 패턴
→ 옵션을 전달해서 모듈을 설정하는 방법
```

|이름|언제|
|---|---|
|`forRoot(options)`|고정된 옵션을 직접 전달 (동기)|
|`forRootAsync(options)`|ConfigService 등 다른 Provider에서 값을 받아 옵션 계산 (비동기)|

```typescript
// forRoot — 값을 직접 넘김
JwtModule.register({
  secret: 'my-secret',
  signOptions: { expiresIn: '15m' },
})

// forRootAsync — ConfigService에서 값을 받아서 옵션 만들기
JwtModule.registerAsync({
  imports:    [ConfigModule],
  useFactory: (configService: ConfigService) => ({
    secret:      configService.getOrThrow('JWT_SECRET'),
    signOptions: { expiresIn: '15m' },
  }),
  inject: [ConfigService],
})
```

```txt
forRootAsync를 쓰는 이유:
  환경변수 값은 런타임에 ConfigService를 통해 읽어야 함
  forRoot에 process.env.JWT_SECRET을 직접 넣으면 undefined일 수 있음
  → forRootAsync + inject: [ConfigService]로 안전하게 읽기
```

---

# 폴더 구조 ⭐️⭐️⭐️

```txt
src/
├── app.module.ts              루트 모듈 — 기능 모듈 조립만
├── prisma/
│   ├── prisma.module.ts       @Global() — 전역 DB 연결
│   └── prisma.service.ts
├── auth/
│   ├── auth.module.ts
│   ├── auth.controller.ts
│   └── auth.service.ts
├── dms/
│   ├── dms.module.ts
│   ├── dms.controller.ts
│   ├── dms.service.ts
│   └── dto/
└── rooms/
    ├── rooms.module.ts
    ├── rooms.controller.ts
    ├── rooms.service.ts
    ├── rooms.gateway.ts       WebSocket
    └── dto/
```

```txt
폴더 안의 파일들:
  module.ts     모듈 선언 (imports/exports/controllers/providers)
  controller.ts HTTP 요청 받기 (@Get, @Post 등)
  service.ts    비즈니스 로직 (DB 쿼리, 계산 등)
  gateway.ts    WebSocket 이벤트 처리 (선택)
  dto/          요청 Body 타입 정의 + 유효성 검사

기능 하나 = 폴더 하나 = 모듈 하나
```

---

# 자주 만나는 에러

| 에러                                | 원인                                           | 해결                            |
| --------------------------------- | -------------------------------------------- | ----------------------------- |
| `Nest can't resolve dependencies` | 서비스를 주입받으려는데 그 모듈이 imports에 없음 또는 exports 누락 | 해당 모듈 imports 추가 + exports 확인 |
| `Circular dependency detected`    | 두 모듈이 서로를 import                             | forwardRef 사용 또는 구조 재설계       |
| `Cannot find module`              | 모듈 파일 경로 오류                                  | import 경로 확인                  |
| 서비스 주입이 undefined                 | providers에 등록 안 됨                            | @Module providers에 서비스 추가     |