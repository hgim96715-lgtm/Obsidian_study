---
aliases: [Body, Controller, createParamDecorator, Param, Query]
tags:
  - NestJS
related:
  - "[[00_NestJS_Ecosystem_HomePage]]"
  - "[[NestJS_JwtGuard]]"
  - "[[NestJS_Response]]"
  - "[[NestJS_Swagger]]"
  - "[[NextJS_API_Client]]"
---
# NestJS_Controller — 컨트롤러

```txt
URL 경로와 HTTP 메서드를 연결하는 라우터
요청을 받아서 Service 로 넘기고 응답을 반환
비즈니스 로직은 직접 처리하지 않음
```

---

# 기본 구조

```typescript
import { Controller, Get, Post, Patch, Delete } from '@nestjs/common';

@Controller('movie')       // 기본 경로 = /movie
export class MovieController {

  @Get()                   // GET /movie
  getMovies() { }

  @Get(':id')              // GET /movie/:id
  getMovie() { }

  @Post()                  // POST /movie
  postMovie() { }

  @Patch(':id')            // PATCH /movie/:id
  patchMovie() { }

  @Delete(':id')           // DELETE /movie/:id
  deleteMovie() { }
}
```

```txt
REST API 경로 패턴:
  GET    /movie        목록 조회
  GET    /movie/:id    단건 조회
  POST   /movie        생성
  PATCH  /movie/:id    수정
  DELETE /movie/:id    삭제
```

---

# 경로 조합 규칙 ⭐️

```txt
@Controller('prefix') + @Get('suffix') → GET /prefix/suffix

예시:
  @Controller('auth') + @Post('register') → POST /auth/register
  @Controller('auth') + @Post('login')    → POST /auth/login
  @Controller('movie') + @Get(':id')      → GET /movie/:id
  @Controller()        + @Get()           → GET / (루트)
```

## 정적 vs 동적 경로 순서 ⭐️

```typescript
// ✅ 정적 경로를 동적 경로 앞에 선언
@Get('popular')         // GET /movie/popular   ← 정적 먼저
getPopular() { }

@Get(':id')             // GET /movie/:id       ← 동적 나중
getMovie() { }
```

```txt
❌ :id 가 먼저 선언되면:
  GET /movie/popular → popular 가 :id 로 잡혀버림
  popular 라는 id 로 DB 조회 → NotFoundException

반드시 정적 경로를 동적 경로보다 위에 선언
```

---

# 요청 데이터 꺼내기

## @Param — Path Variable ⭐️

```typescript
@Get(':id')
getMovie(@Param('id') id: string) {
  const movie = this.movies.find(m => m.id === +id);
  if (!movie) throw new NotFoundException(`${id} 영화 없음`);
  return movie;
}
```

```txt
URL 에서 받은 값은 항상 string
숫자와 비교 시 변환 필요:
  +id           숫자 변환 (단축형)
  parseInt(id)  정수 변환
  Number(id)    숫자 변환

  movie.id === +id  ✅
  movie.id === id   ❌ (string vs number 비교 실패)
```

---

## Pipe — 자동 변환 / 검증 ⭐️⭐️

```txt
Pipe 는 컨트롤러 메서드가 실행되기 "직전"에 끼어들어 파라미터 값을
① 변환(Transformation) 하거나 ② 검증(Validation) 하는 역할

변환 실패 / 검증 실패 시 → 핸들러는 실행되지 않고 자동으로 400 Bad Request 던짐
→ 컨트롤러 코드 안에 if(!isNumber(id)) 같은 검사를 직접 안 써도 됨
```

```typescript
// @Param 에 Pipe 연결 → 자동으로 string → number 변환
@Get(':id')
getMovie(@Param('id', ParseIntPipe) id: number) {
  //                  ↑ 자동 변환
  const movie = this.movies.find(m => m.id === id);
  // +id 없이 바로 숫자로 사용 가능
}
```

```txt
ParseIntPipe 없으면:  id 타입 string → +id 변환 필요
ParseIntPipe 있으면:  id 타입 number → 바로 사용
정수 아닌 값 오면 → 400 Bad Request 자동 반환
```

## 내장 Pipe 종류 ⭐️⭐️⭐️

```txt
ParseIntPipe 는 "Parse* 계열" 내장 Pipe 중 하나일 뿐 — PK 타입이 정수가 아니면
(특히 UUID 를 PK 로 쓰는 Prisma/PostgreSQL 스키마) 다른 Pipe 를 써야 함
```

|Pipe|역할|변환 후 타입|실패 시|
|---|---|---|---|
|`ParseIntPipe`|문자열 → 정수|`number`|400 "numeric string is expected"|
|`ParseFloatPipe`|문자열 → 소수|`number`|400|
|`ParseBoolPipe`|`'true'/'false'/'1'/'0'` → 불린|`boolean`|400|
|`ParseArrayPipe`|구분자로 나눠 배열로 변환 (예: `'a,b,c'` → `['a','b','c']`)|`array`|400|
|`ParseEnumPipe`|TS enum 에 정의된 값인지 검증|enum 타입|400|
|`ParseUUIDPipe`|UUID(v3/v4/v5) 형식인지 **검증만** (값은 그대로 string)|`string`|400 "uuid is expected"|
|`ParseDatePipe`|문자열 → `Date` 객체|`Date`|400|
|`DefaultValuePipe`|값이 없을 때(undefined) 기본값 주입 — 예외 던지지 않음|지정한 타입|—|
|`ValidationPipe`|DTO 클래스 + `class-validator` 데코레이터 기반 전체 검증|DTO 타입|400 + 필드별 에러 메시지|

