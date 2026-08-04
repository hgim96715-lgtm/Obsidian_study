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
  - 불린 변환
  - 조건부 스프레드
  - Truthy
  - Falsy
tags:
  - JavaScript
related:
  - "[[00_JS_Ecosystem_HomePage]]"
  - "[[JS_Promise]]"
  - "[[JS_FunctionPatterns]]"
  - "[[React_AsyncUI]]"
  - "[[JS_Array_Methods]]"
  - "[[JS_Primitive_Methods]]"
---
# JS_Operators — 연산자 & 구조분해

>[!info]
>구조분해 · 스프레드 · 논리 연산자처럼 매일 쓰는 문법이지만 `{ user: me }` 같은 이름 바꾸기나 `??=` 같은 할당 단축형은 처음 보면 헷갈린다.
> Truthy/Falsy 전체 목록도 이 파일에 정리.

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

## 조건부 스프레드 — 조건이 참일 때만 속성 추가 ⭐️⭐️⭐️⭐️

```typescript
const obj = {
  always: 'here',
  ...(condition ? { key: value } : {}),
  //  ↑ 참이면 { key: value }를 spread → 속성 추가됨
  //  ↑ 거짓이면 {}를 spread → 아무것도 추가 안 됨
};
```

```typescript
// 실전 — Prisma 쿼리에서 cursor 있을 때만 추가
const rows = await this.prisma.post.findMany({
  where,
  take: take + 1,
  ...(query.cursor
    ? { cursor: { id: query.cursor }, skip: 1 }  // cursor 있음 → 두 속성 추가
    : {}),                                         // cursor 없음 → 아무것도 안 추가
  orderBy: { createdAt: 'desc' },
});
```

```txt
...(condition ? { ... } : {}) 읽는 법:

  조건이 참  → 객체를 스프레드 → 그 속성들이 바깥 객체에 포함됨
  조건이 거짓 → 빈 객체를 스프레드 → 빈 {}를 펼치면 아무것도 안 추가됨

왜 {} 를 쓰는가:
  ...(condition ? { key: value } : undefined)  ← undefined를 spread하면 에러
  ...(condition ? { key: value } : {})         ← 빈 객체는 안전하게 아무것도 안 함

언제 쓰는가:
  if문 없이 인라인으로 조건부 속성을 추가하고 싶을 때
  Prisma 쿼리, fetch options, API 파라미터 등 객체를 한 번에 만들 때

여러 개 조합도 가능:
  const query = {
    where,
    ...(take != null  ? { take: take + 1 }              : {}),
    ...(query.cursor  ? { cursor: { id: query.cursor },
                          skip: 1 }                     : {}),
    ...(query.orderBy ? { orderBy: query.orderBy }       : {}),
  };
```

## 함수 인자 스프레드

```typescript
const nums = [1, 5, 3, 2, 4];
Math.max(...nums);  // Math.max(1, 5, 3, 2, 4)
```

---

# Truthy · Falsy ⭐️⭐️⭐️⭐️

```txt
JavaScript에서 if문이나 논리 연산자(&&, ||, !)는
값을 true/false로 "해석"한다 — 이게 truthy/falsy

Boolean(값)이 false가 되는 값 = falsy
나머지 전부 = truthy
```

## Falsy 전체 목록

```typescript
Boolean(false)      // false
Boolean(0)          // false ← 숫자 0
Boolean(-0)         // false ← 음수 0
Boolean(0n)         // false ← BigInt 0
Boolean('')         // false ← 빈 문자열
Boolean(null)       // false
Boolean(undefined)  // false
Boolean(NaN)        // false

// 이 7가지만 falsy — 나머지는 전부 truthy
```

```typescript
// Truthy 예시 — 헷갈리는 것들
Boolean('false')    // true  ← 문자열 'false'는 truthy!
Boolean('0')        // true  ← 문자열 '0'은 truthy!
Boolean([])         // true  ← 빈 배열도 truthy
Boolean({})         // true  ← 빈 객체도 truthy
Boolean(-1)         // true  ← 음수도 truthy (0만 falsy)
Boolean(Infinity)   // true
```

```txt
가장 많이 헷갈리는 것:
  '0'   (문자열) → truthy   vs   0 (숫자) → falsy
  'false' (문자열) → truthy  vs  false (불린) → falsy
  []  빈 배열 → truthy
  {}  빈 객체 → truthy
```

