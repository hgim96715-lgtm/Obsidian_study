---
aliases:
  - 디렉토리 명령어
  - mkdir
  - ls -F
  - touch
  - tree
  - cd
  - ls
  - pwd
tags:
  - Linux
related:
  - "[[00_Linux_HomePage]]"
  - "[[Linux_File_Move_Copy]]"
  - "[[Linux_Directory_Structure]]"
  - "[[Linux_Archive_Compress]]"
  - "[[Linux_Permission_Model]]"
---
# Linux_Directory_Commands — 디렉토리 관리

## 한 줄 요약

```
mkdir  → 디렉토리 생성
toudh  → 빈 파일 생성
cd     → 디렉토리 이동
ls     → 기본 목록
ls -F  → 파일/폴더 구분해서 목록 보기
tree   → 전체 구조 시각화
```

---

---

# ① mkdir — 디렉토리 생성 ⭐️

```bash
# 단일 디렉토리
mkdir src

# 여러 개 한 번에
mkdir src config docs
# src/ config/ docs/ 세 개 동시 생성

# 중간 경로 포함 (없으면 자동 생성)
mkdir -p project/src/utils
#  -p = parents (부모 디렉토리도 함께 생성)

# 권한 지정하며 생성
mkdir -m 755 mydir
```

## 프로젝트 구조 한 번에 생성

```bash
# 데이터 엔지니어 프로젝트 구조
mkdir -p myproject/{src,config,docs,logs,tests}

ls -F myproject/
# config/  docs/  logs/  src/  tests/
```

## -m — 권한 지정하며 생성

```bash
# 생성과 동시에 권한 설정
mkdir -m 700 private_dir
# 700: 소유자만 읽기/쓰기/실행 가능
#      그룹 / 기타 사용자는 아무 권한 없음

mkdir -m 750 shared_dir
# 750: 소유자 rwx / 그룹 rx / 기타 없음

# 권한 확인
ls -ld private_dir
# drwx------  ← 700
```

>→ [[Linux_Permission_Model]] 참고 

## -v — verbose 상세 출력

```bash
# 생성할 때마다 메시지 출력
mkdir -v src config docs
# mkdir: created directory 'src'
# mkdir: created directory 'config'
# mkdir: created directory 'docs'

# 여러 디렉토리 한번에 + 확인
mkdir -v ~/project/resources/books \
         ~/project/resources/articles \
         ~/project/resources/videos
```

## 옵션 조합 ⭐️

```bash
# -p (중간경로) + -v (확인) + -m (권한) 동시에
mkdir -pvm 750 ~/project/drafts ~/project/references
# p: 중간 경로 자동 생성
# v: 생성 과정 출력
# m: 750 권한 설정
```

## 기타 옵션

```
-Z                SELinux 보안 컨텍스트를 기본값으로 설정
--context[=CTX]   SELinux / SMACK 보안 컨텍스트를 CTX 로 설정
--help            도움말 메시지 출력
--version         버전 정보 출력

실무에서는 -p / -m / -v 가 대부분
-Z / --context 는 SELinux 환경 (RHEL / CentOS) 에서 사용
```

---

---

# ② touch — 빈 파일 생성 ⭐️

```bash
# 빈 파일 생성
touch hello.txt
touch file1.txt file2.txt file3.txt   # 여러 개 동시

# 파일 있으면 수정 시간만 업데이트
touch existing_file.txt

# 생성 확인
ls -l hello.txt
```

```
touch 두 가지 역할:
  파일 없음 → 빈 파일 새로 생성
  파일 있음 → 수정 시간 업데이트

빈 설정 파일 / 테스트 파일 만들 때 자주 씀
```

## 중괄호 확장 — 한 번에 여러 파일/폴더 ⭐️

```
중괄호 확장 = 쉘이 자동으로 패턴을 펼쳐주는 기능
타이핑 줄이고 실수 방지
```

## 문법 3가지

```bash
# 1. 숫자 범위 {시작..끝}
touch note_{1..5}.txt
# → touch note_1.txt note_2.txt note_3.txt note_4.txt note_5.txt
# 결과: note_1.txt note_2.txt note_3.txt note_4.txt note_5.txt

# 2. 알파벳 범위 {a..z}
mkdir dir_{a..e}
# → mkdir dir_a dir_b dir_c dir_d dir_e
# 결과: dir_a/ dir_b/ dir_c/ dir_d/ dir_e/

# 3. 직접 나열 {항목1,항목2,...}
mkdir -p project/{src,config,docs,logs}
# → mkdir -p project/src project/config project/docs project/logs
```

```
⚠️ 주의:
  {1..5}  숫자 사이에 공백 없어야 함
  {a, b}  ❌ 공백 있으면 안 됨
  {a,b}   ✅
```

## 실전 예시

