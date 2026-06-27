---
aliases:
  - git
  - git commit
  - git push
  - git reset
  - git amend
tags:
  - Git
related:
  - "[[00_Git_HomePage]]"
  - "[[Linux_Shell_Script_Basics]]"
  - "[[Git_Concept]]"
---
# Git_Commands — 자주 쓰는 Git 명령어

## 한 줄 요약

```
버전 관리 + 협업 도구
커밋 수정 / 되돌리기 / 강제 푸시가 자주 필요함
```

---

---

# ① 기본 흐름

```bash
git init                       # 저장소 초기화 (처음 한 번)
git status                     # 변경 파일 확인
git add .                      # 전체 스테이징
git add 파일명                  # 특정 파일만
git commit -m "메시지"          # 커밋
git push origin main            # 원격 푸시
```

## git commit -am — add + commit 한 번에 ⭐️

```bash
# -a + -m 옵션 합친 단축 명령어
git commit -am "변경 내용 커밋"

# 풀어서 쓰면 이것과 동일
git add .
git commit -m "변경 내용 커밋"
```

```
-a (--all):  수정/삭제된 파일을 자동으로 스테이징
-m:          커밋 메시지 지정

⚠️ 중요한 제한:
  이미 추적 중인(tracked) 파일만 자동 스테이징
  새로 만든 파일(untracked)은 포함 안 됨
  → 새 파일은 반드시 git add 먼저

예시:
  hello.txt (이미 커밋된 파일) 수정 → commit -am 가능
  newfile.txt (처음 만든 파일)      → git add 먼저 필요
```

```bash
# 실전 패턴
echo "변경" >> hello.txt
git commit -am "Second change"   # 이미 추적 중인 파일 → -am 가능

touch newfile.txt
git commit -am "Add new file"    # ❌ newfile.txt 는 포함 안 됨
git add newfile.txt
git commit -m "Add new file"     # ✅ 새 파일은 add 필요
```

## git status 출력 해석 ⭐️

```bash
git status

# 1. 아무 파일도 없을 때
# On branch master
# No commits yet
# nothing to commit

# 2. 새 파일 만들었을 때
# Untracked files:
#   message.txt
# → Git 이 파일을 발견했지만 아직 추적 안 함

# 3. git add 후
# Changes to be committed:
#   new file: message.txt
# → Staging Area 에 올라감 (커밋 준비 완료)

# 4. git commit 후
# nothing to commit, working tree clean
# → 모든 변경사항이 저장됨
```

```
상태 3단계:
  Untracked     → git add → Staged → git commit → Committed
  (Git 모름)              (준비 완료)             (저장 완료)
```

## git log — 커밋 이력 확인 ⭐️

```bash
git log                        # 전체 이력 (q 로 종료)
git log --oneline              # 한 줄로 간결하게 ← 자주 씀
git log --oneline -5           # 최근 5개만
git log --oneline --graph      # 브랜치 그래프 포함
```

```
git log 출력:
  commit a1b2c3d4e5... (HEAD -> master)
  Author: Your Name <email>
  Date:   Mon Apr 20 10:00:00 2026

  Send a message to the future

  ↑ 커밋 해시 (고유 ID)
  ↑ HEAD → 현재 내가 있는 위치
  ↑ 브랜치 이름

q 를 눌러 로그 화면 종료
```

## git diff — 변경 사항 확인 ⭐️

```bash
# 워킹 디렉토리 vs 마지막 커밋 비교 (스테이징 전)
git diff

# 스테이징 영역 vs 마지막 커밋 비교 (커밋 전 최종 확인)
git diff --staged
git diff --cached   # --staged 와 동일

# 특정 파일만
git diff greet.js
git diff --staged greet.js
```

## diff 출력 읽는 법 ⭐️

