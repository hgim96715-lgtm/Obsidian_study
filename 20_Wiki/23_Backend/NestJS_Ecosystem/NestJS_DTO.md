---
aliases:
  - DTO
  - class-validator
  - ValidationPipe
tags:
  - NestJS
related:
  - "[[00_NestJS_Ecosystem_HomePage]]"
  - "[[NestJS_Swagger]]"
  - "[[NestJS_Controller]]"
  - "[[NestJS_Prisma]]"
  - "[[TS_Type_Guards]]"
---
# NestJS_DTO — DTO & 유효성 검사

# 한 줄 요약

```txt
DTO = Data Transfer Object
요청 Body 의 구조를 클래스로 정의하고 유효성 검사
설치 → ValidationPipe 등록 → 폴더 구조 → DTO 작성 → 사용
```

---

---

# 설치 ⭐️

```bash
# 모노레포 — 항상 루트에서 --filter 패키지명
pnpm add class-validator class-transformer --filter api

# 단일 프로젝트(모노레포 아님)라면
pnpm add class-validator class-transformer
```

```txt
모노레포에서 apps/api 안으로 cd 해서 설치하면 안 되는 이유는
[[NestJS_Prisma]] 의 "모노레포라면 순서 자체가 다름" 과 완전히 동일함 (package.json 덮어쓰기 / 중복 lockfile 위험)
```

---

---

# ValidationPipe 전역 등록 ⭐️

```typescript
// main.ts
app.useGlobalPipes(
  new ValidationPipe({ whitelist: true, forbidNonWhitelisted: true, transform: true }),
);
```

```txt
등록 안 하면 @IsNotEmpty() 같은 데코레이터가 동작 안 함 — main.ts 에 반드시 등록
whitelist/forbidNonWhitelisted/transform 각 옵션의 역할과 보안 시나리오(필드 주입 방지 등)는
[[NestJS_Controller]] 의 "main.ts — ValidationPipe 옵션 (실전 표준 3종)" 에서 이미 다룸 — 여기서는 등록 위치만 재확인
```

---

---

# DTO 란 ⭐️

```txt
@Body() body: any 로 받으면: 어떤 데이터가 들어오는지 타입 불명확, 필드가 많아지면 복잡해짐
DTO 로 해결: 요청 Body 구조를 클래스로 명확히 정의 + class-validator 데코레이터로 검사 + 타입 안전성
```

```mermaid-beautiful
flowchart TD
    A["Client 요청"] --> B["Controller"]
    B --> C["@Body()<br/>Body 데이터 추출"]
    C --> D["DTO<br/>요청 데이터 구조 정의"]
    D --> E["ValidationPipe<br/>유효성 검사"]
    E --> F{"검증 통과?"}
    F -->|YES| G["Service<br/>비즈니스 로직 처리"]
    F -->|NO| H["400 Bad Request<br/>에러 응답"]
    G --> I["Response"]
    H --> I
```

---

---

# DTO 폴더 구조 ⭐️

```txt
모듈별로 자기 dto/ 폴더 안에 둠 (기본):
  src/movie/dto/create-movie.dto.ts
  src/movie/dto/update-movie.dto.ts

여러 모듈에서 똑같이 재사용하는 DTO 는 공통 위치로 (나중에):
  apps/api/src/common/dto/
    pagination.dto.ts   ← 예: 페이지네이션 쿼리(page, take 등)는 거의 모든 모듈에서 씀
```

```txt
판단 기준:
  그 모듈 안에서만 쓰는 DTO        → 그 모듈의 dto/ 폴더
  2개 이상 모듈에서 내용이 동일하게 반복됨 → common/dto/ 로 옮김
  처음부터 모든 DTO 를 common 에 만들 필요는 없음 — 정말 중복되는 게 보이는 시점에 옮기는 게 자연스러움
```

---

---

# DTO 설계 원칙 — Prisma 모델을 그대로 베끼지 않는다 ⭐️⭐️⭐️

```txt
흔한 오해: "DB 테이블(schema.prisma)에 있는 컬럼이니까 DTO 에도 다 넣어야지"
→ 틀림. Create DTO 는 "이 요청을 할 때 클라이언트가 실제로 보내야 하는 값" 만 담음
  schema.prisma 의 컬럼 수와 Create DTO 의 필드 수가 같아야 할 이유는 없음
```

