---
aliases:
  - 00_Toolbox_HomePage — 개발 도구
tags:
  - HomePage
related:
  - "[[00_Obsidian_HomePage]]"
  - "[[00_VSCode_HomePage]]"
  - "[[00_Postman_HomePage]]"
  - "[[00_Mac_HomePage]]"
---
# 00_Toolbox_HomePage — 개발 도구

> [!info]
> `20_Wiki`는 **언어·프레임워크·인프라 개념**을, `10_Toolbox`는 **매일 쓰는 도구와 작업 환경**을 모아 둔 폴더다.  
> VS Code 설정, Postman으로 API 테스트, Mac 단축키처럼 "코드 밖"에서 손에 익히는 것들이 여기 해당한다.

---

# 폴더 구성 — 지금 있는 것만

| 폴더 | 역할 | 홈페이지 |
|---|---|---|
| **Obsidian/** | vault 설정 · 플러그인 · 스니펫 | [[00_Obsidian_HomePage]] |
| **VSCode/** | 에디터 설정 · 확장 · 린트/포맷 · 단축키 | [[00_VSCode_HomePage]] |
| **Postman/** | HTTP/WebSocket API 테스트 | [[00_Postman_HomePage]] |
| **Mac/** | macOS 작업 환경 · 시스템 단축키 | [[00_Mac_HomePage]] |

```
세 폴더는 서로 독립 — JS Ecosystem처럼 클러스터로 얽히지 않음
다만 Mac 단축키(⌘·⌥ 기호)는 VS Code·터미널 노트를 읽을 때 같이 보면 편함
```

---

# 빠른 진입 — 상황별로 어디로 갈지

| 하고 싶은 일 | 먼저 볼 노트 |
|---|---|
| API 요청 보내고 응답 확인 | [[Postman_Basics]] → [[Postman_Environments]] |
| Socket.IO / WebSocket 테스트 | [[Postman_WebSocket]] |
| VS Code 저장 시 자동 포맷·린트 | [[Settings_json_정리]] → [[eslint]] · [[prettier]] |
| 확장 프로그램 뭐 깔지 고민 | [[VS_Code_Extensions_추천]] |
| VS Code 단축키만 빠르게 | [[VSCode_단축키]] |
| Mac ⌘·⌥ 기호가 뭔지 모르겠음 | [[Mac_단축키]] |
| Obsidian 플러그인 뭐 켜져 있는지 | [[Obsidian_Plugins]] |
| Callout·테마 색이 이상함 | [[Obsidian_Vault_Settings]] |

---

# Wiki와의 경계

| 여기 (`10_Toolbox`) | Wiki (`20_Wiki`) |
|---|---|
| 도구 UI·설정·단축키 | 언어 문법·프레임워크 개념 |
| Postman으로 Nest API 호출 | NestJS Controller·Guard 개념 |
| VS Code ESLint 설정 | ESLint 규칙이 코드에 주는 영향 (개념) |
| Git 명령어 | `20_Wiki/26_Infrastructure/git/` |
