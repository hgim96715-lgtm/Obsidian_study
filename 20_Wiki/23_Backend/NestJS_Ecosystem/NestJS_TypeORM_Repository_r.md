---
aliases: [delete, find, FindOperators, In, Not, relations, Repository CRUD, save, TypeORM Repository, upsert, where]
tags:
  - NestJS
related:
  - "[[00_NestJS_Ecosystem_HomePage]]"
---
# NestJS_TypeORM_Repository — Repository CRUD

## 한 줄 요약

```txt
Repository = 특정 Entity 에 대한 CRUD 쿼리 담당
SQL 직접 작성 없이 메서드로 DB 조작
```

---

---

# ① Repository 란 ⭐️


```typescript
// DataSource 에서 Repository 가져오기
const userRepository = dataSource.getRepository(User);

// 조회 후 수정 저장
const user = await userRepository.findOneBy({ id: 1 });
user.name = 'Code Factory';
await userRepository.save(user);
```

```txt
Repository 역할:
  지정한 Entity 에 대한 CRUD 쿼리를 할 수 있게 해줌
  TypeORM 에 정의된 메서드 사용
  → SQL 직접 작성 불필요
```

## NestJS 에서 Repository 사용 흐름 ⭐️

```txt
1. Module 에서 forFeature 로 Entity 등록
   → Repository 생성됨

2. Service 에서 @InjectRepository 로 주입
   → Repository 사용 가능
```