```typescript
// schema.prisma
model Post {
  id        Int      @id @default(autoincrement())
  title     String
  content   String
  hidden    Boolean  @default(false)
  createdAt DateTime @default(now())
  updatedAt DateTime @updatedAt
}
```

```typescript
// CreatePostDto — title, content 만 받음 (나머지 4개 컬럼은 일부러 안 넣음)
export class CreatePostDto {
  @IsNotEmpty()
  @IsString()
  title: string;

  @IsNotEmpty()
  @IsString()
  content: string;
}
```

## 어떤 필드를 빼는가, 그리고 왜 ⭐️⭐️

|필드|왜 클라이언트가 안 보내는가|실제 값은 어디서 오는가|
|---|---|---|
|`id`|아직 생성되기 전이라 존재하지 않는 값 — DB 가 만들어줌|`@default(autoincrement())`/`uuid()`|
|`hidden`|작성 시점에 사용자가 정할 값이 아니라 서비스 정책으로 정해지는 기본 상태값|`@default(false)` 또는 서버 로직|
|`createdAt`|"작성 시각" 은 서버 시계 기준이어야 정확함|`@default(now())`|
|`updatedAt`|"수정 시각" 도 그 순간에 서버가 직접 기록해야 함|`@updatedAt` (자동 갱신)|

```txt
공통 원칙 — "이 값이 누구의 책임인가" 를 먼저 따짐:
  사용자가 직접 입력하는 내용(제목, 본문 등)        → 사용자 책임 → DTO 에 포함
  시스템이 결정/생성하는 값(id, 생성시각, 기본 상태값) → 서버/DB 책임 → DTO 에서 제외
  → "작성 폼에 그 입력칸이 있는가?" 로 바꿔 생각하면 쉬움
```

```txt
createdAt / updatedAt 을 클라이언트가 보내게 두면 안 되는 이유:
  ① 보낼 이유 자체가 없음 — 서버가 요청을 받는 그 순간이 곧 "생성된 시점"
  ② 조작 방지 — 과거/미래 날짜를 직접 써서 "오래된 글처럼" 또는 "정렬 순서 조작" 이 가능해짐
  ③ Prisma 가 이미 자동으로 처리(@default(now()) / @updatedAt) — 애초에 관여할 이유가 없음

id / hidden 도 같은 논리:
  id     아직 안 생긴 값을 클라이언트가 미리 정해서 보낼 이유가 없음 (DB 가 책임)
  hidden 작성자가 마음대로 정하게 하면 정책을 우회할 수 있음(검수 전 게시물을 hidden:false 로 보내는 등)
```

## DTO 에서 빼는 것만으로 충분한가 — whitelist: true 와의 관계 ⭐️⭐️

```txt
DTO 클래스에 createdAt 을 안 적었다고 해서 "클라이언트가 body 에 createdAt 을 직접 끼워 보내는 것" 자체가
막히는 건 아님 (DTO 에 선언을 안 한 것뿐 — HTTP 요청 자체는 여전히 그 값을 포함해서 올 수 있음)
→ 실제로 그 값을 "걸러내는" 역할은 ValidationPipe 의 whitelist: true 가 함
```

```txt
whitelist: true               → DTO 에 정의 안 된 필드(createdAt 포함)는 응답 처리 전에 자동 제거됨
forbidNonWhitelisted: true     → 한 술 더 떠서, 그런 필드가 있으면 그 자리에서 400 에러로 거부

→ DTO 설계(어떤 필드를 클래스에 넣을지) + ValidationPipe 설정(그 외 필드를 실제로 차단)
  이 둘이 항상 세트로 동작해야 "설계한 대로 안전하게" 막힘
```

## 안 넣은 필드는 주석으로 남기는 관례 ⭐️

```typescript
/**
 * 안 넣음: id · hidden · createdAt
 * → forbidNonWhitelisted 가 켜져 있어 body 에 섞여 와도 400으로 차단됨
 */
export class CreatePostDto {
  @IsNotEmpty() @IsString() title: string;
  @IsNotEmpty() @IsString() content: string;
}
```

```txt
"왜 이 필드가 없는지" 를 코드에 남겨두면, 나중에 다른 사람(또는 미래의 나)이
"이거 빠뜨린 거 아닌가?" 하고 다시 추가하는 실수를 막을 수 있음
→ 단순히 안 적은 것과, "의도적으로 제외했다" 는 걸 명시한 것은 유지보수 관점에서 차이가 큼
```

---

---

# DTO 작성 ⭐️

