---
aliases:
  - docker build
  - 이미지 빌드
  - dockerignore
  - 레이어 캐시
related:
  - "[[00_Docker_HomePage]]"
  - "[[Docker_Dockerfile]]"
  - "[[Docker_Image]]"
tags:
  - Docker
---
# Docker_Build — 이미지 빌드

## 한 줄 요약

```txt
docker build -t 이름:태그 .
Dockerfile 변경 시 반드시 다시 빌드해야 적용됨
. = 현재 디렉토리를 빌드 컨텍스트로 사용
```

---

---

# ① docker build 기본 ⭐️

```bash
# 기본 빌드
docker build -t my-nginx .
#             ↑ 이름     ↑ 빌드 컨텍스트 (현재 디렉토리)

# 태그 포함
docker build -t my-nginx:v1 .
docker build -t my-nginx:latest .
docker build -t polyglot-whale .

# 빌드 후 이미지 확인
docker images
```

```txt
⚠️ 가장 많이 하는 실수:
  Dockerfile 수정 후 docker run 만 다시 실행
  → 변경 사항 적용 안 됨!

  반드시: 수정 → docker build → docker run
```

---

---

# ② 빌드 후 실행까지 흐름 ⭐️

```bash
# 1. Dockerfile 작성 또는 수정
# FROM nginx
# COPY index.html /usr/share/nginx/html/
# RUN chmod +x /entrypoint.sh   ← 실행 권한 잊지 말기

# 2. 빌드 (수정할 때마다 반드시)
docker build -t my-app .

# 3. 기존 컨테이너 있으면 삭제 후 재실행
docker stop my-container
docker rm my-container
docker run -d --name my-container -p 8080:80 my-app

# 또는 --rm 으로 자동 삭제
docker run --rm -p 8080:80 my-app
```

```txt
chmod +x 잊었을 때 에러:
  permission denied: /entrypoint.sh
  → Dockerfile 에 RUN chmod +x /entrypoint.sh 추가
  → 반드시 다시 빌드
```

---

---

# ③ 레이어 캐시 ⭐️

```txt
Docker 는 각 명령어(레이어)를 캐시에 저장
다음 빌드 시 변경 없는 레이어는 캐시 사용 → 빠름
변경된 레이어부터 아래는 모두 재빌드

Dockerfile:
  FROM ubuntu         ← 캐시 사용
  RUN apt-get update  ← 캐시 사용
  COPY . /app         ← 파일 변경됨! 여기부터 재빌드
  RUN npm install     ← 재빌드
```

## 캐시 효율적으로 쓰는 패턴 ⭐️

```dockerfile
# ❌ 비효율 — 소스 변경 시 npm install 도 다시 실행
COPY . /app
RUN npm install

# ✅ 효율 — package.json 먼저 복사 → install → 소스 복사
COPY package*.json /app/
RUN npm install          # package.json 안 바뀌면 캐시 사용
COPY . /app              # 소스 변경은 여기만 영향
```

```txt
핵심 원칙:
  변경이 적은 것(의존성) → Dockerfile 위쪽
  변경이 잦은 것(소스코드) → 아래쪽
  → 자주 바뀌는 레이어 이후만 재빌드
```

## 캐시 무시하고 강제 빌드

```bash
# 캐시 무시 (처음부터 다시 빌드)
docker build --no-cache -t my-app .

# 언제 쓰나:
#   apt-get update 가 오래된 패키지를 가져올 때
#   캐시 때문에 예상과 다른 결과 나올 때
```

---

---

# ④ .dockerignore ⭐️

```txt
빌드 컨텍스트(.) 에서 제외할 파일/폴더 지정
빌드 속도 향상 + 이미지 크기 감소 + 보안
```

```bash
# .dockerignore 파일 (프로젝트 루트에 생성)
node_modules/       # 의존성 (컨테이너에서 다시 설치)
.env                # 환경변수 (보안)
.env.local
.git/               # Git 이력 (불필요)
*.log               # 로그 파일
dist/               # 빌드 산출물
__pycache__/        # Python 캐시
*.pyc
.DS_Store           # Mac 시스템 파일
```

```txt
.gitignore 와 비슷한 문법
COPY . /app 할 때 .dockerignore 에 있는 것은 제외됨

없으면:
  node_modules 전체가 빌드 컨텍스트에 포함됨
  → 빌드 느려짐 / 이미지 커짐
  → .env 도 이미지에 포함될 수 있음 (보안 위험)
```

---

---

# ⑤ chmod +x 잊었을 때 ⭐️

```dockerfile
# 쉘 스크립트 복사 후 반드시 실행 권한 추가
COPY entrypoint.sh /entrypoint.sh
RUN chmod +x /entrypoint.sh     # ← 이거 없으면 permission denied

ENTRYPOINT ["/entrypoint.sh"]
```

