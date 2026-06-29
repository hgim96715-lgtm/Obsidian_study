---
aliases:
  - 구조분해
  - instanceof
  - operators
  - rest
  - spread
tags:
  - JavaScript
related:
  - "[[00_JS_Ecosystem_HomePage]]"
  - "[[JS_Loops_Conditionals]]"
  - "[[JS_OptionalChaining]]"
  - "[[JS_Truthy_Falsy]]"
  - "[[NestJS_Controller]]"
  - "[[NextJS_API_Client]]"
  - "[[React_Context]]"
  - "[[JS_Promise]]"
---
# JS_Operators — 비교 · 논리 · 스프레드/Rest · 구조분해 · instanceof

> [!info] 
>  삼항연산자는 [[JS_Loops_Conditionals]], 옵셔널 체이닝(`?.`)/널리시 코얼레싱(`??`)은 [[JS_OptionalChaining]]에 이미 정리돼 있다. 이 노트는 그 외에 실제 코드에서 계속 등장했던 비교(`===`), 논리(`&&`/`||`), 스프레드/Rest(`...`)와 그걸 가장 많이 쓰는 구조분해 할당, `instanceof`를 다룬다.

```txt
이미 다른 노트에 있는 연산자 — 여기서 중복 안 함:
  ? :        → JS_Loops_Conditionals
  ?. / ??    → JS_OptionalChaining
  typeof     → JS_BrowserAPI의 "환경/기능 감지 패턴"
```

---

# 비교 연산자 — == vs === ⭐️⭐️⭐️⭐️

```typescript
'1' == 1    // true  — 타입을 강제로 변환해서 비교(type coercion)
'1' === 1   // false — 타입까지 정확히 같아야 함

null == undefined   // true  (이 둘만 예외적으로 ==에서 같다고 취급됨)
null === undefined  // false
```

```txt
== 는 비교 전에 양쪽 타입을 서로 맞춰보려고 시도함(문자열을 숫자로 변환하는 등) —
그 변환 규칙이 직관적이지 않은 경우가 많아서 예상 못 한 결과가 나올 수 있음

→ 거의 항상 ===(엄격한 비교)를 쓰는 게 안전함 — 타입 변환 없이 "값과 타입이 둘 다 같은가"만 봄
```

```
⚠️ 객체/배열은 ===로도 "내용"이 아니라 "참조(메모리 주소)"를 비교함

{ id: 1 } === { id: 1 }   // false — 서로 다른 객체이므로
const a = { id: 1 };
a === a                    // true — 같은 객체를 가리키므로

→ 내용이 같은지 비교하려면 필드를 직접 비교하거나(a.id === b.id),
  JSON.stringify로 문자열화해서 비교하는 식의 별도 방법이 필요함
```

---

# 논리 연산자 — && / || 단축 평가(short-circuit) ⭐️⭐️⭐️⭐️

```typescript
a && b   // a가 falsy면 그 즉시 a를 반환(b는 평가도 안 함) — a가 truthy면 b를 평가해서 반환
a || b   // a가 truthy면 그 즉시 a를 반환 — a가 falsy면 b를 평가해서 반환
```

```txt
"단축 평가"라는 이름의 의미: 왼쪽만 보고 답이 정해지면, 오른쪽은 코드 자체를 실행조차 안 함
→ if (user) { doSomething(user); } 를 user && doSomething(user); 한 줄로 줄이는 식으로 자주 씀

falsy/truthy가 정확히 뭘 가리키는지(빈 배열/빈 객체도 truthy라는 것 등)는 [[JS_Truthy_Falsy]] 참고
```

## React에서 흔한 && 조건부 렌더링과 그 함정 ⭐️⭐️⭐️⭐️

```tsx
{user && <Profile user={user} />}
// user가 없으면(null/undefined) 그 즉시 멈추고 false 같은 값이 반환돼서 아무것도 안 그려짐
// user가 있으면 <Profile />까지 평가돼서 그려짐
```