```typescript
// src/movie/dto/create-movie.dto.ts
import { IsNotEmpty, IsString, IsEnum } from 'class-validator';

export class CreateMovieDto {
  @IsNotEmpty()
  @IsString()
  title: string;

  @IsNotEmpty()
  @IsEnum(MovieGenre)
  genre: MovieGenre;
}
```

```typescript
// 컨트롤러에서 사용
@Post()
postMovies(@Body() body: CreateMovieDto) {
  return this.movieService.createMovie(body);
}

@Patch(':id')
updateMovie(
  @Param('id', ParseIntPipe) id: number,
  @Body() body: UpdateMovieDto,
) {
  return this.movieService.updateMovie(id, body);
}
```

---

---

# Mapped Types — UpdateDto 간단하게 ⭐️

```typescript
import { PartialType, OmitType, PickType, IntersectionType } from '@nestjs/mapped-types';

// PartialType — 모든 필드를 optional 로
export class UpdateMovieDto extends PartialType(CreateMovieDto) {}
// → title?: string / genre?: MovieGenre 자동 생성, @IsOptional() 도 자동 적용

// OmitType — 특정 필드 제외
export class UpdateMovieDto extends OmitType(CreateMovieDto, ['genre'] as const) {}

// PickType — 특정 필드만 선택
export class LoginDto extends PickType(CreateMovieDto, ['title'] as const) {}

// IntersectionType — 두 DTO 합치기
export class FullMovieDto extends IntersectionType(CreateMovieDto, AdditionalDto) {}
```

```txt
PartialType 장점: CreateMovieDto 필드 추가 → UpdateMovieDto 자동 반영, @IsOptional() 직접 안 써도 됨
❌ 직접 쓰면: 모든 필드에 @IsOptional() 다시 선언해야 하고, CreateMovieDto 변경 시 수동 수정 필요
```

## ⚠️ PATCH 에서 @IsOptional() 없으면 에러 ⭐️

```txt
문제: title?: string 만 있고 @IsOptional() 없을 때
  PATCH { "genre": "코미디" } ← title 없이 요청
  title 이 undefined 인데 @IsNotEmpty() 검사가 실행됨 → "title should not be empty" 400 에러

원인: title?: string 은 TS 에서만 선택적 — class-validator 는 undefined 필드도 검사함
해결: @IsOptional() 추가 → 값 없으면 검사 자체를 건너뜀, 값 있을 때만 나머지 검사 실행
```

```typescript
// ✅ @IsOptional() 세트로
export class UpdateMovieDto {
  @IsOptional()   // 없으면 아래 검사 건너뜀
  @IsNotEmpty()   // 있으면 빈 값 체크 ('' 방지)
  @IsString()
  title?: string;
}
```

---

---

# class-validator 데코레이터 상세 ⭐️

## 필수 / 선택

```typescript
@IsNotEmpty()   // 빈 값 불가 (null / undefined / '' 불가)
@IsOptional()   // 없어도 되지만 있으면 검사 적용
@IsDefined()    // undefined 불가 (null 은 허용)
```

## 타입

```typescript
@IsString()
@IsNumber()
@IsInt()        // 정수만 (소수 불가)
@IsBoolean()
@IsArray()
@IsDate()
@IsEnum(MovieGenre)
```

## 숫자 범위

```typescript
@Min(1)         // 1 이상
@Max(100)       // 100 이하
@IsPositive()   // 양수 (0.1 이상)
@IsNegative()   // 음수 (-0.1 이하)
@IsDivisibleBy(5)   // 5의 배수 (5, 10, 15...)
```

## 문자열

```typescript
@MinLength(4)      // 최소 4자
@MaxLength(16)     // 최대 16자
@IsEmail()
@IsUrl()           // 옵션 상세는 바로 아래 참고
@IsUUID()
@Contains('test')  // 'test' 포함
@IsAlphanumeric()  // 영문 + 숫자만
@IsHexColor()      // #fff, #ffffff

// 위치 관련
@IsLatLong()       // "37.5665,126.9780" — 위도,경도 합친 문자열
@IsLatitude()      // 위도만 (-90 ~ 90)
@IsLongitude()     // 경도만 (-180 ~ 180)
```

## @IsUrl — 옵션으로 더 엄격하게 ⭐️⭐️

```typescript
@IsString()
@IsUrl({ require_protocol: true, protocols: ['http', 'https'] })
@IsNotEmpty()
embedUrl: string;
```

