---
aliases:
  - HttpCode
  - HttpStatus
  - Redirect
  - ExceptionFilter
  - HttpException
tags:
  - NestJS
related:
  - "[[00_NestJS_Ecosystem_HomePage]]"
  - "[[NestJS_Controller]]"
  - "[[NestJS_Swagger]]"
  - "[[NestJS_Prisma]]"
---

# NestJS_Response — 상태코드, 응답, 예외 처리

> [!info] 
> 성공 응답(상태코드/본문)과 실패 응답(예외)은 결국 같은 주제 
> — "이 요청에 어떤 상태코드와 어떤 body로 답할 것인가"다. 
> `@HttpCode(204)`처럼 "본문 없음"을 의미하는 상태코드를 쓰면, 핸들러가 무엇을 `return`하든 그 값은 실제 응답에 실리지 않는다.

```
이 노트와 NestJS_Controller의 역할 분담:
  NestJS_Controller   요청을 "받는" 쪽 — @Param/@Query/@Body/@Headers, @Req()/@Res() 자체의 동작
  이 노트(NestJS_Response)   응답을 "내보내는" 쪽 — 상태코드, 본문 유무, 예외, Swagger 문서와의 동기화
```

---

# NestJS 기본 응답 — 메서드별 기본 상태코드 ⭐️⭐️

```
아무것도 지정하지 않으면 NestJS가 HTTP 메서드에 따라 기본 상태코드를 자동으로 정함
```

|메서드|기본 상태코드|
|---|---|
|`@Post()`|201 Created|
|`@Get()` / `@Put()` / `@Patch()` / `@Delete()`|200 OK|

```
204처럼 200/201이 아닌 다른 코드가 필요하면 직접 지정해야 함 — 자동으로 안 바뀜
```

---

# @HttpCode — 기본값을 직접 바꾸기 ⭐️⭐️⭐️

```typescript
import { HttpCode, HttpStatus } from '@nestjs/common';

@Delete(':id')
@HttpCode(HttpStatus.NO_CONTENT) // 204 — HttpStatus.NO_CONTENT === 204, 숫자로 그냥 @HttpCode(204)도 동일
async remove(@Param('id', ParseIntPipe) id: number) {
  await this.service.remove(id);
}
```

```
HttpStatus는 자주 쓰는 상태코드를 이름으로 쓸 수 있게 해주는 enum일 뿐 — 숫자 그대로 써도 동작은 같음
HttpStatus.NO_CONTENT 쪼이 "이게 무슨 코드인지" 코드만 보고 바로 알 수 있어서 보통 이쪽을 선호함
```

---

# ⚠️ 204 No Content — return 값이 사라지는 이유 ⭐️⭐️⭐️⭐️

```
204는 "본문이 없다"는 뜻 자체임 (HTTP 스펙상 204 응답은 body를 보내면 안 됨)
→ @HttpCode(204)를 붙인 라우트는, 핸들러가 무엇을 return하든 그 값이 실제 응답 body에 실리지 않음
   이건 NestJS만의 특이 동작이 아니라, 204라는 상태코드 자체의 정의를 지키는 것임
```

```typescript
// POST — 기본 201, body가 정상적으로 응답에 실림
@Post(':id/like')
like(@Param('id', ParseIntPipe) id: number) {
  return this.service.toggleLike(id); // 예: { liked: false } → 응답 body에 그대로 보임
}

// DELETE — 204를 직접 지정, body는 절대 안 보임
@Delete(':id/like')
@HttpCode(HttpStatus.NO_CONTENT)
unlike(@Param('id', ParseIntPipe) id: number) {
  return this.service.toggleLike(id); // { liked: false } 를 return해도 응답 body는 비어있음
}
```

```
"POST에서는 보이던 { liked: false }가 DELETE에서는 안 보인다"의 진짜 원인은
POST와 DELETE의 차이가 아니라 — @HttpCode(204)가 붙어있는가의 차이임
같은 toggleLike 서비스 메서드를 쓰더라도, 204가 붙은 라우트는 항상 body가 비워짐

→ 204를 쓰는 라우트는 return 값 자체가 의미 없어짐 — 차라리 return을 안 하거나,
  return하더라도 "내부적으로 어디까지 처리됐는지 디버깅용" 정도로만 취급하는 게 안전함
  (응답으로 진짜 뭔가를 내려줘야 한다면 204가 아니라 200을 써야 함)
```

---

# @Redirect() — 리다이렉트 응답 ⭐️

