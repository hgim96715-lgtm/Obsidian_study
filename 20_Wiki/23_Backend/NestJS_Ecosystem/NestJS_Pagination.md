---
aliases:
  - Cursor
  - Offset
  - query
tags:
  - NestJS
related:
  - "[[00_NestJS_Ecosystem_HomePage]]"
  - "[[NestJS_Controller]]"
  - "[[NestJS_DTO]]"
  - "[[NestJS_Prisma]]"
  - "[[JS_Array_Methods]]"
  - "[[JS_URL_Encoding]]"
---
# NestJS_Pagination — 페이지네이션

> [!info]
>  페이지네이션 = 대량 데이터를 나눠서 가져오는 방식. 
>  오프셋 방식(page/limit)과 커서 방식(cursor) 두 가지 — 실시간 피드는 커서, 관리 목록은 오프셋이 적합하다.

---

# 오프셋 vs 커서 ⭐️⭐️⭐️⭐️

| |오프셋 (Offset)|커서 (Cursor)|
|---|---|---|
|방식|`skip = (page-1) * limit`|마지막 ID를 기준으로 다음 항목|
|URL|`?page=2&limit=20`|`?cursor=abc123&limit=20`|
|장점|구현 단순, 특정 페이지 이동 가능|실시간 데이터에서 중복/누락 없음|
|단점|데이터 추가 시 페이지 어긋남|특정 페이지 이동 불가|
|언제|관리자 목록, 검색 결과|피드, 채팅, 무한 스크롤|

---

# 커서 + 검색 + 관계 필터 — 실전 패턴 ⭐️⭐️⭐️⭐️

```typescript
// DTO
export class ListMembersQueryDto {
  @IsOptional()
  @IsString()
  @MaxLength(40)
  q?: string;          // 닉네임 검색

  @IsOptional()
  @IsUUID()
  cursor?: string;     // 이전 페이지 마지막 항목 id

  @IsOptional()
  @Type(() => Number)
  @IsInt()
  @Min(1)
  @Max(50)
  limit?: number;
}
```

```typescript
// Service
async listMembers(
  roomId: string,
  userId: string,
  query: { q?: string; cursor?: string; limit?: number } = {},
) {
  // 1. limit 안전하게 처리 — 클라이언트 값을 그대로 믿지 않음
  const take = Math.min(Math.max(query.limit ?? 30, 1), 50);
  const q    = query.q?.trim();

  // 2. 관계 모델 필드로 검색 — user.nickname
  const where = {
    roomId,
    ...(q
      ? {
          user: {
            nickname: { contains: q, mode: 'insensitive' as const },
            //                              ↑ as const 필요 (아래 설명)
          },
        }
      : {}),
  };

  // 3. 전체 수 + 데이터 동시 조회
  const [total, rows] = await Promise.all([
    this.prisma.roomMember.count({ where }),
    this.prisma.roomMember.findMany({
      where,
      take: take + 1,  // 다음 페이지 존재 확인용
      ...(query.cursor
        ? { cursor: { id: query.cursor }, skip: 1 }
        : {}),
      orderBy: [
        { joinedAt: 'asc' },   // 1차: 입장 시각
        { id:       'asc' },   // 2차: id (동시 입장 시 안정 정렬)
      ],
      include: {
        user: { select: { id: true, nickname: true, image: true } },
      },
    }),
  ]);

  // 4. hasMore 판단 + 여분 제거
  const hasMore = rows.length > take;
  const page    = hasMore ? rows.slice(0, take) : rows;

  // 5. 첫 페이지에서만 owner를 앞으로
  if (!query.cursor) {
    page.sort((a, b) => {
      if (a.role === RoomMemberRole.owner && b.role !== RoomMemberRole.owner) return -1;
      if (b.role === RoomMemberRole.owner && a.role !== RoomMemberRole.owner) return 1;
      return a.joinedAt.getTime() - b.joinedAt.getTime();
    });
  }

  return {
    items:      page,
    nextCursor: hasMore ? (page[page.length - 1]?.id ?? null) : null,
    total,
  };
}
```

