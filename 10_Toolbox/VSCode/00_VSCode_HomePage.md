---
aliases:
  - 00_VSCode_HomePage — VS Code
tags:
  - HomePage
related:
  - "[[00_Toolbox_HomePage]]"
  - "[[Mac_단축키]]"
---
# 00_VSCode_HomePage — VS Code

```
에디터 환경 설정 + 확장 + 코드 품질 도구(ESLint/Prettier) + 단축키
Mac 단축키 기호(⌘·⌥)가 헷갈리면 [[Mac_단축키]] 먼저 보기
```

---

# Level 0. 환경 · 기본 설정

| 노트 | 핵심 개념 |
|---|---|
| [[Settings_json_정리]] ⭐ | settings.json 열기 / 저장 시 포맷·린트 / 언어별 설정 / 전체 예시 |
| [[VS_Code_Extensions_추천]] ⭐ | 공통 필수 확장 / JS·TS·Nest / DB·Docker·Python / Obsidian 연동 |
| [[VSCode_단축키]] | 명령 팔레트 · 파일 검색 · 터미널 · 편집 단축키 (Mac 기준) |

---

# Level 1. 코드 품질 — 린트 · 포맷

| 노트 | 핵심 개념 |
|---|---|
| [[eslint]] | ESLint 설치 · 설정 · VS Code 연동 · 저장 시 자동 수정 |
| [[prettier]] | Prettier 설치 · 포맷 규칙 · ESLint와 역할 분담 · 프로젝트별 실행 |

```
eslint = 코드 품질·규칙 검사 / prettier = 들여쓰기·따옴표 등 스타일 통일
둘 다 [[Settings_json_정리]]의 editor.codeActionsOnSave · formatOnSave와 연결됨
```

---

# Level 2. 생산성 — 스니펫

| 노트 | 핵심 개념 |
|---|---|
| [[VSCode_User_Snippets_등록법]] | 스니펫 등록 위치 · prefix · body · tabstop · 지금은 보류해도 되는 이유 |

---

# 아직 없는 노트 — 필요할 때 추가

| 주제 | 비고 |
|---|---|
| Debugger | 브레이크포인트 · call stack — 실습하면서 정리 예정 |
| launch.json | 디버그 실행 설정 |
| tasks.json | 빌드·테스트 자동화 |
