---
aliases:
  - 컨테이너 관리
  - docker start
  - docker exec
  - docker inspect
  - sleep infinity
  - docker cp
  - docker attach
  - TTY
related:
  - "[[00_Docker_HomePage]]"
  - "[[Docker_Commands]]"
  - "[[Docker_Image]]"
tags:
  - Docker
---
# Docker_Container — 컨테이너 관리

```txt
컨테이너 = 이미지로 실행된 프로세스
시작 / 중지 / 재시작 / 삭제 / 내부 접속 / 파일 복사
```

---

---

# 실행 모드 — -d vs -it ⭐️

```bash
# -d : 백그라운드 실행 (Detached mode)
docker run -d --name nginx-server nginx
# 터미널 블록 없음 / 백그라운드에서 계속 실행

# -it : 대화형 실행 (Interactive + TTY)
docker run -it --name ubuntu-shell ubuntu /bin/bash
# 컨테이너 내부 bash 로 접속됨
# exit 입력 → 컨테이너 종료

# 포트 매핑 + 백그라운드
docker run -d --name nginx-port -p 8080:80 nginx
#                                 ↑ 호스트:컨테이너
# localhost:8080 → 컨테이너 80포트
```

```txt
-d    백그라운드 실행 (서버 / DB 등 데몬 프로세스)
-it   대화형 접속 (디버깅 / 탐색 / 쉘 직접 접근)
--rm  종료 시 자동 삭제 (일회성 실행)

-i (interactive)  표준 입력(stdin) 유지 → 키보드 입력 전달
-t (tty)          터미널 에뮬레이터 생성 → 프롬프트 / 색상 표시
                  없으면 bash 는 뜨지만 프롬프트가 안 나옴
→ 항상 -it 묶어서 사용
```

---

---

# sleep infinity / sleep 3600 — 컨테이너 유지 ⭐️

```txt
ubuntu 같은 이미지는 실행할 프로세스가 없으면 즉시 종료됨
→ 아무것도 안 하고 대기하는 명령으로 컨테이너 유지
```

```bash
# sleep infinity = 영원히 대기
docker run -d --name test-ubuntu ubuntu sleep infinity

# sleep 3600 = 3600초(1시간) 대기
docker run -d --name temp-ubuntu ubuntu sleep 3600
#                                              ↑ 초 단위
# 1시간 후 자동 종료

# 실행 후 내부 접속
docker exec -it test-ubuntu bash
```

```txt
sleep infinity vs sleep 3600:
  infinity  무기한 대기 → 수동으로 stop 하기 전까지 유지
  3600      N초 후 자동 종료 → 임시 작업용
  60 = 1분 / 3600 = 1시간 / 86400 = 1일

왜 쓰나:
  ubuntu / alpine 같이 자체 서비스 없는 이미지 테스트
  컨테이너 띄워두고 exec 로 접속해서 작업
  nginx / postgres 는 자체 프로세스 있으므로 불필요
```

---

---

# 생명주기 관리 ⭐️

```bash
docker start nginx-server      # 중지된 컨테이너 다시 시작
docker stop nginx-server       # 컨테이너 중지 (데이터 유지)
docker restart nginx-server    # 재시작

docker ps                      # 실행 중인 컨테이너 목록
docker ps -a                   # 전체 목록 (종료된 것 포함)
docker ps -q                   # ID 만 출력
docker ps -qf "name=nginx"     # 이름 필터 + ID 만 출력
docker ps -f "status=exited"   # 중지된 컨테이너만
```

```txt
stop vs rm:
  stop  중지 (데이터/설정 유지 / 다시 start 가능)
  rm    완전 삭제

-q (quiet):
  ID 만 출력 → 스크립트에서 변수로 쓸 때 유용
  docker exec -it $(docker ps -qf "name=nginx") bash

-f filter 옵션:
  name=     이름으로 필터
  status=   상태로 필터 (running / exited)
```

---

---

# 로그 확인 ⭐️

```bash
docker logs nginx-server                  # 전체 로그
docker logs --tail 10 nginx-server        # 최근 10줄만
docker logs -f nginx-server               # 실시간 (Ctrl+C 종료)
docker logs --timestamps nginx-server     # 타임스탬프 포함
```

---

---

# exec — 내부 접속 & 명령 실행 ⭐️

```bash
# bash 로 대화형 접속
docker exec -it nginx-server /bin/bash
# exit → 호스트로 돌아옴 (컨테이너는 계속 실행 중)

# 명령 바로 실행
docker exec nginx-server cat /etc/nginx/nginx.conf
docker exec nginx-server ls /usr/share/nginx/html
docker exec nginx-server nginx -s reload
```

```txt
exec vs run:
  run   새 컨테이너 생성 + 실행
  exec  이미 실행 중인 컨테이너 안에서 명령 실행
        exit 해도 컨테이너 종료 안 됨
```

---

---

# docker cp — 파일/디렉토리 복사 ⭐️

```txt
문법:
  docker cp 출발지 도착지

  호스트 → 컨테이너:  docker cp 호스트경로  컨테이너명:컨테이너경로
  컨테이너 → 호스트:  docker cp 컨테이너명:컨테이너경로  호스트경로
```

## 파일 복사

