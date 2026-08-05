---
aliases:
  - API 문서화
  - OpenAPI
  - Swagger
  - JWT 토큰 설정
  - "@ApiBearerAuth()"
tags:
  - NestJS
related:
  - "[[00_NestJS_Ecosystem_HomePage]]"
  - "[[NestJS_DTO]]"
  - "[[NestJS_JwtGuard]]"
  - "[[NextJS_Types]]"
---
# NestJS_Swagger — Swagger · OpenAPI 문서화

> [!info] 
> Swagger(OpenAPI) = 코드에 데코레이터를 달면 API 문서와 테스트 UI를 자동 생성. 
> `/api`에서 UI로 직접 테스트, `/api-json`에서 OpenAPI 스펙으로 프론트 타입 자동 생성(`openapi-typescript`)에 활용.
>  프론트 타입 연결 → [[NextJS_Types]]

---

# Swagger란 — 왜 쓰는가 ⭐️⭐️⭐️⭐️

```txt
API를 만들면 "이 API는 어떻게 쓰는가"를 문서로 남겨야 함
Word/Notion에 수동으로 쓰면:
  코드가 바뀔 때 문서도 수동으로 업데이트해야 함
  까먹으면 문서와 실제 API가 달라짐

Swagger = 코드에 데코레이터를 달면 자동으로 문서 생성
  코드가 곧 문서 — 동기화 문제 없음

두 가지 결과물:
  /api      → Swagger UI (웹 브라우저에서 API 직접 테스트 가능)
  /api-json → OpenAPI JSON 스펙 (기계가 읽는 API 명세)
```

```txt
/api-json을 쓰는 이유:
  프론트엔드에서 openapi-typescript로 이 JSON을 읽어
  TypeScript 타입을 자동 생성 → ApiUser, ApiPost 등
  → 수동으로 타입 정의 안 해도 됨 → [[NextJS_Types]] 방법 1
```

---

# 설치 및 기본 설정 ⭐️⭐️⭐️⭐️

```bash
pnpm add @nestjs/swagger
```

```typescript
// main.ts
import { SwaggerModule, DocumentBuilder } from '@nestjs/swagger';

async function bootstrap() {
  const app = await NestFactory.create(AppModule);

  // Swagger 문서 설정
  const config = new DocumentBuilder()
    .setTitle('Music Community API')       // Swagger UI 제목
    .setDescription('API 설명')            // Swagger UI 설명
    .setVersion('1.0')                     // API 버전
    .addBearerAuth(                        // JWT 인증 설정 (아래 섹션 참고)
      {
        type:         'http',
        scheme:       'bearer',
        bearerFormat: 'JWT',
        description:  'POST /auth/login 후 발급받은 토큰을 입력하세요.',
      },
      'access-token',
    )
    .build();

  const document = SwaggerModule.createDocument(app, config);
  SwaggerModule.setup('api', app, document);
  //                   ↑ 이 경로로 Swagger UI 접근: http://localhost:3000/api

  await app.listen(3000);
}
```

```txt
SwaggerModule.setup('api', app, document):
  'api' → Swagger UI 경로: /api
  → http://localhost:3000/api 에서 브라우저로 접근

  /api-json → OpenAPI JSON 스펙 자동 생성
  → 프론트에서 openapi-typescript로 타입 생성에 사용
```

---

# @ApiTags — 컨트롤러 그룹화 ⭐️⭐️⭐️

```typescript
@ApiTags('posts')        // Swagger UI에서 'posts' 그룹으로 묶임
@Controller('posts')
export class PostsController { ... }
```

```txt
태그가 없으면 모든 API가 한 목록에 나열됨 — 찾기 어려움
@ApiTags로 컨트롤러 단위로 그룹화 → UI에서 폴더처럼 접기/펼치기 가능

리소스 이름을 복수형으로 쓰는 게 관례:
  'posts', 'users', 'rooms', 'auth'
```

---

# @ApiOperation — 엔드포인트 설명 ⭐️⭐️⭐️⭐️

```typescript
@Get(':id')
@ApiOperation({
  summary:     '게시글 단건 조회',          // 짧은 한 줄 설명 (목록에 표시)
  description: '게시글 ID로 단건 조회. 삭제된 게시글은 404 반환.',  // 상세 설명
})
findOne(@Param('id', ParseUUIDPipe) id: string) { ... }
```

```txt
summary    → Swagger UI 목록에서 바로 보이는 짧은 설명
description → 클릭해서 펼쳤을 때 보이는 상세 설명

summary만 써도 충분한 경우가 많음
```

---

# @ApiProperty — DTO 필드 문서화 ⭐️⭐️⭐️⭐️

```typescript
export class CreatePostDto {
  @ApiProperty({
    description: '게시글 제목',
    example:     '오늘의 날씨',      // Swagger UI에서 예시값으로 채워짐
    minLength:   1,
    maxLength:   100,
  })
  @IsString()
  @MinLength(1)
  @MaxLength(100)
  title: string;

  @ApiProperty({
    description: '공개 여부',
    example:     true,
    default:     true,
  })
  @IsBoolean()
  isPublic: boolean;

  @ApiPropertyOptional({           // = @ApiProperty({ required: false })
    description: '태그 목록',
    example:     ['NestJS', 'TypeScript'],
    type:        [String],          // 배열 타입은 [String] 또는 () => [String]
  })
  @IsOptional()
  @IsArray()
  @IsString({ each: true })
  tags?: string[];
}
```