| 옵션                 | 기본값                      | 역할                                                                        |
| ------------------ | ------------------------ | ------------------------------------------------------------------------- |
| `protocols`        | `['http','https','ftp']` | 허용할 프로토콜 목록 — `ftp` 등 원치 않는 값 빼고 좁히고 싶을 때 지정                              |
| `require_protocol` | `false`                  | `true` 면 `https://` 없이 `example.com` 만 오면 거부 — 임베드 URL 등 "완전한 주소" 가 필요할 때 |
| `require_tld`      | `true`                   | 최상위 도메인(`.com` 등) 필수 — `http://localhost` 같은 내부 주소를 막는 원인이 되기도 함          |
| `require_host`     | `true`                   | 호스트 부분 필수                                                                 |

```txt
옵션 없이 @IsUrl() 만 쓰면: 기본값(require_tld:true 등)이 그대로 적용됨
  → 로컬 개발 중 http://localhost:3000 같은 값을 검증해야 한다면 require_tld: false 도 같이 고려

⚠️ require_protocol: true 인데 위반하면, 에러 메시지가 "어떤 옵션을 어겼는지" 까지는 안 알려주고
   그냥 "올바른 URL 형식이 아닙니다" 식으로만 나옴 — 커스텀 메시지를 직접 넣어두는 걸 권장:
   @IsUrl({ require_protocol: true }, { message: 'http(s):// 를 포함한 전체 주소를 입력해주세요.' })
   @IsUrl(
    { require_protocol: true, require_tld: false, protocols: ['http', 'https'] },
    { message: 'http(s):// 를 포함한 전체 주소를 입력해주세요.' }
  )
```

## @Matches — 정규식 검증 ⭐️

```typescript
import { Matches } from 'class-validator';

// 닉네임: 한글·영문·숫자·밑줄만 허용
@Matches(/^[가-힣a-zA-Z0-9_]+$/, {
  message: '닉네임은 한글·영문·숫자·밑줄만 사용할 수 있습니다.',
})
nickname: string;

// 비밀번호: 영문+숫자+특수문자 포함
@Matches(/^(?=.*[A-Za-z])(?=.*\d)(?=.*[@$!%*#?&])/, {
  message: '비밀번호는 영문, 숫자, 특수문자를 포함해야 합니다.',
})
password: string;

// 전화번호: 000-0000-0000 형식
@Matches(/^\d{3}-\d{3,4}-\d{4}$/, {
  message: '전화번호 형식이 올바르지 않습니다. (예: 010-1234-5678)',
})
phone: string;

// URL 슬러그: 소문자·숫자·하이픈만
@Matches(/^[a-z0-9-]+$/, {
  message: '슬러그는 소문자, 숫자, 하이픈만 사용할 수 있습니다.',
})
slug: string;
```

```txt
@Matches(regex, options): 첫 번째 인자 정규식 / 두 번째 인자 { message: '에러 메시지' }
@IsAlphanumeric() 으로 안 되는 경우(한글 포함, 특정 문자만 허용, 복잡한 패턴) → @Matches 로 직접 작성
@IsEmail() / @IsUrl() 내부도 정규식 기반 — 세밀한 제어가 필요할 때 @Matches 사용
```

## 허용 값을 별도 상수 파일로 분리하기 — 단일 소스 패턴 ⭐️⭐️⭐️

```txt
@IsIn 에 넘기는 배열을 DTO 파일 안에 직접 적으면, 같은 값 목록을 다른 곳(다른 DTO, 서비스 로직 등)에서
또 써야 할 때 매번 다시 타이핑하게 됨 — 허용값 자체를 별도 상수 파일로 분리해서 "한 곳"에서만 관리
```

```typescript
// src/posts/constants/tags.ts
/** 허용 태그 목록 — DTO @IsIn · DB tags[] 와 동일 문자열로 유지(단일 소스) */
export const TAGS = [
  'A', 'B', 'C', 'D', 'E',
] as const;

export type Tag = (typeof TAGS)[number];

export const MIN_TAGS = 1;
export const MAX_TAGS = 3;

/** class-validator @IsIn 용 — readonly string[] 로 명시해서 "검증용 export" 라는 의도를 드러냄 */
export const TAG_VALUES: readonly string[] = TAGS;
```

