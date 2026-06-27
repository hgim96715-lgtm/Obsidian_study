---
aliases:
  - Docker 명령어
  - docker run
  - docker ps
  - docker create
  - --format
tags:
  - Docker
related:
  - "[[00_Docker_HomePage]]"
  - "[[Docker_Concept]]"
  - "[[Docker_Image]]"
  - "[[Docker_Container]]"
---
# Docker_Commands — 기본 명령어

## 한 줄 요약

```
docker pull    이미지 다운로드
docker create  컨테이너 생성
docker run     컨테이너 실행
docker ps      실행 중인 컨테이너 목록
docker stop    컨테이너 중지
docker rm      컨테이너 삭제
docker rmi     이미지 삭제
```

---

---

#  이미지 관련

```bash
# 이미지 다운로드
docker pull nginx
docker pull nginx:latest         # 버전 생략 시 latest 자동
docker pull python:3.7           # 특정 버전 지정

# 이미지 목록 확인
docker images
docker images python             # 특정 이미지만

# 이미지 삭제
docker rmi python:3.7
docker rmi 이미지ID

# 이미지 검색 (Docker Hub 에서)
docker search nginx
```

```
docker images 출력:
  REPOSITORY   이름 (nginx)
  TAG          버전 (latest / 3.7)
  IMAGE ID     고유 ID
  CREATED      생성 날짜
  SIZE         크기
```

---
---
# docker create — 생성만 (실행 안 함) 

```bash
# 생성만 하고 실행 안 함
docker create --name web1 nginx:1.22.1
# → 상태: Created (실행 안 됨)

# 이후 직접 시작
docker start web1
# → 상태: Running
```

```
docker create vs docker run:
  docker run    생성 + 즉시 실행 (한 번에)
  docker create 생성만 / 실행은 docker start 로 따로

  create 를 쓰는 이유:
    생성해두고 나중에 필요할 때 시작
    여러 컨테이너를 미리 세팅 후 순서대로 시작
    실무에서는 대부분 docker run 사용
```

---

---

# 컨테이너 실행 — docker run ⭐️

## docker run 문법 구조 ⭐️

```bash
docker run [옵션들] 이미지명 [명령어]
#          ↑         ↑        ↑
#          옵션     반드시    이미지 실행 후
#          먼저     맨 마지막  실행할 명령 (선택)

# 이미지명은 반드시 맨 뒤에 (명령어보다 앞)
docker run -d --name my_apache -p 80:80 httpd
#                                        ↑ 이미지 = httpd (맨 뒤)

docker run -d --name my_mysql -e MYSQL_ROOT_PASSWORD=password mysql
#                                                               ↑ 이미지 = mysql (맨 뒤)

# 이미지 뒤에 명령어 붙이면 그 명령어 실행
docker run ubuntu ls -la      # ubuntu 컨테이너에서 ls -la 실행
docker run -it ubuntu /bin/bash   # ubuntu 컨테이너에서 bash 실행
```

```
순서 규칙:
  docker run  [옵션들]  이미지명  [명령어]
  -d -p -e --name 등 옵션은 순서 무관
  이미지명은 옵션 다음, 명령어보다 앞
  이미지명 뒤의 것은 컨테이너 안에서 실행할 명령어로 해석됨
```

## -e — 환경변수 설정 ⭐️

```bash
# 단일 환경변수
docker run -d -e MYSQL_ROOT_PASSWORD=password mysql

# 여러 환경변수 (각각 -e 반복)
docker run -d \
  -e DB_HOST=localhost \
  -e DB_PORT=5432 \
  -e DB_USER=postgres \
  postgres

# 환경변수 파일로 한번에 (--env-file)
docker run -d --env-file .env postgres
```

```
-e KEY=VALUE:
  컨테이너 안에서 process.env.KEY 로 접근 가능
  비밀번호 / DB 접속 정보 / API 키 등
  이미지마다 요구하는 환경변수 이름이 다름
    mysql  → MYSQL_ROOT_PASSWORD (필수)
    postgres → POSTGRES_PASSWORD (필수)
    mongo  → MONGO_INITDB_ROOT_USERNAME / PASSWORD
```

## --link — 컨테이너 간 연결 (구버전 방식)

```bash
# --link [연결할 컨테이너명]:[별칭]
docker run -d --name my_app \
  --link my_mysql:mysql \
  --link my_apache:apache \
  my-app

# 별칭으로 접근
# my_app 컨테이너 안에서:
mysql -h mysql -uroot -ppassword
#          ↑ 별칭으로 접근 (컨테이너 이름 아닌 별칭)
```