```txt
permission denied 에러 → Dockerfile 에 chmod +x 추가 후 다시 빌드

chmod +x 란:
  chmod = change mode (권한 변경)
  +x    = execute 권한 추가 (실행 가능하게)
  → 이거 없으면 파일이 있어도 실행 불가
```

---

---

# ⑥ 빌드 관련 명령어 모음

```bash
# 빌드
docker build -t 이름:태그 .
docker build -t 이름:태그 -f 다른Dockerfile경로 .
docker build --no-cache -t 이름 .   # 캐시 무시

# 이미지 확인
docker images
docker images 이름                   # 특정 이름만

# 이미지 삭제
docker rmi 이미지이름
docker rmi $(docker images -q)       # 전부 삭제 ⚠️

# 안 쓰는 이미지 정리
docker image prune                   # dangling 이미지만
docker image prune -a                # 전부 (실행 중인 컨테이너 이미지 제외)
```

---

---

# 수정 → 재빌드 체크리스트

```txt
Dockerfile 수정 후:
  □ docker build -t 이름 .
  □ 기존 컨테이너 stop + rm
  □ docker run 으로 새 컨테이너 실행

자주 하는 실수:
  □ 빌드 안 하고 run → 변경 사항 없음
  □ chmod +x 빠뜨림 → permission denied
  □ COPY 경로 오타 → 파일 없음 에러
  □ .dockerignore 없어서 node_modules 포함
```

---

---

# ⑦ 멀티 스테이지 빌드 ⭐️

```txt
문제:
  빌드 도구(pip, npm 등)가 최종 이미지에 포함됨
  → 이미지 크기 불필요하게 커짐

해결:
  여러 FROM 단계로 나눔
  빌드 단계 → 설치/컴파일
  최종 단계 → 결과물만 복사
```

```dockerfile
# 1단계: builder (설치 담당)
FROM python:3.9-slim AS builder
WORKDIR /app
COPY requirements.txt .
RUN pip install --user --no-cache-dir -r requirements.txt

# 2단계: 최종 이미지 (결과물만)
FROM python:3.9-slim
WORKDIR /app

COPY --from=builder /root/.local /root/.local
# ↑ "builder 단계의 /root/.local 을 현재 단계로 복사"

COPY app.py .
ENV PATH=/root/.local/bin:$PATH
CMD ["python", "app.py"]
```

```txt
AS builder:
  FROM 단계에 이름 붙이기
  이후 --from=builder 로 참조

COPY --from=builder:
  다른 빌드 단계에서 파일 복사

효과:
  최종 이미지에 pip 캐시 / 빌드 도구 없음
  이미지 크기 감소 (136MB → 129MB 예시)

레이어 생성 여부 정리:
  레이어 생성: FROM / COPY / RUN / ADD
  레이어 없음: WORKDIR / ENV / LABEL / EXPOSE / CMD / ENTRYPOINT / ARG / USER
```

```bash
docker build -t my-app:multi .
docker images | grep my-app   # 크기 비교
```

---

## Go 언어 멀티 스테이지 예시 ⭐️

```dockerfile
# 1단계: builder (Go 컴파일)
FROM golang:1.14-alpine AS builder

WORKDIR /app
COPY main.go .
RUN go build -o app
#            ↑ -o = output 파일명 지정
#            컴파일 결과를 "app" 이라는 실행파일로 저장

# 2단계: 최종 이미지 (실행파일만)
FROM alpine

COPY --from=builder /app/app .
# builder 단계의 /app/app (컴파일된 실행파일) 만 복사

CMD ["./app"]
```

```txt
왜 멀티 스테이지가 효과적인가:
  builder 단계: golang:1.14-alpine (300MB+)
    Go 컴파일러 / 빌드 도구 전부 포함

  최종 단계: alpine (5MB)
    컴파일된 실행파일 하나만 포함
    Go 컴파일러 필요 없음

  최종 이미지 크기: 300MB → 10MB 수준

-o 플래그:
  go build -o app
  = 컴파일 결과 파일명을 "app" 으로 지정
  없으면 디렉토리명이 기본 파일명이 됨
```

---

---

# ⑧ .dockerignore 고급 패턴 ⭐️

```txt
** = 모든 하위 디렉토리 포함
* = 현재 디렉토리만
```

```bash
# Python 프로젝트 .dockerignore
**/.git
**/__pycache__
**/*.pyc
**/*.pyo
**/venv
**/env
**/.env
**/*.log

# Node.js 프로젝트
**/node_modules
**/dist
**/.next

# 공통
.DS_Store
.gitignore
README.md
```

```txt
**/*.pyc 의미:
  /app/utils/helper.pyc
  /app/models/user.pyc
  어디 있든 .pyc 파일 전부 제외

없으면:
  COPY . /app 할 때 venv / node_modules 전체 포함
  → 빌드 느림 / 이미지 커짐 / .env 노출 위험
```