## 패턴 설명

```txt
Math.min(Math.max(query.limit ?? 30, 1), 50):
  클라이언트가 limit=0 이나 limit=9999 를 보낼 수 있음
  Math.max(..., 1)   → 최솟값 1 보장
  Math.min(..., 50)  → 최댓값 50 보장
  ?? 30              → 없으면 기본값 30

mode: 'insensitive' as const:
  Prisma의 mode 타입은 Prisma.QueryMode (리터럴 유니온)
  'insensitive' 를 그냥 쓰면 TS가 string으로 추론 → Prisma 타입 불일치 에러
  as const 로 리터럴 타입 'insensitive'로 좁혀야 에러 없음
  → [[TS_Type_Guards]] as const 참고

관계 모델 필드 검색 (user.nickname):
  where.user.nickname.contains — 중첩 관계의 필드로 필터링
  MemberModel.findMany 에서 User.nickname으로 검색 가능
  → [[NestJS_Prisma]] "관계 필터" 참고

복합 orderBy [{ joinedAt }, { id }]:
  joinedAt만으로는 동시 입장한 멤버의 순서가 불안정
  id(UUID/CUID)를 2차 기준으로 추가 → 항상 같은 순서 보장 (안정 정렬)
  커서 페이지네이션에서 순서가 달라지면 중복/누락 발생 → 필수

cursor 없을 때만 owner 정렬:
  커서 페이지에서 다시 정렬하면 중간 페이지의 순서가 꼬임
  첫 페이지(!query.cursor)에서만 특별 정렬 → 2페이지부터는 DB 정렬 그대로
  [[JS_Array_Methods#sort — 비교 함수 규칙 ⭐️⭐️⭐️⭐️]] 참고 

page[page.length - 1]?.id ?? null:
  hasMore이면 마지막 항목의 id → 다음 요청의 cursor
  items가 비어있을 경우(예외 상황) ?? null 로 안전하게
```

---

# 오프셋 페이지네이션 ⭐️⭐️⭐️⭐️

## Query DTO

```typescript
import { IsOptional, IsInt, Min, Max } from 'class-validator';
import { Type } from 'class-transformer';

export class PaginationDto {
  @IsOptional()
  @IsInt()
  @Min(1)
  @Type(() => Number)
  page?: number = 1;

  @IsOptional()
  @IsInt()
  @Min(1)
  @Max(100)
  @Type(() => Number)
  limit?: number = 20;
}
```

## Service

```typescript
async findAll(dto: PaginationDto) {
  const { page = 1, limit = 20 } = dto;
  const skip = (page - 1) * limit;

  const [items, total] = await Promise.all([
    this.prisma.post.findMany({
      skip,
      take:    limit,
      orderBy: { createdAt: 'desc' },
    }),
    this.prisma.post.count(),
  ]);

  return {
    data: items,
    meta: {
      total,
      page,
      limit,
      totalPages: Math.ceil(total / limit),
      hasNext:    page < Math.ceil(total / limit),
      hasPrev:    page > 1,
    },
  };
}
```

## 응답 형태

```json
{
  "data": [...],
  "meta": {
    "total": 142,
    "page": 2,
    "limit": 20,
    "totalPages": 8,
    "hasNext": true,
    "hasPrev": true
  }
}
```

```txt
Promise.all([findMany, count]):
  count 쿼리와 데이터 쿼리를 동시에 실행 — 순차 실행보다 빠름
  total이 없으면 totalPages/hasNext 계산 불가

skip = (page - 1) * limit:
  page 1 → skip 0  (처음부터)
  page 2 → skip 20 (21번째부터)
  page 3 → skip 40 (41번째부터)
```

---

# 커서 페이지네이션 ⭐️⭐️⭐️⭐️

## Query DTO

```typescript
export class CursorPaginationDto {
  @IsOptional()
  @IsString()
  cursor?: string;  // 마지막으로 받은 항목의 ID

  @IsOptional()
  @IsInt()
  @Min(1)
  @Max(50)
  @Type(() => Number)
  limit?: number = 20;
}
```

## Service

