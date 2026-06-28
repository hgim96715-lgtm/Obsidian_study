---
aliases: [커서 기반, 페이지 기반, 페이지네이션, findAndCount, Pagination]
tags:
  - NestJS
related:
  - "[[00_NestJS_Ecosystem_HomePage]]"
  - "[[NestJS_Controller]]"
  - "[[NestJS_TypeORM_QueryBuilder]]"
  - "[[NestJS_TypeORM_Repository]]"
---
# NestJS_Pagination — 페이지네이션

## 한 줄 요약

```txt
Pagination = 많은 데이터를 부분적으로 나눠서 불러오는 기술
Page Based   → 페이지 번호 기반
Cursor Based → 마지막 id 기반 (무한 스크롤)
```

---

---

# ① Pagination 종류 ⭐️

## Page Based

```txt
페이지 번호 + 개수로 데이터 분할
GET /movies?page=1&take=20

1페이지: 1~20번
2페이지: 21~40번
```

```txt
문제점 ⚠️:
  데이터 추가/삭제 시 페이지 경계 달라짐
  page=1000 → DB 가 1~20000번 다 읽고 버림 (비효율)
```

## Cursor Based ⭐️

```txt
마지막으로 불러온 id 기준으로 다음 데이터 가져오기
GET /movies?cursor=20&take=20  → id < 20 인 것 20개

장점: 데이터 추가/삭제에 안정적 / 대용량 빠름
단점: 특정 페이지 점프 불가 / 무한 스크롤에 적합
```

---

---

#  DTO 로 받기 ⭐️

```txt
@Query('page', ParseIntPipe) 방식 vs @Query() dto 방식

  개별 파라미터 방식:
    @Query('page', new DefaultValuePipe(1), ParseIntPipe) page: number
    → 파라미터 하나씩 / 유효성 검사 불편

  DTO 방식 (권장):
    @Query() dto: PagePaginationDto
    → 여러 파라미터 한번에 / class-validator 적용 / 재사용 가능
```

## PagePaginationDto

```typescript
// common/dto/page-pagination.dto.ts
import { IsInt, IsOptional } from 'class-validator';

export class PagePaginationDto {
  @IsInt()
  @IsOptional()
  page: number = 1;    // 기본값 1

  @IsInt()
  @IsOptional()
  take: number = 5;    // 기본값 5 (한 번에 가져올 개수)
}
```

```txt
page: 몇 번째 페이지인지
take: 한 번에 가져올 개수 (limit 과 같은 의미)

@IsOptional() = 없으면 기본값(= 1, = 5) 사용
@IsInt()      = 있으면 정수인지 검사
```

## CursorPaginationDto

```typescript
// common/dto/cursor-pagination.dto.ts
import { IsInt, IsOptional } from 'class-validator';

export class CursorPaginationDto {
  @IsInt()
  @IsOptional()
  cursor?: number;     // 마지막으로 받은 id (없으면 처음부터)

  @IsInt()
  @IsOptional()
  take: number = 5;
}
```

## 컨트롤러에서 사용

```typescript
// movie.controller.ts
@Get()
getMovies(
  @Request() req: any,
  @Query() dto: PagePaginationDto,   // ← DTO 로 한번에 받기
) {
  return this.movieService.findAll(dto);
}
```

---

---

#  qb.take() / qb.skip() 란 ⭐️

```txt
QueryBuilder 에서 페이지네이션 설정:
  qb.take(n)  → LIMIT n (n개 가져오기)
  qb.skip(n)  → OFFSET n (n개 건너뛰기)

skip 계산:
  page=1 → skip = (1-1) * take = 0  (처음부터)
  page=2 → skip = (2-1) * take = 5  (5개 건너뛰고)
  page=3 → skip = (3-1) * take = 10 (10개 건너뛰고)
```

```typescript
const { page, take } = dto;
const skip = (page - 1) * take;

qb.take(take)   // LIMIT take
qb.skip(skip)   // OFFSET skip
```

---

---

# CommonService — 공통 페이지네이션 로직 ️

```txt
페이지네이션은 movie / genre / director 등 여러 곳에서 반복됨
→ CommonModule / CommonService 로 분리해서 재사용
```

