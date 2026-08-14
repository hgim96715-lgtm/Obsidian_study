---
aliases:
  - class-validator
  - DTO
  - ValidateIf
tags:
  - NestJS
related:
  - "[[00_NestJS_Ecosystem_HomePage]]"
  - "[[NestJS_Swagger]]"
  - "[[NestJS_Pipe]]"
  - "[[NextJS_Types]]"
  - "[[TS_TsConfig]]"
---
# NestJS_DTO — Data Transfer Object

>[!info]
>DTO = "이 엔드포인트는 이런 형태의 데이터를 받는다"는 계약. class-validator 데코레이터가 자동으로 형식·필수·범위를 검증하고, ValidationPipe가 이를 실행한다. `@ValidateIf`로 다른 필드 값에 따라 조건부 검증도 가능. `@ApiProperty({ format: 'uuid' })`·`@ApiProperty({ enum: [...] })` 같은 Swagger 옵션 → [[NestJS_Swagger]] `@ApiProperty` 섹션.

---

# DTO란 — 왜 필요한가 ⭐️⭐️⭐️⭐️

```typescript
// ❌ DTO 없이 — req.body가 any
@Post()
create(@Body() body: any) {
  // body.email이 있는지, 형식이 맞는지 모름
  // body.password가 최소 8자인지 모름
  // 코드에서 일일이 if문으로 검사해야 함
  return this.authService.create(body);
}
```

```typescript
// ✅ DTO 사용 — 자동 검증 + 타입 안전
@Post()
create(@Body() dto: CreateUserDto) {
  // dto.email은 반드시 이메일 형식
  // dto.password는 반드시 8자 이상
  // 틀리면 요청이 여기까지 오지도 않음 → 400 BadRequest 자동 반환
  return this.authService.create(dto);
}
```

```txt
DTO가 해주는 세 가지:
  ① 타입 안전  — dto.email이 string임을 TypeScript가 앎 (any 아님)
  ② 자동 검증  — 형식 틀리면 400 에러 자동 반환 (내가 if문 안 써도 됨)
  ③ 명세 역할  — "이 API는 이런 데이터를 받는다"를 코드로 문서화
```

---

# 요청 DTO ≠ 응답 타입 ⭐️⭐️⭐️⭐️

```txt
DTO라고 하면 보통 "요청 DTO"를 말함

요청 DTO (CreateUserDto, UpdateUserDto, ListUsersQueryDto):
  클라이언트가 서버에 보내는 데이터의 형태
  class-validator 검증이 필요
  반드시 클래스로 만들어야 함 (ValidationPipe가 클래스 기반)

응답 타입:
  서버가 클라이언트에게 보내는 데이터의 형태
  검증 불필요 — 서버가 직접 만들어서 보내는 것
  항상 DTO 클래스를 만들지 않아도 됨
  → Prisma select 결과를 그대로 반환하는 경우가 많음

관계: 요청 DTO 4개 → 응답 필드 15개 → 정상
  → [[NextJS_Types]] "요청 DTO ≠ 응답 타입" 참고
```

---

# 언제 DTO를 만드는가 ⭐️⭐️⭐️⭐️

```txt
클라이언트에서 데이터를 받는 모든 경우:

  @Body() — POST/PATCH의 요청 body
    CreatePostDto, UpdatePostDto, LoginDto

  @Query() — 쿼리 파라미터 (?q=홍길동&status=active)
    ListUsersQueryDto, SearchQueryDto

  @Param() — 경로 파라미터 (:id)
    간단한 경우 ParseUUIDPipe로 대체 가능
    복잡한 검증이 필요하면 DTO

만들지 않아도 되는 경우:
  @Param('id', ParseUUIDPipe) id: string
  → 단순 UUID 검증은 파이프 하나로 충분
```

---

# 기본 구조 ⭐️⭐️⭐️⭐️

```typescript
// create-user.dto.ts
import { IsEmail, IsString, MinLength, IsOptional } from 'class-validator';
import { ApiProperty } from '@nestjs/swagger';

export class CreateUserDto {
  @ApiProperty({ example: 'hong@example.com' })  // Swagger 문서화
  @IsEmail()                                       // 이메일 형식 검증
  email: string;

  @ApiProperty({ example: '비밀번호1234', minLength: 8 })
  @IsString()
  @MinLength(8)
  password: string;

  @ApiProperty({ required: false })
  @IsOptional()  // 없어도 됨
  @IsString()
  nickname?: string;
}
```

