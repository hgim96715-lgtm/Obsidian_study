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
> DOM 측정이 필요하면 `useLayoutEffect`. React Strict Mode에서는 개발 환경에서 두 번 실행된다.

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

# cleanup 함수 ⭐️⭐️⭐️⭐️

```txt
cleanup이 실행되는 두 가지 시점:
  ① 의존성이 바뀌어서 effect가 재실행되기 직전
  ② 컴포넌트가 언마운트될 때
```

```typescript
useEffect(() => {
  const id = setInterval(() => tick(), 1000);

  return () => {
    clearInterval(id); // ① 재실행 직전 or ② 언마운트 시 취소
  };
}, []);
```

```txt
cleanup이 필요한 경우:
  이벤트 리스너 등록 → removeEventListener로 해제
  타이머 설정        → clearInterval / clearTimeout 취소
  데이터 fetch       → cancelled 플래그로 응답 무시
  WebSocket 연결     → disconnect / close

cleanup 없으면:
  컴포넌트가 사라진 뒤에도 타이머·리스너가 실행
  언마운트된 컴포넌트에 setState → React 경고 + 메모리 누수
```

---

# 자주 쓰는 패턴

## 데이터 Fetch — async/await ⭐️⭐️⭐️⭐️

```txt
핵심: .then() 방식과 async/await 방식은 동작이 완전히 같다.
문법만 다른 것 — 둘 중 하나를 고르면 됨.

useEffect에 직접 async를 못 쓰는 이유:
  async 함수는 항상 Promise를 반환
  useEffect는 cleanup 함수(또는 undefined)만 인정
  → 내부에 async function 선언 후 void로 호출이 표준
```

```typescript
// 방법 A — async/await (권장)
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

// 방법 B — Promise chain (.then)
useEffect(() => {
  let cancelled = false;

  fetchRoom(roomId)
    .then((data) => { if (!cancelled) setRoom(data); })
    .catch((err) => { if (!cancelled) setError(err.message); })
    .finally(() => { if (!cancelled) setLoading(false); });

  return () => { cancelled = true; };
}, [roomId]);
```

```txt
.then(data => ...)  = async/await에서 const data = await ...
.catch(err => ...)  = try/catch의 catch 블록
.finally(() => ...) = try/catch의 finally 블록
```

## cancelled 플래그 — Race Condition 방지 ⭐️⭐️⭐️⭐️

```txt
Race Condition(경쟁 조건):
  두 개의 비동기 요청이 "어느 것이 먼저 완료되느냐"에 따라 결과가 달라지는 버그

실제 상황 예시 — 검색창에 빠르게 타이핑
  query = "강"   → 요청 A 시작 (느린 서버, 1.2초 걸림)
  query = "강남" → 요청 B 시작 (빠른 서버, 0.3초 걸림)

  요청 B 완료 → "강남" 결과 표시 ✅
  요청 A 완료 → "강" 결과가 "강남" 결과를 덮어씀 ❌ ← Race Condition

  나중에 시작한 요청(B)이 먼저 끝났는데
  이전 요청(A)이 나중에 도착해서 화면을 덮어쓰는 버그
```

```txt
타임라인 비교

❌ cancelled 없을 때
  t=0ms  roomId='A' → 요청 A 시작
  t=100  roomId='B' → 요청 B 시작
  t=300  요청 B 응답 → setRoom(B 데이터) → 화면: B ✅
  t=800  요청 A 응답 → setRoom(A 데이터) → 화면: A ❌ (B가 덮임)

✅ cancelled 있을 때
  t=0ms  roomId='A' → cancelled_A=false, 요청 A 시작
  t=100  roomId='B' → cleanup(cancelled_A=true), cancelled_B=false, 요청 B 시작
  t=300  요청 B 응답 → if (!cancelled_B) → true  → setRoom(B 데이터) → 화면: B ✅
  t=800  요청 A 응답 → if (!cancelled_A) → false → 무시 ✅
```

```typescript
useEffect(() => {
  if (!query.trim()) { setResults([]); return; }

  let cancelled = false;        // ① 이 effect 인스턴스의 유효성 플래그

  async function search() {
    setLoading(true);
    try {
      const data = await searchPlacesRequest(token, query);
      if (!cancelled) setResults(data); // ③ 아직 유효할 때만 state 갱신
    } catch {
      if (!cancelled) setResults([]);
    } finally {
      if (!cancelled) setLoading(false);
    }
  }

  void search();
  return () => { cancelled = true; }; // ② deps 변경 or 언마운트 시 무효화
}, [query, token]);
```

```txt
cancelled가 막는 것:
  fetch 요청 자체를 취소하는 게 아님 — 네트워크 요청은 끝까지 날아감
  "이미 무효화된 effect의 응답이 state를 갱신하는 것"을 막음

cancelled 플래그 vs AbortController:
  cancelled 플래그  — 요청은 날아감, setState만 차단, 구현 단순
  AbortController  — 요청 자체 중단, 서버 부하 감소, 구현 중간
  외부 API(카카오, 구글 등) → AbortController 권장 → [[JS_Fetch_API]]
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

## 타이머 ⭐️⭐️⭐️

```typescript
// 한 번만 실행
useEffect(() => {
  const id = window.setTimeout(() => setVisible(false), 3000);
  return () => window.clearTimeout(id);
}, []);

