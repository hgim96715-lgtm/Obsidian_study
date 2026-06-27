---
aliases:
  - 테스트 오류
  - Testing Troubleshooting
  - Jest 에러
tags:
  - NestJS
related:
  - "[[00_NestJS_Ecosystem_HomePage]]"
---

# NestJS_Testing_Troubleshooting — 테스트 오류 모음

```txt
테스트 작성 중 만난 오류 / 원인 / 해결법 기록
```

---

---

# Cannot find module 'src/...' ⭐️

## 오류

```txt
Cannot find module 'src/common/entity/base-table.entity'
from 'user/entity/user.entity.ts'
```

## 원인

```txt
프로젝트 코드에서는 src/... 절대 경로로 import 하지만
Jest 는 테스트 실행 시 그 경로가 실제 어느 폴더인지 모름

import { BaseTable } from 'src/common/entity/base-table.entity'
→ Jest 가 실제 파일 경로로 연결 못 함 → Cannot find module
```

## 해결 ⭐️

```json
// package.json — jest 설정
{
  "roots": ["<rootDir>/src"],
  "moduleNameMapper": {
    "^src/(.*)$": "<rootDir>/src/$1"
  }
}
```

```txt
roots:
  Jest 가 테스트할 소스 기준 폴더를 src 로 지정

moduleNameMapper:
  import 의 src/... 경로를 실제 파일 위치로 변환

흐름:
  import from 'src/common/entity/...'
    ↓ ^src/(.*)$ 패턴 매칭
    ↓ <rootDir>/src/common/entity/... 로 변환
    ↓ Jest 가 실제 파일 찾아서 실행

⚠️ rootDir: "src" 설정이 따로 있으면 중복 경로 주의:
  <rootDir>/src/$1 → src/src/common/... 가 될 수 있음
  rootDir: "src" 유지 시 → <rootDir>/$1 사용
```

---

---

# Nest can't resolve dependencies ⭐️

## 오류

```txt
Nest can't resolve dependencies of the UserService (?).
Please make sure that the argument "UserRepository" at index [0]
is available in the RootTestModule context.

Nest can't resolve dependencies of the UserService (UserRepository, ?).
Please make sure that the argument ConfigService at index [1]
is available in the RootTestModule module.
```

## 원인

```txt
Unit Test 에서 Test.createTestingModule() 은
등록한 Provider 만 사용

constructor 에 있는 의존성이 하나라도 등록 안 되면 에러 발생

index [N]:
  constructor 의 N번째 의존성이 누락됐다는 의미
  ? 가 붙은 위치 = 누락된 자리
  index [0] → 첫 번째 인자 (보통 Repository)
  index [1] → 두 번째 인자 (보통 ConfigService / JwtService)
```

## 해결 ⭐️

```typescript
const mockUserRepository = {
  find:    jest.fn(),
  findOne: jest.fn(),
  save:    jest.fn(),
  delete:  jest.fn(),
};

const mockConfigService = {
  get:        jest.fn(),
  getOrThrow: jest.fn(),
};

await Test.createTestingModule({
  providers: [
    UserService,
    {
      provide:  getRepositoryToken(User),  // index [0] — Repository
      useValue: mockUserRepository,
    },
    {
      provide:  ConfigService,             // index [1] — ConfigService
      useValue: mockConfigService,
    },
  ],
}).compile();
```

```txt
getRepositoryToken(Entity):
  @InjectRepository(Entity) 가 찾는 주입 토큰
  Entity 가 다르면 토큰도 다름
    @InjectRepository(User)  → getRepositoryToken(User)
    @InjectRepository(Movie) → getRepositoryToken(Movie)

ConfigService / JwtService 등 일반 서비스:
  provide: ConfigService (클래스 자체가 토큰)
  useValue: { 사용하는 메서드만 jest.fn() }

의존성 추가될 때마다:
  테스트 모듈에도 같은 방식으로 Mock 등록
  오류의 index [N] 번호로 어느 자리인지 확인
```

---

---

# test:integration — No tests found ⭐️

## 오류

```txt
No tests found, exiting with code 1
```

## 원인

