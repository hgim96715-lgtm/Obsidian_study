---
aliases:
  - Boolean coercion
  - falsy
  - truthy
tags:
  - JavaScript
related:
  - "[[00_JS_Ecosystem_HomePage]]"
  - "[[JS_Operators]]"
  - "[[JS_OptionalChaining]]"
  - "[[TS_PartialUpdate]]"
---

# JS_Truthy_Falsy — 어떤 값이 true/false로 취급되는가

> [!info] 
> JS는 `if`, `&&`, `||`, 삼항연산자 같은 곳에서 boolean이 아닌 값도 자동으로 true/false처럼 취급한다. 
> "falsy"한 값은 딱 7개뿐이고, 그 외의 모든 값(빈 배열·빈 객체 포함!)은 truthy다.

---

# Falsy 값 — 정확히 이 7개뿐 ⭐️⭐️⭐️⭐️

|값|비고|
|---|---|
|`false`|boolean 자체|
|`0`|숫자 0 (`-0`도 포함)|
|`''`|빈 문자열|
|`null`||
|`undefined`||
|`NaN`||
|`0n`|BigInt의 0 (잘 안 보이는 드문 경우)|

```txt
이 7개를 제외한 모든 값은 예외 없이 truthy임 — "이것만 외우면" 나머지는 자동으로 정해짐
```

---

# 의외로 truthy인 것들 — 가장 헷갈리는 지점 ⭐️⭐️⭐️⭐️

```typescript
if ('0')      // truthy! — 문자열 '0'은 빈 문자열이 아니므로 falsy가 아님
if ('false')  // truthy! — 문자열일 뿐, boolean false가 아님
if ([])       // truthy! — 빈 배열도 "값이 있는 객체"라서 truthy
if ({})       // truthy! — 빈 객체도 마찬가지
```

```txt
왜 빈 배열/빈 객체가 truthy인가:
  배열/객체는 내용이 비어있어도 "참조(reference)" 자체는 존재함
  falsy 여부는 "내용이 비어있는가"가 아니라 "그 7개 값 중 하나인가"로만 결정되는데,
  []와 {}는 그 7개 목록에 없으므로 무조건 truthy임

→ "배열이 비어있는지" 확인하고 싶으면 배열 자체를 보지 말고 array.length === 0 으로 확인할 것
  if (array) 는 array가 빈 배열이어도 항상 true가 됨 — 흔한 실수
```

---

# 어디서 truthy/falsy 판단이 쓰이는가 ⭐️⭐️⭐️

| 문맥                      | 동작                                                 |
| ----------------------- | -------------------------------------------------- |
| `if (x)`                | x가 truthy면 블록 실행                                   |
| `!x`                    | x가 falsy면 true, truthy면 false (반전)                 |
| `!!x`                   | x를 boolean으로 명시적으로 변환 (`Boolean(x)`와 동일)           |
| `x && y`                | x가 falsy면 x를 그대로 반환 — 자세한 동작은 [[JS_Operators]] 참고  |
| <code>x &#124; y</code> | x가 truthy면 x를 그대로 반환 — 자세한 동작은 [[JS_Operators]] 참고 |
| `x ? a : b`             | x의 truthy/falsy로 분기                                |
| `while (x)`             | x가 truthy인 동안 반복                                   |

```txt
⚠️ ??(널리시 코얼레싱)는 이 truthy/falsy 판단을 안 씀 — null/undefined인지만 봄
  0과 ''은 falsy지만 ??에서는 "있는 값"으로 취급되어 기본값으로 안 넘어감
  → ?? 와 || 의 정확한 차이는 [[JS_OptionalChaining]] 참고 (이 노트의 falsy 개념이 그 차이의 토대임)
```

---

# Boolean(x) — 명시적으로 변환하기 ⭐️

```typescript
Boolean(0)         // false
Boolean('hello')   // true
Boolean([])         // true (빈 배열도 truthy)

!!0          // false — Boolean(0)과 완전히 동일
!!'hello'    // true
```

```txt
!!x 는 Boolean(x)의 축약형 표기 — 한 번 뒤집고(!) 다시 뒤집어서(!) 원래의 truthy/falsy를
순수한 boolean 값으로 바꿔주는 관용적인 트릭
```

---

# 흔한 실수 ⭐️⭐️⭐️⭐️

|실수|문제|해결|
|---|---|---|
|`if (array)`로 배열이 비었는지 확인|빈 배열도 truthy라서 항상 실행됨|`if (array.length === 0)`|
|`if (object)`로 객체가 "내용 없음"인지 확인|빈 객체도 truthy|`Object.keys(object).length === 0`|
|`count && <span>{count}개</span>`|count가 0이면 화면에 "0"이 그대로 찍힘|자세한 내용과 해결은 [[JS_Operators]] 참고|
|문자열 `'false'`를 `if (value)`로만 체크|문자열은 truthy라서 분기가 반대로 됨|boolean 값과 문자열 값을 헷갈리지 않게 타입을 명확히 구분|

---

# 한눈에

```txt
falsy 7개: false, 0, '', null, undefined, NaN, 0n
나머지 전부: truthy (빈 배열 [], 빈 객체 {}, 문자열 '0'/'false' 포함)

배열/객체가 비었는지는 array.length / Object.keys(obj).length 로 확인 — truthy 체크로는 안 됨
??는 truthy/falsy가 아니라 null/undefined만 구분 — ||와 다름
```