```txt
@ApiProperty가 없으면:
  Swagger UI에서 이 필드가 안 보이거나 unknown으로 표시됨
  /api-json 스펙에도 포함 안 됨 → 프론트 타입 자동 생성 시 누락

required: false vs @ApiPropertyOptional:
  @ApiProperty({ required: false })  = @ApiPropertyOptional()
  optional 필드는 @ApiPropertyOptional을 쓰면 더 간결

example:
  Swagger UI에서 "Try it out" 버튼 클릭 시 자동으로 채워지는 예시값
  실제 검증에는 영향 없음 — 문서용
```

## 자주 쓰는 @ApiProperty 옵션

|옵션|설명|예시|
|---|---|---|
|`description`|필드 설명|`'게시글 제목'`|
|`example`|예시값|`'오늘의 날씨'`|
|`required`|필수 여부 (기본 true)|`false`|
|`nullable`|null 허용 여부|`true`|
|`type`|타입 명시 (배열·중첩 DTO)|`[String]`, `() => UserDto`|
|`enum`|열거형|`['draft', 'published']`|
|`default`|기본값|`true`|
|`minimum`/`maximum`|숫자 범위|`1`, `100`|
|`minLength`/`maxLength`|문자열 길이|`1`, `500`|

## 중첩 DTO · 배열

```typescript
// 중첩 DTO
@ApiProperty({ type: () => UserDto })   // 순환 참조 방지를 위해 () => 사용
owner: UserDto;

// 배열
@ApiProperty({ type: [String] })        // string 배열
tags: string[];

@ApiProperty({ type: () => [PostDto] }) // DTO 배열
posts: PostDto[];
```

---

# @ApiResponse — 응답 타입 지정 ⭐️⭐️⭐️

```typescript
@Post()
@ApiResponse({ status: 201, description: '생성 성공', type: CreatePostDto })
@ApiResponse({ status: 400, description: '유효성 검사 실패' })
@ApiResponse({ status: 401, description: '인증 필요' })
create(@Body() dto: CreatePostDto) { ... }
```

```typescript
// 자주 쓰는 응답 데코레이터 단축형
@ApiOkResponse({ type: PostDto })           // 200
@ApiCreatedResponse({ type: PostDto })      // 201
@ApiBadRequestResponse()                    // 400
@ApiUnauthorizedResponse()                  // 401
@ApiForbiddenResponse()                     // 403
@ApiNotFoundResponse()                      // 404
```

---

# @ApiBearerAuth — JWT 인증 표시 ⭐️⭐️⭐️⭐️

```txt
Swagger UI에서 JWT 토큰으로 인증 후 API 테스트하기 위한 설정
두 곳이 연결되어야 함 — 이름이 반드시 일치

  main.ts:    .addBearerAuth({ ... }, 'access-token')
  Controller: @ApiBearerAuth('access-token')
                               ↑ 같아야 함
```

```typescript
// 컨트롤러 전체에 인증 필요 표시
@ApiTags('posts')
@ApiBearerAuth('access-token')   // 🔒 자물쇠 아이콘 표시
@Controller('posts')
export class PostsController {

  @Get()
  findAll() { ... }   // 이 메서드도 인증 필요 (컨트롤러에서 상속)

  @Post()
  create() { ... }
}

// 일부 메서드만 인증 필요
@ApiTags('posts')
@Controller('posts')
export class PostsController {

  @Get()
  // @ApiBearerAuth 없음 → 공개 API
  findAll() { ... }

  @Post()
  @ApiBearerAuth('access-token')  // 이 메서드만 🔒
  create() { ... }
}
```

```txt
@ApiBearerAuth의 역할:
  Swagger UI에 자물쇠(🔒) 아이콘 표시
  "Authorize" 버튼으로 토큰 입력 후 이 API 테스트 가능
  실제 인증은 JwtAuthGuard가 담당 — @ApiBearerAuth는 문서화만

Swagger UI에서 테스트 흐름:
  1. POST /auth/login → access token 발급
  2. 상단 "Authorize" 버튼 클릭
  3. 발급받은 토큰 입력 (Bearer 접두사 없이)
  4. @ApiBearerAuth API 호출 시 Authorization 헤더 자동 추가
```

---

# 자주 쓰는 데코레이터 한눈에

|데코레이터|위치|역할|
|---|---|---|
|`@ApiTags('name')`|Controller|그룹화|
|`@ApiOperation({ summary })`|Method|엔드포인트 설명|
|`@ApiProperty({ ... })`|DTO 필드|필드 문서화 (필수)|
|`@ApiPropertyOptional({ ... })`|DTO 필드|선택 필드 문서화|
|`@ApiBearerAuth('name')`|Controller/Method|JWT 인증 필요 표시|
|`@ApiResponse({ status, type })`|Method|응답 타입 지정|
|`@ApiParam({ name, description })`|Method|경로 파라미터 설명|
|`@ApiQuery({ name, description })`|Method|쿼리 파라미터 설명|
|`@ApiBody({ type })`|Method|요청 body 타입 명시|
|`@ApiExcludeEndpoint()`|Method|Swagger에서 숨기기|

---

# 프론트와 연결 — 타입 자동 생성

```bash
# /api-json 에서 타입 자동 생성
npx openapi-typescript http://localhost:3000/api-json -o src/types/api.ts
```

```txt
이 명령어가 하는 것:
  NestJS Swagger가 생성한 /api-json (OpenAPI 스펙)을 읽어서
  TypeScript 타입 파일(api.ts)을 자동 생성

  @ApiProperty가 달린 DTO → ApiUser, ApiPost 등의 타입 자동 생성
  수동으로 타입 정의하지 않아도 됨

→ [[NextJS_Types]] 방법 1 참고
```