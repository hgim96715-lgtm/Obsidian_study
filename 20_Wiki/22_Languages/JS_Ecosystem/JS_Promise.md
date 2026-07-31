---
aliases:
  - async/await
  - Promise
  - Promise.reject
  - Promise.resolve
  - then
  - Promise.all
  - void
  - unknown
tags:
  - JavaScript
related:
  - "[[00_JS_Ecosystem_HomePage]]"
  - "[[TS_Generics]]"
  - "[[React_AsyncUI]]"
  - "[[JS_Operators]]"
---
# JS_Promise — 비동기 처리

> [!info] 
> Promise = 지금은 모르지만 나중에 결과가 나올 값을 표현하는 객체. 
> async/await는 이것을 더 읽기 쉽게 만든 문법. 
> 대부분은 async/await로 충분하고, new Promise()는 콜백 기반 API를 래핑할 때만 필요.

---

# Promise란 — 왜 필요한가 ⭐️⭐️⭐️⭐️

```txt
JavaScript는 기본적으로 한 번에 한 가지 일만 처리함 (싱글 스레드)
네트워크 요청처럼 "결과가 나올 때까지 기다려야 하는 일"이 문제

기다리는 동안 아무것도 못 하면:
  fetch('/api/data') → 응답 올 때까지 화면이 굳어버림 → 사용자가 아무것도 못 함

해결:
  "지금 당장 결과를 줄 순 없는데, 나중에 완료되면 알려줄게"
  = Promise 객체를 즉시 반환 → JS가 다른 일을 계속 함 → 완료되면 결과 전달
```

## 세 가지 상태

```txt
pending    아직 결과가 안 나온 상태 (요청 중)
fulfilled  성공 — 결과값을 가짐
rejected   실패 — 에러를 가짐

한 번 fulfilled나 rejected가 되면 다시 안 바뀜
```

```javascript
const promise = fetch('/api/data');
// 이 시점: pending (응답 안 왔으니까)
// 나중에:  fulfilled (응답 옴) 또는 rejected (에러)
```

---

# async/await — 일상적인 사용 ⭐️⭐️⭐️⭐️

```txt
fetch, Prisma, axios 같은 API는 이미 Promise를 반환함
→ 직접 Promise를 만들 필요 없이 async/await로 쓰면 됨
→ 이게 일상적인 대부분의 경우
```

```typescript
async function getUser(id: string) {
  try {
    const user = await this.prisma.user.findUnique({ where: { id } }); // pending → fulfilled
    return user;
  } catch (err) {
    // rejected → 여기 들어옴
    throw new NotFoundException('유저를 찾을 수 없습니다.');
  } finally {
    // 성공/실패 무관하게 항상 실행
    console.log('완료');
  }
}
```

```txt
await:
  Promise가 fulfilled/rejected될 때까지 기다렸다가 결과값을 꺼내옴
  async 함수 안에서만 사용 가능

async 함수:
  항상 Promise를 반환함
  return user → 자동으로 Promise.resolve(user)로 감싸짐
  throw error → 자동으로 Promise.reject(error)로 변환됨
```

## .then()/.catch() — async/await의 원형

```typescript
// 아래 둘은 완전히 같은 동작 — 문법만 다름
fetch('/api/data')
  .then(res => res.json())
  .then(data => setData(data))
  .catch(err => setError(err));

// ↕ 동일

async function load() {
  try {
    const res  = await fetch('/api/data');
    const data = await res.json();
    setData(data);
  } catch (err) {
    setError(err);
  }
}
```

```txt
지금은 async/await가 훨씬 더 읽기 쉬워서 기본값으로 사용됨
.then() 방식은 useEffect 안에서 cancelled 플래그와 함께 쓸 때 간결한 경우가 있음
→ [[React_AsyncUI]] useEffect 섹션 참고
```

---

# `Promise<T>` 타입 — 무엇을 넣어야 하는가 ⭐️⭐️⭐️⭐️

