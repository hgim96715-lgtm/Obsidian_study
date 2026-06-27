---
aliases:
  - QueryBuilder
  - createQueryBuilder
  - 복잡한 쿼리
  - identifiers
  - SelectQueryBuilder
tags:
  - NestJS
related:
  - "[[00_NestJS_Ecosystem_HomePage]]"
---
# NestJS_TypeORM_QueryBuilder — QueryBuilder

## 한 줄 요약

```txt
Repository  = 단순 CRUD / 기본 조회
QueryBuilder = 복잡한 쿼리 / 동적 쿼리 / 조건 분기
```

---

---

# Repository vs QueryBuilder ⭐️

```txt
Repository 쓰는 경우:
  find / findOne / save / delete
  단순한 조건 / 관계 조회 (relations)

QueryBuilder 써야 하는 경우:
  복잡한 WHERE 조건 (동적으로 조건이 바뀔 때)
  JOIN 을 직접 제어해야 할 때
  GROUP BY / HAVING
  서브쿼리
  특정 컬럼만 SELECT
  성능 최적화가 필요할 때
```

---

---

#  QueryBuilder 시작하기

```typescript
// 방법 1: Repository 에서 생성
const qb = this.movieRepository.createQueryBuilder('movie');
//                                                   ↑ alias (별칭)

// 방법 2: DataSource 에서 생성
const qb = this.dataSource
  .createQueryBuilder()
  .select('movie')
  .from(Movie, 'movie');
```

```txt
alias (별칭):
  쿼리 안에서 테이블을 부르는 이름
  SQL 의 AS 와 동일
  SELECT movie.* FROM movie AS movie ...
```

---

---

#  :파라미터명 — 바인딩 문법 ⭐️

```txt
TypeORM QueryBuilder 에서 값을 전달하는 방법
:이름  →  { 이름: 값 } 객체로 전달
SQL 인젝션 방지 + 가독성 향상
```

```typescript
// :title 이 파라미터 자리표시자
.where('movie.title LIKE :title', { title: `%마이클%` })
//                    ↑            ↑ 실제 값 전달
//              :title 자리에      title 키의 값이 들어감

// 실행되는 SQL:
// WHERE movie.title LIKE '%마이클%'
```

## 일반 문자열 비교와 차이

```typescript
// ❌ 문자열 직접 삽입 (SQL 인젝션 위험)
.where(`movie.title = '${title}'`)
// 만약 title = "'; DROP TABLE movie; --" 이면 DB 삭제됨!

// ✅ 파라미터 바인딩 (안전)
.where('movie.title = :title', { title })
// TypeORM 이 내부적으로 값을 이스케이프 처리
```

## 다양한 예시

```typescript
// 단일 파라미터
.where('movie.id = :id', { id: 1 })

// 여러 파라미터
.where('movie.genre = :genre AND movie.rating > :min',
  { genre: 'drama', min: 7.0 })

// LIKE (부분 검색)
.where('movie.title LIKE :title', { title: `%${searchText}%` })
//                                          ↑ % 는 자바스크립트에서 붙임

// IN
.where('movie.id IN (:...ids)', { ids: [1, 2, 3] })
//                  ↑↑↑ ...ids = 배열 전달 시

// BETWEEN
.where('movie.rating BETWEEN :min AND :max', { min: 7, max: 9 })
```

```txt
:이름     단일 값
:...이름  배열 (IN 절에서 사용)

% 위치:
  LIKE '%마이클%'  = 앞뒤 어디든
  LIKE '마이클%'   = 마이클로 시작
  LIKE '%마이클'   = 마이클로 끝
  → % 는 JS 코드에서 붙여서 전달
    { title: `%${searchText}%` }
```

---

---

#  실행 타입 5가지

## 1. SELECT ⭐️

```typescript
// 기본 SELECT
const movies = await this.movieRepository
  .createQueryBuilder('movie')
  .select(['movie.id', 'movie.title'])    // 특정 컬럼만
  .where('movie.genre = :genre', { genre: 'drama' })
  .orderBy('movie.createdAt', 'DESC')
  .take(10)                               // LIMIT
  .skip(0)                                // OFFSET
  .getMany();                             // 여러 행 반환

// 단건
const movie = await this.movieRepository
  .createQueryBuilder('movie')
  .where('movie.id = :id', { id: 1 })
  .getOne();                              // 단건 반환 (없으면 null)
```

