---
aliases: [ConfigModule, ConfigService, DataSource, forRoot, forRootAsync, inject, Joi, NestJS DB연결, synchronize, TypeORM]
tags:
  - NestJS
related:
  - "[[00_NestJS_Ecosystem_HomePage]]"
---
# NestJS_TypeORM — TypeORM & DB 연결

## 한 줄 요약

```txt
TypeORM = TypeScript 기반 ORM
Entity 클래스로 DB 테이블을 관리
NestJS + TypeORM + PostgreSQL 연결 패턴
```

---
---
#  TypeORM 특성

```txt
OOP 를 사용해서 DB 테이블을 클래스로 관리하는 ORM

지원 DB:
  MySQL / PostgreSQL / MariaDB / SQLite / Oracle / MongoDB

Active Record + Data Mapper 패턴 모두 지원
Migration 기능 자체 지원 (점진적 DB 구조 변경)
Eager & Lazy 로딩 모두 지원 (데이터 불러오는 방식 완전 제어)
```

>→ Entity 자세한 내용 → [[NestJS_TypeORM_Entity]] 참조 

---

---

# ② 설치

```bash
pnpm i @nestjs/config joi @nestjs/typeorm typeorm pg
#                          ↑ NestJS TypeORM        ↑ PostgreSQL 드라이버
```

---
---
# .env 설정 ⭐️

```properties
ENV=dev

DB_TYPE=postgres
DB_HOST=localhost
DB_PORT=5555
DB_USER=postgres
DB_PASSWORD=postgres
DB_DATABASE=postgres
```

```txt
DataGrip / pgAdmin 연결 시:
  host:     localhost
  port:     .env 의 DB_PORT (5555)
  user:     DB_USER
  password: DB_PASSWORD
  database: DB_DATABASE
```

---

---

# forRoot — 간단하지만 비추 ❌

```typescript
TypeOrmModule.forRoot({
  type:     process.env.DB_TYPE as 'postgres',
  host:     process.env.DB_HOST,
  port:     parseInt(process.env.DB_PORT ?? '5555'),
  username: process.env.DB_USER,
  password: process.env.DB_PASSWORD,
  database: process.env.DB_DATABASE,
  entities: [],
  synchronize: true,
})
```

```txt
문제:
  process.env 직접 사용 → 타입 안전성 없음
  Joi 검증 전에 실행될 수 있음
  ConfigService 의 검증된 값을 쓸 수 없음
  → forRootAsync 사용
```


---

---
# forRootAsync + useFactory ⭐️

```txt
forRootAsync = "나중에 비동기로 설정을 만들어서 줄게"
useFactory   = "설정을 만들어주는 함수"
inject       = "그 함수에 필요한 재료(서비스)"
```


```typescript
TypeOrmModule.forRootAsync({
  useFactory: (configService: ConfigService) => ({
  //            ↑ 이 파라미터는 inject 에서 온다
    type:     configService.get<string>('DB_TYPE') as 'postgres',
    host:     configService.get<string>('DB_HOST'),
    port:     configService.get<number>('DB_PORT'),
    username: configService.get<string>('DB_USER'),
    password: configService.get<string>('DB_PASSWORD'),
    database: configService.get<string>('DB_DATABASE'),
    entities: [],
    synchronize: true,
  }),
  inject: [ConfigService],
  //       ↑ useFactory 파라미터로 전달할 것 명시
})
```

----
---
# useFactory 와 inject 의 관계 ⭐️

```txt
inject 없으면:
  useFactory: (configService) => ...
  configService = undefined → .get() 에러

inject 에 명시하면:
  NestJS IoC 컨테이너가 ConfigService 인스턴스를 꺼내서
  useFactory 의 첫 번째 파라미터로 자동 주입
```

```typescript
// inject 순서 = useFactory 파라미터 순서
inject: [ConfigService, JwtService]
useFactory: (configService, jwtService) => ...
//            ↑ 첫번째       ↑ 두번째
```

```txt
왜 명시적으로 써야 하나:
  forRootAsync 는 모듈 초기화 시점에 실행
  NestJS 가 어떤 서비스가 필요한지 자동으로 모름
  → inject 에 직접 명시해야 주입 가능

  일반 @Injectable() 에서 constructor 주입과 다른 이유가 여기 있음
  constructor 는 NestJS 가 자동으로 주입
  forRootAsync useFactory 는 수동으로 inject 명시 필요
```

---

---

# app.module.ts 전체 설정 ⭐️

```typescript
// 개별 옵션 방식 , Prisma 안사용할때
@Module({
  imports: [
    // 1. ConfigModule 먼저 — Joi 검증 포함
    ConfigModule.forRoot({
      isGlobal: true,
      validationSchema: Joi.object({
        ENV:         Joi.string().valid('dev', 'prod').required(),
        DB_TYPE:     Joi.string().valid('postgres').required(),
        DB_HOST:     Joi.string().required(),
        DB_PORT:     Joi.number().required(),
        DB_USER:     Joi.string().required(),
        DB_PASSWORD: Joi.string().required(),
        DB_DATABASE: Joi.string().required(),
      }),
    }),

    // 2. TypeOrmModule — ConfigService 로 설정
    TypeOrmModule.forRootAsync({
      useFactory: (configService: ConfigService) => ({
        type:     configService.get<string>('DB_TYPE') as 'postgres',
        host:     configService.get<string>('DB_HOST'),
        port:     configService.get<number>('DB_PORT'),
        username: configService.get<string>('DB_USER'),
        password: configService.get<string>('DB_PASSWORD'),
        database: configService.get<string>('DB_DATABASE'),
        autoLoadEntities: true,
        synchronize: configService.get<string>('ENV') !== 'prod',
      }),
      inject: [ConfigService],
    }),
  ],
})
export class AppModule {}
```

