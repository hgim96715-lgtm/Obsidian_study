---
aliases:
  - includes
  - indexOf
  - Math
  - Number
  - padStart
  - slice
  - String
  - toFixed
tags:
  - JavaScript
related:
  - "[[00_JS_Ecosystem_HomePage]]"
  - "[[JS_Array_Methods]]"
  - "[[JS_Operators]]"
  - "[[TS_Generics]]"
---
# JS_Primitive_Methods — String · Number · Math

> [!info] 
> 원시 타입(string, number)의 메서드들. 
> string 메서드는 항상 새 문자열을 반환 — 원본을 바꾸지 않는다. 
> Number.isNaN vs isNaN 혼동 주의. 
> 문자열 조합 패턴(템플릿 리터럴·배열 join)도 이 파일에 정리.

---

# 왜 string에 메서드가 있는가 ⭐️⭐️⭐️

```txt
string은 원시 타입(primitive) — 객체가 아님
하지만 'hello'.toUpperCase()처럼 메서드를 호출할 수 있음

JavaScript가 하는 일 (Boxing):
  'hello'.toUpperCase()
  → 임시로 new String('hello') 객체를 만들어서
  → 메서드 실행
  → 결과(새 문자열) 반환
  → 임시 객체 폐기

개발자는 이 과정을 신경 쓸 필요 없음
그냥 문자열에 메서드를 쓰면 됨
```

```txt
⚠️ string 메서드는 원본을 바꾸지 않음 (불변):
  const str = 'hello';
  str.toUpperCase();    // 'HELLO' 반환
  console.log(str);     // 'hello' — 원본 그대로

  결과를 쓰려면 반드시 변수에 받아야 함:
  const upper = str.toUpperCase();
```

---

# String 메서드 ⭐️⭐️⭐️⭐️

## 포함 여부 확인

```typescript
const str = 'Hello, NestJS World!';

str.includes('NestJS')        // true  — 포함 여부
str.startsWith('Hello')       // true  — 시작 확인
str.endsWith('World!')        // true  — 끝 확인
str.indexOf('NestJS')         // 7     — 첫 번째 위치 (없으면 -1)
str.lastIndexOf('l')          // 15    — 마지막 위치

// 조건 분기에서 자주 쓰는 패턴
if (str.includes('error')) { ... }
if (url.startsWith('https')) { ... }
if (filename.endsWith('.ts')) { ... }
```

## 추출

```typescript
const str = 'Hello, World!';
//           0123456789...

str.slice(7, 12)          // 'World'  — 7부터 12 앞까지
str.slice(7)              // 'World!' — 7부터 끝까지
str.slice(-6)             // 'World!' — 뒤에서 6번째부터
str.slice(0, -1)          // 'Hello, World' — 마지막 제외
str[0]                    // 'H'      — 인덱스 접근
str.at(-1)                // '!'      — 뒤에서 첫 번째 (음수 인덱스)
str.length                // 13
```

```txt
slice vs substring:
  slice(-3)     → 뒤에서 3번째 (음수 지원)
  substring(0, -3) → 음수는 0으로 처리
  → slice를 쓰는 게 대부분 더 직관적
```

## 변환

```typescript
'Hello'.toUpperCase()        // 'HELLO'
'Hello'.toLowerCase()        // 'hello'

'  hello  '.trim()           // 'hello'
'  hello  '.trimStart()      // 'hello  '
'  hello  '.trimEnd()        // '  hello'

'5'.padStart(3, '0')         // '005'  — 앞을 채움
'5'.padEnd(3, '0')           // '500'  — 뒤를 채움
'abc'.padStart(5)            // '  abc' — 기본 공백

'hello'.repeat(3)            // 'hellohellohello'
```

```txt
padStart 사용 예:
  날짜·시간 포맷: String(month).padStart(2, '0')  → '01', '12'
  ID 앞에 0 붙이기: id.toString().padStart(6, '0')  → '000001'
```

## 검색 · 치환

```typescript
'hello world'.replace('world', 'NestJS')      // 'hello NestJS' — 첫 번째만
'aababab'.replaceAll('a', 'x')                // 'xxbxbxb' — 전체 치환

// 정규식으로 치환
'hello  world'.replace(/\s+/g, ' ')           // 'hello world' — 연속 공백 → 하나로
'camelCase'.replace(/([A-Z])/g, '_$1').toLowerCase()  // 'camel_case'
```

