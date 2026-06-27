---
aliases:
  - Entity
  - "@Column"
  - "@PrimaryGeneratedColumn"
  - "@PrimaryColumn"
tags:
  - NestJS
related:
  - "[[00_NestJS_Ecosystem_HomePage]]"
---
# NestJS_TypeORM_Entity — Entity & Column

## 한 줄 요약

```txt
Entity = DB 테이블을 TypeScript 클래스로 표현
@Entity → 테이블 / @Column → 컬럼 / @PrimaryGeneratedColumn → 자동 PK
```

---

---

# 기본 구조 ⭐️️

```typescript
import { Entity, Column, PrimaryGeneratedColumn } from 'typeorm';

@Entity()
export class User {
  @PrimaryGeneratedColumn()
  id: number;

  @Column()
  email: string;

  @Column({ nullable: true })
  nickname: string;
}
```

```txt
클래스 → DB 테이블 자동 변환 (synchronize: true 일 때)

User 클래스:
  user 테이블 생성

  id          int          PRIMARY KEY  AUTO_INCREMENT
  firstName   varchar
  lastName    varchar
  isActive    boolean
```

## 핵심 데코레이터

```txt
@Entity()
  이 클래스를 DB 테이블로 관리
  기본 테이블명 = 클래스명 소문자 (User → user)
  @Entity('tb_user') 로 직접 지정 가능

@PrimaryGeneratedColumn()
  자동 증가 기본키 (AUTO_INCREMENT)
  @PrimaryGeneratedColumn('uuid') → UUID 방식

@PrimaryColumn() ⭐️
  직접 값을 지정하는 기본키 (DB 가 자동 생성 안 함)
  복합 기본키 만들 때 주로 사용
  → 저장 시 반드시 직접 값 제공

@Column()
  일반 컬럼
  TypeScript 타입 → DB 타입 자동 매핑
    string → varchar / number → int / boolean → boolean
```

---
---
# DataSource — DB 연결 정보

```typescript
// TypeORM 직접 사용 시 (NestJS 없이)
const AppDataSource = new DataSource({
  type: 'postgres',
  host: 'localhost',
  port: 5432,
  username: 'test',
  password: 'test',
  database: 'test',
  entities: [User, Movie],   // 사용할 Entity 등록
});
```

```txt
NestJS 에서는 TypeOrmModule.forRootAsync 로 설정
→ [[NestJS_TypeORM]] 참고
```

---
---
# @PrimaryColumn 옵션 ⭐️

```typescript
@PrimaryColumn({
  name: 'userId',   // DB 컬럼명 (camelCase → snake_case 변환 없이 직접 지정)
  type: 'int8',     // DB 타입 직접 지정
})
userId: number;
```

## 주요 옵션

```txt
name:
  DB 에 저장될 컬럼 이름
  없으면 프로퍼티명 그대로 사용 (userId → userId)

type:
  DB 컬럼 타입 직접 지정
  TypeScript 타입에서 자동 추론되는 것을 재정의

  int8 이란:
    8바이트 정수 = bigint
    일반 int(4바이트) 보다 더 큰 숫자 저장 가능
    PostgreSQL 에서 bigint = int8

  int4  = 4바이트 정수 (일반 int)
  int8  = 8바이트 정수 (bigint)
  text  = 길이 제한 없는 문자열
  float = 소수

nullable:
  NULL 값 허용 여부 (기본값 false)

default:
  컬럼 기본값

unique:
  UNIQUE 제약 (기본값 false)
```

## @PrimaryColumn 실전 — 복합 기본키 

```typescript
@Entity()
export class MovieUserLike {
  @PrimaryColumn({ name: 'movieId', type: 'int8' })
  @ManyToOne(() => Movie)
  movie: Movie;

  @PrimaryColumn({ name: 'userId', type: 'int8' })
  @ManyToOne(() => User)
  user: User;
}
```

```txt
생성되는 테이블:
  movie_user_like (movieId, userId)
  PRIMARY KEY (movieId, userId)  ← 복합 PK

  같은 유저가 같은 영화에 중복 좋아요 불가
  (movieId, userId) 조합이 이미 있으면 INSERT 실패
```

---

---
# @Column 주요 옵션 ⭐️

```typescript
@Column({
  type: 'varchar',
  length: 100,
  nullable: true,
  default: 'drama',
  unique: true,
  select: false,    // 조회 시 제외
  update: false,    // 생성 후 수정 불가
})
title: string;
```

