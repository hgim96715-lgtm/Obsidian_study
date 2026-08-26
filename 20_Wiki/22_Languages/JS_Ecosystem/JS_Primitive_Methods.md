---
aliases:
  - 해시 함수
  - fallback
  - FNV-1a
  - Math
  - NFKC
  - normalize
  - Number
  - String
tags:
  - JavaScript
related:
  - "[[00_JS_Ecosystem_HomePage]]"
  - "[[JS_Array_Methods]]"
  - "[[JS_Operators]]"
  - "[[TS_Generics]]"
  - "[[JS_Patterns]]"
---
# JS_Primitive_Methods — String · Number · Math

> [!info] 
> 원시 타입(string, number)의 메서드들. 
> string 메서드는 항상 새 문자열을 반환 — 원본을 바꾸지 않는다.
>  `normalize('NFKC')`로 전각·유니코드 정규화. 
>  `charCodeAt(i)`로 문자 → 숫자 변환 (해시 함수에 사용). 
>  `Math.imul` — 32비트 정수 곱셈. 
>  `Number.isNaN` vs `isNaN` 혼동 주의. 
>  문자열 조합 패턴(템플릿 리터럴·배열 join), 검색 쿼리 정규화·폴백 패턴도 이 파일에 정리.

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

배열에도 slice가 있음 — thread.slice(1), arr.slice(0, N)
→ [[JS_Array_Methods]] # slice 섹션
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

## normalize — 유니코드 정규화 ⭐️⭐️⭐️

```typescript
// 전각 문자 → 반각 문자, 유사한 유니코드 통일
'ａｂｃ'.normalize('NFKC')    // 'abc'  (전각 영문 → 반각)
'１２３'.normalize('NFKC')    // '123'  (전각 숫자 → 반각)
'ｱｲｳ'.normalize('NFKC')     // 'アイウ' (반각 카타카나 → 전각)
'café'.normalize('NFC')       // 'café'  (분리된 é → 합성 é)
```

```txt
유니코드 정규화 형식:
  NFC  — 합성 형식 (문자 합치기)
  NFD  — 분해 형식 (문자 쪼개기)
  NFKC — 호환 합성 (전각/반각 통일 + 합성) ← 검색에 가장 유용
  NFKD — 호환 분해

NFKC가 검색에 유용한 이유:
  '１２３' 과 '123' 은 다른 문자 코드지만 같은 의미
  normalize('NFKC') 하면 둘 다 '123' 으로 통일
  → 전각으로 입력해도 검색 결과가 나옴
```

## 검색 쿼리 정규화 패턴 ⭐️⭐️⭐️

```typescript
/** 검색어 앞뒤·연속 공백, 전각 문자 정리 */
function normalizeSearchQuery(query: string): string {
  return query
    .normalize('NFKC')  // 전각 → 반각, 유니코드 통일
    .trim()             // 앞뒤 공백 제거
    .replace(/\s+/g, ' '); // 연속 공백 → 하나로 (탭, 줄바꿈 포함)
}

normalizeSearchQuery('  hello   world  ')  // 'hello world'
normalizeSearchQuery('１２３　abc')         // '123 abc'  (전각 정리)
```


```typescript
/** 한글 붙여쓰기 → 공백 삽입 폴백 (외부 API 검색 보정용)
 *  폴백(fallback) = 1차 검색이 실패했을 때 대신 시도하는 쿼리 목록 반환
 *  예: "싱스트리트" 검색 결과 없음 → "싱 스트리트"로 재시도
 */
function searchQueryFallbacks(query: string): string[] {
  if (/\s/.test(query) || query.length < 3) return [];
  // 이미 공백 있거나 3자 미만이면 폴백 불필요

  const out: string[] = [];
  out.push(`${query.slice(0, 1)} ${query.slice(1)}`);
  // "싱스트리트" → "싱 스트리트"

  if (query.length >= 4) {
    out.push(`${query.slice(0, 2)} ${query.slice(2)}`);
    // "싱스트리트" → "싱스 트리트"
  }
  return out;
}

searchQueryFallbacks('싱스트리트')  // ['싱 스트리트', '싱스 트리트']
searchQueryFallbacks('어벤져스')    // ['어 벤져스', '어벤 져스']
```

