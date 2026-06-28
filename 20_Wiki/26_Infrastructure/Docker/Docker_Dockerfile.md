---
aliases:
  - Dockerfile
  - FROM
  - RUN
  - COPY
  - ENTRYPOINT
  - ENV
related:
  - "[[00_Docker_HomePage]]"
  - "[[Docker_Image]]"
  - "[[Docker_Commands]]"
  - "[[Docker_Build]]"
  - "[[Docker_Container]]"
  - "[[Docker_Volume]]"
tags:
  - Docker
---
# Docker_Dockerfile — Dockerfile

# 한 줄 요약

```txt
Dockerfile = 이미지를 만드는 설명서
각 명령어 한 줄 = 레이어 하나
docker build -t 이름 .  → 이미지 생성
```

---

---

# 기본 구조

```dockerfile
# 파일명: Dockerfile (D 대문자 / 확장자 없음)

FROM python:3                        # 베이스 이미지
WORKDIR /app                         # 작업 디렉토리
COPY . /app                          # 파일 복사
RUN pip install -r requirements.txt  # 명령어 실행
ENV APP_ENV=production               # 환경변수
EXPOSE 5000                          # 포트 문서화
CMD ["python", "app.py"]             # 컨테이너 시작 명령
```

---

---

# Dockerfile 작성 후 → 빌드 → 실행 흐름 ⭐️

```txt
1. Dockerfile 작성
2. docker build 로 이미지 생성
3. docker run 으로 컨테이너 실행
```

```bash
# 1. Dockerfile 있는 디렉토리에서 빌드
docker build -t my-app .
#             ↑        ↑
#           이미지 이름  빌드 컨텍스트 (현재 디렉토리)

# 2. 만들어진 이미지 확인
docker images

# 3. 이미지로 컨테이너 실행
docker run -d -p 5000:5000 my-app
```

## docker build 옵션 ⭐️

```bash
# 기본
docker build -t my-app .

# 태그(버전) 지정
docker build -t my-app:1.0 .
docker build -t my-app:latest .

# Dockerfile 이름이 다를 때 (-f)
docker build -f Dockerfile.dev -t my-app:dev .
#             ↑ 파일명 직접 지정

# 빌드 인자 전달 (ARG 와 함께)
docker build --build-arg VERSION=1.0 -t my-app .

# 캐시 무시하고 새로 빌드
docker build --no-cache -t my-app .
```

```txt
-t  이미지 이름[:태그] 지정 (tag)
-f  Dockerfile 경로 지정 (기본값: ./Dockerfile)
.   빌드 컨텍스트 — COPY 등 명령어가 접근할 수 있는 범위
    . = 현재 디렉토리 전체를 Docker 데몬에 전송

--no-cache 쓰는 경우:
  RUN apt-get update 같은 명령이 캐시 때문에 오래된 결과를 쓸 때
  Dockerfile 은 안 바뀌었는데 외부 의존성이 바뀌었을 때
```

## 빌드 후 자주 하는 것들

```bash
# 이미지 목록 확인
docker images

# 이미지로 컨테이너 실행 (포그라운드)
docker run my-app

# 이미지로 컨테이너 실행 (백그라운드 + 포트)
docker run -d -p 5000:5000 my-app

# 이미지 삭제
docker rmi my-app

# 빌드 실패 시 중간에 생긴 dangling 이미지 정리
docker image prune
```

```txt
빌드 실패할 때 확인 순서:
  1. 에러 메시지 읽기 — Step N/N 에서 어떤 명령어가 실패했는지
  2. 그 줄 위아래 Dockerfile 문법 확인
  3. --no-cache 붙여서 다시 빌드
  4. 중간 레이어에서 직접 실행해보기
     docker run -it 이미지 bash  → 수동으로 명령어 실행
```

---

---

# FROM — 베이스 이미지 ⭐️

```dockerfile
FROM python:3
FROM python:3.11-slim   # slim = 최소 패키지 / 이미지 크기 감소
FROM nginx:alpine       # alpine 기반 / 매우 가벼움
FROM ubuntu:22.04
```

```txt
모든 Dockerfile 의 첫 줄
베이스 이미지 위에 레이어를 쌓아올림
가벼운 이미지(alpine, slim) 선택 → 최종 이미지 크기 감소
```

---

---

# WORKDIR — 작업 디렉토리 ⭐️

```dockerfile
WORKDIR /app
# 이후 모든 명령어의 기준 경로 = /app
# 폴더 없으면 자동 생성
# 컨테이너 접속 시 시작 위치
```

```dockerfile
# WORKDIR 없이:
COPY requirements.txt /app/requirements.txt
RUN pip install -r /app/requirements.txt

# WORKDIR 사용:
WORKDIR /app
COPY requirements.txt .      # . = /app/requirements.txt
RUN pip install -r requirements.txt
```

---

---

# COPY vs ADD ⭐️

```dockerfile
# COPY — 단순 파일 복사 (권장)
COPY index.html /usr/share/nginx/html/
COPY . /app/
COPY src/ /app/src/

# ADD — URL 다운로드 / tar 압축 자동 해제 가능
ADD . /app
```

```txt
⚠️ ADD 문법 주의:
  ADD./app    ← 에러! (공백 없음)
  ADD . /app  ← 올바름 (출처, 목적지 공백으로 구분)

  에러 메시지:
  "ADD requires at least two arguments, but only one was provided."
  → ADD 와 . 와 /app 사이 공백 반드시 필요

COPY vs ADD:
  COPY   단순 복사 / 예측 가능 / 권장
  ADD    URL 다운로드 / tar 자동 해제 특수 기능
  → 특수 기능 필요 없으면 COPY 사용
```

## requirements.txt 먼저 복사 패턴 ⭐️