```typescript
async findAll(dto: CursorPaginationDto) {
  const { cursor, limit = 20 } = dto;

  const items = await this.prisma.post.findMany({
    take:    limit + 1,       // 하나 더 가져와서 다음 페이지 존재 확인
    cursor:  cursor ? { id: cursor } : undefined,
    skip:    cursor ? 1 : 0,  // cursor 항목 자체는 제외
    orderBy: { createdAt: 'desc' },
  });

  const hasNext = items.length > limit;
  const data    = hasNext ? items.slice(0, limit) : items;  // 여분 제거
  const nextCursor = hasNext ? data[data.length - 1].id : null;

  return { data, nextCursor, hasNext };
}
```

## 응답 형태

```json
{
  "data": [...],
  "nextCursor": "clh7x...",
  "hasNext": true
}
```

```txt
take: limit + 1 트릭:
  20개가 필요하면 21개를 요청
  21개가 오면 → 다음 페이지 있음 (hasNext = true), 실제 반환은 20개
  20개 이하면 → 마지막 페이지 (hasNext = false)

skip: cursor ? 1 : 0:
  cursor가 있으면 cursor 항목 자체를 건너뜀 (이미 클라이언트에 있으므로)
  cursor가 없으면 (첫 요청) 처음부터 시작

nextCursor:
  클라이언트가 다음 요청에서 cursor=nextCursor 로 전달
  null이면 마지막 페이지
```

---

# 검색 + 필터 조합 ⭐️⭐️⭐️

```typescript
export class SearchDto extends PaginationDto {
  @IsOptional()
  @IsString()
  @MaxLength(100)
  keyword?: string;

  @IsOptional()
  @IsEnum(PostStatus)
  status?: PostStatus;
}

async search(dto: SearchDto) {
  const { page = 1, limit = 20, keyword, status } = dto;

  const where: Prisma.PostWhereInput = {
    ...(keyword && {
      OR: [
        { title:   { contains: keyword, mode: 'insensitive' } },
        { content: { contains: keyword, mode: 'insensitive' } },
      ],
    }),
    ...(status && { status }),
  };

  const [items, total] = await Promise.all([
    this.prisma.post.findMany({
      where,
      skip:    (page - 1) * limit,
      take:    limit,
      orderBy: { createdAt: 'desc' },
    }),
    this.prisma.post.count({ where }),
  ]);

  return {
    data:  items,
    meta: { total, page, limit, totalPages: Math.ceil(total / limit) },
  };
}
```

```txt
...(condition && { field: value }) 패턴:
  condition이 falsy면 빈 객체 {} → where에 아무 효과 없음
  condition이 truthy면 { field: value }가 스프레드됨
  → 선택적 필터를 깔끔하게 조합 → [[NestJS_Prisma]] 참고

mode: 'insensitive':
  대소문자 무시 검색 (PostgreSQL)
  MySQL에서는 기본적으로 대소문자 무시라 mode 불필요
```

---

# Controller

```typescript
@ApiTags('posts')
@Controller('posts')
export class PostController {
  @ApiOkResponse({ type: PostListResponseDto })
  @Get()
  findAll(@Query() dto: PaginationDto) {
    return this.postService.findAll(dto);
  }

  @Get('search')
  search(@Query() dto: SearchDto) {
    return this.postService.search(dto);
  }
}
```

---

# 한눈에

```txt
오프셋:
  skip = (page - 1) * limit
  Promise.all([findMany, count])  → 동시 실행
  응답: { data, meta: { total, page, limit, totalPages, hasNext, hasPrev } }

커서:
  take: limit + 1  → 다음 페이지 존재 확인 트릭
  cursor ? { id: cursor } / skip: cursor ? 1 : 0
  응답: { data, nextCursor, hasNext }

선택 기준:
  관리 목록 / 검색 / 특정 페이지 이동 필요  → 오프셋
  피드 / 채팅 / 무한 스크롤                 → 커서

검색 + 필터:
  ...(keyword && { OR: [...] }) — 조건부 where 스프레드
  count({ where })  — 같은 where로 전체 개수
```
