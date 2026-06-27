---
aliases:
  - Docker 이미지
  - docker pull
  - 레이어
  - Dockerfile
  - 이미지 삭제 순서
tags:
  - Docker
related:
  - "[[00_Docker_HomePage]]"
  - "[[Docker_Commands]]"
  - "[[Docker_Dockerfile]]"
---

# Docker_Image — 이미지 & 레이어

## 한 줄 요약

```
이미지 = 컨테이너를 만드는 설계도 (읽기 전용)
태그   = 이미지의 버전 (latest / 3.7 / v1 등)
레이어 = 이미지를 쌓아 올리는 방식 (변경된 부분만 저장)
```

---

---

#  이미지 다운로드 & 버전

```bash
# 기본 (latest 자동)
docker pull nginx
docker pull python

# 특정 버전 지정
docker pull python:3.7
docker pull python:3.11

# 버전별 확인
docker images python
# REPOSITORY   TAG      IMAGE ID       SIZE
# python       latest   98ccd1643c71   920MB
# python       3.7      1f1a7b570fbd   907MB
```

```
같은 이미지라도 TAG 가 다르면 다른 버전
latest = 태그 생략 시 기본값 / 가장 최신 버전
버전 고정 필요할 때는 반드시 TAG 명시 (python:3.7)
```

---

---

#  레이어 (Layer) — 쉽게 이해하기 ⭐️

## 레이어가 뭔가

```
이미지는 여러 레이어(층)를 쌓아서 만들어짐
각 레이어 = 파일 변경 사항 묶음 (스냅샷)

팬케이크 비유:
  1층 (베이스)    Ubuntu OS
  2층             Nginx 설치
  3층             설정 파일 복사
  4층             내가 추가한 html 파일
  ↑ 이렇게 쌓인 것이 하나의 이미지

각 층을 레이어라고 함
내 변경사항 = 맨 위 레이어 1개만 추가됨
```

## 레이어가 왜 좋은가

```
상황:
  nginx 이미지 (3개 레이어)   150MB
  custom-nginx 이미지 (4개 레이어)  153MB  ← 내가 만든 것

  custom-nginx = nginx 3개 레이어 + 내 변경사항 1개 레이어

저장 방식:
  nginx 이미지:          레이어A + 레이어B + 레이어C  150MB
  custom-nginx 이미지:   레이어A + 레이어B + 레이어C + 레이어D

  레이어A,B,C 는 공유! (중복 저장 안 함)
  실제 추가 용량 = 레이어D 3MB 만
```

```
이점 3가지:
  1. 저장 공간 절약
     같은 레이어 여러 이미지에서 공유
     nginx 기반 이미지 10개 만들어도 nginx 레이어는 1개만 저장

  2. 다운로드 속도 향상
     이미 있는 레이어는 다시 다운 안 함
     pull 할 때 "Already exists" = 레이어 재사용

  3. 빌드 속도 향상
     변경 안 된 레이어는 캐시 사용
     코드만 바꿨으면 코드 레이어만 다시 빌드
```

## 레이어 확인

```bash
# 이미지 레이어 목록 확인
docker inspect --format='{{.RootFS.Layers}}' nginx
# [sha256:abc123... sha256:def456... sha256:ghi789...]
#   ↑ 각 레이어의 SHA256 해시값 (고유 ID)

# custom-nginx 레이어 확인 (nginx 보다 1개 많음)
docker inspect --format='{{.RootFS.Layers}}' custom-nginx
```

---

---

# Dockerfile — 이미지 만드는 설명서 ⭐️

```dockerfile
# Dockerfile
FROM nginx                    ← 베이스 이미지 (1번 레이어)
RUN echo "Hello" > /usr/share/nginx/html/hello.html
                              ← 파일 추가 (2번 레이어)
```

```
FROM = 어떤 이미지 위에 쌓을지 (베이스)
RUN  = 명령어 실행 → 새 레이어 생성
COPY = 파일 복사 → 새 레이어 생성

Dockerfile 명령어 1줄 = 레이어 1개
→ 명령어 적을수록 레이어 적어짐 → 이미지 작아짐
```

## 이미지 빌드

```bash
# Dockerfile 이 있는 폴더에서
docker build -t custom-nginx .
#             ↑ 이름:태그    ↑ 현재 폴더에서 Dockerfile 찾기

# 태그 지정
docker build -t my-app:v1 .
docker build -t my-app:latest .
```

```
docker build 출력 읽기:
  Step 1/2 : FROM nginx        ← 레이어 1 (nginx 이미지 사용)
  Step 2/2 : RUN echo "Hello"  ← 레이어 2 (명령어 실행)
  Successfully built 73b62663  ← 완성된 이미지 ID
  Successfully tagged custom-nginx:latest
```

---

---

# 태그 (Tag) ⭐️

```bash
# 이미지에 태그 달기
docker tag nginx:latest my-nginx:v1
# nginx:latest = my-nginx:v1 (같은 이미지 / 이름만 다름)

docker images
# REPOSITORY   TAG      IMAGE ID
# nginx        latest   605c77e6   ← 동일한 IMAGE ID
# my-nginx     v1       605c77e6   ← 동일한 IMAGE ID
```

```
태그 = 이미지에 별칭 붙이기
  같은 IMAGE ID → 실제로는 같은 이미지
  이름/버전을 다르게 부를 수 있을 뿐

언제 쓰나:
  버전 관리: my-app:v1 / my-app:v2
  환경 구분: my-app:dev / my-app:prod
  Docker Hub 업로드: 계정명/이미지명:태그
```

---

---

#  이미지 삭제 순서 ⭐️

```bash
# 이미지 삭제 시도
docker rmi python:3.7

# 에러 발생 시:
# Error: unable to remove - container is using this image

# 해결 순서:
# 1. 해당 이미지 사용하는 컨테이너 확인
docker ps -a

# 2. 컨테이너 먼저 삭제
docker rm 컨테이너ID

# 3. 이미지 삭제
docker rmi python:3.7
```

```
삭제 순서 규칙:
  컨테이너 삭제 → 이미지 삭제
  (이미지를 쓰는 컨테이너가 있으면 이미지 삭제 불가)
```

---

---

# 정리

|개념|설명|
|---|---|
|이미지|컨테이너 설계도 (읽기 전용)|
|태그|이미지 버전 (latest / 3.7 / v1)|
|레이어|이미지를 구성하는 층 (변경사항 단위)|
|FROM|Dockerfile 에서 베이스 이미지 지정|
|RUN|명령어 실행 → 새 레이어 생성|