```txt
getMany()   → Movie[]
getOne()    → Movie | null
getRawMany() → 원시 데이터 (JOIN 결과 등)
getCount()  → 개수만
```

## 2. INSERT ⭐️

```typescript
await this.dataSource
  .createQueryBuilder()   // QueryBuilder 인스턴스 생성
  .insert()               // INSERT 모드로 설정
  .into(Movie)            // INSERT INTO movie 테이블 지정
  .values([               // 넣을 데이터
    { title: '마이클', genre: 'drama' },
    { title: '기생충', genre: 'drama' },
  ])
  .execute();             // SQL 실행 → InsertResult 반환
```

## 각 체이닝 메서드 설명 ⭐️

```txt
createQueryBuilder():
  QueryBuilder 인스턴스 생성
  두 가지 방법:
    this.dataSource.createQueryBuilder()   → 어떤 Entity 든 지정 가능
    this.movieRepository.createQueryBuilder('movie')  → Movie Entity 기준

.insert():
  이 QueryBuilder 를 INSERT 모드로 설정
  이후 into() / values() 를 붙여서 INSERT 쿼리 완성

.into(Entity):
  INSERT INTO 어느 테이블에 넣을지 지정
  .into(Movie)      → movie 테이블에 INSERT
  .into(MovieDetail) → movie_detail 테이블에 INSERT

.values(data):
  INSERT 할 실제 데이터
  단건:  .values({ title: '아바타' })
  여러건: .values([{ title: '아바타' }, { title: '기생충' }])

.execute():
  SQL 실행 → InsertResult 반환
  결과에서 생성된 id 꺼내기:
    const result = await qb.execute();
    result.identifiers     → [{ id: 1 }]
    result.generatedMaps   → [{ id: 1, ... }]
    result.raw             → 원시 DB 응답
```

## qr.manager.createQueryBuilder() — 트랜잭션 내에서 ⭐️

```typescript
// 트랜잭션 안에서 INSERT (QueryRunner 의 manager 사용)
const movieDetail = await qr.manager
  .createQueryBuilder()
  .insert()
  .into(MovieDetail)
  .values({ detail: createMovieDto.detail })
  .execute();

// 왜 qr.manager 를 쓰나:
//   qr = QueryRunner (트랜잭션 관리자)
//   qr.manager = 이 트랜잭션에 묶인 EntityManager
//   → 이 manager 로 실행한 쿼리는 트랜잭션 안에 포함됨
//   → this.dataSource.createQueryBuilder() 로 하면
//      트랜잭션 밖에서 실행될 수 있음 ⚠️

// execute() 결과에서 id 꺼내기
const detailId = movieDetail.identifiers[0].id;
```

```txt
qr.manager vs this.dataSource:
  this.dataSource.createQueryBuilder()  → 트랜잭션 외부
  qr.manager.createQueryBuilder()       → 트랜잭션 내부 (롤백 가능)

  트랜잭션 안에서는 반드시 qr.manager 사용
```

>→ [[NestJS_Transaction]] 참조 

## 3. UPDATE

```typescript
await this.dataSource
  .createQueryBuilder()
  .update(Movie)
  .set({ title: '수정된 제목' })
  .where('id = :id', { id: 1 })
  .execute();
```

## 4. DELETE

```typescript
await this.dataSource
  .createQueryBuilder()
  .delete()
  .from(Movie)
  .where('id = :id', { id: 1 })
  .execute();
```

## 5. RELATIONS (JOIN)

```typescript
// LEFT JOIN 으로 관계 데이터 함께 조회
const movies = await this.movieRepository
  .createQueryBuilder('movie')
  .leftJoinAndSelect('movie.detail', 'detail')
  //                  ↑ 관계 프로퍼티  ↑ alias
  .leftJoinAndSelect('movie.director', 'director')
  .where('movie.id = :id', { id: 1 })
  .getOne();

// movie.detail / movie.director 자동 포함됨
```

