---
aliases: [API Versioning, Header Versioning, URI Versioning]
tags:
  - NestJS
related:
  - "[[00_NestJS_Ecosystem_HomePage]]"
  - "[[NestJS_Controller]]"
---

# NestJS_Versioning — API 버전 관리

## 한 줄 요약

```txt
버전 관리 = 기존 API 를 깨지 않으면서 새 버전을 추가하는 방법
클라이언트가 어떤 버전을 원하는지 서버에 알려주는 방식
```

---

---

#  왜 필요한가 ⭐️

```txt
API 업데이트 상황:
  v1 /movie → title, genre 반환
  v2 /movie → title, genre, rating 추가 반환

  기존 v1 클라이언트를 유지하면서 v2 도 제공해야 함
  → 둘 다 동시에 지원하는 방법 필요

버전 관리 없이:
  /movie 를 바꾸면 기존 클라이언트 깨짐
  → 버전 관리로 하위 호환성 보장
```

---

---

#  버전 관리 방식 3가지 ⭐️

```txt
URI Versioning:
  URL 에 버전 포함
  GET /v1/movie
  GET /v2/movie

Header Versioning:
  요청 헤더에 버전 전달
  GET /movie
  version: 1

Media Type Versioning:
  Accept 헤더에 버전 포함
  GET /movie
  Accept: application/json;v=2
```

---

---

#  URI Versioning 설정 ⭐️

```typescript
// main.ts
import { VersioningType } from '@nestjs/common';

async function bootstrap() {
  const app = await NestFactory.create(AppModule);

  app.enableVersioning({
    type: VersioningType.URI,
    // → /v1/... / /v2/... 형태로 URL 앞에 붙음

    defaultVersion: ['1', '2'],
    // ↑ 버전 지정 없는 요청에 기본 적용할 버전
    // 배열로 여러 버전 동시 기본 지정 가능
    // defaultVersion: '1' 단일 지정도 가능
  });

  await app.listen(3000);
}
```

```txt
defaultVersion:
  @Version() 을 붙이지 않은 엔드포인트에 적용될 기본 버전
  단일: defaultVersion: '1'
  복수: defaultVersion: ['1', '2']  → 두 버전 모두 기본 처리
```

```typescript
// 컨트롤러에 버전 지정
@Controller({ path: 'movie', version: '1' })
export class MovieControllerV1 {
  @Get()
  findAll() { return '버전 1 응답'; }
}

@Controller({ path: 'movie', version: '2' })
export class MovieControllerV2 {
  @Get()
  findAll() { return '버전 2 응답'; }
}
```

```txt
GET /v1/movie → MovieControllerV1
GET /v2/movie → MovieControllerV2
```

---

---

# Header Versioning

```typescript
// main.ts
app.enableVersioning({
  type:   VersioningType.HEADER,
  header: 'version',   // 헤더 키 이름
});
```

```txt
요청:
  GET /movie
  version: 1   ← 헤더에 버전 전달

URL 은 동일 / 헤더로 버전 구분
```

---

---

#  Media Type Versioning

```typescript
// main.ts
app.enableVersioning({
  type: VersioningType.MEDIA_TYPE,
  key:  'v=',   // Accept 헤더에서 추출할 키
});
```

```txt
요청:
  GET /movie
  Accept: application/json;v=2
```

---

---

#  메서드 단위 버전 지정

```typescript
@Controller('movie')
export class MovieController {

  // 단일 버전
  @Version('1')
  @Get()
  findAllV1() { return '버전 1'; }

  // 여러 버전 동시 지원 ⭐️
  @Version(['1', '2'])
  @Get()
  findAllV1V2() { return '버전 1, 2 모두 응답'; }
  // → v1 / v2 요청 둘 다 이 메서드로 처리됨
}
```

---

---

#  VERSION_NEUTRAL — 모든 버전 허용

```typescript
import { VERSION_NEUTRAL } from '@nestjs/common';

@Version(VERSION_NEUTRAL)
@Get('health')
health() {
  return 'ok';
  // 버전 관계없이 항상 접근 가능
}
```

---

---

# 한눈에

|방식|버전 전달 위치|
|---|---|
|`VersioningType.URI`|URL `/v1/movie`|
|`VersioningType.HEADER`|헤더 `version: 1`|
|`VersioningType.MEDIA_TYPE`|`Accept: application/json;v=1`|

```txt
가장 많이 쓰는 방식:
  URI Versioning → URL 만 봐도 버전 알 수 있어서 직관적
```