## ParseIntPipe vs ParseUUIDPipe — PK 타입에 맞게 선택 ⭐️⭐️

```txt
PK 가 auto-increment 정수면        → ParseIntPipe,  id: number 로 변환
PK 가 UUID (예: Prisma @default(uuid())) 면 → ParseUUIDPipe, id: string 그대로 유지

ParseUUIDPipe 는 ParseIntPipe 와 달리 "변환"을 하지 않음 — UUID 는 숫자가 아니므로
원래도 string 인 값을 "올바른 UUID 형식인가" 만 검증하고 그대로 통과시킴
```

```typescript
// PK 가 UUID 인 경우
@Get(':id')
getMovie(@Param('id', ParseUUIDPipe) id: string) {
  //                  ↑ 형식만 검증, 타입은 그대로 string
  return this.movieService.findOne(id);
}

// 특정 UUID 버전만 허용하고 싶을 때 — 인스턴스로 옵션 전달
@Get(':id')
getMovie(@Param('id', new ParseUUIDPipe({ version: '4' })) id: string) { }
```

## Parse* + DefaultValuePipe 조합 — optional 값 처리 ⭐️

```txt
⚠️ Parse* 계열은 값이 null / undefined 로 들어오면 그 즉시 예외를 던짐
   즉, "?title=" 같은 선택적 query parameter 에 ParseIntPipe 만 단독으로 쓰면
   값을 안 보냈을 때 400 에러가 나버림

→ DefaultValuePipe 를 Parse* 보다 앞에 둬서 "기본값을 먼저 채운 다음 변환"
```

```typescript
@Get()
getMovies(
  @Query('page', new DefaultValuePipe(0), ParseIntPipe) page: number,
  @Query('activeOnly', new DefaultValuePipe(false), ParseBoolPipe) activeOnly: boolean,
) {
  // page 를 안 보내면 0 → ParseIntPipe 가 그대로 0 (number) 통과
}
```

## { optional: true } — 기본값 없이 선택적으로 만들기 ⭐️⭐️⭐️

```typescript
@Get()
getUsers(
  @Query('inactiveDays', new ParseIntPipe({ optional: true }))
  inactiveDays?: number,
) {
  // inactiveDays를 안 보내면 → undefined (에러 없음)
  // inactiveDays를 보내면 → number로 변환 (잘못된 값이면 400)
}
```

```txt
DefaultValuePipe 조합과의 차이:

  DefaultValuePipe(0) + ParseIntPipe
    → "값이 없으면 0으로 채운 뒤 변환" — 결과 타입은 항상 number
    → 기본값이 있어야 하는 경우에 적합

  new ParseIntPipe({ optional: true })
    → "값이 없으면 그냥 undefined로 통과, 값이 있으면 변환" — 결과 타입은 number | undefined
    → "이 파라미터는 없어도 되는데, 있으면 반드시 정수여야 한다"는 의미를 한 줄로 표현
    → 매개변수 타입을 number?로 선언해두면 undefined인 경우를 직접 처리할 수 있음

ParseUUIDPipe, ParseEnumPipe 등 다른 Parse* 계열도 같은 { optional: true } 옵션을 지원함
```

| 방법                                     | 값 없을 때         | 값 있을 때 | 결과 타입                            |
| -------------------------------------- | -------------- | ------ | -------------------------------- |
| `ParseIntPipe` 단독                      | 400 에러         | 변환 성공  | `number`                         |
| `DefaultValuePipe(0), ParseIntPipe`    | 0으로 치환 후 변환    | 변환 성공  | `number`                         |
| `new ParseIntPipe({ optional: true })` | `undefined` 통과 | 변환 성공  | <code>number \| undefined</code> |

## Pipe 적용 위치 ⭐️

|적용 범위|문법|언제|
|---|---|---|
|파라미터 레벨|`@Param('id', ParseIntPipe)`|특정 파라미터 하나에만 적용 (가장 흔함)|
|메서드(핸들러) 레벨|`@UsePipes(ValidationPipe)` 를 메서드 위에|그 메서드의 모든 파라미터에 적용|
|클래스(컨트롤러) 레벨|`@UsePipes(ValidationPipe)` 를 클래스 위에|이 컨트롤러의 모든 메서드에 적용|
|전역|`app.useGlobalPipes(new ValidationPipe())` (main.ts)|앱 전체 — DTO 검증은 보통 전역으로 설정|

```bash
ValidationPipe 는 대부분 main.ts 에서 전역으로 한 번만 등록해두고,
@Body() 로 받는 DTO 클래스에 class-validator 데코레이터(@IsString(), @IsInt() 등)를
붙이는 방식으로 사용 — 컨트롤러 메서드마다 반복 선언하지 않음
```

## main.ts — ValidationPipe 옵션 (실전 표준 3종) ⭐️⭐️⭐️

```typescript
// main.ts
async function bootstrap() {
  const app = await NestFactory.create(AppModule);

  app.useGlobalPipes(
    new ValidationPipe({
      transform: true,
      whitelist: true,
      forbidNonWhitelisted: true,
    }),
  );

  await app.listen(3000);
}
```

|옵션|역할|
|---|---|
|`transform: true`|들어온 일반 객체(JSON)를 DTO 클래스의 "인스턴스"로 변환 — DTO 에 선언된 타입대로 string→number 같은 변환도 자동 적용|
|`whitelist: true`|DTO 에 선언되지 않은 필드는 자동으로 제거 — 클라이언트가 엉뚱한 필드를 보내도 조용히 무시됨|
|`forbidNonWhitelisted: true`|`whitelist` 처럼 제거하는 대신, 선언 안 된 필드가 오면 400 에러로 막음 (`whitelist: true` 와 같이 써야 동작)|