```txt
DTO는 일반 클래스지만 두 가지를 결합:
  class-validator 데코레이터 → "이 필드는 이래야 한다"는 규칙
  @ApiProperty → Swagger 문서에 표시될 설명

  class 자체 (interface 아님):
    ValidationPipe가 런타임에 클래스를 보고 검증을 실행
    interface는 런타임에 사라지기 때문에 작동 안 함

email: string — ! 없이도 되는 이유:
  tsconfig.json에 "strictPropertyInitialization": false 설정
  → DTO 프로퍼티를 constructor에서 초기화 안 해도 에러 없음
  → false가 없으면 email!: string 처럼 ! 를 붙여야 함
  → [[TS_TsConfig]] NestJS tsconfig 섹션 참고
```

---

# ValidationPipe 전역 설정 ⭐️⭐️⭐️⭐️

```typescript
// main.ts — 앱 시작 시 전역으로 설정
app.useGlobalPipes(
  new ValidationPipe({
    whitelist:        true,  // DTO에 없는 필드 자동 제거
    forbidNonWhitelisted: false, // true면 DTO에 없는 필드 있으면 400 에러
    transform:        true,  // 타입 자동 변환 (string → number, string → Date)
    transformOptions: { enableImplicitConversion: true },
  }),
);
```

```txt
whitelist: true 가 중요한 이유:
  클라이언트가 DTO에 없는 필드를 보내면 자동으로 제거
  예: { email, password, isAdmin: true } → { email, password }만 통과
  → 악의적인 필드 주입 방지

transform: true 가 중요한 이유:
  HTTP 요청의 모든 값은 기본적으로 문자열
  쿼리 파라미터 ?page=1 → "1" (string)
  transform: true 하면 @Type(() => Number) 선언 시 자동으로 숫자로 변환
```

---

# 자주 쓰는 검증 데코레이터 ⭐️⭐️⭐️⭐️

## 타입 검증

| 데코레이터                                | 검증 내용                  |
| ------------------------------------ | ---------------------- |
| `@IsString()`                        | 문자열                    |
| `@IsNumber()`                        | 숫자 (정수·소수 모두)          |
| `@IsNumber({ maxDecimalPlaces: 1 })` | 소수점 1자리까지만 허용          |
| `@IsBoolean()`                       | 불린                     |
| `@IsInt()`                           | 정수만 (소수 불가)            |
| `@IsArray()`                         | 배열                     |
| `@IsObject()`                        | 객체                     |
| `@IsEmail()`                         | 이메일 형식                 |
| `@IsUrl()`                           | URL 형식                 |
| `@IsUUID()`                          | UUID 형식                |
| `@IsDateString()`                    | ISO 날짜 문자열             |
| `@IsEnum(MyEnum)`                    | TypeScript enum 값 중 하나 |
| `@IsIn([...])`                       | 배열에 포함된 값 중 하나         |
## @IsNumber 옵션 ⭐️⭐️⭐️

```typescript
// 숫자만 검증 (정수·소수 모두 허용)
@IsNumber()
price: number;  // 1, 1.5, 1.234 모두 통과

// 소수점 자릿수 제한
@IsNumber({ maxDecimalPlaces: 1 })
rating: number;  // 4.5 ✅, 4.55 ❌, 4 ✅

// 정수만 (소수 불가)
@IsInt()
count: number;  // 1 ✅, 1.5 ❌

// 옵션 전체
@IsNumber({
  maxDecimalPlaces: 2,   // 소수점 최대 자릿수 (2 = 0.00까지)
  allowNaN:        false, // NaN 허용 여부 (기본 false)
  allowInfinity:   false, // Infinity 허용 여부 (기본 false)
})
```

```txt
@IsNumber() vs @IsInt():
  @IsNumber() → 정수·소수 모두 허용
  @IsInt()    → 정수만 (소수 넣으면 에러)

maxDecimalPlaces 실전:
  평점(별점)  → maxDecimalPlaces: 1  (4.5)
  가격        → maxDecimalPlaces: 0  (@IsInt()와 같음)
  좌표(위도)  → maxDecimalPlaces: 6  (37.123456)
  퍼센트      → maxDecimalPlaces: 2  (99.99%)
```

## @IsIn vs @IsEnum — 열거형 검증 ⭐️⭐️⭐️⭐️

```typescript
// @IsEnum — TypeScript enum 객체가 있을 때
enum RoomStatus { ACTIVE = 'active', CLOSED = 'closed' }

@IsEnum(RoomStatus)
status: RoomStatus;

// @IsIn — 리터럴 유니온 타입일 때 (enum 객체 없음)
@IsIn(['active', 'closed', 'archived'])
status?: 'active' | 'closed' | 'archived';
```

