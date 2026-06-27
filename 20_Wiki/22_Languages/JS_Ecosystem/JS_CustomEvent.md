---
aliases:
  - CustomEvent
  - dispatchEvent
  - addEventListener
  - pub/sub
tags:
  - JavaScript
related:
  - "[[00_JS_Ecosystem_HomePage]]"
  - "[[JS_BrowserAPI]]"
  - "[[React_Context]]"
---

# JS_CustomEvent — 컴포넌트 간 상태 동기화

> [!info] 
> `window.dispatchEvent`로 "이런 일이 있었다"를 발행하고, `window.addEventListener`로 그걸 구독한다. 
> `CustomEvent`는 그 발행 신호에 데이터(`detail`)까지 같이 실어 보낼 수 있는 이벤트 객체다. 서로 부모-자식 관계가 아닌 컴포넌트끼리, Context Provider 없이도 느슨하게 동기화할 때 쓴다.

---

# dispatchEvent / addEventListener — 이벤트 시스템 기초 ⭐️⭐️

```
브라우저의 모든 이벤트(click, scroll, resize...)는 전부 같은 시스템 위에서 동작함:

  EventTarget                  이벤트를 보내고 받을 수 있는 객체 (window/document/DOM 엘리먼트 전부)
  addEventListener(이름, 콜백)   "이 이름의 이벤트가 오면 이 함수를 실행해줘" 등록 (구독)
  dispatchEvent(이벤트객체)      "지금 이 이벤트가 발생했다" 라고 직접 발사(發射) (발행)
```

```javascript
// 이미 알고 있는 click 과 똑같은 구조
button.addEventListener('click', fn);
// 'click' 이벤트가 오면 fn 실행
// 사용자가 버튼을 누르면 → 브라우저가 알아서 dispatchEvent('click') 을 대신 호출해줌
// (이 부분은 브라우저가 처리해서 평소엔 안 보임)

window.addEventListener('my-event', fn);
// 'my-event' 가 오면 fn 실행
// 근데 'my-event' 는 브라우저가 자동으로 만들어주는 표준 이벤트가 아님
// → 그래서 내가 직접 window.dispatchEvent(...) 로 "지금 발생했다" 라고 쏴줘야 함
```

```
dispatchEvent 한 줄 정의:
  "이벤트를 직접, 수동으로 발생시키는 메서드"
  click 이 사용자 동작으로 자동 발생하듯, 내가 만든 이벤트는 dispatchEvent 로 직접 쏴야 함
```

## Event vs CustomEvent — detail 데이터 차이

```javascript
window.dispatchEvent(new Event('my-event'));
//                    ↑ 그냥 "발생했다" 신호만 — 데이터는 못 담음

window.dispatchEvent(new CustomEvent('my-event', { detail: { id: 1 } }));
//                    ↑ CustomEvent 는 detail 속성에 원하는 데이터를 담아서 보낼 수 있음
```

```
일반 Event:   "무슨 일이 있었다" 라는 신호만 전달 (데이터 없음)
CustomEvent:  신호 + 같이 보낼 데이터(detail) 까지 전달

→ "닉네임이 바뀌었다" 는 신호뿐 아니라 "바뀐 닉네임 값" 자체도 같이 넘기고 싶으니까
  CustomEvent 를 쓰는 것 (그냥 Event 면 구독하는 쪽이 데이터를 가져올 방법이 없음)
```

## 관련 메서드 한눈에

|메서드/객체|역할|
|---|---|
|`EventTarget`|이벤트를 주고받을 수 있는 모든 객체 (`window`/`document`/DOM 엘리먼트)|
|`addEventListener(name, fn)`|`name` 이벤트 발생 시 `fn` 실행 등록 (구독)|
|`removeEventListener(name, fn)`|등록 해제 (클린업 — 같은 함수 참조 필요)|
|`dispatchEvent(event)`|이벤트를 직접 발생시킴 (발행)|
|`new Event(name)`|데이터 없는 기본 이벤트 객체|
|`new CustomEvent(name, { detail })`|데이터(`detail`)를 담을 수 있는 이벤트 객체|
|`e.detail`|구독 쪽에서 `CustomEvent` 에 담긴 데이터 꺼내기|

