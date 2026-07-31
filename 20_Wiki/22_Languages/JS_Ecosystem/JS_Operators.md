---
aliases:
  - 구조분해
  - deps 배열
  - rest
  - Promise.all
  - 쉼표로 건너뛰기
  - 스프레드
  - 논리 연산자
  - 삼항 연산자
  - 비교 연산자
  - typeof
  - instanceof
  - 옵셔널 체이닝
  - 불린 강제 변환
  - "[key]: value"
  - void
tags:
  - JavaScript
related:
  - "[[00_JS_Ecosystem_HomePage]]"
  - "[[JS_Promise]]"
  - "[[JS_FunctionPatterns]]"
  - "[[React_AsyncUI]]"
---
# JS_Operators — 연산자 & 구조분해

> [!info] 
> 구조분해 · 스프레드 · 논리 연산자처럼 매일 쓰는 문법이지만 `{ user: me }` 같은 이름 바꾸기나 `??=` 같은 할당 단축형은 처음 보면 헷갈린다.

---

# 구조분해 (Destructuring) ⭐️⭐️⭐️⭐️

## 객체 구조분해 — 기본

```typescript
const person = { name: '홍길동', age: 30, city: '서울' };

const { name, age } = person;
```

## 이름 바꾸기 (Alias) ⭐️⭐️⭐️⭐️

```typescript
const { name: displayName } = person;
// person.name을 꺼내서 displayName이라는 변수에 담음
// name 변수는 생기지 않음 — displayName만 생김

const { user: me } = useAuth();
// useAuth()가 반환한 객체의 user 필드를 me라는 이름으로 받음
```

```txt
{ 원래이름: 새이름 } — 콜론 뒤가 "이 범위에서 쓸 변수명"

왜 이름을 바꾸는가 — 이름 충돌(Naming Collision) 해결:

  function ProfileSheet({ user }: { user: User }) {
    // props의 user = 프로필 대상 (다른 사람)
    // useAuth()도 user를 반환하는데 이건 "로그인한 나"
    // 같은 스코프에서 user가 두 개 → 하나를 me로 이름 바꾸기

    const { user: me } = useAuth();

    // user = 보고 있는 상대방
    // me   = 로그인한 나
  }
```

## 이름 충돌과 deps 배열 ⭐️⭐️⭐️

```typescript
useEffect(() => {
  if (!user || !me) return;
  void loadRelation();
}, [open, user, me]);
//          ↑     ↑
//    props user  useAuth user (이름만 me로 바뀐 것)
```

```txt
deps에 user와 me가 둘 다 있는 이유:
  user  → 어떤 프로필을 보고 있는가 (props에서 온 변수)
  me    → 내가 누구인가 (useAuth에서 온 변수)

  const { user: me } = useAuth()는
  useAuth()의 user를 "me라는 새 변수명으로 꺼낸 것"일 뿐
  props의 user와는 출처부터 완전히 다른 별개의 값
  → 둘 다 deps 배열에 있어야 함
```

## 기본값 설정

```typescript
const { name = '익명', role = 'user' } = person;
// person.name이 undefined일 때만 '익명' 사용
// null이면 기본값 안 씀 (null ≠ undefined)
```

## 이름 바꾸기 + 기본값 조합

```typescript
const { user: me = null } = useAuth();
// user를 me로 꺼내는데, undefined면 null로 대체
```

## 중첩 구조분해

```typescript
const { address: { city, zip } } = person;

// 실전 — API 응답
const { data: { user, token } } = await login(email, password);
```

## 나머지 모으기 — rest

```typescript
const { name, ...rest } = person;
// name을 꺼내고, 나머지는 rest 객체로 모음
// rest = { age: 30, city: '서울' }
```

---

# 배열 구조분해 ⭐️⭐️⭐️

```typescript
const [first, second, ...others] = [1, 2, 3, 4, 5];

// 건너뛰기
const [, second, , fourth] = [1, 2, 3, 4];

// useState — 배열 구조분해의 대표 사례
const [count, setCount] = useState(0);
```

