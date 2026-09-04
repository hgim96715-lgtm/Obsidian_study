---
aliases:
  - Body
  - Controller
  - createParamDecorator
  - Param
  - Pipe
  - Query
  - "@HttpCode"
  - HttpStatus
  - "@Req"
  - "@Res"
  - Req
  - Res
tags:
  - NestJS
related:
  - "[[00_NestJS_Ecosystem_HomePage]]"
  - "[[NestJS_Service_Provider]]"
  - "[[NestJS_Swagger]]"
  - "[[NestJS_Pipe]]"
  - "[[NestJS_Concept]]"
  - "[[NestJS_JwtGuard]]"
  - "[[TS_Type_Guards]]"
  - "[[GitHub_Actions]]"
---
# NestJS_Controller — 컨트롤러

>[!info]
>컨트롤러 = HTTP 요청을 받아서 응답을 돌려주는 클래스. 
>URL 경로와 HTTP 메서드를 핸들러 함수에 연결한다. 
>비즈니스 로직은 Service에 위임하고, 컨트롤러는 "요청 받기 → 서비스 호출 → 응답 반환" 세 가지만 한다.
> DTO → [[NestJS_DTO]], Pipe → [[NestJS_Pipe]], Guard/인증 → [[NestJS_JwtGuard]]

---

# 컨트롤러란 — 역할과 책임 ⭐️⭐️⭐️⭐️

```txt
HTTP 요청이 들어오면:
  1. 컨트롤러가 요청을 받음 (어떤 URL, 어떤 메서드인지)
  2. 요청 데이터를 꺼냄 (@Param, @Query, @Body)
  3. Service에 전달해서 처리
  4. Service의 결과를 클라이언트에게 응답

컨트롤러가 하지 않는 것:
  DB 쿼리, 비즈니스 로직, 이메일 발송 등 → 전부 Service에
  "얇은 컨트롤러(Thin Controller)" 원칙
  컨트롤러 메서드가 5줄을 넘으면 로직이 Service로 가야 하는 신호
```

```typescript
@Controller('posts')           // 모든 라우트의 기본 경로: /posts
export class PostsController {
  constructor(private readonly postsService: PostsService) {}  // Service 주입

  @Get()                       // GET /posts
  findAll() {
    return this.postsService.findAll();   // Service에 위임
  }
}
```

---

# 기본 구조 ⭐️⭐️⭐️⭐️

```typescript
@Controller('경로')            // 기본 URL 경로 설정
export class XxxController {
  constructor(private readonly xxxService: XxxService) {}

  @Get()                       // GET /경로
  @Post()                      // POST /경로
  @Patch(':id')                // PATCH /경로/:id
  @Delete(':id')               // DELETE /경로/:id
  @Put(':id')                  // PUT /경로/:id (전체 교체, 잘 안 씀)
  메서드명() { ... }
}
```

## HTTP 메서드 + 경로 조합

```typescript
@Controller('posts')
export class PostsController {
  @Get()              // GET /posts
  @Get(':id')         // GET /posts/123
  @Get(':id/comments') // GET /posts/123/comments
  @Post()             // POST /posts
  @Patch(':id')       // PATCH /posts/123
  @Delete(':id')      // DELETE /posts/123
}
```

```txt
경로 중복 없이 설계하는 방법:
  GET    /posts         → 목록 조회
  GET    /posts/:id     → 단건 조회
  POST   /posts         → 생성
  PATCH  /posts/:id     → 수정
  DELETE /posts/:id     → 삭제

  이 패턴이 RESTful API의 기본
  추가 동작이 필요하면: POST /posts/:id/publish, POST /posts/:id/like 등
```

---

# 요청 데이터 꺼내기 ⭐️⭐️⭐️⭐️

## @Param — 경로 파라미터

```typescript
// GET /posts/550e8400-e29b-41d4-a716-446655440000
@Get(':id')
findOne(@Param('id', ParseUUIDPipe) id: string) {
  //            ↑        ↑
  //            키       파이프 (UUID 형식 자동 검증)
  return this.postsService.findOne(id);
}
```

