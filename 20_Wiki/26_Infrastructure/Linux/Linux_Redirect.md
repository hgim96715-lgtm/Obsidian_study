---
aliases:
  - 리다이렉션
  - 표준 입출력
  - stderr
  - 파이프
  - << EOF — 히어독
  - ">"
  - ">>"
  - 2>
  - 2>&1
  - /dev/null
tags:
  - Linux
related:
  - "[[00_Linux_HomePage]]"
  - "[[Linux_System_Info]]"
  - "[[Linux_Text_Commands]]"
  - "[[Linux_Search_Filter]]"
  - "[[Linux_Diff]]"
---
# Linux_Redirect — 입출력 리다이렉션

## 한 줄 요약

```txt
>   = 덮어쓰기 (기존 내용 삭제)
>>  = 이어쓰기 (기존 내용 뒤에 추가)
<   = 파일을 입력으로
|   = 앞 명령어 출력을 뒤 명령어 입력으로
2>  = 에러만 저장
```

---

---

#  표준 입출력 개념

```txt
리눅스의 3가지 스트림:
  stdin  (0) = 표준 입력  (키보드 → 프로그램)
  stdout (1) = 표준 출력  (프로그램 → 화면)
  stderr (2) = 표준 에러  (에러 메시지 → 화면)

리다이렉션 = 이 스트림의 방향을 바꾸는 것
  화면 대신 파일로 / 키보드 대신 파일에서
```

---

---

#  > — 덮어쓰기 (Overwrite)

```bash
# 명령어 결과를 파일에 저장 (기존 내용 삭제!)
echo "hello" > output.txt
ls -l > file_list.txt
date > timestamp.txt

# 파일 내용 확인
cat output.txt   # hello
```

```txt
⚠️ > 는 파일을 먼저 비우고 씀
  이미 내용 있는 파일에 > 사용 → 기존 내용 전부 삭제
  주의해서 사용
```

---

---

#  >> — 이어쓰기 (Append) ⭐️

```bash
# 기존 파일 내용 뒤에 추가
echo "line 1" > log.txt       # 파일 생성
echo "line 2" >> log.txt      # 이어붙임
echo "line 3" >> log.txt      # 이어붙임

cat log.txt
# line 1
# line 2
# line 3
```

## 보고서 자동 생성 패턴 ⭐️

```bash
# 시스템 상태 보고서 생성
touch system_report.txt            # 빈 파일 생성

whoami   >> system_report.txt      # 사용자 추가
uname -a >> system_report.txt      # 커널 정보 추가
uptime   >> system_report.txt      # 가동 시간 추가
date     >> system_report.txt      # 현재 시각 추가
df -h    >> system_report.txt      # 디스크 사용량 추가

cat system_report.txt              # 전체 확인
```

```txt
> vs >> 핵심 차이:
  >  한 번만 사용 → 파일 생성 / 덮어쓰기
  >> 계속 추가    → 로그 누적 / 보고서 작성

  보고서 / 로그 파일: >> 사용
  매번 새로 쓰는 상태 파일: > 사용
```

---

---

# < — 파일을 입력으로 (Input Redirection) ⭐️⭐️

```txt
> 는 "명령어의 출력을 어디로 보낼지" 였다면
< 는 그 반대 — "명령어의 입력을 어디서 받을지" 를 바꾸는 것

기본적으로 stdin 은 키보드인데, < 를 쓰면
셸이 그 파일을 직접 열어서 명령어의 stdin 에 연결해줌
(명령어 자신이 파일을 여는 게 아니라, 셸이 미리 열어서 떠먹여주는 것)
```

```bash
sort < unsorted.txt
wc -l < data.txt
mysql -u root < dump.sql
```

## ⚠️ 인자로 주는 것과 뭐가 다른가 — 핵심 ⭐️⭐️

```bash
wc -l access.log     # 파일명을 "인자" 로 넘김
# → 150 access.log   ← 파일명이 같이 출력됨

wc -l < access.log   # 파일을 "입력(stdin)" 으로 넘김
# → 150               ← 숫자만 출력됨 (wc 는 파일명 자체를 모름)
```

