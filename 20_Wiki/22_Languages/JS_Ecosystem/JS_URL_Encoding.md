---
aliases: [decodeURIComponent, encodeURI, encodeURIComponent]
tags:
  - JavaScript
related:
  - "[[00_JS_Ecosystem_HomePage]]"
  - "[[NextJS_Routing]]"
---

# JS_URL_Encoding — URL에 안전하게 문자열 끼워넣기

> [!info] 
> URL에는 `?`, `&`, `{text}=`, `/`, 공백, 한글 같은 문자가 그대로 들어가면 주소 구조가 깨지거나 의미가 달라질 수 있다.
>  `encodeURIComponent()`는 그런 문자들을 `%XX` 형태로 안전하게 바꿔주는, Next.js와 무관한 순수 JS 전역 함수다.

```txt
Next.js 라우팅(NextJS_Routing)과는 별개의 주제임 — 어디서 URL을 직접 문자열로 조립하든
(쿼리스트링, fetch URL, a 태그 href 등) 항상 같은 이유로 필요해지는 범용 JS 함수
```

---

# 왜 필요한가 ⭐️⭐️⭐️

```typescript
const pathname = '/posts/공지사항?id=1'; // 한글, ?, = 가 섞여 있음

// ❌ 그냥 이어붙이면
`/login?callbackUrl=${pathname}`
// → '/login?callbackUrl=/posts/공지사항?id=1'
//   물음표(?)가 두 개가 되어버려서, 두 번째 ?부터는 callbackUrl의 값이 아니라
//   완전히 새로운 쿼리스트링이 시작된 것처럼 깨져서 해석됨

// ✅ encodeURIComponent로 감싸면
`/login?callbackUrl=${encodeURIComponent(pathname)}`
// → '/login?callbackUrl=%2Fposts%2F%EA%B3%B5%EC%A7%80%EC%82%AC%ED%95%AD%3Fid%3D1'
//   특수문자가 모두 %XX 로 바뀌어서, 전체가 callbackUrl 값 하나로 안전하게 인식됨
```

---

# encodeURIComponent vs encodeURI — 뭐가 다른가 ⭐️⭐️⭐️

|함수|어디에 쓰나|`/`, `?`, `&`, `=` 도 인코딩하나|
|---|---|---|
|`encodeURIComponent()`|URL의 "한 조각"(쿼리 파라미터 값 하나 등)을 인코딩할 때|예 — 전부 인코딩함|
|`encodeURI()`|URL "전체"를 인코딩할 때 (이미 `?`, `/`, `=` 등 구조가 살아있어야 함)|아니요 — URL 구조를 이루는 문자는 그대로 둠|

```txt
거의 항상 encodeURIComponent를 쓰게 됨 — "값 하나"를 안전하게 만드는 게 대부분의 상황이기 때문
encodeURI는 "이미 완성된 URL 전체에 비ASCII 문자(한글 등)만 인코딩하고 싶을 때"처럼 드문 경우에 씀
```

```typescript
const url = 'https://example.com/검색?q=리액트';

encodeURI(url);
// → 'https://example.com/%EA%B2%80%EC%83%89?q=%EB%9D%BC%EC%9D%B5%ED%8A%B8'
//   구조 문자(/, ?, =)는 그대로 두고 한글만 인코딩함

encodeURIComponent(url);
// → 'https%3A%2F%2Fexample.com%2F%EA%B2%80%EC%83%89%3Fq%3D%EB%9D%BC%EC%9D%B5%ED%8A%B8'
//   :, /, ?, = 까지 전부 인코딩해버림 — URL 전체에 쓰면 오히려 망가짐
```

---

# 디코딩 — decodeURIComponent ⭐️

```typescript
const param = '%2Fposts%2F1';
decodeURIComponent(param); // '/posts/1'
```

```txt
서버/클라이언트가 쿼리스트링을 읽을 때(예: searchParams.get('callbackUrl'))는
대부분 자동으로 디코딩까지 끝난 값을 돌려줌 — decodeURIComponent를 직접 쓸 일은 흔하지 않음
(직접 만든 URL 문자열을 파싱할 때 등 수동으로 다뤄야 하는 경우에만 필요)
```

---

# 한눈에

|함수|역할|
|---|---|
|`encodeURIComponent(str)`|URL의 한 조각(값 하나)을 안전한 문자열로 — 대부분의 경우 이걸 씀|
|`encodeURI(url)`|완성된 URL 전체에서 구조 문자는 두고 비ASCII 문자만 인코딩 — 드물게 씀|
|`decodeURIComponent(str)`|인코딩된 문자열을 원래대로 복원 — 보통은 자동으로 처리돼서 직접 쓸 일이 적음|

```txt
실전에서 떠올릴 한 줄: "URL 문자열 안에 변수를 끼워 넣는다" → encodeURIComponent부터 떠올리면 됨
```