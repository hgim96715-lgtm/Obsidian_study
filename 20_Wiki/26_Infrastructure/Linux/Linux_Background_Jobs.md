---
aliases:
  - 백그라운드
  - nohup
  - jobs
  - fg
  - bg
tags:
  - Linux
related:
  - "[[00_Linux_HomePage]]"
  - "[[Linux_Process_Monitor]]"
  - "[[Linux_Shell_Cron_Job]]"
  - "[[Linux_Redirect]]"
---

# Linux_Background_Jobs — 백그라운드 작업 제어

## 한 줄 요약

```
포그라운드 = 터미널 차지 / 끝날 때까지 대기
백그라운드 = 터미널 자유 / 동시에 다른 작업 가능
nohup      = 터미널 꺼져도 계속 실행
```

---

---

# ① 포그라운드 vs 백그라운드

```
포그라운드 (foreground):
  명령어 실행 → 터미널이 멈춤
  작업 끝날 때까지 다른 명령어 입력 불가
  Ctrl-C 로 중단 가능

백그라운드 (background):
  명령어 실행 → 터미널 바로 반환
  동시에 다른 명령어 입력 가능
  대용량 파일 복사 / 압축 / 학습 작업 등에 활용
```

```bash
# 포그라운드 실행 (기본)
sleep 300
# → 터미널이 300초 동안 멈춤

# 백그라운드 실행 (&)
sleep 300 &
# [1] 482         ← [Job ID] PID
# → 즉시 프롬프트 반환, 다른 명령어 입력 가능
```

---

---

# ② & — 백그라운드 실행 ⭐️

```bash
# 명령어 끝에 & 붙이기
sleep 300 &
# [1] 482
# ↑    ↑
# Job ID  PID

# 여러 개 실행
sleep 100 &
sleep 200 &
sleep 300 &
# [1] 480
# [2] 481
# [3] 482
```

## Job ID vs PID 차이

```
PID (Process ID):
  시스템 전체에서 고유한 번호
  ps aux 로 확인
  kill 482 (PID 로 종료)

Job ID:
  현재 셸 세션에서만 유효한 번호
  jobs 로 확인
  kill %1 (% + Job ID 로 종료)
  셸 세션 종료하면 사라짐
```

---

---

# ③ jobs — 백그라운드 작업 목록 ⭐️

```bash
jobs
# [1]  - running    sleep 100
# [2]  - running    sleep 200
# [3]  + running    sleep 300
#  ↑                ↑
# Job ID            상태

# 상태 의미:
#   + → 현재 포그라운드로 가져올 기본 작업
#   - → 다음 기본 작업
#   running   → 실행 중
#   stopped   → 일시 중지 (Ctrl-Z)
#   done      → 완료
```

```bash
# 상세 정보 (PID 포함)
jobs -l
# [1]  480 running    sleep 100
# [2]  481 running    sleep 200
# [3]+ 482 running    sleep 300
```

---

---

# ④ fg / bg — 포그라운드 ↔ 백그라운드 전환 ⭐️

## fg — 백그라운드 → 포그라운드

```bash
fg         # + 표시된 기본 작업 포그라운드로
fg %1      # Job ID 1 번을 포그라운드로
fg %2      # Job ID 2 번을 포그라운드로
```

## Ctrl-Z — 일시 중지

```bash
# fg 로 포그라운드에 가져온 후
sleep 300 &
fg %1
# 이제 sleep 이 포그라운드에서 실행 중

Ctrl-Z     # 일시 중지 (SIGTSTP 신호)
# [1]+  Stopped    sleep 300

jobs
# [1]+  stopped    sleep 300   ← 중지 상태
```

```
Ctrl-Z:
  프로세스를 종료하는 게 아님
  일시 중지(suspend) 상태로 만드는 것
  → bg 로 다시 실행 가능
  → kill %1 로 종료 가능
```

## bg — 중지된 작업 → 백그라운드 재개

```bash
# Ctrl-Z 로 중지 후
bg %1
# [1]+ 482 continued  sleep 300

jobs
# [1]+  running    sleep 300   ← 다시 실행 중
```

## 실전 패턴 ⭐️ — & 깜빡했을 때

```bash
# & 없이 실행했는데 오래 걸림
python train.py     # 터미널 멈춤

# 당황하지 말고:
Ctrl-Z              # 일시 중지
bg                  # 백그라운드로 밀어넣기
# 터미널 사용 가능
```

---

---

# ⑤ nohup — 터미널 꺼져도 계속 실행 ⭐️

