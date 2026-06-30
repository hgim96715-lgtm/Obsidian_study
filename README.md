---
aliases:
  - README
tags:
  - README
related:
  - "[[00_Toolbox_HomePage]]"
  - "[[00_Project_HomePage]]"
  - "[[00_JS_Ecosystem_HomePage]]"
  - "[[00_NestJS_Ecosystem_HomePage]]"
  - "[[00_Deployment_HomePage]]"
  - "[[00_SQL_HomePage]]"
  - "[[00_Git_HomePage]]"
  - "[[00_Linux_HomePage]]"
  - "[[00_Docker_HomePage]]"
  - "[[00_Python_HomePage]]"
  - "[[00_Certifications_HomePage]]"
  - "[[00_Career_HomePage]]"
---

# Obsidian_study

백엔드·프론트엔드·인프라·데이터를 중심으로 학습한 내용을 정리한 **개인 Obsidian 지식 베이스**입니다.  
개념 정리 → 실습 → 트러블슈팅 → 배포까지 이어지는 흐름으로 기록합니다.

## 학습 방향

- 개념을 외우기보다 **동작 원리와 흐름** 중심으로 정리
- 실습 중 발생한 오류와 해결 과정을 트러블슈팅 형태로 기록
- 프론트(JS/TS/React/Next.js)와 백엔드(NestJS/Node.js)가 실제로 어떻게 연결되는지 위키링크로 묶어서 관리
- 배포·운영 관점(안정성, 확장성, 비용)까지 기록

## 학습 범위

| 분야 | 주요 주제 |
|------|-----------|
| **JS Ecosystem** | JavaScript, TypeScript, React, Next.js (HTML/CSS 포함) |
| **Backend** | Node.js, NestJS (Module, Auth, Prisma, Testing 등) |
| **Database** | SQL, PostgreSQL, MySQL |
| **Infrastructure** | Linux, Git, Docker, Deployment (CI/CD · 클라우드 배포) |
| **Languages** | Python |
| **Tools** | VS Code, Postman, Obsidian, Mac, DataGrip, Terminal |
| **Career** | 이력서 · 포트폴리오 · 자기소개서 |
| **Certification** | SQLD 취득 · DAsP 준비 중 |

## 저장소 구조

```txt
Obsidian_study/
├── 00_Inbox/                         # 임시 메모·아이디어
├── 10_Toolbox/                       # VS Code, Postman, Obsidian, Mac 등 개발 도구
│   ├── DataGrip/
│   ├── Mac/
│   ├── Obsidian/
│   ├── Postman/
│   ├── Snippets/
│   ├── Terminal/
│   └── VSCode/
├── 20_Wiki/                          # 핵심 학습 노트
│   ├── 22_Languages/
│   │   ├── JS_Ecosystem/             # JS · TS · React · Next.js (통합)
│   │   └── Python/
│   ├── 23_Backend/
│   │   └── NestJS_Ecosystem/         # NestJS · Node.js (통합)
│   ├── 24_Data_Processing/           # (예정)
│   ├── 25_Orchestration/             # (예정)
│   ├── 26_Infrastructure/
│   │   ├── Deployment/               # 배포 · CI/CD · 클라우드
│   │   ├── Docker/
│   │   ├── Linux/
│   │   └── git/
│   └── 27_Database/
│       └── SQL/
├── 30_Project/                       # 프로젝트별 노트 (music-community 등)
├── 40_Certifications(자격증)/        # 자격증 정리 (SQLD, DAsP)
└── 50_Career/                        # 이력서 · 포트폴리오 · 자소서
    ├── Portfolio.md
    └── 이력서.md
```

각 주제 묶음은 홈페이지 노트에서 목차 형태로 연결됩니다.

- `00_Toolbox_HomePage` — Obsidian · VS Code · Postman · Mac
- `00_Project_HomePage` — 프로젝트 (music-community 등)
- `00_JS_Ecosystem_HomePage` — JS · TS · React · Next.js
- `00_NestJS_Ecosystem_HomePage` — NestJS · NodeJS
- `00_Deployment_HomePage` — 배포 · CI/CD · 클라우드 인프라
- `00_SQL_HomePage` — SQL · PostgreSQL
- `00_Docker_HomePage` · `00_Git_HomePage` · `00_Linux_HomePage` — 인프라 기초
- `00_Python_HomePage` — Python
- `00_Certifications_HomePage` — 자격증
- `00_Career_HomePage` — 이력서 · 포트폴리오 · 자소서

## 특징

- **개념 → 실습 → 트러블슈팅** 순서로 정리
- Obsidian 위키링크(`[[노트명]]`)로 관련 주제를 클러스터 단위로 연결
- Postman·VS Code Debugger 등 실무 도구 사용법 병행

## Git에서 제외하는 항목 (`.gitignore`)

| 항목 | 이유 |
|------|------|
| `.obsidian/` | Obsidian vault 설정 (로컬 환경마다 다름) |
| `99_Assets(이미지&첨부파일저장소)/` | 이미지·첨부파일 저장소 |
| `*.png`, `*.jpg`, `*.svg` 등 이미지·바이너리 | 용량·바이너리 관리 회피 |
| `_Deploy/` | 재정리 중인 임시 배포 노트 폴더 |
| `.env`, `node_modules/`, `dist/`, `.next/` 등 | 환경 변수·빌드 산출물 |
| `.vscode/`, `.idea/` | 에디터/IDE 설정 |

Markdown 노트 위주의 학습 기록 저장소입니다.

---

> 지속적으로 업데이트하는 학습 로그입니다.