---

---

# 동적 조건 — 핵심 활용 ⭐️

```typescript
// 동적으로 조건이 추가되는 쿼리
async getMovies(title?: string, genre?: string) {
  const qb = this.movieRepository
    .createQueryBuilder('movie')
    .leftJoinAndSelect('movie.detail', 'detail');

  // 조건이 있을 때만 WHERE 추가
  if (title) {
    qb.andWhere('movie.title LIKE :title', { title: `%${title}%` });
  }

  if (genre) {
    qb.andWhere('movie.genre = :genre', { genre });
  }

  return qb.orderBy('movie.createdAt', 'DESC').getMany();
}
```

```txt
Repository 의 find 에서는:
  where: { title: Like('%마이클%'), genre: 'drama' }
  조건이 없으면 undefined 처리 복잡

QueryBuilder 에서는:
  if (title) qb.andWhere(...)
  if (genre) qb.andWhere(...)
  → 조건 없으면 자동으로 빠짐 → 동적 쿼리에 유리
```

---
---
# SelectQueryBuilder 란 ️

```typescript
import { SelectQueryBuilder } from 'typeorm';

// qb 의 타입이 SelectQueryBuilder<T>
const qb: SelectQueryBuilder<Movie> = this.movieRepository
  .createQueryBuilder('movie');
```

```txt
SelectQueryBuilder:
  TypeORM 에서 제공하는 타입
  createQueryBuilder() 의 반환 타입
  SELECT 쿼리를 만들 수 있는 메서드들을 가진 클래스

  타입 파라미터 <T>:
    어떤 Entity 의 QueryBuilder 인지
    SelectQueryBuilder<Movie> → Movie Entity 용
    SelectQueryBuilder<T extends ObjectLiteral> → 제네릭 (어떤 Entity 든)

왜 타입을 명시하나:
  CommonService 처럼 여러 Entity 에서 쓰는 공통 함수 만들 때
  타입 안정성 확보
```

---

---

# qb.alias — 별칭 동적으로 읽기 ️

```typescript
// qb.alias = createQueryBuilder 에 준 별칭
const qb = this.movieRepository.createQueryBuilder('movie');
console.log(qb.alias);   // 'movie'

const qb = this.genreRepository.createQueryBuilder('genre');
console.log(qb.alias);   // 'genre'
```


```typescript
// CommonService 에서 qb.alias 활용
applyCursorPaginationParamsToQb<T extends ObjectLiteral>(
  qb: SelectQueryBuilder<T>,
  dto: CursorPaginationDto,
) {
  const { id, order } = dto;

  if (id) {
    const direction = order === 'ASC' ? '>' : '<';
    qb.where(`${qb.alias}.id ${direction}= :id`, { id });
    //        ↑ 동적으로 테이블 별칭 참조
    // movie  → 'movie.id >= :id'
    // genre  → 'genre.id >= :id'
  }

  qb.orderBy(`${qb.alias}.id`, order);
  qb.take(dto.take);
}
```

```txt
qb.alias 쓰는 이유:
  CommonService 는 Movie / Genre / Director 등 여러 Entity 에서 사용
  테이블 별칭을 하드코딩하면 각 Entity 마다 함수 따로 필요
  qb.alias = createQueryBuilder('movie') 에서 준 'movie' 를 자동으로 읽음
  → 어떤 Entity 든 같은 함수 재사용 가능
```

---

---

# qb 주요 프로퍼티 & 메서드 ⭐️

## 프로퍼티 (값을 읽는 것)

```typescript
qb.alias          // createQueryBuilder('movie') 에서 준 별칭 → 'movie'
qb.connection     // 현재 DB 연결 객체
qb.expressionMap  // 현재 쿼리 빌더 내부 상태 (디버깅용)
```

## 조건 메서드

```typescript
qb.where('movie.id = :id', { id })         // 첫 번째 WHERE (덮어씀)
qb.andWhere('movie.genre = :genre', { genre }) // AND 조건 추가
qb.orWhere('movie.title LIKE :t', { t })   // OR 조건 추가
```

