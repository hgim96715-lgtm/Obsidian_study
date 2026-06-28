---
aliases: [API 문서화, OpenAPI, Swagger]
tags:
  - NestJS
related:
  - "[[00_NestJS_Ecosystem_HomePage]]"
  - "[[NestJS_DTO]]"
  - "[[NestJS_JwtGuard]]"
---
# NestJS_Swagger — API 문서화

> [!info] 
> Swagger는 API 문서를 자동으로 만들어주는 프레임워크(OpenAPI Specification 표준)다. 
> `main.ts`에서 설정하고, UI에서 직접 API를 테스트할 수 있다. 
> `addBearerAuth()`에 커스텀 이름을 주면, 그 이름을 `@ApiBearerAuth()`에도 똑같이 넘겨야 자물쇠 버튼과 라우트가 정확히 짝지어진다.

## Postman vs Swagger

|구분|용도|
|---|---|
|Swagger UI|서버가 제공하는 API 명세 확인, 간단한 실행 테스트|
|Postman|요청 저장, 환경변수, 테스트 스크립트, 여러 요청 흐름 실행|

```txt
역할이 달라서 둘 다 쓰는 경우가 많음 — Swagger 는 "문서 + 빠른 테스트", Postman 은 "본격적인 테스트 워크플로우"
```

---

# 설치 & main.ts 설정 ⭐️

```bash
pnpm add @nestjs/swagger
```

```typescript
// main.ts
import { SwaggerModule, DocumentBuilder } from '@nestjs/swagger';

async function bootstrap() {
  const app = await NestFactory.create(AppModule);

  const config = new DocumentBuilder()
    .setTitle('My API')
    .setDescription('API 설명')
    .setVersion('0.0.1')
    .addBearerAuth()
    .build();

  const document = SwaggerModule.createDocument(app, config);
  SwaggerModule.setup('api', app, document);

  await app.listen(3000);
}
```

|메서드|역할|
|---|---|
|`.setTitle('제목')`|Swagger UI 상단에 표시되는 제목|
|`.setDescription('설명')`|API 에 대한 간단한 설명/소개|
|`.setVersion('x.y.z')`|API 버전 표시 (문서용 — 아래 별도 설명)|
|`.addBearerAuth()` / `.addBasicAuth()`|인증 방식 추가 — 아래 별도 설명|
|`.build()`|위 설정들을 하나의 config 객체로 완성|

```txt
createDocument(app, config): 앱의 모든 컨트롤러·DTO 를 스캔해서 Swagger 문서 생성
setup('api', app, document): 'api' 경로에 Swagger UI 연결 → http://localhost:3000/api 에서 접속
```

---

# setVersion — 뭘로 맞추나 ⭐️⭐️

```txt
Swagger 의 문서/API 스펙 버전 라벨 — npm 패키지(package.json)의 version 과 같을 필요는 없음
다만 맞춰두면 "코드 버전 = API 문서 버전" 이라 헷갈릴 일이 줄어듦 (작은 프로젝트엔 이게 편함)
```

|값|언제|
|---|---|
|`0.0.1` (또는 `0.x.y`)|개발 중, 아직 외부에 정식 공개하지 않은 단계 (MVP 이전, 계속 구조가 바뀜)|
|`1.0.0`|"외부에 공개하는 API 가 안정됐다" 고 볼 수 있을 때 (첫 정식 배포 등)|
|`1.1.0`, `2.0.0` ...|이후엔 SemVer 관례대로: 기능 추가는 minor, 호환 깨지는 변경은 major|

```txt
튜토리얼에 적힌 '1.0' 은 대부분 그냥 예시 값임 — 그대로 따라 적을 필요 없음
지금 한창 개발/재구현 중이라면 0.0.1 이 맞고, 처음으로 외부에 내놓을 때(포트폴리오 배포 등) 1.0.0 으로 올리면 됨

API 자체의 버전 관리(v1, v2 같은 URI/Header 버전)와는 다른 개념 — 그건 [[NestJS_Versioning]] 참고
```

---

# addBearerAuth() vs addBasicAuth() — 언제 쓰나 ⭐️⭐️⭐️

```txt
이 둘은 "Swagger UI 에 인증 입력 버튼을 추가하는 것" 일 뿐 — 실제로 어떤 걸 추가해야 하는지는
내 API 가 Authorization 헤더를 "어떤 방식으로" 검사하는지에 따라 결정됨
```

