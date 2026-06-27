---
aliases:
  - 환경변수
  - export
  - bashrc
tags:
  - Linux
related:
  - "[[00_Linux_HomePage]]"
  - "[[Linux_Shell_Script_Basics]]"
---

# Linux_Environment_Variables — 환경변수

## 한 줄 요약

```txt
로컬 변수   → 현재 셸 안에서만
환경 변수   → export 하면 자식 셸까지 전달
.bashrc     → 셸 시작할 때 자동으로 실행되는 설정 파일
```

---

---

# ① 로컬 변수 vs 환경 변수 ⭐️

## 변수 3가지 종류 비교 ⭐️

|변수 유형|범위|자식 프로세스|지속성|문법|
|---|---|:-:|---|---|
|**쉘 변수**|현재 세션 전용|❌|종료 시 소멸|`my_var='value'`|
|**환경 변수**|현재 세션|✅|종료 시 소멸|`export VAR='value'`|
|**영구 변수**|모든 새 세션|✅|종료 후에도 유지|`~/.bashrc` 또는 `~/.zshrc` 에 작성|

```txt
부모 셸 (bash)
  ├── 로컬 변수   → 현재 셸 안에서만 존재
  └── 환경 변수   → 자식 프로세스에 전달됨

자식 셸 예시:
  bash 에서 zsh 실행   → zsh 가 자식 셸
  bash 에서 python3   → python3 가 자식 프로세스
  bash 에서 스크립트   → 스크립트가 자식 프로세스
```

```bash
# 로컬 변수 — 현재 셸에서만
MY_VAR=hello

# 자식 셸에서 확인
bash -c 'echo $MY_VAR'    # 빈 값 (전달 안 됨)
zsh -c 'echo $MY_VAR'     # 빈 값

# 환경 변수 — export 로 자식에 전달
export MY_VAR=hello

bash -c 'echo $MY_VAR'    # hello (전달됨)
zsh -c 'echo $MY_VAR'     # hello (전달됨)
```

```txt
로컬 변수:
  MY_VAR=hello
  → 선언한 셸 안에서만 유효

환경 변수:
  export MY_VAR=hello
  → 자식 셸 / 자식 프로세스에 자동 전달

확인:
  echo $MY_VAR    # 값 출력
  env             # 현재 모든 환경 변수 목록
  printenv MY_VAR # 특정 환경 변수만
```

## `$$` — 현재 셸 PID 확인 ⭐️

```text
$$ = 현재 셸의 PID (프로세스 ID)
자식 셸에 들어가면 PID 가 바뀜
→ PID 가 바뀌었다 = 새 프로세스(새 셸) 진입 확인
```

```bash
# 현재 bash PID 확인
echo $$           # 123

# 자식 셸 진입
zsh

echo $$           # 456  ← PID 바뀜 = 새 프로세스

# 프로세스 계층 확인
ps -f
# UID   PID  PPID   CMD
# user  123   100   bash   ← 부모
# user  456   123   zsh    ← 자식 (PPID=123 이 부모)

# 자식 셸 종료 → 부모 셸로 복귀
exit
echo $$           # 123  ← 다시 부모 PID
```

```txt
PPID = Parent PID (부모 프로세스 ID)
ps -f 에서 자식의 PPID = 부모의 PID 와 일치
→ 부모-자식 관계 눈으로 확인 가능

실전:
  zsh 들어갔는데 변수가 없다 → echo $$ 로 셸 바뀐 것 확인
  자식 셸인지 부모 셸인지 헷갈릴 때 → ps -f 로 계층 확인
```

---

---

# ② 변수 기본 문법

```bash
# 선언
MY_VAR=hello           # ⚠️ = 앞뒤 공백 없어야 함
MY_VAR="hello world"   # 공백 포함 시 따옴표

# 사용
echo $MY_VAR
echo ${MY_VAR}         # 중괄호 명시 (권장)

# 해제
unset MY_VAR

# 현재 셸 변수 목록
set                    # 모든 변수 + 함수
env                    # 환경 변수만
printenv               # env 와 동일
```

```txt
⚠️ 자주 하는 실수:
  MY_VAR = hello   ← 에러! (공백 있으면 명령어로 인식)
  MY_VAR=hello     ← 정상
```

---

---

# ③ export — 환경 변수로 승격

