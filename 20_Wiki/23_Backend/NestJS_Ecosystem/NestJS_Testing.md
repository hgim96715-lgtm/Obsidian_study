---
aliases: [테스트, Coverage, E2E Test, Jest, PostgreSQL 통합 테스트 환경 세팅, Testing, Unit Test]
tags:
  - NestJS
related:
  - "[[00_NestJS_Ecosystem_HomePage]]"
---

# NestJS_Testing — 테스팅

```txt
테스팅 = 코드가 예상대로 동작하는지 자동으로 검증하는 과정
흐름: 가짜 의존성 준비 → 테스트 실행 → 결과·예외·호출 검증
```



---

---

# 왜 테스팅인가 ⭐️

```txt
테스트 없으면:
  API 직접 호출해서 눈으로 확인 → 오래 걸리고 일관성 없음
  코드 변경 시 기존 기능 망가졌는지 알기 어려움

테스트 있으면:
  pnpm test 한 번으로 모든 로직 자동 검증
  리팩터링 후 기존 기능 즉시 확인
  테스트 코드 자체가 동작 기준을 설명하는 문서 역할
```

---

---

# 테스팅 종류 ⭐️

|종류|대상|특징|
|---|---|---|
|`Unit Test`|Service / 함수 단위|DB·외부 API 는 Mock 대체 / 가장 빠름|
|`Integration Test`|여러 기능 + 실제 DB 연결|Repository·relation·transaction 확인|
|`E2E Test`|HTTP 요청 → 응답 전체 흐름|Controller·Pipe·Guard·Service 연결 확인 / supertest 사용|

```txt
순서: Unit → Integration → E2E
먼저 작성해야 하는 것: Unit Test
```

## 파일 네이밍 & 실행 스크립트

```txt
Unit Test        {파일명}.spec.ts              pnpm test
Integration Test {파일명}.integration.spec.ts  pnpm test:integration
E2E Test         {파일명}.e2e-spec.ts          pnpm test:e2e
```

```json
// package.json — 스크립트 설정
{
  "scripts": {
    "test":             "jest",
    "test:integration": "jest --testRegex='.*\\.integration\\.spec\\.ts$'",
    "test:e2e":         "jest --config ./test/jest-e2e.json"
  }
}
```

```txt
⚠️ test:integration 은 *.integration.spec.ts 파일이 없으면
   No tests found → exit code 1 에러 발생

   파일 없이 실행만 하면 안 됨
   통합 테스트를 쓸 예정이면
   src/movie/movie.integration.spec.ts 같은 파일을 먼저 만들어야 함

   아직 파일이 없는데 스크립트만 있으면:
   → --passWithNoTests 옵션 추가하거나
   → 파일 만들기 전까지 스크립트 제거

   → [[NestJS_Testing_Troubleshooting]] 참고
```

---

---

# Jest 기초

## 테스트 파일 위치 & 네이밍 ⭐️

```txt
파일명 규칙:
  {대상파일명}.spec.ts

  user.service.ts    → user.service.spec.ts
  user.controller.ts → user.controller.spec.ts
  app.module.ts      → app.e2e-spec.ts  (E2E)

위치:
  Unit Test  → 테스트 대상 파일과 같은 폴더
  E2E Test   → 프로젝트 루트 test/ 폴더
```

## 기본 구조

```typescript
describe('MovieService', () => {
//  ↑ 관련 테스트 묶음 (그룹명)

  let service: MovieService;

  beforeEach(async () => {
    // 각 테스트 실행 전 초기화
  });

  it('영화 목록을 반환한다', async () => {
  //↑ 개별 테스트 케이스 (test() 와 동일)
    const result = await service.findAll();
    expect(result).toBeDefined();
  });
});
```

|함수|역할|
|---|---|
|`describe()`|관련 테스트 묶음|
|`it()` / `test()`|개별 테스트 케이스|
|`beforeEach()`|각 테스트 전 실행 — Mock 초기화|
|`afterEach()`|각 테스트 후 실행|
|`beforeAll()`|전체 테스트 전 1번 — E2E 앱 생성|
|`afterAll()`|전체 테스트 후 1번 — E2E 앱 정리|

## Matcher — 검증 함수 ⭐️

```typescript
// 값 / 객체 검증
expect(service).toBeDefined()          // undefined 가 아닌지
expect(result).toBeNull()              // null 인지
expect(result).toBe(42)                // 원시값 정확히 같은지 (===)
expect(result).toEqual({ id: 1 })      // 객체 내부 값까지 같은지 (깊은 비교)
expect(result).toHaveLength(3)         // 배열 / 문자열 길이
expect(result).toContain('hello')      // 포함 여부
```

```txt
toBe vs toEqual:
  toBe      숫자 / 문자열 / boolean 원시값 비교
  toEqual   객체 / 배열 내부 값까지 재귀적으로 비교
  → 객체·배열 결과는 toEqual 사용
```

```typescript
// 일부 값만 검증 — createdAt 같은 가변값 무시할 때 ⭐️
expect(result).toEqual(
  expect.objectContaining({ id: 1, email: 'test@test.com' })
);

// 배열 안에 특정 항목이 포함됐는지
expect(result).toEqual(
  expect.arrayContaining([
    expect.objectContaining({ email: 'test@test.com' })
  ])
);

// Mock 함수 호출 검증
expect(mockFn).toHaveBeenCalled()                       // 한 번 이상 호출됐는지
expect(mockFn).toHaveBeenCalledTimes(1)                 // 정확히 N번 호출됐는지
expect(mockFn).toHaveBeenCalledWith({ id: 1 })          // 어떤 인자로 호출됐는지
expect(mockFn).not.toHaveBeenCalled()                   // 호출되지 않았는지
expect(mockFn).toHaveBeenNthCalledWith(1, { id: 1 })    // N번째 호출 인자 검증 ⭐️

// 비동기 예외 검증
await expect(service.findOne(999)).rejects.toThrow(NotFoundException);
expect(() => fn()).toThrow();  // 동기 즉시 throw
```

```txt
toHaveBeenCalledWith()         호출된 적 있는지 (순서 무관)
toHaveBeenNthCalledWith(N, ..) N번째 호출의 인자 정확히 검증
                               N 은 1부터 시작
```

## expect 헬퍼

```typescript
expect.anything()              // null/undefined 제외한 모든 값
expect.any(String)             // 특정 타입으로 검증
expect.stringContaining('abc') // 특정 문자열 포함 여부
```

```txt
언제 쓰나:
  내부 상수값이라 정확한 값보다 "호출은 됐냐" 가 중요할 때
  expect(mockFn).toHaveBeenCalledWith(expect.anything())
```

---

---

## ④ Mock 개념

|구분|의미|예시|
|---|---|---|
|`Stub`|정해진 값을 반환|`findOne()` 이 영화 객체 반환|
|`Mock`|반환값 + 호출 여부·인자 검증|`save()` 가 1번 호출됐는지 확인|
|`Fake`|단순화한 실제 구현|메모리 배열 Repository|

```txt
NestJS Unit Test 에서는 보통 jest.fn() 으로
Repository / Logger / JwtService 등의 동작을 대체한다
```

---

---

# Mock 함수 ⭐️

## jest.fn() — 처음부터 가짜 함수 생성

```typescript
const mockFn = jest.fn();

mockFn.mockReturnValue({ id: 1 })               // 동기 반환값
mockFn.mockResolvedValue({ id: 1 })             // async 성공 결과
mockFn.mockRejectedValue(new Error('DB 오류'))   // async 실패 결과
```

|함수|역할|
|---|---|
|`jest.fn()`|가짜 함수 생성|
|`mockReturnValue()`|동기 반환값 설정|
|`mockResolvedValue()`|async 성공 결과 설정|
|`mockRejectedValue()`|async 실패 상황 설정|
|`mockImplementation()`|인자에 따라 다른 값 반환하는 로직 설정|
|`jest.clearAllMocks()`|이전 테스트 호출 기록 초기화|

