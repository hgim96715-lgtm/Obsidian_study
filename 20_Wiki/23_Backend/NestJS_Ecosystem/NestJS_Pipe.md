---
aliases:
  - Pipe
  - 변환
  - 검증
  - 내장 파이프
  - ParseIntPipe
  - ParseUUIDPipe
  - ParseBoolPipe
  - ParseEnumPipe
  - DefaultValuePipe
tags:
  - NestJS
related:
  - "[[00_NestJS_Ecosystem_HomePage]]"
  - "[[NestJS_DTO]]"
  - "[[NestJS_Controller]]"
  - "[[NestJS_Concept]]"
---
# NestJS_Pipe — 파이프

> [!info] 
> 파이프 = 컨트롤러 핸들러가 실행되기 직전에 요청 데이터를 가공하는 클래스. 
> 두 가지 역할: **변환**(string → number, string → UUID)과 **검증**(형식 틀리면 400 에러). 
> DTO 검증은 ValidationPipe, 경로 파라미터 변환은 ParseUUIDPipe·ParseIntPipe가 담당한다. 
> 요청 처리 순서 → [[NestJS_Concept]], 컨트롤러에서 사용 → [[NestJS_Controller]], DTO → [[NestJS_DTO]]

---

# 파이프란 — 두 가지 역할 ⭐️⭐️⭐️⭐️

```txt
컨트롤러가 데이터를 받기 전에 파이프가 먼저 처리함

변환 (Transform):
  URL에서 오는 데이터는 기본적으로 전부 문자열
  ?page=1 → "1" (string)
  :id = "abc-123" (string)
  파이프가 number, UUID 등 원하는 타입으로 변환

검증 (Validate):
  변환 후 형식이 맞는지 확인
  UUID 형식이 아니면 → 400 BadRequest 자동 반환
  핸들러까지 잘못된 데이터가 도달하지 않음
```

## 요청 처리 순서에서의 위치

```txt
클라이언트 요청
      ↓
  미들웨어
      ↓
  Guard (인증·인가)
      ↓
  Interceptor (전처리)
      ↓
  Pipe  ← 여기서 변환·검증
      ↓
  핸들러 (Controller 메서드) ← 파이프 통과한 깨끗한 데이터만 도달
      ↓
  Interceptor (후처리)
      ↓
  응답
```

```txt
파이프가 Guard보다 나중에 실행되는 이유:
  Guard가 먼저 "이 요청이 허용되는가" 판단
  허용된 요청만 파이프에서 데이터 변환·검증
  순서가 바뀌면 인증 안 된 요청에도 DB 쿼리 등이 실행될 수 있음
```

---

# 내장 파이프 — Built-in Pipes ⭐️⭐️⭐️⭐️

## ParseIntPipe — string → number

```typescript
// URL: /posts/42
@Get(':id')
findOne(@Param('id', ParseIntPipe) id: number) {
  // id는 이미 number 타입 (42)
  // "42" → 42 변환 완료
  // "abc" → 400 BadRequest 자동 반환
  return this.postsService.findOne(id);
}
```

## ParseUUIDPipe — UUID 형식 검증

```typescript
// URL: /users/550e8400-e29b-41d4-a716-446655440000
@Get(':id')
findOne(@Param('id', ParseUUIDPipe) id: string) {
  // UUID 형식 맞으면 통과
  // "invalid-id" → 400 BadRequest 자동 반환
  return this.usersService.findOne(id);
}
```

```txt
ParseUUIDPipe를 쓰는 이유:
  UUID 형식이 아닌 값으로 DB 쿼리를 날리면 Prisma가 에러를 던짐
  파이프에서 미리 걸러내면 더 깔끔한 400 에러 반환
  서비스 레이어까지 잘못된 값이 도달하지 않음
```

## ParseBoolPipe — string → boolean

```typescript
// URL: /posts?visible=true
@Get()
findAll(@Query('visible', ParseBoolPipe) visible: boolean) {
  // "true" → true, "false" → false
  // "yes" → 400 BadRequest
  return this.postsService.findAll(visible);
}
```

## ParseEnumPipe — enum 값 검증

```typescript
enum PostStatus { DRAFT = 'draft', PUBLISHED = 'published' }

// URL: /posts?status=draft
@Get()
findAll(@Query('status', new ParseEnumPipe(PostStatus)) status: PostStatus) {
  // "draft" → PostStatus.DRAFT
  // "invalid" → 400 BadRequest
}
```

## DefaultValuePipe — 기본값 설정

```typescript
// URL: /posts 또는 /posts?page=2
@Get()
findAll(
  @Query('page',  new DefaultValuePipe(1),  ParseIntPipe) page: number,
  @Query('limit', new DefaultValuePipe(20), ParseIntPipe) limit: number,
) {
  // page가 없으면 1, limit이 없으면 20 사용
}
```

```txt
DefaultValuePipe 위치:
  ParseIntPipe 앞에 와야 함
  undefined → DefaultValue 적용 → ParseInt로 변환하는 순서

  여러 파이프를 배열로 넘기면 순서대로 실행됨:
  [new DefaultValuePipe(1), ParseIntPipe]
  ① undefined → 1 (string)
  ② "1" → 1 (number)
```

## 내장 파이프 전체 목록

