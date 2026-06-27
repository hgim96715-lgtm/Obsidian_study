---
aliases:
  - Auth.js
  - NextAuth
  - Auth.js Prisma Adapter
tags:
  - NestJS
related:
  - "[[00_NestJS_Ecosystem_HomePage]]"
  - "[[00_JS_Ecosystem_HomePage]]"
  - "[[NestJS_Prisma]]"
  - "[[NestJS_Bcrypt]]"
---

# Auth_Concept — Auth.js란 무엇인가 + Prisma Adapter 표준 스키마

> [!info]
> 
>  Auth.js는 OAuth/이메일/Credentials 로그인을 대신 처리해주는 인증 라이브러리고,
>  
>   DB Adapter(Prisma 등)를 쓰면 User/Account/VerificationToken이라는 정해진 모양의 테이블에 결과를 저장한다. 
>   
>   `provider`+`providerAccountId`는 "어느 외부 서비스의 어떤 계정인지"를 식별하는 키, 토큰 필드들은 OAuth 인증 성공 시 그 서비스가 돌려준 값을 그대로 보관하는 곳이다.

```
이 노트와 [[NextJS_Auth]]의 역할 분담:
  Auth_Concept   Auth.js 자체의 개념 / 표준 스키마 (프레임워크 무관)
  NextJS_Auth    Next.js 안에서 실제로 설정/사용하는 방법 (라우트 핸들러, signIn/redirect 등)
→ NextJS_Auth에 이미 겹치는 내용이 있다면, 다음에 그 노트 내용을 보여주면 정리 가능
```

---

# Auth.js란 무엇인가 ⭐️⭐️

```
Auth.js는 "로그인 처리를 대신 해주는 라이브러리"임 — OAuth(구글/카카오 등),
이메일 매직링크, Credentials(직접 만든 이메일+비번) 등 여러 로그인 방식을 한 가지 인터페이스로 다룸

원래 이름은 NextAuth.js (Next.js 전용) 였다가, Next.js 외 다른 프레임워크
(SvelteKit, Express 등)도 지원하면서 프로젝트 이름이 Auth.js로 바뀜
→ npm 패키지명은 호환성 때문에 지금도 "next-auth" 그대로 씀 (v5 기준)
```

> [!info] 최신 상태(알아두기):
>  2025년 9월부터 Auth.js는 Better Auth 팀이 맡아서 유지보수 모드로 운영 중 — 보안 패치는 계속 들어오지만 신규 기능 개발은 멈춘 상태. 새로 시작하는 프로젝트엔 보통 Better Auth가 권장되지만, 이미 Auth.js v5로 구축된 프로젝트라면 급하게 옮길 필요는 없음.

```
Adapter란:
  Auth.js가 로그인 처리 결과(가입한 유저, 연동된 외부 계정 등)를
  "어떤 DB에 어떤 모양으로 저장할지"를 정해주는 연결 부품
  Prisma Adapter를 쓰면 → User / Account / VerificationToken(+ 필요시 Session) 테이블 모양이
  Auth.js가 미리 정해둔 표준 형태를 따라야 함 (필드를 추가하는 건 자유, 핵심 필드는 유지)
```

---

# 표준 스키마 vs  프로젝트가 커스터마이즈한 부분

```
Auth.js 공식 Prisma Adapter 예시의 User 모델은 보통 다음 필드를 가짐:
  id, name, email, emailVerified, image, accounts[], sessions[]

→ 위 User 모델은 name/image/emailVerified를 빼고
  nickname / passwordHash / role 을 추가한 커스터마이즈 버전임
```

|표준 필드|이 스키마에서는|이유로 추정되는 것|
|---|---|---|
|`name`|`nickname` (unique)|서비스에서 쓰는 닉네임 개념으로 대체|
|`image`|없음|아직 프로필 이미지 기능 없음|
|`emailVerified`|없음|이메일 인증 플로우 아직 미구현 (VerificationToken은 미래를 위해 미리 둔 것)|
|(없음)|`passwordHash`|Credentials(이메일+비번) 로그인을 직접 구현하려면 필요|
|(없음)|`role`|OAuth 표준에는 없는, 이 서비스의 권한 구분용 커스텀 필드|

```
Account / VerificationToken은 위 코드 그대로 Auth.js 공식 Prisma Adapter 표준과 동일함
→ 이 두 모델은 "내가 마음대로 바꾸면 안 되는 약속된 모양"이라고 보면 됨
   (필드 추가는 가능하지만, 핵심 필드 이름/타입을 바꾸면 어댑터가 못 알아봄)
```

---

# provider / providerAccountId — 외부 계정을 식별하는 키 ⭐️⭐️⭐️

|필드|뜻|예시 값|
|---|---|---|
|`provider`|어떤 서비스로 로그인했는지|`"google"`, `"kakao"`, `"github"`|
|`providerAccountId`|그 서비스 안에서의 고유 사용자 ID|구글이 발급한 sub 값 등|

```
이 둘을 묶어서 "외부 계정 한 개"를 식별함:
  provider만으로는 부족함 — "google"이라는 값만으론 어떤 구글 계정인지 모름
  providerAccountId만으로도 부족함 — 서로 다른 provider끼리 ID가 우연히 같을 수도 있음
  → (provider, providerAccountId) 조합이어야 "그 사람의 그 서비스 계정"이 정확히 하나로 잡힘

@@unique([provider, providerAccountId])
→ 똑같은 구글 계정으로 다시 로그인해도 Account 행이 또 생기지 않고
  이미 있는 행을 찾아서 그대로 씀 (중복 가입 방지의 원리가 이거)
```

---

# type — 이 계정이 어떤 방식으로 만들어졌는지 ⭐️