---

## mockImplementation() — 인자에 따라 다른 값 반환 ⭐️

```txt
mockReturnValue / mockResolvedValue 는 항상 같은 값을 반환
mockImplementation 은 실제 함수처럼 인자에 따라 다르게 동작

언제 쓰나:
  같은 함수가 호출될 때마다 인자값에 따라 다른 결과를 줘야 할 때
  ex) configService.getOrThrow('refreshTokenSecret') → 'refresh-secret'
      configService.getOrThrow('accessTokenSecret')  → 'access-secret'
```

```typescript
// beforeEach 안에서 설정 — describe 안 테스트 전체에 적용
beforeEach(() => {
  mockConfigService.getOrThrow.mockImplementation((key: string) => {
    if (key === envVariableKeys.refreshTokenSecret) return 'refresh-secret';
    if (key === envVariableKeys.accessTokenSecret)  return 'access-secret';
    throw new Error(`unknown key: ${key}`);
    //              ↑ 예상치 못한 key 는 에러 → 테스트 실수를 바로 잡아줌
  });
});

it('should issue a refresh token', async () => {
  (jwtService.signAsync as jest.Mock).mockResolvedValue('refresh-token');

  const result = await authService.issueToken(user, true);

  expect(result).toEqual('refresh-token');
  expect(jwtService.signAsync).toHaveBeenCalledWith(
    { sub: user.id, role: user.role, type: 'refresh' },
    { secret: 'refresh-secret', expiresIn: '24h' },
  );
});
```

```txt
mockReturnValue vs mockImplementation:
  mockReturnValue('x')         호출할 때마다 항상 'x' 반환
  mockImplementation((k) => …) 인자 k 에 따라 분기해서 반환

async 버전:
  mockImplementation(async (key) => { … })
  또는 mockImplementationOnce() — 딱 한 번만 적용
```

```typescript
// mockImplementationOnce — 첫 번째 호출만 다르게
mockFn
  .mockImplementationOnce(() => 'first')
  .mockImplementationOnce(() => 'second')
  .mockReturnValue('default');   // 이후 호출은 전부 'default'
```

---

## jest.spyOn() — 이미 존재하는 메서드 감시 ⭐️

```txt
⚠️ spyOn 은 이미 객체에 존재하는 프로퍼티만 감쌀 수 있다

즉, Mock 객체를 만들 때 감시할 메서드를 반드시 먼저 정의해야 한다
```

```typescript
// ❌ 잘못된 예 — mockJwtService 에 decode 가 없으면 spyOn 실패
const mockJwtService = {
  signAsync:   jest.fn(),
  verifyAsync: jest.fn(),
  // decode 없음
};

jest.spyOn(jwtService, 'decode')  // ❌ Property 'decode' does not exist

// ✅ 올바른 예 — 사용할 메서드를 전부 미리 정의
const mockJwtService = {
  signAsync:   jest.fn(),
  verifyAsync: jest.fn(),
  decode:      jest.fn(),   // ← 반드시 먼저 선언
};

// 이후 두 가지 방식으로 사용 가능
// 방법 1) jest.spyOn — 감시 + 반환값 설정
jest.spyOn(jwtService, 'decode').mockReturnValue(payload);

// 방법 2) as jest.Mock 캐스팅 — TypeScript 타입 때문에 필요
(mockJwtService.decode as jest.Mock).mockReturnValue(payload);
```

```txt
왜 캐스팅이 필요한가:
  jest.mock() / jest.fn() 으로 교체했지만
  TypeScript 는 여전히 원래 타입으로 인식
  → jest.Mock 으로 캐스팅해야 mockReturnValue 사용 가능

spyOn vs fn() 정리:
  jest.fn()    처음부터 가짜 함수로 생성  → Repository Mock 만들 때
  jest.spyOn() 기존 객체 메서드 감시      → 실제 Service 메서드 확인 시

spy.mockRestore()  → spyOn 으로 변경한 메서드를 원래 구현으로 복구
```

---

## jest.mock() — 외부 모듈 전체 교체 ⭐️

```typescript
// 파일 최상단 (describe 밖)
jest.mock('bcrypt', () => ({
  hash:    jest.fn(),
  compare: jest.fn(),
}));
```

```typescript
// 이후 테스트 안에서 캐스팅 후 사용
import * as bcrypt from 'bcrypt';

(bcrypt.hash as jest.Mock).mockResolvedValue('hashedPassword');
```

```txt
jest.fn()    특정 함수 하나를 가짜로
jest.mock()  외부 라이브러리 모듈 전체를 가짜로
→ bcrypt / fs / axios 같은 외부 모듈에 사용
테스트 파일 전체에 적용됨
```

---

---

# Service 테스트 ⭐️

```txt
Service 테스트 구조:
  Mock 대상  → Repository (실제 DB 대신)
  주입 방식  → providers 에 getRepositoryToken 으로 등록
  검증 방식  → mockRepository 메서드 호출 여부 / 반환값

흐름:
  Mock Repository 정의 → 테스트 모듈에 주입 → Service 인스턴스 가져오기
  → Mock 반환값 설정 → Service 메서드 실행 → 결과 / 호출 검증
```

## 기본 세팅

```typescript
// director.service.spec.ts

// ① Mock Repository 정의 — describe 밖 / 사용할 메서드 전부 선언
const mockDirectorRepository = {
  find:    jest.fn(),
  findOne: jest.fn(),
  save:    jest.fn(),
  delete:  jest.fn(),
};

describe('DirectorService', () => {
  let service: DirectorService;

  beforeEach(async () => {
    // ② 테스트 모듈 — providers 에 Repository Mock 주입
    const module: TestingModule = await Test.createTestingModule({
      providers: [
        DirectorService,
        {
          provide:  getRepositoryToken(Director),  // @InjectRepository(Director) 토큰
          useValue: mockDirectorRepository,        // 실제 DB 대신 Mock 주입
        },
      ],
    }).compile();

    // ③ Service 인스턴스 가져오기
    service = module.get<DirectorService>(DirectorService);
    jest.clearAllMocks();
  });

  it('service 가 정의되어 있다', () => {
    expect(service).toBeDefined();
  });
});
```

## 의존성이 많을 때

```typescript
// auth.service.spec.ts

// ① Mock 객체 정의 — 사용할 메서드 전부 미리 선언 ⭐️
const mockUserRepository = { findOne: jest.fn(), save: jest.fn() };
const mockConfigService  = { getOrThrow: jest.fn() };
const mockUserService    = { create: jest.fn(), findOne: jest.fn() };
const mockJwtService     = { signAsync: jest.fn(), verifyAsync: jest.fn(), decode: jest.fn() };
const mockCacheManager   = { get: jest.fn(), set: jest.fn(), del: jest.fn() };

describe('AuthService', () => {
  let authService:  AuthService;
  let jwtService:   JwtService;
  let cacheManager: Cache;

  beforeEach(async () => {
    // ② constructor 의존성 전부 providers 에 등록
    const module = await Test.createTestingModule({
      providers: [
        AuthService,
        { provide: getRepositoryToken(User), useValue: mockUserRepository },
        { provide: ConfigService,            useValue: mockConfigService  },
        { provide: UserService,              useValue: mockUserService    },
        { provide: JwtService,               useValue: mockJwtService     },
        { provide: CACHE_MANAGER,            useValue: mockCacheManager   },
      ],
    }).compile();

    // ③ 인스턴스 가져오기
    authService  = module.get<AuthService>(AuthService);
    jwtService   = module.get<JwtService>(JwtService);
    cacheManager = module.get<Cache>(CACHE_MANAGER);

    jest.clearAllMocks();
  });
});
```

