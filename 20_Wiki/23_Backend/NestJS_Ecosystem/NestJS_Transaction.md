---
aliases:
  - 트랜잭션
  - Transaction
  - BEGIN
  - COMMIT
  - ROLLBACK
  - DataSource
tags:
  - NestJS
related:
  - "[[00_NestJS_Ecosystem_HomePage]]"
---
# NestJS_Transaction — 트랜잭션

## 한 줄 요약

```
여러 오퍼레이션을 하나의 논리적인 작업으로 묶기
전부 성공 or 전부 실패 (원자성)
은행 거래처럼: 출금 + 입금이 같이 성공해야 함
```

---

---

# ① 트랜잭션이란 ⭐️

```
예시: 은행 이체
  1. A 계좌에서 10만원 출금
  2. B 계좌에 10만원 입금

  중간에 에러 발생 시:
  트랜잭션 없음 → A 출금됐는데 B 미입금 (돈 증발)
  트랜잭션 있음 → 전부 취소 → A 원상복구

트랜잭션 3요소:
  BEGIN     트랜잭션 시작
  COMMIT    모두 성공 → 영구 반영
  ROLLBACK  하나라도 실패 → 전부 취소
```

---

---

# ② 트랜잭션 문제점 4가지 ⭐️

## Lost Update (업데이트 손실)

```
두 트랜잭션이 같은 데이터를 읽고 업데이트

  T1: 재고 10 읽음 → 9 로 수정 시도
  T2: 재고 10 읽음 → 8 로 수정 시도
  T2 먼저 커밋: 재고 = 8
  T1 커밋: 재고 = 9  ← T2 의 작업 유실!

해결: Optimistic Lock (낙관적 잠금)
```

## Dirty Read (더티 읽기)

```
아직 커밋 안 된 데이터를 다른 트랜잭션이 읽는 경우

  T1: 잔액 1000 → 2000 으로 변경 (아직 커밋 안 함)
  T2: 잔액 2000 읽음 (커밋 안 된 값)
  T1: ROLLBACK → 잔액 다시 1000
  T2: 2000 기준으로 로직 진행 → 잘못된 결과

해결: Read Committed 격리 수준
```

## Non-repeatable Read (반복 불가 읽기)

```
같은 데이터를 두 번 읽었는데 결과가 다른 경우

  T1: 잔액 1000 읽음
  T2: 잔액 1000 → 2000 변경 + 커밋
  T1: 잔액 다시 읽음 → 2000  (처음과 다름!)

해결: Repeatable Read 격리 수준
```

## Phantom Read (팬텀 읽기)

```
조건에 맞는 행을 조회했는데 나중에 다시 조회하면 행 수가 다른 경우

  T1: SELECT * FROM orders WHERE amount > 1000  → 3건
  T2: 새 주문 (amount=1500) INSERT + 커밋
  T1: 같은 쿼리 다시 실행 → 4건  (새 행이 생겼음!)

해결: Serializable 격리 수준
```

---

---

# ③ 격리 수준 (Isolation Level) ⭐️

|격리 수준|Dirty Read|Non-repeatable Read|Phantom Read|
|---|:-:|:-:|:-:|
|Read Uncommitted|발생|발생|발생|
|Read Committed|✅ 방지|발생|발생|
|Repeatable Read|✅ 방지|✅ 방지|발생|
|Serializable|✅ 방지|✅ 방지|✅ 방지|

```
격리 수준 높을수록:
  더 안전 / 더 느림 (성능 저하)

기본값:
  PostgreSQL: Read Committed
  MySQL InnoDB: Repeatable Read

실무:
  대부분 기본값 사용
  특수 상황에서만 조정
```

## startTransaction 에 격리 수준 지정 ⭐️

```typescript
// 기본값 (인자 없음) = DB 기본 격리 수준
await qr.startTransaction();
// PostgreSQL 기본 → READ COMMITTED

// 격리 수준 직접 지정
await qr.startTransaction('READ UNCOMMITTED');  // 거의 안 씀
await qr.startTransaction('READ COMMITTED');    // PostgreSQL 기본
await qr.startTransaction('REPEATABLE READ');   // Non-repeatable Read 방지
await qr.startTransaction('SERIALIZABLE');      // 가장 엄격 / 가장 느림
```