```bash
# 날짜별 디렉토리
mkdir -p logs/{2023,2024,2025}
# logs/2023/  logs/2024/  logs/2025/

# 앞자리 0 유지 (01, 02 ... 12)
mkdir -p monthly/{01..12}
# monthly/01/  monthly/02/  ...  monthly/12/

# 중첩 구조 한번에
mkdir -p project/{src,config,logs/{2024,2025}}
# project/src/
# project/config/
# project/logs/2024/
# project/logs/2025/

# 여러 확장자 파일
touch app.{py,txt,log}
# app.py  app.txt  app.log

# 연도별 보고서 파일
touch report_{2020..2026}.txt
# report_2020.txt ~ report_2026.txt
```

## 동작 원리

```bash
# 쉘이 실행 전에 자동으로 펼침
echo mkdir project/{src,config,logs}
# mkdir project/src project/config project/logs
#  ↑ 실제로 이렇게 변환되어 실행됨

# 확인 방법: echo 로 먼저 펼쳐진 결과 확인
echo touch note_{1..5}.txt
# touch note_1.txt note_2.txt note_3.txt note_4.txt note_5.txt

# 괜찮으면 echo 빼고 실행
touch note_{1..5}.txt
```

---

---

# ③ ls — 목록 확인 ⭐️

```bash
ls              # 기본 목록
ls -l           # 상세 정보 (권한 / 크기 / 날짜)
ls -ld          # 디렉토리 자체의 권한/소유자 정보
ls -a           # 숨김 파일 포함 (. 으로 시작)
ls -la          # 상세 + 숨김 파일
ls -F           # 파일 종류 구분 기호 추가
ls -R           # 재귀 목록 (하위 전체)
ls -lh          # 파일 크기 사람이 읽기 좋게 (KB, MB)
ls -lt          # 수정 시간 순 정렬 (최신 먼저)
ls -ltr         # 수정 시간 순 + 반전 (오래된 것 먼저) ⭐️
ls -lS          # 크기 순 정렬 (큰 것 먼저)
ls -lSr         # 크기 순 + 반전 (작은 것 먼저)
```

## ls -F 출력 해석 ⭐️

```bash
ls -F
# config/   docs/   logs/   src/   README.md   run.py*
#     ↑           ↑                    ↑(없음)     ↑
#  디렉토리    디렉토리              일반 파일    실행 파일

기호 의미:
  /   = 디렉토리
  *   = 실행 가능한 파일
  @   = 심볼릭 링크
  없음 = 일반 파일
```

## -r (reverse) — 정렬 반전 ⭐️

```bash
ls -lt          # 최신 파일이 위 (기본)
ls -ltr         # 오래된 파일이 위 → 최신이 맨 아래

ls -lS          # 큰 파일이 위 (기본)
ls -lSr         # 작은 파일이 위
```

```
-r 은 단독으로 안 쓰고 정렬 옵션과 조합

ls -ltr 실무 활용:
  로그 파일 목록 확인 시 가장 최근 파일이 맨 아래에 보임
  → 화면 스크롤 없이 최신 파일 바로 확인 가능
  → tail 명령어와 자연스럽게 연결

  ls -ltr /var/log/       # 가장 최근 로그 파일 맨 아래
  ls -ltr /var/log/ | tail -5  # 최근 5개만
```

## 특정 디렉토리 엿보기 ⭐️

```bash
# 직접 들어가지 않고 하위 디렉토리 내용 확인
ls -l test/             # test 디렉토리 내용
ls -la /etc/nginx/      # 절대 경로 디렉토리 내용
ls -lh /var/log/        # 로그 디렉토리 내용

# 결과 예시
ls -l test
# total 0               ← 파일 없음 (디렉토리는 존재)
```

```bash
# 와일드카드로 여러 디렉토리 한 번에 확인
ls -l */                # 현재 위치의 모든 하위 디렉토리 내용
ls -l src/ logs/ test/  # 특정 여러 디렉토리 동시 확인
```

```
⚠️ 읽기 권한이 없으면:
  ls: cannot open directory 'private/': Permission denied
  → Linux 보안 모델: 허용된 디렉토리만 접근 가능
  → sudo ls -l private/ 로 권한 얻어서 확인

실무 패턴:
  ls -l 경로/    들어가지 않고 내용 미리 확인
  ls -l */       현재 디렉토리 내 모든 폴더 구조 한눈에
```

## --color — 색상 출력 ⭐

```bash
ls --color=auto    # 기본값 (터미널엔 색상 / 파이프엔 색상 제거)
ls --color=always  # 항상 색상 (파이프·파일 출력에도 색상 포함)
ls --color=never   # 색상 없음 (순수 텍스트만)
```

```
auto (기본):
  터미널에 직접 출력 → 색상 표시
  파이프 / 파일로 보낼 때 → 색상 코드 제거
  → 텍스트 데이터가 깨지지 않음

always:
  파이프 / 파일 출력에도 색상 코드 포함
  ls --color=always | grep 디렉토리  → 색상 코드 섞임 주의

never:
  색상 완전 비활성화
  스크립트 작성 시 출력 일정하게 유지할 때 유용
  색상 코드 없이 순수 텍스트만 처리하고 싶을 때

실무:
  대화형 사용 → auto (기본, 신경 안 써도 됨)
  스크립트에서 ls 출력 파싱 → --color=never 명시
  grep 과 조합 → auto 면 충분
```

