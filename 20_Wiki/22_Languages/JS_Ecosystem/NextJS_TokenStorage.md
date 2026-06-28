---
aliases:
  - Access Token Storage
  - authToken
  - localStorage
tags:
  - NextJS
related:
  - "[[00_JS_Ecosystem_HomePage]]"
  - "[[Auth_Concept]]"
  - "[[JS_BrowserAPI]]"
  - "[[NestJS_JwtGuard]]"
  - "[[NextJS_AuthCache]]"
  - "[[Web_XSS_CSRF]]"
---
# NextJS_TokenStorage — Access Token을 클라이언트에 저장하기

> [!info] 
> 백엔드가 발급한 Bearer 토큰을 매 `fetch` 호출마다 다시 입력할 수는 없으니, 브라우저 어딘가(주로 `localStorage`)에 저장해두고 꺼내 쓴다. 
> 
> `typeof window === 'undefined'` 체크는 Next.js가 서버에서도 같은 코드를 렌더링하기 때문에 필요
> 
> — 서버에는 `localStorage` 자체가 없다.

```txt
이 노트와 다른 노트의 역할 분담:
  Auth_Concept       왜 백엔드(NestJS)가 인증을 전부 소유하고, Web은 토큰을 들고만 있는지(아키텍처 결정)
  NestJS_JwtGuard     이 토큰을 서버가 어떻게 검증하는지
  Web_XSS_CSRF        아래 표에 나오는 "XSS 노출"/"CSRF 노출"이 정확히 어떤 공격인지
  이 노트             그 토큰을 브라우저에서 어떻게 들고 있을지(저장 위치, SSR 이슈, 보안 트레이드오프)
```

---

# 왜 토큰을 클라이언트에 저장해야 하나 ⭐️⭐️

```txt
로그인 응답으로 받은 accessToken은 그 요청이 끝나는 순간 변수에서 사라짐
→ 다음 페이지로 이동하거나 새로고침해도 "로그인된 상태"를 유지하려면
  그 토큰을 어딘가에 저장해두고, 이후의 모든 API 요청에 다시 꺼내서 실어야 함

백엔드가 인증/User/DB를 전부 소유하고 프론트는 API 클라이언트로만 동작하는 구조에서는
(자세한 아키텍처 판단은 [[Auth_Concept]] 참고) 이 "저장"이 Next.js 쪼에서 책임져야 하는 부분임
```

---

# 저장 위치 선택 — localStorage vs sessionStorage vs 메모리 vs 쿠키 ⭐️⭐️⭐️

| 저장 위치                    | 지속성               | XSS 노출           | CSRF 노출           | 비고                        |
| ------------------------ | ----------------- | ---------------- | ----------------- | ------------------------- |
| `localStorage`           | 새로고침/탭 닫아도 유지     | JS로 읽을 수 있어 노출됨  | 해당 없음 (자동 전송 안 됨) | 가장 흔하게 쓰는 선택              |
| `sessionStorage`         | 탭 닫으면 사라짐 (탭별 독립) | JS로 읽을 수 있어 노출됨  | 해당 없음             | "이 탭에서만 로그인 유지"가 필요할 때    |
| 메모리(React state/Context) | 새로고침하면 사라짐        | 페이지에 있는 동안만 노출   | 해당 없음             | 가장 짧은 노출 시간, 단 매번 재로그인 필요 |
| httpOnly 쿠키              | 새로고침/탭 닫아도 유지     | JS로 못 읽음(노출 안 됨) | 요청마다 자동 전송돼서 노출됨  | 서버가 쿠키를 직접 발급해줘야 함        |

```txt
XSS/CSRF가 정확히 어떤 공격이고 왜 위 표처럼 노출 여부가 갈리는지는 [[Web_XSS_CSRF]] 참고 —
"악성 스크립트가 페이지 안에서 실행되는 것"(XSS)과 "다른 사이트가 내 대신 요청을 보내는 것"(CSRF)은
서로 완전히 다른 공격이고, 막는 방법도 다름
```


```txt
정답은 없음 — 무엇에 더 취약해도 괜찮은지를 고르는 트레이드오프:

  localStorage/sessionStorage   "내 사이트에 악성 스크립트가 끼어들면(XSS) 토큰을 털릴 수 있다"는 약점
  httpOnly 쿠키                  "다른 사이트가 내 의지와 무관하게 요청을 보내게 할 수 있다(CSRF)"는 약점
                                  (단, SameSite 속성으로 상당 부분 막을 수 있음)

Bearer 토큰 방식 자체가 "쿠키가 아니라 직접 헤더에 실어 보내는" 전제로 설계된 방식이라,
localStorage(또는 sessionStorage)에 두는 게 Bearer 방식과 가장 자연스럽게 맞물림
→ 이게 흔히 보이는 조합인 이유 (쿠키에 두려면 차라리 Bearer 대신 쿠키 기반 인증으로 가는 게 더 일관적)
```

---

# typeof window === 'undefined' — 왜 필요한가 ⭐️⭐️⭐️