```txt
이 셋을 묶어서 거의 항상 같이 쓰는 이유:
  transform 없으면 → @Body() dto 가 plain object 로 와서 class-validator 데코레이터가 기대대로 동작 안 할 수 있음
  whitelist 없으면 → DTO 에 없는 필드도 그대로 서비스까지 전달됨
  forbidNonWhitelisted 없으면 → 그 필드가 조용히 제거만 되고, 클라이언트는 자기 값이 무시된 걸 알 길이 없음
```

```txt
⚠️ whitelist 가 막아주는 보안 시나리오 — "의도하지 않은 필드 주입"
  DTO 에 role 필드가 없는데 클라이언트가 body 에 role: 'admin' 을 끼워 보냈다면
  whitelist 없이는 그 값이 서비스까지 그대로 전달될 위험이 있음(서비스가 body 를 통째로 다루는 코드라면)
  whitelist: true 하나로 이런 필드는 검증 단계에서 이미 걸러짐
```

---

## @Query — Query Parameter ⭐️

```typescript
@Get()
getMovies(@Query('title') title: string) {
  if (!title) return this.movies;
  return this.movies.filter(m => m.title.startsWith(title));
}
// GET /movie?title=마이클
```

```txt
Path Variable vs Query Parameter:
  /movie/:id    특정 리소스 식별 (필수값)
  /movie?title= 필터링 / 검색 (선택값)

값 없으면 undefined → !title 로 체크
```

---

## @Body — 요청 Body ⭐️

```typescript
@Post()
postMovie(@Body('title') title: string) {
  const movie = { id: this.idCounter++, title };
  this.movies.push(movie);
  return movie;
}

// @Body() 전체 받기
@Post()
postMovie(@Body() body: CreateMovieDto) {
  return this.movieService.create(body);
}
```

```txt
@Body('title')  Body 의 title 필드만 꺼내기
@Body()         Body 전체를 DTO 타입으로 받기

JSON Body (Postman):
  { "title": "겨울왕국" }  ✅ 큰따옴표 필수
  { 'title': '겨울왕국' }  ❌ JSON 파싱 에러
```

---

# @Headers — 헤더 읽기 ⭐️

```typescript
@Get()
getMovies(@Headers('authorization') auth: string) {
  console.log(auth);   // 'Bearer eyJhb...'
}
```

## ⚠️ 헤더 키는 반드시 소문자

```txt
HTTP 스펙 상 헤더 이름은 대소문자 구분 없지만
Node.js (Express / NestJS) 는 수신 헤더 키를 전부 소문자로 변환

  @Headers('Authorization')        ❌ undefined 반환
  @Headers('authorization')        ✅

  req.headers['Authorization']     ❌ undefined
  req.headers['authorization']     ✅
```

## 토큰 꺼내기 패턴

```typescript
@Post()
postMovie(@Headers('authorization') auth: string) {
  if (!auth) throw new UnauthorizedException('토큰 없음');

  const [type, token] = auth.split(' ');
  // 'Bearer eyJhb...' → ['Bearer', 'eyJhb...']

  if (type !== 'Bearer' || !token) {
    throw new UnauthorizedException('토큰 형식 오류');
  }
}
```

```bash
⚠️ 실제 인증 로직은 Guard 로 분리하는 것이 원칙
   Controller 에서 직접 파싱하는 건 Guard 작성 전 단계
# → [[NestJS_Guard]] 참고
```

---

# @Req / @Res — 라이브러리 Request·Response 객체 ⭐️⭐️

```txt
@Req() 는 NestJS 가 감싸기 전의 "원본" Request 객체를 그대로 꺼내주는 데코레이터
(기본 어댑터가 Express 면 express.Request, Fastify 면 fastify.FastifyRequest)

@Param() / @Query() / @Body() / @Headers() 는 사실 내부적으로 이 Request 객체의
req.params / req.query / req.body / req.headers 를 대신 읽어서 꺼내주는 "편의 래퍼" 일 뿐
→ @Req() 는 그 래퍼들이 아직 안 다루는 정보까지 전부 접근할 수 있는 "원본 그 자체"
```

```typescript
import { Req } from '@nestjs/common';
import { Request } from 'express';

@Get()
getMovies(@Req() req: Request) {
  console.log(req.headers['authorization']);
  console.log(req.method);         // 'GET'
  console.log(req.url);            // '/movie'
}
```

## Request 객체의 주요 속성 ⭐️⭐️

|속성 / 메서드|설명|전용 데코레이터로는|
|---|---|---|
|`req.params`|path variable 객체|`@Param('id')`|
|`req.query`|query string 객체|`@Query('title')`|
|`req.body`|요청 바디|`@Body()`|
|`req.headers`|요청 헤더 객체 (키는 소문자)|`@Headers('authorization')`|
|`req.method`|HTTP 메서드 문자열 (`'GET'`, `'POST'` …)|대응 데코레이터 없음 — `@Req()` 필요|
|`req.url` / `req.originalUrl`|요청 경로(+query string)|대응 데코레이터 없음|
|`req.ip`|클라이언트 IP|대응 데코레이터 없음 (직접 커스텀 데코레이터로 뽑아낼 수 있음)|
|`req.cookies`|쿠키 객체 (`cookie-parser` 미들웨어 등록 필요)|대응 데코레이터 없음|
|`req.user`|Guard 가 인증 후 끼워넣은 값 (Express 기본 타입엔 원래 없음)|커스텀 Param 데코레이터로 보통 대체 (아래 참고)|