```bash
# 호스트 → 컨테이너
docker cp hello.html nginx-server:/usr/share/nginx/html/hello.html
#          ↑ 로컬 파일         ↑ 컨테이너명:붙여넣을경로

# 컨테이너 → 호스트
docker cp nginx-server:/etc/nginx/nginx.conf ~/project/nginx.conf
#          ↑ 컨테이너명:가져올경로              ↑ 로컬 저장경로
```

## 디렉토리 복사 ⭐️

```bash
# 디렉토리 준비
mkdir /tmp/myfiles && cd /tmp/myfiles
touch file1 file2

# 호스트 디렉토리 → 컨테이너
docker cp /tmp/myfiles/ my_container:/tmp/
#                     ↑ 끝에 / 붙이면 디렉토리 안의 내용 복사

# 컨테이너 디렉토리 → 호스트
docker cp my_container:/tmp/myfiles/ /home/user/backup/
```

```txt
경로 문법 정리:
  컨테이너명:경로   → 컨테이너 안의 경로 (: 로 구분)
  /tmp/myfiles/   → 끝에 / 있으면 디렉토리 내용 복사
  /tmp/myfiles    → 끝에 / 없으면 디렉토리 자체 복사

언제 쓰나:
  설정 파일을 컨테이너에 밀어넣기
  컨테이너 내 로그 / 결과물을 호스트로 꺼내기
  볼륨 마운트 없이 파일 전달할 때
```

---

---

# inspect — 상세 정보

```bash
docker inspect nginx-server            # 전체 정보 (JSON)

# 특정 값만 추출 (-f = format)
docker inspect -f '{{.State.Status}}' nginx-server
# running / exited / paused

docker inspect -f '{{range .NetworkSettings.Networks}}{{.IPAddress}}{{end}}' nginx-server
# 172.17.0.2

docker port nginx-server               # 포트 매핑 확인
# 80/tcp -> 0.0.0.0:8080
```

---

---

# 환경변수 설정 ⭐️

```bash
# -e 로 직접 지정
docker run -d --name app \
  -e DB_HOST=localhost \
  -e DB_PORT=5432 \
  ubuntu sleep infinity

# 확인
docker exec app env | grep DB_

# --env-file 로 파일에서 읽기
# .env 파일 내용:
# DB_HOST=localhost
# DB_PORT=5432

docker run -d --name app2 \
  --env-file .env \
  ubuntu sleep infinity
```

```txt
-e KEY=VALUE        단일 환경변수
--env-file 파일명   파일에서 여러 개 한번에 로드
```

---

---

# 리소스 제한 ⭐️

```bash
docker run -d --name limited-app \
  --memory=512m \    # 최대 512MB
  --cpus=0.5 \       # 최대 CPU 0.5 코어
  nginx

# 실시간 리소스 사용량
docker stats limited-app
# CONTAINER   CPU %   MEM USAGE / LIMIT   MEM %   NET I/O
```

```txt
왜 필요한가:
  여러 컨테이너가 같은 서버에서 실행될 때
  하나가 자원 독점 → 다른 컨테이너 영향
  → 제한 걸어서 공정하게 분배
```

---

---

# attach — 메인 프로세스에 연결 ⭐️

```bash
# -itd = 대화형 + TTY + 백그라운드 (attach 가능하도록)
docker run -itd --name my-nginx nginx

# 실행 중인 컨테이너 메인 프로세스에 연결
docker attach my-nginx
# nginx 로그가 터미널에 출력됨

# 분리 (컨테이너 종료 없이 빠져나오기)
# Ctrl+P 누른 채로 → Ctrl+Q
```

```txt
attach vs exec:
  attach  PID 1 (메인 프로세스) 에 직접 연결
          분리: Ctrl+P, Ctrl+Q
          ⚠️ exit 또는 Ctrl+C → 컨테이너 종료될 수 있음

  exec    새 프로세스를 컨테이너 안에서 실행
          exit 해도 컨테이너 계속 실행
          → 실무에서 더 많이 씀

Ctrl+P+Q:
  attach 상태에서 컨테이너 종료 없이 빠져나오는 방법
  "read escape sequence" 출력 후 터미널로 돌아옴
  -itd 로 시작한 컨테이너에서만 동작
```

---

---

# 명령어 한눈에

|명령어|역할|
|---|---|
|`docker run -d 이미지`|백그라운드 실행|
|`docker run -it 이미지 bash`|대화형 접속|
|`docker run -d ubuntu sleep infinity`|프로세스 없는 이미지 유지|
|`docker start 컨테이너`|중지된 컨테이너 재시작|
|`docker stop 컨테이너`|컨테이너 중지|
|`docker logs -f 컨테이너`|실시간 로그|
|`docker exec -it 컨테이너 bash`|내부 접속|
|`docker cp 호스트파일 컨테이너:경로`|파일/디렉토리 복사 (→컨테이너)|
|`docker cp 컨테이너:경로 호스트경로`|파일/디렉토리 복사 (→호스트)|
|`docker inspect 컨테이너`|상세 정보|
|`docker port 컨테이너`|포트 매핑 확인|
|`docker stats 컨테이너`|실시간 리소스 사용량|
|`docker attach 컨테이너`|메인 프로세스 연결|
|`docker ps -qf "name=이름"`|ID 필터 출력|