```typescript
import { Redirect } from '@nestjs/common';

@Get('docs')
@Redirect('https://docs.example.com', 302) // 두 번째 인자 생략하면 기본 302
goToDocs() {}
```

```
핸들러 안에서 동적으로 목적지를 바꾸고 싶다면, 객체를 return해서 데코레이터의 인자를 덮어쓸 수 있음
```

```typescript
@Get(':id')
@Redirect() // 기본값만 잡아두고, 실제 목적지는 런타임에 결정
findOne(@Param('id') id: string) {
  if (id === 'legacy') {
    return { url: 'https://old.example.com', statusCode: 301 };
  }
  return { url: `/posts/${id}` };
}
```

---

# Response 객체 import할 때 주의점 — express Response vs 전역 Response ⭐️⭐️⭐️

```typescript
// ⚠️ 이름은 같지만 완전히 다른 두 개의 Response가 존재함
import type { Response } from 'express'; // Express의 응답 객체 — res.status().json() 등
// vs
// 전역 Response (Node 18+ / 브라우저 DOM에 내장된 fetch API의 Response) — res.json()이 "받은 응답을 파싱"하는 의미
```

```
Node.js 18부터 fetch/Request/Response가 전역으로 내장되면서,
"Response"라는 이름이 이미 전역에 존재하는 상태가 됨 (fetch가 돌려주는 응답 객체)
→ express에서 명시적으로 import 안 하면, TS가 그 전역 Response 타입으로 잘못 해석할 수 있음
   둘은 이름만 같고 메서드/의미가 전혀 다름(express Response.json(data)는 "보낼 데이터",
   전역 Response.json()은 "받은 body를 파싱해서 꺼내기")

→ @Res() res: Response 를 쓸 거면 반드시 import type { Response } from 'express' 를
  파일 위에 명시할 것 — 빠뜨리면 res.status(200)... 같은 코드에서 타입 에러가 나거나,
  최악의 경우 엉뚱한 타입으로 조용히 통과해버릴 수 있음

import type을 쓰는 이유:
  Response는 여기서 "타입 선언"으로만 쓰임(직접 new Response()로 만들지 않음, NestJS가 인스턴스를 줌)
  → 런타임에 실제로 필요 없는 import라서, type-only import로 명시하면 컴파일 시 완전히 제거됨
```

---

# Swagger와 실제 상태코드 맞추기 ⭐️⭐️

```
@ApiResponse({ status: 200 }) 같은 Swagger 데코레이터는 순수 문서(설명)일 뿐 —
실제 응답 상태코드를 강제하거나 바꾸지 않음. 즉 코드와 문서가 서로 따로 놀 수 있음
```

```typescript
// ❌ 문서랑 실제 동작이 어긋난 예
@ApiResponse({ status: 200, description: '성공' })
@HttpCode(HttpStatus.NO_CONTENT) // 실제로는 204가 나가는데 문서엔 200이라고 적혀있음
@Delete(':id')
remove() {}

// ✅ 실제 상태코드와 문서를 맞춤
@ApiResponse({ status: 204, description: '성공 (본문 없음)' })
@HttpCode(HttpStatus.NO_CONTENT)
@Delete(':id')
remove() {}
```

```
Swagger 데코레이터 문법 자체(addBearerAuth 등)는 [[NestJS_Swagger]] 참고
이 노트에서 강조하는 건 "@HttpCode를 바꿨으면 @ApiResponse도 같이 바꿔야 한다"는 동기화 책임뿐
```

---

# 예외 처리 — 실패 응답 만들기 ⭐️⭐️⭐️

```typescript
import { NotFoundException, UnauthorizedException } from '@nestjs/common';

// ❌ Error → 500 Internal Server Error
throw new Error('존재하지 않는 ID');

// ✅ NestJS 예외 → 올바른 상태코드 + 자동 응답 형식
throw new NotFoundException(`${id} 영화 없음`);
```

```json
{
  "message": "3 영화 없음",
  "error": "Not Found",
  "statusCode": 404
}
```

```
예외도 결국 "응답"의 한 종류 — throw한 예외를 NestJS가 가로채서 위와 같은 JSON 응답으로 변환해줌
별도 try-catch 없이도 자동으로 이렇게 됨 (Exception Filter가 내부적으로 처리)
```

## 내장 HttpException — 상황별 선택 가이드 ⭐️⭐️⭐️

```typescript
import {
  BadRequestException,            // 400
  UnauthorizedException,          // 401
  ForbiddenException,             // 403
  NotFoundException,              // 404
  ConflictException,              // 409
  UnprocessableEntityException,   // 422
  InternalServerErrorException,   // 500
} from '@nestjs/common';
```