```txt
왜 결과가 다른가:
  wc -l access.log
    → wc 가 "access.log" 라는 인자를 직접 받음
    → wc 가 그 파일을 스스로 열어서 읽고, 결과에 파일명까지 같이 보여줌

  wc -l < access.log
    → wc 는 인자를 아무것도 안 받음 (커맨드 자체엔 파일명이 없음)
    → 셸이 access.log 를 미리 열어서 wc 의 stdin 에 그냥 흘려보냄
    → wc 입장에서는 "어디서 왔는지 모르는 데이터" 를 읽은 것뿐 → 파일명을 보여줄 수가 없음

언제 < 를 쓰는 게 유리한가:
  결과에 파일명이 안 섞인, "순수한 값" 만 필요할 때 (숫자 비교, 변수에 저장 등)
  명령어가 애초에 파일 경로를 인자로 못 받고 stdin 으로만 입력받게 설계된 경우
    (예: mysql -u root < dump.sql → mysql 클라이언트는 SQL 파일을 "인자" 로 안 받고
     표준입력으로 SQL 명령들을 읽어서 실행하는 방식이라 < 가 거의 필수)
```

## 실전 — < 와 > 동시에 쓰기 ⭐️⭐️

```bash
wc -l < access.log > task1_output.txt
```

```txt
한 줄에 방향이 다른 리다이렉션 두 개가 같이 있음 — 각자 역할이 다름:

  < access.log         access.log 의 내용을 wc 의 입력(stdin)으로 흘려보냄
  > task1_output.txt   wc 의 출력(stdout)을 task1_output.txt 에 저장

  결과: task1_output.txt 안에는 "150" 처럼 숫자만 딱 들어감 (파일명 없음)

비교 — < 없이 그냥 인자로 줬다면:
  wc -l access.log > task1_output.txt
  → task1_output.txt 안에 "150 access.log" 가 들어감 (파일명이 같이 저장됨)

→ 결과 파일에 숫자만 깔끔하게 남기고 싶다면 < 를 쓰는 게 정답
  (스크립트에서 그 숫자를 변수로 다시 읽어들일 때, 파일명이 안 섞여 있어야 편함)
```

---
---
#  2> — 에러 리다이렉션

```bash
# 에러 메시지만 파일에 저장
ls /없는경로 2> error.log
# 화면에는 아무것도 안 나옴 / error.log 에 에러 저장

# 에러 파일 확인
cat error.log
# ls: cannot access '/없는경로': No such file or directory

# 에러 추가 모드
ls /없는경로 2>> error.log
```

---

---

#  2>&1 — 표준출력 + 에러 동시에 ⭐️

```bash
# stdout + stderr 모두 같은 파일에 저장
command >> all.log 2>&1

# 예시: 크론탭 로그에서 자주 씀
python3 script.py >> /var/log/myapp.log 2>&1
#                    ↑ 일반 출력          ↑ 에러도 같이

# /dev/null 로 버리기 (출력 무시)
noisy_command > /dev/null 2>&1   # 모든 출력 버림
noisy_command 2> /dev/null       # 에러만 버림
```

```txt
2>&1 읽는 법:
  2  = stderr
  >  = 리다이렉션
  &1 = stdout 과 같은 곳으로

  순서 주의:
  >> log.txt 2>&1   ✅ stdout → 파일, stderr → stdout과 같은 파일
  2>&1 >> log.txt   ❌ 의도와 다름
```

## &> — stdout + stderr 단축형 ⭐️

```bash
# > file 2>&1 와 동일 (bash 단축형)
ls -l . nonexistent &> combined.log
#       ↑ stdout + stderr 모두 combined.log 에 저장

# /dev/null 로 모두 버리기
noisy_command &> /dev/null

# 순서 비교
ls . nonexistent > combined.log 2>&1   # 명시적 (권장)
ls . nonexistent &> combined.log       # 단축형 (동일)
```

---

---

# /dev/null — 블랙홀 ⭐️

```txt
/dev/null = 특수 파일
쓴 데이터는 모두 사라짐 / 읽으면 항상 빈 내용
"비트 버킷(bit bucket)" / "블랙홀"
```

## 출력 버리기

```bash
# stdout 버리기
ls -l > /dev/null

# stderr 만 버리기 (에러 메시지 숨기기)
ls nonexistent 2> /dev/null

# stdout + stderr 모두 버리기
ls . nonexistent > /dev/null 2>&1
ls . nonexistent &> /dev/null     # 단축형
```

## 파일 내용 비우기 ⭐️

```bash
# 파일 내용을 지우면서 파일은 유지
cat /dev/null > app.log
# /dev/null (빈 것) 을 app.log 에 덮어씀 → 0 바이트

# 동일한 결과
> app.log           # 빈 리다이렉션 (더 짧음)
truncate -s 0 app.log

# 실전: 로그 파일 주기적으로 비우기
cat /dev/null > /var/log/app.log
```

## 파일 존재 여부 테스트

```bash
# 출력 없이 파일 존재 확인
if cp test.c /dev/null 2> /dev/null; then
  echo "파일이 존재하고 읽을 수 있음"
else
  echo "파일이 없거나 읽기 불가"
fi
# cp 성공 → /dev/null 로 버려짐 → 파일 확인용
```