```txt
표에서 보듯, @Param/@Query/@Body/@Headers 가 다루는 영역은 req 의 일부일 뿐
→ method/url/ip/cookies 처럼 전용 데코레이터가 없는 값이 필요하면 @Req() 가 정답
→ 반대로 한두 개 값만 꺼내면 되는 흔한 경우는, 매번 @Req() 로 전체를 받기보다
  전용 데코레이터(또는 커스텀 Param 데코레이터)가 타입도 명확하고 테스트도 쉬움
```

## @Res() — Response 객체 직접 제어 ⭐️⭐️⭐️

```txt
@Req() 의 짝 — 응답을 NestJS 표준 방식이 아니라 Express/Fastify 의 res 객체로
직접 다루고 싶을 때 사용 (커스텀 헤더, 스트리밍, 라이브러리 전용 메서드 호출 등)
(상태 코드/응답 형식 자체, Response 객체 import 시 주의점 등은 [[NestJS_Response]] 참고 — 여기선 @Res() 자체만)

⚠️ 가장 중요한 규칙:
@Res() 를 한 번이라도 쓰면 NestJS 의 "표준 응답 처리"가 그 라우트에서 자동으로 꺼짐
  → return 한 값을 자동으로 JSON 직렬화 X
  → @HttpCode() 데코레이터 무시
  → 인터셉터에서 응답을 가로채 변형하는 것(response mapping)도 동작 안 함
→ 표준 방식과 같이 쓰고 싶다면 반드시 { passthrough: true } 옵션 필요
```

```typescript
import { Res, HttpStatus } from '@nestjs/common';
import { Response } from 'express';

// ❌ passthrough 없이 — 이 라우트는 이제 완전히 Express 방식으로만 동작
@Get()
getMovies(@Res() res: Response) {
  res.status(200).json(this.movies);   // 직접 응답을 끝내야 함
}

// ✅ passthrough: true — 헤더/상태코드만 직접 건드리고, 나머지는 NestJS 표준 방식 유지
@Get()
getMovies(@Res({ passthrough: true }) res: Response) {
  res.status(HttpStatus.OK);
  return this.movies;                  // 이 return 값은 그대로 자동 직렬화됨
}
```

## 정리 — 언제 @Req() / @Res() 를 직접 쓰는가 ⭐️

|상황|추천|
|---|---|
|id, title 처럼 값 하나만 필요|`@Param`/`@Query`/`@Body`/`@Headers` — 타입 명확, 테스트 쉬움|
|method/url/ip/cookies 처럼 전용 데코레이터가 없는 정보 필요|`@Req()`|
|같은 추출 로직이 여러 컨트롤러에서 반복됨 (예: req.user)|커스텀 Param 데코레이터 (아래 "커스텀 데코레이터" 섹션)|
|커스텀 헤더, 스트리밍 응답, 쿠키 설정 등 응답을 직접 제어해야 함|`@Res({ passthrough: true })`|
|응답 전체를 직접 끝내야 하는 특수한 경우 (파일 스트림 등)|`@Res()` (passthrough 없이) — 표준 응답 기능을 포기하는 트레이드오프 인지 필요|

## req.user — Guard 가 끼워넣은 값에 접근하기 ⭐️⭐️⭐️

```typescript
@UseGuards(JwtAuthGuard)
@Post()
create(
  @Body() dto: CreateRecommendationDto,
  @Req() req: Request & { user?: JwtPayload },
) {
  return this.recommendationsService.create(dto, req.user!.sub);
}
```

```txt
왜 타입을 Request & { user?: JwtPayload } 로 합치는가:

  Express 의 기본 Request 타입에는 원래 user 라는 속성이 없음
  JwtAuthGuard 가 검증 통과 후 request.user = payload 로 "런타임에 직접 추가" 해준 것일 뿐
  (Guard 가 request.user 를 채우는 동작 자체는 [[NestJS_Guard]] 참고)
  → TS 는 이렇게 나중에 끼워넣은 속성을 알 길이 없어서, 직접 타입으로 알려줘야 함

  Request & { user?: JwtPayload }:
    원래의 Request 타입 + "user 라는 속성이 추가로 있다" 는 정보를 교차 타입(&)으로 합친 것
    → 이후 req.user 로 접근해도 TS 가 더 이상 "그런 속성 없음" 에러를 내지 않음

req.user!  의 ! (non-null assertion) 이 붙는 이유:
  타입 선언은 user?: JwtPayload (있을 수도, 없을 수도) 인데
  이 라우트는 @UseGuards(JwtAuthGuard) 를 거쳐야만 도달하므로
  "여기 도달했다면 user 는 무조건 있다" 는 게 보장됨 → 그 보장을 ! 로 직접 단언하는 것
  ⚠️ Guard 를 안 거치는 라우트에서 이렇게 쓰면 실제로 undefined 일 수 있어 위험함
```

## 더 깔끔한 방법 — 커스텀 데코레이터로 대체 ⭐️⭐️⭐️

```txt
바로 위 코드가 매번 하는 일(req.user 꺼내고, !로 단언하고, .sub 까지 파고들기)은
아래 "커스텀 데코레이터" 섹션에서 만드는 Param 데코레이터가 정확히 해결해주는 문제임
→ @Req() + 타입 교차를 매번 다시 쓰는 대신, 한 번 만들어둔 데코레이터를 재사용하는 게 더 깔끔함
```