```txt
폴백(fallback)이란:
  주요 방법이 실패했을 때 대신 시도하는 것
  1차: "싱스트리트" 검색 → 결과 없음
  폴백: "싱 스트리트"로 재검색 → 결과 있음

  searchQueryFallbacks가 필요한 이유:
  TMDB 같은 외부 API는 한글 제목을 띄어쓰기 기준으로 색인
  사용자가 "싱스트리트"로 붙여서 검색하면 결과 없음
  → 폴백 목록을 만들어 순서대로 재검색

  /\s/.test(query):
  이미 공백이 있으면 → 띄어쓰기를 알고 입력한 것 → 폴백 불필요

  slice(0, 1) + ' ' + slice(1):
  첫 글자 + 공백 + 나머지
  "싱스트리트" → "싱" + " " + "스트리트" = "싱 스트리트"
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

## charCodeAt — 문자 → 숫자 코드 ⭐️⭐️⭐️

```typescript
// 문자열의 특정 위치 문자를 숫자(유니코드 코드 포인트)로 반환
'A'.charCodeAt(0)      // 65
'a'.charCodeAt(0)      // 97
'가'.charCodeAt(0)     // 44032
'hello'.charCodeAt(1)  // 101  ('e')

// 문자열을 순서대로 숫자로 변환
for (let i = 0; i < 'hi'.length; i++) {
  console.log('hi'.charCodeAt(i));  // 104, 105
}
```

```txt
charCodeAt이 필요한 이유:
  문자를 직접 수식에 쓸 수 없음 ('A' + 1 = 'A1' 문자열 연결)
  charCodeAt으로 숫자로 바꾼 다음 수식 적용

  주로 쓰이는 곳:
  해시 함수 — 문자열을 숫자 하나로 변환 (아래 해시 함수 섹션)
  문자열 비교 — 알파벳 순서 비교
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

# Math.imul — 32비트 정수 곱셈 ⭐️⭐️

```typescript
Math.imul(3, 4)            // 12
Math.imul(0xffffffff, 5)   // -5 (32비트 오버플로우)

// 일반 곱셈과의 차이
3 * 4                      // 12       (64비트 부동소수점)
Math.imul(3, 4)            // 12       (32비트 정수)

// 큰 수에서 차이 발생
Math.imul(0xffffffff, 0xffffffff)  // 1  (32비트 넘치면 버림)
0xffffffff * 0xffffffff            // 1.844674... (부동소수점 정밀도 손실)
```

```txt
Math.imul이 필요한 이유:
  해시 함수에서 큰 수를 곱할 때
  일반 * 연산은 부동소수점 → 큰 수에서 정밀도 손실
  Math.imul은 32비트 정수 범위에서 정확하게 곱셈
  → 해시 계산처럼 비트 조작이 필요한 곳에서 사용

>>> 0 (부호 없는 오른쪽 시프트):
  Math.imul 결과가 음수일 수 있음 (32비트 오버플로우)
  >>> 0 으로 양수 32비트 정수로 변환
  h >>> 0 → 항상 0 이상의 정수
```

---

# 해시 함수 — 문자열 → 고정된 숫자 ⭐️⭐️⭐️

## 해시란

```txt
해시(Hash) = 임의 길이의 데이터를 고정 길이의 숫자로 변환하는 것

  입력: "hello-world-post-id-abc123" (긴 문자열)
  출력: 2847291048                   (32비트 숫자 하나)

  "안녕하세요"도 숫자 하나
  "a"도 숫자 하나
  → 길이에 상관없이 항상 같은 크기의 숫자

왜 필요한가:
  문자열을 직접 비교하면 글자 수만큼 시간이 걸림
  숫자 하나로 바꾸면 비교가 O(1)
  배열 인덱스로 쓸 수 있음 (hash % 배열길이)
  같은 입력 → 항상 같은 출력 → 색상·위치 배정 등에 활용
```

