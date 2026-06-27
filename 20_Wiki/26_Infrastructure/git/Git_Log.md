---
aliases:
  - git log
  - 커밋 히스토리
  - git shortlog
  - git show
tags:
  - Git
related:
  - "[[00_Git_HomePage]]"
  - "[[Git_Branch]]"
  - "[[Linux_Search_Filter]]"
---

# Git_Log — 커밋 히스토리 조회

## 한 줄 요약

```txt
git log = 커밋 이력을 확인하는 명령어
언제 / 누가 / 무엇을 바꿨는지 추적
```

---

---

# ① git log 기본

```bash
# 전체 로그 (역순 — 최신 커밋 먼저)
git log

# 출력 항목:
# commit a1b2c3d4e5...   ← 전체 커밋 해시 (고유 식별자)
# Author: gong <gong@email.com>
# Date:   Thu May 15 14:30:00 2026
#
#     feat: 로그인 기능 추가  ← 커밋 메시지

# q 로 종료 (내용 길면 페이저 less 로 표시됨)
```

```bash
# 한 줄 요약 ⭐️ (가장 자주 씀)
git log --oneline
# a1b2c3d feat: 로그인 기능 추가
# f4e5d6c fix: 버그 수정
# 7c8b9a0 Initial commit
#  ↑짧은 해시  ↑커밋 메시지
```

---

---

# ② 출력 형식 옵션

```bash
# 변경된 파일 목록 + 줄 수
git log --stat
# commit a1b2c3d
# src/app.ts  | 10 +++++-----
# 1 file changed, 5 insertions(+), 5 deletions(-)

# 실제 변경 코드 전체 보기 (diff 포함)
git log -p

# 커밋 수 제한
git log -5          # 최근 5개만
git log --oneline -10
```

## --pretty=format 커스텀 포맷

```bash
git log --pretty=format:"%h - %an, %ar : %s"
# a1b2c3d - gong, 2 hours ago : feat: 로그인 기능

# 자주 쓰는 플레이스홀더
# %h   짧은 커밋 해시
# %H   전체 커밋 해시
# %an  작성자 이름
# %ae  작성자 이메일
# %ar  상대적 날짜 (2 hours ago)
# %ad  절대 날짜
# %s   커밋 메시지 제목
# %cn  커미터 이름
```

---

---

# ③ 날짜 필터링

```bash
# 기간 필터
git log --since=1.week          # 지난 1주일
git log --since=2.days          # 지난 2일
git log --since="1 month ago"   # 한 달 전부터

# 특정 날짜 범위
git log --after="2024-01-01" --before="2024-01-31"
git log --since="2024-01-01" --until="2024-01-31"  # 동일
```

```txt
날짜 표현 방식:
  --since / --after  같은 의미 (시작일)
  --before / --until 같은 의미 (종료일)

  자연어도 인식:
    "yesterday"
    "1 month 2 weeks ago"
    "2024-05-01"
```

---

---

# ④ 검색 필터링 ⭐️

## 커밋 메시지 검색

```bash
# 커밋 메시지에 특정 단어 포함
git log --grep="로그인"
git log --grep="fix"

# 대소문자 무시
git log --grep="function" -i
```

## 특정 파일 검색

```bash
# 특정 파일이 변경된 커밋만
git log -- script.js
git log -- src/app.ts

# 파일 변경 내용까지
git log -p -- script.js
```

## -S 옵션 (pickaxe) ⭐️

```bash
# 특정 문자열이 추가/삭제된 커밋 찾기
git log -S "console.log"
git log -S "function login"
```

```txt
-S "문자열" = pickaxe 검색
  해당 문자열의 등장 횟수가 변한 커밋 찾기
  추가된 커밋 또는 삭제된 커밋

언제 쓰나:
  "이 코드가 언제 추가됐지?" 추적
  "이 함수가 언제 사라졌지?" 추적
  버그 도입 시점 찾기

예시:
  git log -S "getUserById"
  → getUserById 함수가 추가되거나 삭제된 커밋 목록
```

## 옵션 조합

```bash
# 지난 1주일 동안 script.js 에서 발생한 변경 상세 확인
git log -p --since=1.week -- script.js

# 특정 작성자 커밋만
git log --author="gong"

# 여러 조건 조합
git log --author="gong" --since=1.month --oneline
```

---

---
# ⑤ git show — 특정 커밋 상세 확인 ⭐️

```bash
# 특정 커밋의 전체 내용 확인
git show a1b2c3d
git show a1b2c3d4e5f6...   # 전체 해시도 가능

# 출력 내용:
# commit a1b2c3d
# Author: gong <gong@email.com>
# Date:   ...
# 커밋 메시지
# --- 변경된 코드 (diff) ---
```

```txt
git show 언제 쓰나:
  git log -S 로 커밋 찾은 후 → git show 로 상세 확인
  "이 커밋에서 정확히 뭘 바꿨지?" 확인
  작성자 / 날짜 / 변경 코드 한번에 보기
```

## pickaxe → show 실전 워크플로우 ⭐️


```bash
# 1. 특정 함수가 사라진 커밋 찾기
git log -S "secretAlgorithm()" --oneline
# a1b2c3d feat: 알고리즘 추가
# f4e5d6c refactor: 코드 정리    ← 삭제된 커밋 의심

# 2. 해당 커밋 상세 확인
git show f4e5d6c
# → 작성자 / 날짜 / 실제 변경 코드 확인

# 3. 결과 저장 (실습/자동화)
echo "f4e5d6c" > ~/user_answer.txt
echo "gong" > ~/user_author.txt
```

```txt
목록에서 가장 최근 커밋:
  가장 위에 표시된 커밋이 최신
  함수가 삭제됐다면 → 최근 커밋에서 삭제됐을 가능성 높음
  git show 로 확인해서 - (삭제 라인) 찾기
```

## 옵션 조합

```bash
# 지난 1주일 동안 script.js 에서 발생한 변경 상세 확인
git log -p --since=1.week -- script.js

# 특정 작성자 커밋만
git log --author="gong"

# 여러 조건 조합
git log --author="gong" --since=1.month --oneline
```

---

---

# ⑤ 통계 명령어

```bash
# 작성자별 커밋 수 요약
git shortlog -s -n
#  15 gong
#   8 teamA
#   3 teamB
# -s = 건수만 / -n = 많은 순 정렬

# 전체 커밋 수
git rev-list --count HEAD

# 특정 작성자 추가/삭제 줄 수
git log --author="gong" --pretty=tformat: --numstat \
| awk '{ add += $1; subs += $2 } END { printf "추가: %s줄, 삭제: %s줄\n", add, subs }'

# 가장 많이 변경된 파일 top 10
git log --pretty=format: --name-only \
| sort | uniq -c | sort -rg | head -10
```

---

---

# 명령어 한눈에

|명령어|역할|
|---|---|
|`git log`|전체 커밋 로그|
|`git log --oneline`|한 줄 요약 ⭐️|
|`git log --stat`|변경 파일 + 줄 수|
|`git log -p`|변경 코드 전체|
|`git log -5`|최근 N개만|
|`git log --since=1.week`|기간 필터|
|`git log --grep="키워드"`|메시지 검색|
|`git log -- 파일명`|특정 파일 이력|
|`git log -S "문자열"`|pickaxe 검색 ⭐️|
|`git log -S "문자열" --oneline`|pickaxe 한 줄 요약|
|`git show 해시`|특정 커밋 상세 확인 ⭐️|
|`git shortlog -s -n`|작성자별 커밋 수|
|`git rev-list --count HEAD`|전체 커밋 수|