```txt
특수 토큰:
  @InjectRepository(Entity) → getRepositoryToken(Entity)
  @Inject(CACHE_MANAGER)    → CACHE_MANAGER 토큰 그대로 사용

⚠️ 의존성 추가될 때마다 Mock 도 함께 providers 에 등록
   오류 메시지의 index [N] 번호로 어느 의존성인지 확인
   → [[NestJS_Testing_Troubleshooting]] 참고
```

## Service 테스트 — 검증 패턴

```typescript
it('전체 감독 목록을 반환한다', async () => {
  // Arrange — Mock Repository 반환값 설정
  const directors = [{ id: 1, name: '봉준호' }];
  mockDirectorRepository.find.mockResolvedValue(directors);

  // Act — Service 메서드 실행
  const result = await service.findAll();

  // Assert — 결과 + Repository 호출 검증
  expect(result).toEqual(directors);
  expect(mockDirectorRepository.find).toHaveBeenCalledTimes(1);
});
```

## QueryBuilder 체이닝 Mock — mockReturnThis() ⭐️

```txt
TypeORM QueryBuilder 는 메서드 체이닝 방식으로 동작
  createQueryBuilder().distinct(true).leftJoinAndSelect(...).where(...).getManyAndCount()

jest.fn() 은 기본 undefined 반환
→ .distinct(true) 결과가 undefined → undefined.leftJoinAndSelect → TypeError

해결:
  체이닝 중간 메서드에 mockReturnThis() — 자기 자신(qb) 을 반환해 체이닝 유지
  마지막 메서드(getManyAndCount) 에 mockResolvedValue — Promise 반환
```

```typescript
it('should return all movies', async () => {
  const movies = [{ id: 1, title: 'test' }, { id: 2, title: 'test2' }];

  // qbMock 을 describe 안에서 직접 선언
  const qbMock = {
    distinct:           jest.fn().mockReturnThis(),
    leftJoinAndSelect:  jest.fn().mockReturnThis(),
    where:              jest.fn().mockReturnThis(),
    getManyAndCount:    jest.fn().mockResolvedValue([movies, 2]),
  };

  // spyOn 으로 createQueryBuilder 가 qbMock 을 반환하도록 설정
  jest.spyOn(movieRepository, 'createQueryBuilder').mockReturnValue(
    qbMock as unknown as SelectQueryBuilder<Movie>,
  );

  // 다른 의존성도 spyOn 으로 처리
  jest.spyOn(commonService, 'applyCursorPaginationParamsToQb').mockResolvedValue({
    nextCursor: null,
  } as Awaited<ReturnType<CommonService['applyCursorPaginationParamsToQb']>>);

  const result = await service.findAll(dto, 1);

  // 체이닝 각 단계 검증
  expect(movieRepository.createQueryBuilder).toHaveBeenCalledWith('movie');
  expect(qbMock.distinct).toHaveBeenCalledWith(true);
  expect(qbMock.leftJoinAndSelect).toHaveBeenCalledTimes(3);
  expect(qbMock.where).toHaveBeenCalledWith('movie.title LIKE :title', { title: '%test%' });
  expect(commonService.applyCursorPaginationParamsToQb).toHaveBeenCalledWith(qbMock, dto);
});
```

```txt
mockReturnThis()    체이닝 중간 메서드 → qb 자신 반환 → 다음 메서드 이어짐
mockResolvedValue() 체이닝 끝 메서드  → Promise 반환
mockReturnValue()   createQueryBuilder → qbMock 객체 반환

as unknown as SelectQueryBuilder<Movie>:
  qbMock 은 실제 TypeORM 타입과 다르기 때문에 이중 캐스팅 필요
  → unknown 으로 먼저 풀고 → 원하는 타입으로 캐스팅

수동 방식(const mockMovieRepository = {...}) 과 차이:
  describe 밖 Mock 객체 → spyOn 없이 바로 사용
  describe 안 qbMock   → spyOn 으로 원하는 테스트에만 적용 가능
                          테스트마다 다른 getManyAndCount 반환값 설정 용이
```

---

---

# Controller 테스트 ⭐️

```txt
Controller 테스트 구조:
  Mock 대상  → Service (Repository 까지 내려가지 않음)
  주입 방식  → providers 에 Service Mock 등록 + controllers 에 Controller 등록
  검증 방식  → jest.spyOn(service, '메서드') 으로 감시
               controller 메서드 호출 → 결과 검증

Service 테스트와 차이:
  Service 테스트    Repository 를 Mock → Service 로직 검증
  Controller 테스트 Service 를 Mock   → Controller 가 Service 를 올바르게 호출하는지 검증
```

## 기본 세팅

```typescript
// director.controller.spec.ts

describe('DirectorController', () => {
  let controller: DirectorController;
  let service:    DirectorService;    // spyOn 을 위해 Service 인스턴스도 가져옴 ⭐️

  beforeEach(async () => {
    const module: TestingModule = await Test.createTestingModule({
      // ✅ Controller 테스트 — controllers + providers(Service Mock)
      controllers: [DirectorController],
      providers: [
        {
          provide:  DirectorService,
          useValue: {                      // Service 메서드를 전부 jest.fn() 으로
            findAll: jest.fn(),
            findOne: jest.fn(),
            create:  jest.fn(),
            update:  jest.fn(),
            remove:  jest.fn(),
          },
        },
      ],
    }).compile();

    // Controller 와 Service 인스턴스 둘 다 가져오기
    controller = module.get<DirectorController>(DirectorController);
    service    = module.get<DirectorService>(DirectorService);
    //           ↑ spyOn 으로 감시하려면 실제 모듈에서 가져와야 함

    jest.clearAllMocks();
  });

  it('controller 가 정의되어 있다', () => {
    expect(controller).toBeDefined();
  });
});
```

## Controller 테스트 — spyOn 검증 패턴 ⭐️

```typescript
it('findAll 은 전체 감독 목록을 반환한다', async () => {
  // Arrange — Service 메서드에 spyOn 으로 반환값 설정
  const directors = [{ id: 1, name: '봉준호' }];

  jest.spyOn(service, 'findAll').mockResolvedValue(directors as Director[]);
  //          ↑ module.get 으로 가져온 service 인스턴스에 spyOn
  //                   ↑ 감시할 메서드 이름

  // Act — Controller 메서드 실행
  const result = await controller.findAll();

  // Assert — 결과 + Service 호출 검증
  expect(result).toEqual(directors);
  expect(service.findAll).toHaveBeenCalledTimes(1);
});

it('findOne 은 특정 감독을 반환한다', async () => {
  const director = { id: 1, name: '봉준호' };

  jest.spyOn(service, 'findOne').mockResolvedValue(director as Director);

  const result = await controller.findOne(1);

  expect(result).toEqual(director);
  expect(service.findOne).toHaveBeenCalledWith(1);
});

it('create 는 새 감독을 생성한다', async () => {
  const createDto = { name: '박찬욱', nationality: '한국' };
  const created   = { id: 2, ...createDto };

  jest.spyOn(service, 'create').mockResolvedValue(created as Director);

  const result = await controller.create(createDto as CreateDirectorDto);

  expect(result).toEqual(created);
  expect(service.create).toHaveBeenCalledWith(createDto);
});
```

## Service vs Controller 테스트 한눈에 비교

```txt
                     Service 테스트          Controller 테스트
─────────────────────────────────────────────────────────────────
Mock 대상           Repository              Service
모듈 설정           providers: [            controllers: [Controller]
                     Service,               providers: [{
                     { provide:               provide: Service,
                       getRepositoryToken,    useValue: { findAll: jest.fn(), ... }
                       useValue: mockRepo }  }]
                    ]

인스턴스 가져오기   service =               controller =
                    module.get(Service)     module.get(Controller)
                                           service =
                                           module.get(Service)  ← spyOn 용

반환값 설정        mockRepo.find            jest.spyOn(service, 'findAll')
                   .mockResolvedValue(...)  .mockResolvedValue(...)

실행               service.findAll()        controller.findAll()

검증               expect(mockRepo.find)    expect(service.findAll)
                   .toHaveBeenCalled()      .toHaveBeenCalled()
─────────────────────────────────────────────────────────────────
```