```txt
@IsEnum(RoomStatus):
  TypeScript enum 객체를 인자로 받음
  enum의 value들 중 하나인지 검증
  → enum 객체가 있을 때 사용

@IsIn(['active', 'closed', 'archived']):
  배열을 직접 전달
  배열에 포함된 값인지 검증
  → 리터럴 유니온 타입처럼 enum 객체 없이 문자열 목록만 있을 때 사용

선택 기준:
  TypeScript enum을 정의해서 쓰고 있다면  → @IsEnum(MyEnum)
  'a' | 'b' | 'c' 같은 리터럴 유니온이면 → @IsIn(['a', 'b', 'c'])
```

## @ApiProperty({ enum }) + @IsIn — 세트로 사용 ⭐️⭐️⭐️⭐️

```typescript
export class UpdateAdminReportDto {
  // @ApiProperty({ enum }) = Swagger 문서화 (드롭다운 표시)
  // @IsIn([])             = 실제 값 검증
  // 둘은 역할이 달라서 항상 함께 써야 함
  @ApiProperty({ enum: ['resolved', 'dismissed'] })
  @IsIn(['resolved', 'dismissed'])
  status!: 'resolved' | 'dismissed';
}
```

```txt
@ApiProperty({ enum: [...] }):
  Swagger UI에서 이 필드를 드롭다운으로 표시
  문서화 역할 — 실제 검증 안 함
  → 없으면 Swagger에서 빈 텍스트 입력창만 나옴

@IsIn([...]):
  실제 요청이 왔을 때 값이 배열 안에 있는지 검증
  → 없으면 이상한 값이 들어와도 통과

둘 다 없으면:
  검증도 없고 문서화도 없음 → status: string 으로만 처리

@ApiProperty만 있고 @IsIn 없으면:
  Swagger에서는 드롭다운 보이지만 실제로는 아무 값이나 들어와도 통과

@IsIn만 있고 @ApiProperty 없으면:
  검증은 되지만 Swagger에 enum 힌트가 없음
  → 문서 보는 사람이 어떤 값 넣어야 하는지 모름
```

## 범위 검증

|데코레이터|검증 내용|
|---|---|
|`@MinLength(n)`|최소 문자 수|
|`@MaxLength(n)`|최대 문자 수|
|`@Min(n)`|최솟값 (숫자)|
|`@Max(n)`|최댓값 (숫자)|
|`@Length(min, max)`|문자 수 범위|

## 존재 여부

|데코레이터|동작|
|---|---|
|`@IsOptional()`|없어도 됨 — undefined이면 이후 검증 건너뜀|
|`@IsNotEmpty()`|빈 문자열 불허 (`""` 거부)|

```txt
@IsOptional() 위치 주의:
  @IsOptional()을 붙이면 값이 undefined일 때 이후 데코레이터를 실행 안 함
  → @IsOptional() @IsEmail() → 없으면 OK, 있으면 이메일 형식이어야 함

  @IsOptional()을 안 붙이면 해당 필드는 필수
```

---

# @ValidateIf — 조건부 검증 ⭐️⭐️⭐️⭐️

```txt
특정 조건이 참일 때만 해당 필드를 검증
@IsOptional()은 "없으면 검증 스킵"이지만
@ValidateIf()는 "조건이 맞을 때만 검증 실행" — 더 세밀한 제어
```

```typescript
// 기본 문법
@ValidateIf((obj: DtoClass) => 조건)
@IsString()
fieldName?: string;
// 조건이 true  → @IsString() 실행
// 조건이 false → @IsString() 건너뜀
```

## 다른 필드 값에 따라 필수 여부 결정 ⭐️⭐️⭐️⭐️

```typescript
export class UpdateAdminRoomDto extends PartialType(CreateAdminRoomDto) {
  @ApiPropertyOptional({ description: '방장 통지용 사유 · 닫기·보관 시 필수' })
  @ValidateIf(
    (o: UpdateAdminRoomDto) => o.status === 'closed' || o.status === 'archived',
  )
  @IsString()
  @IsNotEmpty()
  @MaxLength(500)
  reason?: string;
}
```

