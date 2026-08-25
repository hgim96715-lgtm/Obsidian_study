---
aliases:
  - narrowing
  - async
tags:
  - TypeScript
related:
  - "[[00_JS_Ecosystem_HomePage]]"
  - "[[TS_Type_Guards]]"
---

# TS_AsyncNarrowing — async 경계에서 narrowing 풀림

>[!info]
>TypeScript의 타입 좁힘(Narrowing)은 동기 제어 흐름 안에서만 보장됨.
>async function · 클로저 내부에서 외부 `let` / React state 변수는 다시 `T | null`로 되돌아갈 수 있음.
>해결책: null 검사 직후 const로 값을 캡처 → async 내부에서 타입 안전하게 사용.
> `user!` 단언과의 비교 → 이 노트. 타입 가드 패턴 전체 → [[TS_Type_Guards]]

---

# 왜 async 경계에서 narrowing이 풀리는가

```typescript
// user: User | null  (React state 또는 외부 let 변수)

useEffect(() => {
  if (!user) return;
  // ✅ 여기서는 user: User (좁혀짐)

  async function load() {
    const data = await fetchSomething();
    console.log(user.id); // ⚠️ TS: user는 여전히 User | null
  }

  load();
}, [user]);
```

```txt
왜 풀리는가:

  TypeScript 제어 흐름 분석(Control Flow Analysis)은 동기 경로만 추적
  → async function은 나중에 실행됨 (이벤트 루프 재개 후)
  → 그 사이에 user가 null로 바뀌었을 수 있음

  let / 외부 참조 변수:
    재할당 가능 → TypeScript가 "지금 이 순간에만 User"라고 보장했을 뿐
    async 내부에서도 User라고 보장 못 함

  React state:
    컴포넌트 리렌더 → user 새 값으로 교체
    async 함수 안에서 캡처한 user는 이전 렌더의 값
    → 타입뿐 아니라 런타임 값도 바뀔 수 있음
```

> [!warning] user! 단언이 위험한 이유
> `user!`는 TypeScript에게 "null이 아님을 믿어"라고 강제하는 것
> 컴파일러를 침묵시킬 뿐 — 런타임 보장 없음
> async 재개 시점에 user가 실제로 null이면 `TypeError: Cannot read properties of null` 발생

---

# 해결 — null 검사 직후 const로 캡처

```typescript
useEffect(() => {
  if (!accessToken || !user) return;

  // ✅ 좁혀진 시점에 값을 const로 고정
  const token  = accessToken;   // string (null 없음)
  const userId = user.id;       // string (null 없음)

  let cancelled = false;

  async function load() {
    const data = await fetchSomething(token); // token: string ✅

    if (cancelled) return;

    doSomethingWith(userId); // userId: string ✅ — null 체크 불필요
  }

  load();
  return () => { cancelled = true; };
}, [accessToken, user]);
```

```txt
왜 const 캡처가 안전한가:

  1. 타입 확정
     if (!user) return 직후 → user: User (좁혀진 상태)
     const userId = user.id → userId: string (타입 확정, 이후 변하지 않음)
     TypeScript가 userId를 항상 string으로 인식

  2. 값 고정 (스냅샷)
     async 실행 중 user 상태가 null로 바뀌어도
     → userId는 이미 string 값을 담고 있음
     → 항상 검사 시점의 값 사용

  3. 최소 캡처 원칙
     user 객체 전체를 캡처하지 않고 필요한 필드만 (user.id)
     → user 객체의 나머지 상태 변화에 영향 없음
```

---

# user! 단언 vs const 캡처 비교

```typescript
// ❌ user! 단언 — 위험
async function load() {
  const data = await fetch(...);
  localStorage.setItem(key, user!.id); // 런타임 null 가능
}

// ✅ const 캡처 — 안전
const userId = user.id; // 이 시점에 user는 반드시 non-null
async function load() {
  const data = await fetch(...);
  localStorage.setItem(key, userId);   // 항상 string
}
```

| | `user!` 단언 | `const userId` 캡처 |
|---|---|---|
| TypeScript | 컴파일 통과 (강제) | 타입 자연스럽게 string |
| 런타임 안전 | ❌ null 가능 | ✅ 항상 값 있음 |
| async 이후 | user 상태 변화 반영됨 | 검사 시점 값 유지 |
| 의도 명확성 | 불명확 | "이 시점의 값" 명시적 |

---

# cancelled 플래그 패턴 — async 정리

```typescript
useEffect(() => {
  if (!user) return;

  const userId = user.id;    // 값 캡처
  let cancelled = false;     // 언마운트/재실행 감지 플래그

  async function load() {
    try {
      const data = await fetchData(userId);

      if (cancelled) return; // 이미 클린업됐으면 상태 업데이트 중단
      // → 언마운트된 컴포넌트에 setState 호출 방지

      setState(data);
    } catch (err) {
      if (!cancelled) setError(err);
    }
  }

  load();

  return () => {
    cancelled = true; // 클린업: 컴포넌트 언마운트 or deps 바뀜
  };
}, [user]);
```

```txt
cancelled 플래그가 필요한 이유:

  useEffect deps(user)가 바뀌면 → cleanup 실행 → 새 effect 실행
  이전 effect의 async load()가 아직 실행 중이면
  → cancelled = true 로 결과를 무시하게 함

  AbortController와 차이:
    AbortController → fetch 자체를 취소 (네트워크 요청 중단)
    cancelled 플래그 → fetch는 완료됐지만 결과를 버림
    → 둘을 같이 쓰는 것이 가장 엄격한 패턴
```

---

# 언제 캡처가 필요한가 — 판단 기준

```txt
캡처 필요:
  ✅ async function 내부에서 외부 nullable 변수 사용
  ✅ setTimeout · setInterval 콜백 내부
  ✅ Promise.then() 체인 내부
  ✅ 이벤트 핸들러에서 나중에 실행되는 콜백

캡처 불필요:
  ✅ 동기 코드에서 바로 사용 (narrowing 유효)
  ✅ const 로 선언된 변수 (재할당 불가 → TypeScript가 narrowing 유지)
```

```typescript
// const 변수는 TypeScript가 async 내부에서도 narrowing 유지
const config = getConfig(); // Config | null
if (!config) return;

async function run() {
  config.url; // ✅ TypeScript: config는 Config (const라 재할당 불가)
}
// → config가 const라면 별도 캡처 없이도 narrowing 유지됨
// → React state처럼 렌더마다 새로 바뀌는 경우에만 캡처 필요
```