---
aliases:
  - TypeORM 관계
  - OneToMany
  - ManyToOne
  - ManyToMany
  - OneToOne
tags:
  - NestJS
related:
  - "[[00_NestJS_Ecosystem_HomePage]]"
---


# NestJS_TypeORM_Relations — 관계 (Relationships)

## 한 줄 요약

```
테이블 간 관계를 데코레이터로 정의
Many 붙은 쪽 → 배열 [] / One 만 있는 쪽 → 단일 객체
```

---

---

#  관계 종류 ⭐️

| 데코레이터         | 관계  | 타입       | 예시        |
| ------------- | --- | -------- | --------- |
| `@ManyToOne`  | N:1 | 단일 객체    | 사진 → 유저   |
| `@OneToMany`  | 1:N | 배열 []    | 유저 → 사진들  |
| `@OneToOne`   | 1:1 | 단일 객체    | 유저 ↔ 프로필  |
| `@ManyToMany` | N:M | 배열 [] 양쪽 | 질문 ↔ 카테고리 |

```
타입 결정 규칙:
  데코레이터에 "Many" 가 붙어 있으면 → 배열 []
  "One" 만 있으면                    → 단일 객체

  @OneToMany  → movies: Movie[]   ← Many 쪽이 배열
  @ManyToOne  → director: Director  ← One 쪽 바라봄 → 단일
  @ManyToMany → 양쪽 모두 배열 []
```

---

---

# 파라미터 읽는 법 ⭐️

```typescript
@ManyToOne(
  () => Director,              // 첫 번째: 연결할 Entity
  (director) => director.movies // 두 번째: 상대쪽에서 나를 가리키는 프로퍼티
)
director: Director;
```

```
두 번째 파라미터 = "상대방이 나를 가리키는 프로퍼티 이름"

Movie 의 @ManyToOne 에서:
  (director) => director.movies
  → "Director 클래스의 movies 가 나(Movie)를 가리키고 있어"

Director 의 @OneToMany 에서:
  (movie) => movie.director
  → "Movie 클래스의 director 가 나(Director)를 가리키고 있어"

서로 맞짝 지어주는 역할
→ SQL JOIN ON 절과 같은 개념
```

---

---
# ManyToOne / OneToMany ⭐️

```typescript
// Movie.entity.ts
@Entity()
export class Movie {
  @ManyToOne(
    () => Director,
    (director) => director.movies,
    { cascade: true, nullable: false }
  )
  director: Director;   // 단일 (Director 는 하나)
}

// Director.entity.ts
@Entity()
export class Director {
  @OneToMany(
    () => Movie,
    (movie) => movie.director
  )
  movies: Movie[];   // 배열 (Movie 는 여러 개)
}
```

```
DB 결과:
  movie 테이블 → director_id FK 자동 생성   ← @ManyToOne 쪽
  director 테이블 → 별도 컬럼 없음

FK 는 항상 @ManyToOne 쪽(Many 쪽)에 생성됨
@OneToMany 는 단독 사용 불가 → 반드시 @ManyToOne 과 쌍으로
```

---
---
# OneToOne ⭐️

```typescript
// Profile.entity.ts
@Entity()
export class Profile {
  @OneToOne(() => User, (user) => user.profile)
  @JoinColumn()   // ← FK 를 이 테이블에 생성 (반드시 한쪽만)
  user: User;     // profile 테이블에 user_id FK 생성
}

// User.entity.ts
@Entity()
export class User {
  @OneToOne(() => Profile, (profile) => profile.user)
  profile: Profile;   // FK 없음
}
```

```
@JoinColumn:
  OneToOne 에서 FK 를 어느 쪽에 둘지 명시
  붙인 쪽에 FK 컬럼 생성
  반드시 한쪽에만 (양쪽 적용 불가)

ManyToOne 은 자동으로 Many 쪽에 FK 생기므로 @JoinColumn 불필요
```
️

---

---

#  ManyToMany ⭐️

```typescript
// Question.entity.ts
@Entity()
export class Question {
  @ManyToMany(() => Category, (category) => category.questions)
  @JoinTable()          // ← 중간 테이블 생성 (한쪽만)
  categories: Category[];   // 배열
}

// Category.entity.ts
@Entity()
export class Category {
  @ManyToMany(() => Question, (question) => question.categories)
  questions: Question[];    // 배열
}
```

```
DB 결과:
  question 테이블 / category 테이블 각각 생성
  중간 테이블 자동 생성: question_categories_category
    question_id  FK → question.id
    category_id  FK → category.id

@JoinTable:
  중간 테이블 생성 기준 / 반드시 한쪽에만
  @JoinTable 붙인 테이블명이 중간 테이블 앞에 위치
```

---

---

# cascade vs onDelete ⭐️️

```
cascade: true   TypeORM 레벨 / 저장·수정·삭제 전파
onDelete:       DB 레벨 / 부모 삭제 시 자식을 어떻게 할지 DB 에게 지시
```

## cascade: true

typescript

```typescript
@ManyToOne(() => Director, (d) => d.movies, { cascade: true })
director: Director;
```

```
TypeORM 이 처리
부모 save() 시 자식도 자동 save
JS/TypeScript 코드 레벨에서 동작
삭제는 cascade: ['remove'] 또는 cascade: true 일 때 포함
```