```tsx
// ⚠️ 흔한 버그 — 왼쪽 값이 숫자 0일 때
{count && <span>{count}개</span>}
// count가 0이면? && 는 0이 falsy라서 0을 그대로 반환함
// → 화면에 <span>이 안 그려지는 게 아니라, 숫자 0이 그대로 텍스트로 찍혀버림 ("0"이 보임)

// ✅ boolean으로 명확히 만들어서 방지
{count > 0 && <span>{count}개</span>}
```

```txt
이 함정이 ?.와 연결되는 지점:
  a?.b 는 "a가 없으면 멈추고 undefined"라는 점에서 a && a.b 와 비슷한 발상이지만,
  ?.는 항상 undefined를 반환하는 반면 &&는 "왼쪽 값 그 자체"(0, '', false 등)를 그대로 반환함
  → 그래서 조건부 렌더링에서는 숫자 0이나 빈 문자열을 왼쪽에 두면 ?.보다 의도와 다르게 동작하기 쉬움
```

---
# ! (논리 NOT) — 부정 표현을 정확히 읽기 ⭐️⭐️⭐️⭐️

```
!변수명 을 글자 그대로 직역하면 의미가 왜곡되기 쉬움 — 항상 "원래 의미를 먼저 떠올리고, 그걸 반대로" 읽을 것

  isLoading = "로딩 중이다"          → !isLoading = "로딩 중이 아니다"(= 확인이 끝났다)
  user      = "사용자 객체가 있다"     → !user      = "사용자 정보가 없다"

⚠️ !isLoading을 "로딩이 없다"로 읽으면 헷갈림 — "로딩"이라는 사물이 있다/없다가 아니라,
   "지금 로딩 중인 상태인가"라는 boolean을 뒤집는 것일 뿐. 변수 이름이 동사/형용사형(is~)이면
   먼저 그 문장을 완성해서("로딩 중이다") 읽고, 그 다음에 부정(! → "아니다")을 붙이는 순서가 안전함
```

```typescript
if (!isLoading && !user) {
  router.replace('/login?next=/users/me');
}
```

```
해석 순서:
  !isLoading  → "로딩 중이다"의 반대 → "로딩이 끝났다(확인이 끝났다)"
  !user       → "사용자 정보가 있다"의 반대 → "사용자 정보가 없다"
  둘 다 동시에(&&) → "확인은 끝났는데, 로그인된 사용자가 없다" → 그제서야 로그인 페이지로 이동

isLoading이 끝나길 먼저 확인해야 하는 이유:
  isLoading이 true인 동안(아직 응답을 기다리는 중)에는 user가 아직 null일 수 있는 게 당연함 —
  이 시점에 무작정 "user가 없으니 비로그인"이라고 판단하면, 실제로는 로그인된 사용자인데도
  응답이 오기 전이라는 이유만으로 잘못 로그아웃 처리될 위험이 있음
  → !isLoading으로 "판단해도 되는 시점"이 됐는지부터 확인하고, 그 다음에야 user 유무를 봄
  (이 패턴이 실제로 쓰이는 곳은 [[React_Context]]의 "보호된 페이지" 예시 참고)
```

---

# 스프레드 / Rest — 같은 ... , 반대 의미 ⭐️⭐️⭐️⭐️

```txt
스프레드(spread) = 펼치기 — 배열/객체를 풀어서 새 배열/객체를 만들 때
Rest = 모으기 — 여러 개의 남은 인자/요소를 하나의 배열로 모을 때
같은 문법(...)인데 쓰이는 위치에 따라 정반대로 동작하는 게 헷갈리는 지점
```

## 스프레드 — 펼치기

```typescript
const copy = [...items];               // 배열 복사 (얕은 복사)
const sorted = [...items].sort(...);    // 원본은 그대로 두고 복사본만 정렬 — [[JS_Array_Methods]] 참고

const merged = { ...init, headers };
// init 객체의 모든 필드를 펼쳐 넣고, headers만 새 값으로 덮어씀
// (실제 코드: authFetch가 기존 fetch 옵션은 유지하면서 headers만 교체하던 패턴)
```

