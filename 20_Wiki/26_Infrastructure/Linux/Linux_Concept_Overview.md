---
aliases:
  - 리눅스 개념
  - Linux 개요
  - 커널
  - 쉘
tags:
  - Linux
related:
  - "[[00_Linux_HomePage]]"
  - "[[Linux_Directory_Commands]]"
  - "[[Linux_Permission_Model]]"
---

# Linux_Concept_Overview — Linux 란 무엇인가

## 한 줄 요약

```
Linux = 커널(OS 핵심) + 쉘(명령어 해석기) + 유틸리티 도구
서버의 표준 OS — 전 세계 서버의 90% 이상이 Linux
```

---

---

# ① Linux 구조

```
사용자
  ↓ 명령어 입력
쉘 (Shell)         ← 명령어 해석기 (bash, zsh)
  ↓
커널 (Kernel)       ← OS 핵심 / 하드웨어 직접 제어
  ↓
하드웨어            ← CPU / 메모리 / 디스크 / 네트워크
```

## 커널 (Kernel)

```
Linux 의 핵심 엔진
하드웨어 직접 제어
프로세스 / 메모리 / 파일시스템 / 네트워크 관리
사용자 프로그램 ↔ 하드웨어 중재

커널 버전 확인:
  uname -r   # 5.4.0-162-generic
```

## 쉘 (Shell)

```
사용자가 입력한 명령어를 커널에 전달하는 번역기

주요 쉘:
  bash   Bourne Again SHell (기본값, 가장 많이 씀)
  zsh    bash 확장 (macOS 기본값)
  sh     POSIX 표준 쉘

현재 쉘 확인:
  echo $SHELL    # /bin/bash
```

---

---

# ② 배포판 (Distribution)

```
Linux 커널 + 패키지 + 도구 묶음 = 배포판

주요 배포판:
  Ubuntu    Debian 계열 / 가장 많이 씀 / apt 패키지
  Debian    안정성 최우선
  RHEL      Red Hat Enterprise Linux / 기업용
  CentOS    RHEL 무료 버전 (현재 지원 종료)
  Amazon Linux  AWS EC2 기본 OS

데이터 엔지니어 실무:
  서버 = 대부분 Ubuntu 20/22 LTS 또는 Amazon Linux
```

---

---

# ③ CLI vs GUI ⭐️

```
GUI (Graphic User Interface):
  마우스로 클릭 / 눈에 보이는 창
  Windows / macOS 데스크탑

CLI (Command Line Interface):
  키보드로 명령어 입력
  Linux 서버 환경
```

## 왜 CLI 인가

```
서버에 GUI 없는 이유:
  GUI = 메모리 / CPU 낭비
  서버는 리소스를 서비스에 집중해야 함

CLI 의 장점:
  자동화 가능  → 스크립트로 반복 작업
  원격 접속   → SSH 로 어디서든 제어
  정밀 제어   → GUI 로 못 하는 세밀한 설정
  속도        → 마우스보다 명령어가 빠름
```

---

---

# ④ 왜 서버는 Linux 인가 ⭐️

```
안정성:
  수년간 재부팅 없이 운영 가능
  Windows Server 대비 크래시 빈도 낮음

비용:
  무료 오픈소스 (라이선스 비용 없음)
  Windows Server = 고가 라이선스

보안:
  권한 모델이 엄격 (root 분리)
  취약점 패치 빠름

자동화:
  쉘 스크립트 / crontab 으로 모든 작업 자동화
  Airflow / Docker / Kafka 모두 Linux 기반

생태계:
  Docker / Kubernetes / Kafka / Spark
  → 전부 Linux 환경에서 개발 / 운영
```

---

---

# ⑤ 데이터 엔지니어가 Linux 를 쓰는 이유

```
파이프라인 서버 접속:
  SSH 로 원격 서버 접속
  로그 확인 / 프로세스 모니터링

Docker / Kubernetes:
  컨테이너 내부 = Linux 환경
  컨테이너 디버깅 시 Linux 명령어 필수

Airflow:
  DAG 실행 서버 = Linux
  BashOperator 로 Linux 명령어 직접 실행

데이터 처리:
  대용량 파일 처리 / 압축 / 로그 분석
  awk / sed / grep 으로 빠른 전처리
```

---

---

# ⑥ Windows vs Linux 비교

|구분|Windows|Linux|
|---|---|---|
|인터페이스|GUI 중심|CLI 중심|
|경로 구분자|`\`|`/`|
|드라이브|C:\ D:\|/ (루트 하나)|
|권한|관리자/일반|root/일반|
|패키지 관리|.exe 설치|apt / yum|
|서버 점유율|낮음|90%+|
|라이선스|유료|무료|