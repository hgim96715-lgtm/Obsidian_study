---
aliases:
  - git config
  - git 설정
  - git 설치
tags:
  - Git
related:
  - "[[00_Git_HomePage]]"
  - "[[Git_Concept]]"
  - "[[Git_Commands]]"
  - "[[Git_Remote]]"
---

# Git_Install_Setup — 설치 & 초기 설정

## 한 줄 요약

```
git config = Git 의 제어판
--global   = 이 컴퓨터 전체에 적용
(로컬)     = 이 저장소에만 적용
```

---

---

# Git 설치

```bash
# Ubuntu / Debian
sudo apt install git -y

# 버전 확인
git --version
# git version 2.43.0
```

---

---

# git init — 저장소 초기화

```bash
mkdir my-project
cd my-project
git init
# Initialized empty Git repository in .../my-project/.git/
```

```
.git 폴더가 생기면 Git 저장소. ls-la로 확인 [[Linux_Directory_Commands]] 참고
.git 폴더 안에 모든 이력이 저장됨
절대 .git 폴더 직접 수정하지 말 것
```

---

---

#  git config — 설정 확인 & 변경 ⭐️

## 설정 확인

```bash
# 전체 설정 목록
git config --list

# 특정 값만 확인
git config user.name
git config user.email
git config core.editor
```

## 3가지 설정 범위

```
--system   시스템 전체 (모든 사용자)   /etc/gitconfig
--global   현재 사용자 전체           ~/.gitconfig
(없음)     현재 저장소만              .git/config

우선순위: 로컬 > global > system
→ 로컬 설정이 global 설정을 덮어씀
```

```bash
# global 설정 파일 직접 보기
cat ~/.gitconfig
```

---

---

#  사용자 정보 설정 ⭐️

```bash
# 이름 / 이메일 설정 (커밋에 기록됨)
git config --global user.name "Your Name"
git config --global user.email "your.email@example.com"

# 확인
git config user.name
git config user.email
```

```
사용자 정보가 왜 필요한가:
  커밋할 때마다 "누가 만들었는지" 기록됨
  git log 에서 Author 로 표시
  협업 시 팀원들이 누구의 커밋인지 확인 가능

--global 없이 설정하면:
  현재 저장소에만 적용
  다른 프로젝트는 global 설정 사용
```

## 저장소별 다른 이름 설정 (로컬)

```bash
cd ~/project/git-config-lab

# 이 저장소에서만 다른 이름 사용
git config user.name "Lab User"

# 확인 — 로컬 설정이 우선
git config user.name         # "Lab User"  (로컬)
git config --global user.name  # "Your Name" (전역)
```

---

---

#  에디터 설정

```bash
# nano 로 설정 (초보자 친화적)
git config --global core.editor nano

# vim 으로 설정
git config --global core.editor vim

# VSCode 로 설정
git config --global core.editor "code --wait"

# 확인
git config core.editor
```

```
에디터 언제 쓰나:
  git commit  → 메시지 없이 실행 시 에디터 열림
  git rebase  → 대화형 작업 시
  git merge   → 충돌 해결 시

nano 사용법:
  Ctrl+X → 종료
  Y      → 저장
  Enter  → 확인
```

---

---

#  색상 설정

```bash
# 터미널 출력에 색상 사용
git config --global color.ui auto

# 확인
git config color.ui
```

```
auto:
  터미널 출력 → 색상 사용 (파란색 브랜치, 빨간색 삭제 등)
  파일/파이프로 보낼 때 → 일반 텍스트

git diff / git log / git status 가 훨씬 읽기 쉬워짐
```

---

---

#  줄바꿈 설정 — core.autocrlf

```bash
# macOS / Linux
git config --global core.autocrlf input

# Windows
git config --global core.autocrlf true

# 확인
git config core.autocrlf
```

```
왜 필요한가:
  Windows: 줄바꿈 = CRLF (\r\n)
  Linux/macOS: 줄바꿈 = LF (\n)

  다른 OS 팀원과 협업 시 줄바꿈 문자 충돌 발생
  → autocrlf 설정으로 자동 변환

input (macOS/Linux):
  저장소에 넣을 때: CRLF → LF 변환
  가져올 때: 변환 없음

true (Windows):
  저장소에서 가져올 때: LF → CRLF
  저장소에 넣을 때: CRLF → LF
```

---

---

#  alias — 단축키 설정 ⭐

## alias 구조 이해

```
git config --global alias.별칭 "실제명령어"
                          ↑
                    alias. 뒤에 내가 쓸 단축키 이름

등록:  git config --global alias.st status
사용:  git st   ← git status 와 완전히 동일

규칙:
  alias.[별칭]  → 내가 정하는 이름 (짧을수록 좋음)
  실제 명령어   → git 뒤에 오는 부분만 적음
    status    → alias.st
    checkout  → alias.co
    branch    → alias.br
    commit    → alias.ci
```
️
## 자주 쓰는 단축키 설정


```bash
# 자주 쓰는 명령어 단축키
git config --global alias.st status
git config --global alias.co checkout
git config --global alias.br branch
git config --global alias.ci commit

# 사용
git st       # git status 와 동일
git co main  # git checkout main 과 동일
git br       # git branch 와 동일
```

## 유용한 log 단축키

```bash
git config --global alias.lg "log --color --graph --pretty=format:'%Cred%h%Creset -%C(yellow)%d%Creset %s %Cgreen(%cr) %C(bold blue)' --abbrev-commit"

# 사용
git lg
# 색상 + 그래프 + 브랜치 표시된 한 줄 로그
```

```bash
긴 명령어를 통째로 별칭으로 등록 가능
따옴표로 감싸면 여러 옵션 포함 가능
git lg = git log --color --graph ... 전체
```

## alias 확인

```bash
# 특정 alias 확인
git config --global alias.st    # status

# 전체 alias 목록
git config --list | grep alias
# alias.st=status
# alias.co=checkout
# alias.lg=log --color ...

# ~/.gitconfig 에서 직접 확인
cat ~/.gitconfig
# [alias]
#   st = status
#   co = checkout
#   lg = log --color ...
```

## alias 삭제

```bash
git config --global --unset alias.st
```

---

---

#  전체 설정 파일 직접 편집

```bash
# ~/.gitconfig 직접 편집
git config --global --edit
# 또는
nano ~/.gitconfig
```

```
~/.gitconfig 예시:
  [user]
    name = Your Name
    email = your.email@example.com
  [core]
    editor = nano
    autocrlf = input
  [color]
    ui = auto
  [alias]
    st = status
    lg = log --color --graph ...
```

---

---

# 설정 한눈에

|명령어|역할|
|---|---|
|`git config --list`|전체 설정 확인|
|`git config user.name`|특정 설정 확인|
|`git config --global user.name "이름"`|이름 설정|
|`git config --global user.email "이메일"`|이메일 설정|
|`git config --global core.editor nano`|에디터 설정|
|`git config --global color.ui auto`|색상 설정|
|`git config --global core.autocrlf input`|줄바꿈 설정|
|`git config --global alias.st status`|단축키 설정|
|`git config --global --edit`|설정 파일 직접 편집|

---

---

# 자주 하는 실수

|실수|원인|해결|
|---|---|---|
|커밋 후 Author 가 이상함|user.name/email 설정 안 함|`git config --global user.name` 설정|
|git commit 에 에디터 안 열림|core.editor 미설정|`git config --global core.editor nano`|
|줄바꿈 문자 충돌|autocrlf 미설정|`git config --global core.autocrlf input`|
|로컬 vs 글로벌 혼동|범위 이해 부족|로컬 > global > system 우선순위|