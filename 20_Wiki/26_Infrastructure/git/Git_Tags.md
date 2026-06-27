---
aliases:
  - git tag
  - 태그
  - 버전 태그
  - 경량 태그
  - 주석 태그
tags:
  - Git
related:
  - "[[00_Git_HomePage]]"
  - "[[Git_Commands]]"
  - "[[Git_Branch]]"
---

# Git_Tags — 태그

## 한 줄 요약

```
특정 커밋에 이름표를 붙이는 기능
버전 릴리스 표시 (v1.0 / v2.3.1) 에 주로 사용
브랜치와 달리 고정됨 — 가리키는 커밋이 절대 바뀌지 않음
```

---

---

# ① 태그 2가지 종류

```
경량 태그 (Lightweight)
  특정 커밋을 가리키는 포인터
  메시지 / 작성자 / 날짜 없음
  임시적 / 개인 용도

주석 태그 (Annotated) ⭐️
  Git 데이터베이스에 전체 객체로 저장
  작성자 이름 / 이메일 / 날짜 / 메시지 포함
  GPG 서명 가능
  → 공개 릴리스에 사용
```

---

---

# ② 태그 생성

## 경량 태그

```bash
# 현재 커밋에 태그
git tag v0.1

# 특정 커밋에 태그 (해시 지정)
git log --oneline         # 커밋 해시 확인
git tag v0.0.1 aad8308   # 해당 커밋에 태그
```

## 주석 태그 ⭐️

```bash
# -a = annotated / -m = 메시지
git tag -a v1.0 -m "First major release"

# 특정 커밋에 주석 태그
git tag -a v1.0 -m "First major release" aad8308

# 태그 상세 정보 확인
git show v1.0
# 태그 작성자 / 날짜 / 메시지 + 커밋 정보 출력
```

---

---

# ③ 태그 목록 확인

```bash
# 전체 목록
git tag
# v0.0.1
# v0.1
# v1.0

# 패턴으로 필터링
git tag -l "v1.*"    # v1. 로 시작하는 태그만
git tag -l "v0.*"

# 메시지 포함 상세 목록
git tag -n
# v0.0.1  (경량 태그는 커밋 메시지 표시)
# v1.0    First major release
```

---

---

# ④ 태그 체크아웃 — detached HEAD ⭐️

```bash
# 태그 시점으로 이동
git switch v1.0
# ⚠️ detached HEAD 상태가 됨
```

```
detached HEAD 란:
  브랜치가 아닌 특정 커밋에 HEAD 가 직접 연결된 상태
  코드 확인 / 실험은 가능
  이 상태에서 커밋하면 브랜치에 연결되지 않음
  → 다른 브랜치로 이동 시 고립(orphan)되어 유실 위험

  안전하게 작업하려면 새 브랜치를 만들 것
```

```bash
# detached HEAD 상태에서 새 브랜치 만들기
git switch -c hotfix/v1.0 v1.0

# 또는 태그에서 바로 브랜치 생성
git switch -c branch-name v1.0

# 원래 브랜치로 복귀
git switch master
git switch -    # 직전 브랜치로 이동
```

---

---

# ⑤ 태그 삭제

```bash
# 로컬 태그 삭제
git tag -d v0.0.1

# 여러 태그 한번에 삭제
git tag -d v0.0.1 v0.1

# 원격 태그 삭제
git push origin --delete v0.0.1
git push origin :refs/tags/v0.0.1   # 구버전 방식
```

```
⚠️ 공유된 태그 삭제 주의:
  팀원이 해당 태그를 참조하고 있을 수 있음
  삭제 전 팀원과 소통 필요
```

---

---

# ⑥ 원격 저장소 태그 푸시

```bash
# 태그는 git push 할 때 자동으로 올라가지 않음
git push origin v1.0          # 특정 태그만
git push origin --tags        # 모든 태그 한번에
```

```
⚠️ 태그는 자동 push 안 됨:
  git push origin main 해도 태그는 올라가지 않음
  → 반드시 별도로 git push origin 태그명 필요
```

---

---

# 명령어 한눈에

|명령어|역할|
|---|---|
|`git tag v1.0`|경량 태그 생성|
|`git tag -a v1.0 -m "메시지"`|주석 태그 생성 ⭐️|
|`git tag v1.0 커밋해시`|특정 커밋에 태그|
|`git tag`|전체 태그 목록|
|`git tag -l "v1.*"`|패턴 검색|
|`git tag -n`|메시지 포함 목록|
|`git show v1.0`|태그 상세 정보|
|`git switch v1.0`|태그 체크아웃 (detached HEAD)|
|`git switch -c 브랜치 v1.0`|태그 → 새 브랜치 생성|
|`git tag -d v1.0`|로컬 태그 삭제|
|`git push origin v1.0`|원격에 태그 푸시|
|`git push origin --tags`|모든 태그 푸시|

---

---

# 자주 하는 실수

|실수|원인|해결|
|---|---|---|
|push 했는데 태그가 없음|태그 자동 push 안 됨|`git push origin --tags`|
|detached HEAD 상태에서 커밋 유실|브랜치 없이 커밋|`git switch -c 브랜치명` 먼저|
|경량 태그에 메시지가 없음|경량 태그는 메시지 없음|`-a -m` 옵션으로 주석 태그 사용|