---
aliases:
  - 구조분해
  - 구조분해 할당
  - Destructuring
tags:
  - JavaScript
related:
  - "[[00_JS_Ecosystem_HomePage]]"
  - "[[JS_Spread_Rest]]"
  - "[[JS_Array]]"
  - "[[JS_Function]]"
---
# JS_Destructuring — 구조분해 할당

# 한 줄 요약

```
배열/객체에서 값을 꺼내 변수에 바로 담는 문법

const { title, genre } = movie
→ movie.title, movie.genre 를 각각 변수로 한 줄에
```

---

---

# 왜 쓰는가 ⭐️

```javascript
// ❌ 구조분해 없이 — 같은 객체를 반복해서 접근
const id    = movie.id;
const title = movie.title;
const genre = movie.genre;

// ✅ 구조분해 — 한 줄로 동시에 꺼내기
const { id, title, genre } = movie;
```

```
구조분해가 좋은 이유:
  변수 선언이 짧아짐 (movie. 반복 제거)
  필요한 값만 명시적으로 꺼냄 → 코드 읽기 쉬움
  함수 매개변수 / API 응답 처리에서 특히 자주 사용
```

---

---

# 객체 구조분해 ⭐️

## 기본

```javascript
const movie = { id: 1, title: '마이클', genre: '드라마' };

const { id, title, genre } = movie;
// id = 1, title = '마이클', genre = '드라마'

// 속성 일부만 꺼내도 됨
const { title } = movie;
```

## 이름 바꾸기 ⭐️

```javascript
const { title: movieTitle } = movie;
// movieTitle = '마이클'  (title 을 movieTitle 이라는 새 이름으로)

// 여러 객체에서 같은 이름 속성을 동시에 다룰 때 충돌 방지
const movieA = { title: '마이클' };
const movieB = { title: '악마' };

const { title: titleA } = movieA;
const { title: titleB } = movieB;
// title 이라는 이름이 겹치지 않게 각각 다른 변수명으로
```

## 기본값 ⭐️

```javascript
const { title, year = 2024 } = movie;
// movie 에 year 속성이 없으면 → 기본값 2024 사용
// movie 에 year 가 있으면 → 그 값 사용

// 이름 바꾸기 + 기본값 동시에
const { year: releaseYear = 2024 } = movie;
```

```
기본값이 적용되는 조건:
  속성이 undefined 일 때만 적용
  null 은 기본값 적용 안 됨 (null 그대로 들어옴)

  const { x = 10 } = { x: undefined };  // x = 10
  const { x = 10 } = { x: null };       // x = null ⚠️
```

## 나머지 속성 — `...rest` ⭐️

```javascript
const { id, ...movieRest } = movie;
// id = 1
// movieRest = { title: '마이클', genre: '드라마' }
//             ↑ id 를 제외한 나머지 속성 전부
```

```javascript
// 실전 패턴 — id 빼고 나머지로 업데이트 (NestJS 자주 사용)
async function updateMovie(id: number, body: UpdateMovieDto) {
  const { id: _, ...updateFields } = body;
  //          ↑ body 에 id 가 섞여 와도 무시하고 나머지만 사용
  return this.prisma.movie.update({ where: { id }, data: updateFields });
}
```

```
...rest 규칙:
  반드시 마지막에 위치해야 함
  const { a, ...rest, b } = obj;  ❌ 에러
  const { a, b, ...rest } = obj;  ✅

  객체 구조분해의 ...rest 는 "나머지 속성을 모음"
  함수의 ...args 는 "나머지 인자를 모음" (다른 맥락)
  → [[JS_Function#매개변수 — 기본값 & 나머지 매개변수]] 참고
```

---

---

# 배열 구조분해 ⭐️

## 기본 — 순서가 중요

```javascript
const [first, second, third] = [1, 2, 3];
// first = 1, second = 2, third = 3

// 객체와 다른 점: 배열은 "위치" 기준, 객체는 "이름" 기준
```

## 건너뛰기 — 쉼표로 자리 비우기

```javascript
const [, second] = [1, 2, 3];
// second = 2  (첫 번째 값은 버림)

const [first, , third] = [1, 2, 3];
// first = 1, third = 3  (두 번째 값은 버림)
```

## 나머지 — `...rest`

```javascript
const [first, ...rest] = [1, 2, 3, 4, 5];
// first = 1
// rest  = [2, 3, 4, 5]
```

## 기본값

```javascript
const [a = 10, b = 20] = [1];
// a = 1  (배열에 값 있음)
// b = 20 (배열에 값 없어서 기본값 사용)
```

## 변수 값 교환 — swap 패턴 ⭐️

```javascript
// 임시 변수 없이 두 변수 값 교환
let x = 1, y = 2;
[x, y] = [y, x];
// x = 2, y = 1
```

```
객체 구조분해 vs 배열 구조분해:
  객체  { id, title }      → 속성 이름으로 매칭 (순서 무관)
  배열  [first, second]    → 위치(인덱스)로 매칭 (순서 중요)

  API 응답은 보통 객체 → 객체 구조분해
  useState() 같은 반환값은 배열 → 배열 구조분해
  const [value, setValue] = useState(0);
```

