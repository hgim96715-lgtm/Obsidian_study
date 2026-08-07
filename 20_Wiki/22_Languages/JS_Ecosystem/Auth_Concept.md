---
aliases:
  - Auth
  - OAuth
  - Session
  - Token
  - JSON Web Token(JWT)
  - Access Token
  - Refresh Token
tags:
  - NestJS
related:
  - "[[00_JS_Ecosystem_HomePage]]"
  - "[[00_NestJS_Ecosystem_HomePage]]"
  - "[[NestJS_Auth]]"
  - "[[NestJS_JwtGuard]]"
---
# Auth_Concept — 인증 · 인가 개념

>[!info]
>인증(Authentication) = "당신이 누구인지 확인". 인가(Authorization) = "당신이 무엇을 할 수 있는지 확인". JWT = 서버가 서명한 토큰으로 인증 상태를 증명. NestJS 구현 → [[NestJS_Auth]], Guard(인가) → [[NestJS_JwtGuard]]

---

# 인증 vs 인가 ⭐️⭐️⭐️⭐️

```txt
인증 (Authentication) = "너가 누구야?"
  로그인 → 아이디·비밀번호 확인 → "홍길동이구나"
  → 신원 확인

인가 (Authorization) = "너가 이걸 해도 돼?"
  게시글 수정 요청 → "홍길동이 이 게시글 작성자야?" → 허용/거부
  → 권한 확인

  인증 없이 인가 불가: 누구인지 모르면 권한 확인도 못 함
  인증 ≠ 인가: 로그인했어도 다른 사람 글은 못 수정
```

---

# Session vs Token ⭐️⭐️⭐️⭐️

## Session 방식 (전통적)

```txt
① 로그인 → 서버가 세션 저장소에 { sessionId: 'abc', userId: '123' } 저장
② 브라우저에 sessionId='abc' 쿠키 전달
③ 다음 요청 → 쿠키의 sessionId로 서버가 세션 조회 → userId 확인

장점: 서버에서 세션 삭제로 즉시 로그아웃
단점: 서버가 세션을 저장해야 함 → 서버를 여러 대 쓰면 세션 공유 문제
     서버 부하 증가 (매 요청마다 DB/Redis 조회)
```

## Token(JWT) 방식

```txt
① 로그인 → 서버가 JWT 토큰 생성 (서버는 아무것도 저장 안 함)
② 브라우저에 토큰 전달
③ 다음 요청 → Authorization: Bearer {token} 헤더
④ 서버가 서명 검증 → 유효하면 내용(userId) 꺼내서 사용

장점: 서버가 아무것도 저장 안 해도 됨 → 서버 여러 대 써도 문제 없음
단점: 토큰이 만료되기 전에 강제 로그아웃 어려움
     토큰이 탈취되면 만료까지 사용 가능
```

---

# JWT — JSON Web Token ⭐️⭐️⭐️⭐️

```txt
JWT = "서버가 서명한 JSON"
  서버만 알고 있는 비밀키로 서명
  → 위조 불가 (비밀키 없으면 서명 못 함)
  → 서버는 서명만 검증하면 됨 (DB 조회 불필요)
```

## JWT 구조

```txt
eyJhbGciOiJIUzI1NiIsInR5cCI6IkpXVCJ9.eyJzdWIiOiJ1c2VyLTEyMyIsImlhdCI6MTY5MDAwMDAwMH0.signature

세 부분이 점(.)으로 구분:

Header.Payload.Signature

Header:   알고리즘 정보 (HS256, RS256 등)
Payload:  실제 데이터 — userId, 만료시간 등 (암호화 아님! Base64 인코딩)
Signature: Header+Payload를 비밀키로 서명한 값 (위조 방지)
```

```typescript
// Payload (디코딩하면 읽을 수 있음 — 민감 정보 넣으면 안 됨)
{
  sub:  'user-uuid',    // Subject — 사용자 ID
  iat:  1690000000,     // Issued At — 발급 시각 (Unix timestamp)
  exp:  1690086400,     // Expiration — 만료 시각
}

// sub, iat, exp는 JWT 표준 클레임(claim) 이름
```

```txt
⚠️ JWT Payload는 암호화가 아닌 Base64 인코딩
  누구나 디코딩해서 내용을 볼 수 있음
  → 비밀번호, 카드번호 등 민감 정보를 Payload에 넣으면 안 됨
  → userId 정도만 넣는 것이 안전

서명(Signature)의 역할:
  Payload가 바뀌면 Signature도 달라짐
  서버는 Signature를 검증해서 위조 여부 확인
  → userId를 '123'→'456'으로 바꾸면 Signature 불일치 → 거부
```

## Base64 — Payload가 보이는 이유 ⭐️⭐️⭐️⭐️

```txt
JWT의 Header·Payload는 Base64로 인코딩됨

Base64:
  바이너리/텍스트 데이터를 URL-safe 문자로 표현하는 방식
  암호화가 아님 — 누구나 디코딩 가능

eyJzdWIiOiJ1c2VyLTEyMyJ9
→ atob 으로 디코딩 →
{"sub":"user-123"}
```

