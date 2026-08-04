---
aliases:
  - Body
  - Controller
  - createParamDecorator
  - Param
  - Pipe
  - Query
tags:
  - NestJS
related:
  - "[[00_NestJS_Ecosystem_HomePage]]"
  - "[[NestJS_Service_Provider]]"
  - "[[NestJS_Swagger]]"
  - "[[NestJS_Pipe]]"
  - "[[NestJS_Concept]]"
  - "[[NestJS_JwtGuard]]"
---
# NestJS_Controller — 컨트롤러

> [!info] 
> 컨트롤러 = HTTP 요청을 받아서 응답을 돌려주는 클래스.
>  URL 경로와 HTTP 메서드를 핸들러 함수에 연결한다. 
>  비즈니스 로직은 Service에 위임하고, 컨트롤러는 "요청 받기 → 서비스 호출 → 응답 반환" 세 가지만 한다. 
>  DTO → [[NestJS_DTO]], Pipe → [[NestJS_Pipe]], Guard/인증 → [[NestJS_JwtGuard]]

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

# 응답 제어 ⭐️⭐️⭐️

## 기본 — return 값이 자동으로 JSON

```typescript
@Get(':id')
findOne(@Param('id', ParseUUIDPipe) id: string) {
  return this.postsService.findOne(id);
  // 객체를 return하면 자동으로 JSON.stringify → 200 응답
}
```

## 상태 코드 변경 — @HttpCode

```typescript
@Post()
@HttpCode(201)  // POST 성공 시 201 Created (기본값은 200)
create(@Body() dto: CreatePostDto) {
  return this.postsService.create(dto);
}

@Delete(':id')
@HttpCode(204)  // DELETE 성공 시 204 No Content
async remove(@Param('id', ParseUUIDPipe) id: string) {
  await this.postsService.remove(id);
  // 204면 body가 없음 → return 안 해도 됨
}
```

```txt
주요 상태 코드:
  200  OK — GET · PATCH 기본
  201  Created — POST 성공 시 (생성됨)
  204  No Content — DELETE 성공 시 (응답 body 없음)
  400  Bad Request — 요청 형식 오류 (ValidationPipe 자동 반환)
  401  Unauthorized — 인증 안 됨
  403  Forbidden — 권한 없음
  404  Not Found — 리소스 없음
  409  Conflict — 중복 등 충돌
  500  Internal Server Error — 서버 에러
```

## 헤더 설정 — @Header

```typescript
@Get(':id/download')
@Header('Content-Disposition', 'attachment; filename="file.pdf"')
download(@Param('id') id: string) {
  return this.filesService.getFile(id);
}
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

---

# 자주 만나는 에러

| 에러                  | 원인                                | 해결                                        |
| ------------------- | --------------------------------- | ----------------------------------------- |
| `Cannot GET /경로`    | 라우트 등록 안 됨 또는 경로 오타               | `@Controller` + `@Get` 경로 확인              |
| `@Param`이 undefined | 경로 파라미터 이름 불일치                    | `@Get(':id')`의 `id`와 `@Param('id')` 일치 확인 |
| `@Body()`가 빈 객체     | Content-Type 헤더 누락                | 요청 시 `Content-Type: application/json` 확인  |
| ParseUUIDPipe 에러    | UUID 형식 아닌 값이 경로에 옴               | 클라이언트에서 올바른 UUID 전달                       |
| 401 자동 반환           | JwtAuthGuard 전역 설정인데 @Public() 누락 | 공개 라우트에 `@Public()` 추가                    |