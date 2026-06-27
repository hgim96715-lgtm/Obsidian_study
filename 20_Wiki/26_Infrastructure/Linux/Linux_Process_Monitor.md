---
aliases:
  - 프로세스 모니터링
  - top
  - htop
  - ps aux
  - Load Average
tags:
  - Linux
related:
  - "[[00_Linux_HomePage]]"
  - "[[Linux_System_Info]]"
  - "[[Linux_Background_Jobs]]"
---


# Linux_Process_Monitor — 프로세스 모니터링

## 한 줄 요약

```
ps aux  = 현재 프로세스 스냅샷 (정적)
top     = 실시간 모니터링 (동적)
htop    = top 의 예쁜 버전
kill    = 프로세스 종료
```

---

---

# ① PID 개념

```
PID = Process ID
  각 프로세스에 부여되는 고유 번호
  부팅 시 1번 (systemd) 부터 시작

프로세스 상태:
  R  Running   → CPU 사용 중
  S  Sleeping  → 대기 중 (이벤트 기다림)
  D  Disk wait → 디스크 I/O 대기 (인터럽트 불가)
  Z  Zombie    → 종료됐지만 부모가 회수 안 함
  T  Stopped   → 일시 중지 (Ctrl-Z)
```

---

---

# ② ps — 프로세스 스냅샷 ⭐️

```bash
# 전체 프로세스 (가장 많이 씀)
ps aux

# 특정 프로세스 찾기
ps aux | grep python
ps aux | grep nginx
ps aux | grep -v grep   # grep 자체 제외

# 부모-자식 관계 확인
ps -ef | grep airflow

# CPU 많이 쓰는 순
ps aux --sort=-%cpu | head -10

# 메모리 많이 쓰는 순
ps aux --sort=-%mem | head -10
```

## ps aux 출력 해석

```
USER    PID   %CPU  %MEM   VSZ    RSS   TTY  STAT  TIME  COMMAND
root      1    0.0   0.1  168M   13M    ?    Ss   0:01  /sbin/init
labex   482    0.0   0.0    0     0    pts/0  S   0:00  sleep 300
python  888    5.2   2.3  450M  180M   ?      S   0:45  python3 app.py

%CPU  = CPU 사용률
%MEM  = 메모리 사용률
VSZ   = 가상 메모리 크기
RSS   = 실제 물리 메모리 사용량
STAT  = 프로세스 상태 (S=sleep, R=run, Z=zombie)
```

---

---

# ③ top — 실시간 모니터링 ⭐️

```bash
top           # 실행
htop          # 더 예쁜 버전 (sudo apt install htop)
```

## top 단축키

```
P   CPU 사용량 순 정렬 (기본값)
M   메모리 사용량 순 정렬
k   프로세스 종료 (PID 입력)
r   renice (우선순위 변경)
1   CPU 코어별 사용량
q   종료
↑↓  목록 스크롤
```

## top 상단 영역 해석 ⭐️

```
top - 10:30:00 up 5 days, 2:30,  1 user,  load average: 0.52, 0.41, 0.35
      ↑ 현재시각  ↑ 가동시간      ↑ 접속자   ↑ 1분    ↑ 5분   ↑ 15분 부하

Tasks: 120 total,   1 running, 119 sleeping,   0 stopped,   0 zombie
       ↑ 전체         ↑ 실행중    ↑ 대기중           ↑ 중지       ↑ 좀비

%Cpu(s):  5.0 us,  1.0 sy,  0.0 ni, 93.0 id,  1.0 wa,  0.0 hi,  0.0 si
           ↑ 사용자   ↑ 시스템   ↑ nice  ↑ 유휴    ↑ I/O대기

MiB Mem:  7800 total,  3200 free,  3500 used,  1100 buff/cache
```

## Load Average 해석 ⭐️

```
load average: 1분, 5분, 15분 평균 부하

CPU 코어 수 확인:
  nproc   # 코어 수 출력

부하 판단:
  부하 / 코어수 < 1.0  → 여유 있음
  부하 / 코어수 ≈ 1.0  → 포화 상태
  부하 / 코어수 > 1.0  → 과부하

예시 (4코어):
  load: 2.0 → 2.0/4 = 50% 사용 (여유)
  load: 4.0 → 4.0/4 = 100% 사용 (포화)
  load: 8.0 → 8.0/4 = 200% (과부하)

트러블슈팅:
  1분 부하 >> 15분 부하 → 갑작스러운 과부하
  → top 에서 P 눌러 CPU 많이 쓰는 프로세스 찾기
```

---

---

# ④ kill — 프로세스 종료 ⭐️

```bash
# 기본 종료 요청 (SIGTERM)
kill PID
kill 1234

# 강제 종료 (SIGKILL)
kill -9 PID
kill -9 1234

# 이름으로 종료
killall python3      # python3 프로세스 전부
pkill -f "script.py" # 명령어 패턴으로 찾아서 종료
```

## pgrep — 이름으로 PID 찾기 ⭐️

