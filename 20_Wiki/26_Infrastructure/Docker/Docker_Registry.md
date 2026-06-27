---
aliases:
  - Docker Registry
  - 레지스트리
  - 프라이빗 레지스트리
  - --format
  - --limit
  - docker search
tags:
  - Docker
related:
  - "[[00_Docker_HomePage]]"
  - "[[Docker_Image]]"
  - "[[Docker_Commands]]"
---
# Docker_Registry — 레지스트리 & Docker Hub

# 한 줄 요약

```txt
레지스트리 = Docker 이미지를 저장하고 공유하는 저장소
Docker Hub = Docker 공식 클라우드 레지스트리 (기본값)
```

---

---

# Docker Hub 흐름 ⭐️

```txt
로컬에서 이미지 빌드
    ↓
docker login       Docker Hub 로그인
    ↓
docker tag         이미지에 계정명/이름:태그 형식으로 이름 붙이기
    ↓
docker push        Docker Hub 에 업로드
    ↓
다른 환경에서
docker pull        Docker Hub 에서 다운로드
    ↓
docker run         실행
```

---

---

# docker login ⭐️

```bash
# Docker Hub 로그인
docker login
# Username: 도커허브계정
# Password: 비밀번호 입력
# Login Succeeded

# 특정 레지스트리 로그인 (Docker Hub 외)
docker login registry.example.com

# 로그아웃
docker logout
```

```txt
docker login 이 필요한 경우:
  docker push  → 본인 계정에 이미지 올릴 때
  private 이미지 pull 할 때

로그인 정보 저장 위치:
  ~/.docker/config.json
  인증 토큰이 저장됨 → 이후 push/pull 자동 인증

Docker Hub 계정 없으면:
  https://hub.docker.com 에서 무료 가입
```

---

---

# docker push — 이미지 업로드 ⭐️

```bash
# 1. 빌드한 이미지 확인
docker images

# 2. Docker Hub 형식으로 태그 지정
docker tag my-app 계정명/my-app:latest
#           ↑ 로컬 이미지  ↑ Docker Hub 이미지명

# 3. push
docker push 계정명/my-app:latest

# 버전 태그도 함께 push
docker tag my-app 계정명/my-app:1.0
docker push 계정명/my-app:1.0
```

```txt
Docker Hub 이미지 이름 규칙:
  계정명/이미지명:태그
  johndoe/my-app:latest
  johndoe/my-app:1.0
  ↑ 계정명 빠지면 push 거부됨 ⚠️

tag 를 붙이는 이유:
  로컬 이미지 이름 → Docker Hub 주소 형식으로 변환
  my-app → johndoe/my-app:latest
  원본 이미지는 그대로 / 별칭을 추가하는 것
```

---

---

# docker pull — 이미지 다운로드 ⭐️

```bash
# Docker Hub 에서 공식 이미지 pull
docker pull nginx
docker pull python:3.11-slim
docker pull postgres:16

# 특정 계정의 이미지 pull
docker pull 계정명/my-app:latest

# pull 후 바로 run (이미지 없으면 자동 pull)
docker run nginx
# → 로컬에 없으면 Docker Hub 에서 자동으로 pull
```

```txt
docker run 은 이미지가 없으면 자동으로 pull
명시적으로 pull 이 필요한 경우:
  최신 버전 업데이트 확인
  run 전에 미리 받아놓기
  특정 버전 지정
```

---

---

# docker search — 이미지 검색

```bash
# Docker Hub 에서 검색
docker search nginx

# 필터 옵션
docker search --filter=stars=100 python   # 별점 100 이상
docker search --filter=is-official=true nginx  # 공식 이미지만
docker search --limit 5 node              # 결과 5개만

# 출력 형식 지정
docker search --format "{{.Name}}: {{.StarCount}}" nginx
```

```txt
검색 결과:
  OFFICIAL  [OK] 표시 = Docker 공식 이미지
  STARS     별점 (신뢰도 지표)
  NAME      이미지 이름

공식 이미지 특징:
  계정명 없이 이름만 (nginx / python / postgres)
  Docker 팀이 관리
  보안 업데이트 꾸준히
```

