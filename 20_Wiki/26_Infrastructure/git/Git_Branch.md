---
aliases:
  - git branch
  - 브랜치
  - git merge
  - git switch
tags:
  - Git
related:
  - "[[00_Git_HomePage]]"
  - "[[Git_Concept]]"
  - "[[Git_Commands]]"
  - "[[Git_Undo]]"
---

# Git_Branch — 브랜치 & 병합

## 한 줄 요약

```txt
브랜치 = 메인 코드 건드리지 않고 새 기능 개발하는 별도 시간선
main 에서 브랜치 → 작업 → 다시 main 에 병합
```

---

---

# ① 브랜치 기본 명령어

## 브랜치 목록 확인

```bash
git branch          # 로컬 브랜치 목록
git branch -a       # 원격 포함 전체 목록
git branch --merged # 현재 브랜치에 병합된 브랜치만

# * = 현재 내가 있는 브랜치
# * master
#   feature-dimension
```

## 브랜치 생성

```bash
git branch feature-dimension        # 브랜치 생성 (이동 안 함)
git branch hotfix/login-bug         # 슬래시로 계층 구조 가능
```

## 브랜치 이동

```bash
# 구버전 방식
git checkout feature-dimension

# 신버전 방식 (Git 2.23 이상 권장) ⭐️
git switch feature-dimension
```

## 생성 + 이동 한 번에 ⭐️

```bash
# 구버전
git checkout -b feature-dimension

# 신버전 (권장)
git switch -c feature-dimension
#          ↑ -c = create
```

```txt
checkout vs switch:
  git checkout  브랜치 이동 + 파일 복구 등 여러 기능
  git switch    브랜치 이동 전용 (더 명확 / 권장)

  둘 다 결과 동일 / switch 가 의도를 더 명확히 전달
```

---

---

# ② 브랜치 작업 흐름 ⭐️

```bash
# 1. main 에서 새 브랜치 생성 + 이동
git switch -c feature/새기능

# 2. 작업 후 커밋
git add .
git commit -m "feat: 새 기능 추가"

# 3. main 으로 복귀
git switch main

# 4. 브랜치 병합
git merge feature/새기능

# 5. 브랜치 삭제
git branch -d feature/새기능
```

---

---

# ③ git merge — 병합

```bash
# 현재 브랜치에 다른 브랜치를 병합
git switch main
git merge feature-dimension
# → feature-dimension 의 변경사항이 main 에 합쳐짐
```

## merge 출력 해석

```txt
Fast-forward:
  main 에 새 커밋이 없어서 그냥 앞으로 이동
  가장 깔끔한 병합

Merge commit:
  양쪽 모두 커밋이 있어서 합치는 커밋 생성
  "Merge branch 'feature-...' into main"
```

## 브랜치 간 차이 확인

```bash
# 현재 브랜치와 main 의 차이
git diff main

# 두 브랜치 비교
git diff main..feature-dimension
```

---

---

# ④ 브랜치 삭제

```bash
# 안전한 삭제 (-d) — 병합된 브랜치만 삭제
git branch -d feature-dimension
# 병합 안 됐으면 에러 → 실수 방지 안전장치

# 강제 삭제 (-D) — 병합 여부 무관
git branch -D feature-dimension
# ⚠️ 병합 안 된 커밋도 사라짐 / 신중하게 사용
```

```txt
-d vs -D:
  -d  병합된 것만 삭제 (안전) ← 주로 이걸 쓸 것
  -D  무조건 삭제 (위험)
  
병합 완료 확인:
  git branch --merged  → 목록에 있으면 -d 가능
```

---

---

# ⑤ 브랜치 네이밍 컨벤션

```txt
feature/기능명    새 기능 개발
fix/버그명        버그 수정
hotfix/긴급수정   긴급 패치
refactor/내용     코드 개선
chore/작업        설정 변경 / 패키지 업데이트
release/버전      배포 준비

예시:
  feature/user-login
  fix/payment-null-error
  hotfix/security-patch
```

---
---
# ⑥ cherry-pick — 특정 커밋만 가져오기 ⭐️

