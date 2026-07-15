---
aliases: [API 문서화, OpenAPI, Swagger]
tags:
  - NestJS
related:
  - "[[00_NestJS_Ecosystem_HomePage]]"
  - "[[NestJS_DTO]]"
  - "[[NestJS_JwtGuard]]"
---
# NestJS_Swagger — Swagger API 문서 자동 생성

> [!info] 
> `@nestjs/swagger`로 코드에 데코레이터를 추가하면 `/api` 에서 인터랙티브 API 문서가 자동 생성된다. 
> 별도 문서 작성 없이 실제 코드가 문서의 단일 출처(single source of truth)가 된다.

---

# 설치 & 설정 ⭐️⭐️⭐️

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
    .setVersion('1.0')
    .addBearerAuth()   // JWT Bearer 토큰 인증 버튼 추가
    .build();

  const document = SwaggerModule.createDocument(app, config);
  SwaggerModule.setup('api', app, document);  // /api 에서 접근

  await app.listen(3000);
}
```

```txt
개발 환경에서만 활성화하고 싶다면:
if (process.env.NODE_ENV !== 'production') {
  SwaggerModule.setup('api', app, document);
}
```

---

# DTO 필드 문서화 ⭐️⭐️⭐️⭐️

## @ApiProperty vs @ApiPropertyOptional

```typescript
import { ApiProperty, ApiPropertyOptional, ApiHideProperty } from '@nestjs/swagger';

export class CreatePostDto {
  @ApiProperty({
    description: '게시글 제목',
    example:     '안녕하세요',
    minLength:   2,
    maxLength:   100,
  })
  @IsString()
  title: string;

  @ApiPropertyOptional({      // 선택 필드 — Swagger에서 required: false 로 표시
    description: '게시글 내용',
    example:     '본문 내용입니다.',
  })
  @IsOptional()
  @IsString()
  content?: string;

  @ApiHideProperty()          // Swagger 문서에서 완전히 숨김
  internalFlag?: boolean;
}
```

|데코레이터|Swagger 표시|언제|
|---|---|---|
|`@ApiProperty()`|required: true (필수)|항상 있어야 하는 필드|
|`@ApiPropertyOptional()`|required: false (선택)|`@IsOptional()`과 세트|
|`@ApiHideProperty()`|문서에서 숨김|비밀번호 해시, 내부 상태 등|

```txt
@ApiPropertyOptional({ description: '...' })
=
@ApiProperty({ required: false, description: '...' })
→ 완전히 동일한 축약 표현
```

---

# type: [String] — 배열 타입 표현 ⭐️⭐️⭐️⭐️

```typescript
// Swagger에서 배열을 표현하는 방법
@ApiPropertyOptional({
  type:    [String],              // string[] — 가장 흔한 방법
  example: ['react', 'nestjs'],
})
tags?: string[];

@ApiProperty({
  type:    [Number],              // number[]
})
scores?: number[];

@ApiProperty({
  type: [CreateCommentDto],       // 객체 배열
})
comments?: CreateCommentDto[];

// isArray 플래그 방식 (같은 결과)
@ApiProperty({ type: String, isArray: true })
names?: string[];
```

```txt
왜 type: [String] 이고 type: [string]이 아닌가:

  string (소문자) = TypeScript 타입
    컴파일 후 JS에서 사라짐 → Swagger가 런타임에 읽을 수 없음

  String (대문자) = JavaScript 내장 생성자 함수
    런타임에 존재 → Swagger가 읽을 수 있음

  [String] = "String 생성자를 담은 배열"
    → Swagger가 "이건 배열이고, 요소 타입은 String" 으로 해석

  Swagger 데코레이터에서 타입 지정 시 항상 대문자:
    String / Number / Boolean / Date / 클래스 이름
```

---

# enum 타입 ⭐️⭐️⭐️

```typescript
export enum UserRole { ADMIN = 'admin', USER = 'user' }

export class CreateUserDto {
  @ApiProperty({
    enum:     UserRole,
    enumName: 'UserRole',       // Swagger UI에서 표시할 enum 이름
    example:  UserRole.USER,
  })
  @IsEnum(UserRole)
  role: UserRole;
}
```

```txt
enumName을 지정하는 이유:
  없으면 Swagger 스키마에 enum 값이 inline으로 반복됨
  있으면 $ref로 참조해서 한 곳에서 관리 가능