```typescript
// src/posts/dto/create-post.dto.ts
import { ArrayMaxSize, ArrayMinSize, IsArray, IsIn } from 'class-validator';
import { MAX_TAGS, MIN_TAGS, TAG_VALUES } from '../constants/tags';

export class CreatePostDto {
  /** 태그 1~3개 — TAGS 단일 소스 */
  @IsArray()
  @ArrayMinSize(MIN_TAGS)
  @ArrayMaxSize(MAX_TAGS)
  @IsIn(TAG_VALUES, { each: true })
  tags: string[];
}
```

| 조각                                  | 역할                                                                         |
| ----------------------------------- | -------------------------------------------------------------------------- |
| `as const` 배열                       | 허용값을 한 곳에 모음 — 리터럴 유니온 타입도 같이 추출 가능                                        |
| `(typeof TAGS)[number]`             | 배열 값으로부터 유니온 타입 자동 생성 (<code>'A'\|'B'\|...</code>) — [[TS_Type_Guards]] 참고 |
| `MIN_/MAX_` 상수                      | 개수 제한도 같은 파일에 — `@ArrayMinSize`/`@ArrayMaxSize` 에서 재사용                     |
| `TAG_VALUES: readonly string[]`     | `@IsIn` 전달용으로 명시적으로 분리 — "검증에 쓰는 값" 이라는 의도를 이름으로 드러냄                       |
| `@IsIn(TAG_VALUES, { each: true })` | 배열의 각 요소가 허용 목록 안에 있는지 검사 (`each: true` 없으면 배열 전체를 통째로 검사해서 항상 실패)         |

```txt
Prisma 쪽은 그대로 String[] 로 둔다(DB enum 안 만듦):
  model Post { tags String[] }
  → 허용값 제약은 DB 가 아니라 애플리케이션(DTO) 레벨에서만 검증
  → 값 목록이 자주 바뀔 수 있는 경우, DB enum 마이그레이션 없이 상수 파일만 고치면 되는 게 장점
  → 대신 "DB 가 직접 보장하는 제약" 은 아니라서, 이 상수 파일을 거치지 않는 경로(직접 SQL 등)는 못 막음

이 단일 소스 파일이 가리키는 곳이 한 군데 더 있다는 것도 주석으로 남겨두면 좋음:
  /** ... DB tags[] 와 동일 문자열(단일 소스) */ 처럼, "이 목록이 스키마와 맞아야 한다" 는 걸 명시
```

```txt
@IsEnum vs @IsIn:
  @IsEnum(Enum)  TypeScript enum 검증
  @IsIn(arr)     const 배열 검증 → 더 가볍고, 위처럼 별도 상수 파일로 분리하기도 쉬움
```

## 배열

```typescript
@IsArray()
@ArrayNotEmpty()      // 빈 배열 불가
@ArrayMinSize(1)      // 최소 N개
@ArrayMaxSize(10)     // 최대 N개
@ArrayUnique()        // 중복 불가
```

## 배열 요소 각각 검사 — each: true ⭐️

```typescript
@IsArray()
@ArrayNotEmpty()
@IsNumber({}, { each: true })
//             ↑ 배열 요소 각각에 @IsNumber 검사
genreIds: number[];

@IsArray()
@IsString({ each: true })
tags: string[];

@IsArray()
@IsEnum(MovieGenre, { each: true })
genres: MovieGenre[];
```

```txt
each: true 없으면: 배열 자체를 숫자로 검사 → 항상 실패
each: true 있으면: [1, 2, 3] 각 요소가 숫자인지 검사, [1, '2', 3] → '2' 숫자 아님 → 실패
```

## @IsDate vs @IsDateString ⭐️

```typescript
// @IsDateString — ISO 8601 문자열 형식
@IsDateString()
dob: string;
// "2000-01-01" → ✅ / "2000-01-01T00:00:00.000Z" → ❌

// @IsDate + @Type — Date 객체로 변환 후 검사 (권장)
@Type(() => Date)   // 문자열 → Date 객체 자동 변환
@IsDate()
dob: Date;
// "2000-01-01T00:00:00.000Z" → Date 변환 → ✅
```

```txt
HTTP 요청은 항상 문자열로 들어옴, @IsDate() 는 Date 객체를 기대함 → @Type(() => Date) 로 먼저 변환 필요
```

## Query Parameter 배열 파싱 문제 ⭐️

```txt
?order=id_ASC          → string (1개)
?order=id_ASC&order=id_DESC → string[] (여러 개)
DTO 에 string[] 로 선언해도 값 1개면 string 으로 옴 → @IsArray() 검사 실패
```