```txt
이 코드 읽는 법:
  status가 'closed' 또는 'archived' 일 때만 reason 검증 실행
  → status = 'closed'인데 reason 없으면 → 400 BadRequest
  → status = 'active'이면 reason 없어도 통과

@IsOptional()과 @ValidateIf()의 차이:
  @IsOptional()     → undefined이면 검증 스킵 (있으면 반드시 검증)
  @ValidateIf(fn)   → fn이 false이면 검증 스킵 (조건 기반)

  둘을 같이 쓰면:
  @IsOptional()       undefined이면 → 바로 스킵
  @ValidateIf(fn)     fn이 false이면 → 스킵
  → 하나라도 스킵 조건이면 이후 검증 안 함

@ValidateIf 콜백의 인자:
  (obj, value) =>
    obj   = DTO 전체 객체 → 다른 필드 값에 접근 가능
    value = 이 필드의 현재 값

  타입 안전하게 쓰려면:
  (o: UpdateAdminRoomDto) => ... 처럼 o에 타입 명시
```

## 두 필드 중 하나는 반드시 있어야 할 때 ⭐️⭐️⭐️

```typescript
export class ContactDto {
  // email이 없으면 phone이 필수
  @ValidateIf((o: ContactDto) => !o.email)
  @IsString()
  @IsNotEmpty()
  phone?: string;

  // phone이 없으면 email이 필수
  @ValidateIf((o: ContactDto) => !o.phone)
  @IsEmail()
  email?: string;
}
```

```txt
동작:
  { email: 'test@test.com' }  → phone 검증 안 함 → 통과
  { phone: '010-1234-5678' }  → email 검증 안 함 → 통과
  {}                           → 둘 다 검증 실패 → 400
```

```typescript
// list-users-query.dto.ts
import { IsOptional, IsString, IsInt, IsEnum, Min, Max } from 'class-validator';
import { Type } from 'class-transformer';

export class ListUsersQueryDto {
  @IsOptional()
  @IsString()
  q?: string;            // 검색어

  @IsOptional()
  @IsEnum(UserStatus)
  status?: UserStatus;   // 상태 필터

  @IsOptional()
  @IsString()
  cursor?: string;       // 페이지네이션 cursor

  @IsOptional()
  @Type(() => Number)    // 쿼리스트링은 string → Number로 변환 필요
  @IsInt()
  @Min(1)
  @Max(100)
  take?: number = 20;    // 기본값 20
}
```

```typescript
// 컨트롤러에서
@Get()
findAll(@Query() query: ListUsersQueryDto) {
  return this.usersService.findAll(query);
}
```

```txt
@Type(() => Number) 이 필요한 이유:
  URL 쿼리스트링은 전부 문자열: ?take=20 → "20" (string)
  @IsInt()는 숫자를 기대하는데 "20"은 문자열 → 검증 실패
  @Type(() => Number)을 붙이면 transform: true 설정 시 "20" → 20으로 변환

  @IsString()인 q?, cursor?는 이미 문자열이라 @Type 불필요
```

---

# Utility Types — DTO 재사용 ⭐️⭐️⭐️⭐️

```typescript
// 기존 DTO를 바탕으로 새 DTO 만들기
import { PartialType, OmitType, PickType } from '@nestjs/swagger';

export class CreateUserDto {
  @IsEmail()        email:    string;
  @IsString()       password: string;
  @IsString()       nickname: string;
  @IsOptional()     image?:   string;
}

// PartialType — 모든 필드를 optional로 (PATCH에 사용)
export class UpdateUserDto extends PartialType(CreateUserDto) {}
// → { email?: string; password?: string; nickname?: string; image?: string }

// OmitType — 특정 필드 제외
export class CreateWithoutPasswordDto extends OmitType(CreateUserDto, ['password']) {}
// → { email: string; nickname: string; image?: string }

// PickType — 특정 필드만 선택
export class LoginDto extends PickType(CreateUserDto, ['email', 'password']) {}
// → { email: string; password: string }
```

```txt
왜 직접 새 클래스를 만들지 않는가:
  필드 하나 바꿀 때 모든 DTO를 수정해야 하는 문제 방지
  CreateUserDto의 @ApiProperty 설명·예시도 자동으로 상속
  @nestjs/swagger의 PartialType을 쓰면 Swagger에도 반영됨

  @nestjs/swagger vs @nestjs/mapped-types:
    둘 다 PartialType 제공
    Swagger를 쓰면 @nestjs/swagger 것을 import해야 문서에도 반영
```

---

# 중첩 DTO — @Type 필수 ⭐️⭐️⭐️⭐️

```typescript
// 중첩 객체가 있는 경우
export class AddressDto {
  @IsString() city:   string;
  @IsString() street: string;
}

export class CreateUserDto {
  @IsEmail()    email:   string;

  @ValidateNested()         // 중첩 객체도 검증하겠다
  @Type(() => AddressDto)   // class-transformer에게 타입 알려주기
  address: AddressDto;
}
```

