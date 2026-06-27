---
aliases:
  - 볼륨
  - bind mount
  - -v
  - 데이터 영속성
  - inspect
  - 이름:컨테이너경로
  - Mountpoint
  - 볼륨 백업
  - --mount
tags:
  - Docker
related:
  - "[[00_Docker_HomePage]]"
  - "[[Docker_Commands]]"
  - "[[Docker_Container]]"
  - "[[Docker_Network]]"
  - "[[Docker_Dockerfile]]"
---
# Docker_Volume — 볼륨 & 데이터 영속성

## 한 줄 요약

```txt
컨테이너는 삭제하면 데이터도 사라짐
볼륨 = 데이터를 컨테이너 밖에 저장해서 영구 보존
```

---

---

# 왜 볼륨이 필요한가 ⭐️

```txt
컨테이너의 특성:
  컨테이너는 이미지를 기반으로 만들어진 임시 환경
  컨테이너 안에서 파일을 생성/수정해도
  컨테이너를 삭제하면 → 전부 사라짐

문제 상황:
  PostgreSQL 컨테이너에 데이터 저장
  → 컨테이너 재시작 / 삭제 → DB 데이터 전부 날아감

볼륨으로 해결:
  데이터를 컨테이너 밖(호스트 또는 Docker 관리 영역)에 저장
  컨테이너 삭제해도 → 데이터 유지
  새 컨테이너에 같은 볼륨 연결 → 데이터 이어서 사용
```

---

---

#  -v 마운트 구조 ⭐️

```txt
docker run -v 호스트경로:컨테이너경로
               ↑           ↑
         내 PC 의 폴더   컨테이너 안의 경로

이 둘을 연결(동기화):
  컨테이너 경로에 파일 쓰면 → 호스트 경로에 생성
  호스트 경로 파일 수정     → 컨테이너 안에 반영
  컨테이너 삭제해도         → 호스트 파일 유지
```

```bash
# $(pwd) = Linux 명령어 pwd (현재 디렉토리 경로)
# $()    = 명령어 치환 (실행 결과를 문자열로)

docker run -v $(pwd)/data:/data python-volume bash
#              ↑              ↑
#          현재폴더/data       컨테이너 /data

# /home/labex/project 에 있으면:
# $(pwd)/data = /home/labex/project/data → /data 에 연결
```

```txt
호스트 경로 작성법:
  절대경로: /home/user/data:/app/data    ✅
  ~  사용:  ~/project/data:/app/data    ✅
  $(pwd): $(pwd)/data:/app/data          ✅
  상대경로: ./data:/app/data             ❌ (안 됨)

  → 호스트 폴더 없으면 Docker 가 자동 생성
```

---

---

#  볼륨 종류 2가지

## bind mount — 호스트 폴더 직접 연결

```bash
# 호스트 경로를 직접 지정
docker run -d \
  -v ~/project/data:/app/data \
  my-app
```

```txt
특징:
  호스트 경로를 직접 지정
  개발 환경에서 코드 / 설정 파일 공유에 적합
  호스트 파일 변경 → 컨테이너 즉시 반영 (핫리로드)

언제:
  소스코드 개발 중 실시간 반영
  설정 파일을 컨테이너에 주입할 때
  컨테이너 로그를 호스트에서 직접 볼 때
```

## named volume — Docker 관리 볼륨

```bash
# Docker 가 경로를 관리
docker volume create my-data          # 볼륨 생성
docker run -d -v my-data:/app/data my-app  # 볼륨 사용
```

```txt
특징:
  Docker 가 /var/lib/docker/volumes/ 안에 자동 저장
  호스트 경로를 몰라도 됨
  DB 데이터처럼 영구 보존이 필요한 곳에 적합

언제:
  PostgreSQL / MySQL 등 DB 데이터 보존
  컨테이너 간 데이터 공유
  Docker 가 경로 관리해도 되는 경우
```

## 비교

```txt
bind mount:
  -v /home/user/data:/app/data   호스트 경로 직접 지정
  개발 / 설정 파일 / 코드 공유
  호스트에서 파일 직접 열고 수정 가능

named volume:
  -v my-volume:/app/data         볼륨 이름으로 지정
  DB 데이터 / 영구 저장
  docker volume inspect 로 경로 확인 가능
```

---

---

#  named volume 명령어

```bash
docker volume create my-data       # 볼륨 생성
docker volume ls                   # 목록 확인
docker volume inspect my-data      # 상세 정보
docker volume rm my-data           # 삭제
docker volume prune                # 사용하지 않는 볼륨 전부 삭제
```

## inspect 출력 해석 ⭐️

```json
[{
  "Name":       "my_data",
  "Driver":     "local",
  "Mountpoint": "/var/lib/docker/volumes/my_data/_data",
  "Scope":      "local"
}]
```

```txt
Mountpoint:
  Docker 가 실제로 데이터를 저장하는 호스트 경로
  /var/lib/docker/volumes/볼륨이름/_data
  → sudo 로 직접 접근 가능!!
```

---

---

#  실전 패턴

## PostgreSQL 데이터 영속성