```
언제 기본값 외 격리 수준을 쓰나:

  REPEATABLE READ:
    한 트랜잭션 안에서 같은 데이터를 두 번 읽는 로직
    재고 확인 → 주문 처리 (중간에 다른 트랜잭션이 재고 변경 못 하게)

  SERIALIZABLE:
    팬텀 읽기까지 막아야 할 때
    금융 / 회계 / 재고 정확성이 매우 중요한 경우
    성능 저하가 크므로 꼭 필요한 곳에만 사용

  실무 대부분:
    await qr.startTransaction()  인자 없이 기본값 사용
```

---

---

# ④ TypeORM 트랜잭션 문법 ⭐️

## 방법 1 — QueryRunner (가장 많이 씀) ⭐️

## DataSource 란? ⭐️

```
Repository = 특정 Entity 하나의 CRUD 담당
  @InjectRepository(Movie) → Movie 테이블만

DataSource = TypeORM 의 DB 연결 전체 관리자
  여러 Entity 에 걸친 트랜잭션 처리 가능
  createQueryRunner() → 트랜잭션용 QueryRunner 생성
  query()             → Raw SQL 실행
  getRepository()     → 특정 Entity Repository 가져오기

왜 트랜잭션에 DataSource 가 필요한가:
  트랜잭션 = "여러 쿼리를 하나의 단위로"
  Movie INSERT + MovieDetail INSERT 를 동시에 묶어야 함
  → 특정 Entity Repository 로는 불가
  → DataSource 에서 QueryRunner 를 꺼내서 처리
```

## DataSource 주입

```typescript
import { DataSource } from 'typeorm';

@Injectable()
export class MovieService {
  constructor(
    @InjectRepository(Movie)
    private movieRepository: Repository<Movie>,   // Movie 만 담당

    private dataSource: DataSource,               // DB 전체 관리자
    // ↑ @InjectRepository 같은 데코레이터 없이 바로 주입 가능
    // AppModule 에 TypeOrmModule 등록하면 자동으로 DI 컨테이너에 등록됨
  ) {}
}
```

```
@InjectRepository 는 왜 없나:
  Repository 는 Entity 마다 따로 생성되므로
  @InjectRepository(Movie) 처럼 어떤 Entity 인지 명시 필요

  DataSource 는 TypeORM 전체 연결 객체 하나
  → NestJS DI 컨테이너에 하나만 등록
  → 타입(DataSource) 만으로 자동 주입 가능
```

## QueryRunner 전체 패턴

```typescript
async createMovie(createMovieDto: CreateMovieDto) {
  // 1. QueryRunner 생성
  const qr = this.dataSource.createQueryRunner();

  // 2. DB 연결 + 트랜잭션 시작
  await qr.connect();
  await qr.startTransaction();
  // 기본값: 'READ COMMITTED' (PostgreSQL 기본 격리 수준)

  // 격리 수준 지정하려면:
  // await qr.startTransaction('READ UNCOMMITTED');
  // await qr.startTransaction('READ COMMITTED');   ← PostgreSQL 기본
  // await qr.startTransaction('REPEATABLE READ');
  // await qr.startTransaction('SERIALIZABLE');     ← 가장 엄격


  try {
    // ─── 트랜잭션 내 작업 ───────────────────────────────

    // 3. qr.manager 로 모든 DB 작업 실행
    const genres = await qr.manager.find(Genre, {
      where: { id: In(createMovieDto.genreIds) }
    });
    //             ↑      ↑
    //         manager  Entity 클래스 직접 전달

    const detail = await qr.manager.save(MovieDetail, {
      detail: createMovieDto.detail
    });

    const movieResult = await qr.manager
      .createQueryBuilder()     // ← qr.manager 에서 바로 QueryBuilder
      .insert()
      .into(Movie)
      .values({
        title:    createMovieDto.title,
        detail,
        director,
        genres,
      })
      .execute();

    const movieId = movieResult.identifiers[0].id;

    // ManyToMany 관계 연결
    await qr.manager
      .createQueryBuilder()
      .relation(Movie, 'genres')
      .of(movieId)
      .add(genres.map(g => g.id));

    // ─── 모두 성공 → COMMIT ─────────────────────────────
    await qr.commitTransaction();

    // 4. return 은 트랜잭션 밖에서 일반 repository 사용 가능
    return await this.movieRepository.findOne({
      where: { id: movieId },
      relations: ['detail', 'director', 'genres']
    });

  } catch (error) {
    // 하나라도 실패 → ROLLBACK
    await qr.rollbackTransaction();
    throw error;

  } finally {
    // 반드시 연결 해제 ⚠️
    await qr.release();
  }
}
```

>각각의 문법들은 [[NestJS_TypeORM_QueryBuilder]] 참조 

## qr.manager 란 ⭐️

