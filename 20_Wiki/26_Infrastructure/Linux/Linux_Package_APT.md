---
aliases:
  - apt
  - apt-get
  - dpkg
  - 패키지 관리자
  - remove
  - autoremove
  - -y
tags:
  - Linux
related:
  - "[[00_Linux_HomePage]]"
---

# Linux_Package_APT — APT / dpkg 패키지 관리

## 한 줄 요약

```
dpkg  = .deb 파일 직접 다루기 (저수준)
apt   = dpkg 위에서 의존성 자동 처리 (고수준)

Ubuntu/Debian 계열 → apt / dpkg
Red Hat/CentOS 계열 → rpm / dnf
```

---

---

# ① 패키지 관리자 개념

```
패키지 = 소프트웨어를 설치하기 위한 압축 파일
  .deb  → Debian/Ubuntu 계열
  .rpm  → Red Hat/CentOS 계열

패키지 매니저가 없으면:
  소스 코드 다운로드 → 컴파일 → 설치 → 수동 관리
  의존성도 직접 해결 (매우 번거로움)

패키지 매니저가 있으면:
  명령어 한 줄로 설치 + 의존성 자동 해결 + 업데이트 + 제거
```

## 저장소 (Repository) 개념 ⭐️

```
저장소 = 패키지들이 모여있는 서버

apt install python3 실행 시:
  1. /etc/apt/sources.list 확인 → 저장소 주소 파악
  2. 저장소 서버에 접속
  3. python3.deb 다운로드
  4. 의존성 패키지도 자동 다운로드
  5. 설치

저장소 종류:
  공식 저장소  → Ubuntu/Debian 이 관리
  PPA         → 개인/단체가 만든 외부 저장소 (최신 버전)
  로컬 저장소  → 폐쇄망 환경에서 내부 서버 구성
```

---

---

# ② apt vs apt-get 차이

```
apt-get:
  구버전 / 스크립트에서 사용 권장
  출력이 단순

apt:
  apt-get + apt-cache 기능 통합
  진행률 표시 / 컬러 출력 / 더 직관적
  터미널에서 직접 사용 권장

결론:
  터미널 대화형 → apt
  스크립트 자동화 → apt-get (안정적)
```

---

---

# ③ apt 기본 명령어 ⭐️

## 저장소 업데이트

```bash
sudo apt update
# 저장소 목록 갱신 (실제 설치 아님!)
# 패키지 설치/업그레이드 전에 항상 먼저 실행
```

```
apt update 가 왜 필요한가:
  로컬 캐시에 저장된 패키지 목록이 오래됐을 수 있음
  update = "저장소에서 최신 목록 가져오기"
  설치는 안 함 / 목록만 갱신
```

## 패키지 설치

```bash
sudo apt install 패키지명
sudo apt install -y 패키지명      # 확인 없이 자동 설치
sudo apt install 패키지1 패키지2  # 여러 개 동시
sudo apt install ./local.deb     # 로컬 .deb 파일 설치
```

## -y 옵션이 왜 필요한가 ⭐️

```
apt install 실행 시 묻는 질문:
  "설치하시겠습니까? [Y/n]"

-y 없으면:
  스크립트 자동화 중 사용자 입력 기다리며 멈춤 → 대참사

-y 있으면:
  자동으로 Y 응답 → 멈춤 없이 설치 완료

사용 상황:
  서버 프로비저닝 스크립트 → 필수
  여러 패키지 일괄 설치 → 필수
  수동 설치 → 선택 (확인하고 싶으면 생략)
```

```bash
# 스크립트 자동화 패턴
sudo apt update
sudo apt install -y git curl wget python3 python3-pip
```

## 패키지 제거

```bash
sudo apt remove 패키지명          # 패키지만 제거 (설정 파일 유지)
sudo apt purge 패키지명           # 패키지 + 설정 파일 완전 제거
sudo apt autoremove               # 불필요해진 의존성 패키지 자동 제거
sudo apt autoremove -y            # 확인 없이 자동 제거
```

```
remove vs purge:
  remove  → 실행 파일 삭제 / 설정 파일 남김 (재설치 시 설정 복구 가능)
  purge   → 설정 파일까지 완전 삭제 (깨끗하게 지울 때)

autoremove ⭐️:
  설치 시 딸려온 의존성 패키지가 메인 패키지 삭제 후 고아로 남음
  → autoremove 가 자동으로 찾아서 일괄 제거
  → 디스크 100% 꽉 찼을 때 용량 확보 첫 번째 방법
  정기적으로 실행 권장

  언제 쓰나:
    1. apt remove / purge 직후
       삭제한 패키지의 의존성 잔여물 정리
    2. 시스템 정리 필요할 때
       쌓인 불필요한 라이브러리 제거 → 디스크 공간 확보

  권장 패턴:
    sudo apt remove 패키지명 -y
    sudo apt autoremove -y     ← 바로 이어서 실행
```

## 업데이트

```bash
sudo apt update               # 저장소 목록 갱신
sudo apt upgrade              # 설치된 패키지 전체 업그레이드
sudo apt upgrade 패키지명     # 특정 패키지만 업그레이드
sudo apt full-upgrade         # 의존성 변경도 허용하며 업그레이드
```

```
update vs upgrade:
  update   = 목록 갱신 (다운로드/설치 없음)
  upgrade  = 실제 업그레이드 (설치/교체)
  → 반드시 update 먼저 → upgrade 순서
```

## 검색 & 정보 확인