## 해시 함수의 3가지 특성

```txt
1. 결정론적 (Deterministic):
   같은 입력 → 항상 같은 출력
   "abc" → 언제 실행해도 항상 2851307223

2. 단방향 (One-way):
   출력 → 원래 입력으로 복원 불가
   2851307223 → "abc"를 역으로 계산할 방법 없음
   (암호화와 다름 — 복호화가 설계상 불가)

3. 균일 분포 (Avalanche Effect):
   입력이 조금만 달라도 출력이 크게 달라짐
   "abc" → 2851307223
   "abd" → 9284019283  (한 글자만 다른데 결과가 완전히 다름)
   → 다른 입력이 같은 해시를 갖는 "충돌"을 줄임
```

## FNV-1a란

```txt
FNV = Fowler–Noll–Vo
  1991년 Glenn Fowler, Landon Noll, Phong Vo가 설계한 해시 알고리즘

FNV-1a 변형:
  빠르고 구현이 단순 (XOR + 곱셈 두 연산만)
  균일한 분산 (충돌이 적음)
  암호학적 안전성은 없음 (보안용 아님, 성능용)

사용처:
  데이터베이스 해시 테이블
  캐시 키 생성
  UI에서 id → 색상·위치 배정
  데이터 무결성 체크섬
```

## FNV-1a 코드 분해

```typescript
function hashId(id: string): number {
  let h = 2166136261;
  //       ↑ FNV offset basis: 32비트 해시의 초기값 (마법 상수)
  //         왜 이 숫자? — 해시 분산이 가장 균일하도록 실험적으로 결정됨

  for (let i = 0; i < id.length; i++) {
    h ^= id.charCodeAt(i);
    // ↑ XOR: 현재 해시 h와 이 글자의 코드를 비트 단위로 섞음
    //   XOR(^) = 같으면 0, 다르면 1
    //   각 글자가 해시에 영향을 줌

    h = Math.imul(h, 16777619);
    //              ↑ FNV prime: 32비트용 소수 (마법 상수)
    //                곱셈으로 비트를 넓게 퍼뜨림 (avalanche effect)
    //                Math.imul = 32비트 정수 곱셈 (정밀도 손실 없음)
  }

  return h >>> 0;
  //       ↑ 부호 없는 32비트 양수로 변환
  //         Math.imul 결과가 음수일 수 있어서 항상 양수로
}
```


```txt
알고리즘 흐름 (문자 하나씩):
  초기값 h = 2166136261
  'h' (104) → h = h XOR 104 → h = h × 16777619
  'i' (105) → h = h XOR 105 → h = h × 16777619
  최종 h >>> 0 → 양수 32비트 정수

  각 글자가 XOR로 섞이고 곱셈으로 퍼짐
  → 순서도 중요 ("ab"와 "ba"는 다른 결과)
```

## 실전 활용

```typescript
const COLORS = ['#e74c3c', '#3498db', '#2ecc71', '#9b59b6', '#f39c12'];
const LAYOUTS = ['top-left', 'center', 'bottom-right', 'top-right'];

// id → 항상 같은 색상 (새로고침해도 바뀌지 않음)
const color  = COLORS[hashId(post.id) % COLORS.length];
const layout = LAYOUTS[hashId(post.id) % LAYOUTS.length];

// 같은 id → 항상 같은 위치·색상 배정
// 다른 id → 다른 결과 (균일하게 분산)
```

```txt
주의: FNV-1a는 보안용이 아님
  비밀번호 해시 → bcrypt, argon2 (보안 해시)
  데이터 무결성 → SHA-256 (암호학적 해시)
  UI 배정·캐시 키 → FNV-1a, djb2 (비보안 해시, 빠름)
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