```typescript
// ❌ 매번 @Req() 로 직접 꺼내기 — 타입도 매번 다시 적어야 함
create(
  @Body() dto: CreateRecommendationDto,
  @Req() req: Request & { user?: JwtPayload },
) {
  return this.recommendationsService.create(dto, req.user!.sub);
}

// ✅ 커스텀 데코레이터로 — 타입 선언 반복 없음, 호출부도 짧음
create(
  @Body() dto: CreateRecommendationDto,
  @UserId() userId: number,
) {
  return this.recommendationsService.create(dto, userId);
}
```

```txt
언제 @Req() 를 직접 쓰는 게 맞는가:
  req.method / req.url / req.headers 처럼 user 외의 다른 정보도 같이 필요할 때
  또는 그 라우트 하나에서만 쓰는 임시 코드라 데코레이터를 따로 만들 가치가 없을 때

언제 커스텀 데코레이터가 맞는가:
  같은 추출/판단 로직이 여러 컨트롤러에서 반복될 때
  → 한 번만 정의해두고 어디서나 재사용 (바로 아래에서 그 정의 방법을 일반화해서 다룸)
```

---

# 커스텀 데코레이터 — 범용 패턴 ⭐️⭐️⭐️

```txt
NestJS 데코레이터는 @Controller, @Get 같은 "내장 데코레이터" 만 있는 게 아니라
직접 만들어 쓰는 "커스텀 데코레이터" 도 똑같이 1급 시민으로 동작함

커스텀 데코레이터는 목적에 따라 크게 두 종류로 나뉨 — 이 둘을 구분하는 게 핵심:

  ① Param 데코레이터   요청에서 값을 "추출"해서 메서드 인자로 주입       → createParamDecorator
  ② 메타데이터 데코레이터  클래스/메서드에 "표시(꼬리표)"만 붙여둠      → SetMetadata + Reflector

@UserId() 는 ①의 한 가지 적용 사례일 뿐, 이 패턴 자체는
"req.user 에서 sub 꺼내기" 뿐 아니라 IP, User-Agent, 페이지네이션 옵션 등
요청에서 반복적으로 꺼내 쓰는 어떤 값에도 똑같이 적용 가능
```

## 종류 비교 ⭐️⭐️

|구분|① Param 데코레이터|② 메타데이터 데코레이터|
|---|---|---|
|만드는 함수|`createParamDecorator`|`SetMetadata(key, value)`|
|붙이는 위치|메서드의 **파라미터**|**클래스** 또는 **메서드** 위|
|동작 시점|요청이 들어올 때마다 실행되어 값을 바로 반환|값을 "부착"만 해두고, 나중에 Guard/Interceptor 가 `Reflector` 로 읽음|
|반환하는 것|컨트롤러 메서드에 주입될 값 그 자체|아무것도 반환 안 함 (메타데이터만 저장)|
|대표 예시|`@UserId()`, `@CurrentUser()`, `@ClientIp()`|`@Roles('admin')`, `@Public()`|
|내장 데코레이터로 치면|`@Param()`, `@Query()`, `@Body()` 와 같은 계열|`@Controller()` 자체보다는, Guard 가 읽는 꼬리표 계열|

---

## ① Param 데코레이터 — createParamDecorator

```typescript
// src/common/decorator/current-user.decorator.ts
import { createParamDecorator, ExecutionContext } from '@nestjs/common';

export const CurrentUser = createParamDecorator(
  (data: string | undefined, ctx: ExecutionContext) => {
    const request = ctx.switchToHttp().getRequest();
    const user = request.user;

    // data 를 안 넘기면 user 객체 전체, 넘기면 그 필드만 반환
    return data ? user?.[data] : user;
  },
);
```

```typescript
// 사용 — 같은 데코레이터를 상황에 따라 다르게 호출
@Get('me')
getProfile(@CurrentUser() user: User) {          // 객체 전체
  return user;
}

@Get('me/email')
getEmail(@CurrentUser('email') email: string) {  // 'email' 필드만
  return { email };
}
```

```txt
data 파라미터의 역할:
  @CurrentUser()         → createParamDecorator 콜백의 data = undefined
  @CurrentUser('email')  → data = 'email'
  즉, 데코레이터 호출 시 () 안에 넣은 값이 콜백의 첫 번째 인자로 그대로 전달됨

  → 노트 앞부분의 @UserId() 는 이 콜백 안에서 항상 user.sub 만 반환하도록
    고정시켜둔, "data 를 안 쓰는" 더 단순한 버전일 뿐
    (필요하면 @UserId() 도 위처럼 data 파라미터를 받아 user[data] 형태로 일반화 가능)
```

## ExecutionContext — 컨텍스트별 접근 메서드 ⭐️

|메서드|반환값|언제 쓰는가|
|---|---|---|
|`switchToHttp()`|`HttpArgumentsHost`|REST API — `.getRequest()` / `.getResponse()` 로 Express·Fastify req/res 접근|
|`switchToWs()`|`WsArgumentsHost`|WebSocket — 클라이언트 소켓, 수신 데이터 접근|
|`switchToRpc()`|`RpcArgumentsHost`|마이크로서비스(RPC) — 수신 메시지, 컨텍스트 접근|
|`getClass()`|클래스 참조|이 요청이 어떤 **컨트롤러 클래스**에서 처리되는지 (클래스 레벨 메타데이터 조회용)|
|`getHandler()`|메서드 참조|이 요청이 어떤 **핸들러(메서드)**에서 처리되는지 (메서드 레벨 메타데이터 조회용)|
|`getArgs()`|배열|핸들러에 전달될 원본 인자 전체|