```bash
git diff
# diff --git a/greet.js b/greet.js
# --- a/greet.js          ← 원본 (변경 전)
# +++ b/greet.js          ← 수정본 (변경 후)
# @@ -1,3 +1,7 @@         ← 변경 위치 (원본 1~3줄 → 수정 1~7줄)
#  function greet(name) {  ← 변경 없는 줄 (공백으로 시작)
# +function farewell(name) {   ← 추가된 줄 (+)
# + return 'Goodbye, ' + name;
# +}
```

```
기호 의미:
  ---    원본 파일 (a/)
  +++    수정된 파일 (b/)
  @@     변경 위치 정보
  +      추가된 줄 (초록색)
  -      삭제된 줄 (빨간색)
  공백   변경 없는 줄 (문맥용)

q 로 diff 뷰 종료
```

## 브랜치 비교 ⭐️

```bash
# 두 브랜치 비교
git diff master feature-branch
# "master 에는 없고 feature-branch 에만 있는 것" 을 + 로 표시

# 현재 브랜치 vs 다른 브랜치
git diff master
# 현재 브랜치(feature-branch)를 master 와 비교

# 특정 파일만 브랜치 비교
git diff master feature-branch -- numbers.js
#                              ↑ -- 로 파일명 구분 (브랜치명과 혼동 방지)
```

```
브랜치 비교 읽는 법:
  git diff A B
  - (삭제) = A 에는 있지만 B 에는 없는 것
  + (추가) = B 에만 있는 것

언제 쓰나:
  feature 브랜치 → main 에 PR 올리기 전 확인
  배포 전 staging vs production 비교
```

## git diff 언제 쓰나

```
git add 전   → git diff            워킹 디렉토리 변경 확인
git add 후   → git diff --staged   커밋 전 최종 확인
커밋 후      → git diff HEAD~1     직전 커밋과 비교
브랜치 비교  → git diff main 브랜치명
```

---

---

# ① -1 로컬 폴더 → GitHub 처음 올리기 ⭐️

```
새 프로젝트 폴더를 GitHub 에 처음 연결할 때
GitHub 에서 새 저장소 만든 뒤 아래 명령어 실행
```

```bash
# 로컬 폴더에서 Git 초기화
git init
git add .
git commit -m "first commit"
git branch -M main

# GitHub 원격 저장소 연결
git remote add origin https://github.com/사용자명/저장소명.git

# 처음 push (-u 로 upstream 설정)
git push -u origin main
```

```
-u (--set-upstream) 의미:
  처음 한 번만 붙이면
  이후부터는 git push 만 해도 됨 (origin main 생략 가능)

이미 git init 된 폴더라면:
  git init 건너뛰고
  git remote add origin ... 부터 시작
```

## 이미 로컬에 저장소가 있을 때

```bash
# GitHub 에서 새 저장소 만들고
git remote add origin https://github.com/사용자명/저장소명.git
git branch -M main
git push -u origin main
```

## 원격 저장소 확인 / 변경

```bash
git remote -v                                     # 현재 연결된 원격 확인
git remote set-url origin https://새_주소.git     # 원격 주소 변경
git remote remove origin                          # 원격 연결 제거
```

---

---

# ② commit --amend — 마지막 커밋 수정 ⭐️

```
방금 한 커밋을 수정하고 싶을 때
메시지 수정 / 파일 추가 / 코드 수정
→ 새 커밋을 만들지 않고 마지막 커밋을 덮어씀
```

```bash
# 커밋 메시지만 수정
git commit --amend -m "새 메시지"

# 파일 빠트렸을 때 — 추가 후 이전 커밋에 포함
echo "중요한 내용" >> hello.txt
git add hello.txt
git commit --amend -m "Initial commit with important note"
# → 이전 커밋이 새 내용 포함된 커밋으로 교체됨

# 파일 추가 후 메시지는 유지
git add 빠트린_파일.py
git commit --amend --no-edit
```