|방식|HTTP 헤더 형태|실제로 필요한 경우|
|---|---|---|
|`addBearerAuth()`|`Authorization: Bearer <토큰>`|JWT 기반 인증 — `@UseGuards(JwtAuthGuard)` 가 붙은 라우트가 하나라도 있다면 거의 항상 필요|
|`addBasicAuth()`|`Authorization: Basic <base64(id:pw)>`|엔드포인트가 진짜로 이 형식의 헤더를 직접 검사할 때만 — 흔하지 않음|

```txt
⚠️ 흔한 오해 정정 — "로그인(login) 엔드포인트엔 Basic Auth" 가 아님
  로그인이 email/password 를 요청 Body(DTO)로 받는 일반적인 구조라면, 이건 Basic Auth 와 전혀 무관함
  (Swagger 상에서는 @ApiBody({ type: LoginDto }) 로 표현 — 인증 방식 추가와는 별개)

  addBasicAuth() 가 실제로 맞는 경우는, 어떤 엔드포인트가 "Authorization: Basic ..." 헤더 자체를
  직접 검사하도록 만들어졌을 때뿐 — 예: 외부 웹훅 수신처가 고정된 id:pw 로 인증하는 엔드포인트,
  내부 관리용 도구처럼 단순한 고정 계정 인증이 필요한 경우 등 (일반적인 사용자 로그인 흐름엔 거의 안 씀)
```

## 판단 기준

|상황|필요한 것|
|---|---|
|로그인은 body 로, 이후 요청은 Bearer 토큰 (JWT 기반, 가장 흔한 구조)|`addBearerAuth()` 만 있으면 충분|
|특정 엔드포인트가 `Authorization: Basic` 헤더를 직접 검사함|그 엔드포인트에 한해 `addBasicAuth()` 추가|
|인증이 필요한 엔드포인트가 전혀 없음 (전부 공개 API)|둘 다 생략 가능|

---

# addBearerAuth() 옵션과 보안 스킴 이름 — @ApiBearerAuth()와 짝맞추기 ⭐️⭐️⭐️

```txt
addBearerAuth() 는 인자 없이 써도 동작하지만, 실제 시그니처는 다음과 같음:

  addBearerAuth(options?: SecuritySchemeObject, name?: string): this

  options   Swagger UI에 보여줄 "이 보안 방식이 정확히 뭔지"에 대한 설명 객체
  name      이 보안 스킴(설정)에 붙이는 이름 — 생략하면 기본값 'bearer'
```

```typescript
const config = new DocumentBuilder()
  .setTitle('My API')
  .setDescription('API 설명')
  .setVersion('0.0.1')
  .addBearerAuth(
    {
      type: 'http',
      scheme: 'bearer',
      bearerFormat: 'JWT',
      description: '로그인 후 발급받은 토큰을 입력하세요.',
    },
    'access-token', // ← 이 이름을 @ApiBearerAuth()에도 그대로 써야 함
  )
  .build();
```

## options 객체 필드

|필드|역할|
|---|---|
|`type: 'http'`|OpenAPI가 분류하는 인증 방식 대분류 — Bearer/Basic 둘 다 `'http'`에 속함|
|`scheme: 'bearer'`|`type: 'http'` 안에서 구체적으로 어떤 방식인지 — Bearer 토큰 방식임을 명시|
|`bearerFormat: 'JWT'`|토큰 형식이 JWT라는 힌트 — 문서에만 표시될 뿐, 실제 검증 로직에 영향 없음|
|`description`|Swagger UI의 Authorize 팝업에 보여줄 안내 문구|

## name — 왜 @ApiBearerAuth()와 맞춰야 하나 ⭐️⭐️⭐️

```txt
addBearerAuth(options, 'access-token') 처럼 이름을 직접 지정했다면,
그 보안 스킴을 라우트에 적용할 때도 같은 이름을 @ApiBearerAuth()에 넘겨야 함:

  @ApiBearerAuth('access-token')   ✅ 이름이 일치 — 정확히 이 스킴과 연결됨
  @ApiBearerAuth()                 ⚠️ 인자 없이 쓰면 기본 이름 'bearer'를 찾음
                                       → addBearerAuth에서 'bearer'라는 이름을 안 썼다면 어긋남

이름을 아예 안 지정했다면(addBearerAuth() 그냥 인자 없이) 기본값 'bearer'로 양쪽이
자동으로 맞춰지기 때문에 신경 쓸 일이 없음 — "이름을 커스텀하게 지었을 때만" 챙겨야 하는 문제
```