```txt
useState가 배열을 반환하는 이유:
  객체를 반환하면 { value, setValue }처럼 이름이 고정됨
  배열을 반환하면 원하는 이름으로 자유롭게 받을 수 있음
  → 같은 훅을 여러 번 써도 이름 충돌 없음
```

## Promise.all 결과 구조분해

```typescript
const [friends, requests] = await Promise.all([
  fetchFriends(),
  fetchFriendRequests(),
]);
```

## 필요한 것만 꺼내기 — 쉼표로 건너뛰기 ⭐️⭐️⭐️⭐️

```typescript
const [, , , systemMessage] = await this.prisma.$transaction([
  this.prisma.roomBan.upsert({ ... }),      // [0] — 필요 없음
  this.prisma.roomMember.delete({ ... }),   // [1] — 필요 없음
  this.prisma.room.update({ ... }),         // [2] — 필요 없음
  this.prisma.roomMessage.create({ ... }),  // [3] — 이것만 필요
]);
```

```txt
[, , , systemMessage] 읽는 법:
  쉼표 개수 = 건너뛰는 개수
  const [, b]     → 0번 건너뜀, 1번이 b
  const [, , b]   → 0,1 건너뜀, 2번이 b
  const [, , , b] → 0,1,2 건너뜀, 3번이 b
```

---

# 스프레드 (...) ⭐️⭐️⭐️⭐️

## 배열 스프레드

```typescript
const merged = [...a, ...b];
const copy   = [...a];                // 얕은 복사
const sorted = [...items].sort(...);  // 원본 안 건드리고 정렬
```

## 객체 스프레드

```typescript
const config = { ...defaults, ...overrides };
// 같은 키는 나중에 오는 것이 이김 ⭐️

// state 업데이트 패턴
setState(prev => ({ ...prev, name: '새이름' }));
```

```txt
⚠️ 스프레드는 얕은 복사 — 중첩 객체는 참조가 공유됨
  깊은 복사가 필요하면 structuredClone() → [[JS_BrowserAPI]]
```

## 함수 인자 스프레드

```typescript
const nums = [1, 5, 3, 2, 4];
Math.max(...nums);  // Math.max(1, 5, 3, 2, 4)
```

---

# 논리 연산자 ⭐️⭐️⭐️⭐️

## && — 앞이 truthy일 때만 뒤를 반환

```typescript
user && user.name           // user 있으면 user.name
isLoggedIn && <UserMenu />  // 조건부 렌더링
```

## || — 앞이 falsy면 뒤를 반환

```typescript
name || '익명'  // name이 falsy(undefined, null, '', 0 등)면 '익명'
port || 3000   // ⚠️ port가 0이면 0도 falsy로 처리됨
```

## ?? — null/undefined일 때만 뒤를 반환 ⭐️⭐️⭐️

```typescript
name ?? '익명'  // null 또는 undefined일 때만 '익명'
port ?? 3000   // 0은 그대로 0 (유효한 값 보존)
```

```txt
|| vs ??:
  || = falsy 전부 (0, '', false, null, undefined, NaN)
  ?? = null/undefined만

  포트 번호, 카운터처럼 0도 유효한 값이라면 반드시 ??
  → [[JS_OptionalChaining]] 참고
```

## &&= · ||= · ??= — 논리 할당 단축형 ⭐️⭐️

```typescript
a &&= b   // if (a) a = b
a ||= b   // if (!a) a = b
a ??= b   // if (a == null) a = b

// 실전 패턴
cache ??= await fetchData();    // 캐시가 없을 때만 fetch
user.nickname ||= '익명';        // 닉네임 없으면 기본값 설정
```

---

# 삼항 연산자 ⭐️⭐️⭐️

```typescript
const label = isLoggedIn ? '로그아웃' : '로그인';

// JSX 조건부 렌더링
{isLoading ? <Spinner /> : <Content />}
```

```txt
중첩 삼항은 가독성이 나빠짐 → 변수로 미리 계산하거나 if문 사용 권장
```

---

