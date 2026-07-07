---
aliases:
  - Any
  - void
  - never
  - unknown
tags:
  - TypeScript
related:
  - "[[00_JS_Ecosystem_HomePage]]"
  - "[[TS_Type_Guards]]"
  - "[[TS_Generics]]"
  - "[[TS_TypeAssertion]]"
  - "[[JS_Promise]]"
---
# TS_Unknown_Any — any · unknown · void · never

> [!info] 
> `any`는 타입 검사를 끄는 탈출구, `unknown`은 "타입을 아직 모른다"는 안전한 표현이다.
>  `void`는 "반환값 없음", `never`는 "절대 반환 안 함" — 비슷해 보이지만 쓰이는 자리가 다르다.

---

# 한눈에 비교 ⭐️⭐️⭐️⭐️

|타입|의미|읽기|쓰기|언제|
|---|---|---|---|---|
|`any`|타입 검사 완전 비활성화|뭐든 허용|뭐든 허용|마이그레이션 임시, 피해야 함|
|`unknown`|"타입을 아직 모름"|좁혀야만 사용 가능|뭐든 저장 가능|catch 블록, 외부 데이터, API 응답|
|`void`|"의미 있는 반환값 없음"|undefined만|—|함수 반환 타입, 콜백 타입|
|`never`|"절대 도달 불가"|아무것도 없음|아무것도 없음|항상 throw, 무한루프, 타입 소진|

---

# any — 타입 검사 비활성화 ⭐️⭐️⭐️

```typescript
let x: any = '문자열';
x.toFixed(2);    // ← 문자열에 없는 메서드인데 TS 에러 없음 ⚠️
x = 42;          // OK
x = { a: 1 };   // OK

// any를 다른 타입에 대입해도 에러 없음
const n: number = x;  // OK (런타임에서 터질 수 있음)
```

```txt
any의 문제:
  타입 검사를 완전히 끄기 때문에 자동완성도, 에러 감지도 없어짐
  any 하나가 퍼지면 주변 타입도 any로 오염됨 (any 전파)
  "지금 당장 귀찮으니 나중에 고치자"는 기술 부채

쓰는 게 나은 경우:
  레거시 JS 코드를 TS로 마이그레이션하는 중간 단계
  외부 라이브러리 타입 선언이 없어서 어쩔 수 없을 때
  → 가능하면 unknown으로 대체
```

---

# unknown — 안전한 "타입 모름" ⭐️⭐️⭐️⭐️

```typescript
let x: unknown = '문자열';
x.toFixed(2);    // ← TS 에러: unknown은 좁히기 전에 쓸 수 없음 ✅ (에러가 나는 게 맞음)
x = 42;          // OK — 저장은 됨
x = { a: 1 };   // OK

// unknown을 쓰려면 타입 가드로 먼저 좁혀야 함
if (typeof x === 'number') {
  x.toFixed(2);  // 여기서는 number 확정 → OK
}
```

```txt
unknown vs any:
  둘 다 "어떤 값이든 저장 가능"하지만
  any   → 꺼내서 쓸 때도 검사 없이 그냥 통과 (위험)
  unknown → 꺼내서 쓸 때 반드시 좁혀야 함 (안전)

"타입이 뭔지 모를 때는 any 대신 unknown" 이 원칙
```

## 대표 사용처 — catch 블록 ⭐️⭐️⭐️⭐️

```typescript
try {
  await fetchData();
} catch (err) {
  // err의 타입: unknown (TS 4.0+)
  // → throw된 게 Error 객체인지 아닌지 TS는 모름 (뭐든 throw 가능하므로)

  // ❌ err.message 바로 쓰면 에러 — unknown이라 좁혀야 함
  console.log(err.message);

  // ✅ instanceof로 좁히기
  const message = err instanceof Error ? err.message : String(err);
  setError(message);
}
```