```bash
# 프로세스 이름으로 PID 만 출력
pgrep nginx
# 1234
# 1235

# 전체 명령줄 포함 검색 (-f)
pgrep -f critical_service.sh
# 200

# 프로세스 이름과 PID 함께 출력
pgrep -l nginx
# 1234 nginx
```

```
pgrep vs ps aux | grep:
  ps aux | grep nginx  → 출력이 길고 grep 자신도 포함됨
  pgrep nginx          → PID 숫자만 깔끔하게 출력

  종료하기 전 PID 확인용으로 자주 씀
```

## ps -p — 특정 PID 상세 조회 ⭐️

```bash
# 특정 PID 의 상세 정보 출력
ps -p 200
ps -p 200 -o pid,ppid,cmd
#     ↑ 원하는 컬럼만 지정
# PID  PPID  CMD
# 200   100  bash critical_service.sh

# 컬럼 옵션
# pid   = 프로세스 ID
# ppid  = 부모 프로세스 ID
# cmd   = 실행 명령어
# %cpu  = CPU 사용률
# %mem  = 메모리 사용률
```

```
종료 전 신원 확인 루틴:
  1. pgrep -f 스크립트명     → PID 확인
  2. ps -p PID -o pid,ppid,cmd → 정확한 대상인지 검증
  3. kill PID               → 안전하게 종료
```

## pkill — 이름으로 종료 ⭐️

```bash
# 이름으로 종료 (PID 몰라도 됨)
pkill nginx

# 전체 명령줄 패턴으로 종료 (-f)
pkill -f resource_hog.sh
pkill -f "data_processor"

# 종료 확인
ps aux | grep resource_hog.sh
# grep 자신만 남아있으면 종료 성공
```

```
⚠️ pkill 주의사항:
  pkill python  → 실행 중인 모든 python 프로세스 종료!
  너무 광범위한 이름 사용 금지

  안전한 순서:
  pgrep -f 정확한패턴  → 대상 확인
  pkill -f 정확한패턴  → 종료
```

## SIGTERM vs SIGKILL

```
kill (SIGTERM):
  "정중하게 종료해달라" 요청
  프로세스가 정리 후 종료
  DB 연결 끊기 / 파일 저장 후 종료
  거부할 수 있음

kill -9 (SIGKILL):
  OS 가 즉시 강제 종료
  정리 없이 죽임 → 데이터 손실 위험
  거부 불가

원칙:
  먼저 kill (SIGTERM) 시도
  응답 없으면 kill -9 사용
```

---

---

# ⑤ 좀비 프로세스 (Zombie)

```
좀비 프로세스:
  자식 프로세스가 종료됐는데
  부모 프로세스가 종료 상태를 아직 읽지 않은 것

  ps 에서 Z (Zombie) 또는 defunct 로 표시
  실제로 CPU/메모리 쓰지 않음 → 직접적 해는 없음
  하지만 PID 테이블 점유 → 너무 많으면 새 프로세스 생성 불가

확인:
  ps aux | grep -w Z
  ps aux | grep defunct

해결:
  부모 프로세스 재시작 또는 종료
  → 부모가 종료되면 좀비도 init 이 회수
  kill -9 로 직접 종료 불가 (이미 종료된 상태)
```

---

---

# ⑥ 실전 패턴

## 서버 과부하 트러블슈팅

```bash
# 1. 현재 부하 확인
uptime
# load average: 8.5, 6.2, 3.1  → 최근 과부하 발생

# 2. CPU 독식 프로세스 확인
top   # P 눌러 CPU 순 정렬
ps aux --sort=-%cpu | head -5

# 3. 메모리 확인
free -h
ps aux --sort=-%mem | head -5

# 4. 문제 프로세스 종료
kill PID
kill -9 PID   # 응답 없으면
```

## 특정 서비스 프로세스 확인

```bash
# Airflow 관련 프로세스
ps aux | grep airflow | grep -v grep

# Python 스크립트 실행 중인지
ps aux | grep "script.py"

# 포트 사용 중인 프로세스
ss -tulnp | grep :8080
```

---

---

# 명령어 한눈에

|명령어|역할|
|---|---|
|`ps aux`|전체 프로세스 스냅샷|
|`ps aux \| grep 이름`|특정 프로세스 찾기|
|`ps aux --sort=-%cpu \| head -10`|CPU 많이 쓰는 순|
|`top`|실시간 모니터링|
|`htop`|top 의 예쁜 버전|
|`kill PID`|정상 종료 요청|
|`kill -9 PID`|강제 종료|
|`killall 이름`|이름으로 전부 종료|
|`pkill -f 패턴`|패턴으로 종료|

---

---

# 자주 하는 실수

|실수|원인|해결|
|---|---|---|
|kill -9 먼저 사용|습관|먼저 `kill` 시도 후 -9|
|ps grep 결과에 grep 자체 포함|grep 도 프로세스|`grep -v grep` 추가|
|좀비 kill -9 해도 안 죽음|이미 종료된 상태|부모 프로세스 종료|
|Load Average 코어 수 무시|절대값으로만 판단|`nproc` 으로 코어 수 확인 후 비율 계산|