## 정렬 / 페이징 메서드 ⭐️

```typescript
qb.orderBy('movie.id', 'DESC')    // 정렬 (기존 정렬 덮어씀)
qb.addOrderBy('movie.title', 'ASC') // 정렬 추가 (기존 유지)

qb.take(10)   // LIMIT 10 (가져올 개수)
qb.skip(20)   // OFFSET 20 (건너뛸 개수)
```

## SELECT / JOIN 메서드

```typescript
qb.select(['movie.id', 'movie.title'])     // 특정 컬럼만
qb.addSelect('detail.description')         // 컬럼 추가
qb.leftJoinAndSelect('movie.detail', 'detail')  // LEFT JOIN
qb.innerJoinAndSelect('movie.genres', 'genre')  // INNER JOIN
```

## 결과 반환 메서드

```typescript
qb.getMany()         // Entity[] 반환
qb.getOne()          // Entity | null 반환
qb.getCount()        // 개수만 반환
qb.getManyAndCount() // [Entity[], number] → 페이징에 유용
qb.getRawMany()      // 원시 데이터 (집계 등)
qb.execute()         // INSERT / UPDATE / DELETE 실행 결과
```

## getManyAndCount() ⭐️

```typescript
// [데이터 배열, 전체 개수] 튜플 반환
const [data, count] = await qb.getManyAndCount();
//      ↑       ↑
//   Movie[]  number (take/skip 무시한 전체 개수)
```

```txt
getManyAndCount 가 반환하는 것:
  [0] data   → take(5) 로 가져온 실제 데이터 (Movie[])
  [1] count  → take/skip 무시한 전체 일치 개수 (number)

왜 유용한가:
  페이지네이션에서 "전체 몇 개인지" 도 필요할 때

  getMany() 만 쓰면:
    데이터는 오지만 전체 개수 모름 → totalPages 계산 불가

  getCount() 따로 쓰면:
    쿼리 두 번 날려야 함 (비효율)

  getManyAndCount() 하나로:
    데이터 + 전체 개수 한 번에 → 쿼리 1번
```

```typescript
// 실전 — 페이지네이션 응답
const [data, count] = await qb.getManyAndCount();

return {
  data,
  total: count,
  page,
  take,
  totalPages: Math.ceil(count / take),
};
```

## 실전 — Cursor Pagination 패턴

```typescript
// CommonService 에서 qb 메서드 조합
applyCursorPaginationParamsToQb<T extends ObjectLiteral>(
  qb: SelectQueryBuilder<T>,
  dto: CursorPaginationDto,
) {
  const { id, order } = dto;

  if (id) {
    const direction = order === 'ASC' ? '>' : '<';
    qb.where(`${qb.alias}.id ${direction}= :id`, { id });
    //        ↑ qb.alias 로 동적 테이블명
  }

  qb.orderBy(`${qb.alias}.id`, order);  // 정렬
  qb.take(dto.take);                     // LIMIT
}
```

---

---

#  execute() vs getMany() ⭐️

```txt
반환값에 따라 선택:

getMany()      → Entity[] 반환 (SELECT)
getOne()       → Entity | null 반환 (SELECT)
getRawMany()   → 원시 데이터 반환 (집계 등)
getCount()     → number 반환

execute()      → INSERT / UPDATE / DELETE 실행 결과 반환
                 { identifiers, generatedMaps, raw }
```

```typescript
// SELECT → getMany / getOne
const movies = await qb.getMany();
const movie  = await qb.where('id = :id', { id }).getOne();

// INSERT / UPDATE / DELETE → execute
const result = await qb.insert().into(Movie).values({...}).execute();
const result = await qb.update(Movie).set({...}).execute();
const result = await qb.delete().from(Movie).execute();
```

---

---

#  identifiers — INSERT 후 생성된 ID ⭐️

```typescript
// INSERT execute() 반환값 구조
const result = await this.movieRepository
  .createQueryBuilder()
  .insert()
  .into(Movie)
  .values({ title: '마이클', genre: 'drama' })
  .execute();

// result.identifiers = [{ id: 5 }]  ← 삽입된 행의 PK
const movieId = result.identifiers[0].id;
```