```
qr.manager = EntityManager
  QueryRunner 에 속한 DB 작업 실행기
  이 manager 로 실행하는 모든 작업이 같은 트랜잭션에 묶임

qr.manager.save(Entity클래스, 데이터)
  첫 번째 인자: Entity 클래스 직접 전달 (Genre, Movie 등)
  두 번째 인자: 저장할 데이터

  일반 Repository.save() 와 차이:
    this.movieRepository.save()   → 트랜잭션 밖 (별도 연결)
    qr.manager.save(Movie, ...)   → 트랜잭션 안 (같은 연결)

qr.manager.find(Entity클래스, 옵션)
  일반 Repository.find() 와 동일하지만
  트랜잭션 안에서 실행됨

qr.manager.createQueryBuilder()
  별도 Repository 없이 QueryBuilder 시작
  → this.movieRepository.createQueryBuilder() 와 같은 결과
  → qr 에서 직접 실행하므로 트랜잭션에 포함됨
```

## return 은 일반 repository 써도 됨 ⭐️

```typescript
// COMMIT 후 return 은 트랜잭션 밖 → 일반 repository 사용 가능
return await this.movieRepository.findOne({
  where: { id: movieId },
  relations: ['detail', 'director', 'genres']
});

// 왜 qr.manager 로 안 하나:
//   COMMIT 이미 됐으므로 트랜잭션 안에 있을 필요 없음
//   일반 repository 가 더 간단하고 가독성 좋음
//   qr.release() 후에는 qr.manager 사용 불가
```

## ⚠️ 자주 하는 실수

```
release() 빠뜨리기:
  finally 없으면 에러 시 release() 안 됨
  → 커넥션 풀 고갈 → 전체 서버 장애
  → 반드시 finally { await qr.release(); }

일반 repository 와 qr.manager 섞기:
  this.movieRepository.save()   ← 트랜잭션 밖
  qr.manager.save(Movie, ...)   ← 트랜잭션 안
  → 둘 다 쓰면 일관성 깨짐
  → 트랜잭션 내부에서는 반드시 qr.manager 로만
```

## 방법 2 — transaction() 헬퍼

```typescript
async createMovie(dto: CreateMovieDto) {
  return await this.dataSource.transaction(async (manager) => {
    // manager 로 작업 (자동으로 트랜잭션 묶임)
    const detail = await manager.save(MovieDetail, { detail: dto.detail });
    const movie  = await manager.save(Movie, { title: dto.title, detail });

    return movie;
    // 에러 없으면 자동 COMMIT
    // 에러 발생 시 자동 ROLLBACK
  });
}
```

```
방법 1 vs 방법 2:
  QueryRunner  더 세밀한 제어 가능 / 중간에 상태 확인 가능
  transaction()  코드 간결 / 자동 커밋/롤백
  → 복잡한 로직은 QueryRunner / 단순하면 transaction()
```

---

---

# ⑤ QueryBuilder + 트랜잭션

```typescript
const queryRunner = this.dataSource.createQueryRunner();
await queryRunner.connect();
await queryRunner.startTransaction();

try {
  // QueryBuilder 도 queryRunner.manager 로 실행
  const detailResult = await queryRunner.manager
    .createQueryBuilder()
    .insert()
    .into(MovieDetail)
    .values({ detail: dto.detail })
    .execute();

  const movieDetailId = detailResult.identifiers[0].id;

  const movieResult = await queryRunner.manager
    .createQueryBuilder()
    .insert()
    .into(Movie)
    .values({ title: dto.title, detail: { id: movieDetailId } })
    .execute();

  const movieId = movieResult.identifiers[0].id;

  // ManyToMany 관계 연결
  await queryRunner.manager
    .createQueryBuilder()
    .relation(Movie, 'genres')
    .of(movieId)
    .add(dto.genreIds);

  await queryRunner.commitTransaction();
  return movieId;

} catch (e) {
  await queryRunner.rollbackTransaction();
  throw e;
} finally {
  await queryRunner.release();
}
```

---

---

# 한눈에

|용어|설명|
|---|---|
|BEGIN|트랜잭션 시작|
|COMMIT|모두 성공 → 영구 반영|
|ROLLBACK|실패 → 전부 취소|
|Dirty Read|커밋 안 된 데이터 읽기|
|Non-repeatable Read|같은 데이터 두 번 읽었는데 다름|
|Phantom Read|같은 조건 두 번 조회했는데 행 수 다름|
|QueryRunner|NestJS 트랜잭션 핵심 도구|
|release()|커넥션 반드시 해제 ⚠️|