>[[NestJS_Module#커스텀 모듈 만들기 ⭐️]] 참고 

## 폴더 구조

```txt
src/
  common/
    common.module.ts
    common.service.ts
    dto/
      page-pagination.dto.ts
      cursor-pagination.dto.ts
```

## CommonService

```typescript
// common/common.service.ts
import { Injectable } from '@nestjs/common';
import { ObjectLiteral, SelectQueryBuilder } from 'typeorm';
import { PagePaginationDto } from './dto/page-pagination.dto';

@Injectable()
export class CommonService {

  applyPagePaginationParamsToQb<T extends ObjectLiteral>(
    qb: SelectQueryBuilder<T>,
    dto: PagePaginationDto,
  ) {
    const { page, take } = dto;

    if (take && page) {               // take 와 page 가 있을 때만 적용
      const skip = (page - 1) * take;
      qb.take(take);                  // LIMIT
      qb.skip(skip);                  // OFFSET
    }
  }
}
```

```txt
<T extends ObjectLiteral>:
  TypeScript 제네릭
  어떤 Entity 의 QueryBuilder 에도 사용 가능
  Movie / Genre / Director 등 모두 적용

if (take && page):
  둘 다 있어야 skip 계산 가능
  하나라도 없으면 페이지네이션 적용 안 함

왜 CommonService 로 분리하나:
  movie.service / genre.service 등 여러 곳에서 동일 로직 필요
  CommonService 에 한번 만들어두면 주입해서 재사용
```

## CommonModule

```typescript
// common/common.module.ts
import { Module } from '@nestjs/common';
import { CommonService } from './common.service';

@Module({
  providers: [CommonService],
  exports: [CommonService],   // ← 다른 모듈에서 주입받으려면 export 필요
})
export class CommonModule {}
```

## 다른 모듈에서 사용

```typescript
// movie.module.ts
@Module({
  imports: [CommonModule],   // ← CommonModule import
  providers: [MovieService],
  controllers: [MovieController],
})
export class MovieModule {}

// movie.service.ts
@Injectable()
export class MovieService {
  constructor(
    @InjectRepository(Movie)
    private readonly movieRepository: Repository<Movie>,
    private readonly commonService: CommonService,   // ← 주입
  ) {}

  async findAll(dto: PagePaginationDto) {
    const qb = this.movieRepository.createQueryBuilder('movie')
      .orderBy('movie.id', 'DESC');

    this.commonService.applyPagePaginationParamsToQb(qb, dto);
    // ↑ 한 줄로 take / skip 적용

    return qb.getMany();
  }
}
```

---
---
# 실제 SQL — Page vs Cursor 비교 ⭐️

## Page Based SQL

```sql
-- 1페이지: 처음 5개 (id 오름차순)
SELECT id, "likeCount", title
FROM movie
ORDER BY id ASC
LIMIT 5;

-- 2페이지: 5개 건너뛰고 (likeCount 내림차순)
SELECT id, "likeCount", title
FROM movie
ORDER BY "likeCount" DESC, id DESC
LIMIT 5
OFFSET 5;
```

```txt
OFFSET 5 = 앞 5개 건너뛰고 → 6번째부터 10번째
page=2, take=5 → OFFSET = (2-1) * 5 = 5
```

## Cursor Based SQL ⭐️

```sql
-- 단일 컬럼 cursor (id 기준)
SELECT id, "likeCount", title
FROM movie
WHERE id < 252          -- 커서 이후 (내림차순)
ORDER BY id DESC
LIMIT 5;
```

### 다중 컬럼 cursor — OR 방식

```sql
-- likeCount DESC, id DESC 정렬 기준
-- cursor = (likeCount=20, id=35)
SELECT id, "likeCount", title
FROM movie
WHERE "likeCount" < 20                       -- likeCount 가 더 작거나
   OR ("likeCount" = 20 AND id < 35)        -- likeCount 같으면 id 로 비교
ORDER BY "likeCount" DESC, id DESC
LIMIT 5;
```

### 다중 컬럼 cursor — 튜플 비교 방식 (PostgreSQL 전용)

```sql
-- 위 OR 쿼리와 완전 동일 / 더 간결
SELECT id, "likeCount", title
FROM movie
WHERE ("likeCount", id) < (20, 35)
--     ↑ 튜플 비교 = (likeCount < 20) OR (likeCount = 20 AND id < 35)
ORDER BY "likeCount" DESC, id DESC
LIMIT 5;
```

>[[PG_Specific#튜플 비교 — 다중 컬럼]] 참고  

```txt
튜플 비교 (PostgreSQL 전용):
  (a, b) < (x, y)
  = a < x OR (a = x AND b < y)

  OR 여러 개 쓰는 것과 같은 결과 / 더 간결
  MySQL 에서는 동작 안 함 → OR 방식 사용

왜 다중 컬럼 cursor 가 필요한가:
  ORDER BY likeCount DESC, id DESC 정렬 시
  likeCount 값이 같은 행이 여러 개 있을 수 있음
  → id 만 cursor 로 쓰면 같은 likeCount 내 순서 불명확
  → (likeCount, id) 두 값으로 정확히 다음 위치 특정

cursor 값은 프론트에서 전달:
  첫 요청 → cursor 없음 → 처음부터
  이후 요청 → 마지막으로 받은 (likeCount, id) 를 cursor 로 전달
```

---

---

# ⑤ Page vs Cursor 선택 기준

| |Page Based|Cursor Based|
|---|---|---|
|특정 페이지 이동|✅|❌|
|무한 스크롤|△|✅|
|데이터 추가/삭제 안정성|❌|✅|
|대용량 성능|❌|✅|
|구현 복잡도|간단|중간|

```txt
게시판 / 검색 결과 → Page Based  (페이지 점프 필요)
피드 / 알림 / 채팅 → Cursor Based (무한 스크롤)
```