```txt
TS 4.0 이전에는 catch의 err가 any였음 → err.message를 그냥 쓸 수 있었음
TS 4.0+에서 unknown으로 바뀜 → 더 안전해졌지만 instanceof 체크가 필요해짐

tsconfig의 useUnknownInCatchVariables: true (TS 4.4+ 기본값, strict 모드 포함)
  → 이 설정이 활성화되면 catch(err)의 err가 unknown
```

## 대표 사용처 — 외부 데이터, API 응답 ⭐️⭐️

```typescript
// JSON.parse는 반환 타입이 any — 직접 unknown으로 받는 게 더 안전
const raw: unknown = JSON.parse(responseText);

if (
  typeof raw === 'object' &&
  raw !== null &&
  'message' in raw &&
  typeof (raw as { message: unknown }).message === 'string'
) {
  console.log(raw.message); // 여기서는 string 확정
}
```

---

# void ⭐️⭐️⭐️⭐️

```txt
void는 두 가지 다른 자리에서 쓰임 — 헷갈리기 쉬운 포인트
```

## ① 함수 반환 타입으로서의 void

```typescript
function log(msg: string): void {
  console.log(msg);
  // return; 또는 아무것도 안 반환
}

// 콜백 타입에서
type Handler = () => void;
const onClick: Handler = () => console.log('클릭'); // OK
```

```txt
void 반환 타입의 의미:
  "이 함수를 호출하는 쪽에서 반환값을 사용하지 않을 것" 이라는 선언
  실제로 undefined를 반환해도 되고 아무것도 안 반환해도 됨

  ⚠️ void 반환 타입 함수도 값을 return할 수 있음 (TS가 그냥 무시함)
  → 반환값을 사용할 수 없다는 것이지, return 자체를 막는 게 아님
```

## ② JS 연산자로서의 void ⭐️⭐️⭐️

```typescript
// void 표현식 = 뒤에 오는 걸 실행하고 결과를 undefined로 버림
void runAction(() => likePost(id));
void load();

// onClick에서 floating promise 방지
<button onClick={() => void runAction(() => likePost(id))}>
//                      ↑ runAction()이 반환하는 Promise를 undefined로 버림
//                        onClick이 Promise를 반환하면 ESLint 경고 발생
```

```txt
이 void는 타입이 아니라 JS 연산자 — typeof / instanceof 같은 것처럼
  void 0       === undefined (오래된 코드에서 undefined 대신 씀)
  void fn()    fn을 실행하고 결과를 버림

floating promise:
  async 함수를 호출하면 항상 Promise를 반환함
  그걸 아무도 await/catch 안 하면 "떠있는 Promise" → ESLint 경고
  void로 "의도적으로 버린다"고 선언하면 경고가 사라짐
```

## `Promise<void>` vs `Promise<unknown> `⭐️⭐️⭐️⭐️

```typescript
// Promise<void> — "resolve하는 값이 없다"
async function doSomething(): Promise<void> {
  await fetch('/api/action');
  // return; 또는 아무것도 안 반환
}

// Promise<unknown> — "resolve하는 값의 타입이 뭔지 상관없다"
const runAction = async (fn: () => Promise<unknown>) => {
  await fn(); // fn()이 무엇을 resolve하든 어차피 안 씀
};
```

```txt
이게 이번에 헷갈렸던 부분:

  fn: () => Promise<void>
    → fn이 반드시 void(undefined)를 resolve해야 함
    → createFriendRequest()가 Promise<ApiFriendship>을 반환하면 TS 에러

  fn: () => Promise<unknown>
    → fn이 뭘 resolve하든 상관없음
    → runAction은 await fn()만 하고 결과를 쓰지 않으므로 가장 넓은 타입이 맞음

  void runAction(...)  ← onClick에서 쓰는 void
    → 타입이 아니라 JS 연산자: runAction()의 반환 Promise를 버림
    → 이 void는 fn의 타입과 아무 관계 없음

정리:
  Promise<void>    = resolve 값 없음 (함수 반환 타입)
  Promise<unknown> = resolve 값이 뭔지 모르거나 상관없음 (콜백 인자 타입)
  void expr        = JS 연산자, 표현식을 실행하고 결과를 버림 (타입 아님)
```

