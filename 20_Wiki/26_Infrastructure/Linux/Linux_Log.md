---
aliases:
  - 로그
  - dmesg
  - journalctl
  - syslog
  - logrotate
tags:
  - Linux
related:
  - "[[00_Linux_HomePage]]"
  - "[[Linux_System_Info]]"
  - "[[Linux_Search_Filter]]"
  - "[[Linux_Archive_Compress]]"
---

# Linux_Log — 로그 시스템

## 한 줄 요약

```
dmesg       = 커널 / 하드웨어 메시지 (부팅 시 로그)
journalctl  = systemd 기반 전체 시스템 로그 (현재 표준)
syslog      = 전통적 로그 파일 (/var/log/)
logrotate   = 로그 파일 자동 관리 (압축 / 삭제)
```

---

---

# ① 로그 시스템 구조

```
커널 메시지:
  커널 링 버퍼 → dmesg 로 조회
  부팅 과정 / 드라이버 / 하드웨어 이벤트

시스템 로그:
  systemd → journald → journalctl 로 조회
  /var/log/ → 파일로 저장 (syslog / auth.log 등)

주요 로그 파일:
  /var/log/syslog       시스템 전체 로그 (Ubuntu)
  /var/log/messages     시스템 전체 로그 (RHEL)
  /var/log/auth.log     인증 로그 (로그인 / sudo)
  /var/log/kern.log     커널 로그
  /var/log/nginx/       Nginx 웹서버 로그
  /var/log/postgresql/  PostgreSQL 로그
```

---

---

# ② dmesg — 커널 링 버퍼 메시지 ⭐️

```
dmesg = Diagnostic MeSsaGe
  커널이 부팅 과정에서 남긴 메시지
  하드웨어 인식 / 드라이버 로드 / 장치 오류
  시스템 실행 중 발생한 커널 이벤트
```

## 기본 사용

```bash
dmesg               # 전체 커널 메시지 출력
sudo dmesg          # 권한 필요한 경우

# 페이지 단위로 보기
dmesg | less

# 실시간 보기 (새 메시지 추적)
dmesg -w            # -w = watch (Ctrl+C 로 종료)
dmesg --follow      # 동일
```

## dmesg 검색 패턴 ⭐️

```bash
# 에러 / 실패 메시지 찾기
sudo dmesg | grep -iE 'fail|error|warn'
sudo dmesg | grep -E 'fail|error' > ~/boot_issue.txt

# 대소문자 무시로 더 넓게
sudo dmesg | grep -i "error"

# 특정 장치 관련
sudo dmesg | grep -i "usb"        # USB 장치
sudo dmesg | grep -i "disk"       # 디스크
sudo dmesg | grep -i "eth\|wlan"  # 네트워크 카드
sudo dmesg | grep -i "nvidia\|amd" # GPU

# 최근 N줄만
dmesg | tail -20
dmesg | tail -50
```

## dmesg 출력 해석

```bash
dmesg | head -5
# [    0.000000] Initializing cgroup subsys cpuset
# [    0.000000] Linux version 5.4.0-162-generic ...
# [    0.123456] ACPI: BIOS _OSI(Linux) query ignored
# ↑ 부팅 후 경과 시간 (초)

# 타임스탬프 보기 좋게
dmesg -T          # 사람이 읽기 좋은 날짜/시간 형식
# [Thu Apr 20 10:30:00 2026] USB Device attached
```

## 언제 dmesg 쓰나

```
새 하드웨어 장착 후 인식 안 됨
  → sudo dmesg | tail -30  (방금 발생한 이벤트)

USB 연결 확인
  → sudo dmesg | grep -i usb

디스크 에러 의심
  → sudo dmesg | grep -iE 'error|fail|I/O error'

부팅 중 드라이버 로드 실패
  → sudo dmesg | grep -i 'fail\|not found'
```

---

---

# ③ journalctl — systemd 통합 로그 ⭐️

```
systemd 가 관리하는 모든 서비스 로그를 한 곳에서 조회
dmesg + 서비스 로그 + 시스템 이벤트 통합
현재 리눅스의 표준 로그 조회 도구
```

## 기본 사용

```bash
# 전체 로그 (매우 많음 — less 로 보기)
journalctl | less
journalctl           # 기본: 전체 로그

# 최근 N줄
journalctl -n 50     # 최근 50줄
journalctl -n 100

# 실시간 (tail -f 처럼)
journalctl -f        # -f = follow
```

## 시간 범위 필터 ⭐️

```bash
# 오늘 로그만
journalctl --since today
journalctl --since "today"

# 특정 시간 이후
journalctl --since "2026-04-20 10:00"
journalctl --since "1 hour ago"
journalctl --since "2 hours ago"

# 특정 시간 범위
journalctl --since "2026-04-20 10:00" --until "2026-04-20 11:00"

# 부팅 이후 로그만
journalctl -b        # 현재 부팅
journalctl -b -1     # 이전 부팅
journalctl -b -2     # 2번 전 부팅
```