## onDelete ⭐️

typescript

```typescript
@ManyToOne(() => Director, (d) => d.movies, {
  onDelete: 'CASCADE',   // ← DB 에게 "부모 삭제 시 나도 삭제해줘"
})
director: Director;
```

```
DB 레벨에서 처리 (TypeORM 코드 없이 DB 가 직접)
부모 Entity 삭제 시 자식을 어떻게 할지 결정

onDelete 옵션:
  'CASCADE'     부모 삭제 → 자식도 자동 삭제
  'SET NULL'    부모 삭제 → 자식의 FK 컬럼을 NULL 로
  'RESTRICT'    자식이 있으면 부모 삭제 불가 (기본값)
  'NO ACTION'   아무것도 안 함 (RESTRICT 와 유사)

예시:
  Director 삭제 시 그 Director 의 Movie 도 삭제 → onDelete: 'CASCADE'
  User 탈퇴 시 User 의 게시글 작성자를 NULL 로  → onDelete: 'SET NULL'
```

## cascade vs onDelete 비교

```
                cascade: true        onDelete: 'CASCADE'
처리 주체        TypeORM (코드)        DB (SQL)
언제 동작        save() / remove()    DB 에서 DELETE 쿼리
방향             부모 → 자식 저장      부모 삭제 → 자식 삭제
주 용도          관계 Entity 자동 저장  부모 삭제 시 데이터 정리
```

|옵션|작동 주체|언제 작동?|영향을 받는 쪽|
|---|---|---|---|
|`onDelete: 'CASCADE'`|데이터베이스 FK|참조 대상 행이 삭제될 때|현재 FK를 가진 행|
|`cascade: true`|TypeORM|`save`, `remove`, `softRemove` 등 ORM 작업 시|연결된 엔티티 객체|

```typescript
// 실전 — 둘 다 쓰는 패턴
@ManyToOne(() => Director, (d) => d.movies, {
  cascade: true,        // Director 저장 시 Movie 도 자동 저장
  onDelete: 'CASCADE',  // Director 삭제 시 Movie 도 자동 삭제
  nullable: false,
})
director: Director;
```

```
cascade: true 설정 시
부모 save() 할 때 자식도 자동 저장됨
```


```typescript
// Movie Entity
@OneToOne(() => MovieDetail, (d) => d.id, { cascade: true })
@JoinColumn()
detail: MovieDetail;
```


```typescript
// Service — movie 저장 시 detail 도 자동 저장
await movieRepository.save({
  title: '마이클',
  detail: { description: '상세 내용' },   // MovieDetail 도 같이 INSERT
});
```

```
cascade 없으면:
  MovieDetail 먼저 따로 save → Movie save
  두 번의 저장 필요

cascade: true 있으면:
  movie save 한 번으로 detail 도 자동 저장

범위 지정:
  cascade: true                    모든 동작 전파 (삭제 포함)
  cascade: ['insert', 'update']    저장·수정만
```

---
---

# nullable: false ⭐️

```typescript
@ManyToOne(() => Director, (d) => d.id, {
  cascade: true,
  nullable: false,   // ← director 없이 저장 불가
})
director: Director;
```

```
nullable: false 효과:
  DB 레벨: FK 컬럼에 NOT NULL 적용
  실수로 연관 데이터 삭제 방지

nullable: true (기본):
  director 없어도 movie 저장 가능 (director_id = NULL)

nullable: false + cascade: true 세트 패턴:
  movie 저장 시 director 반드시 포함 + 자동 저장
```

---
---
# 관계 데이터 조회 — relations 옵션 ⭐️

```
관계 데이터는 기본적으로 자동으로 가져오지 않음
relations 옵션을 명시해야 JOIN 처럼 함께 가져옴
```

```typescript
// 단일 관계 조회
const movie = await movieRepository.findOne({
  where: { id: 1 },
  relations: ['director', 'detail'],
});
// movie.director → Director 객체
// movie.detail   → MovieDetail 객체

// 중첩 관계 조회
const movie = await movieRepository.findOne({
  where: { id: 1 },
  relations: ['director', 'genres', 'likedUsers'],
});
// movie.genres      → Genre[] 배열
// movie.likedUsers  → User[] 배열

// QueryBuilder 에서 관계 조회
const movie = await movieRepository
  .createQueryBuilder('movie')
  .leftJoinAndSelect('movie.director', 'director')
  .leftJoinAndSelect('movie.genres', 'genres')
  .where('movie.id = :id', { id: 1 })
  .getOne();
```

```
relations 없으면:
  movie.director → undefined
  movie.genres   → undefined

relations 있으면:
  JOIN 쿼리 자동 실행 → 관계 데이터 포함

성능 주의:
  relations 많을수록 JOIN 쿼리 증가
  필요한 것만 선택적으로 지정
```

---
---
# @JoinColumn / @JoinTable 정리

```
@JoinColumn  OneToOne 에서 FK 위치 지정 (한쪽만)
@JoinTable   ManyToMany 에서 중간 테이블 생성 기준 (한쪽만)

어느 쪽에 붙이냐:
  중요도가 높은 쪽 / 주체가 되는 쪽에 붙이는 것이 관례
  ManyToOne 은 @JoinColumn 불필요 (자동으로 Many 쪽에 FK)
```

