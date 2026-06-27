---
aliases:
  - 시스템 정보
  - whoami
  - uname
  - uptime
tags:
  - Linux
related:
  - "[[00_Linux_HomePage]]"
  - "[[Linux_Process_Monitor]]"
  - "[[Linux_Directory_Commands]]"
  - "[[Linux_User_Group]]"
---

# Linux_System_Info — 시스템 기본 정보 확인

## 한 줄 요약

```
서버에 처음 접속했을 때 가장 먼저 확인해야 하는 명령어들
"여기가 어디고, 누구이고, 서버 상태는 어떤가"
```

---

---

# ① whoami — 현재 사용자 확인

```bash
whoami
# labex
```

```
로그인한 계정 이름 출력
sudo -i 로 root 가 됐는지 확인할 때도 사용

스크립트에서 활용:
  CURRENT_USER=$(whoami)
  echo "현재 사용자: $CURRENT_USER"
```

---

---

# ② uname — 커널 / 시스템 정보 ⭐️

```bash
uname           # Linux  ← OS 이름만
uname -a        # 전체 상세 정보
# Linux 69f156a6 5.4.0-162-generic #179-Ubuntu SMP Mon Aug 14 08:51:31 UTC 2023 x86_64 x86_64 x86_64 GNU/Linux

uname -r        # 커널 버전만
# 5.4.0-162-generic

uname -m        # 아키텍처만
# x86_64

uname -n        # 호스트명만
# hostname
```

## uname -a 출력 해석

```
Linux   69f156a6633c   5.4.0-162-generic   #179-Ubuntu SMP   ...   x86_64
↑ OS    ↑ 호스트명      ↑ 커널 버전          ↑ 빌드 정보              ↑ 아키텍처
```

---

---

# ③ uptime — 서버 가동 시간 & 부하 ⭐️

```bash
uptime
# 08:56:59 up 236 days, 16:23,  0 users,  load average: 0.12, 0.21, 0.18
# ↑ 현재시각  ↑ 가동 시간         ↑ 접속자     ↑ 1분  ↑ 5분  ↑ 15분 평균 부하
```

## Load Average 해석 ⭐️

```
Load Average = CPU + I/O 대기 중인 프로세스 평균 수

CPU 코어 수 기준으로 해석:
  CPU 1코어 기준:
    1.0  = CPU 100% 사용 (포화 상태)
    0.5  = CPU 50% 사용 (여유)
    2.0  = CPU 200% → 과부하

  CPU 4코어 기준:
    4.0  = 모든 코어 100% (포화)
    2.0  = 50% 사용 (여유)

현재 코어 수 확인:
  nproc          # 코어 수
  grep -c ^processor /proc/cpuinfo
```

```
트러블슈팅 패턴:
  1분 부하 > 15분 부하  → 갑작스러운 과부하 (지금 뭔가 몰리고 있음)
  1분 부하 < 15분 부하  → 부하가 줄어드는 중

  1분 값이 코어 수보다 훨씬 크면 → top 으로 원인 프로세스 찾기
```

---

---

# ④ id — 사용자 & 그룹 정보 ⭐️

```bash
id                  # 현재 사용자 정보
id labex            # 특정 사용자 정보
id jack             # 다른 사용자 정보 확인
id root             # root 정보
```

## 출력 해석

```
uid=5000(labex)   → 사용자 고유 번호 (UID)
                     일반 사용자: 1000번대 이상
                     root: 0

gid=5000(labex)   → 기본 그룹(Primary Group) 번호

groups=5000(labex),27(sudo),121(ssl-cert)
                  → 소속된 모든 그룹 목록
                  27(sudo) 포함 = 관리자 권한 있음 ✅

새로 생성한 사용자:
  id jack
  uid=1001(jack) gid=1001(jack) groups=1001(jack)
  → UPG: 사용자 이름과 같은 기본 그룹 자동 생성

sudo 그룹 추가 후:
  id jack
  uid=1001(jack) gid=1001(jack) groups=1001(jack),27(sudo),1002(developers)
```
---

---

# ⑤ hostname — 서버 이름

```bash
hostname         # 호스트명 출력
hostname -I      # 모든 IP 주소 출력
# 172.16.50.3 172.17.0.1
```

---

---

# ⑥ top — 실시간 프로세스 모니터링

```bash
top
# q 로 종료
```

```
단축키:
  P   CPU 사용량 순 정렬
  M   메모리 사용량 순 정렬
  q   종료
```

→ 자세한 내용은 [[Linux_Process_Monitor]] 참고

---

---

# ⑦ 시스템 보고서 자동 생성 ⭐️

```bash
# 빈 보고서 파일 생성
touch system_report.txt

# 각 명령어 결과를 파일에 추가 (>> = 이어쓰기)
whoami >> system_report.txt
uname -a >> system_report.txt
uptime >> system_report.txt
id >> system_report.txt
date >> system_report.txt

# 결과 확인
cat system_report.txt
```

```
> vs >> 차이:
  >  = 덮어쓰기 (기존 내용 삭제)
  >> = 이어쓰기 (기존 내용 뒤에 추가)

보고서 자동화 스크립트 패턴:
  여러 명령어 결과를 하나의 파일에 모아서
  → 서버 점검 보고서 / 상태 로그 생성
```

---

---

# 명령어 한눈에

|명령어|역할|
|---|---|
|`whoami`|현재 사용자명|
|`id`|UID / GID / 그룹 목록|
|`uname -a`|커널 버전 / 아키텍처 전체|
|`uname -r`|커널 버전만|
|`uptime`|가동 시간 / Load Average|
|`hostname`|서버 이름|
|`hostname -I`|IP 주소 목록|
|`nproc`|CPU 코어 수|