|상황|addBearerAuth()|@ApiBearerAuth()|결과|
|---|---|---|---|
|둘 다 이름 생략|기본값 `'bearer'`|기본값 `'bearer'`|자동으로 일치 ✅|
|addBearerAuth에만 커스텀 이름|`'access-token'`|이름 생략 (`'bearer'` 찾음)|불일치 ⚠️|
|둘 다 같은 커스텀 이름|`'access-token'`|`'access-token'`|일치 ✅|

```txt
왜 굳이 이름을 커스텀하게 짓는 경우가 있나:
  Bearer 토큰을 종류별로 여러 개 문서화해야 할 때 (예: access-token / refresh-token을
  서로 다른 보안 스킴으로 따로 보여주고 싶을 때) — 이런 경우가 아니면 그냥 기본값(이름 생략)으로 충분
```

---

# OpenAPI CLI Plugin — 자동 유추 ⭐️

```txt
문제: @ApiProperty 를 모든 DTO 필드에 하나씩 붙이면 코드가 복잡해짐
해결: TypeScript Reflection 으로 유추 가능한 값은 자동 처리 — @ApiProperty 생략해도 스키마 자동 생성
```

```json
// nest-cli.json
{
  "collection": "@nestjs/schematics",
  "sourceRoot": "src",
  "compilerOptions": {
    "plugins": ["@nestjs/swagger"]
  }
}
```

```txt
Plugin 활성화 후:
  DTO 의 타입/필드명 → 자동 유추
  @IsOptional() → required: false 자동 반영
  @ApiProperty 는 추가 설명(description, example 등)이 필요할 때만 명시
```

## 옵션을 직접 지정하고 싶다면 — 객체 형태 ⭐️⭐️⭐️

```txt
문자열("@nestjs/swagger")만 적으면 전부 기본값으로 동작함
세부 동작을 바꾸고 싶다면 { name, options } 객체 형태로 적어야 함
```

```json
// nest-cli.json
{
  "compilerOptions": {
    "plugins": [
      {
        "name": "@nestjs/swagger",
        "options": {
          "classValidatorShim": true,
          "introspectComments": true,
          "dtoFileNameSuffix": [".dto.ts"],
          "controllerFileNameSuffix": [".controller.ts"]
        }
      }
    ]
  }
}
```

|옵션|기본값|역할|
|---|---|---|
|`classValidatorShim`|`true`|class-validator 데코레이터(`@IsOptional()` 등)를 읽어서 Swagger 스키마(required 여부 등)에 반영 — 명시 안 해도 기본 켜져 있음|
|`introspectComments`|`false`|DTO 필드/컨트롤러 메서드 위의 주석을 읽어서 `description`(DTO)·`summary`(컨트롤러)로 자동 사용 — 기본은 꺼져 있어서 켜야 동작|
|`dtoFileNameSuffix`|`['.dto.ts', '.entity.ts']`|이 이름 패턴의 파일만 "DTO 파일"로 인식해서 스캔 — 네이밍 컨벤션이 다르면 직접 지정|
|`controllerFileNameSuffix`|`['.controller.ts']`|이 이름 패턴의 파일만 "Controller 파일"로 인식해서 스캔|

```typescript
// introspectComments: true 일 때 — 주석이 description 으로 자동 반영됨
export class CreateMovieDto {
  /** 영화 제목 (2~100자) */
  @IsString()
  title: string;            // @ApiProperty({ description: '...' }) 없이도 위 주석이 자동 반영됨
}
```

```txt
⚠️ dtoFileNameSuffix 를 직접 지정하면 기본값(['.dto.ts', '.entity.ts'])이 "대체" 됨(합쳐지는 게 아님)
   → entity.ts 파일도 같이 스캔하고 싶다면 ['.dto.ts', '.entity.ts'] 처럼 둘 다 명시해야 함

플러그인 옵션을 바꿨다면 dist/ 를 지우고 다시 빌드해야 반영됨 (캐시된 컴파일 결과가 남아있을 수 있음)
```

---

# 데코레이터

## 컨트롤러 / 엔드포인트

```txt
@ApiTags 없어도 자동 태그 ⭐️:
  @nestjs/swagger 는 기본적으로 autoTagControllers: true
  @ApiTags() 가 없어도 Controller 이름에서 'Controller' 를 제거한 값을 자동 태그로 사용
  MovieController → 'Movie' 로 자동 그룹화

@ApiTags 를 직접 쓰는 경우: 그룹명을 바꾸고 싶을 때, 여러 Controller 를 같은 그룹으로 묶고 싶을 때
```