```
문제:
  백그라운드(&) 실행해도
  터미널 창 닫으면 → 프로세스도 종료됨
  (SIGHUP 신호가 자식 프로세스에게 전달)

해결:
  nohup = No Hang Up
  터미널과 프로세스를 분리
  터미널 닫아도 프로세스 계속 실행
```

## 기본 사용

```bash
# nohup + 명령어 + & (백그라운드 필수)
nohup python train.py &
# nohup: ignoring input and appending output to 'nohup.out'
# [1] 482

# 출력은 nohup.out 에 저장
tail -f nohup.out   # 실시간 확인
```

## 출력 파일 지정

```bash
# 출력 파일 직접 지정
nohup python train.py > output.log 2>&1 &
#                      ↑           ↑   ↑
#                   표준출력     에러도 같이  백그라운드

# 출력 버리기
nohup python train.py > /dev/null 2>&1 &
```

## 2>&1 상세 ⭐️

```
> output.log    stdout(1) 을 파일로 저장
2>&1            stderr(2) 를 stdout(1) 이 가는 곳(파일)으로 합침
                → 에러 메시지도 같은 로그 파일에 기록

결과:
  일반 출력 → output.log
  에러 출력 → output.log  (같은 파일)

순서 중요 ⚠️:
  ✅ > log.txt 2>&1    stdout 먼저 → 파일로, stderr 도 그쪽으로
  ❌ 2>&1 > log.txt    stderr 가 현재 stdout(터미널)으로 → 의도와 다름

직접 실행 예시:
  nohup ./data_processor.sh > processor.log 2>&1 &
  tail -f processor.log    ← 일반 출력 + 에러 동시에 확인 가능
```

## 실행 중 확인 / 종료

```bash
# 실행 중 확인
ps aux | grep python
jobs   # 같은 세션이면 jobs 에도 보임

# 종료
kill %1
kill 482   # PID 로
```

## & vs nohup 비교 ⭐️

```
명령어 &:
  백그라운드 실행 O
  터미널 닫으면 종료 X (SIGHUP 받으면 종료)
  같은 세션에서만 유효

nohup 명령어 &:
  백그라운드 실행 O
  터미널 닫아도 계속 실행 O
  서버 배포 / 장시간 실행 작업에 적합
```

---

---

# ⑥ disown — 이미 실행 중인 프로세스를 분리

```
& 로 실행했는데 나중에 터미널 닫아야 할 때
nohup 없이 이미 실행 중인 프로세스 분리
```

```bash
# 이미 백그라운드로 실행 중
python train.py &
# [1] 482

# 셸에서 분리 (터미널 닫아도 살아남음)
disown %1
disown 482   # PID 로

# 모든 작업 분리
disown -a
```

---

---

# ⑦ 작업 제어 전체 흐름 한눈에

```
명령어 실행 (포그라운드)
  ↓
오래 걸림 → Ctrl-Z (일시 중지)
  ↓
bg (백그라운드 재개)  또는  fg (다시 포그라운드)
  ↓
종료: kill %1  또는  Ctrl-C (포그라운드일 때)

처음부터 백그라운드:
  명령어 & → 즉시 백그라운드

터미널 닫아도 유지:
  nohup 명령어 &
  또는 disown (이미 실행 중일 때)
```

## 신호 정리

|신호|단축키|의미|
|---|---|---|
|SIGINT (2)|Ctrl-C|포그라운드 프로세스 종료|
|SIGTSTP (20)|Ctrl-Z|포그라운드 프로세스 일시 중지|
|SIGHUP (1)|터미널 닫힘|터미널 연결 끊김 → 프로세스 종료|
|SIGTERM (15)|kill|정상 종료 요청|
|SIGKILL (9)|kill -9|강제 종료|

---

---

# 자주 하는 실수

|실수|원인|해결|
|---|---|---|
|터미널 닫았더니 프로세스 종료|& 만 씀|`nohup` 사용|
|Ctrl-Z 후 프로세스 사라진 줄 앎|일시 중지 모름|`jobs` 확인 후 `bg` 또는 `fg`|
|& 없이 nohup 사용|포그라운드로 실행됨|`nohup 명령어 &` 항상 & 붙이기|
|jobs 에 아무것도 안 보임|다른 세션에서 실행됨|`ps aux \| grep 명령어` 로 확인|
|nohup.out 용량 커짐|출력이 계속 쌓임|`> output.log 2>&1` 로 파일 지정 또는 `/dev/null`|