## 분리 · 합치기

```typescript
'a,b,c'.split(',')            // ['a', 'b', 'c']
'hello'.split('')             // ['h', 'e', 'l', 'l', 'o']
'hello world'.split(' ')      // ['hello', 'world']
'line1\nline2\nline3'.split('\n')  // ['line1', 'line2', 'line3']

// 배열 → 문자열 (join은 Array 메서드)
['a', 'b', 'c'].join(', ')   // 'a, b, c'
```

---

# 문자열 조합 패턴 ⭐️⭐️⭐️⭐️

## 템플릿 리터럴 — 변수 삽입

```typescript
const name   = '홍길동';
const action = '닫기';

// 변수 삽입
`안녕하세요, ${name}님!`

// 표현식
`총 ${items.length}개`

// 삼항 연산자
`상태: ${isActive ? '활성' : '비활성'}`

// 메서드 호출
`이름: ${name.toUpperCase()}`
```

## 배열 join — 여러 줄 텍스트 ⭐️⭐️⭐️⭐️

```typescript
// 배열 항목 하나 = 줄 하나
const body = [
  `운영이 「${opts.roomName}」 방을 ${action}했어요.`,
  `사유: ${opts.reason}`,
  '',                                        // 빈 줄
  '이 대화에서 운영에게 문의할 수 있어요.',
].join('\n');
```

```txt
배열 join을 쓰는 이유:
  줄이 많을 때 각 줄을 배열 항목으로 분리 → 읽기 쉬움
  빈 문자열('')로 빈 줄 삽입 가능
  조건부 줄 추가가 자연스러움

결과:
  운영이 「채팅방」 방을 닫기했어요.
  사유: 규칙 위반
  (빈 줄)
  이 대화에서 운영에게 문의할 수 있어요.
```

## 조건부 줄 포함 ⭐️⭐️⭐️

```typescript
// filter(Boolean)으로 조건부 줄 처리
const lines = [
  `안녕하세요, ${name}님!`,
  opts.reason ? `사유: ${opts.reason}` : '',  // 없으면 빈 문자열
  opts.note   ? `비고: ${opts.note}`   : '',
  '',
  '감사합니다.',
].filter(Boolean);  // 빈 문자열('') 제거

const message = lines.join('\n');
```

```typescript
// 조건이 복잡하면 push로
const lines: string[] = [];
lines.push(`안녕하세요, ${name}님!`);
if (opts.reason) lines.push(`사유: ${opts.reason}`);
if (opts.note)   lines.push(`비고: ${opts.note}`);
lines.push('', '감사합니다.');
const message = lines.join('\n');
```

```txt
filter(Boolean) vs push + if:
  filter(Boolean)   → 짧고 선언적, 줄이 적을 때
  push + if         → 조건이 복잡하거나 줄이 많을 때

멀티라인 템플릿 리터럴 vs 배열 join:
  조건부 줄 없음 → 백틱 멀티라인이 더 단순
  조건부 줄 있음 → 배열 join + filter(Boolean)이 더 깔끔
```

## 구분자 join ⭐️⭐️⭐️

```typescript
const tags = ['NestJS', 'TypeScript', 'Prisma'];

tags.join(', ')      // "NestJS, TypeScript, Prisma"
tags.join(' · ')     // "NestJS · TypeScript · Prisma"
tags.join(' | ')     // "NestJS | TypeScript | Prisma"

// 마지막만 다른 구분자
const authors = ['홍길동', '김철수', '박영희'];
`${authors.slice(0, -1).join(', ')} 및 ${authors.at(-1)}`
// "홍길동, 김철수 및 박영희"
```

---

# Number 메서드 ⭐️⭐️⭐️

```typescript
const n = 3.14159;

n.toFixed(2)          // '3.14'  — 소수점 자리수 고정 (문자열 반환)
n.toFixed(0)          // '3'
(1000).toFixed(2)     // '1000.00'

n.toString()          // '3.14159'
(255).toString(16)    // 'ff'    — 16진수
(255).toString(2)     // '11111111' — 2진수
```

## 변환 함수