---

---

# 함수 매개변수 구조분해 ⭐️

```javascript
// ❌ 객체를 그대로 받으면 매번 movie. 반복
function getMovie(movie) {
  console.log(movie.title, movie.genre);
}

// ✅ 매개변수에서 바로 구조분해
function getMovie({ title, genre }) {
  console.log(title, genre);
}

getMovie({ title: '마이클', genre: '드라마' });
```

## 기본값 포함

```javascript
function getMovie({ title, genre = '드라마' }) {
  console.log(title, genre);
}

getMovie({ title: '악마' });   // genre 없으면 '드라마'
```

## 객체 자체가 없을 때 대비 — 기본값 ⭐️

```javascript
// ⚠️ 호출 시 인자를 아예 안 넘기면 에러
function getMovie({ title }) {
  console.log(title);
}
getMovie();   // TypeError: Cannot destructure ... of 'undefined'

// ✅ 매개변수 자체에 기본값 {} 부여
function getMovie({ title } = {}) {
  console.log(title);   // undefined (에러 없이 안전)
}
getMovie();   // 정상 동작
```

## NestJS / React 실전 패턴 ⭐️

```typescript
// NestJS DTO 구조분해
async createMovie({ title, genre, detail }: CreateMovieDto) {
  // DTO 전체 대신 필요한 필드만 바로 사용
}

// React props 구조분해
function MovieCard({ title, genre, onDelete }: MovieCardProps) {
  return (
    <div onClick={() => onDelete()}>
      {title} - {genre}
    </div>
  );
}
```

```
함수 매개변수 구조분해를 쓰는 이유:
  함수 본문에서 obj.xxx 반복 제거
  함수 시그니처만 봐도 "이 함수가 무엇을 쓰는지" 드러남
  DTO / props 받는 함수에서 사실상 표준 패턴
```

---

---

# 중첩 구조분해 ⭐️

```javascript
const user = {
  name: '홍길동',
  address: { city: '서울', zip: '12345' },
};

const { name, address: { city } } = user;
// name = '홍길동', city = '서울'
// ⚠️ address 자체는 변수로 안 만들어짐 — city 만 꺼낸 것
```

```javascript
// 더 깊은 중첩
const response = {
  data: { user: { profile: { nickname: '철수' } } },
};

const {
  data: {
    user: {
      profile: { nickname },
    },
  },
} = response;
// nickname = '철수'
```

```javascript
// 중첩 + 이름 바꾸기 + 기본값 동시에
const { address: { city: userCity = '서울' } = {} } = user;
//                                            ↑ address 자체가 없을 때 대비
```

```
중첩 구조분해 주의:
  너무 깊게 들어가면 가독성 오히려 떨어짐
  2~3단계 넘으면 단계별로 나눠서 구조분해하는 게 나음

  const { data } = response;
  const { user } = data;
  const { nickname } = user.profile;
  → 깊은 한 줄보다 이렇게 나누는 게 읽기 편할 수도 있음
```

---

---

# 자주 만나는 실수 ⭐️

```javascript
// 1. 객체를 배열처럼, 배열을 객체처럼 구조분해
const movie = { title: '마이클' };
const [title] = movie;        // ❌ 에러 (객체는 [] 로 구조분해 불가)
const { title } = movie;      // ✅

// 2. 매개변수 기본값 {} 빠뜨림
function f({ a }) { }
f();                           // ❌ TypeError
function f({ a } = {}) { }
f();                           // ✅

// 3. null 은 기본값 적용 안 됨
const { a = 1 } = { a: null };
console.log(a);                // null ⚠️ (1 이 아님)

// 4. ...rest 위치 (마지막이어야 함)
const { a, ...rest, b } = obj; // ❌ 문법 에러
const { a, b, ...rest } = obj; // ✅
```

---

---

# 한눈에

|패턴|코드|결과|
|---|---|---|
|기본 객체|`const { a } = obj`|`obj.a` 값|
|이름 바꾸기|`const { a: b } = obj`|`b` 라는 이름으로|
|기본값|`const { a = 1 } = obj`|없으면 `1`|
|나머지 (객체)|`const { a, ...rest } = obj`|`a` 제외 나머지|
|기본 배열|`const [a, b] = arr`|위치 순서대로|
|건너뛰기|`const [, b] = arr`|첫 값 버림|
|나머지 (배열)|`const [a, ...rest] = arr`|첫 값 제외 나머지|
|swap|`[x, y] = [y, x]`|값 교환|
|함수 매개변수|`function f({ a }) {}`|호출 시 객체 분해|
|매개변수 안전장치|`function f({ a } = {}) {}`|인자 없어도 안전|
|중첩|`const { a: { b } } = obj`|내부 값 바로 꺼내기|

```
객체 → 이름(키) 기준 / 순서 무관
배열 → 위치(인덱스) 기준 / 순서 중요

함수 매개변수에서 쓸 때는 항상 = {} 기본값 고려
중첩은 2~3단계 넘으면 나눠서 작성 고려
```