```typescript
// 브라우저 개발자 도구에서 직접 확인 가능
const token = 'eyJhbGciOiJIUzI1NiJ9.eyJzdWIiOiJ1c2VyLTEyMyJ9.서명값';
const payload = JSON.parse(atob(token.split('.')[1]));
// { sub: 'user-123', iat: 1234, exp: 5678 }
// → 토큰을 가진 누구나 Payload 내용을 볼 수 있음
```

```txt
그럼 왜 안전한가:
  Payload를 읽을 수는 있지만 바꿀 수 없음
  Payload를 조작하면 Signature가 달라짐
  JWT_SECRET 없이는 올바른 Signature를 만들 수 없음
  → 서버가 Signature 검증에서 탈락시킴

원칙:
  Payload에는 "읽혀도 괜찮은 것"만 (userId 등)
  비밀번호·카드번호 등은 절대 Payload에 넣지 않음
```

---

# Access Token · Refresh Token ⭐️⭐️⭐️⭐️

## 각각 무엇인가

```txt
Access Token:
  API를 호출할 때 "나는 로그인한 사용자야"를 증명하는 토큰
  모든 요청의 Authorization 헤더에 담아서 보냄
  Authorization: Bearer {access_token}
  → 서버가 이 토큰으로 누가 요청했는지 확인

Refresh Token:
  Access Token이 만료됐을 때 새 Access Token을 발급받기 위한 토큰
  API 호출에 직접 사용하지 않음 — 오직 재발급 요청에만 사용
  POST /auth/refresh 엔드포인트에만 보냄
  → "내 Access Token 만료됐는데 새로 줘"
```

## 왜 두 가지가 필요한가

```txt
JWT를 하나만 쓰면:
  만료 시간이 짧으면 → 자주 로그인해야 해서 불편
  만료 시간이 길면  → 탈취 시 오래 사용 가능 → 위험

해결: 두 가지 토큰을 함께 사용
  Access Token  → 실제 API 호출에 사용, 만료 짧음 (15분~1시간)
  Refresh Token → Access Token 재발급에만 사용, 만료 김 (7일~30일)
```

## 흐름

```txt
① 로그인
   → Access Token (15분) + Refresh Token (7일) 발급

② API 호출 (15분 이내)
   Authorization: Bearer {access_token}
   → 서버가 Access Token 검증

③ Access Token 만료
   → Refresh Token으로 새 Access Token 요청
   POST /auth/refresh
   → 새 Access Token (15분) 발급

④ Refresh Token도 만료
   → 다시 로그인

⑤ 로그아웃
   → Refresh Token을 DB에서 삭제 or 블랙리스트 처리
   → Access Token은 만료 기다리거나 짧은 만료시간으로 관리
```

```txt
왜 Refresh Token은 DB에 저장하는가:
  로그아웃 = Refresh Token 무효화
  JWT는 서버가 아무것도 저장 안 하므로 강제 만료 불가
  → Refresh Token만 DB에 저장해서 로그아웃 시 삭제
  → Access Token은 짧은 만료로 자동 소멸에 의존
```

---

# OAuth 2.0 — 소셜 로그인 ⭐️⭐️⭐️

```txt
OAuth = "내 비밀번호를 상대방에게 안 알려줘도 권한을 위임하는 프로토콜"

구글 로그인 예시:
  ① 사용자가 "구글로 로그인" 클릭
  ② 구글 로그인 화면으로 이동 (구글이 인증 담당)
  ③ 구글이 "이 앱이 당신의 이름·이메일을 보려 합니다" 동의 요청
  ④ 동의 → 구글이 authorization code를 앱에 전달
  ⑤ 앱이 구글에 code + 앱 비밀키로 access_token 요청
  ⑥ 구글 access_token으로 사용자 정보 조회 (이름, 이메일)
  ⑦ 앱이 자체 JWT 발급 → 이후 자체 인증 시스템 사용
```

```txt
핵심:
  구글 비밀번호를 앱에 알려주지 않음
  구글이 인증을 처리, 앱은 결과(사용자 정보)만 받음
  구글 access_token ≠ 앱 JWT
  → 앱은 구글 정보로 자체 사용자를 만들고 자체 JWT를 발급

소셜 로그인이 필요한 이유:
  비밀번호 관리 부담 없음
  2FA, 보안 등을 구글/애플이 대신 처리
  사용자 편의 (새 계정 안 만들어도 됨)
```

---

# 토큰 저장 위치 ⭐️⭐️⭐️

```txt
브라우저에서 토큰을 어디 저장하는가:

localStorage:
  JS로 쉽게 접근 가능
  XSS 공격에 취약 (악성 스크립트가 토큰 탈취 가능)
  → Access Token 저장에 주의 필요

httpOnly Cookie:
  JS에서 접근 불가 (document.cookie로 못 읽음)
  서버가 Set-Cookie로 설정
  XSS에 안전, CSRF 주의 필요
  → Refresh Token 저장에 적합

메모리 (React state):
  페이지 새로고침 시 사라짐
  가장 안전 (JS로도 못 탈취)
  → Access Token 저장에 적합 (짧은 만료라 재발급으로 해결)
```