>자세한 설명 → [[NestJS_TypeORM_Entity#④ forRoot vs forFeature ⭐️]]  참고 

## ① forFeature 로 Entity 등록 (ex) movie.module.ts)

```typescript
@Module({
  imports: [TypeOrmModule.forFeature([Movie])],
  //                                  ↑ Movie Repository 생성 
  providers: [MovieService],
})
export class MovieModule {}
```

## ② @InjectRepository 로 주입 (movie.service.ts) ⭐️

```typescript
import { InjectRepository } from '@nestjs/typeorm';
import { Repository } from 'typeorm';
import { Movie } from './entities/movie.entity';

@Injectable()
export class MovieService {
  constructor(
    @InjectRepository(Movie) 
    private readonly movieRepository: Repository<Movie>,
    //               ↑ 이름은 마음대로 / 보통 Entity명+Repository //[[NestJS_TypeORM_Repository]]
  ) {}

  // 이후 this.movieRepository 로 아래 메서드 전부 사용
  async findOne(id: number) {
    const movie = await this.movieRepository.findOne({ where: { id } });
    //                       ↑ 이 노트 예시의 repository = this.movieRepository
    if (!movie) throw new NotFoundException(`${id} 없음`);
    return movie;
  }
}
```

```txt
@InjectRepository(Movie):
  NestJS DI 컨테이너에서 Movie 의 Repository 를 주입해달라는 표시
  forFeature([Movie]) 로 등록된 Repository 를 가져옴

private readonly movieRepository: Repository<Movie>:
  TypeScript Parameter Property 축약 문법
  → 속성 선언 + this.movieRepository = 값 자동 처리
  → [[TS_Class#Parameter Property]] 참고

⚠️ forFeature 없이 @InjectRepository 만 쓰면:
  "No repository for Movie was found" 에러
  → forFeature([Movie]) 등록이 반드시 선행
```

---

---

# ② Repository 주요 메서드 목록

|분류|메서드|
|---|---|
|Create & Delete|`create()` / `save()` / `upsert()` / `delete()` / `softDelete()` / `restore()`|
|Update|`update()` / `increment()` / `decrement()`|
|Find|`find()` / `findAndCount()` / `findOne()` / `findOneBy()` / `exists()` / `query()`|
|통계|`count()` / `sum()` / `average()` / `minimum()` / `maximum()`|

---

---

# ③ create() — 객체 생성 (DB 저장 안 함) ⭐️

```typescript
// new User() 와 동일 — DB 저장 안 함!
const user = repository.create();

// 초기값 포함
const user = repository.create({
  id: 1,
  firstName: 'Timber',
  lastName: 'Saw',
});
```

```txt
create() vs save():
  create()  객체만 생성 (메모리에만 존재 / DB 저장 안 됨)
  save()    실제 DB 에 저장

  create() 로 객체 만들고 → save() 로 저장하는 패턴이 일반적
  create() 는 new Entity() 의 TypeORM 버전
```

---

---

# ④ save() — DB 저장 / 업데이트 ⭐️

```typescript
// 단일 저장
await repository.save(user);

// 여러 개 한번에 저장
await repository.save([category1, category2, category3]);
```

```txt
save() 주의사항 ⚠️:
  이미 Row 가 존재하면 (primary key 기준) → UPDATE
  없으면 → INSERT

  즉, save() 는 INSERT + UPDATE 를 모두 함
  이미 있는 id 를 save 하면 덮어씌워짐!

  update 용도로 쓸 때는 의도한 것인지 확인할 것
```

---

---

# ⑤ upsert() — INSERT + UPDATE 동시 ⭐️

```typescript
await repository.upsert(
  [
    { externalId: 'abc123', firstName: 'Code' },
    { externalId: 'bca321', firstName: 'Factory' },
  ],
  ['externalId'],   // 충돌 감지 기준 컬럼
);

// 실행되는 SQL:
// INSERT INTO user VALUES (externalId=abc123, firstName=Code), ...
// ON CONFLICT (externalId) DO UPDATE firstName = EXCLUDED.firstName
```

```txt
upsert() vs save():
  save()    primary key 기준으로 insert/update
            여러 번 실행 시 각각 별도 트랜잭션

  upsert()  지정한 컬럼 기준으로 충돌 감지
            하나의 transaction 에서 실행 → 더 안전
            외부 ID (externalId) 기반으로 동기화할 때 유용
```

---

---

# ⑥ delete() — 삭제

```typescript
// Primary Key 로 삭제
await repository.delete(1);

// 여러 개 삭제
await repository.delete([1, 2, 3]);

// 조건으로 삭제 (findOptionsWhere)
await repository.delete({ firstName: 'Timber' });
```

---

---

# ⑦ find / findOne — 조회 ⭐️

```typescript
// 전체 조회
const users = await repository.find();

// 조건 조회
const users = await repository.find({
  where: { isActive: true },
  order: { firstName: 'ASC' },
  take: 10,    // LIMIT
  skip: 0,     // OFFSET
});

// 단건 조회
const user = await repository.findOne({
  where: { id: 1 },
});

// 단건 조회 (where 단축형)
const user = await repository.findOneBy({ id: 1 });

// 없으면 null 반환 → null 체크 필수
if (!user) throw new NotFoundException('없음');
```

---

---

# ⑧ NestJS Service 에서 Repository 패턴

```typescript
// movie.module.ts — forFeature 로 Entity 등록
@Module({
  imports: [TypeOrmModule.forFeature([Movie])],
  providers: [MovieService],
})
export class MovieModule {}

// movie.service.ts — @InjectRepository 로 주입
import { InjectRepository } from '@nestjs/typeorm';
import { Repository } from 'typeorm';

@Injectable()
export class MovieService {
  constructor(
    @InjectRepository(Movie)
    private movieRepository: Repository<Movie>,
  ) {}

  // 전체 조회
  async findAll(): Promise<Movie[]> {
    return this.movieRepository.find();
  }

  // 단건 조회
  async findOne(id: number): Promise<Movie> {
    const movie = await this.movieRepository.findOneBy({ id });
    if (!movie) throw new NotFoundException(`${id} 없음`);
    return movie;
  }

  // 생성
  async create(title: string): Promise<Movie> {
    const movie = this.movieRepository.create({ title });
    return this.movieRepository.save(movie);
  }

  // 수정
  async update(id: number, title: string): Promise<Movie> {
    const movie = await this.findOne(id);
    movie.title = title;
    return this.movieRepository.save(movie);
  }

  // 삭제
  async remove(id: number): Promise<void> {
    await this.movieRepository.delete(id);
  }
}
```

---

---

# ⑨ softDelete() / restore() — 소프트 삭제 ⭐️

```typescript
// 비영구 삭제 (DeleteDateColumn 필요)
await repository.softDelete(1);
// → deletedAt 컬럼에 현재 시간 기록 / 행은 그대로 유지
// → 이후 find() 에서 자동으로 제외됨

// 복구
await repository.restore(1);
// → deletedAt 을 null 로 업데이트 → 다시 조회 가능
```

```typescript
// Entity 에 @DeleteDateColumn 필요
@Entity()
export class Movie {
  @PrimaryGeneratedColumn()
  id: number;

  @DeleteDateColumn()
  deletedAt: Date;   // null = 정상 / 값 있음 = 삭제됨
}
```

```txt
softDelete vs delete:
  delete()      Row 를 실제로 삭제 (복구 불가)
  softDelete()  deletedAt 에 시간만 기록 (복구 가능)

언제 softDelete:
  실수로 삭제했을 때 복구 필요한 경우
  삭제 이력을 남겨야 하는 경우
  관계 테이블 정합성 유지
```

---

---

# ⑩ update() — 직접 UPDATE

```typescript
// update(조건, 변경값)
await repository.update(1, { firstName: 'Updated' });

// 조건으로 여러 개 수정
await repository.update(
  { firstName: 'Timber' },
  { isActive: false }
);
```

```txt
update() vs save():
  save()    Entity 인스턴스 필요 → 조회 후 수정
  update()  조건만 알면 바로 UPDATE → 조회 없이 수정

  save()   : find → 수정 → save (2번 DB 접근)
  update() : 조건 + 변경값만 전달 (1번 DB 접근)
  → 성능 면에서 update() 가 유리
```

---

---

# ⑪ exists() — 존재 여부 확인

```typescript
// 조건에 맞는 Row 가 있는지 boolean 반환
const exists = await repository.exists({
  where: { email: 'test@email.com' }
});
// true / false

// 활용 패턴: 중복 체크
if (await repository.exists({ where: { email } })) {
  throw new ConflictException('이미 존재하는 이메일');
}
```

---

---

# ⑫ findAndCount() — 조회 + 개수

```typescript
const [movies, total] = await repository.findAndCount({
  where: { genre: 'drama' },
  take: 10,
  skip: 0,
});
// movies: Movie[]  조회된 데이터
// total: number    전체 개수 (take/skip 무시)
```

```txt
페이징 API 에서 자주 씀:
  데이터 목록 + 전체 개수 한번에 필요할 때
  → 프론트에서 페이지 수 계산 가능
```

---

---

# ⑬ preload() — 기존 데이터 + 입력값 병합

```typescript
const movie = await repository.preload({
  id: 1,              // Primary Key 기준으로 DB 에서 조회
  title: '수정 제목', // 이 값으로 title 덮어씌움
});
// → DB 에서 id=1 조회 + title 을 '수정 제목' 으로 덮어씌운 객체 반환
// ⚠️ DB 업데이트는 안 됨 → save() 로 저장해야 함
```

```txt
preload() 패턴:
  1. preload() 로 기존 값 + 새 값 병합된 객체 생성
  2. save() 로 실제 저장

  PATCH 요청 처리 시 유용:
  일부 필드만 넘어와도 기존 값 유지 + 새 값 적용
```

---

---

# ⑭ FindOptions — 조회 옵션 ⭐️

```typescript
// FindOneOptions / FindManyOptions 에 공통 적용
await repository.find({
  where: { isActive: true, genre: 'drama' },   // 조건
  relations: ['user', 'comments'],              // 관계 테이블 함께 조회
  order: { createdAt: 'DESC' },                 // 정렬
  select: ['id', 'title'],                      // 특정 컬럼만
  cache: 1000,                                  // 캐싱 (ms)
});

// FindManyOptions 추가 옵션
await repository.find({
  take: 10,    // LIMIT , 10개 가져오겠다는 뜻
  skip: 20,    // OFFSET , 20개 스킵하고 21개부터 
});
```

| 옵션          | 설명                       |
| ----------- | ------------------------ |
| `where`     | 필터링 조건                   |
| `relations` | 함께 조회할 관계 테이블            |
| `order`     | 정렬 기준                    |
| `select`    | 특정 컬럼만 조회                |
| `cache`     | 캐싱 시간 (ms)               |
| `take`      | LIMIT (FindManyOptions)  |
| `skip`      | OFFSET (FindManyOptions) |

```txt
FindOneOptions vs FindManyOptions:
  FindOneOptions   단건 조회 (findOne / findOneBy)
  FindManyOptions  복수 조회 (find) → + take / skip 추가
```

---

---

## relations — 관계 테이블 선택 조회 ⭐️

```txt
relations 를 지정하지 않으면:
  관계 Entity 데이터를 가져오지 않음 (성능 최적화)

relations 를 지정하면:
  SQL JOIN 처럼 관계 데이터를 함께 가져옴
```

## TypeORM 버전별 relations 문법 ⭐️

```typescript
// ❌ TypeORM 0.2.x 문자열 배열 방식 (1.x 에서 타입 오류)
relations: ['detail', 'director', 'genres']

// ✅ TypeORM 1.x 객체 방식 (FindOptionsRelations)
relations: { detail: true, director: true, genres: true }
```

```txt
TypeORM 1.0.0 부터:
  relations 타입이 string[] 아님
  → FindOptionsRelations<Entity> 객체 타입
  → { detail: true, director: true } 형태

  문자열 배열로 쓰면:
  "Type 'string[]' is not assignable to type 'FindOptionsRelations<Movie>'"
  에러 발생

  설치된 버전 확인:
  cat package.json | grep typeorm
```

## 기본 사용


```typescript
// TypeORM 1.x 객체 방식
await this.movieRepository.findOne({
  where: { id: movieId },
  relations: { detail: true, director: true, genres: true },
});

// 여러 개 한번에
await this.movieRepository.find({
  relations: { detail: true, director: true, genres: true },
});
```

## 실전 — 목록 vs 상세 페이지 ⭐️

```typescript
// 목록 조회 — detail 불필요 (성능 최적화)
async getMovies() {
  return this.movieRepository.find();
  // id / title / genre 만 반환
}

// 상세 조회 — detail 포함해서 반환
async getMovieById(id: number) {
  const movie = await this.movieRepository.findOne({
    where: { id },
    relations: { detail: true },   // ← 1.x 객체 방식
  });
  if (!movie) throw new NotFoundException(`${id} 없음`);
  return movie;
}
```

```txt
왜 목록에서 relations 를 안 쓰나:
  목록 = 카드/리스트 → title, genre 만 필요
  상세 = 클릭해서 들어간 페이지 → detail 도 필요

  relations 쓰면 JOIN 쿼리 실행 → 데이터 더 많이 가져옴
  → 불필요한 곳에 쓰면 성능 낭비
  → 필요한 곳에서만 선택적으로 사용
```

## relations 로드 후 배열 체크 ⭐️

```typescript
// Genre 삭제 시 연결된 Movie 있으면 삭제 불가 패턴
async remove(id: number) {
  const genre = await this.genreRepository.findOne({
    where: { id },
    relations: { movies: true }   // movies 배열 함께 로드
  });

  if (!genre) {
    throw new NotFoundException('존재하지 않는 장르입니다.');
  }

  // relations 로 로드했으면 빈 배열 [] 로 확인 가능
  if (genre.movies.length > 0) {
    throw new ConflictException(
      `영화에서 사용중인 장르는 삭제할 수 없습니다.
      연결된 영화 ID: ${genre.movies.map(movie => movie.id).join(',')}`
    );
  }

  await this.genreRepository.delete(id);
  return `${id}가 삭제되었습니다.`;
}
```

```txt
relations: { movies: true } 로 로드했을 때:
  연결된 movie 없음 → movies = []  (빈 배열)
  연결된 movie 있음 → movies = [Movie{}, Movie{}, ...]

  → genre.movies.length > 0 으로 바로 체크 가능
  → relations 로드 후에는 movies 가 undefined 아님

왜 genre.movies?.length ?? 0 을 쓰면 안 좋은가:
  relations 없이 조회했을 때 movies = undefined 가 될 수 있음
  → 그래서 처음에 옵셔널 체이닝 + nullish 병합 쓰고 싶어짐

  하지만 relations: { movies: true } 로 명시적으로 로드하면
  movies 는 반드시 배열 (undefined 아님)
  → .length 직접 접근 가능 / ?. 불필요

  결론:
    relations 로드 O → genre.movies.length         ✅ 명확
    relations 로드 X → genre.movies?.length ?? 0   ⚠️ 방어적
```

## 중첩 관계 조회

```typescript
// 관계의 관계도 가져올 때 (1.x 객체 방식)
const movie = await this.movieRepository.findOne({
  where: { id },
  relations: {
    detail: { images: true },   // detail 안의 images 도 함께
  },
});

// 여러 관계 동시에
const movie = await this.movieRepository.findOne({
  where: { id: movieId },
  relations: { detail: true, director: true, genres: true },
});
```


---
---


|메서드|DB 실행|설명|
|---|:-:|---|
|`create()`|❌|객체만 생성|
|`save()`|✅|INSERT / UPDATE (id 기준)|
|`upsert()`|✅|INSERT / UPDATE (지정 컬럼 기준) 단일 트랜잭션|
|`delete()`|✅|삭제|
|`find()`|✅|전체 / 조건 조회|
|`findOne()`|✅|단건 조회|
|`findOneBy()`|✅|단건 조회 단축형|
|`count()`|✅|건수|
|`update()`|✅|직접 UPDATE|

---

---

# ⑮ FindOperators — 고급 조건 ⭐️

```typescript
import {
  Equal, Not, LessThan, LessThanOrEqual,
  MoreThan, MoreThanOrEqual, Between,
  Like, ILike, In, IsNull, Or, And,
  ArrayContains, ArrayContainedBy, ArrayOverlap,
} from 'typeorm';
```

## 비교 오퍼레이터

```typescript
// Equal — 같은 값 (기본값과 동일)
where: { age: Equal(25) }
where: { age: 25 }   // 동일

// Not — 아닌 값
where: { genre: Not('drama') }
```

## Not — 중복 체크 실전 패턴 ⭐️

```typescript
// PATCH 시 자기 자신 제외하고 이름 중복 확인
const duplicate = await this.genreRepository.findOne({
  where: {
    name: updateGenreDto.name,
    id: Not(id)     // 현재 수정 중인 자신은 제외
  }
});
if (duplicate) throw new ConflictException('이미 존재하는 이름');
```

```txt
왜 Not(id) 가 필요한가:
  PATCH /genre/1 { name: "drama" }
  자기 자신(id=1) 이름이 "drama" 인데 그대로 업데이트하는 경우

  Not(id) 없으면:
    name = 'drama' 검색 → 자기 자신 나옴 → "중복!" 에러
    → 이름 변경 없이 다른 필드만 수정해도 에러 발생

  Not(id) 있으면:
    id != 1 이면서 name = 'drama' 인 것 검색
    → 자기 자신 제외 → 다른 row 에만 중복 있으면 에러

SQL 로 보면:
  WHERE name = 'drama' AND id != 1
```

```typescript
// LessThan / LessThanOrEqual — 미만 / 이하
where: { age: LessThan(30) }         // age < 30
where: { age: LessThanOrEqual(30) }  // age <= 30

// MoreThan / MoreThanOrEqual — 초과 / 이상
where: { age: MoreThan(20) }         // age > 20
where: { age: MoreThanOrEqual(20) }  // age >= 20

// Between — 사이 값
where: { age: Between(20, 30) }      // 20 <= age <= 30
```

---

## 패턴 매칭 오퍼레이터

```typescript
// Like — 대소문자 구분
where: { title: Like('%마이클%') }    // title LIKE '%마이클%'
where: { title: Like('마%') }         // 마 로 시작

// ILike — 대소문자 구분 안 함 (PostgreSQL)
where: { title: ILike('%MICHAEL%') }  // 대소문자 무관
```

---

## 집합 오퍼레이터

```typescript
// In — 목록 중 하나
where: { genre: In(['drama', 'action', 'comedy']) }
// genre IN ('drama', 'action', 'comedy')
```

## In — 여러 id 한번에 조회 ⭐️

```typescript
import { In } from 'typeorm';

// DTO 로 받은 여러 id 로 한번에 조회
const genres = await this.genreRepository.find({
  where: { id: In(createMovieDto.genreIds) }
  // genreIds = [1, 2, 3] → WHERE id IN (1, 2, 3)
});
```

```txt
언제 쓰나:
  프론트에서 genreIds: [1, 2, 3] 배열로 받았을 때
  루프로 하나씩 조회 (N번) 대신
  In 으로 한 번에 조회 (1번) → 성능 좋음

  // ❌ N번 조회 (느림)
  for (const id of genreIds) {
    const genre = await genreRepository.findOneBy({ id });
  }

  // ✅ In 으로 한 번에 (빠름)
  const genres = await genreRepository.find({
    where: { id: In(genreIds) }
  });
```

## In + 존재 여부 검증 패턴 ⭐️

```typescript
async createMovie(createMovieDto: CreateMovieDto) {
  // 1. 요청한 id 로 실제 존재하는 것만 조회
  const genres = await this.genreRepository.find({
    where: { id: In(createMovieDto.genreIds) }
  });

  // 2. 조회된 수 vs 요청한 수 비교
  if (genres.length !== createMovieDto.genreIds.length) {
    throw new NotFoundException(
      `존재하지 않는 장르가 있습니다. 존재하는 ids → ${genres.map(g => g.id).join(',')}`
    );
  }

  // 3. 모두 존재하면 진행
  const movie = await this.movieRepository.save({
    title: createMovieDto.title,
    genres,
  });
  return movie;
}
```

```txt
검증 패턴 핵심:
  요청: genreIds = [1, 2, 99]  (99는 없는 id)
  조회: genres = [Genre{id:1}, Genre{id:2}]  (2개만 나옴)

  genres.length (2) !== genreIds.length (3)
  → 불일치 → 존재하지 않는 id 가 있다는 뜻
  → NotFoundException 던짐

  일치하면 (3 === 3) → 모두 존재 → 진행
```

---

## Array 오퍼레이터 (PostgreSQL array 타입) ️

```txt
예시 데이터:
  Record 1: tags = ['a', 'b']
  Record 2: tags = ['b', 'c']
  필터: ['b', 'c']

ArrayContains(['b', 'c'])
  → 엔티티 tags 가 필터와 완전히 같은 것
  → Record 2 만 반환 (['b', 'c'] = ['b', 'c'])

ArrayContainedBy(['b', 'c'])
  → 엔티티 tags 가 필터 안에 모두 포함되는 것
  → Record 1 (['a','b'] → 'a' 가 필터에 없으므로 ❌)
  → Record 2 (['b','c'] → 모두 포함 ✅)

ArrayOverlap(['b', 'c'])
  → 엔티티 tags 와 필터가 겹치는 것이 하나라도 있는 것
  → Record 1 ('b' 겹침 ✅), Record 2 ('b','c' 겹침 ✅) 둘 다 반환
```

```typescript
where: { tags: ArrayContains(['b', 'c']) }
where: { tags: ArrayContainedBy(['b', 'c']) }
where: { tags: ArrayOverlap(['b', 'c']) }
```

## 기타 오퍼레이터

```typescript
// IsNull — NULL 값 필터
where: { deletedAt: IsNull() }       // deletedAt IS NULL

// Or — OR 조건
where: Or(
  { genre: 'drama' },
  { genre: 'action' }
)
// genre = 'drama' OR genre = 'action'

// And — AND 조건 (복합 조건)
where: And(
  MoreThan(5),
  LessThan(10)
)
// > 5 AND < 10
```

---

---

# ⑯ 통계 메서드

```typescript
// count — 해당되는 개수
const total = await repository.count({
  where: { isActive: true }
});

// sum — 컬럼 값 합계
const totalPrice = await repository.sum('price', { isActive: true });

// average — 컬럼 값 평균
const avgAge = await repository.average('age', { genre: 'drama' });

// minimum — 최솟값
const minPrice = await repository.minimum('price');

// maximum — 최댓값
const maxPrice = await repository.maximum('price');
```

|메서드|설명|
|---|---|
|`count(options)`|조건에 맞는 개수|
|`sum(컬럼, where)`|컬럼 합계|
|`average(컬럼, where)`|컬럼 평균|
|`minimum(컬럼, where)`|최솟값|
|`maximum(컬럼, where)`|최댓값|
