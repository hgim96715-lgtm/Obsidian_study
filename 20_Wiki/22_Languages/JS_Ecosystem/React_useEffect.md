---
aliases:
  - useEffect
tags:
  - React
related:
  - "[[00_JS_Ecosystem_HomePage]]"
  - "[[React_useMemo_useCallback]]"
  - "[[React_useRef]]"
  - "[[React_AsyncUI]]"
---
# React_useEffect — 사이드 이펙트

>[!info]
>`useEffect` = 컴포넌트가 외부 시스템과 동기화하는 훅.
> 데이터 fetch, 이벤트 리스너, 타이머, WebSocket 구독. cleanup 함수로 이전 것을 정리한다.
>  DOM 측정이 필요하면 `useLayoutEffect`. React Strict Mode에서는 개발 환경에서 두 번 실행된다.

---

# useEffect란 ⭐️⭐️⭐️⭐️

```txt
React 컴포넌트의 역할: 데이터 → UI 렌더링
useEffect의 역할: 렌더링 이후 외부 세계와 동기화

  fetch('/api/data')         → 서버와 동기화
  addEventListener(...)      → 브라우저 이벤트 시스템과 동기화
  setInterval(...)           → 타이머 시스템과 동기화

이런 것들은 렌더링 자체와 다른 "부수 효과(Side Effect)"
→ useEffect 안에서 처리
```

---

# 기본 구조 ⭐️⭐️⭐️⭐️

```typescript
useEffect(
  () => {
    // 실행할 코드 (렌더링 후 실행)

    return () => {
      // cleanup 함수 (선택) — 다음 실행 전 또는 언마운트 시 실행
    };
  },
  [의존성], // 언제 실행할지 제어
);
```

## 의존성 배열 3가지 ⭐️⭐️⭐️⭐️

|형태|실행 시점|
|---|---|
|`useEffect(fn, [])`|마운트 시 딱 1번|
|`useEffect(fn, [a, b])`|마운트 1번 + a 또는 b가 바뀔 때마다|
|`useEffect(fn)`|매 렌더마다 — 의도적으로 쓸 일 거의 없음|

```txt
의존성 배열 = "이게 바뀌면 다시 실행해"
  [] → "처음 한 번만" — 데이터 최초 로딩
  [id] → "id가 바뀔 때마다" — 특정 리소스 fetch
  [open] → "open 상태가 바뀔 때마다"

의존성을 빠뜨리면:
  오래된 값(stale closure)으로 실행될 수 있음
  ESLint exhaustive-deps 규칙이 경고해줌
```

---

# 자주 쓰는 패턴

## 데이터 Fetch — .then vs async/await ⭐️⭐️⭐️⭐️

```txt
핵심: .then() 방식과 async/await 방식은 동작이 완전히 같다.
문법만 다른 것 — 둘 중 하나를 고르면 됨.
```

```typescript
// 방법 A — Promise chain (.then)
useEffect(() => {
  let cancelled = false;

  fetchRoom(roomId)
    .then((data) => {
      if (!cancelled) setRoom(data);
    })
    .catch((err) => {
      if (!cancelled) setError(err.message);
    })
    .finally(() => {
      if (!cancelled) setLoading(false);
    });

  return () => { cancelled = true; };
}, [roomId]);


// 방법 B — async/await (내부 함수)
useEffect(() => {
  let cancelled = false;

  async function load() {
    try {
      const data = await fetchRoom(roomId);
      if (!cancelled) setRoom(data);
    } catch (err) {
      if (!cancelled) setError(err.message);
    } finally {
      if (!cancelled) setLoading(false);
    }
  }

  void load();
  return () => { cancelled = true; };
}, [roomId]);
```

```txt
.then(data => ...)  = async/await에서 const data = await ...
.catch(err => ...)  = try/catch의 catch 블록
.finally(() => ...) = try/catch의 finally 블록

useEffect에 직접 async를 못 쓰는 이유:
  async 함수는 항상 Promise를 반환
  useEffect는 cleanup 함수(또는 undefined)만 인정
  → 내부에 async function 선언 후 void로 호출이 표준

취향 차이:
  .then()   → 중첩 없이 평탄한 구조
  async/await → try/catch 구조가 더 익숙한 경우
```

## cancelled 플래그 ⭐️⭐️⭐️⭐️

```typescript
useEffect(() => {
  let cancelled = false;          // ① 이 effect 인스턴스의 유효성 플래그

  fetchRoom(roomId).then((data) => {
    if (!cancelled) setRoom(data); // ③ 아직 유효할 때만 state 갱신
  });

  return () => { cancelled = true; }; // ② 언마운트 or deps 변경 시 무효화
}, [roomId]);
```

