---
aliases:
  - 연산자
  - 옵셔널 체이닝
  - nullish
  - 병합
  - operators
  - truthy
  - falsy
  - "`!!` — Boolean 변환"
tags:
  - JavaScript
related:
  - "[[00_JS_Ecosystem_HomePage]]"
  - "[[JS_Variables]]"
  - "[[JS_Function]]"
  - "[[JS_Object]]"
---
# JS_Operators — 연산자

# 한 줄 요약

```
값을 계산하고 비교하고 조합하는 기호들
산술 → 비교 → 논리 → 대입 → 모던 JS 전용(옵셔널체이닝/nullish) 순으로 정리
```

---

---

# 산술 연산자 ⭐️

```javascript
5 + 3   // 8   더하기
5 - 3   // 2   빼기
5 * 3   // 15  곱하기
5 / 3   // 1.666...  나누기
5 % 3   // 2   나머지 (modulo)
5 ** 3  // 125 거듭제곱 (5의 3승)
```

```
% (나머지) 자주 쓰는 곳:
  짝수/홀수 판별   n % 2 === 0  → 짝수
  순환 인덱스      i % arr.length  → 배열 끝에서 다시 처음으로
  페이지네이션     (page - 1) % itemsPerPage

** (거듭제곱):
  Math.pow(5, 3) 의 축약형 (ES2016+)
  2 ** 10  // 1024
```

## 문자열에서 + 의 동작 ⭐️

```javascript
'2' + 3      // '23'  (문자열 우선 → 숫자가 문자열로 변환됨)
3 + '2'      // '32'
1 + 1 + '1'  // '21'  (왼쪽부터: 1+1=2, 2+'1'='21')
'1' + 1 + 1  // '111' (왼쪽부터: '1'+1='11', '11'+1='111')

// 숫자만 더하고 싶으면 - 나 * 사용 (문자열 변환 안 일어남)
'5' - '2'    // 3   (둘 다 숫자로 변환되어 계산됨)
'5' * '2'    // 10
```

```
+ 만 특별한 이유:
  문자열 이어붙이기(concatenation) 와 숫자 더하기를 동시에 담당
  피연산자 중 하나라도 문자열이면 전체가 문자열로 변환됨
  -, *, / 는 이어붙이기 의미가 없어서 항상 숫자로 변환 시도
```

## 증감 연산자 — ++ / --

```javascript
let count = 0;

count++;   // 후위: 현재 값 사용 후 증가
count--;   // 후위: 현재 값 사용 후 감소
++count;   // 전위: 먼저 증가 후 값 사용
--count;   // 전위: 먼저 감소 후 값 사용

// 차이가 드러나는 경우
let a = 5;
console.log(a++);   // 5  (먼저 출력하고 그 다음 증가)
console.log(a);     // 6

let b = 5;
console.log(++b);   // 6  (먼저 증가하고 그 다음 출력)
```

---

---

# 대입(할당) 연산자

```javascript
let x = 10;

x += 5;   // x = x + 5   → 15
x -= 3;   // x = x - 3   → 12
x *= 2;   // x = x * 2   → 24
x /= 4;   // x = x / 4   → 6
x %= 4;   // x = x % 4   → 2
x **= 3;  // x = x ** 3  → 8
```

```
실전 — 누적 합계 / 카운터:
  let total = 0;
  for (const price of prices) {
    total += price;   // total = total + price
  }
```

## 논리 대입 연산자 ⭐️

```javascript
// ||=  값이 falsy 일 때만 대입
let name = '';
name ||= '기본값';   // name = name || '기본값' 과 같음 → '기본값'

// ??=  값이 null/undefined 일 때만 대입
let count = 0;
count ??= 10;        // count 는 0(falsy 지만 null/undefined 아님) → 그대로 0

// &&=  값이 truthy 일 때만 대입
let user = { name: '철수' };
user.name &&= user.name.toUpperCase();  // '철수' truthy → 'CHUL SU' 로 변경됨(예시)
```

```bash
||= 와 ??= 차이는 || 와 ?? 차이와 동일:
  ||=  falsy 면 전부 교체 (0, '' 도 교체됨)
  ??=  null/undefined 일 때만 교체 (0, '' 는 유지)
#  → [[JS_Operators#Nullish 병합 ??]] 참고
```

---

---

# 비교 연산자 & 동등성 연산자  ⭐️