```txt
⚠️ 얕은 복사(shallow copy)라는 점 주의:
  { ...obj } 는 obj의 "한 단계"까지만 새로 복사함
  obj 안에 또 다른 객체/배열이 들어있다면, 그 안쪽 것은 원본과 같은 참조를 그대로 공유함
  → 중첩된 객체까지 완전히 복사하려면 별도의 깊은 복사(structuredClone 등)가 필요함
```

## Rest — 모으기

```typescript
export const Roles = (...roles: Role[]) => SetMetadata(ROLES_KEY, roles);
// Roles('admin', 'editor') 처럼 몇 개를 넘기든, roles는 ['admin', 'editor'] 배열로 모아짐
```

```typescript
// 구조분해할당에서도 동일한 발상
const [first, ...rest] = [1, 2, 3];
// first = 1, rest = [2, 3]

// 객체 구조분해에서도 — 특정 필드만 빼고 나머지를 모음
const { id, ...updateFields } = body;
// id는 따로 변수로, 나머지 필드는 updateFields 객체로 모임
// (NestJS에서 PK를 제외하고 나머지 필드만 update에 넘길 때 자주 쓰는 패턴)
```

```txt
구분하는 법: 함수를 "정의"할 때 매개변수 자리의 ...는 Rest(모으기)
            배열/객체 리터럴 "안"에 들어가는 ...는 스프레드(펼치기)
```

---

# 구조분해 할당(Destructuring) — 객체/배열에서 값 바로 꺼내기 ⭐️⭐️⭐️⭐️

```txt
엄밀히는 연산자가 아니라 "할당 패턴"이지만, 위 Rest(...)를 가장 많이 같이 쓰는 문법이라 여기 같이 정리함
배열/객체의 값을 변수로 꺼낼 때 obj.xxx를 반복하지 않고 한 줄로 바로 꺼내는 문법
```

## 객체 구조분해 — 이름(키) 기준

```typescript
const movie = { id: 1, title: '마이클', genre: '드라마' };
const { title, genre } = movie;   // movie.title, movie.genre 를 각각 변수로

const { title: movieTitle } = movie;   // 이름 바꾸기 — movieTitle 이라는 새 이름으로
const { year = 2024 } = movie;         // 기본값 — year 속성이 없으면(undefined) 2024 사용
```

```txt
⚠️ 기본값은 undefined일 때만 적용됨 — null이면 적용 안 되고 null이 그대로 들어옴
  const { x = 10 } = { x: undefined };  // x = 10
  const { x = 10 } = { x: null };       // x = null  ← 의외로 기본값이 안 먹음
```

## 배열 구조분해 — 위치(인덱스) 기준

```typescript
const [first, second] = [1, 2, 3];     // 위치대로 매칭
const [, second2] = [1, 2, 3];          // 쉼표로 자리를 비워서 건너뛰기 — second2 = 2
```

```txt
객체 vs 배열 구조분해의 핵심 차이:
  객체  { id, title }    → 속성 "이름"으로 매칭, 순서 무관
  배열  [first, second]  → "위치(인덱스)"로 매칭, 순서 중요

  API 응답처럼 키가 있는 데이터는 객체 구조분해
  useState() 처럼 "값 + 그 값을 바꾸는 함수" 순서로 오는 반환값은 배열 구조분해
  const [value, setValue] = useState(0);
```

## 함수 매개변수에서 바로 구조분해 ⭐️⭐️⭐️

```typescript
// ❌ 객체를 그대로 받으면 매번 movie. 반복
function getMovie(movie) { console.log(movie.title); }

// ✅ 매개변수 자리에서 바로 구조분해 — 함수 시그니처만 봐도 뭘 쓰는지 드러남
function getMovie({ title, genre = '드라마' }) { console.log(title, genre); }
```