```txt
왜 Controller 테스트에서 spyOn 을 쓰나:
  useValue 로 Service 전체를 jest.fn() 으로 교체했지만
  module.get(Service) 로 가져온 인스턴스는 그 Mock 객체
  → spyOn 으로 감시하면 호출 여부 / 인자 / 반환값 전부 제어 가능

⚠️ spyOn 은 이미 존재하는 메서드만 감시 가능
   → useValue 에 정의한 메서드만 spyOn 대상 가능
   → findAll, findOne 등 사용할 메서드 전부 jest.fn() 으로 미리 선언 필수
```

## TransactionInterceptor 가 있는 Controller 테스트 ⭐️

```txt
Controller 에 TransactionInterceptor 가 적용돼 있으면
테스트 모듈에도 Interceptor 가 실행되어 실제 QueryRunner 를 만들려다 에러 발생

해결:
  .overrideInterceptor() 로 TransactionInterceptor 를 가짜로 교체
  → intercept 가 그냥 next.handle() 만 호출하도록 bypass

Controller 단위 테스트에서는:
  트랜잭션 / req.queryRunner 주입은 검증하지 않음
  Service 에 인자가 맞게 넘어가는지만 검증
  queryRunner 는 { manager: {} } as QueryRunner 처럼 직접 넘김
```

```typescript
// movie.controller.spec.ts

const mockMovieService = {
  findAll:  jest.fn(),
  findOne:  jest.fn(),
  create:   jest.fn(),
  update:   jest.fn(),
  remove:   jest.fn(),
};

describe('MovieController', () => {
  let controller: MovieController;
  let service:    MovieService;

  beforeEach(async () => {
    jest.clearAllMocks();

    const module: TestingModule = await Test.createTestingModule({
      controllers: [MovieController],
      providers: [
        {
          provide:  MovieService,
          useValue: mockMovieService,
        },
      ],
    })
      // ↓ TransactionInterceptor 를 bypass — next.handle() 만 통과
      .overrideInterceptor(TransactionInterceptor)
      .useValue({
        intercept: (_context: unknown, next: { handle: () => unknown }) => next.handle(),
      })
      .compile();

    controller = module.get<MovieController>(MovieController);
    service    = module.get<MovieService>(MovieService);
  });
});
```

```typescript
// create 테스트 — queryRunner 는 직접 넘김
it('create 는 영화를 생성한다', async () => {
  const createDto = { title: '기생충' };
  const created   = { id: 1, title: '기생충' };
  const qr        = { manager: {} } as QueryRunner;
  //                  ↑ Controller 단위 테스트에서는 manager 만 있으면 됨

  jest.spyOn(service, 'create').mockResolvedValue(created as Movie);

  const result = await controller.create(createDto as CreateMovieDto, qr);

  expect(result).toEqual(created);
  expect(service.create).toHaveBeenCalledWith(createDto, qr.manager);
});
```

```txt
.overrideInterceptor(TransactionInterceptor).useValue({...}):
  실제 TransactionInterceptor 대신 bypass 구현으로 교체
  intercept 가 next.handle() 만 호출 → Interceptor 로직 건너뜀
  → 실제 DB 연결 없이 Controller 로직만 테스트 가능

{ manager: {} } as QueryRunner:
  Service 가 qr.manager 를 받아 사용하므로 최소한의 구조만 흉내
  실제 QueryRunner 기능은 Service 테스트에서 따로 검증
```

## QueryRunner Mock — 트랜잭션 Service 테스트 ⭐️

```txt
Service 에서 직접 QueryRunner 를 만들어 트랜잭션을 제어하는 경우
DataSource.createQueryRunner() 가 반환하는 QueryRunner 를 Mock 으로 대체

트랜잭션 흐름:
  qr.connect() → qr.startTransaction() → 로직 실행
  → 성공: qr.commitTransaction() → qr.release()
  → 실패: qr.rollbackTransaction() → qr.release()
```

```typescript
// Service 에서 트랜잭션을 직접 제어하는 경우
it('트랜잭션 성공 시 commit 을 호출한다', async () => {
  const managerMock = {
    save:   jest.fn().mockResolvedValue({ id: 1 }),
    delete: jest.fn(),
  };

  const qrMock = {
    connect:             jest.fn().mockResolvedValue(undefined),
    startTransaction:    jest.fn().mockResolvedValue(undefined),
    commitTransaction:   jest.fn().mockResolvedValue(undefined),
    rollbackTransaction: jest.fn().mockResolvedValue(undefined),
    release:             jest.fn().mockResolvedValue(undefined),
    manager:             managerMock,
  } as unknown as QueryRunner;

  // dataSource.createQueryRunner() 가 qrMock 을 반환하도록
  jest.spyOn(dataSource, 'createQueryRunner').mockReturnValue(qrMock);

  await service.create(createDto);

  expect(qrMock.connect).toHaveBeenCalled();
  expect(qrMock.startTransaction).toHaveBeenCalled();
  expect(qrMock.commitTransaction).toHaveBeenCalled();
  expect(qrMock.release).toHaveBeenCalled();
  expect(qrMock.rollbackTransaction).not.toHaveBeenCalled();
});

it('트랜잭션 실패 시 rollback 을 호출한다', async () => {
  const managerMock = {
    save: jest.fn().mockRejectedValue(new Error('DB 오류')),
  };

  const qrMock = {
    connect:             jest.fn().mockResolvedValue(undefined),
    startTransaction:    jest.fn().mockResolvedValue(undefined),
    commitTransaction:   jest.fn().mockResolvedValue(undefined),
    rollbackTransaction: jest.fn().mockResolvedValue(undefined),
    release:             jest.fn().mockResolvedValue(undefined),
    manager:             managerMock,
  } as unknown as QueryRunner;

  jest.spyOn(dataSource, 'createQueryRunner').mockReturnValue(qrMock);

  await expect(service.create(createDto)).rejects.toThrow('DB 오류');

  expect(qrMock.rollbackTransaction).toHaveBeenCalled();
  expect(qrMock.release).toHaveBeenCalled();
  expect(qrMock.commitTransaction).not.toHaveBeenCalled();
});
```

```txt
qrMock 구조:
  connect / startTransaction / commitTransaction / rollbackTransaction / release
  → 전부 mockResolvedValue(undefined) — 반환값 없이 호출만 확인

  manager
  → 실제 DB 작업(save / delete) 을 흉내내는 Mock 객체
  → Service 가 qr.manager.save() 를 쓰면 managerMock.save 로 연결됨

as unknown as QueryRunner:
  qrMock 이 실제 QueryRunner 타입과 다르므로 이중 캐스팅 필요

release 검증:
  성공 / 실패 둘 다 release 가 호출됐는지 반드시 확인
  → finally 블록에 release 가 있는지 검증하는 것과 같음
```

---

---

# 실전 패턴 — Arrange / Act / Assert ⭐️

```txt
Arrange  Mock 반환값 준비
Act      Service 메서드 실행
Assert   결과 / 예외 / 호출 인자 검증
```