---

---

# 로컬 레지스트리 구축

```bash
# 사내 / 폐쇄망 환경에서 자체 레지스트리 운영
docker run -d \
  -p 5000:5000 \
  --name registry \
  registry:2

# 로컬 레지스트리에 push
docker tag my-app localhost:5000/my-app
docker push localhost:5000/my-app

# 로컬 레지스트리에서 pull
docker pull localhost:5000/my-app
```

```txt
언제 쓰나:
  인터넷 안 되는 서버 환경
  사내 보안 정책으로 외부 레지스트리 사용 불가
  CI/CD 파이프라인에서 빠른 이미지 공유
```

---

---

# docker rmi — 이미지 삭제 ⭐️

```bash
# 로컬 이미지 삭제
docker rmi 계정명/my-app:latest

# 태그만 삭제 (같은 이미지 ID 를 가리키는 태그가 여러 개면 태그만 제거)
docker rmi 계정명/my-app:1.0

# 이미지 ID 로 삭제
docker rmi abc123def456

# 여러 개 한번에
docker rmi 계정명/my-app:latest 계정명/my-app:1.0
```

```txt
push 후 로컬 이미지 삭제 패턴:
  Docker Hub 에 올린 후 로컬 용량 확보

  docker push 계정명/my-app:latest   # Hub 에 올리고
  docker rmi 계정명/my-app:latest    # 로컬에서 삭제
  docker rmi my-app                  # 원본 이미지도 삭제

  → Docker Hub 에는 그대로 남아있음
  → 필요할 때 docker pull 로 다시 받으면 됨
```

```bash
# rmi 가 안 될 때 — 컨테이너가 이미지를 사용 중
docker rmi 계정명/my-app
# Error: image is being used by container abc123

# 해결 방법
docker ps -a                                       # 전체 컨테이너 목록
docker ps -a --filter ancestor=계정명/my-app:v1   # 이 이미지로 만든 컨테이너만 필터
#                      ↑ ancestor = 부모 이미지 기준으로 필터

docker stop 컨테이너ID               # 실행 중이면 먼저 정지
docker rm 컨테이너ID                 # 컨테이너 삭제
docker rmi 계정명/my-app            # 그 다음 이미지 삭제

# 강제 삭제 (-f)
docker rmi -f 계정명/my-app         # 컨테이너 있어도 강제 삭제 (주의)
```

```txt
--filter ancestor=이미지명:태그:
  해당 이미지로 만들어진 컨테이너만 골라서 보여줌
  rmi 전에 어떤 컨테이너가 이 이미지를 쓰는지 정확히 확인할 때 사용
  docker ps -a 로 전체 보는 것보다 훨씬 정확

  태그 생략하면 모든 태그 포함:
    --filter ancestor=계정명/my-app     모든 버전
    --filter ancestor=계정명/my-app:v1  v1 태그만
```

---

---

# 한눈에

|명령어|역할|
|---|---|
|`docker login`|Docker Hub 로그인|
|`docker logout`|로그아웃|
|`docker tag 이미지 계정/이름:태그`|이미지에 Hub 형식 이름 붙이기|
|`docker push 계정/이름:태그`|Docker Hub 에 업로드|
|`docker pull 이름:태그`|Docker Hub 에서 다운로드|
|`docker search 키워드`|Docker Hub 검색|
|`docker rmi 이미지`|로컬 이미지 삭제|
|`docker rmi -f 이미지`|강제 삭제 (컨테이너 사용 중이어도)|

```txt
push 후 정리 순서:
  docker push 계정명/my-app:latest   Hub 에 업로드
  docker rmi 계정명/my-app:latest    로컬 태그 삭제
  docker rmi my-app                  로컬 원본 이미지 삭제

rmi 안 될 때:
  docker ps -a 로 사용 중인 컨테이너 확인
  docker rm 컨테이너ID 로 먼저 삭제 후 rmi
```