```
--link 동작:
  연결된 컨테이너의 호스트명을 별칭으로 등록
  --link my_mysql:mysql → mysql 이라는 호스트명으로 접근 가능

⚠️ --link 는 레거시 방식 (deprecated)
현재는 --network 방식 사용 권장
```

## --network — 컨테이너 간 연결 (현재 표준) ⭐️

```bash
# 1. 네트워크 생성
docker network create my-app-network

# 2. 같은 네트워크로 각 컨테이너 실행
docker run -d --name my_mysql \
  --network my-app-network \
  -e MYSQL_ROOT_PASSWORD=password \
  mysql

docker run -d --name my_apache \
  --network my-app-network \
  -p 80:80 \
  httpd

docker run -d --name my_app \
  --network my-app-network \
  -e DB_HOST=my_mysql \
  -e DB_USER=root \
  -e DB_PASSWORD=password \
  my-app

# 같은 네트워크 안 → 컨테이너 이름으로 바로 접근
# my_app 안에서: mysql -h my_mysql -uroot -ppassword
```

```bash
--link vs --network:
  --link    구버전 / 단방향 / 별칭 필요 / deprecated
  --network 현재 표준 / 같은 네트워크 내 모든 컨테이너 통신
            컨테이너 이름이 자동으로 호스트명
            별칭 없이 컨테이너 이름 그대로 사용

→ 자세한 내용: [[Docker_Network]]
```

---
## 기타 주요 옵션

```bash
# 포그라운드 실행 (기본 — 로그 바로 출력, Ctrl+C 로 종료)
docker run nginx

# 대화형 실행 (-it = interactive + tty)
docker run -it --name my-ubuntu ubuntu /bin/bash
# exit 입력 → 컨테이너 종료

# 실행 후 자동 삭제 (--rm)
docker run --rm python:latest python --version

# 포트 매핑 (-p 호스트포트:컨테이너포트)
docker run -d -p 8080:80 nginx
# localhost:8080 → 컨테이너 80포트

# 볼륨 마운트 (-v 호스트경로:컨테이너경로)
docker run -d -v ~/project/data:/usr/share/nginx/html nginx

# 환경변수 파일로 설정 (--env-file)
docker run -d --env-file .env postgres

# 리소스 제한
docker run -d --memory=512m --cpus=0.5 nginx

# 네트워크 지정
docker run -d --network my-network nginx

# 재시작 정책 (--restart)
docker run -d --restart unless-stopped nginx

# 작업 디렉토리 지정 (-w)
docker run -d -w /app my-app
#              ↑ 컨테이너 내부 작업 디렉토리

# 여러 옵션 조합
docker run -d \
  --name my-app \
  -p 3000:3000 \
  -v ~/data:/app/data \
  -e NODE_ENV=production \
  --restart unless-stopped \
  my-app:latest
```

```
docker run 주요 옵션:
  -d           백그라운드 실행
  -it          대화형 터미널
  --rm         종료 시 자동 삭제
  --name       컨테이너 이름
  -p           포트 매핑 (호스트:컨테이너)
  -v           볼륨 마운트 (호스트경로:컨테이너경로)
  -e           환경변수
  --env-file   환경변수 파일
  --memory     메모리 제한 (256m / 1g)
  --cpus       CPU 제한 (0.5 = 반 코어)
  --network    네트워크 연결
  --restart    재시작 정책
  -w           작업 디렉토리
```

## --restart 재시작 정책 ⭐️

```bash
# 정책 종류
docker run -d --restart no            nginx  # 기본값 — 자동 재시작 안 함
docker run -d --restart on-failure    nginx  # 에러 종료 시만 재시작
docker run -d --restart always        nginx  # 항상 재시작 (docker 재시작 시도)
docker run -d --restart unless-stopped nginx  # 직접 stop 하지 않는 한 재시작

# 재시작 정책 확인
docker inspect -f '{{.HostConfig.RestartPolicy.Name}}' 컨테이너명
```

```
실무:
  서버/DB 컨테이너 → unless-stopped (가장 많이 씀)
  일회성 작업      → no (기본)
  크래시 복구용    → on-failure
```


---

---

# 컨테이너 목록 & 생명주기 ⭐️

```bash
# 실행 중인 컨테이너만
docker ps

# 전체 (종료된 것 포함)
docker ps -a

# 중지
docker stop my-nginx

# 중지된 컨테이너 다시 시작
docker start my-nginx

# 재시작
docker restart my-nginx

# 컨테이너 삭제 (정지 후)
docker rm my-nginx

# 실행 중인 것 강제 삭제
docker rm -f my-nginx
```

## --format — 출력 포맷 지정 ⭐️