# 비교 연산자 ⭐️⭐️

```typescript
===  // 값 + 타입이 같음 (항상 이걸 쓸 것)
!==  // 값 또는 타입이 다름
==   // 타입 변환 후 비교 → '0' == 0 → true — 피할 것

NaN === NaN          // false — NaN은 자기 자신과도 같지 않음
Number.isNaN(value)  // NaN 확인은 이걸 쓸 것
```

---

# typeof · instanceof ⭐️⭐️⭐️

```typescript
typeof 'hello'      // 'string'
typeof 42           // 'number'
typeof null         // 'object' ← 버그, null이 아님
typeof undefined    // 'undefined'

[] instanceof Array  // true
err instanceof Error // true
```

```txt
런타임 타입 확인 → [[TS_Type_Guards]] 참고
```

---

# 옵셔널 체이닝 (?.) ⭐️⭐️⭐️

```typescript
user?.name           // user가 null/undefined면 undefined
user?.address?.city  // 중간 어디서든 없으면 undefined
arr?.[0]             // 배열도 ?.로
fn?.()               // 함수 존재할 때만 호출
```

```txt
자세한 내용 → [[JS_OptionalChaining]]
```

---

# !! — 불린 강제 변환 ⭐️⭐️⭐️⭐️

```typescript
!!user   // null/undefined → false, 객체 → true
!!''     // false
!!'hi'   // true
!!0      // false
```

```txt
! 하나:  논리 부정
!! 둘:   truthy/falsy를 boolean 타입으로 변환

왜 쓰는가 — 타입이 boolean이 되어야 할 때:
  const canDelete = actionMsg && user && room;
  // 타입: Message | User | Room | undefined (마지막 truthy 값)

  const canDelete = !!actionMsg && !!user && !!room;
  // 타입: boolean ✅
```

```typescript
const canDeleteEveryone =
  !!actionMsg &&
  !!user &&
  !!room &&
  (actionMsg.senderId === user.id || room.ownerId === user.id);
```

---

# 계산된 속성명 — [key]: value ⭐️⭐️⭐️⭐️

```typescript
const key = 'name';
const obj = { [key]: '홍길동' };
// → { name: '홍길동' }
```

```typescript
// 실전 — display 객체의 특정 키만 토글
const setDisplay = (key: keyof DisplayType, on: boolean) => {
  patch({ display: { ...d, [key]: on } });
  // key = 'mood', on = true → { ...d, mood: true }
};
```

```txt
대괄호 안의 표현식을 평가해서 그 결과를 속성 이름으로 사용
keyof 타입과 조합하면 타입 안전한 동적 키 접근 가능
```

---

# void — 의도적 무시 ⭐️⭐️⭐️

```typescript
void markRoomRead(roomId);
// → Promise를 반환하는 함수를 호출하되, 그 결과를 무시하겠다고 명시
```

```txt
void 연산자:
  표현식을 평가하고 항상 undefined를 반환
  async 함수나 Promise 앞에 붙이면 "이 Promise를 의도적으로 처리하지 않겠다"는 표시

ESLint no-floating-promises 규칙:
  await나 .catch()로 처리하지 않은 Promise를 에러로 잡음
  void를 붙이면 "의도적으로 무시하는 것"으로 인식 → 경고 없음

  ❌ markRoomRead(roomId);       // no-floating-promises 경고
  ✅ void markRoomRead(roomId);  // 의도적 무시임을 명시
  ✅ await markRoomRead(roomId); // 결과를 기다림
```

```typescript
// 이벤트 핸들러 — onClick은 void 반환을 기대하는데 async 함수는 Promise 반환
<button onClick={() => void handleSubmit()}>저장</button>

// useEffect 안에서 async 함수 즉시 실행
useEffect(() => {
  void load();
  return () => { cancelled = true; };
}, [deps]);
```

```txt
void vs await 판단 기준 → [[React_AsyncUI]] "fire-and-forget" 섹션
함수 옵션 객체 패턴 (force = false, Partial<T>) → [[JS_FunctionPatterns]]
```