```txt
result 구조:
  {
    identifiers: [{ id: 5 }],       ← 삽입된 id 목록
    generatedMaps: [{ id: 5, ... }], ← DB 가 자동 생성한 컬럼
    raw: { ... }                      ← DB 원시 응답
  }

왜 identifiers 쓰나:
  INSERT 후 생성된 id 를 바로 알아야 할 때
  → 다음 쿼리에서 FK 로 사용
  → cascade 없이 수동으로 관계 연결할 때
```

## 실전 — INSERT 체인 패턴

```typescript
// 1. MovieDetail 먼저 INSERT
const detailResult = await this.movieDetailRepository
  .createQueryBuilder()
  .insert()
  .into(MovieDetail)
  .values({ detail: createMovieDto.detail })
  .execute();

const movieDetailId = detailResult.identifiers[0].id;  // ← 생성된 id

// 2. Movie INSERT (detail FK 포함)
const movieResult = await this.movieRepository
  .createQueryBuilder()
  .insert()
  .into(Movie)
  .values({
    title:    createMovieDto.title,
    detail:   { id: movieDetailId },  // ← detail FK
    director,
    genres,
  })
  .execute();

const movieId = movieResult.identifiers[0].id;  // ← 생성된 movie id
```

---

---

#  values() 에 뭘 넣나 ⭐️

```typescript
.values({
  // 일반 컬럼
  title:    'string 값',
  rating:   7.5,
  isActive: true,

  // 관계 (FK) — id 만 넣기
  detail:   { id: movieDetailId },  // OneToOne FK
  director: { id: directorId },     // ManyToOne FK

  // 배열 관계 (ManyToMany) → values 에 못 넣음 ⚠️
  // genres: genres  ← 이렇게 하면 안 됨
  // → 별도로 .relation() 으로 처리해야 함
})
```

```txt
values 에 넣을 수 있는 것:
  일반 컬럼 값    title / rating / isActive
  FK 관계        { id: xxx } 형태 (OneToOne / ManyToOne)

values 에 못 넣는 것:
  ManyToMany 관계 → 중간 테이블이 따로 있기 때문
  → .relation() 으로 별도 처리
```

---

---

# ManyToMany — relation() ⭐️

```txt
ManyToMany 는 중간 테이블 때문에
INSERT values() 에 바로 넣을 수 없음
→ .relation() 으로 중간 테이블 별도 관리
```

## add() — 관계 추가

```typescript
await this.movieRepository
  .createQueryBuilder()
  .relation(Movie, 'genres')   // Movie 의 genres 관계
  .of(movieId)                 // 어느 Movie 의
  .add(genres.map(g => g.id)); // genre id 들 추가
```

## addAndRemove() — 추가 + 삭제 동시

```typescript
await this.movieRepository
  .createQueryBuilder()
  .relation(Movie, 'genres')
  .of(movieId)
  .addAndRemove(
    newGenres.map(g => g.id),    // 추가할 genre id 들
    movie.genres.map(g => g.id)  // 제거할 genre id 들 (기존 것)
  );
// → 기존 관계 제거 + 새 관계 추가 (교체 패턴)
```

## remove() — 관계 제거

```typescript
await this.movieRepository
  .createQueryBuilder()
  .relation(Movie, 'genres')
  .of(movieId)
  .remove(genreId);
```

```txt
relation() 구조:
  .relation(Entity클래스, '프로퍼티명')  어떤 관계인지
  .of(id)                              대상 Entity id
  .add(id 또는 id[])                   관계 추가
  .remove(id 또는 id[])                관계 제거
  .addAndRemove(추가[], 제거[])         교체

언제 쓰나:
  ManyToMany 관계 수정 시
  INSERT 후 genres 연결 시
  PATCH 에서 genres 목록 교체 시
```


---
---
# 실전 

