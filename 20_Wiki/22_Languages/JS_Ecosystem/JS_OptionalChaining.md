---
aliases: [coalescing, doubleQuestion, nullish, optional chaining, questionDot]
tags:
  - JavaScript
related:
  - "[[00_JS_Ecosystem_HomePage]]"
  - "[[NextJS_API_Client]]"
---

# JS_OptionalChaining — ?. 와 ??

> [!info] 
> `?.`는 "중간에 null/undefined가 있으면 거기서 멈추고 undefined를 반환"하는 옵셔널 체이닝이고, 
> `??`는 "왼쪽이 null/undefined일 때만(0이나 ''는 살리고) 오른쪽 기본값을 쓰는" 널리시 코얼레싱이다. 
> 거의 모든 비동기/응답 처리 코드에서 이 둘이 짝으로 등장한다.

---

# ?. — 옵셔널 체이닝 ⭐️⭐️⭐️⭐️

```typescript
user?.passwordHash
// user가 null/undefined면 → 그 자리에서 멈추고 undefined 반환 (에러 안 남)
// user가 있으면 → user.passwordHash 그대로 읽음

// ?. 없이 쓰면:
user.passwordHash
// user가 null이면 → "Cannot read properties of null" 같은 TypeError로 그 자리에서 죽음
```

```txt
체인이 길어도 중간 어디서든 끊어주는 게 ?.의 핵심:
  a?.b?.c?.d
  → a가 없으면 거기서 끝 (b/c/d는 시도조차 안 함, undefined 반환)
  → a는 있는데 b가 없으면 거기서 끝
  → 끝까지 다 있으면 d 값까지 정상적으로 도달
```

## 함수 호출에도 적용 — ?.()

```typescript
onClose?.();  // onClose가 함수면 호출, undefined/null이면 그냥 아무 일도 안 일어남
```

## 배열/동적 키 접근 — ?.[ ]

```typescript
arr?.[0]            // arr가 없으면 undefined, 있으면 첫 번째 요소
obj?.[dynamicKey]    // 대괄호 접근에도 동일하게 적용
```

---

# ?? — 널리시 코얼레싱 ⭐️⭐️⭐️⭐️

```typescript
const message = error?.message ?? '알 수 없는 오류';
// error?.message가 null 또는 undefined일 때만 '알 수 없는 오류' 사용
```

```txt
?.와 ??가 항상 짝으로 보이는 이유:
  ?.로 "혹시 없을 수도 있는 값"을 안전하게 꺼내고
  ??로 "그게 진짜 없으면(null/undefined) 쓸 기본값"을 바로 옆에 정해두는 조합이 매우 흔함
```

## ?? vs || — 가장 헷갈리는 차이 ⭐️⭐️⭐️⭐️

```typescript
const count = 0;
count || 10   // 10  ← 0은 falsy라서 ||가 기본값으로 넘어가버림 (의도와 다를 수 있음)
count ?? 10   // 0   ← 0은 null/undefined가 아니므로 ??는 그대로 0을 유지함
```

| 연산자             | 기본값으로 넘어가는 조건                                                 |
| --------------- | ------------------------------------------------------------- |
| <code>\|</code> | 왼쪽이 falsy 전부 (`0`, `''`, `false`, `null`, `undefined`, `NaN`) |
| `??`            | 왼쪽이 정확히 `null` 또는 `undefined`일 때만                             |

```txt
"값이 0이거나 빈 문자열일 수도 있는데, 그것도 유효한 값으로 살리고 싶다"면 반드시 ??
페이지 번호(0페이지가 유효), 빈 문자열 입력 등을 다룰 때 ||를 쓰면 의도와 다르게 동작하기 쉬움
```

---

# 실전 — 에러 메시지 추출 패턴 ⭐️⭐️⭐️

```typescript
const error = (await res.json()) as { message?: string | string[] } | null;
const message = Array.isArray(error?.message) ? error.message[0] : error?.message;
throw new Error(message ?? `요청 실패: ${res.status} ${res.statusText}`);
```

```txt
한 줄씩:
  error?.message       — error 자체가 null일 수도 있어서 ?. 로 안전하게 접근
  Array.isArray(...)    — message가 string인지 string[]인지 분기
  message ?? `...`      — 위 과정에서 message 자체가 끝까지 없었다면(null/undefined)
                          그제서야 직접 만든 기본 메시지로 대체
```

---

# 한눈에

| 연산자                 | 역할                                                     |
| ------------------- | ------------------------------------------------------ |
| `a?.b`              | a가 null/undefined면 그 자리에서 멈추고 undefined 반환             |
| `fn?.()`            | fn이 함수면 호출, 아니면 아무 일도 안 함                              |
| `arr?.[i]`          | 배열/객체가 없으면 undefined, 있으면 그 인덱스/키 값                    |
| `a ?? b`            | a가 null/undefined일 때만 b 사용 (0, '', false는 그대로 유지)      |
| <code>a \| b</code> | a가 모든 falsy 값(0, '', false 포함)일 때 b 사용 — `??`보다 범위가 넓음 |
| 자주 같이 쓰는 조합         | `error?.message ?? '기본 메시지'`                           |