```typescript
// url 하나로 연결하는 방식 , Prisma 와 DATABASE_URL 환경변수를 공유할때 
@Module({
  imports: [
    ConfigModule.forRoot({
      isGlobal: true,
      validationSchema: Joi.object({
        ENV:          Joi.string().valid('dev', 'prod').required(),
        DATABASE_URL: Joi.string().required(),
        DB_TYPE:      Joi.string().valid('postgres').required(),
      }),
    }),

    TypeOrmModule.forRootAsync({
      useFactory: (configService: ConfigService) => ({
        // url 하나로 연결하는 방식 ⭐️
        url:  configService.get<string>(envVariableKeys.databaseUrl),
        type: configService.get<string>(envVariableKeys.dbType) as 'postgres',
        // host / port / username / password / database 생략 가능
        autoLoadEntities: true,
        synchronize: configService.get<string>(envVariableKeys.env) !== 'prod',
      }),
      inject: [ConfigService],
    }),
  ],
})
export class AppModule {}
```

```bash
# url 하나로 연결하는 방식이 있음 → Prisma 와 DATABASE_URL 환경변수를 공유할 때 편리 [[NestJS_Prisma]] 참고 
url 하나로 연결하는 방식:
  url = "postgresql://username:password@host:port/database"
  host / port / username / password / database 를 개별로 쓸 필요 없음
  → Prisma 와 DATABASE_URL 환경변수를 공유할 때 편리

  .env:
    DATABASE_URL=postgresql://postgres:password@localhost:5556/mydb

# Prisma 안사용할때 
개별 옵션 방식 (기존):
  host / port / username / password / database 각각 설정
  → DB_HOST / DB_PORT / DB_USER / DB_PASSWORD / DB_DATABASE 전부 필요
```

```txt
순서 중요:
  ConfigModule → TypeOrmModule
  ConfigModule 이 먼저 환경변수를 로드해야
  TypeOrmModule useFactory 에서 ConfigService 로 값을 꺼낼 수 있음
```

>joi에 대해서도 isGlobal, ConfigService는 →[[NestJS_Env_Config]]참조 


---
---
# 주요 옵션 ⭐️

## synchronize

```typescript
synchronize: configService.get<string>('ENV') !== 'prod'
// dev  → true  : Entity 변경 시 DB 자동 반영 (편리)
// prod → false : 자동 변경 금지 (데이터 손실 방지)

synchronize: true,
// Entity 정의가 바뀌면 DB 테이블 자동으로 변경

// ⚠️ 운영에서 true 이면 컬럼 삭제 시 데이터 날아감
// → 운영에서는 반드시 Migration 으로 관리
```

## autoLoadEntities

```typescript
autoLoadEntities: true
// forFeature([Movie]) 에 등록한 Entity 자동 인식
// entities: [Movie, User, ...] 직접 나열할 필요 없음
// → 추가할 때마다 수동 등록 실수 방지
```

## entities

```typescript
// autoLoadEntities 안 쓸 때 직접 나열
entities: [Movie, User, Order]

// 파일 패턴으로 지정
entities: [__dirname + '/**/*.entity{.ts,.js}']
```

---

# forFeature — 모듈에서 Entity 등록

```typescript
// movie.module.ts
@Module({
  imports: [
    TypeOrmModule.forFeature([Movie, MovieDetail])
    //                        ↑ 이 모듈에서 사용할 Entity 등록
  ],
  controllers: [MovieController],
  providers: [MovieService],
})
export class MovieModule {}
```

```txt
forRoot    앱 전체 DB 연결 설정 (app.module.ts 에 한 번)
forFeature 특정 모듈에서 사용할 Entity 등록
           → @InjectRepository(Movie) 사용 가능해짐
```

---
---
# ⚠️ 자주 하는 실수

## Entity metadata not found

```txt
오류: TypeORMError: Entity metadata for Movie#detail was not found
원인: 관계(Relation) 에서 참조하는 Entity 가 등록 안 됨
```

```typescript
// ❌ MovieDetail 등록 누락
entities: [Movie]

// ✅ 관련 Entity 전부 등록
entities: [Movie, MovieDetail]

// 또는 autoLoadEntities: true + forFeature 로 자동 처리
```

## No repository for Entity was found

```txt
오류: No repository for Movie was found
원인: 해당 모듈에 forFeature 없음
해결: TypeOrmModule.forFeature([Movie]) 추가
```

## DB 연결 실패

```txt
오류: Unable to connect to the database
원인: DB 서버 꺼짐 / .env 포트 불일치 / 비밀번호 틀림
확인: docker ps / .env DB_PORT 확인
```

---

---

# 연결 흐름 정리

```txt
.env
  ↓
ConfigModule (Joi 검증)
  ↓ 검증 실패 → 서버 시작 실패
  ↓ 검증 성공
ConfigService (안전하게 값 접근)
  ↓
TypeOrmModule.forRootAsync (useFactory + inject)
  ↓
PostgreSQL 연결
  ↓
Entity 등록 (autoLoadEntities or forFeature)
  ↓
Repository 사용 가능
  ↓
Service → Controller
```