```txt
Param 데코레이터는 거의 항상 switchToHttp().getRequest() 만 쓰면 충분
getClass() / getHandler() 는 ②번 메타데이터 데코레이터를 Guard/Interceptor 에서
읽을 때 주로 사용 (바로 아래 참고)
```

---

## ② 메타데이터 데코레이터 — SetMetadata + Reflector ⭐️⭐️⭐️

```txt
"이 라우트는 인증 없이 접근 가능" / "이 라우트는 admin 만 접근 가능" 같은 정보를
Guard 코드 안에 if 문으로 박아넣지 않고, 데코레이터 한 줄로 라우트 위에 "표시"해두는 패턴

데코레이터 자체는 판단을 하지 않음 — 그냥 꼬리표를 붙일 뿐
실제 판단(허용/차단)은 Guard 나 Interceptor 가 그 꼬리표를 읽어서 함
```

```mermaid-beautiful
graph LR
  A["정의<br/>SetMetadata(key, value)"] --> B["부착<br/>'@Roles(admin)' 을<br/>컨트롤러 메서드에 적용"]
  B --> C["요청 발생"]
  C --> D["읽기<br/>Reflector.get(key, context.getHandler())"]
  D --> E["Guard 가 값으로<br/>허용/차단 판단"]
```

```typescript
// roles.decorator.ts — 정의 + 부착용 헬퍼
import { SetMetadata } from '@nestjs/common';

export const ROLES_KEY = 'roles';
export const Roles = (...roles: string[]) => SetMetadata(ROLES_KEY, roles);
```

```typescript
// roles.guard.ts — 읽기 + 판단
import { CanActivate, ExecutionContext, Injectable } from '@nestjs/common';
import { Reflector } from '@nestjs/core';
import { ROLES_KEY } from './roles.decorator';

@Injectable()
export class RolesGuard implements CanActivate {
  constructor(private reflector: Reflector) {}

  canActivate(context: ExecutionContext): boolean {
    const roles = this.reflector.get<string[]>(ROLES_KEY, context.getHandler());
    if (!roles) return true;                       // 표시 없으면 통과

    const { user } = context.switchToHttp().getRequest();
    return roles.some((role) => user?.roles?.includes(role));
  }
}
```

```typescript
// 사용
@UseGuards(JwtAuthGuard, RolesGuard)
@Roles('admin')
@Delete(':id')
deleteMovie(@Param('id', ParseIntPipe) id: number) { }
```

```txt
Reflector.get(key, target) 의 target 자리에 무엇을 넘기느냐:
  context.getHandler()  → 이 메서드(@Delete(':id') 자체)에 붙은 메타데이터
  context.getClass()    → 이 컨트롤러 클래스 전체에 붙은 메타데이터
  → 메서드 레벨에 없으면 클래스 레벨도 확인하고 싶다면 둘 다 조회해서 병합
  (getAllAndOverride로 둘을 한 번에 조회하는 패턴은 [[NestJS_JwtGuard]] 참고)

⚠️ SetMetadata 는 정적인 값(역할 목록, true/false 플래그 등)에만 사용
   매번 달라지는 로직은 메타데이터가 아니라 Guard/Interceptor 안에 작성
```

### 자주 쓰는 메타데이터 데코레이터 예시 ⭐️

|이름|용도|읽는 쪽|
|---|---|---|
|`@Roles('admin', 'editor')`|역할 기반 접근 제어|`RolesGuard`|
|`@Public()`|인증 자체를 건너뛰는 라우트 표시|`JwtAuthGuard` (Public 이면 true 반환)|
|`@CacheTTL(60)`|응답 캐싱 시간(초) 표시|`CacheInterceptor`|
|`@LogAction('CREATE_POST')`|어떤 동작인지 로깅용 표시|`LoggingInterceptor`|

```txt
공통 패턴: "정의(SetMetadata) → 부착(데코레이터를 라우트에 사용) → 읽기(Reflector.get)"
```

---

## 데코레이터 합치기 — applyDecorators ⭐️

```txt
인증이 필요한 라우트마다
  @UseGuards(JwtAuthGuard, RolesGuard)
  @Roles('admin')
  @ApiBearerAuth()
4줄을 매번 반복하는 게 번거로울 때, 여러 데코레이터를 하나로 묶는 방법
```

```typescript
// auth.decorator.ts
import { applyDecorators, UseGuards } from '@nestjs/common';
import { ApiBearerAuth, ApiUnauthorizedResponse } from '@nestjs/swagger';
import { JwtAuthGuard } from './jwt-auth.guard';
import { RolesGuard } from './roles.guard';
import { Roles } from './roles.decorator';

export function Auth(...roles: string[]) {
  return applyDecorators(
    Roles(...roles),
    UseGuards(JwtAuthGuard, RolesGuard),
    ApiBearerAuth(),
    ApiUnauthorizedResponse({ description: 'Unauthorized' }),
  );
}
```

```typescript
// 사용 — 4줄이 1줄로
@Auth('admin')
@Delete(':id')
deleteMovie(@Param('id', ParseIntPipe) id: number) { }
```

```txt
applyDecorators 는 ①, ② 어떤 종류든 합칠 수 있음 — 단순히 "여러 데코레이터를
순서대로 적용해주는 합성 함수" 이고, 데코레이터 종류를 가리지 않음
```

---

## 언제 무엇을 만드는가 ⭐️⭐️