## Falsy를 이용한 패턴

```typescript
// 값이 있는지 확인
if (value) { ... }          // null, undefined, '', 0, false, NaN → 통과 안 됨

// 기본값 설정 (||)
const name = input || '익명';   // input이 falsy면 '익명'

// 조건부 실행 (&&)
user && doSomething(user);      // user가 truthy일 때만 실행

// Boolean 변환
!!value                         // truthy → true, falsy → false
Boolean(value)                  // 동일
```

```txt
⚠️ || 를 기본값으로 쓸 때 주의:
  0이나 '' 도 유효한 값인데 falsy라서 기본값으로 넘어감
  count || 10  →  count가 0이면 10이 됨 (의도와 다를 수 있음)
  → 0, '' 도 유효한 값이면 ?? (nullish coalescing) 사용
  count ?? 10  →  count가 null/undefined일 때만 10
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

## ?? — null/undefined일 때만 뒤를 반환 ⭐️⭐️⭐️⭐️

```typescript
name ?? '익명'  // null 또는 undefined일 때만 '익명'
port ?? 3000   // 0은 그대로 0 (유효한 값 보존)
```

## ?? vs || — 가장 헷갈리는 차이 ⭐️⭐️⭐️⭐️

```typescript
const count = 0;
count || 10  // 10  ← 0은 falsy라 || 가 기본값으로 넘어감 (의도와 다를 수 있음)
count ?? 10  //  0  ← 0은 null/undefined가 아니므로 ?? 는 그대로 0 유지
```

| 연산자               | 기본값으로 넘어가는 조건                                             |
| ----------------- | --------------------------------------------------------- |
| <code>\|\|</code> | falsy 전부 (`0`, `''`, `false`, `null`, `undefined`, `NaN`) |
| `??`              | 정확히 `null` 또는 `undefined`일 때만                             |

```txt
"0이나 빈 문자열도 유효한 값으로 살리고 싶다"면 반드시 ??
페이지 번호(0이 유효한 값), 카운터, 빈 문자열 입력 → || 쓰면 의도와 다르게 동작
```

## ?.와 ?? 조합 — 에러 메시지 추출 패턴 ⭐️⭐️⭐️

```typescript
// error?.message ?? '기본 메시지'
// ?.와 ??가 짝으로 자주 보이는 이유:
//   ?.  → "혹시 없을 수도 있는 값"을 안전하게 꺼내고
//   ??  → "그게 진짜 없으면 쓸 기본값"을 바로 옆에 정해두는 조합

const error = (await res.json()) as { message?: string | string[] } | null;
const message = Array.isArray(error?.message)
  ? error.message[0]
  : error?.message;
throw new Error(message ?? `요청 실패: ${res.status} ${res.statusText}`);
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

# 옵셔널 체이닝 (?.) ⭐️⭐️⭐️⭐️

```typescript
user?.name           // user가 null/undefined면 undefined (에러 안 남)
user?.address?.city  // 중간 어디서든 없으면 거기서 멈추고 undefined 반환
arr?.[0]             // 배열/동적 키 접근에도 동일
fn?.()               // 함수 존재할 때만 호출, 없으면 아무 일도 안 함
```

```txt
?. 없이 쓰면:
  user.name  →  user가 null이면 "Cannot read properties of null" TypeError로 죽음

?. 쓰면:
  user?.name  →  user가 null/undefined면 그 자리에서 멈추고 undefined 반환
  체인이 길어도 중간 어디서든 끊어줌 (a?.b?.c?.d)
```

## ?.() — 함수 호출에 적용 ⭐️⭐️⭐️⭐️

```typescript
onClose?.();
// onClose가 함수면 호출, undefined/null이면 아무 일도 안 일어남
// if (onClose) onClose(); 와 동일
```

## prev?.() — 기존 콜백 보존 패턴 ⭐️⭐️⭐️⭐️

```typescript
// 전역 이벤트 핸들러에 새 동작을 추가하되, 기존 콜백도 유지해야 할 때
const prev = window.onSomeEvent;  // 기존 콜백 저장

window.onSomeEvent = () => {
  prev?.();       // 기존 콜백이 있으면 먼저 실행
  doNewThing();   // 그다음 새 동작
};
```

