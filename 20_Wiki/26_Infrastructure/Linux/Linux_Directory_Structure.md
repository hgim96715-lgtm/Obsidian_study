---
aliases:
  - 디렉토리 구조
  - 파일시스템
  - 루트
tags:
  - Linux
related:
  - "[[00_Linux_HomePage]]"
  - "[[Linux_Concept_Overview]]"
  - "[[Linux_Directory_Commands]]"
  - "[[Linux_File_Move_Copy]]"
---

# Linux_Directory_Structure — 디렉토리 구조

## 한 줄 요약

```
Linux 는 / (루트) 하나에서 시작하는 트리 구조
Windows 의 C:\ D:\ 같은 드라이브 없음
"모든 것이 파일" — 장치도 파일, 설정도 파일
```

---

---

# ① 전체 트리 구조

```
/ (루트)
├── home/       사용자 홈 디렉토리
│   └── ubuntu/ 사용자별 폴더
├── etc/        설정 파일 모음
├── var/        자주 변하는 데이터 (로그, DB)
├── tmp/        임시 파일 (재부팅 시 삭제)
├── usr/        사용자 프로그램 / 라이브러리
├── bin/        기본 실행 파일 (ls, cp, mv)
├── sbin/       시스템 관리 명령어 (root 용)
├── opt/        서드파티 소프트웨어
├── dev/        장치 파일 (디스크, USB)
├── proc/       프로세스 / 커널 정보 (가상)
├── sys/        하드웨어 장치 정보 (가상)
├── boot/       부트로더 / 커널 이미지
├── lib/        공유 라이브러리
├── mnt/        임시 마운트 포인트
└── root/       root 사용자 홈
```

---

---

# ② 핵심 디렉토리 ⭐️

## /home — 사용자 홈

```bash
/home/ubuntu/     ← ubuntu 계정 홈
/home/labex/      ← labex 계정 홈

~  = 현재 사용자 홈 디렉토리 단축키
cd ~              # 홈으로 이동
ls ~/project      # 홈 아래 project 폴더

데이터 엔지니어 프로젝트 기본 위치:
  ~/project/
  ~/airflow/dags/
```

## /etc — 설정 파일 ⭐️

```bash
/etc/hosts            # IP ↔ 호스트명 매핑
/etc/ssh/sshd_config  # SSH 서버 설정
/etc/crontab          # 시스템 cron 스케줄
/etc/nginx/nginx.conf # Nginx 웹서버 설정
/etc/passwd           # 사용자 계정 정보
/etc/shadow           # 비밀번호 해시
/etc/fstab            # 파일시스템 마운트 설정
/etc/hosts            # 로컬 DNS

# 설정 파일 수정 전 반드시 백업
sudo cp /etc/nginx/nginx.conf /etc/nginx/nginx.conf.bak
```

## /var — 가변 데이터 ⭐️

```bash
/var/log/             # 로그 파일 모음
/var/log/syslog       # 시스템 로그
/var/log/auth.log     # 인증 로그
/var/log/nginx/       # Nginx 로그
/var/lib/postgresql/  # PostgreSQL 데이터
/var/run/             # 실행 중인 프로세스 PID

# 로그 확인
tail -f /var/log/syslog
ls -lh /var/log/
```

## /tmp — 임시 파일

```bash
/tmp/
  재부팅 시 자동 삭제
  누구나 읽기/쓰기 가능 (sticky bit)
  스크립트 중간 결과물 임시 저장

# 임시 파일 만들기
touch /tmp/test.txt
cp big_file.csv /tmp/   # 처리 중 임시 위치
```

## /opt — 서드파티 소프트웨어

```bash
/opt/airflow/         # Airflow 설치 위치
/opt/kafka/           # Kafka 설치 위치
/opt/spark/           # Spark 설치 위치
/opt/conda/           # Conda 설치 위치
```

## /proc — 프로세스 / 커널 정보

```bash
cat /proc/cpuinfo     # CPU 정보
cat /proc/meminfo     # 메모리 정보
cat /proc/version     # 커널 버전
ls /proc/1234/        # PID 1234 프로세스 정보
```

---

---

# ③ 절대 경로 vs 상대 경로 ⭐️

```
절대 경로 = / (루트) 부터 시작하는 전체 경로
상대 경로 = 현재 위치 기준 경로
```

```bash
# 절대 경로 — 어디서 실행해도 동일한 경로
/home/ubuntu/project/data.csv
/etc/nginx/nginx.conf
/var/log/syslog

# 상대 경로 — 현재 위치 기준
./data.csv            # 현재 폴더의 data.csv
../config/app.conf    # 한 단계 위 → config 폴더
../../logs/app.log    # 두 단계 위 → logs 폴더
```

## 특수 경로 기호

```bash
.     현재 디렉토리
..    부모 디렉토리 (한 단계 위)
~     홈 디렉토리 (/home/사용자)
-     직전 디렉토리 (cd - 로 이전 위치 복귀)

# 예시
cd /home/ubuntu/project/src
cd ..          # /home/ubuntu/project/ 이동
cd ../..       # /home/ubuntu/ 이동
cd ~           # /home/ubuntu/ 이동 (홈)
cd -           # 직전 디렉토리로 복귀
```

---

---

# ④ 자주 가는 경로 정리

```bash
# 현재 위치
pwd

# 홈으로
cd ~

# 로그 보러 갈 때
cd /var/log

# 설정 파일 볼 때
cd /etc

# Airflow DAG 작업
cd ~/airflow/dags
```

---

---

# 자주 하는 실수

|실수|원인|해결|
|---|---|---|
|`cd home/ubuntu` 에러|앞에 `/` 없음|`cd /home/ubuntu`|
|경로 헷갈림|현재 위치 모름|`pwd` 로 확인|
|`/tmp` 파일 사라짐|재부팅 시 삭제|중요 파일은 `/home` 에 저장|
|`/etc` 직접 수정 실패|root 권한 필요|`sudo` 앞에 붙이기|