|옵션|기본값|설명|
|---|---|---|
|`type`|자동|DB 컬럼 타입|
|`name`|프로퍼티명|DB 컬럼 이름|
|`nullable`|false|NULL 허용|
|`default`|-|기본값|
|`unique`|false|중복 불가|
|`select`|true|false = 조회 시 자동 제외|
|`update`|true|false = 저장 후 수정 불가|
|`enum`|-|허용 값 목록|
|`array`|false|배열 타입 (PostgreSQL)|

---
---

# @Unique — 복합 유니크 제약 ⭐️

```typescript
@Entity()
@Unique('UQ_DIRECTOR_NAME_DOB', ['name', 'dob'])
export class Director {
  @Column()
  name: string;

  @Column()
  dob: Date;   // date of birth
}
```

```txt
@Column({ unique: true })    단일 컬럼 유니크 → 동명이인 불가
@Unique(['name', 'dob'])     두 컬럼 조합 유니크 → 이름+생일이 같아야 중복
                             동명이인 허용 (이름 같아도 생일 다르면 OK) 관례: UQ_{테이블명}_{컬럼명들}
```

---
---
# 특수 컬럼 ⭐️

```typescript
@Entity()
export class Movie {
  @PrimaryGeneratedColumn()
  id: number;

  @CreateDateColumn()
  createdAt: Date;    // INSERT 시 자동 저장

  @UpdateDateColumn()
  updatedAt: Date;    // UPDATE 시 자동 저장

  @DeleteDateColumn()
  deletedAt: Date;    // soft delete 시 저장 (null = 삭제 안 됨)

  @VersionColumn()
  version: number;    // UPDATE 마다 1씩 증가, 몇 번 수정됐는지 추적
}
```

---
---
# BaseTable — 공통 컬럼 상속 ⭐️

```typescript
// base-table.entity.ts
export abstract class BaseTable {
  @PrimaryGeneratedColumn()
  id: number;

  @CreateDateColumn()
  createdAt: Date;

  @UpdateDateColumn()
  updatedAt: Date;

  @VersionColumn()
  version: number;
}

// 사용
@Entity()
export class Movie extends BaseTable {
  // id / createdAt / updatedAt / version 자동 포함
  @Column()
  title: string;
}
```

```txt
BaseTable 쓰면 좋은 경우:
  모든 Entity 가 같은 PK 방식 + 날짜 컬럼 필요할 때

쓰면 안 되는 경우:
  Entity 마다 PK 방식이 다를 때 (uuid / 복합키 등)
  → id 를 각 Entity 에서 따로 선언
```

---
---
# forRoot vs forFeature ⭐️

```txt
forRoot    → app.module.ts 에서 1번 (DB 연결)
forFeature → 각 모듈에서 (Entity 를 Repository 로 쓸 때)
```

```typescript
// app.module.ts
TypeOrmModule.forRootAsync({ ... autoLoadEntities: true })

// movie.module.ts
TypeOrmModule.forFeature([Movie, MovieDetail])
// ↑ Movie, MovieDetail 을 이 모듈에서 Repository 로 사용

// movie.service.ts
@InjectRepository(Movie)
private movieRepository: Repository<Movie>
```

```txt
forFeature 빠뜨리면:
  "No repository for Movie was found" 에러
  관계 Entity 도 같이 등록 필요 (MovieDetail 등)
```

---
---
# Embedding vs Inheritance

```txt
Embedding  @Column(() => Base)  → 응답에 중첩 객체로 나옴 { base: {...} }
Inheritance extends Base         → 응답에 같은 레벨로 나옴 { createdAt: ... }
→ 보통 Inheritance 방식이 응답 구조가 더 깔끔
```

## Embedding

```typescript
export class Name {
  @Column() first: string;
  @Column() last: string;
}

@Entity()
export class User {
  @Column(() => Name)
  name: Name;
}
// DB: nameFirst / nameLast 컬럼 생성
```

## Inheritance

```typescript
export abstract class Content {
  @PrimaryGeneratedColumn() id: number;
  @Column() title: string;
}

@Entity()
export class Photo extends Content {
  @Column() size: string;
}
// DB: photo 테이블 (id, title, size)
```

## Single Table Inheritance

```typescript
@Entity()
@TableInheritance({ column: { type: 'varchar', name: 'type' } })
export class Content { ... }

@ChildEntity()
export class Photo extends Content { ... }
```

```txt
모든 자식을 하나의 테이블에 저장
type 컬럼으로 구분 → NULL 컬럼 많아짐 → 실무에서 잘 안 씀
```






