```txt
@Type(() => AddressDto) 이 필요한 이유:
  JavaScript는 런타임에 타입 정보를 잃어버림
  class-transformer가 plain object를 AddressDto 인스턴스로 변환하려면
  어떤 클래스인지 런타임에 알아야 함 → @Type()으로 명시

  @ValidateNested() 없으면:
    address 자체는 객체인지 검증하지만 내부 필드는 검증 안 함
    address: { city: 123, street: null } 통과됨

  @ValidateNested() + @Type() 세트로:
    address 안의 city, street 까지 재귀적으로 검증
```

---

# @ApiProperty — Swagger 문서화 ⭐️⭐️⭐️

```typescript
export class CreatePostDto {
  @ApiProperty({
    description: '게시글 제목',
    example:     '오늘의 날씨',
    minLength:   1,
    maxLength:   100,
  })
  @IsString()
  @MinLength(1)
  @MaxLength(100)
  title: string;

  @ApiProperty({ required: false, description: '태그 목록' })
  @IsOptional()
  @IsArray()
  @IsString({ each: true })  // 배열의 각 요소가 string인지
  tags?: string[];

  @ApiProperty({ enum: PostStatus, description: '공개 상태' })
  @IsEnum(PostStatus)
  status: PostStatus;
}
```

```txt
@ApiProperty가 없으면:
  Swagger UI에서 이 필드가 보이지 않거나 unknown으로 표시됨

required: false:
  Swagger에서 선택 필드로 표시
  실제 검증은 @IsOptional()이 담당 (@ApiProperty는 문서만)

enum 배열의 각 요소 검증:
  @IsString({ each: true }) → tags 배열의 각 항목이 string인지 체크
  @IsInt({ each: true }) → numbers 배열의 각 항목이 정수인지 체크
```

---

# 자주 만나는 에러

| 에러                               | 원인                          | 해결                                                   |
| -------------------------------- | --------------------------- | ---------------------------------------------------- |
| 숫자 필드인데 `@IsNumber()` 실패         | 쿼리 파라미터가 string으로 들어옴       | `@Type(() => Number)` 추가 + `transform: true` 설정      |
| `@ValidateNested()` 해도 내부 검증 안 됨 | `@Type()` 누락                | `@Type(() => 중첩Dto)` 함께 추가                           |
| Optional 필드인데 검증 실패              | `@IsOptional()` 누락 또는 순서 문제 | `@IsOptional()`을 다른 데코레이터보다 위에 선언                    |
| whitelist 설정인데 필드가 사라짐           | DTO에 선언 안 된 필드              | DTO에 `@ApiProperty()` + 해당 필드 추가                     |
| `class-validator` 데코레이터가 작동 안 함  | ValidationPipe 전역 설정 누락     | `main.ts`에 `useGlobalPipes(new ValidationPipe())` 추가 |

---
# Response DTO — 응답 형태 문서화 ⭐️⭐️⭐️

```typescript
// auth/dto/auth-response.dto.ts
import { ApiProperty } from '@nestjs/swagger';

// 중첩 타입 먼저 선언
export class AuthUserDto {
  @ApiProperty()
  id: string;

  @ApiProperty({ example: 'test@example.com' })
  email: string;
}

export class AuthResponseDto {
  @ApiProperty()
  accessToken: string;

  @ApiProperty({ type: AuthUserDto })  // 중첩 DTO
  user: AuthUserDto;
}
```

```typescript
// Controller에서 응답 타입 명시
@ApiCreatedResponse({ type: AuthResponseDto })  // 201
@Post('register')
register(@Body() dto: RegisterDto) { ... }

@ApiOkResponse({ type: AuthResponseDto })       // 200
@Post('login')
login(@Body() dto: LoginDto) { ... }
```


```txt
Request DTO vs Response DTO:
  Request DTO → 입력 검증 (class-validator 데코레이터 필요)
  Response DTO → 응답 문서화 (@ApiProperty만 있으면 됨)
  → class-validator 불필요, ValidationPipe와 관계없음

파일 위치:
  auth/dto/auth-response.dto.ts
  같은 도메인 dto/ 폴더 안에 위치

Swagger에서 자동으로:
  @ApiCreatedResponse({ type: AuthResponseDto })
  → Swagger UI에 응답 Schema가 표시됨
  → API 사용자가 어떤 값이 오는지 확인 가능
  자세한 설명 → [[NestJS_Swagger]] @ApiResponse 섹션
```