```bash
apt search 키워드             # 패키지 검색
apt show 패키지명             # 패키지 상세 정보 (버전/의존성)
apt list --installed          # 설치된 패키지 목록
apt list --installed | grep python
apt list --upgradable         # 업그레이드 가능한 목록
```

---

---

# ④ dpkg — .deb 파일 직접 관리

```
dpkg = Debian Package manager
.deb 파일을 직접 다루는 저수준 도구
의존성 자동 처리 없음 (apt 가 처리)

언제 dpkg 씀:
  인터넷 없는 환경에서 .deb 파일 직접 설치
  이미 설치된 패키지 상태 조회
  설치 목록 일괄 확인
```

## dpkg 주요 명령어

```bash
# .deb 파일 직접 설치
sudo dpkg -i 패키지.deb

# 설치 후 의존성 에러 발생 시 해결
sudo apt-get install -f
# -f = fix (의존성 자동 수정)

# 패키지 제거
sudo dpkg -r 패키지명       # 설정 파일 유지
sudo dpkg -P 패키지명       # 완전 제거 (purge)

# 설치 여부 확인
dpkg -l 패키지명
# ii = 정상 설치
# un = 미설치

# 설치된 전체 패키지 목록
dpkg -l
dpkg -l | grep python

# 패키지가 설치한 파일 목록
dpkg -L 패키지명

# 파일 → 패키지 역추적
dpkg -S /usr/bin/python3
# python3-minimal: /usr/bin/python3
```

```
dpkg 옵션 구분:
  -s (status)   패키지 상세 정보 + 현재 상태 ⭐️
  -l (list)     설치 목록 (ii/un 상태코드 한 줄)
  -L (Listfiles) 패키지가 설치한 파일 목록
  -S (Search)   파일 → 어떤 패키지인지 역추적
  -I (Info)     .deb 파일 설치 전 정보 확인
  -i (install)  .deb 파일 직접 설치
  -r (remove)   제거 (설정 파일 유지)
  -P (Purge)    완전 제거

-s vs -l:
  dpkg -s nginx             nginx 상세 (버전/의존성/설명 전부)
  dpkg -l nginx             상태코드만 (ii/un 한 줄)
  dpkg -s nginx | grep Status  설치 여부만 빠르게 확인 ⭐️
```

```
dpkg -l 상태 코드:
  ii = installed (정상 설치)
  rc = removed (제거됐지만 설정 남음)
  un = unknown / not installed
```

---

---

# ⑤ 저장소 관리 — sources.list ⭐️

```
/etc/apt/sources.list
→ apt 가 패키지를 가져오는 저장소 주소 목록
```

## sources.list 형식

```
deb https://archive.ubuntu.com/ubuntu jammy main restricted
↑    ↑                                ↑      ↑
타입  저장소 주소                       배포판  구성요소

타입:
  deb      = 바이너리 패키지
  deb-src  = 소스 패키지

구성요소:
  main        = 공식 / 오픈소스
  restricted  = 공식 / 비오픈소스 (드라이버)
  universe    = 커뮤니티 / 오픈소스
  multiverse  = 커뮤니티 / 비오픈소스
```

## PPA 추가 — 최신 버전 설치

```bash
# PPA 추가 (Personal Package Archive)
sudo add-apt-repository ppa:deadsnakes/ppa
sudo apt update
sudo apt install python3.11

# PPA 제거
sudo add-apt-repository --remove ppa:deadsnakes/ppa
```

```
PPA 언제 씀:
  공식 저장소 버전이 너무 오래됐을 때
  특정 개발자/단체의 최신 버전 필요할 때
  주의: PPA 는 공식 검증 없음 → 신뢰할 수 있는 출처만
```

## 저장소 갱신 및 확인

```bash
# 저장소 목록 갱신
sudo apt update

# 현재 등록된 저장소 목록
cat /etc/apt/sources.list
ls /etc/apt/sources.list.d/     # 추가 저장소 파일들
```

---

---

# ⑥ 캐시 관리

```bash
# 다운로드된 .deb 캐시 파일 위치
ls /var/cache/apt/archives/

# 캐시 삭제 (디스크 공간 확보)
sudo apt clean              # 전체 캐시 삭제
sudo apt autoclean          # 오래된 캐시만 삭제
```

---

---

# ⑦ apt vs dpkg vs rpm/dnf 비교

|기능|apt (Debian)|dpkg (Debian)|dnf (Red Hat)|rpm (Red Hat)|
|---|---|---|---|---|
|수준|고수준|저수준|고수준|저수준|
|의존성|자동|수동|자동|수동|
|설치|`apt install`|`dpkg -i`|`dnf install`|`rpm -ivh`|
|제거|`apt remove`|`dpkg -r`|`dnf remove`|`rpm -e`|
|목록|`apt list --installed`|`dpkg -l`|`dnf list installed`|`rpm -qa`|
|파일→패키지|`dpkg -S`|`dpkg -S`|`rpm -qf`|`rpm -qf`|
|저장소|sources.list|-|.repo 파일|-|

---

---

# 자주 하는 실수

|실수|원인|해결|
|---|---|---|
|`apt install` 전 `apt update` 안 함|오래된 목록|항상 `update` 먼저|
|의존성 에러|dpkg 직접 설치|`apt-get install -f` 로 수정|
|설정 파일까지 지우고 싶음|`remove` 사용|`apt purge` 사용|
|불필요한 패키지 쌓임|autoremove 안 함|`sudo apt autoremove`|
|PPA 추가 후 에러|신뢰 안 되는 PPA|PPA 출처 확인 후 추가|
|캐시 용량 큼|자동 정리 없음|`sudo apt clean`|