```javascript
// == (느슨한 비교) — 타입 변환 후 비교
1 == '1'            // true  (타입 다르지만 같음)
null == undefined   // true

// === (엄격한 비교) — 타입 + 값 모두 같아야
1 === '1'   // false (타입 다름)
1 === 1     // true

// 실무: 반드시 === 사용 (예측 불가능한 버그 방지)
```

```javascript
// 대소 비교
5 > 3    // true
5 < 3    // false
5 >= 5   // true
5 <= 4   // false

// 문자열도 비교 가능 (사전순/유니코드 순서)
'a' < 'b'   // true
'apple' < 'banana'   // true
```

---

---

# 논리 연산자

```javascript
// && (AND) — 둘 다 true 여야 true
true && true    // true
true && false   // false

// || (OR) — 하나라도 true 면 true
true || false   // true
false || false  // false

// ! (NOT) — 반대로 뒤집기
!true    // false
!false   // true
```

## 단축 평가 ⭐️

```javascript
// && — 앞이 false(falsy) 면 뒤는 실행 자체를 안 함
user && user.getName()   // user 가 없으면 getName() 호출 안 됨 (에러 방지)

// || — 앞이 truthy 면 앞 값을 그대로 반환 (뒤는 평가 안 함)
const name = user.name || '익명';
// user.name 이 있으면 그 값 / 없으면(falsy) '익명'
```

---

---

# 옵셔널 체이닝 ?. ⭐️

```javascript
const user = {
  name: '홍길동',
  address: { city: '서울' },
};

// 일반 접근 — 중간 경로가 없으면 에러
user.address.city      // '서울'
user.profile.avatar    // ❌ TypeError (profile 자체가 없음)

// 옵셔널 체이닝 — 없으면 에러 없이 undefined 반환
user.profile?.avatar       // undefined
user.profile?.getName?.()  // undefined (메서드 호출에도 사용 가능)

// 배열에도 사용
arr?.[0]   // arr 가 없으면 undefined
```

```
?. 의 의미:
  앞의 값이 null / undefined 면 → 거기서 멈추고 undefined 반환
  있으면 → 정상적으로 다음 단계 접근

NestJS / 실전에서:
  const title = movie?.title ?? '제목 없음';
  //            ↑ movie 가 없을 수도 있는 상황에서 안전하게 접근
```

---

---

# Nullish 병합 ?? ⭐️

```javascript
// ?? — null 또는 undefined 일 때만 오른쪽 값 사용
const name = user.name ?? '익명';
// user.name 이 null/undefined 면 '익명'
// user.name 이 '' (빈 문자열) 이면 '' 그대로 반환 (|| 와 다름!)
```

## || vs ?? — 가장 헷갈리는 차이 ⭐️

```javascript
const a = '' || '기본값';    // '기본값'  ('' 는 falsy → 교체됨)
const b = '' ?? '기본값';    // ''        ('' 는 null/undefined 아님 → 유지)

const c = 0 || 100;          // 100  (0 은 falsy → 교체됨)
const d = 0 ?? 100;          // 0    (0 은 null/undefined 아님 → 유지)
```

```
|| 는 falsy 값 전부(0, '', false, null, undefined) 일 때 오른쪽으로 교체
?? 는 null / undefined 일 때만 오른쪽으로 교체

→ 0 이나 빈 문자열도 "의미 있는 유효한 값"으로 취급해야 한다면 ?? 사용
  예: 할인율 0%, 페이지 번호 0 처럼 0 자체가 정상 값인 경우
```

---

---

# 삼항 연산자

```javascript
// 조건 ? 참일 때 : 거짓일 때
const message = age >= 18 ? '성인' : '미성년자';

// 중첩 — 가독성 나빠지므로 가능하면 피하기
const grade = score >= 90 ? 'A'
            : score >= 80 ? 'B'
            : score >= 70 ? 'C' : 'F';
```

---

---

# 스프레드 & 나머지 연산자

```javascript
// 스프레드 ... — 펼치기
const arr1 = [1, 2, 3];
const arr2 = [...arr1, 4, 5];   // [1, 2, 3, 4, 5]

const obj1 = { id: 1, title: '마이클' };
const obj2 = { ...obj1, title: '수정' };   // { id: 1, title: '수정' }

// 나머지 ... — 모으기
function sum(...nums) {
  return nums.reduce((a, b) => a + b, 0);
}
sum(1, 2, 3);   // 6

const [first, ...rest] = [1, 2, 3, 4];
// first = 1, rest = [2, 3, 4]
```

```
spread/rest 상세는 → [[JS_Destructuring]] 참고
```

---

---

# 타입 변환 연산자

