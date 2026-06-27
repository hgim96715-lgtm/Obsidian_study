---
aliases:
  - 쉘 스크립트
  - bash 스크립트
  - shebang
  - 변수
tags:
  - Linux
related:
  - "[[00_Linux_HomePage]]"
  - "[[Linux_Shell_Cron_Job]]"
  - "[[Linux_Redirect]]"
  - "[[Linux_Permission_Model]]"
---

# Shell_Script_Basics — 쉘 스크립트 기초

## 한 줄 요약

```
쉘 스크립트 = 리눅스 명령어들을 파일에 모아놓고 순서대로 실행
반복 작업 자동화 / 서버 관리 / 백업 등에 필수
```

---

---

# ① 첫 번째 스크립트 만들기

## 파일 생성 → 내용 작성 → 권한 부여 → 실행

```bash
# 1. 파일 생성
nano log_manager.sh

# 2. 내용 작성 (nano 에디터 안에서)
#!/bin/bash
echo "Log Manager Initialized."

# 3. 저장 후 종료 (Ctrl+X → Y → Enter)

# 4. 실행 권한 부여
chmod +x log_manager.sh

# 5. 실행
./log_manager.sh
# Log Manager Initialized.
```

## Shebang — #!/bin/bash ⭐️

```bash
#!/bin/bash
```

```
스크립트 파일의 맨 첫 줄에 반드시 써야 함
"이 파일을 /bin/bash 로 실행해라" 는 의미
없으면 어떤 인터프리터로 실행할지 몰라 오류 발생

#!  = Shebang (샤뱅)
/bin/bash = bash 인터프리터 경로
```

## chmod +x — 실행 권한 ⭐️

```bash
chmod +x log_manager.sh

# 권한 확인
ls -l log_manager.sh
# -rwxr-xr-x  ← x 가 있어야 실행 가능

# 실행 방법
./log_manager.sh   # 현재 디렉토리에서 실행
bash log_manager.sh # chmod +x 없이도 실행 가능
```

```
Permission denied 에러 = chmod +x 안 한 것
실행 전에 chmod +x 는 습관으로
```