```typescript
// 성공 케이스
it('영화 목록을 반환한다', async () => {
  // Arrange
  const movies = [{ id: 1, title: '기생충' }];
  mockMovieRepository.find.mockResolvedValue(movies);

  // Act
  const result = await service.findAll();

  // Assert
  expect(result).toEqual(movies);
  expect(mockMovieRepository.find).toHaveBeenCalledTimes(1);
});

// 예외 케이스
it('존재하지 않는 영화면 예외를 던진다', async () => {
  mockMovieRepository.findOne.mockResolvedValue(null);

  await expect(service.findOne(999)).rejects.toThrow(NotFoundException);

  expect(mockMovieRepository.findOne).toHaveBeenCalledWith({ where: { id: 999 } });
});

// DTO 인스턴스 만들기
it('유저를 생성한다', async () => {
  const createUserDto: CreateUserDto = {
    email:    'test@test.com',
    password: 'password',
  };
  // 타입 지정 → 필드 빠뜨리면 컴파일 에러로 바로 확인
});
```

---

---

# 예외 검증 패턴 ⭐️

## 기본 구조

```typescript
// 비동기 예외 검증 — await 반드시 필요
await expect(service.create(dto)).rejects.toThrow(ConflictException);

// 예외 메시지까지 검증
await expect(service.create(dto)).rejects.toThrow('동일한 이름과 생년월일을');

// 예외 타입 + 메시지 분리해서 각각 검증 (권장) ⭐️
await expect(service.create(dto)).rejects.toThrow(ConflictException);
await expect(service.create(dto)).rejects.toThrow('동일한 이름');
```

---

## ⚠️ 자주 하는 실수 3가지

### 실수 1 — await 누락

```typescript
// ❌ await 없으면 assertion 이 실행되지 않음
// 테스트가 통과해도 실제로 검증이 안 된 것
expect(service.create(dto)).rejects.toThrow(ConflictException);

// ✅ 반드시 await
await expect(service.create(dto)).rejects.toThrow(ConflictException);
```

```txt
rejects.toThrow() 는 Promise 를 반환
await 없으면 Promise 가 실행은 되지만 결과를 기다리지 않음
→ 예외가 안 던져져도 테스트 통과 → 의미 없는 테스트
```

---

### 실수 2 — 하나의 it 에서 service 두 번 호출 ⭐️

```typescript
// ❌ 잘못된 예 — service.create 를 두 번 호출
it('중복 감독이면 ConflictException 을 던진다', async () => {
  jest.spyOn(mockDirectorRepository, 'findOne')
    .mockResolvedValueOnce(existingDirector)   // 1번째 호출에 소진
    .mockResolvedValueOnce(existingDirector);  // 2번째 호출에 소진

  await expect(service.create(dto)).rejects.toThrow(ConflictException);
  // ↑ 1번째 service.create → findOne 이 existingDirector 반환 → 예외 발생 ✅
  //   여기서 mockResolvedValueOnce 1개 소진

  await expect(service.create(dto)).rejects.toThrow('동일한 이름...');
  // ↑ 2번째 service.create → findOne 이 existingDirector 반환 → 예외 발생 ✅ 처럼 보이지만
  //   실제로는 이미 2개 다 소진됐을 수 있어 undefined / null 반환 → 예외 안 던짐
  //   → Received promise resolved instead of rejected ❌
});
```

```typescript
// ✅ 올바른 예 1 — it 을 분리해서 각각 독립적으로 테스트
it('중복 감독이면 ConflictException 타입 예외를 던진다', async () => {
  jest.spyOn(mockDirectorRepository, 'findOne')
    .mockResolvedValue(existingDirector);   // Once 아님 → 항상 반환

  await expect(service.create(dto)).rejects.toThrow(ConflictException);
});

it('중복 감독이면 올바른 메시지를 포함한 예외를 던진다', async () => {
  jest.spyOn(mockDirectorRepository, 'findOne')
    .mockResolvedValue(existingDirector);

  await expect(service.create(dto)).rejects.toThrow('동일한 이름과 생년월일');
});
```

```typescript
// ✅ 올바른 예 2 — 하나의 it 에서 한 번만 호출하고 타입 + 메시지 동시 검증
it('중복 감독이면 ConflictException 을 던진다', async () => {
  jest.spyOn(mockDirectorRepository, 'findOne')
    .mockResolvedValue(existingDirector);

  const promise = service.create(dto);   // 한 번만 실행

  await expect(promise).rejects.toThrow(ConflictException);
  await expect(promise).rejects.toThrow('동일한 이름과 생년월일');
  //           ↑ 같은 promise 재사용 → service 두 번 호출 아님
});
```

```txt
mockResolvedValueOnce vs mockResolvedValue:
  Once  딱 한 번 반환 후 소진 → 그 이후 undefined 반환
        여러 번 호출 시나리오에서 순서 지정할 때 사용
  없음  항상 같은 값 반환 (소진 없음)
  → 예외 검증이 목적이면 Once 보다 mockResolvedValue 권장
```

---

## 실수 3 — 에러 메시지 공백·줄바꿈 불일치 ⭐️

```typescript
// 서비스 코드 — 템플릿 리터럴 안에 들여쓰기 포함
throw new ConflictException(`동일한 이름과 생년월일을 가진 감독이 존재합니다.
         id:${duplicateDirector.id}번을 확인해주세요`);
//         ↑ 줄바꿈(\n) + 공백 9칸 실제로 포함됨
```

```typescript
// ❌ 들여쓰기 칸 수가 다르면 메시지 불일치
await expect(service.create(dto)).rejects.toThrow(
  `동일한 이름과 생년월일을 가진 감독이 존재합니다.\n   id:${existingDirector.id}번을 확인해주세요`,
);
// → expect(received).rejects.toThrow(expected) 실패

// ✅ 해결법 1 — 서비스의 실제 문자열과 완전히 동일하게
await expect(service.create(dto)).rejects.toThrow(
  `동일한 이름과 생년월일을 가진 감독이 존재합니다.\n         id:${existingDirector.id}번을 확인해주세요`,
);

// ✅ 해결법 2 — 일부 문자열만 검증 (공백 문제 회피) ⭐️ 권장
await expect(service.create(dto)).rejects.toThrow('동일한 이름과 생년월일');
await expect(service.create(dto)).rejects.toThrow(`id:${existingDirector.id}번`);
```

```txt
왜 발생하나:
  템플릿 리터럴 안에서 줄바꿈하면 들여쓰기 공백이 문자열에 포함됨
  서비스 코드와 테스트 코드의 들여쓰기 칸이 다르면 불일치

해결 방향:
  전체 메시지 비교 → 서비스 코드 들여쓰기까지 정확히 복사
  일부 문자열 비교 → 핵심 키워드만 검증 (권장)
  에러 코드 비교   → toThrow(ConflictException) 타입으로만 검증
```

---

## 예외 검증 패턴 요약

```typescript
// 패턴 1 — 예외 타입만
await expect(service.create(dto)).rejects.toThrow(ConflictException);

// 패턴 2 — 메시지 일부 포함 여부
await expect(service.create(dto)).rejects.toThrow('동일한 이름');

// 패턴 3 — promise 재사용으로 한 번만 호출 + 둘 다 검증 ⭐️
const promise = service.create(dto);
await expect(promise).rejects.toThrow(ConflictException);
await expect(promise).rejects.toThrow('동일한 이름');

// 패턴 4 — it 분리 (가장 명확)
it('ConflictException 타입 검증', async () => { ... });
it('에러 메시지 검증', async () => { ... });
```

---

---

# E2E 테스트 ⭐️

```txt
E2E 테스트 = 실제 HTTP 요청 → 응답 전체 흐름 검증
supertest 로 HTTP 요청 → 실제 DB 에 연결 → 응답 검증

통합 테스트와 차이:
  통합 테스트  service.findAll() 직접 호출 → DB 응답 검증
  E2E 테스트   GET /v1/movie HTTP 요청      → 응답 status / body 검증
  → Guard / Pipe / Versioning 까지 실제로 통과하는지 확인 가능
```

## 파일 구조

```txt
test/
├── jest-e2e.json              E2E 전용 Jest 설정
├── load-integration-env.ts    .env.test 로드
└── movie.e2e-spec.ts          E2E 테스트 파일
```