```

---

# 컨트롤러 — 응답 타입 & 에러 코드 ⭐️⭐️⭐️⭐️

```typescript
import {
  ApiTags,
  ApiBearerAuth,
  ApiOkResponse,
  ApiCreatedResponse,
  ApiBadRequestResponse,
  ApiUnauthorizedResponse,
  ApiNotFoundResponse,
} from '@nestjs/swagger';

@ApiTags('users')             // Swagger UI에서 그룹핑
@ApiBearerAuth()              // 이 컨트롤러 전체에 JWT 인증 필요
@Controller('users')
export class UserController {

  @ApiCreatedResponse({ type: UserResponseDto })
  @ApiBadRequestResponse({ description: '유효성 검사 실패' })
  @Post()
  create(@Body() dto: CreateUserDto) {}

  @ApiOkResponse({ type: UserResponseDto })      // 단건 응답
  @ApiNotFoundResponse({ description: '사용자 없음' })
  @Get(':id')
  findOne(@Param('id') id: string) {}

  @ApiOkResponse({ type: [UserResponseDto] })    // 배열 응답
  @Get()
  findAll() {}

  @ApiOkResponse({ type: UserResponseDto })
  @ApiUnauthorizedResponse({ description: '인증 필요' })
  @ApiBearerAuth()            // 이 라우트만 인증 필요
  @Get('me')
  getMe() {}
}
```

## 자주 쓰는 응답 데코레이터

|데코레이터|HTTP 상태|
|---|---|
|`@ApiOkResponse`|200|
|`@ApiCreatedResponse`|201|
|`@ApiNoContentResponse`|204|
|`@ApiBadRequestResponse`|400|
|`@ApiUnauthorizedResponse`|401|
|`@ApiForbiddenResponse`|403|
|`@ApiNotFoundResponse`|404|
|`@ApiConflictResponse`|409|

---

# Response DTO 문서화 ⭐️⭐️⭐️

```typescript
export class UserResponseDto {
  @ApiProperty({ example: '550e8400-e29b-41d4-a716-446655440000' })
  id: string;

  @ApiProperty({ example: 'user@example.com' })
  email: string;

  @ApiProperty({ example: '닉네임' })
  nickname: string;

  @ApiPropertyOptional({ example: '자기 소개' })
  bio?: string;

  @ApiProperty()
  createdAt: Date;

  @ApiHideProperty()
  passwordHash?: string;   // 응답에 포함되지 않도록 숨김
}
```

---

# PartialType 등에서 import 위치 ⭐️⭐️⭐️

```typescript
// Swagger를 쓰는 프로젝트 → @nestjs/swagger에서 import
// → 상속 관계가 Swagger 문서에도 올바르게 반영됨
import { PartialType, OmitType, PickType } from '@nestjs/swagger';

// Swagger를 쓰지 않는 경우 → @nestjs/mapped-types에서 import
import { PartialType } from '@nestjs/mapped-types';
```

```txt
⚠️ @nestjs/swagger와 @nestjs/mapped-types를 동시에 설치하고 혼용하면
   두 패키지가 서로 충돌해서 예상치 못한 동작이 생길 수 있음
   → 하나만 선택해서 사용
```

---

# 한눈에

```txt
설정:
  DocumentBuilder → SwaggerModule.createDocument → SwaggerModule.setup('api', ...)
  .addBearerAuth() → Swagger UI에 JWT 인증 버튼 추가

DTO 필드:
  @ApiProperty()          필수 (required: true)
  @ApiPropertyOptional()  선택 (required: false) — @IsOptional()과 세트
  @ApiHideProperty()      문서에서 숨김

배열 타입:
  type: [String]   string[]
  type: [Number]   number[]
  type: [MyDto]    MyDto[]
  대문자 생성자 함수를 배열에 넣는 것 — 소문자 string/number는 런타임에 없음

컨트롤러:
  @ApiTags('그룹명')      Swagger UI에서 라우트 그룹핑
  @ApiBearerAuth()       JWT 인증 필요 표시
  @ApiOkResponse({ type: Dto })         단건 응답
  @ApiOkResponse({ type: [Dto] })       배열 응답

PartialType / OmitType / PickType:
  Swagger 쓰면 @nestjs/swagger에서 import (문서에 상속 반영됨)

유효성 검사 로직 → [[NestJS_DTO]]
```