|상황|선택|
|---|---|
|요청에서 값을 꺼내서 컨트롤러 인자로 쓰고 싶다|① `createParamDecorator`|
|특정 필드만 골라서 받고 싶다 (재사용 가능하게)|① + `data` 파라미터 활용|
|라우트에 "이건 admin 만" 같은 표시만 해두고 싶다|② `SetMetadata` + Guard 의 `Reflector`|
|그 표시를 Interceptor 에서 읽어 캐싱/로깅하고 싶다|② `SetMetadata` + Interceptor 의 `Reflector`|
|여러 데코레이터를 매번 같이 쓰는 게 반복된다|`applyDecorators` 로 하나로 묶기|

---

# Guard / Pipe / Interceptor — 클래스 레벨로 한 번에 적용하기 ⭐️⭐️⭐️

```txt
지금까지 본 @UseGuards 등은 전부 "메서드 위" 에 붙인 예시뿐이었음
같은 데코레이터를 컨트롤러 "클래스 위" 에 붙이면, 그 컨트롤러의 모든 메서드에 자동 적용됨
→ Guard/Pipe/Interceptor/Filter 전부 이 규칙이 동일하게 적용됨 (Pipe 만의 얘기가 아님)
```

```typescript
@UseGuards(JwtAuthGuard)   // 클래스 위에 — 이 컨트롤러의 모든 라우트가 로그인 필요
@Controller('movie')
export class MovieController {
  @Get() findAll() { /* 이 메서드도 JwtAuthGuard 적용됨 */ }
  @Post() create() { /* 이 메서드도 마찬가지 */ }
}
```

|적용 범위|예시|
|---|---|
|메서드만|`@UseGuards(...)` 를 그 메서드 위에만|
|컨트롤러 전체|`@UseGuards(...)` 를 클래스 위에|
|앱 전체|`app.useGlobalGuards(...)` 또는 `APP_GUARD` provider ([[NestJS_Guard]] 참고)|

```txt
판단 기준: 컨트롤러의 거의 모든 라우트가 같은 보호/변환이 필요하다면 클래스 레벨로 한 번에
일부 라우트만 예외(공개 엔드포인트 등)라면 → 클래스 레벨 + @Public() 같은 메타데이터로 예외 처리
  (전역 Guard + @Public() 예외 처리 패턴은 [[NestJS_Guard]] 참고)
```

---

# Controller 메서드가 꼭 async 여야 하는가 ⭐️⭐️

```txt
NestJS 는 핸들러가 반환하는 값이 일반 값이든 Promise 든 상관없이 전부 알아서 처리함
→ Service 가 반환한 Promise 를 그대로 return 하면, await 없이도 NestJS 가 대신 resolve 해서 응답으로 보냄
```

```typescript
// 둘 다 동일하게 동작 (Service 가 Promise 를 반환하는 경우 — Prisma 등)
getMovies() {
  return this.movieService.findAll();         // Promise 를 그대로 return — NestJS 가 await 해줌
}

async getMovies() {
  return await this.movieService.findAll();   // 직접 await — 결과는 동일
}
```

|상황|async 필요?|
|---|---|
|Service 호출 결과를 그대로 return|불필요 (있어도 동작은 같음)|
|결과를 가공하거나 여러 호출을 조합해야 함|필요 — `await` 로 값을 먼저 받아야 가공 가능|
|try/catch 로 에러를 직접 잡아야 함|필요 — `await` 없이는 reject 를 catch 로 못 잡음|

```txt
실무에서는 결과를 가공하거나 여러 호출을 조합하는 경우가 많아서, 관례적으로 거의 항상 async 를 붙임
```

---

# 상태코드, 응답, 예외 처리는 별도 노트로 ⭐️

```txt
204 No Content 만들기, @HttpCode/@Redirect, Response 객체 import 주의점,
Swagger 상태코드 동기화, 예외 클래스 선택(400/401/403/404/409/422), Exception Filter 등은
모두 "응답을 어떻게 내보내는가"라는 한 주제라 [[NestJS_Response]] 로 모아둠
```

---

# 실전 CRUD 전체 코드 (인증 포함)

```typescript
import {
  Controller, Get, Post, Patch, Delete, UseGuards, HttpCode,
  Param, Body, Query, ParseIntPipe,
} from '@nestjs/common';
import { JwtAuthGuard } from './jwt-auth.guard';
import { UserId } from './common/decorator/user-id.decorator';

@Controller('movie')
export class MovieController {
  constructor(private readonly movieService: MovieService) {}

  // GET /movie  /  GET /movie?title=검색어
  @Get()
  getMovies(@Query('title') title?: string) {
    return this.movieService.findAll(title);
  }

  // GET /movie/popular  ← 정적 경로 먼저 ⭐️
  @Get('popular')
  getPopular() {
    return this.movieService.findPopular();
  }

  // GET /movie/:id  ← 동적 경로 나중
  @Get(':id')
  getMovie(@Param('id', ParseIntPipe) id: number) {
    return this.movieService.findOne(id);
  }

  // POST /movie — 로그인한 유저만, 작성자 id 까지 같이 전달
  @UseGuards(JwtAuthGuard)
  @Post()
  postMovie(
    @Body() body: CreateMovieDto,
    @UserId() userId: number,
  ) {
    return this.movieService.create(body, userId);
  }

  // PATCH /movie/:id
  @Patch(':id')
  patchMovie(
    @Param('id', ParseIntPipe) id: number,
    @Body() body: UpdateMovieDto,
  ) {
    return this.movieService.update(id, body);
  }

  // DELETE /movie/:id — admin 만 + 204 No Content (상태코드 자세한 건 NestJS_Response 참고)
  @Auth('admin')
  @HttpCode(204)
  @Delete(':id')
  async deleteMovie(@Param('id', ParseIntPipe) id: number) {
    await this.movieService.remove(id);
  }
}
```