>[[Linux_Permission_Model#③ chmod — 권한 변경 ⭐️]] 참고 

---

---

# ② 변수 ⭐️

```bash
# 변수 선언 — = 양쪽 공백 절대 금지!
LOG_DIR="/home/labex/project/app_logs"
BACKUP_DIR="/home/labex/project/backups"
COUNT=10

# ❌ 틀린 방법 (공백 있으면 에러)
LOG_DIR = "/home/labex/project/app_logs"   # 에러!

# 변수 사용 — $ 붙이기
echo $LOG_DIR
echo "로그 경로: $LOG_DIR"
echo "백업 경로: ${BACKUP_DIR}"   # {} 로 감싸면 더 안전
```

```
변수 규칙:
  선언: 변수명=값    (= 앞뒤 공백 없음)
  사용: $변수명      ($ 붙이기)
  문자열: 큰따옴표로 감싸기 권장
  대문자 관례: 환경변수/설정값은 대문자로
```

## 자주 쓰는 변수

```bash
#!/bin/bash

LOG_DIR="/home/labex/project/app_logs"
BACKUP_DIR="/home/labex/project/backups"
DATE=$(date +%Y%m%d)    # 명령어 결과를 변수에 저장
SCRIPT_NAME=$(basename $0)  # 현재 스크립트 이름

echo "날짜: $DATE"
echo "스크립트: $SCRIPT_NAME"
```

---

---

# ③ 사용자 입력 — read

```bash
#!/bin/bash

echo "Enter the backup filename: "
read BACKUP_FILENAME

echo "Backing up logs to: $BACKUP_FILENAME"
```

```
read 명령어:
  스크립트 실행 멈추고 키보드 입력 대기
  Enter 누르면 변수에 저장

-p 옵션으로 프롬프트 한 줄로:
  read -p "파일명 입력: " BACKUP_FILENAME
```

```bash
# 비밀번호 입력 (화면에 안 보이게)
read -s -p "패스워드: " PASSWORD

# 타임아웃 설정 (10초 안에 입력 없으면 기본값)
read -t 10 -p "계속할까요? [Y/n]: " ANSWER
ANSWER=${ANSWER:-Y}   # 입력 없으면 Y 기본값
```

---

---

# ④ 조건문 — if / else / fi ⭐️

## 기본 구조

```bash
if [ 조건 ]; then
    # 조건 참일 때 실행
elif [ 조건2 ]; then
    # 조건2 참일 때 실행
else
    # 조건 거짓일 때 실행
fi    # if 를 거꾸로 쓴 것 → 반드시 닫아야 함
```

## 파일 / 디렉토리 검사 ⭐️

```bash
# -d : 디렉토리 존재 여부
if [ -d "$LOG_DIR" ]; then
    echo "디렉토리 존재함"
else
    echo "Error: 디렉토리 없음"
    exit 1
fi

# -f : 파일 존재 여부
if [ -f "config.txt" ]; then
    echo "파일 있음"
fi

# -e : 파일 또는 디렉토리 존재 여부
if [ ! -e "$LOG_DIR" ]; then   # ! = NOT
    echo "경로 없음"
fi
```

## 숫자 비교

```bash
COUNT=5

if [ $COUNT -gt 3 ]; then   # greater than
    echo "3보다 크다"
fi

# 비교 연산자
# -eq  같음 (equal)
# -ne  다름 (not equal)
# -gt  초과 (greater than)
# -lt  미만 (less than)
# -ge  이상 (greater or equal)
# -le  이하 (less or equal)
```


|연산자|의미|예시|
|---|---|---|
|`-eq`|같음 (equal)|`[ $A -eq $B ]`|
|`-ne`|다름 (not equal)|`[ $A -ne $B ]`|
|`-gt`|초과 (greater than)|`[ $A -gt 3 ]`|
|`-lt`|미만 (less than)|`[ $A -lt 10 ]`|
|`-ge`|이상 (greater or equal)|`[ $A -ge 5 ]`|
|`-le`|이하 (less or equal)|`[ $A -le 5 ]`|

## 문자열 비교

```bash
STATUS="active"

if [ "$STATUS" = "active" ]; then
    echo "활성 상태"
fi

if [ -z "$VAR" ]; then    # 비어있으면
    echo "변수가 비어있음"
fi

if [ -n "$VAR" ]; then    # 비어있지 않으면
    echo "변수에 값이 있음"
fi
```

## exit 1 — 에러 종료 ⭐️

```bash
if [ ! -d "$LOG_DIR" ]; then
    echo "Error: Log directory not found."
    exit 1   # 비정상 종료 (에러 코드 1)
fi

# exit 코드:
# 0 = 성공 (정상 종료)
# 1 = 일반 에러
# 기타 = 프로그램별 에러 코드
```

```
Airflow 에서의 의미:
  exit 0 → 태스크 성공
  exit 1 → 태스크 실패 → 재시도 or 알림
```

---

---

# ⑤ 반복문 — for ⭐️

## 파일 순회 패턴

```bash
#!/bin/bash

LOG_DIR="/home/labex/project/app_logs"
BACKUP_DIR="/home/labex/project/backups"

# .log 파일 전부 순회
for file in $LOG_DIR/*.log; do
    echo "복사 중: $file"
    cp "$file" "$BACKUP_DIR/"
done

echo "백업 완료."
```

```
for file in $LOG_DIR/*.log; do
    ...
done

흐름:
  $LOG_DIR/*.log → 해당 경로의 .log 파일 전부 목록
  하나씩 file 변수에 담아서 do ~ done 안 실행
  모두 순회하면 종료
```

## 기타 for 패턴

```bash
# 숫자 범위
for i in 1 2 3 4 5; do
    echo "번호: $i"
done

# 시퀀스 (seq)
for i in $(seq 1 10); do
    echo $i
done

# 배열
SERVERS=("server1" "server2" "server3")
for server in "${SERVERS[@]}"; do
    echo "접속 중: $server"
done

# 디렉토리 안 파일 전부
for file in /var/log/*.log; do
    echo $file
done
```

## while 반복문

```bash
COUNT=0

while [ $COUNT -lt 5 ]; do
    echo "카운트: $COUNT"
    COUNT=$((COUNT + 1))   # 산술 연산
done
```

---

---

# ⑥ 전체 예시 — 로그 백업 스크립트

```bash
#!/bin/bash

# 설정값
LOG_DIR="/home/labex/project/app_logs"
BACKUP_DIR="/home/labex/project/backups"

echo "Log Manager Initialized."

# 1. 로그 디렉토리 확인
if [ -d "$LOG_DIR" ]; then
    echo "Log directory found. Proceeding..."

    # 2. 백업 파일명 입력 받기
    read -p "Enter the backup filename: " BACKUP_FILENAME

    echo "Backing up logs to: $BACKUP_FILENAME"

    # 3. .log 파일 순회하며 백업
    for file in $LOG_DIR/*.log; do
        echo "Copied $file"
        cp "$file" "$BACKUP_DIR/"
    done

    echo "Backup complete."

else
    echo "Error: Log directory not found."
    exit 1
fi
```

## 실행 방법

```bash
chmod +x log_manager.sh   # 실행 권한 부여
./log_manager.sh           # 실행
```

---

---

# 문법 주의사항 ⭐️

|실수|원인|해결|
|---|---|---|
|`LOG_DIR = "/path"` 에러|= 양옆 공백|`LOG_DIR="/path"` (공백 없음)|
|Permission denied|chmod +x 안 함|`chmod +x 스크립트.sh`|
|`fi` 누락|if 안 닫음|if 마지막에 반드시 `fi`|
|`done` 누락|for 안 닫음|for 마지막에 반드시 `done`|
|변수에 $ 안 붙임|변수명 그대로 출력|`$변수명`|
|`[ $VAR = "값" ]` 에러|VAR 이 비어있음|`[ "$VAR" = "값" ]` 따옴표 감싸기|

---

---

# 명령어 한눈에

|문법|의미|
|---|---|
|`#!/bin/bash`|쉐뱅 (bash 로 실행)|
|`chmod +x 파일`|실행 권한 부여|
|`VAR="값"`|변수 선언 (공백 금지)|
|`$VAR`|변수 사용|
|`read VAR`|사용자 입력 받기|
|`if [ 조건 ]; then`|조건문 시작|
|`fi`|조건문 종료|
|`for f in *.log; do`|파일 순회 시작|
|`done`|반복문 종료|
|`exit 0`|정상 종료|
|`exit 1`|에러 종료|
|`[ -d "경로" ]`|디렉토리 존재 확인|
|`[ -f "파일" ]`|파일 존재 확인|