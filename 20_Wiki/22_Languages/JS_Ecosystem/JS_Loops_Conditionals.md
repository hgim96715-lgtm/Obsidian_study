---
aliases:
  - if else
  - switch
  - for loop
  - for of
  - for in
  - while
tags:
  - JavaScript
related:
  - "[[00_JS_Ecosystem_HomePage]]"
  - "[[JS_Loops_Conditionals]]"
  - "[[NextJS_API_Mapper]]"
---

# JS_Loops_Conditionals — 조건문과 반복문

> [!info] 
> 조건문(if/switch/삼항)은 "분기"고 반복문(for/while/for-of/for-in)은 "반복"이다. 
> 배열을 돌 때는 보통 [[JS_Array_Methods]]의 메서드들로 충분하지만, 중간에 멈춰야 하거나(break) 순서대로 await해야 할 때는 반복문이 더 적합하다.

---

# 조건문 — 분기 ⭐️⭐️⭐️

## if / else / else if

```typescript
if (status === 401) {
  redirectToLogin();
} else if (status === 403) {
  showForbidden();
} else {
  showGenericError();
}
```

## switch — 같은 값에 대해 여러 분기

```typescript
switch (status) {
  case 401:
    redirectToLogin();
    break;        // ⚠️ 빠뜨리면 다음 case로 그대로 흘러감(fall-through)
  case 403:
    showForbidden();
    break;
  default:
    showGenericError();
}
```

```txt
switch가 if/else보다 나은 경우: "하나의 값"을 여러 고정된 케이스와 비교할 때 — 가독성이 더 좋음
break를 빠뜨리면 그 case가 끝나도 멈추지 않고 다음 case까지 계속 실행됨(fall-through)
  → 의도적으로 여러 case를 묶어서 같은 동작을 시키고 싶을 때를 빼면, 항상 break를 챙길 것
```

## 삼항 연산자 — 값을 바로 만들 때

```typescript
const label = isDark ? '다크 모드' : '라이트 모드';
```

```txt
if/else는 "실행할 코드 블록"이 다를 때, 삼항연산자는 "변수에 대입하거나 바로 return할 값"이 다를 때
  → const x; if (cond) { x = a; } else { x = b; } 를 한 줄로 줄인 것이 삼항연산자
중첩 삼항연산자(a ? b : c ? d : e)는 가독성이 떨어져서 보통 if/else나 switch로 푸는 게 나음
```

---

# 반복문 — 반복 ⭐️⭐️⭐️

## for — 가장 기본적인 형태

```typescript
for (let i = 0; i < arr.length; i++) {
  console.log(arr[i]);
}
```

```txt
인덱스(i)를 직접 다뤄야 할 때(예: 거꾸로 돌기, 두 칸씩 건너뛰기) 외에는
요즘은 아래 for...of나 [[JS_Array_Methods]]의 메서드로 대체하는 경우가 더 많음
```

## for...of — 값을 직접 순회

```typescript
for (const item of arr) {
  console.log(item); // 인덱스 없이 값 자체를 바로 받음
}
```

## for...in — 키(또는 인덱스)를 순회 ⚠️

```typescript
const obj = { a: 1, b: 2 };
for (const key in obj) {
  console.log(key, obj[key]); // 'a' 1, 'b' 2
}
```

```txt
⚠️ for...in을 배열에 쓰면 안 되는 이유:
  배열에 for...in을 쓰면 인덱스를 "문자열"로 순회함('0', '1', '2'...) — 숫자 인덱스가 아님
  거기다 배열에 직접 추가한 커스텀 속성까지 같이 순회될 수 있어서 예상치 못한 값이 섞일 위험이 있음
  → 배열은 for...of(값) 또는 배열 메서드, 객체는 for...in(키) 또는 Object.keys/entries
```

|반복문|순회하는 것|주로 쓰는 대상|
|---|---|---|
|`for`|인덱스를 직접 제어|인덱스 자체가 필요한 경우|
|`for...of`|값|배열, Map, Set 등 (iterable)|
|`for...in`|키(또는 인덱스를 문자열로)|객체 — 배열에는 권장 안 함|

## while / do-while

```typescript
let attempts = 0;
while (attempts < 3) {
  attempts++;
}
```

```txt
"몇 번 돌지 미리 모르고, 조건이 참인 동안 계속 반복"해야 할 때 — for보다 while이 자연스러움
do-while은 "일단 한 번은 무조건 실행하고, 그 다음부터 조건을 검사"하는 드문 경우에만 씀
```

---

# 언제 배열 메서드 대신 반복문을 쓰나 ⭐️⭐️⭐️⭐️

```txt
대부분의 "배열을 돌면서 뭔가 하는" 코드는 [[JS_Array_Methods]]의 map/filter/find 등으로 충분함
반복문(특히 for...of)이 더 적합한 경우는 다음 둘 중 하나일 때:
```

|상황|이유|
|---|---|
|중간에 멈춰야 함 (break)|map/filter/forEach는 도중에 멈추는 기능이 없음 — 끝까지 다 돎|
|배열 요소를 순서대로 하나씩 await해야 함|forEach는 async를 기다려주지 않음(자세한 함정은 [[JS_Array_Methods]] 참고) — for...of + await가 정확히 순서대로 기다림|

```typescript
// break가 필요한 예 — 찾으면 더 이상 돌 필요 없음
for (const user of users) {
  if (user.id === targetId) {
    found = user;
    break; // 찾았으니 나머지는 안 돌아도 됨
  }
}
// (참고: 이 경우는 보통 배열 메서드의 find()로 더 간단히 대체 가능 — find가 정확히 이 일을 함)
```

```txt
실무에서는 "찾기"는 거의 항상 find()로 충분하고, 진짜로 반복문이 필요한 경우는
"순서대로 await"하는 경우가 가장 흔함 — 그 외엔 배열 메서드 쪽이 더 짧고 의도가 명확함
```

---

# 한눈에

| 문법                   | 역할                                                  |
| -------------------- | --------------------------------------------------- |
| `if/else`, `else if` | 조건에 따라 다른 코드 블록 실행                                  |
| `switch`             | 하나의 값을 여러 고정된 케이스와 비교 — break 빠뜨리면 fall-through     |
| `A ? B : C`          | 조건에 따라 다른 "값"을 바로 만들 때                              |
| `for`                | 인덱스를 직접 다뤄야 할 때                                     |
| `for...of`           | 배열 등 iterable의 값을 직접 순회                             |
| `for...in`           | 객체의 키를 순회 — 배열에는 권장 안 함                             |
| `while`              | 몇 번 돌지 모르고 조건이 참인 동안 반복                             |
| 반복문이 배열 메서드보다 나은 경우  | break로 중간에 멈춰야 할 때, for...of + await로 순서대로 기다려야 할 때 |