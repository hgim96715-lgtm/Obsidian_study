---
aliases:
  - API Versioning
  - Header Versioning
  - URI Versioning
tags:
  - NestJS
related:
  - "[[00_NestJS_Ecosystem_HomePage]]"
  - "[[NestJS_Controller]]"
  - "[[NestJS_Module]]"
  - "[[NestJS_Concept]]"
  - "[[NestJS_Swagger]]"
---
# NestJS_Versioning — API 버전 관리

>[!info]
>API 버전 관리 = 기존 클라이언트를 깨뜨리지 않으면서 새 버전의 엔드포인트를 추가하는 방법. 
>`app.enableVersioning()`으로 전역 설정하고 `@Controller({ version: '1' })`으로 버전을 지정한다.
> URI·Header·Media Type 세 가지 방식 중 선택.

---

# 왜 버전 관리가 필요한가 ⭐️⭐️⭐️⭐️

```txt
문제:
  /users API의 응답 형태를 변경해야 함
  { name: 'hong' } → { firstName: 'hong', lastName: 'gil' }

  기존 클라이언트(앱·프론트)가 name 필드를 쓰고 있음
  → 그냥 바꾸면 기존 클라이언트가 깨짐

해결 — 버전 관리:
  GET /v1/users → 기존 응답 유지 (name)
  GET /v2/users → 새 응답 (firstName, lastName)

  기존 클라이언트는 v1 유지
  새 클라이언트는 v2 사용
  → 둘 다 동작, 점진적 마이그레이션 가능
```

---

# 버전 지정 방식 3가지 ⭐️⭐️⭐️⭐️

## URI 버전 (가장 직관적)

```typescript
// main.ts
app.enableVersioning({
  type: VersioningType.URI,
  // prefix 기본값: 'v'
  // GET /v1/users, GET /v2/users
});

// prefix를 바꾸려면
app.enableVersioning({
  type:   VersioningType.URI,
  prefix: 'api/v',  // GET /api/v1/users
});
```

```txt
클라이언트가 URL로 버전 선택:
  GET /v1/users
  GET /v2/users

장점: URL만 보면 버전이 보임 — 가장 직관적
단점: URL이 길어짐, REST 순수주의에서는 URL이 리소스만 나타내야 한다고 봄
```

## Header 버전

```typescript
// main.ts
app.enableVersioning({
  type:   VersioningType.HEADER,
  header: 'version',   // 요청 헤더의 키 이름
  //      ↑ 이 이름의 헤더를 읽어서 버전 결정
});
```

```txt
클라이언트가 헤더로 버전 선택:
  GET /users
  version: 1       ← 헤더에 버전 지정

  curl -H "version: 1" http://localhost:3030/users
  fetch('/users', { headers: { version: '1' } })

header: 'version':
  NestJS가 읽을 헤더 키 이름
  클라이언트는 이 이름으로 헤더를 보내야 함
  원하는 이름으로 변경 가능:
    header: 'x-api-version'  → 헤더: x-api-version: 1
    header: 'api-version'    → 헤더: api-version: 2
    header: 'version'        → 헤더: version: 1

장점: URL이 깔끔하게 유지됨
단점: 헤더를 직접 설정해야 해서 브라우저 주소창에서 테스트 불편
      Swagger에서 헤더 설정 필요
```

## Media Type 버전 (Accept 헤더)

```typescript
// main.ts
app.enableVersioning({
  type: VersioningType.MEDIA_TYPE,
  key:  'v=',  // Accept 헤더에서 찾을 키
  // Accept: application/json;v=1
});
```

```txt
클라이언트가 Accept 헤더로 버전 선택:
  GET /users
  Accept: application/json;v=1

장점: HTTP 표준에 가장 가까운 방식
단점: 가장 복잡하고 잘 안 씀
```

---

# main.ts에 등록 ⭐️⭐️⭐️⭐️

```typescript
// main.ts
import { VersioningType } from '@nestjs/common';

async function bootstrap() {
  const app = await NestFactory.create(AppModule);

  app.enableVersioning({
    type:           VersioningType.URI,    // URI 방식
    defaultVersion: '1',                   // 버전 없는 요청 → v1으로
  });

  await app.listen(3030);
}
```

```txt
defaultVersion: '1':
  @Version() 데코레이터가 없는 컨트롤러에 기본 버전 지정
  없으면 버전 지정 없는 엔드포인트는 404
```

---

# Controller · Method 레벨 버전 지정 ⭐️⭐️⭐️⭐️

## Controller 레벨

```typescript
// v1 컨트롤러
@Controller({ path: 'users', version: '1' })
export class UsersControllerV1 {
  @Get()
  findAll() {
    return [{ name: 'hong' }];  // 구 응답
  }
}

// v2 컨트롤러
@Controller({ path: 'users', version: '2' })
export class UsersControllerV2 {
  @Get()
  findAll() {
    return [{ firstName: 'hong', lastName: 'gil' }];  // 새 응답
  }
}
```

## Method 레벨 (하나의 컨트롤러에서 버전별 메서드)

```typescript
@Controller('users')
export class UsersController {
  @Get()
  @Version('1')                  // GET /v1/users
  findAllV1() {
    return [{ name: 'hong' }];
  }

  @Get()
  @Version('2')                  // GET /v2/users
  findAllV2() {
    return [{ firstName: 'hong', lastName: 'gil' }];
  }
}
```

## 여러 버전 동시 지원

```typescript
@Get()
@Version(['1', '2'])    // v1과 v2 모두 이 메서드로 처리
findAll() {
  return users;
}
```

## VERSION_NEUTRAL — 버전 무관

```typescript
import { VERSION_NEUTRAL } from '@nestjs/common';

@Controller({ path: 'health', version: VERSION_NEUTRAL })
export class HealthController {
  @Get()
  check() {
    return 'ok';  // 어느 버전으로 요청해도 응답
  }
}
```

```txt
VERSION_NEUTRAL:
  /v1/health, /v2/health, /health 전부 동작
  버전이 의미 없는 공통 엔드포인트 (헬스체크, 상태 확인 등)
```

---

# 실전 — 두 버전 함께 운영하는 패턴 ⭐️⭐️⭐️

```typescript
// app.module.ts — 두 버전 컨트롤러 모두 등록
@Module({
  controllers: [
    UsersControllerV1,
    UsersControllerV2,
  ],
})

// main.ts
app.enableVersioning({
  type:           VersioningType.URI,
  defaultVersion: '1',  // 버전 없는 요청 → v1
});
```

```bash
# URI 방식 테스트
curl http://localhost:3030/v1/users   # V1 응답
curl http://localhost:3030/v2/users   # V2 응답
curl http://localhost:3030/users      # defaultVersion → V1

# Header 방식 테스트
curl -H "version: 1" http://localhost:3030/users
curl -H "version: 2" http://localhost:3030/users
```

---

# 언제 어떤 방식을 쓰는가 ⭐️⭐️⭐️

|방식|장점|단점|적합한 경우|
|---|---|---|---|
|URI|직관적, 브라우저·curl에서 쉽게 테스트|URL이 길어짐|공개 API, Swagger 문서화|
|Header|URL 깔끔|헤더 설정 필요|내부 API, 모바일 클라이언트|
|Media Type|HTTP 표준에 가까움|복잡, 잘 안 씀|REST 순수주의|

```txt
이 프로젝트처럼 NestJS + Next.js 구조:
  URI 방식이 가장 단순하고 Swagger에서 확인하기 쉬움
  Next.js의 fetchApi에서 URL만 바꾸면 됨

  fetchApi('/v1/users')  →  fetchApi('/v2/users')
```