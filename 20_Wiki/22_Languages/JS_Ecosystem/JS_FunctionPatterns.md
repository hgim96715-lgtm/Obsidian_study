---
aliases:
  - 조기반환
  - 함수 설계 패턴
  - force=false
  - Early Return
  - 내부 함수 추출
  - 화살표 함수 암시적 반환
tags:
  - JavaScript
related:
  - "[[00_JS_Ecosystem_HomePage]]"
  - "[[JS_Operators]]"
  - "[[React_AsyncUI]]"
  - "[[NextJS_WebSocket]]"
  - "[[JS_Promise]]"
---
# JS_FunctionPatterns — 함수 설계 패턴

>[!info]
>함수 설계 패턴 모음. 
>옵션 객체, 조기 반환, async 래퍼, 화살표 함수 암시적 반환(`=>{}` vs `=>`) 등. 
>중괄호 없으면 자동 return, 중괄호 있으면 return 키워드 필수 — map 콜백에서 자주 실수.

---

# 옵션 객체 패턴 — `{ force = false }` ⭐️⭐️⭐️⭐️

```txt
함수 인자가 많아지거나 일부가 선택적일 때
boolean 인자를 여러 개 나열하는 대신 객체 하나로 묶는 패턴
```

```typescript
// ❌ 인자가 많아지면 순서 기억이 어려움
touchLastActiveAt(userId, role, true, false, 'admin');

// ✅ 옵션 객체 — 이름으로 의미가 명확
touchLastActiveAt(userId, role, { force: true });
```

## TypeScript 타입 정의

```typescript
interface TouchOptions {
  force?:  boolean;
  silent?: boolean;
}

async function touchLastActiveAt(
  userId:  number,
  role:    string,
  options: TouchOptions = {},  // 통째로 안 넘겨도 됨 (기본값 빈 객체)
): Promise<void> {
  const { force = false, silent = false } = options;

  if (!force && checkRecentlyUpdated()) return;  // force: true면 이 early return 건너뜀
  await update();
}
```

```txt
options = {} 기본값의 의미:
  호출하는 쪽이 세 번째 인자를 아예 안 넘겨도
  내부에서 {}(빈 객체)를 받은 것처럼 동작
  → { force = false, silent = false }가 모두 기본값으로 처리됨

  options?: TouchOptions로 선언하면:
  → options가 undefined일 수 있어서 구조분해 전에 ?? {} 처리 필요 → 번거로움
  → = {} 기본값이 더 깔끔
```

## 호출 방법

```typescript
await touchLastActiveAt(userId, role);                          // 기본값
await touchLastActiveAt(userId, role, { force: true });         // 일부만
await touchLastActiveAt(userId, role, { force: true, silent: true }); // 여러 개
```

## 파라미터에서 바로 구조분해 — 더 짧게

```typescript
async function touchLastActiveAt(
  userId: number,
  role:   string,
  { force = false }: TouchOptions = {},  // 파라미터에서 바로 구조분해
) {
  if (!force && checkRecentlyUpdated()) return;
  await update();
}
```

## force 플래그가 자주 쓰이는 패턴

```txt
"보통은 스킵하지만 이 경우엔 반드시 실행해야 해"
  → 서비스 레벨 스로틀 건너뛰기 → [[NestJS_Throttle]] 참고
  → 캐시 무효화 강제 갱신
  → 조건부 early return 무력화
```

## `Partial<T>` — 옵션 객체 타입 유틸리티 ⭐️⭐️

```typescript
interface UserUpdateData {
  name:  string;
  email: string;
  image: string;
}

// Partial<T> = 모든 필드를 optional로 만듦
async function updateUser(userId: number, data: Partial<UserUpdateData>) {
  // name만, email만, 셋 다 — 어떤 조합이든 타입 통과
}

updateUser(1, { name: '새이름' });  // ✅
```

```txt
옵션 객체는 보통 모든 필드가 선택적
→ interface에 ? 전부 붙이거나 Partial<T>를 쓰거나
→ Partial<T> 상세 → [[TS_Utility_Types]]
```

