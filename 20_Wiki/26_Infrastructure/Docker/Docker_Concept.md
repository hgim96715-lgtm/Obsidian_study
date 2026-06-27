---
aliases:
  - Docker 개념
  - 컨테이너
  - Docker Hub
tags:
  - Docker
related:
  - "[[00_Docker_HomePage]]"
  - "[[Docker_Commands]]"
  - "[[Docker_Image]]"
---

# Docker_Concept — Docker 개념

## 한 줄 요약

```txt
애플리케이션 + 실행 환경을 하나로 묶어서 어디서든 동일하게 실행
"내 컴퓨터에서는 되는데?" 문제 해결
```

---

---

# ① 핵심 개념 4가지 ⭐️

## Container (컨테이너)

```txt
소프트웨어를 실행하는 데 필요한 모든 것을 포함하는
가볍고 독립적이며 실행 가능한 패키지

포함되는 것:
  코드 / 런타임 / 라이브러리 / 환경변수 / 설정 파일
  → 어떤 환경에서도 동일하게 동작

VM(가상머신) 과 차이:
  VM   = 운영체제 전체를 가상화 → 무겁고 느림
  Container = 운영체제 커널 공유 → 가볍고 빠름
```

## Image (이미지)

```txt
컨테이너를 만들기 위한 설계도 / 템플릿
컨테이너를 생성하는 데 필요한 모든 지침 포함

비유:
  이미지 = 붕어빵 틀
  컨테이너 = 붕어빵 (틀로 찍어낸 것)
  하나의 이미지 → 여러 컨테이너 생성 가능

특징:
  읽기 전용 (수정 불가)
  레이어로 구성 → 변경 부분만 저장 (효율적)
  TAG 로 버전 관리 (latest / 1.0 / 2.1 등)
```

## Docker Hub

```txt
Docker 이미지를 위한 GitHub 같은 곳
클라우드 기반 이미지 레지스트리

역할:
  이미지 저장 / 공유 / 배포
  docker pull → Hub 에서 이미지 다운로드
  docker push → Hub 에 이미지 업로드

https://hub.docker.com

종류:
  Official Image   Docker 에서 직접 관리 (유지보수 잘 됨)
  공개 이미지      누구나 올릴 수 있음
  개인/팀 이미지   비공개 저장소
```

## Docker Engine

```txt
사용자 컴퓨터에서 컨테이너를 실행하고 관리하는 핵심 기술
백그라운드에서 실행되는 데몬 (서비스)

구성:
  Docker Daemon  백그라운드에서 실행 (컨테이너 관리)
  Docker Client  사용자가 입력하는 명령어 (docker run 등)
  → Client 가 Daemon 에 명령 전달
```

---

---

# ② 전체 흐름 ⭐️

```txt
Docker Hub (이미지 저장소)
    ↕ pull / push
Docker Engine (로컬 실행 환경)
  ↓ create
Containers (실행 중인 앱)
  ↑ 기반
Images (설계도)
```

## docker run 실행 흐름

```bash
docker run hello-world
```

```txt
1. Docker 클라이언트 → Docker 데몬에 명령 전달
2. 데몬이 로컬에 이미지 있는지 확인
3. 없으면 → Docker Hub 에서 자동 pull
4. 이미지로 새 컨테이너 생성
5. 컨테이너 실행 → 출력 결과 터미널로 전달
```

---

---

# ③ 기본 명령어 맛보기

```bash
# 컨테이너 실행 (이미지 없으면 자동 pull)
docker run hello-world

# 로컬에 있는 이미지 목록
docker images
# REPOSITORY    TAG       IMAGE ID       CREATED        SIZE
# hello-world   latest    feb5d9fea6a5   2 weeks ago    13.3kB
#  ↑이름         ↑버전      ↑고유 ID       ↑생성 날짜     ↑크기

# 이미지만 미리 다운로드
docker pull hello-world

# 실행 중인 컨테이너 목록
docker ps

# 전체 컨테이너 목록 (종료된 것 포함)
docker ps -a
```

## docker images 출력 읽기

```txt
REPOSITORY    이미지 이름
TAG           버전 (지정 안 하면 latest 자동 사용)
IMAGE ID      이미지 고유 ID (앞 12자리만 표시)
CREATED       이미지 생성 시점
SIZE          디스크 크기

두 번째 실행 시:
  이미 로컬에 이미지 있음 → Hub 에서 다운로드 없이 즉시 실행
  → 훨씬 빠름
```

---

---

# ④ Docker Hub 이미지 찾기

```txt
https://hub.docker.com 에서 검색

확인할 것:
  Official Image 배지  → Docker 공식 관리 이미지 (신뢰)
  Tags             → 버전 목록 (latest / 특정 버전)
  Pull Command     → 다운로드 명령어
  Dockerfile       → 이미지 빌드 방법 (참고용)
```

```bash
# Docker Hub 에서 이미지 검색 (터미널)
docker search postgres

# 특정 버전 pull
docker pull postgres:15
docker pull postgres:latest

# TAG 지정 없으면 latest 자동
docker pull postgres      # = docker pull postgres:latest
```

---

---

# ⑤ Docker 가 필요한 이유

```txt
문제 상황:
  개발자 A: "내 맥에서는 잘 돼요"
  서버:     "여기선 에러가 납니다"
  이유:     Python 버전 다름 / 라이브러리 버전 다름

Docker 해결:
  코드 + 실행 환경을 통째로 이미지로 만듦
  → 어디서 실행해도 동일한 환경 보장

데이터 엔지니어에게 유용한 이유:
  PostgreSQL / Kafka / Airflow / Spark
  → 로컬에 직접 설치 없이 컨테이너로 실행
  → 버전 충돌 없음
  → 쓰고 나서 컨테이너 삭제 → 깔끔
```