```
정리:
  addEventListener = 듣기 (subscribe)
  dispatchEvent     = 쏘기 (publish)
  CustomEvent       = 쏠 때 데이터까지 같이 담는 봉투
```

---

# 왜 필요한가 — 트리 관계 없는 컴포넌트 동기화 ⭐️⭐️⭐️

```
일반적인 문제 상황:
  서로 부모-자식 관계가 아닌 두 컴포넌트(또는 레이아웃)가
  각자 따로 데이터를 들고 있는데, 한쪽에서 값이 바뀌면
  다른 쪽도 그 변화를 알아야 하는 상황

  예: 헤더의 알림 개수 / 장바구니 아이콘 / 닉네임 표시 / 테마 설정 등
      "어딘가에서 값이 바뀌었다" 를 트리 관계 없이 전체에 알려야 할 때

해결 방향:
  값이 바뀐 곳 → window 전체에 dispatchEvent 로 "이런 일이 있었다" 발행
  관심 있는 컴포넌트들 → 각자 addEventListener 로 그 이벤트를 구독, 자기 상태만 갱신
  → 두 컴포넌트가 서로의 존재를 몰라도 됨 (느슨한 연결)
```

---

# 기본 패턴 — 3단계 템플릿 ⭐️⭐️⭐️

```typescript
// 1. 이벤트 정의 (이름 + 데이터 타입)
export const MY_EVENT = 'my-app-something-updated';

// 2. 발행 함수 — 값이 바뀐 곳에서 호출
export const notifySomethingUpdated = <T>(data: T) => {
  if (typeof window === 'undefined') return;   // 서버에는 window 없음
  window.dispatchEvent(new CustomEvent<T>(MY_EVENT, { detail: data }));
};
```

```tsx
// 3. 구독 — 관심 있는 컴포넌트마다 (여러 군데 반복 사용 가능)
useEffect(() => {
  const handler = (e: Event) => {
    const data = (e as CustomEvent<T>).detail;
    // data 로 자기 상태 갱신
  };
  window.addEventListener(MY_EVENT, handler);
  return () => window.removeEventListener(MY_EVENT, handler);
}, []);
```

|단계|누가|무엇을|
|---|---|---|
|① 정의|공통 파일(`lib/*.ts`)|이벤트 이름 상수 + 데이터 타입|
|② 발행|값을 바꾸는 쪽|`window.dispatchEvent(new CustomEvent(...))`|
|③ 구독|값을 보여주는 쪽 (여러 곳 가능)|`useEffect` 안에서 `addEventListener` + 클린업|

```
이 템플릿이 일반적인 이유:
  바뀌는 데이터가 닉네임이든, 알림 개수든, 장바구니 개수든
  ①②③ 구조 자체는 항상 동일 — 이름/타입만 바뀜
  → 새로운 동기화가 필요할 때마다 이 템플릿을 그대로 복사해서 재사용 가능
```

---

# 실전 적용 예 — 유저 정보 동기화 ⭐️

```typescript
// lib/auth-user-sync.ts — 위 템플릿을 유저 정보 케이스에 적용한 것
import type { AuthUser } from '@/lib/auth-api';

export const AUTH_USER_UPDATED_EVENT = 'app-auth-user-updated';

export const notifyAuthUserUpdated = (user: AuthUser) => {
  if (typeof window === 'undefined') return;
  window.dispatchEvent(new CustomEvent<AuthUser>(AUTH_USER_UPDATED_EVENT, { detail: user }));
};
```

```tsx
// 헤더 / 다른 레이아웃 — 둘 다 같은 구독 코드 그대로 사용
useEffect(() => {
  const onUserUpdated = (e: Event) => {
    setNickname((e as CustomEvent<AuthUser>).detail.nickname);
  };
  window.addEventListener(AUTH_USER_UPDATED_EVENT, onUserUpdated);
  return () => window.removeEventListener(AUTH_USER_UPDATED_EVENT, onUserUpdated);
}, []);
```

```typescript
// 설정 페이지 — 닉네임 변경 성공 시 발행
const result = await updateNickname(newNickname);
notifyAuthUserUpdated(result.user);
```