```typescript
// ✅ @Transform 으로 단일 문자열을 배열로 변환
@IsOptional()
@IsArray()
@IsString({ each: true })
@Transform(({ value }) =>
  value === undefined ? ['id_DESC'] : Array.isArray(value) ? value : [value],
)
order: string[] = ['id_DESC'];
```

---

---

# class-transformer — 응답 직렬화 ⭐️

```bash
pnpm add class-transformer --filter api
```

```typescript
// main.ts — ClassSerializerInterceptor 전역 등록
import { ClassSerializerInterceptor } from '@nestjs/common';
import { Reflector } from '@nestjs/core';

app.useGlobalInterceptors(new ClassSerializerInterceptor(app.get(Reflector)));
```

## @Exclude — 응답에서 필드 제외 ⭐️

```typescript
import { Exclude } from 'class-transformer';

export class UserEntity {
  id:    number;
  email: string;

  @Exclude()
  password: string;   // 응답에서 자동 제거
}
```

## @Expose — 특정 필드만 노출

```typescript
import { Expose } from 'class-transformer';

export class UserEntity {
  @Expose()
  id: number;

  @Expose()
  email: string;

  password: string;   // @Expose 없으면 제거
}
```

## @Transform — 값 변환 ⭐️

```typescript
import { Transform } from 'class-transformer';

export class MovieEntity {
  @Transform(({ value }) => value.toUpperCase())
  title: string;

  @Transform(({ value }) => (value ? '있음' : '없음'))
  hasDetail: boolean;
}
```

### @Transform 상세 — 요청 데이터 가공 ⭐️

```typescript
// 문자열 → 숫자 변환
@Transform(({ value }) => Number(value))
@IsNumber()
price: number;

// 문자열 → 불리언 변환
@Transform(({ value }) => value === 'true' || value === true)
@IsBoolean()
isPublished: boolean;

// 문자열 트림
@Transform(({ value }) => value?.trim())
@IsString()
title: string;

// 기본값 처리
@Transform(({ value }) => value ?? 'unknown')
@IsString()
category: string;
```

```txt
({ value }) 구조: value(현재 필드 값) / key(필드 이름) / obj(DTO 인스턴스 전체) / type(변환 타입)

obj 로 다른 필드 참조:
  @Transform(({ value, obj }) => obj.isAdmin ? value.toUpperCase() : value)
  title: string;

실행 순서: @Transform 이 먼저 실행된 후 @IsString() 등 검증 실행 — 변환 후 검증하는 구조
```

```txt
@Exclude   응답에서 그 필드를 아예 제거 (password 등 민감 정보)
@Expose    excludeExtraneousValues 옵션 시 이 필드만 노출
@Transform 값을 원하는 형태로 가공해서 반환
ClassSerializerInterceptor 가 등록되어 있어야 동작
```

---

---

# @ValidatorConstraint — 커스텀 Validator ⭐️

```txt
class-validator 기본 데코레이터로 표현 못 하는 복잡한 검증 규칙을 직접 데코레이터로 만들어서 사용
```

## 동기 — 기본 값 검증

```typescript
import {
  ValidatorConstraint,
  ValidatorConstraintInterface,
  registerDecorator,
} from 'class-validator';

@ValidatorConstraint()
class PasswordValidator implements ValidatorConstraintInterface {
  validate(value: any): boolean {
    return value.length > 4 && value.length < 8;
  }
  defaultMessage(): string {
    return '비밀번호는 4자 이상 8자 이하로 입력해주세요.';
  }
}

function IsPasswordValid(validationOptions?: any) {
  return function (object: Object, propertyName: string) {
    registerDecorator({
      target: object.constructor,
      propertyName,
      options: validationOptions,
      validator: PasswordValidator,
    });
  };
}

export class CreateUserDto {
  @IsPasswordValid()
  password: string;
}
```

## 비동기 — DB 조회 등 ⭐️

```typescript
// { async: true } → validate() 에서 Promise 반환 가능
@ValidatorConstraint({ async: true })
class IsEmailUniqueConstraint implements ValidatorConstraintInterface {
  constructor(private readonly userRepository: UserRepository) {}

  async validate(email: string): Promise<boolean> {
    const user = await this.userRepository.findOne({ where: { email } });
    return !user;
  }
  defaultMessage(): string {
    return '이미 사용 중인 이메일입니다.';
  }
}

function IsEmailUnique(validationOptions?: any) {
  return function (object: Object, propertyName: string) {
    registerDecorator({
      target: object.constructor,
      propertyName,
      options: validationOptions,
      validator: IsEmailUniqueConstraint,
    });
  };
}

export class CreateUserDto {
  @IsEmailUnique()
  @IsEmail()
  email: string;
}
```