// 반복 실행
useEffect(() => {
  const id = window.setInterval(() => {
    setCount(c => c + 1); // 함수형 업데이트 — count를 deps에 안 넣어도 됨
  }, 1000);
  return () => window.clearInterval(id);
}, []);
```

```txt
setCount(count + 1) 이면 count가 deps에 필요 → 바뀔 때마다 interval 재등록
setCount(c => c + 1) 이면 이전 값을 인자로 받아 계산 → deps 없이도 항상 최신값
```

## 디바운스 (Debounce) ⭐️⭐️⭐️⭐️

```txt
디바운스: 연속 입력 중 마지막 입력 후 N ms가 지났을 때만 실행
  검색창 타이핑 중 매 키입력마다 API 요청 방지
  setTimeout + clearTimeout(cleanup) + cancelled 플래그를 함께 사용
```

```typescript
useEffect(() => {
  const query = value.trim();
  if (query.length < 2) return;  // 최소 글자 수 미달이면 요청 안 함

  let cancelled = false;

  const timer = window.setTimeout(() => {
    async function search() {
      try {
        const results = await searchRequest(query);
        if (!cancelled) setResults(results);
      } catch {
        if (!cancelled) setResults([]);
      }
    }
    void search();
  }, 300);  // 300ms 동안 deps 변경 없으면 실행

  return () => {
    cancelled = true;
    window.clearTimeout(timer);  // 다음 effect 실행 전 이전 타이머 취소
  };
}, [value]);
```

```txt
타이머 + cancelled 플래그 둘 다 필요한 이유:
  clearTimeout — 타이머가 아직 실행 안 됐을 때 (타이핑 중 debounce)
  cancelled    — 타이머 실행 후 API 응답 대기 중에 deps가 바뀐 경우 (race condition)

타이핑 "강" → "강남" → "강남역" (300ms 내 연타):
  "강"   → timer 설정 → "강남" deps 변경 → cleanup: timer 취소 (cancelled 필요 없음, 아직 실행 전)
  "강남" → timer 설정 → "강남역" deps 변경 → cleanup: timer 취소
  "강남역" → 300ms 경과 → timer 실행 → API 요청 1건만 날아감

타이핑 "강" → 300ms 후 요청 → 타이핑 "강남" → 300ms 후 요청 (두 요청 동시 비행):
  "강" 응답이 느리면 "강남" 응답 후에 도착할 수 있음 → cancelled 플래그로 setState 차단
```

## 외부 구독 ⭐️⭐️⭐️

```typescript
// WebSocket, EventEmitter 등 외부 이벤트 구독
useEffect(() => {
  const off = onItemCreated((item) => {
    setItems(prev => [...prev, item]);
  });
  return off;  // cleanup = 구독 해제 함수
}, []);

// 브라우저 네트워크 상태 구독
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

## 외부 상태 동기화 ⭐️⭐️⭐️

```typescript
// open이 바뀔 때마다 body overflow 제어 (모달/다이얼로그 패턴)
useEffect(() => {
  document.body.style.overflow = open ? 'hidden' : '';
  return () => { document.body.style.overflow = ''; };
}, [open]);
```

---

# 의존성 — 무엇을 넣어야 하는가 ⭐️⭐️⭐️⭐️

```txt
규칙: useEffect 안에서 쓰는 값은 전부 의존성에 넣어야 함

넣지 않아도 되는 것:
  setState 함수 — React가 안정적 참조 보장
  useRef.current — ref 자체는 렌더 간 동일 참조
  외부 상수 (모듈 스코프 변수)
  순수 함수 (매 렌더마다 같은 결과)

넣어야 하는 것:
  props (userId, open 등)
  state (count, items 등)
  컴포넌트 안에서 선언된 변수·함수
```

## 함수를 의존성에 넣으면 무한루프 ⭐️⭐️⭐️⭐️

```typescript
// ❌ 무한루프
function Component() {
  const fetchData = async () => {      // 매 렌더마다 새 함수 객체 생성
    const data = await api.get('/items');
    setItems(data);
  };

  useEffect(() => {
    void fetchData();
  }, [fetchData]);   // fetchData가 매 렌더마다 새 참조 → 무한 재실행
}

// ✅ 방법 1 — useEffect 안에서 함수 정의 (가장 단순)
useEffect(() => {
  async function fetchData() { ... }
  void fetchData();
}, [userId]);

// ✅ 방법 2 — useCallback으로 참조 안정화
const fetchData = useCallback(async () => {
  const data = await api.get(`/items/${userId}`);
  setItems(data);
}, [userId]);   // userId가 바뀔 때만 새 함수

useEffect(() => {
  void fetchData();
}, [fetchData]);
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

// ❌ props가 바뀔 때 state 초기화 — 불필요한 렌더 발생
useEffect(() => {
  setComment('');
}, [postId]);

// ✅ key prop으로 컴포넌트 리셋
<CommentForm key={postId} postId={postId} />
```

```txt
useEffect를 쓰기 전에:
  "이걸 렌더링 중에 계산하거나 이벤트 핸들러에서 할 수 없나?" 먼저 고려

useEffect가 필요 없는 대표적인 경우:
  렌더링 중 계산할 수 있는 값 → 변수로
  props에서 파생되는 state → 렌더링 중 계산
  이벤트 핸들러에서 할 수 있는 것 → 이벤트 핸들러로
```

---

# useLayoutEffect — DOM 측정 후 즉시 ⭐️⭐️⭐️

```typescript
// useEffect:       화면에 그린 뒤 실행 → 위치 잘못된 채로 잠깐 보일 수 있음
// useLayoutEffect: DOM 업데이트 후, 화면에 그리기 전 실행 → 깜빡임 없음

useLayoutEffect(() => {
  const rect = ref.current?.getBoundingClientRect();
  setPos({ top: rect.top, left: rect.left });
}, [open]);
```

```txt
useLayoutEffect를 쓰는 경우:
  DOM 측정이 필요한 경우 (getBoundingClientRect, offsetHeight 등)
  측정 결과로 즉시 위치·크기를 업데이트해야 할 때 (드롭다운, 툴팁)

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