```txt
ParseUUIDPipe를 항상 붙이는 이유:
  UUID 형식이 아닌 값으로 DB 쿼리 → Prisma 에러 → 500
  ParseUUIDPipe로 미리 걸러내면 → 자동 400 BadRequest

여러 경로 파라미터:
  @Get(':roomId/messages/:messageId')
  findMessage(
    @Param('roomId',    ParseUUIDPipe) roomId:    string,
    @Param('messageId', ParseUUIDPipe) messageId: string,
  )
```

## @Query — 쿼리 파라미터

```typescript
// GET /posts?q=검색어&status=published&cursor=abc
@Get()
findAll(@Query() query: ListPostsQueryDto) {
  // query.q, query.status, query.cursor — DTO에서 타입과 검증 정의
  return this.postsService.findAll(query);
}
```

```txt
@Query() 전체 vs @Query('key') 하나:
  @Query() query: ListPostsQueryDto  → 전체를 DTO로 받음 (권장)
  @Query('page') page: string        → 특정 키 하나만 (간단한 경우)

DTO로 받아야 하는 이유:
  타입 변환 (@Type(() => Number) 로 string → number)
  기본값 설정 (take?: number = 20)
  유효성 검사 (@IsEnum, @IsInt 등)
  → [[NestJS_DTO]] 쿼리 파라미터 DTO 섹션
```

## @Query optional 파라미터 순서 에러 ⭐️⭐️⭐️

```typescript
// ❌ TS 에러 — optional(?) 파라미터 뒤에 필수 파라미터 올 수 없음
@Get()
findAll(
  @Query('machineId') machineId?: string,   // optional
  @Query('page', new DefaultValuePipe(1), ParseIntPipe) page: number,
  // ↑ 에러: Required parameter 'page' cannot follow optional parameter
) {}
```

```typescript
// ✅ 해결 1 — ? 대신 | undefined 로 명시
@Get()
findAll(
  @Query('machineId') machineId: string | undefined,  // optional이지만 필수 위치
  @Query('page', new DefaultValuePipe(1), ParseIntPipe) page: number,
) {}

// ✅ 해결 2 — optional을 뒤로 이동
@Get()
findAll(
  @Query('page', new DefaultValuePipe(1), ParseIntPipe) page: number,
  @Query('machineId') machineId?: string,   // 필수 파라미터 뒤로
) {}
```

```txt
TS 규칙:
  함수 파라미터에서 optional(?)은 required 뒤에만 올 수 있음
  optional 뒤에 required가 오면 에러

  string | undefined vs string?:
    string?     = TS 선택적 파라미터 (뒤에 required 오면 에러)
    string | undefined = 타입은 undefined 포함이지만 파라미터는 "필수 위치"
                 → TS 파라미터 순서 규칙 통과

  권장:
  @Query() DTO 방식을 쓰면 이 문제 자체가 없음
  DTO 필드에서 @IsOptional()로 optional 처리

  TS 파라미터 위치 규칙 원리 → [[TS_Type_Guards]] string? vs string | undefined 섹션
```


## @Body — 요청 body

```typescript
// POST /posts
@Post()
create(@Body() dto: CreatePostDto) {
  // dto.title, dto.content — DTO에서 검증
  return this.postsService.create(dto);
}
```

## @UserId() — 로그인한 사용자 ID 꺼내기

```typescript
// JWT 토큰에서 자동으로 사용자 ID 추출 (커스텀 데코레이터)
@Post()
create(
  @UserId()          userId: string,  // 로그인한 사람의 ID
  @Body() dto:       CreatePostDto,
) {
  return this.postsService.create(userId, dto);
}
```

```txt
@UserId()는 커스텀 파라미터 데코레이터
  JwtAuthGuard가 request.user에 payload를 저장
  @UserId()가 request.user.sub을 꺼내줌
  → [[NestJS_JwtGuard]] 파라미터 데코레이터 섹션
```

---

# 응답 제어 ⭐️⭐️⭐️⭐️

## return 값이 자동으로 JSON이 되는 이유

```typescript
@Get(':id')
findOne(@Param('id', ParseUUIDPipe) id: string) {
  return this.postsService.findOne(id);
  // 객체를 return하면 자동으로 JSON.stringify → 200 응답
}
```