```javascript
// 단항 + — 숫자로 변환
+'42'     // 42
+true     // 1
+false    // 0
+''       // 0
+'abc'    // NaN

// 단항 - — 숫자로 변환 + 부호 반전
-'42'     // -42

// 실전 — URL 파라미터는 항상 string, 숫자로 변환 필요
const id = +param;             // string → number
movies.find(m => m.id === +id);

// !!  — boolean 으로 강제 변환
!!'hello'   // true
!!''        // false
!!0         // false
!!1         // true
```

```
+ 단항 vs Number():
  +'42'        짧고 빠름
  Number('42') 명시적이라 가독성 좋음
  실무에서는 둘 다 흔하게 쓰임
```

---
---
# `!!` — Boolean 변환 ⭐️

```
!  한 번  → truthy/falsy 를 반대로 변환
!! 두 번  → 원래 의미를 유지하면서 boolean 타입으로 변환

!!value === Boolean(value)
```

```javascript
!!'hello'      // true
!!''           // false
!!0            // false
!!1            // true
!!null         // false
!!undefined    // false
!!{}           // true
!![]           // true


// session.user 있음  → true / session.user 없음  → false
<UploadButton isLoggedIn={!!session?.user} /> 
```

---
---

# Truthy & Falsy ⭐️

## Falsy 값 — 딱 7가지만 기억하면 됨 ⭐️

```javascript
false
0          // 숫자 0
-0         // 음수 0
0n         // BigInt 0
""         // 빈 문자열
null
undefined
NaN

// if 문에서 falsy → 블록 실행 안 됨
if (0)         {}   // 실행 안 됨
if ("")        {}   // 실행 안 됨
if (null)      {}   // 실행 안 됨
if (undefined) {}   // 실행 안 됨
```

## Truthy 값 — falsy 7가지를 제외한 나머지 전부

```javascript
true
1             // 0 이 아닌 숫자
"0"           // 내용 있는 문자열 (빈 문자열이 아님)
"false"       // 내용 있는 문자열
[]            // 빈 배열 ← truthy ⭐️
{}            // 빈 객체 ← truthy ⭐️
function(){}  // 함수

if ([])  { console.log('truthy') }   // 실행됨
if ({})  { console.log('truthy') }   // 실행됨
if ("0") { console.log('truthy') }   // 실행됨
```

```
가장 많이 헷갈리는 것:
  []  → truthy (빈 배열도 truthy)
  {}  → truthy (빈 객체도 truthy) ⭐️
  "0" → truthy (글자가 있는 문자열일 뿐)
  "false" → truthy (글자가 있는 문자열일 뿐, 진짜 false 아님)

NestJS @Public() 패턴:
  @Public() 데코레이터 → isPublic = {} (빈 객체)
  if (isPublic) → {} 는 truthy → return true → Guard 통과

  데코레이터 없음 → isPublic = undefined
  if (isPublic) → undefined 는 falsy → 통과 안 됨
```

## 실전 패턴

```javascript
// 값 존재 여부 확인
const user = null;
if (user) {
  console.log(user.name);   // null 은 falsy → 실행 안 됨
}

// 배열이 비어있는지 확인 → length 사용 ([] 자체는 truthy 라서)
if (arr.length)        { /* 요소 있으면 */ }
if (arr.length === 0)  { /* 비어있으면 */ }

// || 기본값 패턴
const name = user.name || '익명';
const port = process.env.PORT || 3000;

// ?? nullish 병합 (null/undefined 만 체크)
const val1 = "" ?? '기본값';     // ''     반환 (falsy 지만 ?? 는 통과시킴)
const val2 = null ?? '기본값';   // '기본값' 반환
```

---

---

# 한눈에

|값|truthy / falsy|
|---|---|
|`false`|falsy|
|`0`, `-0`, `0n`|falsy|
|`""` (빈 문자열)|falsy|
|`null`|falsy|
|`undefined`|falsy|
|`NaN`|falsy|
|`[]` 빈 배열|**truthy** ⭐️|
|`{}` 빈 객체|**truthy** ⭐️|
|`"0"`, `"false"`|**truthy** ⭐️|
|그 외 나머지|truthy|

```
산술:  + - * / % **        ++  --
대입:  += -= *= /= %= **=  ||= ??= &&=
비교:  ==(느슨) === (엄격, 항상 이거 사용) > < >= <=
논리:  && || !
모던:  ?.(옵셔널체이닝)  ??(nullish병합)  ...(spread/rest)

|| 는 falsy 전부 교체 / ?? 는 null·undefined 만 교체
```