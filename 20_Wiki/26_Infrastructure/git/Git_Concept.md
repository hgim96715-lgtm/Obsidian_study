---
aliases:
  - Git 개념
  - 버전 관리
  - Staging Area
  - HEAD
tags:
  - Git
related:
  - "[[00_Git_HomePage]]"
  - "[[Git_Commands]]"
  - "[[Git_Branch]]"
---

# Git_Concept — Git 이란 무엇인가

## 한 줄 요약

```txt
Git = 코드의 변경 이력을 추적하는 버전 관리 시스템
"소설최종.txt / 소설최종v2.txt" 대신
Git 이 변경 이력을 전부 기억해줌
```

---

---

# ① 왜 Git 이 필요한가

```txt
Git 없이:
  소설최종.txt
  소설최종v2.txt
  소설진짜_최종.txt
  소설진짜_최종_수정본.txt
  → 파일이 넘쳐남 / 뭐가 최신인지 모름

Git 있으면:
  파일 하나 + 변경 이력 전부 저장
  언제든지 과거 버전으로 되돌아갈 수 있음
  팀원과 동시에 작업 가능
```

---

---

# ② Git 핵심 개념 ⭐️

## 저장소 (Repository)

```txt
Git 이 관리하는 프로젝트 폴더
.git 폴더가 생기면 저장소

로컬 저장소  = 내 컴퓨터
원격 저장소  = GitHub / GitLab 서버
```

## 커밋 (Commit)

```txt
비디오 게임의 '세이브 포인트'
특정 시점의 파일 상태를 기록
되돌아갈 수 있는 지점

커밋마다 고유한 해시값 (ID) 존재
  a1b2c3d4e5... → 이 커밋을 가리키는 주소
```

## Staging Area (스테이징 영역) ⭐️

```txt
작업 폴더 → Staging Area → 저장소

왜 두 단계인가:
  변경된 파일이 10개 있어도
  이번 커밋에는 3개만 담고 싶을 때
  → Staging 에 3개만 올리고 커밋

  작업 공간 (Working Directory)
    ↓ git add
  Staging Area (커밋 준비 구역)
    ↓ git commit
  Repository (실제 저장)
```

## 파일의 3가지 상태

```txt
Untracked   Git 이 아직 모르는 파일 (새로 만든 파일)
Staged      커밋 준비 완료 (git add 후)
Committed   저장소에 기록됨 (git commit 후)
```

## HEAD

```txt
HEAD = 현재 내가 보고 있는 커밋을 가리키는 포인터
보통 가장 최신 커밋을 가리킴

git log 에서 (HEAD -> master) 표시
→ 현재 master 브랜치의 최신 커밋에 있다는 뜻
```

## 브랜치 (Branch)

```txt
하나의 시간선
master (또는 main) = 메인 시간선

브랜치를 나누면:
  메인 코드 건드리지 않고 새 기능 개발 가능
  완성되면 메인에 합치기 (merge)
```

---

---

# ③ 로컬 vs 원격

```txt
로컬 저장소:
  내 컴퓨터에 있는 .git 폴더
  git init 으로 생성
  인터넷 없이 작업 가능

원격 저장소:
  GitHub / GitLab 같은 서버에 있는 저장소
  팀원과 공유 / 백업 역할
  git push 로 올리고 git pull 로 받음
```

---

---

# ④ Git 작업 흐름 ⭐️

```txt
① git init          → 저장소 초기화 (처음 한 번)
② 파일 작업         → 코드 작성 / 수정
③ git status        → 변경 상태 확인
④ git add 파일      → Staging Area 에 올리기
⑤ git commit -m ""  → 커밋 (세이브 포인트)
⑥ git push          → 원격 저장소에 업로드

반복: ② → ③ → ④ → ⑤ → ⑥
```

---

---

# 비유로 이해하기

|Git 용어|비유|
|---|---|
|저장소 (Repository)|프로젝트 폴더|
|커밋 (Commit)|게임 세이브 포인트|
|브랜치 (Branch)|시간선 / 평행 우주|
|Staging Area|커밋 장바구니|
|HEAD|지금 내가 있는 위치|
|merge|두 시간선 합치기|
|push|서버에 업로드|
|pull|서버에서 다운로드|