```bash
# 이름만 출력
docker ps --format '{{.Names}}'
# web1
# my-nginx

# 이름 + 상태
docker ps --format '{{.Names}}\t{{.Status}}'
# web1      Up 2 hours
# my-nginx  Up 5 minutes

# 이름 + 포트
docker ps --format '{{.Names}}\t{{.Ports}}'

# 테이블 형식 (헤더 포함)
docker ps --format 'table {{.Names}}\t{{.Image}}\t{{.Status}}'
# NAMES     IMAGE   STATUS
# web1      nginx   Up 2 hours
```

```
자주 쓰는 {{.필드}}:
  {{.Names}}     컨테이너 이름
  {{.Image}}     이미지 이름
  {{.Status}}    상태 (Up / Exited)
  {{.Ports}}     포트 매핑
  {{.ID}}        컨테이너 ID (전체)
  {{.Command}}   실행 명령어
  {{.CreatedAt}} 생성 시간

docker inspect 에서도 같은 방식 사용:
  docker inspect --format '{{.State.Status}}' 컨테이너명
  docker inspect --format '{{.NetworkSettings.IPAddress}}' 컨테이너명
```

```
stop vs rm:
  stop  중지 (데이터 유지 / start 로 재시작 가능)
  rm    완전 삭제
```

---

---

# 실행 중 컨테이너 조작

```bash
# 내부 bash 접속
docker exec -it my-nginx bash
# exit → 호스트로 돌아옴 (컨테이너 유지)

# 단일 명령 실행
docker exec my-nginx echo "Hello"

# 로그 확인
docker logs my-nginx
docker logs --tail 20 my-nginx    # 최근 20줄
docker logs -f my-nginx           # 실시간 (Ctrl+C 종료)
docker logs --timestamps my-nginx # 타임스탬프 포함

# 파일 복사
docker cp 호스트파일 my-nginx:/컨테이너/경로    # 호스트→컨테이너
docker cp my-nginx:/etc/nginx/nginx.conf ./     # 컨테이너→호스트

# 상세 정보
docker inspect my-nginx
docker inspect -f '{{.State.Status}}' my-nginx  # 상태만
docker port my-nginx                             # 포트 매핑 확인

# 실시간 리소스 사용량
docker stats my-nginx    # Ctrl+C 종료
docker stats             # 전체 컨테이너
```

---

---

#  이미지 저장 & 불러오기

```bash
# 이미지 → tar 파일로 저장 (레지스트리 없이 전달할 때)
docker save nginx > nginx.tar
docker save -o nginx.tar nginx    # 동일

# tar 파일 → 이미지로 불러오기
docker load < nginx.tar
docker load -i nginx.tar          # 동일

# 이미지 태그 달기
docker tag nginx:latest my-nginx:v1
# nginx:latest 와 my-nginx:v1 은 같은 이미지 (이름만 다름)
```

```
언제 save/load 쓰나:
  인터넷 없는 서버로 이미지 전달
  특정 버전 백업
  레지스트리 없이 팀원과 공유
```

---

---

#  정리 명령어

```bash
# 정지된 컨테이너 전부 삭제
docker container prune

# 사용하지 않는 이미지 삭제
docker image prune

# 전체 정리 (컨테이너/이미지/네트워크/볼륨)
docker system prune

# 사용량 확인
docker system df
```

---

---

# 명령어 한눈에

|명령어|역할|
|---|---|
|`docker pull 이미지:태그`|이미지 다운로드|
|`docker images`|이미지 목록|
|`docker rmi 이미지`|이미지 삭제|
|`docker run -d 이미지`|백그라운드 실행|
|`docker run -it 이미지 bash`|대화형 실행|
|`docker run --rm 이미지`|실행 후 자동 삭제|
|`docker run -e KEY=VAL`|환경변수 설정|
|`docker run --memory=512m --cpus=0.5`|리소스 제한|
|`docker ps`|실행 중 컨테이너 목록|
|`docker ps -a`|전체 컨테이너 목록|
|`docker start 컨테이너`|중지된 컨테이너 시작|
|`docker stop 컨테이너`|중지|
|`docker restart 컨테이너`|재시작|
|`docker rm 컨테이너`|컨테이너 삭제|
|`docker exec -it 컨테이너 bash`|내부 접속|
|`docker logs -f 컨테이너`|실시간 로그|
|`docker cp 호스트 컨테이너:경로`|파일 복사|
|`docker inspect 컨테이너`|상세 정보|
|`docker stats 컨테이너`|리소스 사용량|
|`docker save 이미지 > 파일.tar`|이미지 저장|
|`docker load < 파일.tar`|이미지 불러오기|
|`docker tag 원본 새이름:태그`|태그 달기|