---

# 조기 반환 (Early Return) ⭐️⭐️⭐️

```typescript
// ❌ 중첩이 깊어짐
function processUser(user: User | null) {
  if (user) {
    if (user.isActive) {
      if (user.role === 'admin') {
        // 실제 로직
      }
    }
  }
}

// ✅ early return으로 평탄하게
function processUser(user: User | null) {
  if (!user) return;
  if (!user.isActive) return;
  if (user.role !== 'admin') return;

  // 실제 로직 — 조건이 모두 통과된 상태에서 작성
}
```

```txt
early return의 장점:
  실패 케이스를 함수 위에서 먼저 처리 → 아래로 내려올수록 조건이 보장됨
  중첩 if를 없애서 인지 부하 감소
  → useEffect 안의 조건 가드 패턴도 같은 원리
    if (!user || !roomId) return;
```

---

# async 래퍼 패턴 ⭐️⭐️⭐️

```typescript
// 여러 이벤트 핸들러에서 같은 try/catch/finally 구조가 반복될 때
const runAction = async (fn: () => Promise<unknown>) => {
  setActing(true);
  setError('');
  try {
    await fn();
    await reload();
  } catch (err) {
    setError(err instanceof Error ? err.message : '요청에 실패했어요.');
  } finally {
    setActing(false);
  }
};

// 사용 — 함수를 인자로 넘김
void runAction(() => deleteComment(id));
void runAction(() => likeComment(id));
```

```txt
fn: () => Promise<unknown>:
  반환 타입을 unknown으로 선언 — 어떤 API 함수든 래핑 가능
  내부에서 결과를 쓰지 않을 때는 unknown이 any보다 안전

runAction이 적합한 경우:
  공통 후처리(reload)가 모든 액션에 동일
  에러 메시지 포맷이 전부 같을 때

개별 핸들러가 더 적합한 경우:
  액션마다 다른 후처리 (삭제 성공 후 편집 모드 닫기 등)
  대상별 pending id를 각자 관리해야 할 때
  → [[React_AsyncUI]] 이벤트 핸들러 섹션 참고
```

---

# 내부 함수 추출 — 즉시 실행 vs 참조로 전달 ⭐️⭐️⭐️⭐️

```txt
같은 코드를 두 가지 시점에 실행해야 할 때
내부 함수로 추출하면 중복 없이 두 경우 모두 처리 가능
```

```typescript
// ❌ 내부 함수 없이 — 같은 코드가 두 번 반복
if (isReady) {
  doWork(data);                   // 즉시 실행
} else {
  waitForReady.once('ready', () => {
    doWork(data);                 // 나중에 실행 — 완전히 같은 코드 반복
  });
}

// ✅ 내부 함수로 추출 — 한 곳에서 정의, 두 곳에서 사용
const execute = () => {
  doWork(data);
};

if (isReady) execute();           // 즉시 실행
else waitForReady.once('ready', execute);  // 나중에 참조로 전달
```

```txt
execute()   → 소괄호 있음 → "지금 즉시 실행"
execute     → 소괄호 없음 → "나중에 실행할 함수를 참조로 넘김"

  once('ready', execute())  — ❌ once 호출 시점에 즉시 실행
  once('ready', execute)    — ✅ ready 이벤트가 오면 그때 execute 실행

이 구분을 모르면:
  once(event, fn()) — fn이 즉시 실행되고 그 반환값(보통 undefined)이 콜백으로 등록
  → 이벤트가 와도 아무 일도 안 일어남
```

## 실전 예 — WebSocket 연결 대기

```typescript
// 연결이 됐으면 지금 emit, 아직이면 연결 완료 후 emit
const doEmit = () => {
  socket.emit('featureA:join', { resourceId }, callback);
};

if (socket.connected) doEmit();            // 즉시 실행
else socket.once('connect', doEmit);       // 참조로 전달 → 나중에 실행
```

```txt
자세한 설명 → [[NextJS_WebSocket]] acknowledgement 섹션
```