```typescript
@ApiTags('movie')
@Controller('movie')
export class MovieController {

  @ApiOperation({ summary: '영화 전체 조회' })
  @ApiResponse({ status: 200, description: '조회 성공' })
  @Get()
  findAll() {}

  @ApiParam({ name: 'id', description: '영화 ID' })
  @Get(':id')
  findOne(@Param('id') id: string) {}

  @ApiQuery({ name: 'title', required: false })
  @Get('search')
  search(@Query('title') title: string) {}

  @ApiOperation({ summary: '영화 생성' })
  @ApiBody({ type: CreateMovieDto })
  @ApiBearerAuth()              // 로그인 필요 — Bearer 토큰 인증 (커스텀 이름 썼다면 @ApiBearerAuth('이름'))
  @Post()
  create(@Body() dto: CreateMovieDto) {}
}
```

```txt
로그인 자체(이메일/비밀번호 검증)는 별도 컨트롤러에 @ApiBody({ type: LoginDto }) 로 표현 —
@ApiBasicAuth() 를 붙이는 게 아님 (위 "언제 쓰나" 섹션 참고)
```

## DTO 에서 @ApiProperty

```typescript
export class CreateMovieDto {
  @ApiProperty({ description: '영화 제목', example: '기생충' })
  @IsString()
  title: string;

  @ApiProperty({ description: '장르', enum: ['action', 'drama', 'comedy'] })
  @IsString()
  genre: string;

  @ApiProperty({ description: '평점', example: 8.5, required: false })
  @IsOptional()
  @IsNumber()
  rating?: number;
}
```

```txt
@ApiProperty 없으면 Swagger 가 해당 필드를 문서에 표시 못 함 (Plugin 활성화 시엔 기본값은 자동 유추됨)
description / example / required / enum 등으로 문서를 더 풍부하게 만들 수 있음
```

---

# 데코레이터 정리

|데코레이터|위치|역할|
|---|---|---|
|`@ApiTags('그룹명')`|컨트롤러|API 그룹화 (없으면 Controller명 자동)|
|`@ApiOperation({ summary })`|메서드|엔드포인트 설명|
|`@ApiResponse({ status, description })`|메서드|응답 정의 (여러 개 가능)|
|`@ApiParam({ name })`|메서드|Path 파라미터 정의|
|`@ApiQuery({ name })`|메서드|쿼리 파라미터 정의|
|`@ApiBody({ type })`|메서드|Body 정의|
|`@ApiProperty({ description })`|DTO 프로퍼티|DTO 필드 설명|
|`@ApiHideProperty()`|DTO 프로퍼티|Swagger 문서에서 필드 숨기기|
|`@ApiPropertyOptional()`|DTO 프로퍼티|`@ApiProperty({ required: false })` 단축|
|`@ApiBearerAuth(name?)`|메서드|Bearer 토큰 필요 표시 — `name`을 커스텀했다면 반드시 같은 문자열 전달|
|`@ApiBasicAuth()`|메서드|Basic 인증 필요 표시 (드물게 사용 — 위 참고)|
|`@ApiExcludeEndpoint()`|메서드|해당 엔드포인트 문서에서 숨기기|
|`@ApiExcludeController()`|컨트롤러|컨트롤러 전체 문서에서 숨기기|

## @ApiHideProperty ⭐️

```typescript
export class UserDto {
  @ApiProperty({ description: '이메일' })
  email: string;

  @ApiHideProperty()   // Swagger 문서에서 이 필드 숨김 — 코드 동작엔 영향 없음
  password: string;
}
```

```txt
언제 쓰나: password / 내부 토큰처럼 문서에 노출되면 안 되는 필드
@ApiExcludeEndpoint(): 내부 관리용/테스트용 엔드포인트를 문서에서 통째로 숨김
@ApiExcludeController(): 컨트롤러 전체를 문서에서 숨김
```

---

# Swagger UI 인증 버튼(Authorize 🔒) ⭐️

```txt
Swagger UI 의 Authorize 버튼은 DocumentBuilder 에 추가한 인증 방식만큼만 나타남
→ 필요한 방식만 추가하면 됨 — addBasicAuth/addBearerAuth 둘 다 필수는 아님 (위 "언제 쓰나" 참고)
이름을 커스텀하게 지었다면 위 "addBearerAuth() 옵션과 보안 스킴 이름" 섹션의 매칭 규칙을 따를 것
```

```typescript
const config = new DocumentBuilder()
  .setTitle('My API')
  .setVersion('0.0.1')
  .addBearerAuth()   // JWT 기반이라면 이것만으로 충분한 경우가 많음
  .build();
```