---

---

```bash
# 파일을 명령어의 입력으로
sort < unsorted.txt        # 파일 내용을 sort 에 전달
wc -l < data.txt          # 줄 수 세기
mysql -u root < dump.sql  # SQL 파일 실행
```

---

---

#  | — 파이프 (Pipe) ⭐️

```txt
앞 명령어의 출력 → 뒤 명령어의 입력
명령어를 조합해서 강력한 처리
```

```bash
# 기본 파이프
ls -l | grep ".py"           # .py 파일만 필터
ps aux | grep python         # python 프로세스만
cat log.txt | sort | uniq    # 정렬 후 중복 제거

# 여러 개 연결
cat /etc/passwd | cut -d: -f1 | sort   # 사용자 목록 정렬

# 파이프 + 파일 저장
ps aux | grep python > python_procs.txt

# less 로 페이지 단위 보기
ls -la | less
dmesg | less
```

## tee — 화면 + 파일 동시

```bash
# 화면에 출력하면서 파일에도 저장
ls -l | tee output.txt

# 이어쓰기 모드
ls -l | tee -a output.txt

# sudo 명령 결과를 시스템 파일에 저장
echo "설정값" | sudo tee /etc/설정파일
echo "module" | sudo tee -a /etc/modules-load.d/module.conf
```

---

---

#  자주 쓰는 조합 패턴

```bash
# 특정 에러만 로그에 추가
command 2>> error.log

# 크론탭 로그 패턴
0 2 * * * /opt/backup.sh >> /var/log/backup.log 2>&1

# 실시간 로그 보기
tail -f /var/log/myapp.log

# 파이프로 필터링
cat /var/log/syslog | grep ERROR | tail -20

# wc 로 줄 수 세기
cat error.log | wc -l
grep ERROR /var/log/app.log | wc -l
```

---

---

# 리다이렉션 한눈에

| 기호          | 의미            | 예시                      |
| ----------- | ------------- | ----------------------- |
| `>`         | 덮어쓰기          | `echo hi > file.txt`    |
| `>>`        | 이어쓰기          | `date >> log.txt`       |
| `<`         | 파일을 입력으로      | `sort < data.txt`       |
| `2>`        | 에러만 저장        | `cmd 2> err.log`        |
| `2>&1`      | 에러를 stdout 으로 | `cmd >> log.txt 2>&1`   |
| `\|`        | 파이프           | `ps aux \| grep python` |
| `tee`       | 화면 + 파일 동시    | `cmd \| tee output.txt` |
| `/dev/null` | 출력 버리기        | `cmd > /dev/null 2>&1`  |

---

---

# << EOF — 히어독 (여러 줄 입력) ⭐️

```txt
Heredoc = Here Document
여러 줄의 내용을 한 번에 파일에 입력할 때 사용
cat << EOF > 파일명 형태로 많이 씀
```

```bash
# 기본 구조
cat << EOF > multiline.txt
Line 1: Hello, Linux
Line 2: This is a multiline file.
Line 3: Created using a here-document.
EOF

cat multiline.txt
# Line 1: Hello, Linux
# Line 2: This is a multiline file.
# Line 3: Created using a here-document.
```

```txt
구조 설명:
  cat << EOF  → EOF 가 나올 때까지 입력 받아서 cat 으로 출력
  > 파일명    → cat 출력을 파일로 저장
  EOF         → 입력 끝을 알리는 마커 (대문자 관례 / 다른 단어도 가능)

echo 여러 번 vs heredoc:
  ❌ echo "line1" > file.txt
     echo "line2" >> file.txt
     echo "line3" >> file.txt

  ✅ cat << EOF > file.txt
     line1
     line2
     line3
     EOF
```

```bash
# >> 로 기존 파일에 이어붙이기
cat << EOF >> existing.txt
추가할 내용 1
추가할 내용 2
EOF

# 변수 사용 가능
NAME="Linux"
cat << EOF > greeting.txt
Hello, $NAME!
Today is $(date)
EOF
```

---

---

# 자주 하는 실수

|실수|원인|해결|
|---|---|---|
|`>` 로 기존 파일 날림|덮어쓰기|로그는 항상 `>>`|
|`2>&1 >> log` 순서 틀림|순서가 중요함|`>> log 2>&1` 순서|
|파이프 없이 긴 출력|화면 넘침|`\| less` 로 페이지 보기|
|sudo 결과 파일 저장 안 됨|권한 문제|`\| sudo tee -a 파일`|
|EOF 앞에 공백 있음|인식 못 함|EOF 는 줄 맨 앞에|