```dockerfile
WORKDIR /app

COPY requirements.txt .              # 1. 의존성 파일 먼저
RUN pip install -r requirements.txt  # 2. 설치
COPY . .                             # 3. 소스코드 나중에
```

```txt
왜 순서가 중요한가 — 레이어 캐시:
  ❌ 비효율:
    COPY . .                  소스 변경 시
    RUN pip install ...       pip install 도 다시 실행

  ✅ 효율:
    COPY requirements.txt .   requirements 안 바뀌면
    RUN pip install ...       캐시 사용 (빠름)
    COPY . .                  소스코드만 다시 복사
```

---

---

# RUN — 명령어 실행 ⭐️

```dockerfile
RUN apt-get update && apt-get install -y curl

# 여러 패키지 백슬래시로 줄 나누기
RUN apt-get update && \
    apt-get install -y \
      curl \
      wget && \
    rm -rf /var/lib/apt/lists/*   # 캐시 삭제 → 이미지 크기 감소
```

```txt
RUN 실행 시점:
  docker build 할 때 실행 → 이미지 레이어로 저장
  컨테이너 실행 시가 아님

RUN 한 줄 = 레이어 하나
&& 로 묶으면 레이어 하나에 처리 → 이미지 작아짐

-y 플래그:
  apt-get install -y curl
  → 설치 확인 질문에 자동 yes → 빌드 자동화 시 필수

RUN vs CMD:
  RUN   docker build 시 실행 (이미지에 저장)
  CMD   docker run 시 실행 (컨테이너 기본 명령)
```

---

---

# ENV — 환경변수 설정 ⭐️

```dockerfile
ENV APP_ENV=production
ENV PORT=5000

# 여러 개 한번에
ENV NODE_ENV=production \
    PORT=3000
```

```txt
컨테이너 런타임에도 환경변수로 남아 있음
docker run -e PORT=8080 → 런타임에 덮어쓰기 가능

ARG vs ENV:
  ARG  빌드 시점만 / 컨테이너 런타임에 없음
  ENV  빌드 + 런타임 모두 사용
```

---

---

# CMD vs ENTRYPOINT ⭐️

```dockerfile
# CMD — 기본 실행 명령 (docker run 에서 덮어쓰기 가능)
CMD ["python", "app.py"]
CMD ["node", "index.js"]
CMD ["nginx", "-g", "daemon off;"]

# ENTRYPOINT — 항상 실행 (덮어쓰기 어려움)
ENTRYPOINT ["/start.sh"]

# 조합 패턴
ENTRYPOINT ["nginx"]
CMD ["-g", "daemon off;"]   # docker run 에서 이 부분만 교체 가능
```

```txt
CMD vs ENTRYPOINT:
  CMD         docker run 뒤 명령어로 덮어씌워짐
  ENTRYPOINT  항상 실행 / docker run 인자는 CMD 를 대체

실전:
  고정 실행 파일 → ENTRYPOINT
  기본값만 제공 → CMD
  유연한 스크립트 → ENTRYPOINT + CMD 조합

RUN vs CMD:
  RUN  docker build 시 (이미지 레이어)
  CMD  docker run 시 (컨테이너 시작)
```

## ENTRYPOINT + 쉘 스크립트 패턴

```dockerfile
COPY start.sh /start.sh
RUN chmod +x /start.sh    # ⚠️ 실행 권한 필수 (없으면 Permission denied)
ENTRYPOINT ["/start.sh"]
```

---

---

# VOLUME — 볼륨 마운트 선언

```dockerfile
VOLUME /data
```

```bash
docker run -it -v $(pwd)/data:/data my-image bash
```

```txt
VOLUME 선언만으로는 호스트 경로 지정 안 됨
docker run -v 로 실제 마운트 연결 필요
→ [[Docker_Volume]] 참조
```

---

---

# 기타 명령어

```dockerfile
EXPOSE 80           # 포트 문서화 (실제 오픈 아님)
EXPOSE 8080 50000   # 여러 포트

ARG BUILD_VERSION   # 빌드 시점 인자 (docker build --build-arg)
USER appuser        # 비루트 사용자 (보안)

HEALTHCHECK --interval=30s --timeout=3s \
    CMD curl -f http://localhost/ || exit 1
```

```txt
⚠️ EXPOSE 는 실제로 포트를 열지 않음
   문서화 / 가독성 용도
   실제 포트 오픈 = docker run -p 8080:8080
```

---

---

# 레이어 생성 여부

```txt
레이어 생성 O (이미지 크기에 영향):
  FROM / COPY / ADD / RUN

레이어 생성 X (메타데이터만):
  WORKDIR / ENV / LABEL / EXPOSE / VOLUME / USER
```

---

---

# 명령어 한눈에

|명령어|역할|주의|
|---|---|---|
|`FROM 이미지`|베이스 이미지|첫 줄|
|`WORKDIR /경로`|작업 디렉토리|자동 생성|
|`COPY 출처 목적지`|파일 복사|권장|
|`ADD 출처 목적지`|파일 복사+특수기능|공백 주의 ⚠️|
|`RUN 명령어`|레이어 생성|build 시 실행|
|`ENV KEY=VALUE`|환경변수|런타임 유지|
|`CMD ["명령"]`|기본 실행|덮어쓰기 가능|
|`ENTRYPOINT ["명령"]`|항상 실행|고정|
|`EXPOSE 포트`|포트 문서화|실제 오픈 아님|
|`VOLUME /경로`|볼륨 선언|-v 로 실제 연결|

```txt
빌드 → 실행 한 줄 요약:
  docker build -t my-app .    이미지 생성
  docker images               이미지 확인
  docker run -d -p 포트 my-app  컨테이너 실행
```