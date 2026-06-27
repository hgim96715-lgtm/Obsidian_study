---
aliases:
  - git remote
  - git fetch
  - git pull
  - git clone
  - submodule
tags:
  - Git
related:
  - "[[00_Git_HomePage]]"
  - "[[Git_Commands]]"
  - "[[Git_Branch]]"
---

# Git_Remote — 원격 저장소

## 한 줄 요약

```txt
remote = 원격 저장소 연결 관리
clone  = 원격 저장소 복제
fetch  = 원격 변경사항 가져오기 (병합 안 함)
pull   = fetch + merge 한번에
push   = 로컬 변경사항 원격에 올리기
```

---

---

#  remote — 원격 저장소 관리

```bash
# 원격 저장소 목록 확인
git remote -v
# origin  https://github.com/user/repo.git (fetch)
# origin  https://github.com/user/repo.git (push)

# 원격 저장소 추가
git remote add origin https://github.com/user/repo.git
#              ↑이름   ↑ URL

# 원격 저장소 URL 변경
git remote set-url origin https://github.com/user/new-repo.git

# 원격 저장소 제거
git remote remove origin

# 원격 저장소 이름 변경
git remote rename origin upstream
```

```txt
origin:
  원격 저장소의 기본 이름 (관례)
  git clone 하면 자동으로 origin 으로 설정됨
  여러 원격 저장소 추가 가능 (origin / upstream 등)
```

---

---

# clone — 원격 저장소 복제

```bash
# HTTPS 로 복제
git clone https://github.com/user/repo.git

# 특정 폴더명으로 복제
git clone https://github.com/user/repo.git my-folder

# 특정 브랜치만
git clone -b develop https://github.com/user/repo.git

# SSH 로 복제
git clone git@github.com:user/repo.git
```

```txt
clone vs init:
  clone  원격 저장소를 그대로 복제 (remote 자동 설정)
  init   빈 저장소 새로 시작 (remote 직접 설정 필요)
```

## --depth — 히스토리 제한 복제 

```bash
# 최신 커밋 1개만 (히스토리 없이)
git clone --depth=1 https://github.com/github/gitignore

# 최근 5개 커밋만
git clone --depth=5 https://github.com/user/repo.git
```

```txt
--depth 옵션:
  전체 히스토리 없이 최신 N개 커밋만 복제
  shallow clone (얕은 복제) 이라고 부름

왜 쓰나:
  히스토리가 수천 개인 대형 저장소 → 전체 clone 느림
  CI/CD 파이프라인 / 코드만 필요할 때
  --depth=1 → 최신 상태만 빠르게 받기

단점:
  git log 로 전체 이력 볼 수 없음
  이전 커밋으로 reset 불가
  → 개발 목적이면 depth 없이 전체 clone
```

---
---
# submodule — 저장소 안에 저장소 ⭐️

```bash
# 외부 저장소를 서브모듈로 추가
git submodule add https://github.com/labex-labs/git-playground.git ./git-playground
#                      ↑ 추가할 저장소 URL                              ↑ 로컬 경로

# 서브모듈 목록 확인
git submodule status

# 서브모듈 초기화 + 업데이트 (clone 후 서브모듈 없을 때)
git submodule update --init --recursive
```

```txt
submodule 이란:
  저장소 안에 다른 Git 저장소를 포함시키는 기능
  외부 라이브러리 / 공통 코드를 별도 저장소로 관리하면서
  내 프로젝트에 포함할 때 사용

  git submodule add 실행 후:
    .gitmodules 파일 생성 (서브모듈 경로·URL 기록)
    git-playground/ 폴더 생성 (서브모듈 코드)
    → git add + commit 해야 저장됨

처음 봤을 때 당황하는 이유:
  git clone 으로 받은 저장소에 서브모듈 있으면
  서브모듈 폴더가 비어있음
  → git submodule update --init --recursive 로 초기화 필요
```

```bash
# 서브모듈 있는 저장소 clone 후 초기화
git clone https://github.com/user/repo.git
cd repo
git submodule update --init --recursive
```

---

---

# fetch — 가져오기 (병합 안 함) ⭐️

```bash
# 원격 변경사항 가져오기 (로컬 브랜치 변경 없음)
git fetch origin

# 특정 브랜치만
git fetch origin main

# 가져온 내용 확인
git log origin/main --oneline

# 가져온 내용을 현재 브랜치에 병합
git merge origin/main
```

```txt
fetch 언제 쓰나:
  원격에 무슨 변경이 있는지 먼저 확인
  직접 merge 할 시점 선택하고 싶을 때

fetch vs pull:
  fetch  가져오기만 (내 작업 영향 없음)
  pull   가져오기 + 자동 merge
  → 안전하게: fetch → 확인 → merge
  → 빠르게:  pull
```

---

---

# pull — 가져오기 + 병합

```bash
# 원격 main 가져오기 + 현재 브랜치에 merge
git pull origin main

# 현재 브랜치 추적 원격 기준으로
git pull

# rebase 방식으로 pull
git pull --rebase origin main
```

```txt
pull = fetch + merge
pull --rebase = fetch + rebase (히스토리 깔끔)

협업 시 권장:
  git pull --rebase  → 불필요한 merge 커밋 방지
```

---

---

#  push — 원격에 올리기

```bash
# 기본 push
git push origin main

# 최초 push (upstream 설정)
git push -u origin main
#         ↑ -u = upstream 설정
#           이후로는 git push 만 해도 됨

# 강제 push (주의!)
git push --force origin main
git push -f origin main

# 태그 push
git push origin --tags
```

```txt
push 거부될 때:
  원격에 내가 없는 커밋 있음 → pull 먼저

force push 주의:
  협업 브랜치에서 force push → 동료 히스토리 망가짐
  혼자 쓰는 브랜치에서만 / reset 후 정리할 때만
```

---

---

# 명령어 한눈에

|명령어|역할|
|---|---|
|`git remote -v`|원격 저장소 목록|
|`git remote add origin URL`|원격 저장소 추가|
|`git clone URL`|원격 저장소 복제|
|`git fetch origin`|가져오기 (병합 안 함)|
|`git pull origin main`|가져오기 + 병합|
|`git pull --rebase`|가져오기 + rebase|
|`git push origin main`|올리기|
|`git push -u origin main`|upstream 설정 + 올리기|