```txt
@ValidatorConstraint({ async: true }): validate() 에서 async/Promise<boolean> 반환 가능 (DB 조회 등)
  비동기 Validator 를 NestJS DI 와 연결하려면
  main.ts 에 useContainer(app.select(AppModule), { fallbackOnErrors: true }) 추가 필요

name 옵션: @ValidatorConstraint({ name: '...', async: false }) — 에러/디버깅 시 식별용, 생략하면 클래스명 사용

동기 vs 비동기: 동기는 값 자체 검증(길이/형식/범위), 비동기는 외부 데이터 참조 필요(DB 중복 확인 등)
```

### ValidationArguments — 다른 필드 참조 ⭐️

```typescript
@ValidatorConstraint()
class IsPasswordMatchConstraint implements ValidatorConstraintInterface {
  validate(confirmPassword: string, args: ValidationArguments): boolean {
    const [relatedPropertyName] = args.constraints;
    const relatedValue = (args.object as any)[relatedPropertyName];
    return confirmPassword === relatedValue;
  }
  defaultMessage(): string {
    return '비밀번호가 일치하지 않습니다.';
  }
}

function IsPasswordMatch(property: string, validationOptions?: any) {
  return function (object: Object, propertyName: string) {
    registerDecorator({
      target: object.constructor,
      propertyName,
      options: validationOptions,
      constraints: [property],
      validator: IsPasswordMatchConstraint,
    });
  };
}

export class CreateUserDto {
  @IsString()
  password: string;

  @IsPasswordMatch('password')
  confirmPassword: string;
}
```

```txt
ValidationArguments: value(현재 필드 값) / object(DTO 인스턴스 전체) / constraints(registerDecorator 에서 전달한 값) / property(현재 필드명)
언제 쓰나: 두 필드 비교(비밀번호 확인), 다른 필드 값에 따라 검증 규칙이 달라질 때
```

---

---

# Swagger — DTO 기반 API 문서 자동화 ⭐️

```txt
@nestjs/swagger 는 작성한 DTO 클래스를 그대로 활용해서 API 문서를 자동으로 만들어줌
→ class-validator 데코레이터를 단 같은 DTO 에, Swagger 용 데코레이터를 추가로 얹는 구조
(전체 설정/옵션은 [[NestJS_Swagger]] 참고 — 여기서는 DTO 와 맞물리는 부분만)
```

```txt
⚠️ 새 DTO(또는 새 Controller) 파일을 만든 직후엔 Swagger 문서에 안 보일 수 있음
   SwaggerModule.createDocument(app, config) 는 main.ts 부팅 시점에 "한 번" 스캔해서 문서를 만듦
   → --watch 가 기존 파일의 "수정"은 바로 반영해도, 새로 추가된 파일은 못 잡는 경우가 있음
   → start:dev 를 완전히 재시작하면 부팅이 다시 일어나면서 새 DTO/Controller 까지 다시 스캔됨
```

```bash
# 모노레포일때 다른 설치 방법은 [[NestJS_Swagger]]
pnpm add @nestjs/swagger --filter api 
```

```typescript
// swagger.ts
import { INestApplication } from '@nestjs/common';
import { DocumentBuilder, SwaggerModule } from '@nestjs/swagger';

export function setupSwagger(app: INestApplication) {
  const config = new DocumentBuilder()
    .setTitle('My API')
    .setDescription('My API description')
    .setVersion('0.0.1')
    .addBearerAuth()
    .build();

  const document = SwaggerModule.createDocument(app, config);
  SwaggerModule.setup('api', app, document);
}
```

```typescript
// main.ts
import { setupSwagger } from './swagger';

async function bootstrap() {
  const app = await NestFactory.create(AppModule);
  app.useGlobalPipes(new ValidationPipe({ transform: true, whitelist: true, forbidNonWhitelisted: true }));
  setupSwagger(app);
  await app.listen(process.env.PORT ?? 3000);
}
bootstrap();
```

## @ApiProperty — DTO 필드 문서화