```typescript
// ⚠️ 호출 시 인자를 아예 안 넘기면 에러
function getMovie({ title }) { console.log(title); }
getMovie();   // TypeError: Cannot destructure ... of 'undefined'

// ✅ 매개변수 자체에 기본값 {} 를 줘서 안전하게
function getMovie({ title } = {}) { console.log(title); }   // undefined, 에러 없음
getMovie();
```

```typescript
// 실전 — NestJS DTO / React props에서 거의 표준 패턴
async createMovie({ title, genre, detail }: CreateMovieDto) { /* ... */ }

function MovieCard({ title, genre, onDelete }: MovieCardProps) {
  return <div onClick={() => onDelete()}>{title} - {genre}</div>;
}
```

## 중첩 구조분해 — 짧게만 ⭐️

```typescript
const { address: { city } } = user;   // address 자체는 변수로 안 남고, city만 바로 꺼내짐
```

```txt
2~3단계 넘는 깊은 중첩은 가독성이 오히려 떨어짐 — 그럴 땐 단계별로 나눠서
const { address } = user; const { city } = address; 처럼 쪼개는 게 더 읽기 편한 경우가 많음
```

---

# instanceof — 인스턴스 타입 체크 ⭐️⭐️⭐️

```typescript
catch (e) {
  if (e instanceof ApiError && e.status === 401) {
    // ApiError의 인스턴스일 때만, 그리고 그 경우에만 status 같은 ApiError 고유 필드에 접근 가능
  }
}
```

```
instanceof는 "이 값이 특정 클래스(또는 그 클래스를 상속한 클래스)로 만들어졌는가"를 확인함
typeof와 다른 점: typeof는 string/number/boolean 같은 원시 타입에 씀, instanceof는 클래스로
만든 객체(직접 만든 클래스, Error, Array, Date 등)에 씀
```

|연산자|확인하는 대상|예시|
|---|---|---|
|`typeof`|원시 타입|`typeof x === 'string'`|
|`instanceof`|클래스의 인스턴스|`e instanceof ApiError`|

---

# 할당 연산자 축약형 ⭐️

```typescript
count += 1;     // count = count + 1
count ||= 10;    // count가 falsy면 10을 대입 (count = count || 10)
count ??= 10;    // count가 null/undefined일 때만 10을 대입 (count = count ?? 10)
```

```
??=는 "값이 아직 없을 때만 기본값을 채워넣기"에 자주 씀 —
?? 자체의 동작(0/''는 안 건드림)은 [[JS_OptionalChaining]] 참고
```

---

# 한눈에

| 연산자                                     | 핵심                                             |
| --------------------------------------- | ---------------------------------------------- |
| `===`                                   | 타입 변환 없이 값+타입까지 정확히 비교 — 거의 항상 `==` 대신 이걸 쓸 것  |
| 객체/배열 `===`                             | 내용이 아니라 참조 비교 — `{} === {}`는 항상 false          |
| `&&`                                    | 왼쪽이 falsy면 즉시 멈추고 왼쪽 반환 — 조건부 렌더링에서 숫자 0 함정 주의 |
| <code>\|</code>                         | 왼쪽이 truthy면 즉시 멈추고 왼쪽 반환                       |
| `[...arr]` / `{...obj}`                 | 배열/객체를 펼쳐서 새로 만들기 — 얕은 복사                      |
| `(...args)`                             | 함수 매개변수에서 나머지를 배열로 모으기(Rest)                   |
| `const { a } = obj` / `const [a] = arr` | 구조분해 — 객체는 이름 기준, 배열은 위치 기준                    |
| `function f({ a } = {})`                | 함수 매개변수 구조분해 + 인자 없을 때 안전장치                    |
| `instanceof`                            | 클래스 인스턴스인지 확인 — 원시 타입엔 `typeof`                |
| `??=` / <code>\|=</code>                | 값이 없을 때만(또는 falsy일 때만) 기본값 대입                  |