```txt
cancelled가 막는 것:
  fetch를 멈추는 게 아님 — 요청은 끝까지 날아감
  "이미 무효화된 effect의 응답이 setState하는 것"을 막음

왜 필요한가 (race condition):
  roomId = 'A' → fetch 시작
  roomId = 'B' 로 바뀜 → cleanup(cancelled=true) → 새 effect 시작
  'A' 응답이 나중에 도착 → if (!cancelled) → cancelled=true → 무시 ✅

  이 체크가 없으면:
  현재 화면은 'B'인데 'A'의 데이터가 덮어쓰는 경쟁 조건 발생

비동기 패턴 전체 → [[React_AsyncUI]]
```

```typescript
useEffect(() => {
  let cancelled = false;   // 클린업 플래그

  async function load() {
    try {
      const data = await fetchUser(userId);
      if (!cancelled) setUser(data);   // 언마운트 후면 setState 안 함
    } catch (err) {
      if (!cancelled) setError(err);
    }
  }

  void load();

  return () => { cancelled = true; };  // 정리
}, [userId]);   // userId가 바뀔 때마다 재실행
```

```txt
cancelled 플래그가 왜 필요한가:
  userId가 빠르게 바뀌면 이전 요청이 나중에 완료될 수 있음
  이미 다른 userId로 바뀐 상태인데 이전 응답으로 setState → 잘못된 데이터 표시
  cancelled = true → 응답이 와도 setState 안 함

async 함수를 직접 useEffect에 못 쓰는 이유:
  useEffect 콜백은 cleanup 함수(동기)를 반환해야 함
  async function은 항상 Promise를 반환 → 타입 불일치
  → 내부에 async function 선언 후 즉시 호출 (void load())
```

## 이벤트 리스너 ⭐️⭐️⭐️⭐️

```typescript
useEffect(() => {
  if (!open) return;   // 조건부 등록

  const onKeyDown = (e: KeyboardEvent) => {
    if (e.key === 'Escape') onClose();
  };

  window.addEventListener('keydown', onKeyDown);
  return () => window.removeEventListener('keydown', onKeyDown);
  // ↑ 반드시 해제 — 안 하면 언마운트 후에도 리스너 남아있음

}, [open, onClose]);
```

## 외부 상태 동기화 ⭐️⭐️⭐️

```typescript
// open이 바뀔 때마다 body overflow 제어
useEffect(() => {
  document.body.style.overflow = open ? 'hidden' : '';
  return () => { document.body.style.overflow = ''; };  // 정리
}, [open]);
```

## 타이머 ⭐️⭐️⭐️

```typescript
useEffect(() => {
  const id = window.setTimeout(() => {
    setVisible(false);
  }, 3000);

  return () => window.clearTimeout(id);  // 언마운트 시 타이머 취소
}, []);
```

---

# cleanup 함수 ⭐️⭐️⭐️⭐️

```typescript
useEffect(() => {
  // 실행
  const id = setInterval(() => tick(), 1000);

  return () => {
    // cleanup — 다음 두 가지 경우에 실행:
    //   ① 의존성이 바뀌어서 effect가 재실행되기 전
    //   ② 컴포넌트가 언마운트될 때
    clearInterval(id);
  };
}, []);
```

```txt
cleanup이 필요한 경우:
  이벤트 리스너 등록 → 해제
  타이머 설정 → 취소
  데이터 fetch → 응답 무시 (cancelled 플래그)
  WebSocket 연결 → 해제

cleanup 없으면:
  컴포넌트가 사라진 뒤에도 타이머·리스너가 실행
  언마운트된 컴포넌트에 setState → React 경고 + 메모리 누수
```

---

# 의존성 — 무엇을 넣어야 하는가 ⭐️⭐️⭐️⭐️

```typescript
// 규칙: useEffect 안에서 쓰는 값은 전부 의존성에 넣어야 함
useEffect(() => {
  const data = transform(input, config);
  setResult(data);
}, [input, config]);  // transform은? → 외부 함수면 안정적이라 생략 가능
```

```txt
의존성에 넣지 않아도 되는 것:
  setState 함수 — React가 보장하는 안정적 참조
  useRef.current — ref는 안정적 (current가 바뀌어도 ref 자체는 안 바뀜)
  외부 상수 (모듈 스코프 변수)
  순수 함수 (매 렌더마다 같은 결과)

의존성에 넣어야 하는 것:
  props (userId, open 등)
  state (count, items 등)
  컴포넌트 안에서 선언된 변수/함수
```

## 함수를 의존성에 넣으면 무한루프 ⭐️⭐️⭐️⭐️

```typescript
// ❌ 무한루프
function Component() {
  const fetchData = async () => {      // 매 렌더마다 새 함수 생성
    const data = await api.get('/items');
    setItems(data);
  };

  useEffect(() => {
    void fetchData();
  }, [fetchData]);   // fetchData가 매 렌더마다 새 참조 → 무한 재실행
}

// ✅ 방법 1 — useEffect 안에서 함수 정의
useEffect(() => {
  async function fetchData() { ... }
  void fetchData();
}, [userId]);  // 함수를 의존성에 안 넣어도 됨

// ✅ 방법 2 — useCallback으로 참조 안정화
const fetchData = useCallback(async () => {
  const data = await api.get(`/items/${userId}`);
  setItems(data);
}, [userId]);   // userId가 바뀔 때만 새 함수

useEffect(() => {
  void fetchData();
}, [fetchData]);   // 이제 userId가 바뀔 때만 실행
```

