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
  - "[[Deploy_CloudMVP]]"
  - "[[NestJS_Module]]"
---
# NestJS_Versioning — API 버전 관리

> [!info] 
> 버전 관리 = 기존 API를 깨지 않으면서 새 버전을 추가하는 방법. 클라이언트(구앱·신앱·파트너)마다 다른 버전을 동시에 제공할 수 있다.

---

# 왜 필요한가 ⭐️⭐️⭐️

```txt
API 업데이트 상황:
  v1 /movie → title, genre 반환
  v2 /movie → title, genre, rating 추가 반환

버전 관리 없이:
  /movie를 바꾸면 기존 클라이언트(구앱)가 그대로 깨짐
  → 버전 관리로 하위 호환성 보장하면서 새 버전 추가 가능
```

## 구앱 / 신앱 시나리오 — 모바일 앱에서 특히 중요 ⭐️⭐️⭐️⭐️

```txt
웹 앱은 서버가 배포되면 사용자가 새로고침만 해도 최신 버전을 씀
→ API를 바꿔도 사용자가 곧 최신 앱을 쓰게 됨

모바일 앱(iOS/Android)은 다름:
  앱스토어 심사 시간 + 사용자가 업데이트를 안 할 수 있음
  → 구앱(v1 API 사용)과 신앱(v2 API 사용)이 동시에 살아있는 기간이 반드시 존재
  → 이 기간 동안 서버는 v1, v2를 모두 지원해야 함

예시:
  사용자 A: 앱 업데이트 안 함 → 여전히 v1 API 호출
  사용자 B: 최신 앱    →  v2 API 호출
  → 서버: 두 버전 동시 응답 가능해야 함

파트너/외부 클라이언트도 마찬가지:
  외부 서비스가 우리 API를 직접 호출하는 경우
  그쪽 코드를 우리가 강제로 바꿀 수 없음
  → 구버전 API를 유지하면서 신버전도 제공해야 함
```

---

# 버전 관리 방식 3가지 ⭐️

|방식|버전 전달 위치|예시|
|---|---|---|
|`URI Versioning`|URL 경로|`GET /v1/movie`|
|`Header Versioning`|요청 헤더|`GET /movie` + `version: 1`|
|`Media Type Versioning`|Accept 헤더|`GET /movie` + `Accept: application/json;v=2`|

```txt
가장 많이 쓰는 방식: URI Versioning
  URL만 봐도 버전을 알 수 있어서 직관적
  서버 로그에서도 버전별로 바로 구분 가능
  브라우저 주소창, curl, Postman 등 어디서든 테스트하기 쉬움
```

---

# URI Versioning 설정 ⭐️⭐️⭐️⭐️

```typescript
// main.ts
import { VersioningType } from '@nestjs/common';

async function bootstrap() {
  const app = await NestFactory.create(AppModule);

  app.enableVersioning({
    type:           VersioningType.URI,
    defaultVersion: ['1', '2'],
    // ↑ @Version() 없는 엔드포인트에 기본 적용할 버전
    // 단일: defaultVersion: '1'
    // 복수: ['1', '2'] → 두 버전 모두 기본 처리
  });

  await app.listen(3000);
}
```

```typescript
// 컨트롤러 레벨 버전 지정
@Controller({ path: 'movie', version: '1' })
export class MovieControllerV1 {
  @Get()
  findAll() { return { title: '마이클', genre: '드라마' }; }
}

@Controller({ path: 'movie', version: '2' })
export class MovieControllerV2 {
  @Get()
  findAll() { return { title: '마이클', genre: '드라마', rating: 8.5 }; }
}
```

```txt
GET /v1/movie → MovieControllerV1 (구앱이 호출)
GET /v2/movie → MovieControllerV2 (신앱이 호출)
```

## 모듈에 두 컨트롤러 함께 등록

```typescript
@Module({
  controllers: [MovieControllerV1, MovieControllerV2],
  providers:   [MovieService],
})
export class MovieModule {}
```

---

# Header Versioning

```typescript
// main.ts
app.enableVersioning({
  type:   VersioningType.HEADER,
  header: 'version',   // 헤더 키 이름
});
```

```http
GET /movie
version: 1
```

```txt
URL은 동일 / 헤더로 버전 구분
장점: URL을 깔끔하게 유지
단점: 브라우저 주소창에서 바로 테스트 불가, 로그에서 버전 구분 어려움
```

---

# Media Type Versioning

```typescript
app.enableVersioning({
  type: VersioningType.MEDIA_TYPE,
  key:  'v=',   // Accept 헤더에서 추출할 키
});
```

```http
GET /movie
Accept: application/json;v=2
```

---

# 메서드 단위 버전 지정 ⭐️⭐️

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

```txt
컨트롤러 레벨 vs 메서드 레벨:
  컨트롤러 레벨 → 그 컨트롤러의 모든 메서드에 적용
  메서드 레벨   → 해당 메서드 하나에만 적용

두 방식 혼합도 가능:
  컨트롤러는 version: '1' 지정
  특정 메서드만 @Version('2')로 오버라이드
```

---

# VERSION_NEUTRAL — 버전 무관 엔드포인트

```typescript
import { VERSION_NEUTRAL } from '@nestjs/common';

@Version(VERSION_NEUTRAL)
@Get('health')
health() {
  return 'ok';
  // 버전 지정 없이도, 어떤 버전 요청에도 항상 접근 가능
}
```

```txt
사용 사례:
  헬스체크 (/health, /ping)
  버전 무관한 공통 유틸 엔드포인트
  버전 전환 중 임시로 모든 버전에 응답해야 할 때
```

---

# 실전 — 구버전 폐기(Deprecation) 전략 ⭐️⭐️

```txt
버전을 무한정 유지할 수는 없음 — 구버전 폐기 시 절차:

  ① 폐기 예고 (Deprecation Notice)
     응답 헤더에 경고 추가
     Deprecation: true
     Sunset: Sat, 31 Dec 2025 23:59:59 GMT  ← 지원 종료 날짜

  ② 모니터링
     v1 엔드포인트 호출 수가 0에 가까워지면 제거 검토

  ③ 제거
     v1 컨트롤러 삭제
     Swagger/문서에서도 제거
```

```typescript
// 폐기 예정 버전 응답 헤더 추가 예시
@Version('1')
@Get()
findAllV1(@Res({ passthrough: true }) res: Response) {
  res.setHeader('Deprecation', 'true');
  res.setHeader('Sunset', 'Sat, 31 Dec 2025 23:59:59 GMT');
  return this.movieService.findAllV1();
}
```

---

# 한눈에

```txt
버전 방식 선택:
  URI Versioning    → /v1/movie  (가장 직관적, 실무 표준)
  Header Versioning → 헤더에 version: 1 (URL 깔끔하지만 테스트 불편)
  Media Type        → Accept: application/json;v=1 (REST 원칙에 가깝지만 복잡)

설정:
  main.ts: app.enableVersioning({ type: VersioningType.URI, defaultVersion: '1' })
  컨트롤러: @Controller({ path: 'movie', version: '1' })
  메서드:   @Version('1') / @Version(['1', '2']) / VERSION_NEUTRAL

구앱/신앱 시나리오:
  모바일 앱은 강제 업데이트 불가 → v1(구앱)과 v2(신앱) 동시 운영 기간 발생
  → 두 컨트롤러 함께 모듈에 등록, 같은 Service 공유 가능
  → 충분히 v1 호출이 줄면 폐기(Deprecation) 예고 후 제거
```