## jest-e2e.json ⭐️

```json
{
  "moduleFileExtensions": ["js", "json", "ts"],
  "rootDir": ".",
  "testEnvironment": "node",
  "testRegex": ".e2e-spec.ts$",
  "transform": {
    "^.+\\.(t|j)s$": "ts-jest"
  },
  "moduleNameMapper": {
    "^src/(.*)$": "<rootDir>/../src/$1"
  }
}
```

```txt
rootDir: "."
  test/ 폴더 기준으로 경로 해석

testRegex: ".e2e-spec.ts$"
  .e2e-spec.ts 로 끝나는 파일만 실행

moduleNameMapper:
  "^src/(.*)$" → "<rootDir>/../src/$1"
    test/ 에서 src/ 경로로 import 할 때 경로 변환

uuid 패키지 대신 crypto.randomUUID() 사용:
  uuid v9+ 는 ESM 방식 배포 → Jest(CJS) 에서 에러 발생
  Node 18+ 내장 crypto.randomUUID() 로 대체 → ESM 문제 없음
  uuid.mock.ts 파일 불필요 / moduleNameMapper 에서 uuid 제거
```

## crypto.randomUUID() — uuid 대체 ⭐️

```txt
uuid 패키지 → crypto.randomUUID() 로 교체 권장
Node 18+ 내장 모듈 → 별도 설치 불필요 / ESM 문제 없음
v4 UUID 와 동일한 형식 반환
```

```typescript
// 기존 uuid 방식
import { v4 } from 'uuid';
callback(null, `${v4()}_${Date.now()}.${extension}`);

// ✅ crypto.randomUUID() 방식 (권장)
import { randomUUID } from 'crypto';
callback(null, `${randomUUID()}_${Date.now()}.${extension}`);

const filename = `${randomUUID()}_${Date.now()}.${extension}`;
```

## 테스트에서 crypto.randomUUID() Mock ⭐️

```typescript
// uuid.mock.ts 파일 필요 없음
// jest.mock('crypto') 로 직접 처리

jest.mock('crypto', () => ({
  ...jest.requireActual<typeof import('crypto')>('crypto'),
  //  ↑ crypto 의 나머지 메서드는 실제 구현 유지
  randomUUID: jest.fn(() => 'test-uuid'),
  //          ↑ randomUUID 만 고정값 반환으로 교체
}));

it('파일명에 uuid 가 포함된다', () => {
  const filename = `${randomUUID()}_${Date.now()}.mp4`;
  expect(filename).toContain('test-uuid');
});
```

```txt
jest.requireActual:
  mock 으로 교체할 때 나머지 메서드는 실제 구현을 그대로 사용
  randomUUID 만 고정값으로 바꾸고 crypto 의 다른 기능은 정상 동작

uuid.mock.ts 방식과 차이:
  uuid.mock.ts    파일 생성 + moduleNameMapper 설정 필요
  jest.mock('crypto')  테스트 파일 안에서 바로 처리 / 설정 불필요
```

## package.json 스크립트

```json
{
  "scripts": {
    "test:e2e":       "jest --config ./test/jest-e2e.json",
    "test:e2e:movie": "jest --config ./test/jest-e2e.json -- test/movie.e2e-spec.ts"
  }
}
```

```txt
test:e2e        전체 E2E 테스트 실행
test:e2e:movie  movie.e2e-spec.ts 만 실행
```

## E2E 테스트 파일 구조 ⭐️

```typescript
// test/movie.e2e-spec.ts
import './load-integration-env';   // ← 맨 위 / 환경 변수 먼저 로드

import { INestApplication, ValidationPipe, VersioningType } from '@nestjs/common';
import { Test, TestingModule }  from '@nestjs/testing';
import request                  from 'supertest';
import { DataSource }           from 'typeorm';
import { AppModule }            from '../src/app.module';

// ① DB 초기화 함수
async function resetE2eData(dataSource: DataSource) {
  await dataSource.query(`
    TRUNCATE TABLE
      movie_user_like, movie_file, movie_genres_genre,
      movie, movie_detail, genre, director, "user"
    RESTART IDENTITY CASCADE
  `);
}

// ② 앱 생성 함수 — main.ts 설정과 동일하게
async function createTestApp(): Promise<INestApplication> {
  const moduleFixture: TestingModule = await Test.createTestingModule({
    imports: [AppModule],
  }).compile();

  const app = moduleFixture.createNestApplication();

  // main.ts 와 동일하게 설정 → E2E 에서는 자동 적용 안 됨 ⚠️
  app.enableVersioning({ type: VersioningType.URI, defaultVersion: '1' });
  app.useGlobalPipes(new ValidationPipe({
    whitelist: true,
    forbidNonWhitelisted: true,
    transform: true,
    transformOptions: { enableImplicitConversion: true },
  }));

  await app.init();
  return app;
}

describe('Movie (e2e)', () => {
  let app:        INestApplication;
  let dataSource: DataSource;
  let movies:     Movie[];

  // ③ beforeAll — 앱 1번 생성 (60초 타임아웃)
  beforeAll(async () => {
    app        = await createTestApp();
    dataSource = app.get(DataSource);
  }, 60_000);

  afterAll(async () => {
    await app?.close();
  });

  // ④ beforeEach — 매 테스트 전 DB 초기화 + 시드 데이터
  beforeEach(async () => {
    await resetE2eData(dataSource);

    const movieRepository       = dataSource.getRepository(Movie);
    const movieDetailRepository = dataSource.getRepository(MovieDetail);
    const userRepository        = dataSource.getRepository(User);
    const genreRepository       = dataSource.getRepository(Genre);
    const directorRepository    = dataSource.getRepository(Director);

    const users     = await userRepository.save([
      userRepository.create({ email: 'e2e@test.com', password: 'password' }),
    ]);
    const directors = await directorRepository.save([
      directorRepository.create({ name: 'e2e-director', dob: new Date('1990-01-01'), nationality: 'Korean' }),
    ]);
    const genres    = await genreRepository.save([
      genreRepository.create({ name: 'e2e-genre' }),
    ]);

    movies = await movieRepository.save([
      movieRepository.create({
        title: 'e2e movie one', detail: movieDetailRepository.create({ detail: 'detail one' }),
        director: directors[0], genres: [genres[0]], creator: users[0],
        createAt: new Date('2026-01-01'), updateAt: new Date('2026-01-01'), files: [],
      }),
      movieRepository.create({
        title: 'e2e movie two', detail: movieDetailRepository.create({ detail: 'detail two' }),
        director: directors[0], genres: [genres[0]], creator: users[0],
        createAt: new Date('2026-01-02'), updateAt: new Date('2026-01-02'), files: [],
      }),
    ]);
  });
```

## 실제 HTTP 요청 검증 패턴 ⭐️

```typescript
  // GET 목록
  it('returns paginated movie list', async () => {
    const res = await request(app.getHttpServer())
      .get('/v1/movie')
      .query({ take: 10, order: 'id_ASC' })
      .expect(200);

    expect(res.body.count).toBe(2);
    expect(res.body.data).toHaveLength(2);
    expect(res.body.data[0]).toMatchObject({ id: movies[0].id, title: 'e2e movie one' });
  });

  // 쿼리 필터
  it('filters by title', async () => {
    const res = await request(app.getHttpServer())
      .get('/v1/movie')
      .query({ title: 'e2e movie one', take: 10, order: 'id_ASC' })
      .expect(200);

    expect(res.body.data).toHaveLength(1);
    expect(res.body.data[0].title).toBe('e2e movie one');
  });

  // 단건 조회
  it('returns a movie by id', async () => {
    const res = await request(app.getHttpServer())
      .get(`/v1/movie/${movies[0].id}`)
      .expect(200);

    expect(res.body).toMatchObject({ id: movies[0].id, title: 'e2e movie one' });
    expect(res.body.detail).toMatchObject({ detail: 'detail one' });
  });

  // 404
  it('returns 404 for unknown id', async () => {
    await request(app.getHttpServer()).get('/v1/movie/99999').expect(404);
  });

  // 인증이 필요한 요청
  it('returns 401 without token', async () => {
    await request(app.getHttpServer()).post('/v1/movie').expect(401);
  });

  it('returns result with Bearer token', async () => {
    const res = await request(app.getHttpServer())
      .post('/v1/movie')
      .set('Authorization', `Bearer ${token}`)
      .send(createMovieDto)
      .expect(201);

    expect(res.body).toHaveProperty('id');
  });
});
```

