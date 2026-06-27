---
aliases:
  - Suites
  - automock
  - TestBed
tags:
  - NestJS
related:
  - "[[00_NestJS_Ecosystem_HomePage]]"
---

# NestJS_Testing_Suites — 자동 Mock 테스팅

```
Suites = NestJS DI 메타데이터를 읽어서 Mock 을 자동 생성해주는 Unit Test 프레임워크
수동으로 jest.fn() / getRepositoryToken / providers 등록 없이 테스트 세팅 가능
```

---

---

# @automock/jest 참고 (deprecated)

```
@automock/jest 를 아직 쓰는 프로젝트가 있어서 참고용으로 남겨둠
2026년 6월 30일 완전 deprecated 예정
critical fix 만 받으며 새 기능 없음
→ 새 프로젝트에는 @suites/unit 사용 권장
```

```bash
# 기존 automock 제거
pnpm uninstall @automock/jest @automock/adapters.nestjs
```

---

---

# @suites/unit 이란 ⭐️

```
@automock/jest 의 후속 프로젝트
NestJS constructor 의 의존성 메타데이터를 자동으로 읽어서
모든 의존성을 자동으로 Mocked<T> 타입으로 만들어줌

기존 수동 Mock 방식:
  const mockRepo = { find: jest.fn(), save: jest.fn() };
  providers: [{ provide: getRepositoryToken(Movie), useValue: mockRepo }]
  → 의존성 추가할 때마다 Mock 도 수동으로 추가해야 함

Suites 방식:
  const { unit, unitRef } = await TestBed.solitary(MovieService).compile();
  movieRepository = unitRef.get(getRepositoryToken(Movie) as string);
  → 의존성 자동 감지 + 자동 Mock 생성
  → 추가 등록 없이 unitRef.get 으로 바로 꺼내서 사용
```

---

---

# 설치 ⭐️

```bash
pnpm install --save-dev @suites/unit @suites/di.nestjs @suites/doubles.jest
```

```
패키지 역할:
  @suites/unit          TestBed 핵심 API
  @suites/di.nestjs     NestJS DI 메타데이터 읽기
  @suites/doubles.jest  Jest Mock 연동
```

---

---

# tsconfig.json 확인

```json
{
  "compilerOptions": {
    "emitDecoratorMetadata": true,
    "experimentalDecorators": true
  }
}
```

```
Suites 는 constructor 의존성 메타데이터를 읽어서 Mock 을 자동 생성
→ emitDecoratorMetadata 가 꺼져 있으면 메타데이터를 읽지 못함

NestJS 프로젝트면 보통 이미 켜져 있음
설치 후 테스트 실패 시 먼저 이 설정 확인
```

---

---

# global.d.ts 설정

프로젝트 루트에 `global.d.ts` 파일 생성 후 아래 내용 추가.

```typescript
/// <reference types="@suites/doubles.jest/unit" />
/// <reference types="@suites/di.nestjs/types" />
```

```
Mocked<T> 같은 Suites 타입을 전역으로 사용하려면 필요
이 설정 없으면 Mocked<Repository<Movie>> 타입 인식 안 됨
```

---

---

# `{ts}Mocked<T>` 타입 ⭐️

```typescript
import { type Mocked } from '@suites/unit';
import { Repository }  from 'typeorm';
import { Movie }       from './entity/movie.entity';

let movieRepository: Mocked<Repository<Movie>>;
//                   ↑ Repository<Movie> 의 모든 메서드를 자동으로 jest.fn() 으로 변환
```

```
Mocked<T> 란:
  T 의 모든 메서드를 jest.Mock 타입으로 변환
  → mockResolvedValue / mockReturnValue 등 Mock 메서드 바로 사용 가능
  → 수동으로 (repo.find as jest.Mock) 캐스팅 불필요

수동 방식:
  (movieRepository.find as jest.Mock).mockResolvedValue(movies);

Suites 방식:
  movieRepository.find.mockResolvedValue(movies);  // 캐스팅 없이 바로 사용
```

---

---

# 기본 세팅 ⭐️