```typescript
// Promise<T> = T 타입의 값으로 resolve되는 Promise

async function getUser(): Promise<User> { ... }         // User 객체로 resolve
async function deletePost(): Promise<void> { ... }      // 값 없이 resolve
async function doSomething(): Promise<unknown> { ... }  // 어떤 타입이든 가능
```

## void vs unknown vs T — 언제 뭘 쓰는가

|타입|의미|언제|
|---|---|---|
|`Promise<void>`|반환값이 없거나 무시됨|삭제·업데이트처럼 결과 값이 필요 없을 때|
|`Promise<T>`|T 타입의 값으로 resolve|결과를 받아서 써야 할 때 (User, Post 등)|
|`Promise<unknown>`|어떤 타입이든 받음|어떤 함수든 넣을 수 있게 할 때 (래퍼 패턴)|

```typescript
async function deleteComment(id: number): Promise<void> {
  await this.prisma.comment.delete({ where: { id } });
  // 반환값 없음
}

async function createComment(dto: CreateDto): Promise<Comment> {
  return this.prisma.comment.create({ data: dto });
  // Comment 객체 반환
}
```

## `action: () => Promise<unknown> `— 래퍼 패턴의 핵심

```typescript
// 다양한 API 함수들
const fetchUser   = (): Promise<User>            => api.get('/user');
const deletePost  = (): Promise<void>            => api.delete('/post');
const likePost    = (): Promise<{ liked: boolean }> => api.post('/like');

// action 타입이 () => Promise<void>이면:
run(id, fetchUser)  // ❌ Promise<User>는 Promise<void>에 대입 안 됨
run(id, deletePost) // ✅
run(id, likePost)   // ❌ Promise<{liked:boolean}>은 Promise<void>에 대입 안 됨

// action 타입이 () => Promise<unknown>이면:
run(id, fetchUser)  // ✅ unknown은 어떤 타입이든 받음
run(id, deletePost) // ✅
run(id, likePost)   // ✅ 전부 통과
```

```txt
unknown의 의미:
  "어떤 타입이든 받겠다, 대신 실제로 그 값을 쓰려면 타입을 좁혀야 함"
  → 래퍼 안에서 await action()의 결과를 사용하지 않을 때 가장 적합
  → any보다 안전 (any는 타입 체크를 완전히 끄지만, unknown은 사용 전 좁혀야 함)

void와 unknown의 차이:
  void  → "이 함수는 의미 있는 값을 반환하지 않는다"
  unknown → "이 함수가 뭘 반환하든 받겠다, 대신 쓰지 않을 것"

래퍼에서 결과를 쓰려면:
  Promise<unknown>이면 await action()의 결과가 unknown 타입
  → 그대로 사용 불가, 타입 단언이나 가드 필요
  → 결과를 쓰고 싶으면 제네릭으로 받는 게 나음 (아래 "반환값이 있는 버전" 참고)
```

---

# async 래퍼 패턴 — 언제 만드는가 ⭐️⭐️⭐️⭐️

```txt
언제 필요한가:
  버튼이 여러 개인데 각 버튼마다 이 구조가 반복될 때

  버튼 A:             버튼 B:             버튼 C:
  setBusyId(id)       setBusyId(id)       setBusyId(id)
  setError('')        setError('')        setError('')
  try {               try {               try {
    await acceptRequest()  await declineRequest()  await blockUser()
    await load()       await load()         await load()
  } catch ...         } catch ...         } catch ...
  finally {           finally {           finally {
    setBusyId(null)     setBusyId(null)     setBusyId(null)
  }                   }                   }

  다른 건 await ~~() 한 줄뿐 — 나머지 8~10줄이 전부 반복
  → 공통 부분을 래퍼로 추출, "무엇을 할지"만 인자로 받음
```