```txt
NestJS는 컨트롤러 메서드가 return한 값을:
  객체/배열이면 → JSON.stringify()해서 body에 담아 응답
  문자열이면   → 그대로 text/plain으로 응답
  undefined이면 → body 없이 응답

직접 res.json()이나 res.send()를 호출할 필요 없음
return만 하면 NestJS가 알아서 처리

Promise를 return해도:
  async 함수가 Promise를 반환하면
  NestJS가 await한 결과로 응답
  → async/await 자연스럽게 사용 가능
```

## @HttpCode — 상태 코드 변경 ⭐️⭐️⭐️⭐️

```typescript
import { HttpCode, HttpStatus } from '@nestjs/common';

// 숫자 직접 사용
@Post()
@HttpCode(201)
create() { ... }

// HttpStatus enum 사용 (권장 — 이름만 봐도 의미 명확)
@Post()
@HttpCode(HttpStatus.CREATED)        // 201
create() { ... }

@Delete(':id')
@HttpCode(HttpStatus.NO_CONTENT)     // 204
async remove() {
  await this.service.remove(id);
  // 204면 body 없음 → return 안 해도 됨
}
```

```txt
숫자 vs HttpStatus enum:
  @HttpCode(204)                      → 숫자 직접 (짧지만 의미 파악에 불편)
  @HttpCode(HttpStatus.NO_CONTENT)    → enum 사용 (이름만 봐도 의미 명확)
```

## 자주 쓰는 상태 코드 ⭐️⭐️⭐️⭐️

|상황|코드|HttpStatus|
|---|---|---|
|조회 성공|200|`HttpStatus.OK`|
|생성 성공|201|`HttpStatus.CREATED`|
|삭제·처리 완료 (body 없음)|204|`HttpStatus.NO_CONTENT`|
|요청 형식 오류|400|`HttpStatus.BAD_REQUEST`|
|인증 안 됨 (로그인 필요)|401|`HttpStatus.UNAUTHORIZED`|
|권한 없음 (로그인 했지만 거부)|403|`HttpStatus.FORBIDDEN`|
|리소스 없음|404|`HttpStatus.NOT_FOUND`|
|중복·충돌|409|`HttpStatus.CONFLICT`|
|서버 에러|500|`HttpStatus.INTERNAL_SERVER_ERROR`|

```txt
기본값:
  @HttpCode 없으면 → 200 (POST도 기본 200)

POST 상태 코드 구분:
  생성 POST   → 201 Created    (POST /posts, POST /rooms)
  액션형 POST → 200 OK 그대로  (POST /rooms/:id/join, POST /posts/:id/like, POST /posts/:id/hide)

  생성 POST: 새 리소스가 DB에 만들어짐 → "생성됐다"를 201로 표현
  액션형 POST: 어떤 동작을 수행 (좋아요·숨기기·입장 등) → 리소스 생성이 아니므로 200

  → 생성용 POST에만 @HttpCode(HttpStatus.CREATED) 일괄 적용
  → join·like·hide·read 같은 액션형은 @HttpCode 생략 (200 기본값 사용)

204 No Content:
  응답 body를 보내지 않음
  DELETE, 토글처럼 결과 데이터가 필요 없는 경우
  return 값이 있어도 무시됨

400 vs 401 vs 403:
  400 BadRequest   → 데이터 형식 자체가 잘못됨 (ValidationPipe 자동)
  401 Unauthorized → 로그인이 필요한데 토큰 없음
  403 Forbidden    → 로그인은 했지만 "당신은 이걸 할 권한이 없음"
```


## HttpException 클래스 — throw로 직접 사용 ⭐️⭐️⭐️⭐️

NestJS 내장 예외 클래스. `throw`하면 ExceptionFilter가 자동으로 JSON 에러 응답 생성.  
Controller뿐 아니라 **Service에서 throw해도 자동 버블링**됨.

```typescript
import {
  NotFoundException, BadRequestException, ForbiddenException,
  UnauthorizedException, ConflictException,
  ServiceUnavailableException, InternalServerErrorException,
} from '@nestjs/common';

throw new NotFoundException('사용자를 찾을 수 없습니다.');
throw new ServiceUnavailableException('외부 결제 API 응답 시간 초과');
```

