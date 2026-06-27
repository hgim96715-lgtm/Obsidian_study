---
aliases:
  - cron
  - crontab
  - 크론잡
  - 스케줄러
  - 자동화
tags:
  - Linux
related:
  - "[[00_Linux_HomePage]]"
  - "[[Linux_Archive_Compress]]"
  - "[[Linux_Background_Jobs]]"
  - "[[Linux_Shell_Script_Basics]]"
---

# Shell_Cron_Job — cron 스케줄러

## 한 줄 요약

```txt
cron     = 지정한 시간에 명령을 자동 실행하는 스케줄러
crontab  = 사용자별 cron 작업 목록
"기억에 의존하는 백업은 이미 망한 백업이다"
→ cron 으로 자동화
```

---

---

# ① crontab 기본 명령어

```bash
crontab -e    # 편집 (등록/수정)
crontab -l    # 목록 확인
crontab -r    # 전체 삭제 ⚠️
```

```txt
crontab -e 처음 실행 시 편집기 선택
  1 = nano (추천)
  2 = vim
```

---

---

# ② cron 표현식 — 5자리 ⭐️

```txt
*  *  *  *  *  실행할 명령어
│  │  │  │  │
│  │  │  │  └── 요일 (0=일요일 ~ 6=토요일)
│  │  │  └───── 월 (1~12)
│  │  └──────── 일 (1~31)
│  └─────────── 시 (0~23)
└────────────── 분 (0~59)

* = 모든 값 (매 분 / 매 시 / 매일...)
```

## 자주 쓰는 패턴

```bash
# 매 분
* * * * * 명령어

# 매 시간 정각
0 * * * * 명령어

# 매일 오전 2시
0 2 * * * 명령어

# 매일 자정 (00:00)
0 0 * * * 명령어

# 매주 일요일 오전 3시
0 3 * * 0 명령어

# 매월 1일 오전 1시
0 1 1 * * 명령어

# 5분마다
*/5 * * * * 명령어

# 평일(월~금) 오전 9시
0 9 * * 1-5 명령어
```

## 크론 표현식 읽는 법

```txt
0 2 * * *   → 매일 새벽 2시 0분
0 0 * * 0   → 매주 일요일 00시 00분
*/30 * * * * → 30분마다 (0, 30분)
0 9-18 * * 1-5 → 평일 9시~18시 매 시간 정각
```

---

---

# ③ 실전 패턴 — 로그 압축 자동화 ⭐️

## 매일 자동 백업

```bash
crontab -e
# 아래 내용 추가:

# 매일 새벽 2시 백업
0 2 * * * tar -czf /backup/logs_$(date +\%Y\%m\%d).tar.gz /var/log/myapp/

# 매주 일요일 자정 전체 백업
0 0 * * 0 tar -czf /backup/full_$(date +\%Y\%m\%d).tar.gz /home/ubuntu/project/
```

## 날짜 타임스탬프 패턴 ⭐️

```bash
# 터미널에서 직접 실행 (% 그대로)
tar -czf backup_$(date +%Y-%m-%d_%H-%M-%S).tar.gz /data/

# cron 에서 (% → \% 이스케이프 필수!)
0 2 * * * tar -czf /backup/backup-$(date +\%Y-\%m-\%d_\%H-\%M-\%S).tar.gz /data/
```

```txt
cron 에서 % 주의사항 ⭐️:
  % = 줄바꿈 특수문자로 인식
  → \% 로 이스케이프 해야 문자로 인식

  터미널:   date +%Y-%m-%d    ← % 그대로
  crontab: date +\%Y-\%m-\%d  ← \ 붙이기

날짜 포맷:
  %Y = 2026   (연도)
  %m = 04     (월)
  %d = 20     (일)
  %H = 10     (시)
  %M = 30     (분)
  %S = 00     (초)

결과: backup-2026-04-20_10-30-00.tar.gz
→ 파일명이 겹치지 않음 (덮어쓰기 방지)
```

## 여러 폴더 일괄 백업

```bash
# 매일 새벽 2시 — data, config, logs 백업
0 2 * * * tar -czf /backup/system-$(date +\%Y\%m\%d).tar.gz -C /home/ubuntu/project data config logs
```

---

---

# ④ 로그 출력 저장

```bash
# cron 실행 결과를 로그로 저장
0 2 * * * /opt/backup.sh >> /var/log/backup.log 2>&1

# /dev/null 로 출력 버리기 (조용히 실행)
0 2 * * * /opt/backup.sh > /dev/null 2>&1
```

```txt
>> log 2>&1:
  >> log     stdout 을 로그 파일에 이어쓰기
  2>&1       stderr 도 같은 파일로
  → 성공/실패 모두 로그에 기록
```

---

---

# ⑤ 실전 백업 자동화 전체 패턴 ⭐️

## 매니페스트 파일 — 백업 대상 목록 관리