```typescript
// 실전 예시 — DM 목록 페이지
async function run(dmId: string, action: () => Promise<unknown>) {
  setBusyId(dmId);      // 이 DM 항목에 로딩 표시
  setError('');
  try {
    await action();     // 실제 작업 (수락 / 거절 / 차단 등)
    await load();       // 공통 후처리: 목록 새로고침
  } catch (error) {
    setError(error instanceof Error ? error.message : '요청을 처리하지 못했어요.');
  } finally {
    setBusyId(null);    // 로딩 표시 해제
  }
}

// 사용 — 각 버튼은 "무엇을 할지"만 전달
<button onClick={() => void run(dm.id, () => acceptDm(dm.id))}>수락</button>
<button onClick={() => void run(dm.id, () => declineDm(dm.id))}>거절</button>
<button onClick={() => void run(dm.id, () => blockUser(dm.userId))}>차단</button>
```

```txt
() => acceptDm(dm.id) 로 감싸는 이유:
  acceptDm(dm.id) 라고 쓰면 그 자리에서 즉시 실행됨
  () => acceptDm(dm.id) 라고 쓰면 "나중에 호출될 함수"가 됨
  → run 안의 await action() 이 실행될 때 비로소 acceptDm이 호출됨

void run(...) 앞의 void:
  run()은 async 함수 → Promise를 반환
  onClick은 그 Promise를 처리하지 않음 → floating promise 경고
  void로 "의도적으로 버린다"고 명시 → 경고 없음
  → [[JS_Operators]] void 섹션 참고
```

## 반환값이 있는 버전 — 제네릭

```typescript
// 결과를 받아서 쓰고 싶을 때 — 제네릭으로 타입 전달
async function run<T>(action: () => Promise<T>): Promise<T | null> {
  setLoading(true);
  setError('');
  try {
    return await action();  // T 타입의 결과 반환
  } catch (err) {
    setError(err instanceof Error ? err.message : '오류');
    return null;
  } finally {
    setLoading(false);
  }
}

// 사용
const post = await run(() => fetchPost(id));  // post: Post | null
```

```txt
fn: () => Promise<unknown>   → 결과를 쓰지 않을 때
fn: () => Promise<T>         → 결과를 받아서 써야 할 때

제네릭 <T> 문법 → [[TS_Generics]] 참고
```

## 조건부 early return — Promise.resolve()

```typescript
void run(dm.id, () => {
  if (!canAccept) return Promise.resolve(); // 조건 불만족 → 아무것도 안 함
  return acceptDm(dm.id);
});
```

```txt
fn: () => Promise<unknown> 타입이라 모든 코드 경로에서 Promise를 반환해야 함

  if (!canAccept) return;
    → undefined 반환 → 타입 불일치 → TS 에러

  if (!canAccept) return Promise.resolve();
    → 즉시 fulfilled인 Promise 반환
    → await action() 이 즉시 완료 → 이후 load() 등 공통 로직은 그대로 실행

  "load()까지 포함해서 아무것도 안 하고 싶다"면:
    void run(dm.id, ...) 호출 자체를 조건으로 감쌀 것
    if (canAccept) void run(dm.id, () => acceptDm(dm.id));
```

---

# new Promise() — 언제 직접 만드는가 ⭐️⭐️⭐️

```txt
이미 Promise를 반환하는 함수(fetch, prisma, axios)는 → async/await로 충분
콜백 기반 API(setTimeout, 이벤트, 구형 라이브러리)는 → new Promise()로 래핑

판단 기준: 이 API가 Promise를 반환하는가?
  반환함 → async/await로 바로 사용
  안 함  → new Promise()로 감싸서 Promise로 만들기
```

```javascript
// new Promise() 기본 구조
const promise = new Promise((resolve, reject) => {
  // executor: 즉시 실행됨
  // 성공 시: resolve(값) 호출 → fulfilled
  // 실패 시: reject(에러) 호출 → rejected
});
```

## 가장 기본적인 예시 — setTimeout 래핑