| 클래스 | 상태 코드 | 사용 시점 |
|--------|----------|----------|
| `BadRequestException` | 400 | 입력값 형식 오류 (ValidationPipe 자동 사용) |
| `UnauthorizedException` | 401 | 인증 없음 (토큰 없음·만료) |
| `ForbiddenException` | 403 | 인증은 됐지만 권한 없음 |
| `NotFoundException` | 404 | 리소스 없음 (DB 조회 결과 null) |
| `ConflictException` | 409 | 중복·충돌 (이미 존재하는 이메일 등) |
| `InternalServerErrorException` | 500 | 예상 못한 서버 에러 |
| `ServiceUnavailableException` | 503 | 외부 API·DB 등 의존 서비스 불가 |

```txt
503 ServiceUnavailableException 언제 쓰나?
  — 카카오·네이버·결제 등 외부 API 호출 실패
  — fetch 타임아웃 (AbortSignal.timeout) 초과
  — 내 서버 로직 문제가 아닌 "외부가 안 됨"을 표현할 때

Service에서 throw하면?
  Controller까지 자동 버블링 → GlobalExceptionFilter가 잡음
  try/catch 없어도 됨 (의도적 에러는 throw, 예상 못한 에러만 catch)
```

## @Header — 응답 헤더 추가 ⭐️⭐️

```typescript
import { Header } from '@nestjs/common';

@Get(':id/download')
@Header('Content-Disposition', 'attachment; filename="file.pdf"')
@Header('Content-Type', 'application/pdf')
download(@Param('id') id: string) {
  return this.filesService.getFile(id);
}
```

## @Redirect — 리다이렉트 ⭐️⭐️

```typescript
import { Redirect } from '@nestjs/common';

@Get('docs')
@Redirect('https://docs.nestjs.com', 302)  // 기본 302
redirectToDocs() {}

// 동적으로 리다이렉트 URL 결정
@Get('version')
@Redirect('https://docs.nestjs.com', 302)
getVersion() {
  return { url: 'https://docs.nestjs.com/v5/', statusCode: 301 };
  // return 값으로 @Redirect 덮어쓰기 가능
}
```

## @Req() — Express Request 직접 접근 ⭐️⭐️⭐️

```typescript
import { Req } from '@nestjs/common';
import type { Request } from 'express';  // ← 반드시 express에서
//            ↑ import type 권장 — 런타임 번들에서 제거됨

@Get()
findAll(@Req() req: Request) {
  console.log(req.headers.authorization);  // 헤더
  console.log(req.cookies.refreshToken);   // 쿠키
  console.log(req.user);                   // Guard가 저장한 payload
  console.log(req.ip);                     // 클라이언트 IP
}
```

```txt
⚠️ Request import 주의:
  import type { Request } from 'express'   ← 올바름
  import type { Request } from 'node-fetch' or 글로벌 Fetch Request ← 잘못됨

  글로벌 Fetch API의 Request를 쓰면:
  req.user, req.cookies 같은 Express 전용 속성이 타입에 없음
  → TypeScript 에러 발생
  반드시 'express'에서 import해야 함

@Req()가 필요한 경우:
  cookies — req.cookies로 직접 읽을 때
  ip — req.ip, req.headers['x-forwarded-for']
  headers — 커스텀 헤더를 읽을 때
  session — req.session으로 세션 접근할 때

보통은 @Req() 없이 전용 데코레이터로 해결:
  req.user.sub   → @UserId() / @OptionalUserId()
  req.body       → @Body()
  req.params.id  → @Param('id')
  req.query.page → @Query('page')
  → @Req()는 전용 데코레이터가 없는 것을 꺼낼 때만

@Req() vs @Headers():
  @Req() req: Request → req.headers 전체 접근
  @Headers('authorization') auth: string → 특정 헤더 하나만

실전 사용 예:
  쿠키에서 refreshToken 읽기 → req.cookies.refreshToken
  클라이언트 IP 로깅 → req.ip
  세션에서 OAuth 상태 읽기 → req.session.oauthNext
```

## @Res() — Express Response 직접 (잘 안 씀) ⭐️⭐️