```txt
⚠️ addBearerAuth() 를 안 추가하면: @ApiBearerAuth() 를 컨트롤러에 붙여도
   UI 에 토큰 입력란 자체가 안 나타나서 인증된 테스트가 불가능함
   → @ApiXxxAuth() 데코레이터(엔드포인트에 붙이는 표시)와 addXxxAuth()(UI 버튼 생성)는 항상 짝이 맞아야 함
```

---

# Authorization 헤더를 직접 다뤄야 할 때 — 커스텀 데코레이터로 숨기기 ⭐️

```txt
@Headers('authorization') 를 파라미터에 직접 쓰면 Swagger 가 이를 "일반 헤더 입력값" 으로 인식해서
UI 에 별도 입력란이 노출됨 — @ApiBearerAuth()/@ApiBasicAuth() 의 자물쇠 버튼과는 별개로 중복 표시됨
```

```typescript
// ❌ Authorization 입력란이 자물쇠 버튼과 별개로 또 노출됨
@Get('introspect')
@ApiBearerAuth()
introspect(@Headers('authorization') token: string) {}
```

```typescript
// ✅ 커스텀 데코레이터로 추출 로직을 숨기기
// common/decorator/authorization.decorator.ts
import { createParamDecorator, ExecutionContext } from '@nestjs/common';

export const Authorization = createParamDecorator(
  (data: unknown, context: ExecutionContext) => {
    const request = context.switchToHttp().getRequest();
    return request.headers['authorization'];
  },
);
```

```typescript
@Get('introspect')
@ApiBearerAuth()
introspect(@Authorization() token: string) {}
// → Swagger UI 에 중복 입력란 없이, 자물쇠 버튼 하나로만 인증 표현
```

```txt
이런 패턴이 필요해지는 경우: Guard 를 거치지 않고 라우트 안에서 토큰 원문을 직접 다뤄야 할 때
(토큰 검증은 보통 Guard 가 처리하므로 — 이 패턴 자체는 자주 쓰이진 않음, 필요할 때만)
커스텀 Param 데코레이터 자체의 일반 패턴은 [[NestJS_Controller]] 참고
```

---

# PartialType import 변경 ⭐️

```txt
Swagger 와 함께 UpdateDto 를 쓰려면 PartialType 을 @nestjs/mapped-types 가 아닌 @nestjs/swagger 에서 import 해야 함
```

```typescript
// ❌ Swagger 문서에 UpdateDto 필드 안 나타남
import { PartialType } from '@nestjs/mapped-types';
export class UpdateMovieDto extends PartialType(CreateMovieDto) {}

// ✅ Swagger 에 필드 정상 표시
import { PartialType } from '@nestjs/swagger';
export class UpdateMovieDto extends PartialType(CreateMovieDto) {}
```

|import 출처|동작|
|---|---|
|`@nestjs/mapped-types`|NestJS 검증 전용 — Swagger 메타데이터 전달 안 함|
|`@nestjs/swagger`|Swagger 메타데이터(`@ApiProperty` 정보)도 같이 상속|

```txt
OmitType / PickType / IntersectionType 도 동일한 규칙 — Swagger 와 함께 쓰면 @nestjs/swagger 에서 import
DTO 자체의 자세한 패턴은 [[NestJS_DTO]] 참고
```

---

# 한눈에

```txt
setVersion: package.json 과 같을 필요 없음 — 개발 중 0.0.1, 외부 정식 공개 시 1.0.0
addBearerAuth(): JWT 기반이면 거의 항상 필요 (Bearer 토큰 인증)
addBasicAuth(): "Authorization: Basic" 헤더를 진짜로 검사하는 엔드포인트가 있을 때만 — 로그인 자체와는 무관
  → 둘 다 필수가 아니라, 내 API 가 실제로 쓰는 방식만 추가

addBearerAuth(options, name) 의 name 은 기본값 'bearer' — 커스텀 이름을 지었다면
  @ApiBearerAuth(name)에도 똑같은 이름을 넘겨야 자물쇠 버튼과 라우트가 정확히 짝지어짐
  (둘 다 이름 생략하면 신경 쓸 필요 없음 — 자동으로 'bearer'로 일치)

@ApiXxxAuth() 데코레이터(엔드포인트 표시) ↔ addXxxAuth()(UI 버튼) 는 항상 짝이 맞아야 함
@Headers('authorization') 직접 쓰면 UI 에 중복 입력란 노출 → 커스텀 데코레이터로 숨기기

PartialType/OmitType/PickType 은 Swagger 쓸 때 @nestjs/mapped-types 대신 @nestjs/swagger 에서 import
OpenAPI CLI Plugin 켜두면 @ApiProperty 거의 안 적어도 됨 — 설명 추가할 때만 명시
```