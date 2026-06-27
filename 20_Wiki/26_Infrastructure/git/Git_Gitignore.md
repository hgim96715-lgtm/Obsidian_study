---
aliases:
  - gitignore
  - git 제외
  - 파일 무시
tags:
  - Git
related:
  - "[[00_Git_HomePage]]"
  - "[[Git_Commands]]"
  - "[[Git_Concept]]"
---

# Git_Gitignore — 파일 제외 설정

## 한 줄 요약

```txt
.gitignore = Git 이 추적하지 않을 파일 목록
로그 파일 / 환경변수 / 빌드 결과물 / 의존성 폴더 제외
```

---

---

# ① .gitignore 기본 사용

```bash
# .gitignore 파일 생성
echo "*.log" > .gitignore        # .log 파일 전부 무시
echo "debug.log" >> .gitignore   # 특정 파일 무시

# Git 에 추가 후 커밋
git add .gitignore
git commit -m "Add .gitignore"

# 확인 — 무시된 파일은 git status 에 안 보임
echo "This is a log file" > debug.log
git status    # debug.log 가 나타나지 않음 ✅
```

---

---

# ② .gitignore 패턴 문법 ⭐️

```txt
# 주석

*.log          확장자가 .log 인 파일 전부
*.txt          확장자가 .txt 인 파일 전부
debug.log      특정 파일명
/debug.log     루트 디렉토리의 debug.log 만

build/         build 디렉토리 전체
**/logs        어느 경로에 있든 logs 폴더

*.log          무시
!important.log 무시 제외 (! = 예외)

doc/*.txt      doc 폴더 바로 아래 .txt 만
doc/**/*.pdf   doc 폴더 아래 모든 .pdf
```

## 실전 예시

```gitignore
# 로그 파일
*.log
logs/

# 환경변수 / 보안
.env
.env.local
*.pem
secrets.json

# Python
__pycache__/
*.pyc
*.pyo
.venv/
venv/

# 의존성
node_modules/

# 빌드 결과물
dist/
build/
*.egg-info/

# IDE
.idea/
.vscode/
*.DS_Store

# Airflow
airflow.db
airflow.cfg
```

---

---

# ③ 이미 올라간 파일 제거 ⭐️

```txt
.gitignore 추가 전에 이미 커밋된 파일은
.gitignore 에 넣어도 계속 추적됨

→ git rm --cached 로 추적 해제 필요
  (실제 파일은 삭제 안 됨)
```

```bash
# 추적 해제 (파일은 로컬에 유지)
git rm --cached 파일명
git rm --cached debug.log

# 폴더 전체 추적 해제
git rm --cached -r 폴더명/
git rm --cached -r __pycache__/

# 추적 해제 후 커밋
git add .gitignore
git commit -m "Remove tracked files now in .gitignore"
```

```txt
--cached 없으면:
  git rm 파일명  → 파일 자체도 삭제됨 ⚠️
  반드시 --cached 옵션으로 추적만 해제
```

## 이미 올라간 .env 처리

```bash
# .env 추적 해제
git rm --cached .env

# .gitignore 에 추가
echo ".env" >> .gitignore

# 커밋
git add .gitignore
git commit -m "Remove .env from tracking"

# .env 는 로컬에 그대로 있음 ✅
# GitHub 에는 더 이상 올라가지 않음 ✅
```

---

---

# ④ .gitignore 확인 명령어

```bash
# 무시된 파일 목록 확인
git status --ignored

# 특정 파일이 왜 무시되는지 확인
git check-ignore -v debug.log
# .gitignore:1:*.log  debug.log
#   ↑ 파일명  ↑ 줄번호  ↑ 패턴

# 특정 파일이 무시되는지만 확인
git check-ignore debug.log
# 출력 있으면 무시 중 / 없으면 추적 중
```

---

---

# ⑤ 글로벌 .gitignore — 모든 저장소에 적용

```bash
# 전역 .gitignore 파일 생성
nano ~/.gitignore_global

# Git 에 등록
git config --global core.excludesFile ~/.gitignore_global
```

```gitignore
# ~/.gitignore_global 예시
.DS_Store        # macOS
Thumbs.db        # Windows
.idea/           # IntelliJ
.vscode/         # VSCode
*.log
```

```txt
로컬 .gitignore vs 전역 .gitignore:
  로컬  → 해당 프로젝트에만 적용 / 팀원과 공유
  전역  → 내 컴퓨터 모든 저장소에 적용 / 개인 설정
```

---

---

# 명령어 한눈에

|명령어|역할|
|---|---|
|`echo "*.log" > .gitignore`|.gitignore 생성|
|`git status --ignored`|무시된 파일 목록|
|`git check-ignore -v 파일명`|무시 이유 확인|
|`git rm --cached 파일명`|이미 추적 중인 파일 추적 해제|
|`git rm --cached -r 폴더/`|폴더 전체 추적 해제|