```typescript
Number('42')           // 42
Number('3.14')         // 3.14
Number('')             // 0  ← 주의
Number('abc')          // NaN
Number(true)           // 1
Number(false)          // 0
Number(null)           // 0
Number(undefined)      // NaN

parseInt('42px')       // 42    — 숫자 부분만 파싱
parseInt('3.14')       // 3     — 정수만
parseInt('ff', 16)     // 255   — 16진수 파싱
parseFloat('3.14')     // 3.14
parseFloat('3.14abc')  // 3.14
```

## Number.isNaN vs isNaN ⭐️⭐️⭐️⭐️

```typescript
// ❌ 전역 isNaN — 변환 후 확인 (혼란스러움)
isNaN('hello')     // true  ← 'hello'를 Number로 변환하면 NaN이라서
isNaN(undefined)   // true  ← undefined를 변환하면 NaN이라서
isNaN('')          // false ← ''를 변환하면 0이라서

// ✅ Number.isNaN — 정확히 NaN인지만 확인
Number.isNaN('hello')    // false ← 문자열이지 NaN이 아님
Number.isNaN(undefined)  // false
Number.isNaN(NaN)        // true  ← NaN만 true
Number.isNaN(0/0)        // true
```

```txt
NaN이란:
  Not a Number — 숫자가 아님을 나타내는 특수값
  typeof NaN === 'number' (모순적으로 보이지만 숫자 타입의 특수값)
  NaN === NaN  → false (자기 자신과도 같지 않음)

실전에서 NaN 체크:
  Number.isNaN(value)  → 정확한 NaN 체크
  isNaN(value)         → 피하는 게 좋음 (변환 포함)
```

## 유용한 Number 정적 메서드

```typescript
Number.isInteger(3)         // true
Number.isInteger(3.14)      // false
Number.isFinite(Infinity)   // false
Number.isFinite(3.14)       // true
Number.parseInt('42')       // 42 (전역 parseInt와 동일)
Number.parseFloat('3.14')   // 3.14

Number.MAX_SAFE_INTEGER     // 9007199254740991 (2^53 - 1)
Number.MIN_SAFE_INTEGER     // -9007199254740991
```

---

# Math 객체 ⭐️⭐️⭐️⭐️

```typescript
// 반올림 계열
Math.floor(3.7)   // 3   — 내림 (항상 작은 쪽)
Math.ceil(3.2)    // 4   — 올림 (항상 큰 쪽)
Math.round(3.5)   // 4   — 반올림
Math.trunc(3.7)   // 3   — 소수점 버리기 (-3.7 → -3)

Math.floor(-3.2)  // -4  ← 주의: 내림이라 더 작아짐
Math.trunc(-3.7)  // -3  ← 소수점 제거 (0 방향으로)
```

```typescript
// 최대 · 최소
Math.max(1, 2, 3)             // 3
Math.min(1, 2, 3)             // 1
Math.max(...[1, 5, 3, 9, 2])  // 9 — 배열은 스프레드

// 절댓값 · 거듭제곱 · 제곱근
Math.abs(-5)        // 5
Math.pow(2, 10)     // 1024
2 ** 10             // 1024 (** 연산자로도 가능)
Math.sqrt(16)       // 4

// 랜덤
Math.random()           // 0 이상 1 미만 소수
Math.floor(Math.random() * 10)      // 0~9 정수
Math.floor(Math.random() * (max - min + 1)) + min  // min~max 정수
```

```txt
Math.floor vs Math.trunc (음수에서 다름):
  Math.floor(-3.2) = -4  (내림 = 더 작은 쪽)
  Math.trunc(-3.2) = -3  (소수점 제거 = 0 방향)

  양수에서는 동일, 음수에서 다름
  일반적으로 양수 계산에는 Math.floor
  음수도 포함하면 Math.trunc 고려
```

---

# 자주 쓰는 패턴 모음

```typescript
// 소수점 2자리 표시
const price = 9.999;
price.toFixed(2)        // '10.00' (문자열)
Math.round(price * 100) / 100  // 10 (숫자)

// 문자열 → 숫자 안전하게
const val = '42';
const n = Number(val);
if (Number.isNaN(n)) throw new Error('숫자가 아님');

// 범위 내 값으로 제한 (clamp)
const clamped = Math.min(Math.max(value, min), max);

// 배열에서 최대·최솟값
const max = Math.max(...numbers);
const min = Math.min(...numbers);

// 랜덤 정수
const randomInt = (min: number, max: number) =>
  Math.floor(Math.random() * (max - min + 1)) + min;
```