```javascript
function delay(ms: number): Promise<void> {
  return new Promise((resolve) => {
    setTimeout(() => resolve(), ms);
  });
}

await delay(1000);
console.log('1초 후');
```

## 스크립트 로드 래핑

```typescript
function loadScript(src: string): Promise<void> {
  if (document.querySelector(`script[src="${src}"]`)) {
    return Promise.resolve(); // 이미 있으면 즉시 완료
  }
  return new Promise((resolve, reject) => {
    const script = document.createElement('script');
    script.src     = src;
    script.onload  = () => resolve();
    script.onerror = () => reject(new Error(`로드 실패: ${src}`));
    document.body.appendChild(script);
  });
}

await loadScript('https://www.youtube.com/iframe_api');
```

## 흔한 실수 — 불필요한 래핑

```typescript
// ❌ 이미 Promise를 반환하는 함수를 또 new Promise로 감쌈
return new Promise(async (resolve) => {
  const data = await fetchData();
  resolve(data);
});

// ✅ 그냥 async/await
return await fetchData();
```

---

# Promise.all — 동시에 실행 ⭐️⭐️⭐️⭐️

```typescript
// 받은 요청 · 보낸 요청 — 서로 결과 안 보고 각자 조회 → 동시에 돌려도 됨
const [received, sent] = await Promise.all([
  this.prisma.friendship.findMany({ where: { addresseeId: userId } }),
  this.prisma.friendship.findMany({ where: { requesterId: userId } }),
]);
```

```txt
직렬 (순서대로):
  받은 조회 완료 → 보낸 조회 완료
  총 시간 = A + B

병렬 Promise.all:
  받은 조회 ──┐
  보낸 조회 ──┴── 둘 다 완료
  총 시간 = max(A, B)

언제 Promise.all을 쓰는가:
  다음 작업이 이전 작업의 결과를 필요로 하지 않을 때
  서로 독립적인 여러 요청을 동시에 보낼 때

언제 순서대로 await를 쓰는가:
  B가 A의 결과를 필요로 할 때
  const user = await getUser(id);
  const posts = await getPostsByUser(user.nickname); // user 결과 필요
```

## ⚠️ 하나라도 실패하면 전체 실패

```txt
Promise.all은 하나라도 reject되면 즉시 전체 reject
다른 것들이 이미 성공했어도 그 결과는 버려짐

일부 실패를 허용하고 싶으면:
  Promise.allSettled → 성공/실패 무관하게 전부 기다린 뒤 각각의 결과를 배열로 반환
```

## 배열 구조분해

```typescript
// Promise.all 반환: Promise<[T1, T2, ...]>
// 넣은 순서 = 꺼내는 순서 (먼저 끝난 순서 아님)

const [friends, requests] = await Promise.all([
  fetchFriends(),   // → friends
  fetchRequests(),  // → requests
]);
```

---

# Promise 메서드 비교

|메서드|동작|실패 처리|
|---|---|---|
|`Promise.all([...])`|전부 완료될 때까지 기다림|하나라도 실패 → 즉시 전체 실패|
|`Promise.allSettled([...])`|전부 완료될 때까지 기다림|성공/실패 무관하게 전부 결과 수집|
|`Promise.race([...])`|가장 먼저 끝난 것(성공/실패 무관) 하나만|가장 빠른 것이 실패면 전체 실패|
|`Promise.any([...])`|가장 먼저 성공한 것 하나만|전부 실패해야 전체 실패|
|`Promise.resolve(value)`|즉시 fulfilled인 Promise|—|
|`Promise.reject(reason)`|즉시 rejected인 Promise|—|

```typescript
// Promise.allSettled — 실패해도 나머지 결과를 수집
const results = await Promise.allSettled([fetchA(), fetchB(), fetchC()]);
results.forEach((result) => {
  if (result.status === 'fulfilled') console.log(result.value);
  if (result.status === 'rejected')  console.log(result.reason);
});
```