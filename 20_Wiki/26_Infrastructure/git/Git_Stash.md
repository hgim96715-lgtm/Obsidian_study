---
aliases:
  - git stash
  - 임시 저장
  - stash
tags:
  - Git
related:
  - "[[00_Git_HomePage]]"
  - "[[Git_Undo]]"
  - "[[Git_Branch]]"
  - "[[Git_Commands]]"
---

# Git_Stash — 임시 저장

## 한 줄 요약

```
작업 중인 변경사항을 임시로 보관하고
나중에 꺼내 쓰는 서랍 같은 기능
커밋하기 애매한 상태에서 브랜치 이동할 때 사용
```

---

---

# ① git stash — 기본 저장

```bash
# 기본 stash (tracked 파일만)
git stash

# 추적 안 된 파일(untracked)도 포함
git stash -u
git stash --include-untracked

# 메시지 붙여서 저장 (권장) ⭐️
git stash push -m "로그인 기능 작업 중"
git stash push -u -m "미완성 기능 + 새 파일 포함"
```

```
기본 git stash 가 저장하는 것:
  ✅ tracked 파일의 변경사항 (이미 git add 된 파일)
  ✅ staged 변경사항
  ❌ untracked 파일 (새로 만든 파일) → -u 옵션 필요

git stash vs git stash push:
  git stash        = git stash push 단축형 (동일)
  git stash push   = 명시적 / -m 메시지 붙일 때 권장
```

---

---

# ② stash 목록 확인

```bash
# 목록 보기
git stash list
# stash@{0}: On main: 세 번째 변경 사항   ← 가장 최근
# stash@{1}: On main: 두 번째 변경 사항
# stash@{2}: On main: 첫 번째 변경 사항   ← 가장 오래됨

# 특정 stash 변경 내용 확인
git stash show stash@{1}         # 변경된 파일 목록
git stash show -p stash@{1}      # 전체 diff 확인
```

```
stash 번호 규칙:
  stash@{0} = 가장 최근 저장 (Last In)
  stash@{1} = 그 전
  숫자가 클수록 오래된 stash

q 로 목록 화면 종료
```

---

---

# ③ stash 꺼내기 — apply vs pop ⭐️

```bash
# apply — 적용 후 stash 목록에 유지
git stash apply           # 가장 최근 stash 적용
git stash apply stash@{1} # 특정 stash 적용

# pop — 적용 후 stash 목록에서 삭제
git stash pop             # 가장 최근 stash 적용 + 삭제
git stash pop stash@{1}   # 특정 stash 적용 + 삭제
```

|명령어|적용|stash 목록|
|---|---|---|
|`apply`|✅|유지|
|`pop`|✅|삭제|

```
언제 apply, 언제 pop:
  apply → 같은 stash 를 여러 브랜치에 적용할 때
  pop   → 일반적인 경우 (적용 후 불필요하므로 삭제)
  → 대부분 pop 사용
```

---

---

# ④ stash 삭제 & 정리

```bash
# 특정 stash 삭제
git stash drop stash@{2}

# 전체 stash 삭제
git stash clear
# ⚠️ 되돌릴 수 없음 / 신중하게 사용
```

---

---

# ⑤ stash 에서 브랜치 생성 ⭐️

```bash
# stash 작업을 새 브랜치로 분리
git stash branch feature-branch

# 자동으로:
# 1. feature-branch 새 브랜치 생성
# 2. stash 한 시점으로 이동
# 3. stash 내용 적용
# 4. stash 에서 제거 (pop 처럼)
```

## 브랜치 생성 후 master 병합 ⭐️


```bash
# 1. stash → 새 브랜치 생성 (자동으로 stash 내용 적용됨)
git stash branch feature-branch

# 2. 작업 마무리 후 커밋
git add README.md notes.txt
git commit -m "Start new feature"

# 3. master 의 최신 변경사항을 feature-branch 에 반영
git merge master

# 4. 필요하면 main 으로 돌아가서 최종 병합
git switch master
git merge feature-branch
```

```
왜 git merge master 가 필요한가:
  stash 는 작업 중에 임시로 저장한 것
  그 사이 master 에 다른 변경사항이 생겼을 수 있음
  → feature-branch 에 master 최신 내용을 반영해야
    충돌 없이 나중에 master 로 병합 가능

흐름 정리:
  master  ──────●────────────●
                              ↑ merge master
  feature       └────●───────●
                  stash 작업  커밋 후 master 반영
```

```
언제 쓰나:
  stash 했는데 생각보다 작업이 커질 때
  → 새 브랜치로 분리해서 계속 작업
  → merge master 로 최신 상태 동기화
  → 별도 PR 로 올릴 때
```

---

---

# ⑥ 실전 워크플로우

## 급한 버그 수정 패턴 ⭐️

```bash
# 1. 기능 개발 중...
echo "feature code" >> app.py

# 2. 갑자기 긴급 버그 수신
git stash push -m "로그인 기능 작업 중"

# 3. main 으로 이동해서 버그 수정
git switch main
# 버그 수정 작업...
git commit -m "fix: 긴급 버그 수정"

# 4. 원래 작업으로 복귀
git switch feature/login
git stash pop
# 이어서 작업
```

## 여러 stash 관리

```bash
# 작업 단위로 메시지 붙여 저장
git stash push -m "API 연결 작업"
git stash push -u -m "새 컴포넌트 + 테스트 파일"
git stash push -m "DB 스키마 수정"

git stash list
# stash@{0}: DB 스키마 수정
# stash@{1}: 새 컴포넌트 + 테스트 파일
# stash@{2}: API 연결 작업

# 원하는 것만 꺼내기
git stash apply stash@{2}   # API 연결 작업 적용
```

---

---

# 명령어 한눈에

|명령어|역할|
|---|---|
|`git stash`|기본 저장 (tracked만)|
|`git stash -u`|저장 (untracked 포함)|
|`git stash push -m "메시지"`|메시지 붙여 저장 ⭐️|
|`git stash list`|목록 확인|
|`git stash show stash@{N}`|변경 파일 목록|
|`git stash show -p stash@{N}`|전체 diff|
|`git stash apply`|적용 (목록 유지)|
|`git stash pop`|적용 + 삭제 ⭐️|
|`git stash drop stash@{N}`|특정 stash 삭제|
|`git stash clear`|전체 삭제 ⚠️|
|`git stash branch 브랜치명`|stash → 브랜치 생성|

---

---

# 자주 하는 실수

|실수|원인|해결|
|---|---|---|
|새 파일이 stash 안 됨|기본은 untracked 제외|`git stash -u`|
|stash 했는데 어디 저장됐는지 모름|메시지 없음|`git stash push -m "설명"`|
|apply 했는데 stash 가 남아있음|apply 는 유지|완료 후 `git stash drop`|
|stash clear 후 복구 못 함|되돌릴 수 없음|clear 전 신중하게|