```
왜 헤더와 다른 레이아웃이 각자 유저 정보를 따로 구독하나:
  서로 다른 레이아웃 트리라 한쪽 상태를 다른 쪽이 직접 알 방법이 없었음
  → 이벤트 발행 한 번으로 양쪽 다 동시에 갱신 (페이지 전체를 다시 불러올 필요 없어짐)
```

---

# 실전 적용 예 — 테마 동기화 ⭐️

```typescript
// lib/theme-sync.ts — 같은 템플릿을 테마 케이스에 적용
export const THEME_EVENT = 'app-theme-updated';

export const notifyThemeChange = (preference: ThemePreference) => {
  if (typeof window === 'undefined') return;
  window.dispatchEvent(new CustomEvent<ThemePreference>(THEME_EVENT, { detail: preference }));
};
```

```tsx
// 테마를 표시/적용하는 모든 곳 — 동일한 구독 코드
useEffect(() => {
  const onThemeChange = (e: Event) => {
    const preference = (e as CustomEvent<ThemePreference>).detail;
    applyTheme(preference);   // 적용 로직 자체는 [[JS_BrowserAPI]]의 "종합 — applyTheme" 참고
  };
  window.addEventListener(THEME_EVENT, onThemeChange);
  return () => window.removeEventListener(THEME_EVENT, onThemeChange);
}, []);
```

```typescript
// 설정 페이지 — 테마 변경 시 발행
const handleThemeChange = (preference: ThemePreference) => {
  localStorage.setItem('theme', preference);
  applyTheme(preference);
  notifyThemeChange(preference);   // 다른 레이아웃/컴포넌트에도 알림
};
```

```
유저 정보 예제와 정확히 같은 정의·발행·구독 구조 — 이름과 타입만 ThemePreference 로 바뀐 것
```

---

# (`e as CustomEvent<T>`) 타입 단언이 항상 필요한 이유

```
addEventListener 콜백의 매개변수 타입은 기본적으로 Event
실제로 들어오는 건 detail 을 가진 CustomEvent
→ TypeScript 에게 "이건 사실 CustomEvent야" 라고 알려줘야 detail 에 접근 가능
→ 어떤 데이터를 동기화하든 이 부분은 항상 동일하게 필요
```

---

# CustomEvent vs Context — 언제 무엇을 ⭐️⭐️

| |CustomEvent|Context (`useContext`)|
|---|---|---|
|관계|서로 모르는 컴포넌트끼리 (느슨한 연결)|Provider 트리 안에서 (명확한 계층)|
|설정|window 에 바로 발행/구독, Provider 불필요|최상위에 Provider 로 감싸야 함|
|적합한 경우|레이아웃이 달라 트리 관계가 먼 컴포넌트 동기화|앱 전역에서 자주 같이 쓰는 상태|
|예|헤더 ↔ 다른 레이아웃의 닉네임/알림 동기화|로그인 유저 정보를 앱 전체에서 공유|

```
판단 기준:
  이미 Context Provider 로 관리 중인 상태  → 그 Context 에 갱신 함수 추가가 더 자연스러움
  Provider 트리가 다르거나 새로 만들기엔 과한 상황  → CustomEvent 가 가벼움

Context 자체(createContext/useContext/Provider 만들 때 useMemo·useCallback이 왜 필요한지)는
[[React_Context]] 참고
```

---

# 한눈에

| 메서드/패턴                                                    | 역할                                                 |
| --------------------------------------------------------- | -------------------------------------------------- |
| `window.dispatchEvent(new CustomEvent(name, { detail }))` | 커스텀 이벤트 발행                                         |
| `window.addEventListener(name, cb)`                       | 커스텀 이벤트 구독                                         |
| `window.removeEventListener(name, cb)`                    | 구독 해제 (클린업 필수)                                     |
| `(e as CustomEvent<T>).detail`                            | 구독 쪽에서 데이터 꺼내기                                     |
| 3단계 템플릿                                                   | 정의(이름+타입) → 발행(값 바뀐 곳) → 구독(보여주는 곳, useEffect+클린업) |
| CustomEvent vs Context                                    | 느슨한 연결(트리 무관) vs 명확한 계층(Provider 트리 안)             |