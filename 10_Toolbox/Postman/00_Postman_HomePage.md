---
aliases:
  - 00_Postman_HomePage — Postman
tags:
  - HomePage
related:
  - "[[00_Toolbox_HomePage]]"
  - "[[Postman_Basics]]"
---
# 00_Postman_HomePage — Postman

```txt
Postman = GUI로 HTTP/WebSocket 요청을 보내고 응답을 확인하는 API 테스트 도구
GET / POST / PUT / DELETE · Header / Body 직접 입력 · 환경변수로 토큰 관리

다운로드: https://www.postman.com/downloads/
```

---

# 학습 순서 — 위에서 아래로

| 순서 | 노트 | 핵심 개념 |
|:---:|---|---|
| 1 | [[Postman_Basics]] | Method / URL / Params / Headers / Body / 요청 흐름 |
| 2 | [[Postman_Environments]] ⭐ | `{{host}}` · `{{accessToken}}` · 환경변수 · 토큰 자동 저장 |
| 3 | [[Postman_WebSocket]] ⭐ | Socket.IO 연결 · 이벤트 송신·수신 · Listen to events |

---

# 아직 없는 노트 — 필요할 때 추가

| 주제 | 비고 |
|---|---|
| Postman_Scripts | Tests 탭 · `pm.environment` · pre-request 스크립트 |
| Postman_Auth | Bearer Token · Inherit auth · 폴더 단위 인증 |

```txt
[[Postman_Environments]]에 토큰 저장·환경변수 흐름이 이미 들어 있어서
Scripts·Auth는 스크립트를 더 깊게 쓸 때 따로 정리하면 됨
```