```
-m "메시지"     새 메시지로 교체
--no-edit       기존 메시지 유지

언제 쓰나:
  오타 있는 커밋 메시지 수정
  파일 빠트리고 커밋했을 때
  코드 작은 수정 후 이전 커밋에 포함

⚠️ push 한 커밋은 amend 하면 force push 필요
  혼자 쓰는 브랜치에서만 / 팀 공유 브랜치에서 금지
```

```bash
# 원격에 반영 (강제 푸시)
git push --force-with-lease   # 권장 — 원격 변경 있으면 거부
git push --force              # 위험 — 무조건 덮어씀
```

---

---

# ③ reset — 커밋 되돌리기

```
reset 종류:
  --soft   커밋만 취소, 변경사항 staged 상태 유지  ← 자주 씀
  --mixed  커밋 취소 + unstaged 상태 (기본값)
  --hard   커밋 취소 + 변경사항 전부 삭제 ⚠️
```

```bash
# 마지막 커밋 취소 (변경사항 유지)
git reset --soft HEAD~1

# 여러 개 취소
git reset --soft HEAD~3        # 최근 3개 커밋 취소

# 특정 커밋으로 되돌리기
git reset --soft 해시값         # git log 에서 해시 확인
```

```bash
# 원격에 반영
git push origin main --force
```

```
HEAD~1 vs 해시:
  HEAD~1    현재에서 1개 전 커밋
  HEAD~3    현재에서 3개 전 커밋
  abc1234   git log 에서 복사한 특정 커밋 해시

--soft 후 상태:
  취소된 커밋의 변경사항이 staged 로 남아있음
  → 다시 수정 후 커밋 가능
```

---

---

# ④ log — 커밋 이력 확인

```bash
git log                        # 전체 이력
git log --oneline              # 한 줄로 간결하게 ← 자주 씀
git log --oneline -5           # 최근 5개만
```

```
예시 출력:
  abc1234 (HEAD -> main) 세 번째 커밋
  def5678 두 번째 커밋
  ghi9012 첫 번째 커밋

→ reset --soft def5678 하면 "세 번째 커밋" 만 취소됨
```

---

---

# ⑤ 자주 쓰는 패턴

```bash
# 커밋 메시지 오타 수정
git commit --amend -m "올바른 메시지"
git push --force

# 파일 빠트리고 커밋했을 때
git add 빠트린_파일.py
git commit --amend --no-edit
git push --force

# 커밋 여러 개를 하나로 합치기 (squash)
git reset --soft HEAD~3        # 3개 취소
git commit -m "합친 커밋 메시지"
git push origin main --force

# 실수로 .env 커밋했을 때
git reset --soft HEAD~1        # 마지막 커밋 취소
# .gitignore 에 .env 추가
git add .gitignore
git commit -m "fix: remove .env"
git push origin main --force
```

---

---

# ⑥ 기타 유용한 명령어

```bash
# 원격 저장소 확인
git remote -v

# 브랜치 목록
git branch
git branch -a                  # 원격 포함

# 변경사항 임시 저장 (stash)
git stash                      # 임시 저장
git stash pop                  # 복원

# 특정 파일 변경사항 되돌리기
git checkout -- 파일명

# 스테이징 취소
git restore --staged 파일명
```

---

---

# 한눈에 정리

|상황|명령어|
|---|---|
|새 폴더 GitHub 에 처음 올리기|`git init` → `git remote add origin` → `git push -u origin main`|
|이미 있는 폴더 GitHub 연결|`git remote add origin` → `git push -u origin main`|
|마지막 커밋 메시지 수정|`git commit --amend -m "새 메시지"`|
|파일 빠트리고 커밋|`git add 파일` + `git commit --amend --no-edit`|
|마지막 커밋 취소 (변경사항 유지)|`git reset --soft HEAD~1`|
|특정 커밋으로 되돌리기|`git reset --soft 해시값`|
|강제 푸시|`git push --force-with-lease`|
|커밋 이력 확인|`git log --oneline`|