```txt
request(app.getHttpServer()):
  supertest 가 NestJS 앱 서버로 HTTP 요청 전송

.query({})    Query Parameter 설정
.send({})     Request Body 설정
.set('Authorization', 'Bearer ...')  헤더 설정
.expect(200)  상태코드 검증

toMatchObject:
  res.body 의 일부 필드만 검증 (toEqual 은 완전 일치)
  → createdAt 같은 가변값 무시하고 핵심 필드만 확인

createTestApp() 에서 main.ts 설정 복사 ⚠️:
  enableVersioning → /v1/movie 경로 동작
  useGlobalPipes   → DTO 검증 / 타입 변환
  설정 없으면 /v1/movie 404 / DTO 검증 안 됨
```

---

---

# PostgreSQL 통합 테스트 ⭐️

## 단위 테스트와 뭐가 다른가

```txt
단위 테스트   Mock 으로 DB 대신       → 빠름 / 실제 쿼리 검증 안 됨
통합 테스트   실제 DB 에 연결         → 느림 / 실제 쿼리·트랜잭션 검증 가능

언제 쓰나:
  findAll 의 JOIN / LIKE 검색이 실제로 동작하는지
  캐시가 실제로 저장되는지
  트랜잭션 롤백이 제대로 되는지
  → 이런 건 Mock 으로는 검증 불가 → 통합 테스트 필요
```

---

## 흐름 한눈에

```txt
① DB 준비          pnpm db:test:create → movie_test DB 생성
② 모듈 세팅        beforeAll — 실제 DB 연결 + 모듈 1번 생성 (느리므로 1번만)
③ 데이터 초기화    beforeEach — 매 테스트 전 TRUNCATE + 시드 데이터 삽입
④ 테스트 실행      실제 service 메서드 호출
⑤ 결과 검증        DB 에 실제로 저장됐는지 확인
⑥ 정리             afterAll — DB 연결 해제 + 모듈 종료
```

---

## 환경 준비 (처음 한 번)

```txt
필요한 파일:
  .env.test                          실제 값 (gitignore)
  .env.test.example                  템플릿 (커밋 가능)
  test/load-integration-env.ts       .env.test 로드
  scripts/create-test-database.js    테스트 DB 생성
```

```bash
# .env.test / .env.test.example
# 나머지 환경 변수는 .env 에서 그대로 사용
# DB 이름만 테스트 전용으로 변경
DB_DATABASE=movie_test
```

```typescript
// test/load-integration-env.ts
import * as dotenv from 'dotenv';
import * as path from 'path';

dotenv.config({ path: path.resolve(process.cwd(), '.env.test') });
```

```json
// package.json
{
  "scripts": {
    "db:test:create":   "node scripts/create-test-database.js",
    "test:integration": "jest --testRegex='.*\\.integration\\.spec\\.ts$'"
  }
}
```

```txt
실행 순서:
  pnpm db:test:create    → movie_test DB 없으면 생성
  pnpm test:integration  → 통합 테스트 실행
```

---

## 통합 테스트 파일 구조 ⭐️

```typescript
// movie.service.integration.spec.ts

import '../../test/load-integration-env';  // ← 파일 맨 위 / 환경변수 먼저 로드

import { Test, TestingModule }           from '@nestjs/testing';
import { ConfigModule, ConfigService }   from '@nestjs/config';
import { TypeOrmModule }                 from '@nestjs/typeorm';
import { CacheModule, Cache, CACHE_MANAGER } from '@nestjs/cache-manager';
import { DataSource }                    from 'typeorm';
import * as Joi from 'joi';

// ① 테스트 전 DB 초기화 함수 — 매 테스트마다 깨끗하게
async function resetTestData(dataSource: DataSource) {
  await dataSource.query(`
    TRUNCATE TABLE
      movie_user_like, movie_file, movie_genres_genre,
      movie, movie_detail, genre, director, "user"
    RESTART IDENTITY CASCADE
  `);
}

describe('MovieService - Integration Test', () => {
  let service:      MovieService;
  let dataSource:   DataSource;
  let moduleRef:    TestingModule;
  let cacheManager: Cache;

  // 시드 데이터 — beforeEach 에서 생성한 것을 테스트에서 참조
  let users:     User[];
  let directors: Director[];
  let genres:    Genre[];
  let movies:    Movie[];

  // ② beforeAll — 모듈·DB 연결 1번만 (느리므로)
  beforeAll(async () => {
    moduleRef = await Test.createTestingModule({
      imports: [
        ConfigModule.forRoot({
          isGlobal:      true,
          ignoreEnvFile: true,   // load-integration-env 에서 이미 로드 → 중복 방지
          validationSchema: Joi.object({
            ENV: Joi.string().valid('dev', 'prod').required(),
            DB_TYPE: Joi.string().valid('postgres').required(),
            DB_HOST: Joi.string().required(),
            DB_PORT: Joi.number().required(),
            DB_USER: Joi.string().required(),
            DB_PASSWORD: Joi.string().required(),
            DB_DATABASE: Joi.string().required(),
            SALT_ROUNDS: Joi.number().required(),
            ACCESS_TOKEN_SECRET: Joi.string().required(),
            REFRESH_TOKEN_SECRET: Joi.string().required(),
          }),
        }),
        CacheModule.register({ isGlobal: true }),
        TypeOrmModule.forRootAsync({
          inject: [ConfigService],
          useFactory: (configService: ConfigService) => ({
            type:       configService.get('DB_TYPE') as 'postgres',
            host:       configService.get('DB_HOST'),
            port:       configService.get<number>('DB_PORT'),
            username:   configService.get('DB_USER'),
            password:   configService.get('DB_PASSWORD'),
            database:   configService.get('DB_DATABASE'),
            entities:   [Movie, MovieDetail, MovieFile, Director, Genre, MovieUserLike, User],
            synchronize: true,   // 테스트 DB 에서만 허용
          }),
        }),
        TypeOrmModule.forFeature([Movie, MovieDetail, MovieFile, Director, Genre, MovieUserLike, User]),
      ],
      providers: [MovieService, CommonService],
    }).compile();

    service      = moduleRef.get<MovieService>(MovieService);
    dataSource   = moduleRef.get<DataSource>(DataSource);
    cacheManager = moduleRef.get<Cache>(CACHE_MANAGER);
  }, 30_000);   // DB 연결 시간 여유를 위해 타임아웃 30초 설정

  // ③ afterAll — DB 연결 해제 + 모듈 종료
  afterAll(async () => {
    if (dataSource?.isInitialized) await dataSource.destroy();
    await moduleRef?.close();
  });

  // ④ beforeEach — 매 테스트 전 초기화 + 시드 데이터 삽입
  beforeEach(async () => {
    await cacheManager.clear();
    await resetTestData(dataSource);   // 테이블 전체 TRUNCATE

    // 시드 데이터 삽입 — 각 테스트가 동일한 초기 상태에서 시작
    const userRepository     = dataSource.getRepository(User);
    const directorRepository = dataSource.getRepository(Director);
    const genreRepository    = dataSource.getRepository(Genre);
    const movieRepository    = dataSource.getRepository(Movie);
    const movieDetailRepository = dataSource.getRepository(MovieDetail);

    users = await userRepository.save([
      userRepository.create({ email: 'test1@test.com', password: 'password' }),
      userRepository.create({ email: 'test2@test.com', password: 'password' }),
    ]);

    directors = await directorRepository.save([
      directorRepository.create({ name: 'director1', dob: new Date('1990-01-01'), nationality: 'Korean' }),
      directorRepository.create({ name: 'director2', dob: new Date('1991-01-01'), nationality: 'Korean' }),
    ]);

    genres = await genreRepository.save([
      genreRepository.create({ name: 'genre1' }),
      genreRepository.create({ name: 'genre2' }),
    ]);

    movies = await movieRepository.save(
      Array.from({ length: 10 }, (_, i) =>
        movieRepository.create({
          title:    `movie ${i + 1}`,
          detail:   movieDetailRepository.create({ detail: `detail${i + 1}` }),
          director: directors[0],
          genres:   genres.slice(0, 2),
          creator:  users[0],
        })
      )
    );
  });
```