```txt
다른 브랜치에서 특정 커밋 하나만 현재 브랜치에 적용
전체 merge 없이 원하는 것만 쏙 뽑아옴
```

```bash
# 다른 브랜치의 최신 커밋 가져오기
git cherry-pick feature-branch

# 특정 커밋 해시로 가져오기
git log feature-branch --oneline   # 해시 확인
git cherry-pick abc1234             # 해당 커밋만 적용

# 결과 확인
git log --oneline
```

```txt
cherry-pick 의 특징:
  원본 커밋을 이동/복사가 아닌 새 커밋 생성
  커밋 내용은 같지만 해시값은 달라짐

언제 쓰나:
  특정 버그 수정만 다른 브랜치에 적용
  release 브랜치에 긴급 패치 적용

⚠️ 주의:
  두 브랜치 코드 차이가 크면 충돌 발생 가능
  많이 쓰면 히스토리 파악 어려워짐
  전체 merge 가 맞으면 cherry-pick 대신 merge 사용
```

---
---
# ⑦ rebase -i — 커밋 히스토리 정리 ⭐️

```txt
커밋 히스토리를 재작성하는 강력한 도구
push 전 로컬 커밋 정리에 사용
⚠️ 이미 push 한 커밋 / 공유 브랜치에서 절대 금지
```

## 기본 사용

```bash
# 최근 3개 커밋 대화형 편집
git rebase -i HEAD~3

# 처음부터 전체 (루트 리베이스 — 실험용만)
git rebase -i --root
```

## 에디터에서 쓸 수 있는 명령어

|명령어|축약|동작|
|---|---|---|
|`pick`|`p`|그대로 사용 (기본)|
|`reword`|`r`|커밋 메시지만 수정|
|`squash`|`s`|이전 커밋과 합치기 (메시지 합침)|
|`fixup`|`f`|이전 커밋과 합치기 (메시지 버림)|
|`edit`|`e`|커밋 내용 수정|
|`drop`|`d`|커밋 삭제|


```bash
# 에디터 예시 (오래된 것이 위)
pick abc111 First change
squash def222 Second change   ← First 와 합치기
reword ghi333 Third change    ← 메시지만 수정
```

```txt
저장 후 닫으면 순서대로 처리됨
에디터 종료:
  Vim:  Esc → :wq → Enter
  nano: Ctrl+X → Y → Enter

문제 생기면 언제든 취소:
  git rebase --abort
```

## 언제 쓰나

```txt
push 전 정리:
  "wip", "fix typo" 같은 작은 커밋 squash 로 합치기
  커밋 메시지 전체 정리 (reword)

⚠️ 절대 금지:
  이미 push 한 커밋
  팀원과 공유 중인 브랜치
  → 히스토리 재작성 → 다른 사람 커밋과 충돌
```

---

---

# 명령어 한눈에

|명령어|역할|
|---|---|
|`git branch`|브랜치 목록|
|`git branch 이름`|브랜치 생성|
|`git switch 이름`|브랜치 이동|
|`git switch -c 이름`|생성 + 이동 ⭐️|
|`git checkout -b 이름`|생성 + 이동 (구버전)|
|`git merge 브랜치`|병합|
|`git branch -d 이름`|브랜치 삭제 (안전)|
|`git branch -D 이름`|브랜치 강제 삭제|
|`git branch --merged`|병합된 브랜치 목록|
|`git branch -a`|원격 포함 전체 목록|
|`git diff main`|현재 브랜치 vs main 차이|

---

---

# 자주 하는 실수

|실수|원인|해결|
|---|---|---|
|main 에서 작업함|브랜치 이동 안 함|작업 전 `git switch -c 브랜치명`|
|merge 후 브랜치 안 지움|깜빡|`git branch --merged` 확인 후 `-d`|
|`-D` 로 잘못 삭제|실수|`git reflog` 로 복구 가능|
|checkout 으로 파일 복구됨|checkout 의 다중 기능 때문|이동은 `switch` / 파일 복구는 `restore`|