```txt
test:integration 스크립트는 *.integration.spec.ts 파일만 실행

"test:integration": "jest --testRegex='.*\\.integration\\.spec\\.ts$'"

→ 이 이름의 파일이 하나도 없으면
  Jest 가 No tests found → exit code 1 발생

설정 오류가 아님
돌릴 테스트 파일이 없다는 뜻
```

현재 테스트 스크립트별 대상

|스크립트|대상 파일|
|---|---|
|`pnpm test`|`src/**/*.spec.ts` (단위 테스트)|
|`pnpm test:e2e`|`test/*.e2e-spec.ts`|
|`pnpm test:integration`|`*.integration.spec.ts` → 없으면 에러|

## 해결

```txt
방법 1 — 파일 먼저 만들기 (통합 테스트 쓸 예정이면)
  src/movie/movie.integration.spec.ts
  src/user/user.integration.spec.ts
  → 파일 만든 후 pnpm test:integration 실행

방법 2 — passWithNoTests 옵션 추가 (파일 없어도 통과)
```

```json
// package.json
{
  "scripts": {
    "test:integration": "jest --testRegex='.*\\.integration\\.spec\\.ts$' --passWithNoTests"
  }
}
```

```txt
방법 3 — 전체 앱 흐름만 보려면 E2E 사용
  pnpm test:e2e  (이미 test/ 아래 e2e 설정 있음)
```

```txt
파일 네이밍 규칙:
  Unit Test        {파일명}.spec.ts
  Integration Test {파일명}.integration.spec.ts  ← 이 이름이어야 인식됨
  E2E Test         {파일명}.e2e-spec.ts
```

---
---
# Jest experimental-vm-modules (Prisma adapter) ⭐️

## 오류

```txt
SyntaxError: Cannot use import statement in a module
또는 Prisma adapter 관련 ESM 에러
```

## 해결

```json
// package.json
{
  "scripts": {
    "test:integration": "NODE_OPTIONS=\"$NODE_OPTIONS --experimental-vm-modules\" jest --testRegex='.*\\.integration\\.spec\\.ts$' --runInBand",
    "test:integration:watch": "NODE_OPTIONS=\"$NODE_OPTIONS --experimental-vm-modules\" jest --testRegex='.*\\.integration\\.spec\\.ts$' --watch --runInBand"
  }
}
```

```txt
--experimental-vm-modules:
  Prisma adapter 가 ESM 모듈을 사용
  Jest 기본 설정에서 ESM 처리 안 됨
  → NODE_OPTIONS 로 실험적 vm 모듈 활성화

--runInBand:
  테스트를 순차 실행 (병렬 아님)
  DB 연결 테스트에서 충돌 방지
```

---

---

# Jest Cannot find module ./internal/class.js

## 오류

```txt
Cannot find module './internal/class.js' from '...'
```

## 원인

```txt
Jest 가 .js 확장자를 그대로 resolve 하려다 실패
TypeScript 에서 import 시 .js 확장자를 명시하는 경우 발생
```

## 해결

```json
// package.json jest 설정
{
  "jest": {
    "moduleNameMapper": {
      "src/(.*)": "<rootDir>/src/$1",
      "^(\\.{1,2}/.*)\\.js$": "$1"
    }
  }
}
```

```txt
"^(\\.{1,2}/.*)\\.js$": "$1"
  ./ 또는 ../ 로 시작하는 .js import 를
  확장자 없는 경로로 변환
  → Jest 가 .ts 파일을 찾을 수 있게 됨
```

---

---

# GET /movie — pg DeprecationWarning

## 오류

```txt
DeprecationWarning: ...
또는 쿼리 결과 이상 / 연결 경고
```

## 원인

```txt
동일 DB 연결에서 findMany 와 count 를 병렬(Promise.all)로 실행
pg 드라이버가 같은 연결을 동시에 사용하면 경고 또는 충돌
```

## 해결

```typescript
// ❌ 병렬 실행
const [movies, count] = await Promise.all([
  this.prisma.movie.findMany({ ... }),
  this.prisma.movie.count({ ... }),
]);

// ✅ 순차 실행
const movies = await this.prisma.movie.findMany({ ... });
const count  = await this.prisma.movie.count({ ... });
```