---

## 테스트 작성 패턴 ⭐️

```typescript
  // 기본 — 정의 확인
  it('should be defined', () => {
    expect(service).toBeDefined();
  });

  // 캐시 검증 — 실제 cacheManager 에 저장됐는지
  it('should cache the recent movies', async () => {
    const result = await service.findMovieRecent();
    const cached = await cacheManager.get('MOVIE_RECENT');
    expect(cached).toEqual(result);
  });

  // 실제 DB 쿼리 검증
  it('should return a movie by id', async () => {
    const result = await service.findOne(movies[0].id);
    expect(result!.title).toBe(movies[0].title);
    expect(result!.director.id).toBe(movies[0].director.id);
  });

  // 실제 트랜잭션 — create 테스트
  it('should create a movie correctly', async () => {
    const qr = dataSource.createQueryRunner();
    await qr.connect();
    await qr.startTransaction();

    try {
      const result = await service.create(createDto, ['test.mp4'], qr, users[0].id);
      await qr.commitTransaction();
      expect(result).toHaveProperty('id');
    } finally {
      await qr.release();
    }
  });

  // 예외 검증
  it('should throw NotFoundException if movie not found', async () => {
    await expect(service.remove(999)).rejects.toThrow(NotFoundException);
  });
});
```

---

## 핵심 포인트 정리

```txt
beforeAll   모듈 + DB 연결을 1번만 생성 (30초 타임아웃 설정)
beforeEach  TRUNCATE + 시드 데이터 삽입 → 테스트 간 격리

TRUNCATE ... RESTART IDENTITY CASCADE:
  모든 테이블 데이터 삭제 + ID 시퀀스 초기화
  외래키 관계도 CASCADE 로 같이 삭제
  → 매 테스트가 동일한 초기 상태 보장

ignoreEnvFile: true:
  load-integration-env.ts 에서 .env.test 를 직접 로드했으므로
  ConfigModule 이 .env 파일 재로드하지 않도록 차단

synchronize: true:
  테스트 DB 에서만 허용
  Entity 변경 시 테이블 자동 반영

단위 테스트 vs 통합 테스트:
  단위   mock.find.toHaveBeenCalled()    → 호출 여부만
  통합   실제 DB 조회 후 result.title 검증 → 실제 동작 검증
```

---

---

# Coverage ⭐️

```bash
pnpm test:cov
```

## 결과표 읽기

|항목|의미|
|---|---|
|`% Stmts`|실행문 중 테스트 중 실행된 비율|
|`% Branch`|조건 분기(if/else) 중 실행된 경로 비율|
|`% Funcs`|함수 중 테스트 중 호출된 비율|
|`% Lines`|코드 라인 중 실행된 비율|
|`Uncovered Line #s`|테스트가 실행하지 않은 줄 번호|

```txt
수치가 낮다 → 성공·실패·조건 분기 테스트가 부족할 수 있음
수치가 높다 → 좋은 테스트를 자동으로 의미하지 않음
→ 빠진 분기를 찾는 참고 지표로 사용
```

## Coverage 설정

```json
// package.json — jest 설정
{
  "collectCoverageFrom": ["**/*.(t|j)s"],
  "coveragePathIgnorePatterns": [
    "\\.module\\.ts$",
    "\\.dto\\.ts$",
    "\\.entity\\.ts$"
  ]
}
```

## 테스트하지 않는 것

|제외 대상|이유|
|---|---|
|`*.module.ts`|Provider 연결 설정 중심|
|로직 없는 `*.dto.ts`|필드 선언만 있음|
|로직 없는 `*.entity.ts`|컬럼·관계 선언만 있음|
|NestJS `@Get()`, `@UseGuards()` 자체|프레임워크 기능|
|TypeORM / Winston 내부 동작|외부 라이브러리 기능|

```txt
단, 내 로직이 있으면 제외하면 안 됨:
  @Transform() 으로 query 값 변환하는 DTO
  계산 메서드가 있는 Entity
  access token 만 통과시키는 Guard 로직
  직접 작성한 Pipe 검증 로직

핵심:
  단순 선언만 있음      → 테스트 X / Coverage 제외 가능
  조건·변환·계산이 있음  → 테스트 O / Coverage 포함
```

## 특정 경로만 Coverage 확인

```json
// package.json scripts 에 등록
{
  "scripts": {
    "test:user": "jest --testPathPatterns=src/user --coverage --collectCoverageFrom=src/user/**/*.ts"
  }
}
```

```txt
--testPathPatterns=src/user
  경로에 src/user 가 포함된 테스트 파일만 실행

--collectCoverageFrom=src/user/**/*.ts
  src/user 내부 TypeScript 파일만 Coverage 대상으로 수집
  테스트 없는 파일도 Coverage 결과에 표시

⚠️ E2E 테스트가 test/ 폴더에 있으면 실행 안 됨
```

---

---

# @suites/unit — 자동 Mock 세팅

```bash
지금까지의 방식 = 수동 Mock
  const mockRepo = { find: jest.fn(), save: jest.fn() };
  providers: [{ provide: getRepositoryToken(Movie), useValue: mockRepo }]
  → 의존성 추가할 때마다 Mock 도 수동으로 등록

@suites/unit = 자동 Mock
  TestBed.solitary(Service).compile() 한 줄로 모든 의존성 자동 Mock 생성
  Mocked<T> 타입으로 캐스팅 없이 바로 mockResolvedValue 사용 가능

# → 자세한 내용은 [[NestJS_Testing_Suites]] 참고
```

```bash
pnpm install --save-dev @suites/unit @suites/di.nestjs @suites/doubles.jest
```

```typescript
// 수동 방식
const mockMovieRepository = { find: jest.fn(), findOne: jest.fn() };
providers: [{ provide: getRepositoryToken(Movie), useValue: mockMovieRepository }]
(mockMovieRepository.find as jest.Mock).mockResolvedValue(movies);

// @suites/unit 방식
const { unit, unitRef } = await TestBed.solitary(MovieService).compile();
service         = unit;
movieRepository = unitRef.get(getRepositoryToken(Movie) as string);
movieRepository.find.mockResolvedValue(movies);  // 캐스팅 없이 바로 사용
```

```txt
@automock/jest 는 2026년 6월 30일 deprecated 예정
새 프로젝트에는 @suites/unit 권장
```

---

---

# 명령어 모음

```bash
pnpm test           # 전체 유닛 테스트
pnpm test:watch     # 파일 감시 모드 (저장 시 자동 실행)
pnpm test:cov       # Coverage 확인
pnpm test:e2e       # E2E 테스트
pnpm test:user      # 특정 경로만 테스트 (package.json 에 등록 후)

# HTML Coverage 리포트 확인 (Mac)
open coverage/lcov-report/index.html
```