```typescript
import { Res } from '@nestjs/common';
import { Response } from 'express';

@Get()
findAll(@Res() res: Response) {
  // Express res 객체를 직접 사용
  res.status(200).json({ data: 'hello' });
}
```

```txt
@Req() vs @Res() 차이:
  @Req() → 요청(Request)을 읽기만 함 → NestJS 동작에 영향 없음
  @Res() → 응답(Response)을 직접 제어 → NestJS 자동 응답 처리가 꺼짐

@Res()를 직접 쓰면:
  NestJS의 자동 응답 처리가 비활성화됨
  res.send() 또는 res.json()을 직접 호출해야 함
  Interceptor, 예외 필터 등 NestJS 기능과 호환이 떨어짐
  → 거의 쓸 일 없음

예외적으로 쓰는 경우:
  파일 스트리밍 (res.pipe())
  쿠키 직접 설정 (res.cookie())
  SSE(Server-Sent Events)

쿠키를 설정해야 한다면:
  @Res({ passthrough: true }) 옵션으로 NestJS 자동 처리 유지하면서 res도 사용 가능
```

---

# Guard 적용 ⭐️⭐️⭐️⭐️

```typescript
// JwtAuthGuard가 전역(APP_GUARD)으로 설정되어 있다면 → 기본적으로 모든 라우트 인증 필요
// @Public()을 붙이면 해당 라우트만 인증 생략

@Controller('posts')
export class PostsController {

  @Get()
  @Public()                   // 비로그인도 볼 수 있는 목록
  findAll(@Query() query: ListPostsQueryDto) {
    return this.postsService.findAll(query);
  }

  @Get(':id')
  @Public()                   // 비로그인도 볼 수 있는 단건
  findOne(@Param('id', ParseUUIDPipe) id: string) {
    return this.postsService.findOne(id);
  }

  @Post()                     // 로그인 필수 (전역 Guard가 처리)
  create(
    @UserId()          userId: string,
    @Body() dto:       CreatePostDto,
  ) {
    return this.postsService.create(userId, dto);
  }

  @Delete(':id')
  @Roles('admin')             // 어드민만 삭제 가능
  remove(@Param('id', ParseUUIDPipe) id: string) {
    return this.postsService.remove(id);
  }
}
```

---

# 전형적인 CRUD 컨트롤러 ⭐️⭐️⭐️⭐️

```typescript
@ApiTags('posts')
@Controller('posts')
export class PostsController {
  constructor(private readonly postsService: PostsService) {}

  // 목록 조회 — 비로그인 가능
  @ApiOperation({ summary: '게시글 목록' })
  @Get()
  @Public()
  findAll(@Query() query: ListPostsQueryDto) {
    return this.postsService.findAll(query);
  }

  // 단건 조회 — 비로그인 가능
  @ApiOperation({ summary: '게시글 단건' })
  @Get(':id')
  @Public()
  findOne(@Param('id', ParseUUIDPipe) id: string) {
    return this.postsService.findOne(id);
  }

  // 생성 — 로그인 필수
  @ApiOperation({ summary: '게시글 작성' })
  @Post()
  @HttpCode(201)
  create(
    @UserId()    userId: string,
    @Body() dto: CreatePostDto,
  ) {
    return this.postsService.create(userId, dto);
  }

  // 수정 — 로그인 + 본인만
  @ApiOperation({ summary: '게시글 수정' })
  @Patch(':id')
  update(
    @UserId()              userId: string,
    @Param('id', ParseUUIDPipe) id: string,
    @Body() dto:           UpdatePostDto,
  ) {
    return this.postsService.update(id, userId, dto);
  }

  // 삭제 — 로그인 + 본인만
  @ApiOperation({ summary: '게시글 삭제' })
  @Delete(':id')
  @HttpCode(204)
  async remove(
    @UserId()              userId: string,
    @Param('id', ParseUUIDPipe) id: string,
  ) {
    await this.postsService.remove(id, userId);
  }
}
```

---

# 컨트롤러 vs 서비스 — 뭘 어디에 ⭐️⭐️⭐️⭐️

