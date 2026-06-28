---
aliases: [base64, Buffer, decoding, encoding]
tags:
  - NodeJS
related:
  - "[[00_NestJS_Ecosystem_HomePage]]"
  - "[[NestJS_JwtGuard]]"
  - "[[NodeJS_HTTP_Request]]"
---

# NodeJS_Buffer — Buffer와 인코딩/디코딩

> [!info] 
> 문자열은 사람이 읽는 텍스트지만, 컴퓨터가 실제로 저장/전송하는 건 바이트(byte)다.
>  `Buffer`는 Node.js가 그 바이트를 직접 다루기 위해 제공하는 객체이고, "인코딩"은 바이트 ↔ 문자열을 어떤 규칙으로 변환할지를 정하는 약속이다.

---

# Buffer란 무엇인가 ⭐️⭐️⭐️

```txt
파일, 네트워크 전송, 암호화 같은 건 전부 "바이트 단위"로 동작함
문자열('hello')은 사람이 읽기 좋게 추상화된 형태일 뿐, 실제로는 바이트로 변환돼서 저장/전송됨

Buffer = 그 원시 바이트들을 직접 들고 있는 Node.js 객체
  문자열은 불변(immutable)이고 "텍스트"로 다루지만
  Buffer는 가변(mutable)이고 "바이트 배열"로 다룸
```

```typescript
const buf = Buffer.from('hello', 'utf-8');
buf;              // <Buffer 68 65 6c 6c 6f>  ← 각 글자가 16진수 바이트로 표현됨
buf.length;        // 5 (바이트 개수)
buf.toString();     // 'hello' (다시 문자열로)
```

---

# 인코딩 — 바이트 ↔ 문자열 변환 규칙 ⭐️⭐️⭐️⭐️

```txt
인코딩 = "이 바이트들을 어떤 규칙으로 문자/기호로 해석할지"에 대한 약속
같은 바이트라도 인코딩이 다르면 완전히 다른 문자열로 해석될 수 있음
```

|인코딩|용도|
|---|---|
|`utf-8`|기본값 — 한글/영문 등 일반 텍스트|
|`base64`|텍스트만 허용되는 곳(URL, HTTP 헤더, JSON 등)에 바이너리 데이터를 안전하게 끼워 넣을 때|
|`base64url`|base64와 같지만 URL에서 문제되는 `+`/`/`/`=` 문자를 안전한 문자로 대체한 버전|
|`hex`|16진수 문자열 — 해시값 출력, 디버깅에 흔히 보임|
|`ascii` / `latin1`|영문/특수문자 위주의 단순한 인코딩 (한글 등은 깨짐)|

```typescript
Buffer.from('hello').toString('hex');     // '68656c6c6f'
Buffer.from('hello').toString('base64');  // 'aGVsbG8='
```

```txt
⚠️ 인코딩/디코딩은 항상 짝이 맞아야 함:
  Buffer.from(str, 'A인코딩').toString('B인코딩')  ← A와 B가 다르면 원래 문자열로 안 돌아옴
  (암호화/복호화와는 다른 개념 — 인코딩은 "비밀"이 아니라 그냥 "표현 방식 변환"일 뿐,
   누구나 디코딩 가능함. 보안이 필요하면 암호화/해싱을 따로 써야 함 — 자세한 건 [[NestJS_Bcrypt]] 참고)
```

---

# 실전 — Basic Auth 헤더 직접 만들어보기 ⭐️⭐️⭐️

```typescript
const credentials = Buffer.from(`${id}:${password}`).toString('base64');
const header = `Basic ${credentials}`;
// Authorization: Basic ZGVtbzpwYXNzd29yZA==
```

```typescript
// 반대로 디코딩 — 서버가 헤더를 받아서 원래 id:password로 복원
const decoded = Buffer.from(credentials, 'base64').toString('utf-8');
const [id, password] = decoded.split(':');
```

```txt
[[NodeJS_HTTP_Request]]에서 "Basic <base64(id:pw)>"라고만 적어뒀던 그 base64가
정확히 이 Buffer.from(...).toString('base64') 한 줄로 만들어지는 것

⚠️ base64는 암호화가 아님 — 누구나 똑같이 디코딩해서 원래 id:password를 바로 볼 수 있음
   Basic Auth가 HTTPS 없이는 위험한 이유가 바로 이것(그냥 "인코딩"만 돼있을 뿐, 안 숨겨짐)
```

---

# 실전 — JWT가 base64로 만들어지는 원리 ⭐️⭐️⭐️⭐️

```txt
JWT는 header.payload.signature 세 부분을 점(.)으로 이어붙인 문자열인데,
header와 payload 각각은 "JSON을 base64url로 인코딩한 것"일 뿐임
```

```typescript
const payload = { sub: 'user-1', role: 'admin' };
const encoded = Buffer.from(JSON.stringify(payload)).toString('base64url');
// JWT의 payload 부분과 정확히 같은 원리

// 디코딩하면 원래 JSON으로 — JWT의 payload를 "디코딩만 해서" 누구나 읽을 수 있는 이유
JSON.parse(Buffer.from(encoded, 'base64url').toString('utf-8'));
// { sub: 'user-1', role: 'admin' }
```

```txt
⚠️ JWT의 payload는 암호화돼있지 않음 — 그냥 base64url로 인코딩만 돼있어서
   누구나 디코딩해서 내용을 읽을 수 있음 (jwt.io 같은 사이트가 그냥 디코딩만 해서 보여주는 것)
   signature 부분만 비밀키로 서명돼있어서, "내용을 못 읽게" 하는 게 아니라
   "내용이 위조되지 않았음을 증명"하는 역할일 뿐임
   ([[NestJS_JwtGuard]]에서 다룬 payload/표준 클레임 설명이 이 원리 위에 있는 것)
```

---

# 한눈에

| 개념                      | 핵심                                                    |
| ----------------------- | ----------------------------------------------------- |
| `Buffer`                | Node.js가 원시 바이트를 직접 다루는 객체                            |
| `Buffer.from(str, enc)` | 문자열 → 바이트(특정 인코딩 기준)                                  |
| `buf.toString(enc)`     | 바이트 → 문자열(특정 인코딩 기준)                                  |
| 인코딩 ≠ 암호화               | 누구나 디코딩 가능 — "표현 방식 변환"일 뿐, 비밀 유지 기능 없음               |
| `base64`                | 텍스트만 허용되는 곳에 바이너리를 안전하게 끼워 넣는 인코딩 — Basic Auth, JWT 등 |
| `hex`                   | 16진수 표현 — 해시값 출력 등에 흔함                                |
| JWT payload             | 사실 JSON을 base64url로 인코딩한 것 — 암호화 아님, 누구나 읽을 수 있음      |