```txt
Next.js는 같은 컴포넌트/모듈 코드를 서버에서도 한 번 실행함
(서버 사이드 렌더링, 혹은 빌드 시점의 정적 생성) — 이때는 브라우저가 아니라 Node.js 환경이라
window/localStorage 같은 "브라우저 전역 객체" 자체가 존재하지 않음

→ 서버에서 실행되는 코드 경로 중에 localStorage.getItem(...) 이 호출되면
  "window is not defined" 같은 에러로 그 자리에서 죽어버림

typeof window === 'undefined' 로 먼저 확인해서:
  서버에서 실행 중이면 → 그냥 null 반환하고 끝 (에러 없이 조용히 빠져나감)
  브라우저에서 실행 중이면 → window가 있으니 그제서야 localStorage 접근
```

```txt
window/localStorage 자체가 뭔지, 어떤 메서드가 있는지는 [[JS_BrowserAPI]] 참고
이 노트는 "Next.js 환경에서 왜 이 한 줄이 필요한지"에만 집중
```

```txt
getApiAccessToken()에만 이 체크가 있고 setApiAccessToken()/clearApiAccessToken()엔 없는 이유:
  set/clear는 보통 "로그인 성공 처리"처럼 브라우저에서 실제로 발생한 이벤트 이후에만 호출되므로
  실무에서는 생략하는 경우가 많음 — 다만 일관성과 방어적 코드를 더 중요하게 본다면
  세 함수 모두에 같은 가드를 넣어도 됨 (틀린 방법은 아니고, 취향/팀 컨벤션 차이)
```

---

# 함수 3개로 감싸는 이유 — get / set / clear ⭐️⭐️

```typescript
const STORAGE_KEY = 'access_token'; // 같은 도메인의 다른 앱과 안 겹치게 접두사를 붙이기도 함

export function getApiAccessToken(): string | null {
  if (typeof window === 'undefined') return null;
  return localStorage.getItem(STORAGE_KEY);
}

export function setApiAccessToken(token: string): void {
  localStorage.setItem(STORAGE_KEY, token);
}

export function clearApiAccessToken(): void {
  localStorage.removeItem(STORAGE_KEY);
}
```

```txt
localStorage.getItem(STORAGE_KEY)를 코드 곳곳에 직접 흩어 쓰지 않고 함수로 감싸는 이유:

  1. 저장 키(STORAGE_KEY) 이름이 한 곳에만 있음 — 오타로 다른 키를 읽는 사고 방지
  2. 나중에 저장 방식을 바꿔도(localStorage → sessionStorage, 혹은 메모리 방식 등)
     이 세 함수 내부만 고치면 됨 — 호출하는 쪽 코드는 그대로
  3. string | null 같은 반환 타입을 함수 시그니처로 명시해서, 호출하는 쪽이
     "토큰이 없을 수도 있다"는 걸 타입으로 강제로 알게 됨
```

|함수|쓰이는 시점|
|---|---|
|`getApiAccessToken()`|API를 호출하기 직전 — Bearer 헤더에 실을 값을 가져올 때|
|`setApiAccessToken(token)`|로그인/회원가입 성공 응답을 받은 직후|
|`clearApiAccessToken()`|로그아웃, 또는 토큰이 만료되어 더 이상 못 쓸 때|

---

# 다음 단계 — 이 토큰이 실제로 쓰이는 곳 (예약)

```txt
getApiAccessToken()은 여기서 끝이 아니라, 모든 API 요청을 감싸는 fetch 래퍼(흔히 authFetch 같은
이름)가 Authorization: Bearer 헤더에 이 값을 실어 보내는 데 씀 — 그 래퍼 자체의 설계
(401 처리, 토큰 만료 시 동작 등)는 아직 다루지 않음, 만들게 되면 이 노트에 이어서 추가하거나
필요한 만큼 커지면 새 노트로 분리할 것
```

---

# 한눈에

```txt
토큰을 브라우저에 저장해야 하는 이유: 요청마다 다시 로그인할 수 없으니 어딘가에 들고 있어야 함

저장 위치 선택은 트레이드오프:
  localStorage/sessionStorage → XSS에 약함 / CSRF 안전 / Bearer 방식과 자연스럽게 맞음
  httpOnly 쿠키 → XSS 안전 / CSRF에 약함(SameSite로 보완) / Bearer가 아니라 쿠키 기반 인증에 맞음

typeof window === 'undefined': Next.js가 서버에서도 같은 코드를 실행하기 때문에
  브라우저 전역 객체(window/localStorage)가 없는 환경을 미리 걸러내는 가드

get/set/clear로 함수를 감싸는 이유: 저장 키 단일화, 저장 방식 교체 용이성, 타입 명시
```

| 키워드                             | 한 줄 정리                                  |
| ------------------------------- | --------------------------------------- |
| `STORAGE_KEY`                   | localStorage에 저장할 때 쓰는 키 이름 — 한 곳에서만 관리 |
| `typeof window === 'undefined'` | 서버 렌더링 환경인지 먼저 확인하는 가드                  |
| XSS 노출                          | localStorage는 같은 페이지의 악성 스크립트가 읽을 수 있음  |
| CSRF 노출 없음                      | Bearer 토큰은 자동으로 전송되지 않아 CSRF 공격엔 해당 없음  |
| 함수로 감싸는 이유                      | 저장 방식을 나중에 바꿔도 호출하는 쪽 코드는 안 바뀜          |