```bash
# 선언과 동시에 export
export MY_VAR=hello

# 이미 선언된 변수를 export
MY_VAR=hello
export MY_VAR

# export 목록 확인
export -p              # 현재 export 된 변수 전체
```

## set -o allexport — 자동 export ⭐️

```txt
set -o  →  옵션 켜기  (on)
set +o  →  옵션 끄기  (off)

-o = on  /  +o = off  ← 헷갈리기 쉬움 주의
```

```bash
# 자동 export 켜기
set -o allexport      # 이후 선언하는 모든 변수 자동 export

MY_VAR=hello          # export 없이도 환경 변수가 됨
DB_HOST=localhost

# 자동 export 끄기
set +o allexport      # 이후부터는 다시 수동 export 필요
```

```txt
set -o allexport 켜면:
  export MY_VAR=hello  와
  MY_VAR=hello         가 완전히 동일
  선언만 해도 자동으로 환경 변수

단점:
  의도치 않은 변수도 전부 자식에 노출됨
  → .env 파일 통째로 불러올 때만 사용 권장
  → 다 쓰면 set +o allexport 로 반드시 끄기

현재 set 옵션 상태 확인:
  set -o           # 모든 옵션 on/off 상태 출력
  set -o | grep allexport  # allexport 만 확인
```

---

---

# ④ 별칭 (alias) ⭐️

```txt
자주 쓰는 긴 명령어를 짧게 줄이기
alias 이름='실제 명령어'
```

## 기본 사용

```bash
# 별칭 선언
alias ll='ls -la'
alias gs='git status'
alias py='python3'
alias ..='cd ..'
alias ...='cd ../..'

# 여러 명령어 조합
alias update='sudo apt update && sudo apt upgrade -y'

# 사용
ll            # ls -la 와 동일
gs            # git status 와 동일
```

## 확인 / 해제

```bash
# 전체 별칭 목록 확인
alias

# 특정 별칭 확인
alias ll       # alias ll='ls -la'

# 별칭 해제 (현재 세션에서 삭제)
unalias ll

# 모든 별칭 해제
unalias -a
```

## 별칭 일시 무시

```bash
# \ 붙이면 별칭 무시하고 원본 명령어 실행
\ls           # alias ls 가 있어도 원본 ls 실행
```

## 자식 셸 상속 문제

```bash
# 부모 셸에서 선언
alias ll='ls -la'

# 자식 셸에서는 사라짐
bash -c 'alias'     # 빈 출력 (전달 안 됨)
zsh -c 'alias'      # 빈 출력

# 이유: alias 는 현재 셸 세션에만 존재
# 터미널 닫으면 사라짐 / 자식 셸에 안 넘어감
```

```txt
영구 적용하려면:
  ~/.bashrc  (bash 사용 시)
  ~/.zshrc   (zsh 사용 시)
  에 추가 후 source ~/.bashrc 실행
```

---

---

# ⑤ .bashrc / .bash_profile — 영구 적용 ⭐️

```txt
셸 시작할 때 자동 실행되는 설정 파일
여기에 export / alias 넣으면 매번 자동 적용
```

## 파일별 역할

```txt
~/.bashrc
  인터랙티브 셸 (터미널 열 때마다) 실행
  alias / 함수 / 프롬프트 설정

~/.bash_profile (또는 ~/.profile)
  로그인 셸 (SSH 접속 / 첫 로그인) 실행
  환경 변수 설정 (PATH 등)
  보통 .bashrc 를 여기서 불러옴

~/.zshrc
  zsh 사용 시 .bashrc 역할
  macOS 기본 쉘 / 일부 Ubuntu 환경
```

## 설정 추가 방법

```bash
# .bashrc 또는 .zshrc 열기
nano ~/.bashrc     # bash
nano ~/.zshrc      # zsh (macOS / zsh 환경)

# 파일 맨 아래에 추가
export MY_ENV_VAR="This is an environment variable"
export PATH="$PATH:$HOME/my_scripts"
alias ll='ls -la'
```

```bash
# 수정 후 즉시 반영 (새 터미널 안 열어도 됨)
source ~/.bashrc
source ~/.zshrc
. ~/.bashrc        # source 의 단축형
```

## source vs bash 차이 ⭐️

```bash
# source (또는 .) — 현재 셸에서 직접 실행
source ~/.zshrc    # ✅ 현재 세션에 즉시 적용

# bash — 새 자식 셸에서 실행
bash ~/.zshrc      # ❌ 자식 셸에서만 실행 → 현재 셸 영향 없음
```