|예외 클래스|상태코드|언제|
|---|---|---|
|`BadRequestException`|400|요청 형식 자체가 잘못됨 (타입 오류, 필수 필드 누락) — DTO 유효성 검사 실패|
|`UnauthorizedException`|401|"누구인지 모름" — 토큰 없음/만료/로그인 안 됨|
|`ForbiddenException`|403|"누구인지는 알지만 권한 없음" — 다시 로그인해도 해결 안 됨|
|`NotFoundException`|404|리소스 없음|
|`ConflictException`|409|중복 데이터|
|`UnprocessableEntityException`|422|형식은 맞지만 비즈니스 규칙 위반|
|`InternalServerErrorException`|500|서버 에러|

## 400 vs 422 — 형식 오류 vs 규칙 위반 ⭐️

```
BadRequestException (400):
  요청 형식 자체가 잘못됨 — 타입 오류 / 필수 필드 누락 / 이메일 형식 아님
  → 보통 DTO 유효성 검사(ValidationPipe) 단계에서 자동으로 발생

UnprocessableEntityException (422):
  형식은 맞는데 비즈니스 규칙 위반 — 값 자체는 유효하지만 논리적으로 처리 불가
  → 서비스 레이어에서 직접 throw

예: title 필드 없음 → 400 / 시작일 > 종료일 → 422 / 이미 예약된 시간대 → 422
```

## 401 vs 403 — 인증 vs 인가 ⭐️

```
401 Unauthorized: "누구인지 모름" — 토큰 없음/만료/로그인 안 됨 → 다시 로그인하면 해결됨
403 Forbidden:    "누구인지는 알지만 권한 없음" — 로그인은 됐지만 접근 불가 → 다시 로그인해도 안 풀림

예: 토큰 없이 요청 → 401 / 일반 유저가 admin API 접근 → 403
```

## HttpException — 커스텀 상태코드가 필요할 때

```typescript
import { HttpException, HttpStatus } from '@nestjs/common';

throw new HttpException('커스텀 메시지', HttpStatus.I_AM_A_TEAPOT); // 418

throw new HttpException(
  { statusCode: 418, message: '커스텀 에러', custom: '추가 데이터' },
  418,
);
```

---

# Exception Filter — 응답 형식을 직접 커스터마이징 ⭐️⭐️

```
왜 쓰나: 기본 에러 응답 형식을 바꾸고 싶을 때, 에러 로그를 남기고 싶을 때,
timestamp/path 등 추가 정보를 넣고 싶을 때
```

```typescript
// common/filter/http-exception.filter.ts
import { ArgumentsHost, Catch, ExceptionFilter, HttpException } from '@nestjs/common';
import type { Request, Response } from 'express';

@Catch(HttpException)
export class HttpExceptionFilter implements ExceptionFilter {
  catch(exception: HttpException, host: ArgumentsHost) {
    const ctx = host.switchToHttp();
    const response = ctx.getResponse<Response>();
    const request = ctx.getRequest<Request>();
    const status = exception.getStatus();

    response.status(status).json({
      statusCode: status,
      timestamp: new Date().toISOString(),
      path: request.url,
      message: exception.message,
    });
  }
}
```

```
@Catch(HttpException): HttpException 및 하위 클래스(NotFoundException 등) 모두 잡힘
@Catch() 비어있으면 모든 예외를 잡음

⚠️ @Catch와 catch 파라미터 타입은 반드시 일치해야 함
   @Catch(ForbiddenException)이면 catch(exception: ForbiddenException, ...) 타입도 일치
```

## DB 에러를 가로채기 — Prisma 예시 ⭐️⭐️

```
DB 제약(unique 등) 위반 시 Prisma가 던지는 PrismaClientKnownRequestError는
HttpException이 아니라서 위 필터로는 안 잡힘 — 별도 필터가 필요함
```

```typescript
import { ArgumentsHost, Catch, ExceptionFilter } from '@nestjs/common';
import { Prisma } from '@prisma/client';
import type { Request, Response } from 'express';

@Catch(Prisma.PrismaClientKnownRequestError)
export class PrismaExceptionFilter implements ExceptionFilter {
  catch(exception: Prisma.PrismaClientKnownRequestError, host: ArgumentsHost) {
    const ctx = host.switchToHttp();
    const response = ctx.getResponse<Response>();
    const request = ctx.getRequest<Request>();

    let status = 500;
    let message = '데이터베이스 에러가 발생했습니다.';

    if (exception.code === 'P2002') {       // unique 제약 위반
      status = 409;
      message = '이미 존재하는 값입니다.';
    } else if (exception.code === 'P2025') { // 수정/삭제 대상이 없음
      status = 404;
      message = '존재하지 않는 데이터입니다.';
    }

    response.status(status).json({
      statusCode: status,
      timestamp: new Date().toISOString(),
      path: request.url,
      message,
    });
  }
}
```