```txt
prev?.()가 필요한 이유:
  다른 라이브러리가 이미 window.onSomeEvent를 등록해뒀을 수 있음
  그냥 덮어쓰면 기존 콜백이 사라짐 → 다른 기능이 망가질 수 있음
  → 기존 것을 prev에 저장 → 새 핸들러에서 prev?.()로 기존 것도 같이 실행

  prev가 undefined면 prev?.()는 아무 일도 안 함
  prev가 함수면 먼저 실행하고 새 동작 실행

외부 스크립트 로드 시 자주 등장 (YouTube IFrame API 등)
```

---

# ! — NOT 연산 (논리 부정) ⭐️⭐️⭐️⭐️

```typescript
!true           // false
!false          // true
!isRoomMuted()  // isRoomMuted()가 true면 false, false면 true
!user           // user가 null/undefined/''/''/0 이면 true
```

```txt
! 는 "아닌" — 조건을 반대로 뒤집음

  !isRoomMuted(userId, roomId)
  → "뮤트가 아닌" — 뮤트 안 된 방

  !e.shiftKey
  → "Shift 키 안 눌림"

  !e.nativeEvent.isComposing
  → "한글 조합 중 아님"
```

---

# !! vs Boolean() — 불린 변환 ⭐️⭐️⭐️⭐️

```typescript
// 셋 다 완전히 동일한 동작
!!room.unread           // false → false, 0 → false, "abc" → true
Boolean(room.unread)    // 동일
room.unread ? true : false  // 동일 (비권장 — 길기만 하고 같음)
```

```txt
!! vs Boolean() — 동작은 같고 스타일 차이:
  !!          짧고 자주 쓰임, 코드 안에서 인라인으로
  Boolean()   읽을 때 "불린으로 변환한다"는 의도가 더 명확
              특히 setState나 함수 인자에 단독으로 쓸 때 가독성 좋음

왜 변환이 필요한가 — 타입이 boolean이 아닌 경우:
  room.unread 가 number | undefined 타입이라면
  setRoomsUnread(room.unread)  → TS 에러 (boolean 자리에 number 올 수 없음)
  setRoomsUnread(!!room.unread)      → ✅ boolean
  setRoomsUnread(Boolean(room.unread)) → ✅ boolean

  && 체이닝 결과:
  const canDelete = actionMsg && user && room;
  // 타입: Message | User | Room | undefined (마지막 truthy 값, boolean 아님)

  const canDelete = !!actionMsg && !!user && !!room;
  // 타입: boolean ✅
```

## 복합 조건 패턴 — &&, ||, ! 조합 ⭐️⭐️⭐️⭐️

```typescript
// 실전 예시
setRoomsUnread(
  mine.some(
    (room) => Boolean(room.unread) && !isRoomMuted(user.id, room.id),
  ),
);
setDmsUnread(dms.some((dm) => dm.unread) || requests.length > 0);
```

```txt
위 코드 읽는 법:

  mine.some((room) => Boolean(room.unread) && !isRoomMuted(user.id, room.id))
  → "방 목록(mine) 중에서 하나라도 (읽지 않았고 && 뮤트가 아닌) 방이 있는가?"
  → .some() = "하나라도" → boolean 반환 → setRoomsUnread(boolean)에 딱 맞음

  dms.some((dm) => dm.unread) || requests.length > 0
  → "DM 중 읽지 않은 게 하나라도 있거나 || 친구 요청이 하나라도 있으면"
  → 둘 중 하나라도 true면 전체 true

&& 조건 연결:
  A && B    A가 true이고 B도 true일 때 → 둘 다 만족
  A && !B   A가 true이고 B는 false일 때 → "A인데 B는 아닌"

|| 조건 연결:
  A || B    A가 true이거나 B가 true일 때 → 하나라도 만족

! 단독:
  !fn()     fn()의 반환값을 반대로 — "이 함수가 false를 반환할 때"
```

```typescript
// 복합 조건을 변수로 분리해서 읽기 쉽게
const hasUnreadRoom = mine.some(
  (room) => Boolean(room.unread) && !isRoomMuted(user.id, room.id),
);
const hasUnreadDm = dms.some((dm) => dm.unread);
const hasPendingRequest = requests.length > 0;

setRoomsUnread(hasUnreadRoom);
setDmsUnread(hasUnreadDm || hasPendingRequest);
// 각각 변수 이름이 의도를 설명해줌 → 한 줄이 너무 복잡하면 이 방식 권장
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