---

# useEffect가 필요 없는 경우 ⭐️⭐️⭐️

```typescript
// ❌ 렌더링 중 계산 가능한 것을 useEffect로 하면 안 됨
useEffect(() => {
  setFullName(`${firstName} ${lastName}`);
}, [firstName, lastName]);

// ✅ 그냥 렌더링 중에 계산
const fullName = `${firstName} ${lastName}`;

// ❌ props가 바뀔 때 state 초기화 — useEffect로 하면 불필요한 렌더 발생
useEffect(() => {
  setComment('');
}, [postId]);

// ✅ key prop으로 컴포넌트 리셋
<CommentForm key={postId} postId={postId} />
```

```txt
useEffect가 필요 없는 대표적인 경우:
  렌더링 중 계산할 수 있는 값 → 변수로
  props에서 파생되는 state → 렌더링 중 계산
  이벤트 핸들러에서 할 수 있는 것 → 이벤트 핸들러로

useEffect를 쓰기 전에:
  "이걸 렌더링 중에 계산하거나 이벤트 핸들러에서 할 수 없나?" 먼저 고려
```

---

# setInterval — 주기적 실행 ⭐️⭐️⭐️

```typescript
useEffect(() => {
  const id = window.setInterval(() => {
    setCount(c => c + 1);   // 함수형 업데이트로 deps 불필요
  }, 1000);

  return () => window.clearInterval(id);  // 반드시 cleanup
}, []);  // 1회 등록
```

```txt
setInterval cleanup이 빠지면:
  컴포넌트가 언마운트되어도 interval이 계속 실행
  언마운트된 컴포넌트에 setState → "Can't perform a React state update on an unmounted component"

setCount(c => c + 1) — 함수형 업데이트:
  setCount(count + 1) 이면 count가 deps에 필요 → 바뀔 때마다 interval 재등록
  setCount(c => c + 1) 이면 이전 값을 받아서 계산 → deps 없이도 항상 최신값
```

---

# useLayoutEffect — DOM 측정 후 즉시 ⭐️⭐️⭐️

```typescript
// useEffect: 화면에 그린 뒤 실행 → 위치가 잘못된 상태로 잠깐 보일 수 있음
// useLayoutEffect: DOM 업데이트 후, 화면에 그리기 전 실행 → 깜빡임 없음

useLayoutEffect(() => {
  // DOM 측정 (getBoundingClientRect 등)
  const rect = ref.current?.getBoundingClientRect();
  setPos({ top: rect.top, left: rect.left });
}, [open]);
```

```txt
useLayoutEffect를 쓰는 경우:
  DOM 측정이 필요한 경우 (getBoundingClientRect, offsetHeight 등)
  측정 결과로 즉시 위치/크기를 업데이트해야 할 때 (드롭다운, 툴팁)
  → 화면에 그려지기 전에 계산 완료 → 깜빡임 없음

useEffect를 쓰는 경우:
  데이터 fetch, 이벤트 리스너 등록
  DOM 측정이 필요 없는 대부분의 경우

SSR(Next.js) 주의:
  useLayoutEffect는 서버에서 실행되지 않음
  서버에서 경고 발생 → suppressHydrationWarning 또는 조건부 처리
```

---

# React Strict Mode — 개발 환경에서 두 번 실행 ⭐️⭐️⭐️

```typescript
// 개발 환경에서 useEffect가 두 번 실행되는 것처럼 보임
useEffect(() => {
  console.log('실행');   // 개발에서 두 번 출력됨
  return () => {
    console.log('cleanup');  // 첫 번째 실행 후 cleanup → 두 번째 실행
  };
}, []);
```

```txt
왜 두 번 실행하는가:
  React Strict Mode가 cleanup이 제대로 작동하는지 검증
  마운트 → cleanup → 마운트 순서로 두 번 실행

  실제 프로덕션 빌드: 한 번만 실행
  개발에서 두 번 실행은 cleanup 누락 버그를 찾아주는 것

두 번 실행되면 안 되는 코드 → cleanup에서 취소 처리가 있어야 함:
  fetch → cancelled 플래그
  addEventListener → removeEventListener
  setInterval → clearInterval
```

---

# 외부 구독 패턴 ⭐️⭐️⭐️

```typescript
// WebSocket, EventEmitter 등 외부 이벤트 구독
useEffect(() => {
  const off = onItemCreated((item) => {
    setItems(prev => [...prev, item]);
  });
  return off;  // cleanup = 구독 해제 함수
}, []);

// 브라우저 이벤트 구독
useEffect(() => {
  const handler = () => setOnline(navigator.onLine);
  window.addEventListener('online',  handler);
  window.addEventListener('offline', handler);
  return () => {
    window.removeEventListener('online',  handler);
    window.removeEventListener('offline', handler);
  };
}, []);
```