```
exception.code는 Prisma가 에러 종류별로 붙여주는 식별자 — P2002/P2025는 자주 보는 두 가지일 뿐
나머지 코드는 필요해질 때마다 Prisma 공식 문서에서 찾아서 추가하면 됨
```

## 적용 방법 — 메서드 / 컨트롤러 / 전역 ⭐️

```typescript
// 메서드에만
@UseFilters(HttpExceptionFilter)
@Get()
findAll() {}

// 컨트롤러 전체
@UseFilters(HttpExceptionFilter)
@Controller('movie')
export class MovieController {}

// 전역 (main.ts)
app.useGlobalFilters(new HttpExceptionFilter());

// 전역 + DI 가능 (AppModule)
{ provide: APP_FILTER, useClass: HttpExceptionFilter }
```

---

# 한눈에

|키워드|한 줄 정리|
|---|---|
|기본 상태코드|POST → 201, 나머지(GET/PUT/PATCH/DELETE) → 200|
|`@HttpCode(HttpStatus.X)`|기본 상태코드를 직접 바꿈|
|204 No Content|"본문 없음"이 정의 자체 — return 값이 뭐든 응답 body에 안 실림|
|`@Redirect(url, status)`|리다이렉트 응답, 기본 302 — `return { url, statusCode }`로 동적 변경 가능|
|`import type { Response } from 'express'`|전역 fetch Response와 이름이 같아서 명시적 import 필수|
|`@ApiResponse` vs `@HttpCode`|문서 vs 실제 동작 — 둘은 자동으로 안 맞춰짐, 수동 동기화 필요|
|400 vs 422|형식 오류 vs 형식은 맞지만 비즈니스 규칙 위반|
|401 vs 403|인증 안 됨(다시 로그인하면 풀림) vs 권한 없음(로그인해도 안 풀림)|
|Exception Filter|`@Catch(타입)`으로 특정 예외/에러를 가로채서 응답 형식을 직접 제어|
|Prisma 에러는 HttpException이 아님|`PrismaClientKnownRequestError`는 별도 `@Catch`로 잡아야 함|

---

# 자주 하는 실수

| 실수                                                    | 원인                                                                 | 해결                                                                 |
| ----------------------------------------------------- | ------------------------------------------------------------------ | ------------------------------------------------------------------ |
| 204 라우트인데 프론트에서 `res.json()` 호출 → 파싱 에러               | 204는 body 자체가 없는데 빈 문자열을 JSON으로 파싱 시도                              | 프론트에서 status 204(또는 204/205) 분기 후 파싱 스킵 — [[NextJS_API_Client]] 참고 |
| `{ liked: false }`를 return했는데 응답에 안 보임                | 그 라우트에 `@HttpCode(204)`가 붙어있어서 body 자체가 비워짐                        | 204를 꼭 써야 하면 return 값 기대를 버리고, 데이터를 내려줘야 하면 200으로 변경               |
| `new Error()`로 throw → 500만 찍힘                        | 일반 Error는 NestJS 예외 계층을 안 타서 항상 500으로 처리됨                          | `NotFoundException` 등 NestJS 내장 예외로 throw                          |
| `@Res() res: Response` 타입 에러 또는 이상한 자동완성              | express Response 대신 전역(fetch) Response로 추론됨                        | `import type { Response } from 'express'` 명시                       |
| Swagger 문서는 200인데 실제로는 204/404 등이 나감                  | `@ApiResponse`는 문서일 뿐, 실제 상태코드와 자동 연동 안 됨                          | `@HttpCode`/실제 throw하는 예외에 맞춰 `@ApiResponse`도 수동으로 갱신              |
| Prisma unique 제약 위반인데 500만 찍힘                         | `PrismaClientKnownRequestError`는 `HttpException`이 아니라서 일반 필터로 안 잡힘 | 별도 `@Catch(Prisma.PrismaClientKnownRequestError)` 필터 추가            |
| `@Catch(ForbiddenException)`인데 catch 파라미터는 `Error` 타입 | `@Catch`와 catch 파라미터 타입이 다름                                        | 두 타입을 반드시 일치시킬 것                                                   |