```typescript
import { ApiProperty } from '@nestjs/swagger';

export class CreateMovieDto {
  @ApiProperty({ description: '영화 제목', example: '인터스텔라' })
  @IsNotEmpty()
  @IsString()
  title: string;
}
```

```txt
같은 필드 위에 class-validator(@IsString 등)와 Swagger(@ApiProperty)가 같이 붙는 구조
  class-validator → 실제 검증(런타임에 막아주는 역할) / @ApiProperty → 문서화 — 둘은 역할이 다름
```

## CLI 플러그인으로 자동화

```json
// nest-cli.json
"plugins": [
  {
    "name": "@nestjs/swagger",
    "options": {
      "classValidatorShim": true,
      "introspectComments": true
    }
  }
]
```

```txt
introspectComments: true 면 필드 위 JSDoc 주석(/** ... */, @example)이 description/example 으로 자동 반영
→ 옵션별 전체 설명과 dtoFileNameSuffix 등 나머지 옵션은 [[NestJS_Swagger]] 참고
```

## 프론트엔드 타입 자동 생성과 연결 ⭐️

```txt
Swagger 를 켜두면 OpenAPI 스펙(JSON)을 그대로 받을 수 있음 (예: http://localhost:3000/api-json)
이 JSON 으로 프론트엔드 타입을 자동 생성할 수 있음:
  npx openapi-typescript http://localhost:3000/api-json -o web/lib/types/api.d.ts

→ DTO 가 바뀌면 이 명령을 다시 돌리는 것만으로 프론트 타입도 같이 갱신됨
→ 관련 내용은 [[NextJS_API_Integration]] 의 "API 타입 정의" 참고
  (소규모에서는 직접 타입을 작성해도 충분 — DTO 가 많아지고 자주 바뀌는 시점부터 자동화가 유리)
```

---

---

# 커스텀 에러 메시지

```typescript
@IsNotEmpty({ message: '이름을 입력해주세요!' })
name: string;

@IsEmail({}, { message: '정확한 이메일 주소를 입력해주세요!' })
email: string;

@Min(1, { message: '1 이상이어야 합니다' })
age: number;
```

---

---

# 에러 응답 구조

```json
{
  "statusCode": 400,
  "message": ["title should not be empty", "genre must be a valid enum value"],
  "error": "Bad Request"
}
```

---

---

# 전체 흐름

```txt
pnpm add class-validator class-transformer --filter api
  ↓
main.ts — ValidationPipe(whitelist 포함) + ClassSerializerInterceptor 전역 등록
  ↓
DTO 폴더 구조 결정 (모듈별 dto/ 또는 common/dto/)
  ↓
DTO 작성 — Prisma 모델 전체가 아니라 "클라이언트가 실제로 보낼 값" 만
  ↓
UpdateDto — PartialType(CreateMovieDto)
  ↓
Controller — @Body() body: CreateMovieDto
  ↓
유효성 검사 실패 → 400 에러 자동 반환 / 통과 → Service 로 전달
```

---

---

# 한눈에

|목적|도구|
|---|---|
|유효성 검사|`class-validator` 데코레이터|
|DTO 에 안 넣은 필드 실제로 차단|`whitelist: true`(`ValidationPipe`) ⭐️|
|PATCH DTO|`PartialType(CreateDto)`|
|응답에서 password 제거|`@Exclude()`|
|값 변환|`@Transform()`|
|string → Date|`@Type(() => Date)`|
|정규식 패턴 검증|`@Matches(/regex/, { message })` ⭐️|
|URL 형식을 더 엄격하게|`@IsUrl({ require_protocol, protocols })` ⭐️|
|허용 값 목록 검증 (+ 단일 소스 파일)|`@IsIn(VALUES, { each: true })` + `as const` ⭐️|
|배열 요소 각각 검사|`{ each: true }`|
|query 배열 파싱|`@Transform` 단일값 → 배열|
|DB 중복 확인|`@ValidatorConstraint({ async: true })`|

```txt
DTO 설계 시 가장 먼저 떠올릴 질문 — "이 필드, 누구의 책임인가?"
  사용자 입력 → DTO 에 포함
  서버/DB 가 결정(id, 생성·수정 시각, 기본 상태값) → DTO 에서 제외 + whitelist:true 로 차단 + 주석으로 이유 남기기

허용값이 여러 곳(DTO, 다른 DTO, 서비스 로직)에서 반복되면 → 상수 파일로 분리해서 단일 소스로 관리
```