```txt
source:
  현재 셸 프로세스 안에서 파일 실행
  → export 한 변수가 현재 셸에 바로 적용

bash ~/.zshrc:
  새 자식 셸(프로세스) 을 만들어 실행
  → 현재 셸에 아무 영향 없음
  → 파일 적용 안 됨

.bashrc 수정 후 반드시 source 사용
```

## PATH 추가 패턴

```bash
# 기존 PATH 뒤에 추가 (덮어쓰지 않도록)
export PATH="$PATH:$HOME/my_scripts"
#             ↑ 기존 유지  ↑ : 구분자  ↑ 새 경로

# 앞에 추가 (우선순위 높이기)
export PATH="/새로운/경로:$PATH"
```

```txt
PATH 의 역할:
  명령어를 입력했을 때 어느 디렉토리에서 찾을지 목록
  : 으로 구분된 경로들을 순서대로 탐색

echo $PATH
# /usr/local/bin:/usr/bin:/bin:/usr/sbin:/sbin
```

## 스크립트로 쉘 변수 vs 환경 변수 테스트 ⭐️

```bash
# 쉘 변수만 선언
my_var="Hello"

# 환경 변수 선언
export MY_ENV_VAR="환경 변수입니다"

# 스크립트 생성
cat << 'EOF' > test_vars.sh
#!/bin/bash
echo "쉘 변수: $my_var"
echo "환경 변수: $MY_ENV_VAR"
EOF

chmod +x test_vars.sh
./test_vars.sh

# 결과:
# 쉘 변수:             ← 빈칸 (자식 프로세스 접근 불가)
# 환경 변수: 환경 변수입니다  ← 정상 출력
```

---

---

# ⑤-1 내장 환경변수 ⭐️

```bash
echo $HOME    # 홈 디렉토리 경로 (/home/labex)
echo $USER    # 현재 로그인 사용자 (labex)
echo $SHELL   # 기본 쉘 경로 (/bin/bash 또는 /bin/zsh)
echo $PWD     # 현재 작업 디렉토리 (pwd 명령어와 동일)
echo $TERM    # 터미널 유형 (xterm-256color 등)
echo $PATH    # 실행 파일 검색 경로
```

```txt
자주 쓰는 내장 변수:
  $HOME   → 홈 경로 대신 ~ 로도 쓸 수 있음
  $USER   → 현재 사용자 확인
  $PWD    → 스크립트 안에서 현재 위치 확인할 때
  $PATH   → 실행 파일 경로 목록
```

---

---

# ⑥ 변수 확인 명령어 정리

```bash
env             # 환경 변수 전체
printenv        # env 와 동일
printenv PATH   # 특정 변수만
set             # 셸 변수 + 환경 변수 + 함수 전체
echo $MY_VAR    # 특정 변수 값
export -p       # export 된 변수만
```

---

---

# ⑦ 실전 패턴

## .env 파일 → 셸에 불러오기

```bash
# .env 파일 내용
# API_KEY=abc123
# DB_HOST=localhost

# 불러오기
export $(cat .env | xargs)

# 또는
set -o allexport
source .env
set +o allexport
```

## 임시 환경 변수로 명령어 실행

```bash
# 해당 명령어 실행 동안만 변수 적용
MY_ENV=prod python3 app.py

# 영구 설정 없이 특정 실행에만 변수 전달
DEBUG=true ./script.sh
```

---

---

# 자주 하는 실수

|실수|원인|해결|
|---|---|---|
|`MY_VAR = hello` 에러|= 앞뒤 공백|`MY_VAR=hello` 붙여 써야 함|
|자식 셸에서 변수 없음|export 안 함|`export MY_VAR=hello`|
|터미널 새로 열면 alias 사라짐|.bashrc 에 없음|`~/.bashrc` 에 추가 후 `source ~/.bashrc`|
|.bashrc 수정했는데 적용 안 됨|source 안 함|`source ~/.bashrc` 실행|
|zsh 에서 bash alias 안 됨|셸별 설정 파일 다름|`~/.zshrc` 에 별도 추가|
|자식 셸인지 부모 셸인지 모름|눈으로 확인 불가|`echo $$` 로 PID / `ps -f` 로 계층 확인|