## 실전 예 — 조건에 따라 다른 시점에 같은 작업

```typescript
// 초기화가 됐으면 바로, 아직이면 초기화 완료 후 실행
const initialize = () => {
  setupPlayer(videoId);
};

if (isApiReady) {
  initialize();                            // 즉시
} else {
  window.onApiReady = initialize;          // API 준비되면 실행 (참조)
}
```

```txt
함수가 "값"처럼 쓰이는 것:
  JavaScript에서 함수는 일급 객체 — 변수에 담거나, 인자로 넘기거나, 반환할 수 있음
  execute 처럼 이름을 붙여두면 여러 곳에서 참조로 전달 가능
  → [[JS_Promise]] 비동기 흐름과 같이 자주 나오는 패턴
```

---

# 화살표 함수 암시적 반환 — 중괄호 유무 ⭐️⭐️⭐️⭐️

```typescript
// 중괄호 없음 — 암시적 반환 (expression이 자동으로 return됨)
const double = (x: number) => x * 2;
//                            ↑ 자동으로 return

// 중괄호 있음 — 명시적 반환 (return 키워드 필수)
const double = (x: number) => {
  return x * 2;  // return 없으면 undefined 반환
};

// 중괄호 있는데 return 빠진 경우 ❌
const double = (x: number) => {
  x * 2;  // return 없음 → undefined 반환 (계산 결과가 버려짐)
};
```

```txt
중괄호의 의미:
  {} 없음 → "=> 뒤의 식(expression) 하나를 바로 반환"
  {} 있음 → "함수 본문 블록 시작, return을 직접 써야 값 반환"

  이 차이가 map, filter 등 콜백에서 가장 자주 문제가 됨
```

## map에서의 함정 ⭐️⭐️⭐️⭐️

```typescript
// ✅ 중괄호 없음 — upsert Promise가 자동 반환됨
await this.prisma.$transaction(
  rows.map(({ metric, count }) =>
    this.prisma.snapshot.upsert({ ... })  // 자동으로 return
  )
);
// $transaction에 Promise[]가 전달됨

// ❌ 중괄호 있는데 return 없음
await this.prisma.$transaction(
  rows.map(({ metric, count }) => {
    this.prisma.snapshot.upsert({ ... })  // return 없음 → undefined
  })
);
// $transaction에 undefined[]가 전달됨 → 오류!
```

```txt
이 실수가 발생하는 이유:
  "중괄호 = 함수 본문"이라는 건 알지만
  "중괄호 없음 = 자동 return"이라는 것을 잊어버림

  중괄호를 쓰고 싶다면 반드시 return 추가:
  rows.map(({ metric, count }) => {
    return this.prisma.snapshot.upsert({ ... });
  })
```

## 객체를 반환할 때 추가 주의 ⭐️⭐️⭐️

```typescript
// ❌ 객체 리터럴의 {}가 함수 블록으로 해석됨
const toObj = (x: number) => { value: x };
//                             ↑ 함수 블록으로 해석 → undefined 반환

// ✅ 객체를 암시적 반환하려면 소괄호로 감싸기
const toObj = (x: number) => ({ value: x });
//                             ↑ 객체 리터럴임을 명시

// 실전
items.map(item => ({
  id:    item.id,
  label: item.name,
}));
```

```txt
({ ... }) 패턴:
  소괄호로 감싸면 "이건 블록이 아니라 객체야"라고 JS에게 알림
  map, reduce에서 객체를 반환할 때 필수
```

## 한눈에 비교

```typescript
// ① 암시적 반환 — 단일 식
x => x * 2
x => fetch(url)                   // Promise 반환
x => ({ id: x.id, name: x.name }) // 객체 반환 (소괄호 필요)

// ② 명시적 반환 — 블록 본문
x => {
  const result = x * 2;
  return result;     // ← 필수
}

// ③ 흔한 실수 — return 없는 블록
x => {
  x * 2;             // ← 계산은 하지만 반환 안 함 → undefined
}
```