```typescript
// movie.service.spec.ts
import { TestBed, type Mocked } from '@suites/unit';
import { Repository, DataSource }  from 'typeorm';
import { getRepositoryToken }  from '@nestjs/typeorm';
import { CACHE_MANAGER }       from '@nestjs/cache-manager';

import { MovieService }        from './movie.service';
import { Movie }               from './entity/movie.entity';
import { MovieDetail }         from './entity/movie-detail.entity';
import { MovieFile }           from './entity/movie-file.entity';
import { Director }            from 'src/director/entity/director.entity';
import { Genre }               from 'src/genre/entity/genre.entity';
import { MovieUserLike }       from './entity/movie-user-like.entity';
import { User }                from 'src/user/entity/user.entity';
import { CommonService }       from 'src/common/common.service';

describe('MovieService', () => {
  let service: MovieService;

  // 수동 방식처럼 jest.fn() 선언 없이 Mocked<T> 로 선언만
  let movieRepository:         Mocked<Repository<Movie>>;
  let movieDetailRepository:   Mocked<Repository<MovieDetail>>;
  let movieFileRepository:     Mocked<Repository<MovieFile>>;

  beforeEach(async () => {
    // TestBed.solitary — MovieService 의 의존성 자동 감지 + 전부 Mock 생성
    const { unit, unitRef } = await TestBed.solitary(MovieService).compile();

    service = unit;

    // unitRef.get 으로 자동 생성된 Mock 꺼내기
    movieRepository         = unitRef.get(getRepositoryToken(Movie) as string);
    movieDetailRepository   = unitRef.get(getRepositoryToken(MovieDetail) as string);
    movieFileRepository     = unitRef.get(getRepositoryToken(MovieFile) as string);
  });

  it('should be defined', () => {
    expect(service).toBeDefined();
  });
});
```

```
TestBed.solitary(Service).compile():
  Service 의 constructor 를 읽어서 모든 의존성 자동 Mock 생성
  await 필요

unit:
  테스트할 실제 Service 인스턴스

unitRef.get(토큰):
  자동 생성된 Mock 인스턴스 꺼내기
  Repository → getRepositoryToken(Entity) as string
  일반 Service / DataSource → 클래스 그대로 전달
  CACHE_MANAGER → CACHE_MANAGER 토큰 그대로
```

---

---

# 수동 방식 vs Suites 비교 ⭐️

```
                수동 Mock 방식                   Suites 방식
──────────────────────────────────────────────────────────────────
Mock 선언    const mockRepo = {                let movieRepository:
              find: jest.fn(),                  Mocked<Repository<Movie>>;
              save: jest.fn(),                  (선언만, jest.fn() 없음)
             };

모듈 세팅    providers: [{                     TestBed
              provide:                           .solitary(MovieService)
               getRepositoryToken(Movie),        .compile()
              useValue: mockRepo,
             }]

꺼내기       module.get<MovieService>()         unit (바로 사용)
             module.get<Repository<Movie>>       unitRef.get(getRepositoryToken(Movie))

Mock 호출    (mockRepo.find as jest.Mock)        movieRepository.find
             .mockResolvedValue(movies)           .mockResolvedValue(movies)
                                                (캐스팅 불필요)

의존성 추가  providers 에 수동 등록 필요          자동 감지 → 자동 추가
──────────────────────────────────────────────────────────────────
```

---

---

# TestBed.solitary vs TestBed.create

```
TestBed.solitary(Service):
  테스트 대상 Service 의 모든 의존성을 전부 Mock 으로 대체
  외부 의존성 없이 Service 로직만 격리해서 테스트
  Unit Test 에서 가장 많이 사용 ⭐️

TestBed.create(Service):
  구버전 automock 방식 (TestBed.solitary 로 대체)
  Suites 에서는 solitary 사용 권장
```

---

---

# 검증 패턴 — Arrange / Act / Assert

```typescript
it('전체 영화 목록을 반환한다', async () => {
  // Arrange — Mocked<T> 라서 캐스팅 없이 바로 mockResolvedValue
  const movies = [{ id: 1, title: '기생충' }];
  movieRepository.find.mockResolvedValue(movies as Movie[]);

  // Act
  const result = await service.findAll();

  // Assert
  expect(result).toEqual(movies);
  expect(movieRepository.find).toHaveBeenCalledTimes(1);
});

it('존재하지 않는 영화면 NotFoundException 을 던진다', async () => {
  movieRepository.findOne.mockResolvedValue(null);

  await expect(service.findOne(999)).rejects.toThrow(NotFoundException);

  expect(movieRepository.findOne).toHaveBeenCalledWith(
    expect.objectContaining({ where: { id: 999 } })
  );
});
```

```bash
수동 Mock 방식과 검증 방법은 동일
차이는 Mock 선언 / 세팅 방식만
# → Matcher / rejects.toThrow 등은 [[NestJS_Testing]] 참고
```