---

# 한눈에

|데코레이터 / 함수|역할|예시|
|---|---|---|
|`@Controller('경로')`|기본 경로|`@Controller('movie')`|
|`@Get()` / `@Post()` / `@Patch(':id')` / `@Delete(':id')`|HTTP 메서드 라우팅||
|`@Param('id')`|Path Variable|`/movie/1` → id='1'|
|`@Param('id', ParseIntPipe)`|Path Variable 숫자 변환|→ id=1 (number)|
|`@Query('title')`|Query Parameter|`?title=값`|
|`@Body()`|Body 전체|DTO 타입으로|
|`@Body('title')`|Body 필드 하나||
|`@Headers('authorization')`|헤더 — 소문자 필수 ⭐️||
|`@Req()`|원본 Request 객체 전체|method/url/ip/cookies 등 전용 데코레이터 없는 값|
|`@Res({ passthrough: true })`|원본 Response 객체|커스텀 헤더/상태코드 직접 제어, 표준 응답과 공존|
|`ParseIntPipe` / `ParseUUIDPipe` 등|Pipe — 변환 또는 검증|PK 가 정수면 Int, UUID 면 UUID|
|`DefaultValuePipe`|Pipe — 누락 시 기본값|Parse* 보다 앞에 둬서 조합|
|`createParamDecorator`|① Param 데코레이터 정의|요청에서 값 추출 → 인자로 주입 (`@UserId()`, `@CurrentUser()`)|
|`SetMetadata(key, value)`|② 메타데이터 부착|클래스/메서드에 정적 꼬리표 (`@Roles()`, `@Public()`)|
|`Reflector.get(key, target)`|② 메타데이터 읽기|Guard/Interceptor 안에서 부착된 값 꺼내기|
|`applyDecorators(...)`|여러 데코레이터 합치기|`@Auth('admin')` 하나로 4개 묶기|

```txt
@Req() + Request & { user?: ... } + ! 와 createParamDecorator 기반 데코레이터는
같은 문제(Guard 가 넣어준 user 꺼내기)를 다른 방식으로 푼 것
→ 반복되면 ① Param 데코레이터, 한 번/다른 정보도 필요하면 @Req()

createParamDecorator 와 SetMetadata 는 이름은 둘 다 "커스텀 데코레이터" 지만
전자는 "값을 즉시 추출", 후자는 "표시만 부착 후 나중에 읽음" 으로 동작 시점이 다름

Guard/Pipe/Interceptor 는 메서드 위 vs 클래스 위 어디 붙이느냐로 적용 범위가 결정됨 (Pipe 만의 얘기가 아님)
Service 호출 결과를 그대로 return 한다면 async 는 있어도 없어도 동작은 같음 — 가공/조합이 필요할 때만 필수

상태코드/응답/예외 처리 전반은 → [[NestJS_Response]]
```

---

# 자주 하는 실수

|실수|원인|해결|
|---|---|---|
|`movie.id === id` 비교 오류|string vs number|`+id` 또는 `ParseIntPipe`|
|UUID PK 인데 `ParseIntPipe` 사용|UUID 는 숫자로 변환 불가|`ParseUUIDPipe` 사용 (string 유지)|
|optional query 에 `ParseIntPipe` 단독 사용 → 값 안 보내면 400|Parse* 는 undefined 들어오면 즉시 예외|`DefaultValuePipe` 를 Parse* 보다 앞에 둬서 조합|
|JSON Body 파싱 에러|작은따옴표 사용|`"key"` 반드시 큰따옴표|
|GET /popular → :id 로 잡힘|정적 경로가 동적 경로 아래 선언|popular 를 :id 위로 이동 ⭐️|
|헤더 undefined|`'Authorization'` 대문자|`'authorization'` 소문자 ⭐️|
|`req.user` 에 TS 에러|기본 Request 타입엔 user 없음|`Request & { user?: ... }` 로 교차, 또는 Param 데코레이터로 회피 ⭐️|
|`@Res()` 썼더니 `return` 한 값이 응답에 안 나감|`@Res()` 사용 시 표준 응답 처리가 자동으로 꺼짐|`@Res({ passthrough: true })` 로 표준 방식과 공존|
|메타데이터 데코레이터 안에 인증/권한 로직을 직접 작성|`SetMetadata` 는 "표시"용, 판단은 Guard/Interceptor 책임|데코레이터는 선언적으로 유지, 판단은 `Reflector.get` 으로 읽는 쪽에서|
|`Reflector.get` 이 항상 undefined|`getHandler()` 만 보고 클래스 레벨 메타데이터는 못 읽음|메서드/클래스 둘 다 필요하면 `getHandler()` 와 `getClass()` 둘 다 조회|
|DTO 에 없는 필드를 보냈는데 에러도 안 나고 그냥 사라짐|`forbidNonWhitelisted` 없이 `whitelist` 만 켬|에러로 막고 싶다면 `forbidNonWhitelisted: true` 도 같이|

```txt
204 응답, 404/500 예외, Response import 충돌 등 응답 관련 실수는 [[NestJS_Response]]의
"자주 하는 실수"처럼 다루지 — 거기서 한 번에 정리해둠 (여기는 Controller 자체의 실수만)
```