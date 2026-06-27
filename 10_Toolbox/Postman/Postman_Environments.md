---
aliases:
  - Postman 환경변수
  - Postman Environments
tags:
  - Postman
related:
  - "[[00_Postman_HomePage]]"
  - "[[Postman_Scripts]]"
---

# Postman_Environments — 환경변수

```txt
환경변수 = URL / 토큰 값을 변수로 관리
{{host}} / {{accessToken}} 처럼 사용
→ 값을 매번 복붙 안 해도 됨
→ 배포 환경 전환 시 변수 값만 바꾸면 됨
```

---

---

# 환경 생성 & 변수 추가 ⭐️

```txt
우측 상단 Environments 클릭
→ + 버튼 → 환경 이름 입력 (예: Local / Dev / Prod)
→ 변수 추가

⚠️ 반드시 환경을 선택해야 변수가 적용됨
   우측 상단 드롭다운에서 환경 선택
```

```txt
자주 쓰는 변수:
  host            http://localhost:3000
  accessToken     로그인 후 받은 JWT
  refreshToken    리프레시 토큰
```

---

---

# 변수 사용법 ⭐️

```txt
URL:    {{host}}/movie
        → host = http://localhost:3000 이면 자동 치환

Header: Authorization: Bearer {{accessToken}}
        → accessToken 값 자동 치환

Body:
  {
    "userId": "{{userId}}"
  }
```

---

---

# 토큰 자동 저장 — Tests 탭 ⭐️

```txt
로그인 API 호출 후 응답에서 토큰을 자동으로 환경변수에 저장
→ 이후 모든 요청에서 {{accessToken}} 자동 사용
```

```javascript
// POST /auth/login 요청 → Tests 탭에 작성
pm.test("Status code is 201", function () {
  pm.response.to.have.status(201);

  const body = pm.response.json();

  pm.environment.set("accessToken", body.accessToken);
  pm.environment.set("refreshToken", body.refreshToken);
});
```

```txt
동작 흐름:
  POST /auth/login 전송
    ↓
  응답에서 accessToken / refreshToken 꺼냄
    ↓
  pm.environment.set 으로 환경변수에 저장
    ↓
  이후 요청에서 {{accessToken}} 자동 참조

pm.environment.set("key", value)  환경변수 저장
pm.environment.get("key")         환경변수 읽기
pm.response.json()                응답 JSON 파싱
pm.response.to.have.status(N)     상태코드 검증
```

---

---

# 폴더 전체에 토큰 적용 ⭐️

```txt
매 요청마다 Authorization 헤더를 설정하는 것은 비효율적
→ 최상위 폴더에 한 번 설정하면 하위 전체 자동 적용
```

```txt
최상위 폴더 클릭 → Authorization 탭
  → Type: Bearer Token
  → Token: {{accessToken}}

register / login 요청:
  → Authorization 탭 → No Auth
  (로그인 전이라 토큰 없음)

나머지 모든 요청:
  → Authorization 탭 → Inherit auth from parent
  (상위 폴더 설정 자동 상속)
```

```txt
효과:
  토큰 갱신되면 환경변수 값만 업데이트
  → 하위 모든 요청 자동으로 새 토큰 사용
  → 요청마다 직접 바꿀 필요 없음
```