```txt
컨트롤러에 있어야 할 것:
  @Param, @Query, @Body로 데이터 꺼내기
  @UserId()로 로그인 사용자 확인
  Service 메서드 호출
  return으로 응답 반환
  @HttpCode, @Header로 응답 제어

서비스에 있어야 할 것:
  Prisma 쿼리 (DB 조회·생성·수정·삭제)
  비즈니스 로직 (권한 확인, 계산, 조건 분기)
  이메일 발송, 파일 처리 등 부수 작업
  다른 Service 호출

잘못된 예 (컨트롤러가 너무 두꺼운 경우):
  @Post()
  async create(@Body() dto: CreatePostDto) {
    const user = await this.prisma.user.findUnique(...);  // ❌ DB 직접 접근
    if (!user.isActive) throw new ForbiddenException();   // ❌ 비즈니스 로직
    const post = await this.prisma.post.create(...);      // ❌ DB 직접 접근
    await this.emailService.sendNotification(...);        // ❌ 부수 작업
    return post;
  }

올바른 예:
  @Post()
  create(@UserId() userId: string, @Body() dto: CreatePostDto) {
    return this.postsService.create(userId, dto);  // ✅ 서비스에 전부 위임
  }
```

---

# 자주 쓰는 패턴

## 선택적 인증 — 비로그인도 되지만 로그인하면 개인화

```typescript
@Get()
@UseGuards(OptionalJwtAuthGuard)      // 토큰 있으면 검증, 없어도 통과
findAll(
  @Query()             query:          ListPostsQueryDto,
  @OptionalUserId()    viewerUserId?:  string,   // 비로그인이면 undefined
) {
  return this.postsService.findAll(query, viewerUserId);
}
```

## 중첩 리소스 — 부모 리소스의 자식

```typescript
// /rooms/:roomId/messages
@Controller('rooms')
export class RoomsController {
  @Get(':roomId/messages')
  findMessages(
    @Param('roomId', ParseUUIDPipe) roomId: string,
    @Query() query: ListMessagesQueryDto,
  ) {
    return this.roomsService.findMessages(roomId, query);
  }

  @Post(':roomId/messages')
  createMessage(
    @UserId()                       userId:  string,
    @Param('roomId', ParseUUIDPipe) roomId:  string,
    @Body() dto:                    CreateMessageDto,
  ) {
    return this.roomsService.createMessage(roomId, userId, dto);
  }
}
```

## 토글 · 좋아요 패턴

```typescript
// POST /posts/:id/like (좋아요 토글)
@Post(':id/like')
toggleLike(
  @UserId()              userId: string,
  @Param('id', ParseUUIDPipe) id: string,
) {
  return this.postsService.toggleLike(id, userId);
}
```

## 외부 시스템 cron 엔드포인트 — x-cron-secret 패턴

```txt
x-cron-secret이란:
  HTTP 요청 헤더에 담아 보내는 공유 비밀키(shared secret)
  "이 요청은 신뢰할 수 있는 시스템(GitHub Actions 등)이 보낸 것"을 증명

  왜 JWT 대신 secret 헤더인가:
    JWT는 "로그인한 유저"가 발급받는 토큰 → GitHub Actions는 유저가 아님
    cron 작업은 서버끼리의 통신 → 사전에 공유한 비밀키 비교로 충분
    구조: 서버 .env ↔ GitHub Secret → curl 요청 헤더 → 서버에서 비교 → 불일치 시 401

  x- 접두사:
    HTTP 표준 헤더가 아닌 "커스텀 헤더"를 나타내는 관례
    Authorization이나 Cookie 같은 표준 인증 헤더와 구분
    이름은 자유롭게 지정 (x-api-key, x-cron-secret 등)
```

```typescript
// ❌ 안티패턴 — 메서드마다 검증 로직 반복
@Post('seed-pool/cron')
seedPoolCron(@Headers('x-cron-secret') secret: string | undefined) {
  if (!process.env.CRON_SECRET || secret !== process.env.CRON_SECRET) {
    throw new UnauthorizedException('잘못된 cron secret입니다.');
  }
  // ...
}
```