|값|의미|
|---|---|
|`"oauth"`|OAuth 2.0 (구글, 카카오 등 대부분의 소셜 로그인)|
|`"oidc"`|OpenID Connect (OAuth 위에 ID 토큰까지 포함된 방식)|
|`"email"`|이메일 매직링크 로그인|
|`"credentials"`|직접 만든 이메일+비밀번호 로그인|
|`"webauthn"`|패스키 등 WebAuthn 기반 로그인 (비교적 최근 추가)|

```
같은 Account 테이블에 여러 방식이 섞여 들어올 수 있어서,
"이 행이 어떤 방식으로 만들어졌는지"를 구분하려고 type을 같이 저장함
```

---

# 토큰 필드들 — OAuth 인증 성공 후 받아오는 값 ⭐️⭐️

```
OAuth 2.0 로그인이 성공하면, Google/Kakao 같은 외부 서비스가
"이 사용자를 대신해서 API를 호출해도 된다"는 증명서 역할의 토큰들을 돌려줌
Auth.js는 이 값들을 그대로 Account 행에 저장해둠
```

|필드|의미|
|---|---|
|`access_token`|API 호출에 실제로 쓰는 짧은 수명의 토큰|
|`refresh_token`|access_token이 만료됐을 때 재발급받기 위한 토큰 (없는 provider도 있음)|
|`expires_at`|access_token 만료 시각 (UNIX timestamp, 초 단위)|
|`token_type`|보통 `"Bearer"` — HTTP 요청 시 `Authorization: Bearer {access_token}` 형태로 씀|
|`scope`|이 토큰으로 접근 가능한 권한 범위 (예: `"email profile"`)|
|`id_token`|OpenID Connect에서 쓰는, 사용자 신원 정보를 담은 JWT|
|`session_state`|일부 provider(예: 구 Keycloak/Azure AD 계열)가 추가로 주는 세션 상태 값|

```
⚠️ Auth.js는 이 토큰들을 "저장"만 해주고, 만료된 access_token을 자동으로
   재발급(rotation)해주는 로직은 코어에 들어있지 않음 — 필요하면 직접 구현해야 함
   (refresh_token이 있다는 건, 그 재발급 로직을 직접 짤 수 있는 재료가 있다는 뜻)
```

---

# VerificationToken — 정확한 용도 ⭐️

```
공식 용도: "이메일 매직링크 로그인"에 쓰는 일회성 토큰
  사용자가 이메일을 입력하면 → 이 토큰을 만들어 메일로 링크 전송
  → 사용자가 그 링크를 클릭하면 token이 맞는지 확인하고 로그인 완료
  identifier = 보통 그 사용자의 이메일 주소
  @@unique([identifier, token]) → 같은 이메일에 발급된 토큰들 중 정확히 하나만 찾기 위함

지금 이 스키마에 Credentials provider만 있고 Email provider를 아직 안 붙였다면
VerificationToken 테이블은 당장은 비어있는 상태일 수 있음 — 나중에 매직링크나
이메일 인증(회원가입 후 "이메일을 확인해주세요") 기능을 붙일 때를 위해 미리 자리만 잡아둔 것
```

---

# 왜 Session 모델이 없을 수 있는가 — Credentials Provider 제약 ⭐️

```
Auth.js의 세션 보관 방식은 두 가지:
  jwt 전략       세션 정보를 암호화해서 쿠키에만 저장 — DB 테이블 불필요
  database 전략  Session 테이블에 저장 — Adapter를 쓰면 기본값이 이쪽으로 잡힘

그런데 Credentials provider(직접 만든 이메일+비번 로그인)는
database 전략을 지원하지 않음 — 반드시 jwt 전략이어야 함
(Auth.js가 던지는 에러: UnsupportedStrategy —
 "Signing in with credentials only supported if JWT strategy is enabled")

→ Credentials provider를 쓰면서 Adapter도 같이 쓰는 경우,
  session: { strategy: 'jwt' } 를 직접 명시해줘야 함
  (Adapter가 있으면 기본값이 database 전략으로 잡히기 때문에, 안 그러면 충돌남)
  이 경우 User/Account/VerificationToken은 그대로 쓰이지만
  Session 테이블은 필요 없어서 스키마에서 빠지는 것 — 이 프로젝트 스키마와 맞아떨어짐
```

---

# 한눈에

| 키워드                                                                                   | 한 줄 정리                                                                 |
| ------------------------------------------------------------------------------------- | ---------------------------------------------------------------------- |
| Auth.js                                                                               | 여러 로그인 방식을 한 인터페이스로 처리해주는 인증 라이브러리 (現 Better Auth 유지보수, 유지보수 모드)       |
| Adapter                                                                               | 로그인 결과를 어떤 DB에 어떤 모양으로 저장할지 정하는 연결 부품                                  |
| `provider`                                                                            | 어떤 서비스로 로그인했는지 (`"google"` 등)                                          |
| `providerAccountId`                                                                   | 그 서비스 안에서의 고유 사용자 ID                                                   |
| `@@unique([provider, providerAccountId])`                                             | 같은 외부 계정으로 중복 가입 방지                                                    |
| `type`                                                                                | `"oauth"` / `"oidc"` / `"email"` / `"credentials"` / `"webauthn"` 중 하나 |
| `access_token` / `refresh_token` / `expires_at` / `token_type` / `scope` / `id_token` | OAuth 인증 성공 시 외부 서비스가 돌려준 값 그대로 보관                                     |
| `VerificationToken`                                                                   | 이메일 매직링크 로그인용 일회성 토큰 (Email provider 전용)                               |
| Session 모델 없음                                                                         | Credentials provider는 jwt 전략 강제 → database 세션 테이블 불필요                  |