```bash
# backup-list.txt 생성 (백업 대상 경로 목록)
nano ~/project/backup-list.txt
# 내용:
# /home/labex/project/data
# /home/labex/project/config
# /home/labex/project/logs

# -T 옵션으로 목록 파일 읽어서 백업
tar -czvf backups/system-backup.tar.gz -T backup-list.txt
```

```txt
-T 옵션이 유용한 이유:
  백업 경로가 많을 때 명령줄에 일일이 쓰면 오타 위험
  backup-list.txt 에 한 줄씩 관리
  스크립트/cron 에서 재사용 가능

-czvf 옵션 순서:
  c = 생성 / z = gzip / v = 출력 / f = 파일명 (f는 항상 마지막)
```

## 백업 무결성 검증 ⭐️

```bash
# 1. 백업 생성
tar -czvf backups/system-backup.tar.gz -T backup-list.txt

# 2. 내용 목록 확인 (압축 풀지 않고)
tar -tzvf backups/system-backup.tar.gz > backup-contents.txt
cat backup-contents.txt

# 3. 검증 완료 후 원본 정리 or 보관
```

```txt
백업은 만드는 것보다 검증이 더 중요
-t 옵션 = 압축 풀지 않고 내용 목록만 확인
→ 파일이 제대로 들어있는지 먼저 확인
→ 빈 껍데기 백업 방지
```

## 특정 파일만 부분 복원 ⭐️

```bash
# 전체 압축 해제 대신 특정 파일 하나만 복원
tar -xzvf backups/system-backup.tar.gz config/app.conf

# 확인
ls -l ~/project/config/app.conf
cat ~/project/config/app.conf
```

```txt
부분 복원이 필요한 이유:
  수십 GB 전체 백업을 풀면 복구 시간(Downtime) 급증
  장애 원인 파일 하나만 콕 집어서 복원 → 빠른 복구

사용 패턴:
  tar -xzvf 아카이브명 복원할파일경로
  아카이브 안의 경로 기준으로 명시
```

## cron 자동 백업 — 타임스탬프 네이밍

```bash
# crontab -e 에 등록
* * * * * tar -czf /home/labex/project/backups/backup-$(date +\%Y-\%m-\%d_\%H-\%M-\%S).tar.gz \
  -C /home/labex/project data config logs
```

```txt
-C 옵션 = 기준 디렉토리 지정
  tar -czf backup.tar.gz -C /home/labex/project data config logs
                                ↑ 이 경로 기준으로         ↑ 이 폴더들 압축
  → 아카이브 안에 절대경로 대신 상대경로로 저장됨
  → 복원 시 경로 꼬임 방지

타임스탬프 네이밍:
  같은 이름으로 덮어씌워지는 것 방지
  backup-2026-04-20_02-00-00.tar.gz 처럼 날짜별 보관

cron 에서 % → \% 이스케이프 필수 ⚠️:
  터미널:   date +%Y-%m-%d
  crontab: date +\%Y-\%m-\%d
```

---

---

# ⑤ crontab 설정 및 확인

```bash
# 등록
crontab -e

# 등록된 목록 확인
crontab -l
# 0 2 * * * tar -czf /backup/...
# */30 * * * * /opt/health_check.sh

# cron 서비스 상태 확인
sudo systemctl status cron
sudo systemctl status crond    # RHEL 계열

# 실행 로그 확인
grep CRON /var/log/syslog
```

---

---

# ⑥ 자주 쓰는 cron 자동화 패턴

## 오래된 파일 자동 삭제

```bash
# 매일 자정 30일 이상 된 로그 삭제
0 0 * * * find /var/log/myapp -name "*.log" -mtime +30 -delete
```

## Airflow DAG 자동 백업

```bash
# 매주 일요일 새벽 1시 DAG 백업
0 1 * * 0 tar -czf /backup/dags_$(date +\%Y\%m\%d).tar.gz /opt/airflow/dags/
```

## 헬스체크

```bash
# 5분마다 서버 상태 체크
*/5 * * * * ping -c 1 8.8.8.8 > /dev/null 2>&1 || echo "네트워크 이상 $(date)" >> /var/log/health.log
```

---

---

# 명령어 한눈에

|명령어|역할|
|---|---|
|`crontab -e`|cron 작업 편집|
|`crontab -l`|등록된 작업 목록|
|`crontab -r`|전체 삭제 ⚠️|
|`0 2 * * *`|매일 새벽 2시|
|`*/5 * * * *`|5분마다|
|`0 0 * * 0`|매주 일요일 자정|
|`date +\%Y\%m\%d`|cron 에서 날짜 포맷|

---

---

# 자주 하는 실수

|실수|원인|해결|
|---|---|---|
|cron 실행 안 됨|표현식 순서 틀림|분 시 일 월 요일 순서 확인|
|날짜 파일명 안 찍힘|% 이스케이프 빠짐|`\%` 로 변경|
|경로 못 찾음|상대 경로 사용|cron 은 절대 경로 사용|
|실행 결과 모름|로그 없음|`>> /var/log/cron.log 2>&1` 추가|
|crontab -r 로 전부 삭제|-r 과 -e 혼동|삭제 전 `crontab -l` 로 확인|