```typescript
// COUNT / SUM / AVG / MAX / MIN
const result = await this.movieRepository
  .createQueryBuilder('movie')
  .select('movie.genre', 'genre')
  .addSelect('COUNT(movie.id)', 'count')
  .addSelect('AVG(movie.rating)', 'avgRating')
  .groupBy('movie.genre')
  .having('COUNT(movie.id) > :min', { min: 5 })
  .getRawMany();
// result: [{ genre: 'drama', count: '12', avgRating: '8.5' }, ...]

// 단순 카운트
const count = await this.movieRepository
  .createQueryBuilder('movie')
  .where('movie.isActive = true')
  .getCount();
```

```txt
.select('컬럼', '별칭')     SELECT 컬럼 AS 별칭
.addSelect('집계식', '별칭') 집계 함수 추가
.groupBy('컬럼')            GROUP BY
.addGroupBy('컬럼')         GROUP BY 추가
.having('조건', 파라미터)    HAVING

getRawMany()   → 원시 데이터 (집계 결과는 원시로 받아야 함)
getMany()      → Entity 인스턴스 (집계 컬럼 없음)
```

---

---

#  Subquery — 서브쿼리 ⭐️

```typescript
// WHERE 절 서브쿼리
const movies = await this.movieRepository
  .createQueryBuilder('movie')
  .where(qb => {
    const subQuery = qb
      .subQuery()
      .select('director.id')
      .from(Director, 'director')
      .where('director.nationality = :country')
      .getQuery();
    return 'movie.directorId IN ' + subQuery;
  })
  .setParameter('country', 'Korea')
  .getMany();
```

```typescript
// FROM 절 서브쿼리 (인라인 뷰)
const result = await this.dataSource
  .createQueryBuilder()
  .select('sub.genre', 'genre')
  .addSelect('AVG(sub.rating)', 'avgRating')
  .from(subQuery => {
    return subQuery
      .select('movie.genre', 'genre')
      .addSelect('movie.rating', 'rating')
      .from(Movie, 'movie')
      .where('movie.isActive = true');
  }, 'sub')
  .groupBy('sub.genre')
  .getRawMany();
```

```txt
서브쿼리 패턴:
  WHERE IN 서브쿼리:
    qb => { const sub = qb.subQuery()...getQuery(); return 'id IN ' + sub; }

  FROM 서브쿼리:
    .from(subQuery => subQuery.select()...from()..., 'alias')
```

---

---

# Raw Query — 직접 SQL 실행 ⭐️

```typescript
// query() — SQL 직접 실행
const result = await this.dataSource.query(
  `SELECT * FROM movie WHERE id = $1`,
  [1]     // 파라미터 바인딩 (SQL 인젝션 방지)
);

// 복잡한 집계 쿼리
const stats = await this.dataSource.query(`
  SELECT
    genre,
    COUNT(*) AS count,
    ROUND(AVG(rating)::numeric, 2) AS avg_rating
  FROM movie
  WHERE is_active = true
  GROUP BY genre
  ORDER BY count DESC
`);
```

```typescript
// getRepository + query
const result = await this.movieRepository.query(
  `SELECT m.*, d.name as director_name
   FROM movie m
   LEFT JOIN director d ON m.director_id = d.id
   WHERE m.id = $1`,
  [movieId]
);
```

```txt
Raw Query 언제 쓰나:
  QueryBuilder 로 표현하기 너무 복잡한 쿼리
  PostgreSQL 전용 문법 (DISTINCT ON, 배열 함수 등)
  성능 최적화가 중요한 쿼리
  기존 SQL 을 그대로 포팅할 때

  $1, $2 = PostgreSQL 파라미터 (SQL 인젝션 방지)
  ?       = MySQL 파라미터
```

---

---

# 한눈에

|상황|선택|
|---|---|
|단순 CRUD|Repository|
|조건 동적으로 추가|QueryBuilder ⭐️|
|복잡한 JOIN|QueryBuilder|
|GROUP BY / 집계|QueryBuilder + getRawMany()|
|WHERE IN 서브쿼리|QueryBuilder subQuery()|
|PostgreSQL 전용 문법|Raw Query|
|INSERT 후 id 필요|execute() + identifiers[0].id ⭐️|
|ManyToMany 관계 추가/교체|.relation().of().addAndRemove() ⭐️|