## ls -ld — 디렉토리 자체 권한 확인 ⭐️

```bash
# 디렉토리 내용이 아닌 디렉토리 자체 정보
ls -ld ~/project/private
# drwx------  1 user group  4096 May 23  private
# ↑ d = 디렉토리 / rwx------ = 700 권한

ls -ld .        # 현재 디렉토리 자체 정보
ls -ld /etc     # /etc 디렉토리 자체 정보
```

```
ls -l   디렉토리 안의 파일 목록
ls -ld  디렉토리 자체의 권한/소유자 정보
        → mkdir -m 으로 설정한 권한 확인 시 사용
```

## ls -R — 재귀 목록 (하위 전체)

```bash
# 하위 디렉토리 포함 전체 목록
ls -R test_dir
# test_dir:
# file1.txt  file2.txt  subdir1  subdir2
#
# test_dir/subdir1:
# subfile1.txt
```

```
tree 없는 환경에서 하위 구조 확인 시 대체 사용
tree 가 있으면 tree 쪽이 더 보기 좋음
```


---

---

# ③ cd — 디렉토리 이동

```bash
cd /home/labex/project    # 절대 경로로 이동
cd project                # 현재 위치에서 상대 경로
cd ..                     # 한 단계 위로
cd ../..                  # 두 단계 위로
cd ~                      # 홈 디렉토리로
cd -                      # 직전 디렉토리로 (토글)
cd                        # 홈 디렉토리로 (~ 생략)
```

---

---

# ④ pwd — 현재 위치 확인

```bash
pwd           # 현재 위치 출력
pwd -L        # 논리적 경로 (심볼릭 링크 경로 그대로)
pwd -P        # 물리적 경로 (실제 파일시스템 위치)
```

```
-L vs -P:
  심볼릭 링크가 없으면 → 둘 다 동일한 결과

  심볼릭 링크가 있을 때:
    symlink_dir → real_dir 를 가리키는 링크

    cd symlink_dir
    pwd -L   /home/labex/project/symlink_dir  ← 링크 경로 그대로
    pwd -P   /home/labex/project/real_dir     ← 실제 물리 경로

  -L (logical)   링크를 따라가서 논리 경로 표시
  -P (physical)  링크 무시하고 실제 위치 표시
  → Windows 의 "바로가기" 와 유사한 개념
```

---

---

# ⑤ tree — 전체 구조 시각화

```bash
# 설치 (없으면)
sudo apt-get install tree

# 기본 (현재 디렉토리)
tree

# 특정 디렉토리
tree ~/project
tree test_dir
tree extracted_tar     # 압축 해제 후 구조 확인

# 깊이 제한
tree -L 2       # 2단계까지만
tree -L 1       # 바로 아래 한 단계만

# 숨김 파일 포함
tree -a

# 디렉토리만 (파일 제외)
tree -d

# 파일 크기 포함
tree -h         # human-readable (KB, MB)
tree -s         # bytes 단위

# 권한 정보 포함 ⭐️
tree -p ~/project
# [-rwxr-xr-x]  file.py
# [drwxr-x---]  private_dir/
# ↑ 각 파일/폴더의 권한을 함께 표시
```

## tree 출력 읽는 법

```
phoenix_project/
├── config/           ← ├── 중간 항목
│   ├── config.json
│   └── config.json.bak   ← └── 마지막 항목
├── docs/
│   └── README.md
└── src/              ← └── 마지막 항목 (더 이상 형제 없음)
    └── main_app.py
```

```
기호:
  ├──  다음 항목이 더 있음
  └──  마지막 항목
  │    위 항목의 하위가 계속됨
```


```
tree vs ls -R:
  tree      그래픽 트리 형태 / 보기 좋음 / 설치 필요
  ls -R     기본 내장 / 항상 사용 가능 / 덜 직관적

  → tree 없으면 ls -R 로 대체 → ls 섹션 참고
```

---

---

# ⑥ 실전 패턴 — 프로젝트 구조 설계

```bash
# 1. 프로젝트 루트 이동
cd ~/project/phoenix_project

# 2. 기본 폴더 구조 생성
mkdir src config docs

# 3. 생성 확인 (파일/폴더 구분)
ls -F
# config/  docs/  src/

# 4. 이동하면서 작업
cd config
pwd    # 현재 위치 확인

# 5. 전체 구조 확인
cd ~/project/phoenix_project
tree -L 2
```

---

---

# 자주 하는 실수

| 실수               | 원인       | 해결               |
| ---------------- | -------- | ---------------- |
| `mkdir a/b/c` 에러 | 중간 경로 없음 | `mkdir -p a/b/c` |
| 디렉토리인지 파일인지 헷갈림  | ls 기본 출력 | `ls -F` 로 `/` 확인 |
| 잘못된 위치에서 작업      | pwd 안 함  | 작업 전 `pwd` 습관화   |