```bash
docker run -d \
  --name postgres \
  -v postgres-data:/var/lib/postgresql/data \
  -e POSTGRES_PASSWORD=secret \
  postgres

# 컨테이너 삭제 후에도 postgres-data 볼륨에 DB 유지
docker stop postgres && docker rm postgres

# 새 컨테이너에 같은 볼륨 연결 → 데이터 이어서 사용
docker run -d \
  --name postgres-new \
  -v postgres-data:/var/lib/postgresql/data \
  -e POSTGRES_PASSWORD=secret \
  postgres
```

## 읽기 전용 마운트

```bash
# :ro = read-only (컨테이너가 수정 못 하게)
docker run -d \
  -v ~/config.yml:/app/config.yml:ro \
  my-app
```

## 볼륨 데이터 직접 읽기 ⭐️

```bash
# 볼륨이 마운트된 컨테이너 안의 파일 읽기
docker exec my-container cat /app/data/hello.txt

# 볼륨 안 파일 목록 확인
docker exec my-container ls /app/data

# 로그 파일 실시간 확인
docker exec my-container tail -f /app/data/app.log
```

```txt
볼륨에 쓴 데이터는 컨테이너 안에서 파일처럼 접근 가능
docker exec + cat / ls / tail 로 바로 읽을 수 있음

확인이 필요할 때:
  앱이 볼륨에 제대로 파일 썼는지 확인
  볼륨 안 로그 파일 내용 확인
  DB 컨테이너의 설정 파일 확인
```

---

---

# 컨테이너 간 데이터 공유 ⭐️

```bash
# 같은 볼륨을 두 컨테이너에 연결
docker run -d --name container_a -v my_data:/app/data ubuntu sleep infinity
docker run -d --name container_b -v my_data:/app/data ubuntu sleep infinity

# A 가 파일 생성
docker exec container_a sh -c "echo 'Hello from A' > /app/data/test.txt"

# B 에서 A 가 만든 파일 보임
docker exec container_b cat /app/data/test.txt
# Hello from A
```

```txt
같은 named volume → 여러 컨테이너 실시간 공유
활용: 로그 수집 / 파일 처리 파이프라인
```

---

---

#  볼륨 백업 & 복구 ⭐️

```bash
# 1. 백업 — 임시 Alpine 컨테이너로 볼륨을 tar.gz 압축
docker run --rm \
  -v myvolume:/source \
  -v /home/labex/project:/backup \
  alpine \
  tar czf /backup/myvolume.tar.gz -C /source .

# 2. 새 볼륨 생성
docker volume create mynewvolume

# 3. 복구 — 백업 파일을 새 볼륨에 압축 해제
docker run --rm \
  -v mynewvolume:/target \
  -v /home/labex/project:/backup \
  alpine \
  sh -c "tar xzf /backup/myvolume.tar.gz -C /target"

# 4. 복구 확인 — 새 볼륨에서 파일 읽기
docker run --rm -v mynewvolume:/app/data alpine cat /app/data/hello.txt
```

```txt
흐름:
  1. 원본 볼륨(myvolume) + 백업 폴더 → 임시 컨테이너에 마운트
     tar czf 로 압축 → /backup/myvolume.tar.gz 생성
     --rm → 작업 후 컨테이너 자동 삭제

  2. 복구할 새 볼륨 생성

  3. 새 볼륨(mynewvolume) + 백업 폴더 → 임시 컨테이너에 마운트
     tar xzf 로 압축 해제 → 새 볼륨에 데이터 복원

  4. 새 볼륨 마운트 후 cat 으로 파일 읽어서 복구 성공 확인

Alpine 을 쓰는 이유:
  alpine = 용량이 매우 작은 Linux 이미지 (~5MB)
  tar 명령어만 있으면 되는 임시 작업에 적합
  ubuntu 보다 가볍고 빠름

tar 옵션:
  c  압축 생성 (create)
  x  압축 해제 (extract)
  z  gzip 압축 사용
  f  파일 이름 지정
  v  verbose — 처리 중인 파일 목록 출력
  -C 작업 디렉토리 지정

cvf vs czf ⭐️:
  cvf  압축 없음 (.tar) / 파일 목록 터미널 출력
  czf  gzip 압축 (.tar.gz) / 출력 없음

  백업할 때 czf 권장:
    파일 용량 30~70% 줄어듦
    .tar.gz 이 표준 백업 형식
    v 는 파일 많으면 터미널 도배됨
    → 필요할 때만 czfv 로 추가
```

---

---
# 참고 — --mount 문법

```bash
# 아래 두 줄은 동일한 동작
-v nginx-vol:/usr/share/nginx/html
--mount source=nginx-vol,target=/usr/share/nginx/html
```

```txt
-v 와 --mount 는 같은 동작
--mount 가 더 명시적이고 읽기 쉬움
  source=  볼륨 이름 또는 호스트 경로
  target=  컨테이너 안의 경로

Docker 공식 문서에서 --mount 권장하지만
실무에서는 -v 를 더 많이 사용
```

---
---

# 한눈에

|방식|문법|언제|
|---|---|---|
|bind mount|`-v /호스트경로:/컨테이너경로`|코드·설정 파일 공유|
|named volume|`-v 볼륨이름:/컨테이너경로`|DB·영구 데이터 저장|
|읽기 전용|`-v 경로:경로:ro`|설정 파일 보호|

```txt
VOLUME /data (Dockerfile):
  컨테이너 안 마운트 포인트 선언만
  → 실제 호스트 경로는 docker run -v 로 지정
```