```typescript
// ✅ Guard로 추출 — 범용적으로 재사용
// cron-secret.guard.ts
@Injectable()
export class CronSecretGuard implements CanActivate {
  canActivate(context: ExecutionContext): boolean {
    const req = context.switchToHttp().getRequest<Request>();
    const secret = req.headers['x-cron-secret'];
    if (!process.env.CRON_SECRET || secret !== process.env.CRON_SECRET) {
      throw new UnauthorizedException('잘못된 cron secret입니다.');
    }
    return true;
  }
}
```

## ⚠️ 전역 JWT Guard가 있으면 @Public() 필수 ⭐️⭐️⭐️⭐️

```txt
문제:
  JwtAuthGuard가 APP_GUARD(전역)로 등록되면 모든 라우트에 JWT 검사가 먼저 실행됨
  GitHub Actions는 JWT 토큰이 없음 → JWT Guard에서 401 반환
  → CronSecretGuard까지 도달하지 못하고 차단됨

  증상: Railway에 요청은 도달하는데 401 → curl --fail-with-body → Actions 실패

해결:
  @Public()을 붙여서 JWT Guard를 스킵
  대신 @UseGuards(CronSecretGuard)로 자체 인증

  Guard 실행 순서:
    전역 JWT Guard → @Public() 확인 → true면 스킵
    메서드 레벨 CronSecretGuard → x-cron-secret 검증 → 통과
```

```typescript
// ✅ 전역 JWT Guard + CronSecretGuard 조합 — 완성형 패턴
@Public()                    // ① JWT Guard 스킵 (GitHub Actions는 유저 토큰 없음)
@UseGuards(CronSecretGuard)  // ② x-cron-secret으로 자체 인증
@Post('seed-pool/cron')
seedPoolCron(
  @Query('pages', new DefaultValuePipe(3), ParseIntPipe) pages: number,
) {
  return this.someService.execute(pages);
}
```

```typescript
// @Public() 데코레이터 구현 (보통 common/decorators/public.decorator.ts)
import { SetMetadata } from '@nestjs/common';

export const IS_PUBLIC_KEY = 'isPublic';
export const Public = () => SetMetadata(IS_PUBLIC_KEY, true);
```

```typescript
// JwtAuthGuard — IS_PUBLIC_KEY를 확인해서 스킵 여부 판단
@Injectable()
export class JwtAuthGuard extends AuthGuard('jwt') {
  constructor(private reflector: Reflector) {
    super();
  }

  canActivate(context: ExecutionContext) {
    const isPublic = this.reflector.getAllAndOverride<boolean>(IS_PUBLIC_KEY, [
      context.getHandler(),   // 메서드 레벨 데코레이터 먼저
      context.getClass(),     // 클래스 레벨 데코레이터
    ]);
    if (isPublic) return true;  // @Public() 있으면 JWT 검사 스킵
    return super.canActivate(context);
  }
}
```

```txt
패턴 요약:
  .env          CRON_SECRET="랜덤 긴 문자열 (openssl rand -hex 32 등)"
  GitHub Secret 동일한 값 등록
  curl 요청     -H "x-cron-secret: ${{ secrets.CRON_SECRET }}"
  서버 검증     @Public() → JWT 스킵 → CronSecretGuard → secret 비교 → 불일치 시 401

  ⚠️ @Public()만 붙이고 @UseGuards(CronSecretGuard)를 안 붙이면
     인증 없이 누구나 호출 가능 — 반드시 세트로

GitHub Actions 워크플로우 전체 → [[GitHub_Actions]]
```

---

# 자주 만나는 에러

| 에러                  | 원인                                | 해결                                        |
| ------------------- | --------------------------------- | ----------------------------------------- |
| `Cannot GET /경로`    | 라우트 등록 안 됨 또는 경로 오타               | `@Controller` + `@Get` 경로 확인              |
| `@Param`이 undefined | 경로 파라미터 이름 불일치                    | `@Get(':id')`의 `id`와 `@Param('id')` 일치 확인 |
| `@Body()`가 빈 객체     | Content-Type 헤더 누락                | 요청 시 `Content-Type: application/json` 확인  |
| ParseUUIDPipe 에러    | UUID 형식 아닌 값이 경로에 옴               | 클라이언트에서 올바른 UUID 전달                       |
| 401 자동 반환           | JwtAuthGuard 전역 설정인데 @Public() 누락 | 공개 라우트에 `@Public()` 추가                    |