---

# never ⭐️⭐️⭐️

```typescript
// 항상 throw하거나 무한루프 → 반환 자체가 불가능한 함수
function fail(msg: string): never {
  throw new Error(msg);
}

function infinite(): never {
  while (true) {}
}
```

## void vs never ⭐️⭐️⭐️

| |`void`|`never`|
|---|---|---|
|의미|반환값이 없음 (undefined)|반환 자체가 불가능|
|실행 흐름|함수 끝나면 다음 줄로 이동|다음 줄에 절대 도달 안 함|
|예시|`console.log()`|`throw`, 무한루프|

## never로 타입 소진 확인 ⭐️⭐️⭐️

```typescript
type Shape = 'circle' | 'square' | 'triangle';

function getArea(shape: Shape): number {
  switch (shape) {
    case 'circle':   return Math.PI;
    case 'square':   return 1;
    case 'triangle': return 0.5;
    default:
      const _exhaustive: never = shape; // 모든 케이스를 처리했으면 shape = never
      throw new Error(`처리되지 않은 shape: ${shape}`);
  }
}
```

```txt
Shape에 'pentagon'이 추가됐는데 switch에 case를 안 추가하면
default의 shape가 'pentagon' (never가 아님) → const _exhaustive: never = shape 에서 TS 에러
→ 새 타입이 추가됐을 때 처리를 빠뜨리면 컴파일 타임에 바로 알 수 있음
```

---

# 실전 패턴 모음 ⭐️⭐️⭐️

## err 처리 — 가장 자주 쓰는 unknown 좁히기

```typescript
// 패턴 1 — instanceof Error (가장 흔한 경우)
const message = err instanceof Error ? err.message : String(err);

// 패턴 2 — 여러 케이스 분기
function getErrorMessage(err: unknown): string {
  if (err instanceof Error)        return err.message;
  if (typeof err === 'string')     return err;
  if (typeof err === 'object' && err !== null && 'message' in err) {
    return String((err as { message: unknown }).message);
  }
  return '알 수 없는 오류';
}
```

## any를 unknown으로 점진적 대체

```typescript
// 레거시 코드: any
function parseConfig(raw: any) {
  return raw.host; // 타입 검사 없음
}

// 개선: unknown + 좁히기
function parseConfig(raw: unknown) {
  if (typeof raw !== 'object' || raw === null) throw new Error('invalid config');
  if (!('host' in raw)) throw new Error('host missing');
  return (raw as { host: string }).host;
}
```

---

# 한눈에

```txt
any
  타입 검사 완전 비활성화
  → 마이그레이션 임시 또는 타입 선언이 없는 외부 라이브러리에서만 (피해야 함)

unknown
  저장은 가능, 꺼내서 쓸 때는 반드시 타입 가드로 좁혀야 함
  → catch(err), JSON.parse, 외부 API 응답, 제네릭 콜백 인자

void (타입)
  "의미 있는 반환값 없음" — undefined를 반환하거나 아무것도 안 반환
  → 함수 반환 타입, 콜백 타입 (onClick: () => void)

void (JS 연산자)
  표현식을 실행하고 결과를 undefined로 버림 — 타입이 아님
  → onClick에서 floating promise 방지 (void runAction(...))

never
  절대 도달 불가 — throw하거나 무한루프
  → 타입 소진 확인 (default: never에 나머지 할당해서 빠뜨린 case 감지)

Promise<void>     resolve 값이 없는 Promise
Promise<unknown>  resolve 값이 뭐든 상관없는 Promise (콜백의 반환을 안 쓸 때)
void runAction()  JS 연산자로 Promise를 버림 — 위 두 개와 전혀 다른 개념

타입 좁히기 방법 → [[TS_Type_Guards]]
```