## 서비스별 필터 ⭐️

```bash
# 특정 서비스 로그
journalctl -u nginx
journalctl -u ssh
journalctl -u postgresql
journalctl -u docker

# 실시간 + 특정 서비스
journalctl -u nginx -f

# 최근 N줄 + 특정 서비스
journalctl -u nginx -n 50
```

## 우선순위(레벨) 필터

```bash
# -p 로 우선순위 필터
journalctl -p err          # 에러만
journalctl -p warning      # 경고 이상
journalctl -p debug        # 디버그 이상 (전부)

# 우선순위 번호로
journalctl -p 3            # err (0=emerg ~ 7=debug)
```

## 커널 로그만

```bash
journalctl -k              # 커널 메시지만 (dmesg 대체)
journalctl -k --since today
```

---

---

# ④ /var/log — 로그 파일 직접 보기

```bash
# 주요 로그 파일들
ls /var/log/

# 실시간 로그 보기
tail -f /var/log/syslog
tail -f /var/log/auth.log     # 로그인 시도 실시간

# 최근 에러 확인
grep "error\|ERROR" /var/log/syslog | tail -20
grep "Failed" /var/log/auth.log    # 로그인 실패

# Nginx 로그
tail -f /var/log/nginx/access.log
tail -f /var/log/nginx/error.log

# 로그 파일 크기 확인
ls -lh /var/log/
du -sh /var/log/
```

---

---

# ⑤ logrotate — 로그 자동 관리 ⭐️

```
문제: 로그 파일이 계속 커져서 디스크 꽉 참
해결: logrotate = 로그 파일 자동 순환 (rotate)

기능:
  일정 크기/기간 되면 → 파일 압축
  오래된 파일 → 자동 삭제
  새 파일로 교체
```

## 설정 파일 위치

```bash
# 기본 설정
cat /etc/logrotate.conf

# 서비스별 설정 (개별 파일)
ls /etc/logrotate.d/
cat /etc/logrotate.d/nginx
cat /etc/logrotate.d/postgresql
```

## logrotate 설정 예시

```
# /etc/logrotate.d/myapp
/var/log/myapp/*.log {
    daily              # 매일
    rotate 7           # 7개 보관 (7일치)
    compress           # gzip 압축
    delaycompress      # 가장 최근 것은 압축 안 함
    missingok          # 파일 없어도 에러 없음
    notifempty         # 빈 파일은 rotate 안 함
    postrotate
        systemctl reload myapp   # rotate 후 재시작
    endscript
}
```

## 수동 실행

```bash
# 테스트 (실제 적용 안 함)
sudo logrotate -d /etc/logrotate.conf

# 강제 실행
sudo logrotate -f /etc/logrotate.conf

# 특정 서비스만
sudo logrotate -f /etc/logrotate.d/nginx
```

---

---

# ⑥ 실전 트러블슈팅 패턴

## 서버 과부하 로그 분석

```bash
# 1. 최근 시스템 에러 확인
journalctl -p err --since "1 hour ago"

# 2. 특정 서비스 로그 확인
journalctl -u nginx --since today | grep -i error

# 3. 커널 메시지 확인
sudo dmesg | grep -iE 'error|fail' | tail -20
```

## 디스크 관련 에러 확인

```bash
sudo dmesg | grep -iE 'I/O error|disk|bad sector'
journalctl -k | grep -i 'error\|fail'
```

## SSH 로그인 실패 추적

```bash
grep "Failed password" /var/log/auth.log | tail -20
grep "Failed password" /var/log/auth.log | wc -l
journalctl -u ssh | grep "Failed" | tail -10
```

---

---

# 명령어 한눈에

|명령어|역할|
|---|---|
|`dmesg`|커널 링 버퍼 메시지|
|`dmesg -T`|타임스탬프 포함|
|`dmesg -w`|실시간 커널 메시지|
|`sudo dmesg \| grep -iE 'fail\|error'`|에러 필터|
|`journalctl -f`|실시간 로그|
|`journalctl -u 서비스`|특정 서비스 로그|
|`journalctl -p err`|에러 레벨만|
|`journalctl --since today`|오늘 로그|
|`journalctl -b`|현재 부팅 이후|
|`tail -f /var/log/syslog`|syslog 실시간|
|`logrotate -f`|로그 순환 강제 실행|

---

---

# 자주 하는 실수

|실수|원인|해결|
|---|---|---|
|`dmesg` 권한 없음|일반 사용자 제한|`sudo dmesg`|
|로그 너무 많음|전체 출력|`-n 50` 또는 `--since today`|
|journalctl 서비스 로그 없음|서비스명 오타|`systemctl list-units` 로 이름 확인|
|/var/log 디스크 꽉 참|logrotate 미설정|logrotate 설정 추가|