|파이프|역할|
|---|---|
|`ValidationPipe`|DTO class-validator 검증|
|`ParseIntPipe`|string → number (정수)|
|`ParseFloatPipe`|string → number (부동소수)|
|`ParseBoolPipe`|string → boolean|
|`ParseUUIDPipe`|UUID 형식 검증|
|`ParseArrayPipe`|string → array|
|`ParseEnumPipe`|enum 값 검증|
|`DefaultValuePipe`|값이 없을 때 기본값|

---

# 파이프 적용 위치 ⭐️⭐️⭐️⭐️

## ① 파라미터에 직접 (가장 많이 씀)

```typescript
// @Param, @Query, @Body 뒤에 파이프 추가
@Get(':id')
findOne(
  @Param('id', ParseUUIDPipe)             id: string,    // UUID 검증
  @Query('take', new DefaultValuePipe(20), ParseIntPipe) take: number,
  @Body() dto: CreatePostDto,              // ValidationPipe가 전역으로 처리
) { ... }
```

## ② 메서드에 적용 — @UsePipes()

```typescript
@Post()
@UsePipes(new ValidationPipe({ whitelist: true }))
create(@Body() dto: CreatePostDto) { ... }
// 이 메서드에만 적용
```

## ③ 전역 적용 — main.ts (권장)

```typescript
// main.ts
app.useGlobalPipes(
  new ValidationPipe({
    whitelist:              true,   // DTO에 없는 필드 자동 제거
    transform:              true,   // 타입 자동 변환
    transformOptions:       { enableImplicitConversion: true },
    forbidNonWhitelisted:   false,  // true면 DTO에 없는 필드 있으면 400
  }),
);
```

```txt
전역 ValidationPipe를 설정하면:
  모든 @Body()가 자동으로 DTO 검증을 받음
  개별 컨트롤러에 @UsePipes() 붙이지 않아도 됨

개별 파이프(ParseUUIDPipe 등)는 전역 설정과 별도로 파라미터에 직접 적용
```

---

# ValidationPipe 옵션 상세 ⭐️⭐️⭐️

```typescript
new ValidationPipe({
  whitelist: true,
  // DTO에 정의되지 않은 필드를 자동으로 제거
  // { email, password, isAdmin: true } → { email, password }
  // 악의적인 필드 주입 방어

  forbidNonWhitelisted: false,
  // true로 설정 시: DTO에 없는 필드가 오면 400 에러
  // false(기본): 조용히 제거만 함

  transform: true,
  // HTTP 요청의 string 값을 DTO 타입으로 자동 변환
  // @Type(() => Number) 선언 시 "20" → 20 자동 변환

  transformOptions: { enableImplicitConversion: true },
  // @Type() 없이도 TypeScript 타입 정보 기반으로 자동 변환 시도
  // number 타입 필드면 string → number 자동 시도

  disableErrorMessages: false,
  // true로 설정 시 에러 메시지 숨김 (운영 환경 보안)

  exceptionFactory: (errors) => new BadRequestException(errors),
  // 검증 실패 시 던질 예외 커스터마이즈
})
```

---

# 커스텀 파이프 ⭐️⭐️⭐️

```typescript
// PipeTransform 인터페이스를 구현
import { PipeTransform, Injectable, ArgumentMetadata, BadRequestException } from '@nestjs/common';

@Injectable()
export class PositiveIntPipe implements PipeTransform {
  transform(value: string, metadata: ArgumentMetadata): number {
    const num = parseInt(value, 10);

    if (isNaN(num)) {
      throw new BadRequestException(`${metadata.data}는 숫자여야 합니다.`);
    }
    if (num <= 0) {
      throw new BadRequestException(`${metadata.data}는 양수여야 합니다.`);
    }

    return num;
  }
}

// 사용
@Get(':count')
getItems(@Param('count', PositiveIntPipe) count: number) { ... }
```

```txt
PipeTransform 인터페이스:
  transform(value, metadata): 변환된 값 또는 throw

  value    = 현재 파라미터의 값
  metadata = { type: 'param'|'query'|'body', data: '파라미터이름' }

  반환값이 핸들러에 전달될 값
  예외를 던지면 그 예외가 클라이언트에게 응답됨

언제 커스텀 파이프가 필요한가:
  내장 파이프로 표현할 수 없는 변환·검증 로직
  여러 곳에서 반복되는 변환 로직을 파이프로 추출
  도메인 특화 검증 (예: 특정 형식의 코드 검증)
```

---

# 자주 만나는 에러

| 에러                                               | 원인                                       | 해결                                           |
| ------------------------------------------------ | ---------------------------------------- | -------------------------------------------- |
| `Validation failed (numeric string is expected)` | ParseIntPipe인데 문자열이 옴                    | URL에 숫자가 오는지 확인                              |
| `Validation failed (uuid is expected)`           | ParseUUIDPipe인데 UUID 형식 아님               | 올바른 UUID인지 확인                                |
| `@Body()` 검증이 안 됨                                | ValidationPipe 전역 설정 누락                  | `main.ts`에 `useGlobalPipes` 추가               |
| DTO에 없는 필드가 서비스까지 옴                              | `whitelist: true` 설정 안 함                 | ValidationPipe에 `whitelist: true` 추가         |
| 숫자 타입인데 string으로 받음                              | `transform: true` 설정 안 함 또는 `@Type()` 누락 | `transform: true` + `@Type(() => Number)` 추가 |