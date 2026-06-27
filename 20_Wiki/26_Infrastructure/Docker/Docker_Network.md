---
aliases:
  - 도커 네트워크
  - --network
  - 컨테이너 통신
  - bridge
  - host
tags:
  - Docker
related:
  - "[[00_Docker_HomePage]]"
  - "[[Docker_Commands]]"
  - "[[Docker_Compose]]"
  - "[[Docker_Container]]"
  - "[[Docker_Volume]]"
---
# Docker_Network — 네트워크

```
컨테이너끼리 통신하려면 같은 네트워크에 있어야 함
같은 네트워크 안에서는 컨테이너 이름으로 통신 가능
외부(호스트/브라우저)에서 접근하려면 -p 포트 매핑 필요
```


---

---

# 네트워크 타입

| |bridge (기본)|사용자 정의 bridge|host|none|
|---|---|---|---|---|
|컨테이너 이름 DNS|❌|✅ ⭐️|-|-|
|격리|✅|✅|❌|완전 격리|
|포트 매핑|필요|필요|불필요|불가|
|성능|보통|보통|최고|-|
|포트 충돌|없음|없음|주의 ⚠️|-|
|OS|전체|전체|Linux 전용|전체|

```
bridge (기본):
  Docker 가 자동으로 만드는 가상 네트워크
  --network 생략 시 기본 bridge 에 연결됨
  ⚠️ 기본 bridge 에서는 컨테이너 이름 DNS 안 됨
     → 사용자 정의 bridge 를 만들어야 이름으로 통신 가능

사용자 정의 bridge:
  docker network create 로 직접 생성
  이름으로 컨테이너 간 통신 가능 ⭐️
  실무에서 가장 많이 사용

host:
  컨테이너가 호스트 네트워크 스택 직접 공유
  Linux 전용

none:
  네트워크 인터페이스 없음 (완전 격리)
```

---

---

# 네트워크 생성 ⭐️

```bash
# 기본 생성 (자동으로 bridge 드라이버 사용)
docker network create my-network

# 드라이버 명시 (bridge 명시적으로 지정)
docker network create --driver bridge my-network2
#                      ↑ --driver = 네트워크 타입 지정
#                        생략하면 기본값 bridge
```

```
--driver 옵션:
  bridge  컨테이너 전용 가상 네트워크 (기본값)
  host    호스트 네트워크 직접 사용
  none    네트워크 없음

docker network create 만 써도 bridge 드라이버로 생성됨
--driver bridge 는 명시적으로 드라이버를 확인/지정할 때 사용
```

---

---

# 컨테이너 실행 & 네트워크 연결 ⭐️

```bash
# 네트워크 생성
docker network create my-network

# 컨테이너 실행 시 네트워크 지정
docker run -d --name db    --network my-network postgres
docker run -d --name app   --network my-network my-app
docker run -d --name nginx --network my-network nginx

# -itd 조합 (alpine 처럼 인터랙티브 컨테이너)
docker run -itd --name alpine-container --network my-network alpine
#           ↑
#  -i = stdin 열기 / -t = 터미널 / -d = 백그라운드
```

```
-d  (Detached) 백그라운드 실행
  있으면:  컨테이너 ID 출력 후 프롬프트 즉시 반환
  없으면:  컨테이너 로그가 터미널에 계속 출력됨
          Ctrl+C 로 종료하면 컨테이너도 종료

  서버 / 데몬 프로세스 → 항상 -d 사용
  디버깅 / 로그 확인   → -d 없이 (또는 docker logs)

같은 네트워크 안에서:
  컨테이너 이름 = 호스트명으로 바로 사용 가능
  app → db 접속: host="db", port=5432
  IP 주소 외울 필요 없음 ⭐️
```

---

---

# 컨테이너 간 통신 — DNS 자동 해석 ⭐️

```bash
# 같은 네트워크 안 컨테이너끼리 이름으로 통신
docker exec alpine-container  ping -c 4 nginx-container
docker exec container1        curl -s   container2
#                                   ↑ silent 모드 (진행 상태 출력 생략)
```

```
사용자 정의 bridge 네트워크 내부 DNS:
  Docker 내장 DNS 서버가 컨테이너 이름 → IP 자동 해석
  container2 → 172.18.0.3 (Docker 가 자동으로 찾아줌)

  ✅ 사용자 정의 bridge  이름으로 통신 가능
  ❌ 기본 bridge         이름 DNS 안 됨 (IP 직접 지정 필요)

다른 네트워크 컨테이너:
  통신 자체 불가 (격리)
```

## ping 결과 해석

```bash
docker exec container1 ping -c 4 container2

# 성공 케이스
PING container2 (172.18.0.3) 56(84) bytes of data.
64 bytes from container2 (172.18.0.3): icmp_seq=1 ttl=64 time=0.052 ms

2 packets transmitted, 2 received, 0% packet loss
```

```
PING container2 (172.18.0.3)
  → 이름을 IP 로 DNS 해석 성공

0% packet loss
  → 통신 정상

# 실패 케이스
ping: container3: Temporary failure in name resolution
  → DNS 해석 실패 = 다른 네트워크 = 통신 불가

-c 옵션:
  ping -c 4  4번만 보내고 종료
  -c 없으면  Ctrl+C 누를 때까지 무한 전송 ⚠️
  컨테이너 환경에서는 항상 -c 로 횟수 제한
```

---

---

# 네트워크 관리 명령어

```bash
# 목록 확인
docker network ls

# 상세 정보 (어떤 컨테이너가 연결됐는지)
docker network inspect my-network
docker network inspect bridge

# 생성 / 삭제
docker network create my-network
docker network rm my-network
docker network prune              # 사용하지 않는 네트워크 전부 삭제

# 실행 중인 컨테이너에 네트워크 추가 / 제거
docker network connect    my-network 컨테이너명
docker network disconnect my-network 컨테이너명
```

## ⚠️ docker network connect 순서 주의 ⭐️

```bash
# ✅ 올바른 순서: 네트워크명 먼저, 컨테이너명 나중
docker network connect my-network container2

# ❌ 잘못된 순서 → 에러
docker network connect container2 my-network
# Error: No such container: my-network
# → Docker 가 my-network 를 컨테이너 이름으로 해석
```

```
문법:
  docker network connect  <네트워크명>  <컨테이너명>
                              ↑ 첫번째      ↑ 두번째

기억법:
  "네트워크에 컨테이너를 붙인다"
  → 네트워크가 먼저, 컨테이너가 나중
```

## docker network inspect 주요 항목

```
Subnet      컨테이너들의 IP 범위  (bridge 기본: 172.17.0.0/16)
Gateway     외부 통신 게이트웨이  (bridge 기본: 172.17.0.1)
Containers  현재 연결된 컨테이너 목록
enable_icc  true = 컨테이너 간 통신(ICC) 허용
```

## docker inspect — 컨테이너 상세 정보

```bash
docker inspect 컨테이너명                      # 전체 JSON 출력
docker inspect 컨테이너명 | grep IPAddress     # grep 으로 원하는 값만

# --format 으로 특정 필드만 추출
docker inspect --format '{{.HostConfig.NetworkMode}}' 컨테이너명
# bridge / host / none

docker inspect --format '{{.NetworkSettings.IPAddress}}' 컨테이너명
# 172.17.0.3

docker inspect --format '{{.State.Status}}' 컨테이너명
# running / exited / paused
```

```
--format 없으면 JSON 전체 출력 → grep 과 조합
--format '{{.필드}}'  → 원하는 필드만 깔끔하게 추출
```

---

---

# Docker Compose — 자동 네트워크 ⭐️

### docker-compose.yml 예시

```yaml
#[[Docker_Compose]] 참고 
version: "3"
services:
  nginx:
    image: nginx
    ports:
      - "80:80"
    networks:
      - my-network

  alpine:
    image: alpine
    command: ping nginx       # nginx 컨테이너 이름으로 바로 통신
    networks:
      - my-network

networks:
  my-network:                 # 네트워크 선언만 해도 자동 생성
```

```bash
# docker-compose 설치 (없는 환경)
sudo curl -L "https://github.com/docker/compose/releases/download/v2.6.1/docker-compose-$(uname -s)-$(uname -m)" \
  -o /usr/local/bin/docker-compose
sudo chmod +x /usr/local/bin/docker-compose
docker-compose version

# 실행
docker-compose up
docker-compose up -d          # 백그라운드 실행
docker-compose down           # 컨테이너 + 네트워크 정리
```

```
Docker Compose 사용 시:
  networks: 에 선언한 네트워크를 자동으로 생성
  서비스 이름이 컨테이너 이름 = 호스트명
  수동으로 docker network create 안 해도 됨

위 예시에서:
  alpine 컨테이너에서 ping nginx 가능
  nginx 서비스 이름이 DNS 호스트명으로 자동 등록됨
```

---

---

# host 네트워크 — 격리 없음 ⭐️

```bash
docker run -d --name host-container --network host nginx

# NetworkMode 확인
docker inspect --format '{{.HostConfig.NetworkMode}}' host-container
# host
```

```
host 네트워크:
  컨테이너가 호스트 네트워크 스택을 직접 공유
  호스트 IP = 컨테이너 IP (격리 없음)

  -p 포트 매핑 불필요 / 무시됨:
    컨테이너가 이미 호스트 포트 직접 사용
    nginx 가 80 포트 → 호스트 80 포트 그대로 열림

  장점: 네트워크 성능 최적화
  ⚠️ 주의:
    다른 서비스와 포트 충돌 가능
    Linux 에서만 완전 지원
```

---

---

# none 네트워크 — 완전 격리 ⭐️

```bash
docker run --network none --name isolated -d alpine sleep infinity

# 네트워크 인터페이스 확인
docker exec isolated ip addr
# lo (루프백) 만 보임 / eth0 없음

# 외부 접속 시도 → 실패
docker exec isolated ping -c 2 google.com
# ping: bad address 'google.com'
# Network is unreachable
```

```
none 네트워크:
  eth0 같은 네트워크 인터페이스 없음
  루프백(lo) 만 존재
  외부 인터넷 / 다른 컨테이너 모두 접근 불가
  보안상 극도의 격리가 필요한 작업에 사용
```

---

---

# 여러 네트워크에 동시 연결 ⭐️

```bash
# container2 를 두 번째 네트워크에도 추가 연결
docker network connect my-second-bridge container2

# container2 는 이제 두 네트워크에 동시 소속
```

```
통신 가능 범위:

  [network-A]               [network-B]
  container1 ←→ container2 ←→ container3
                     ↑
             두 네트워크 모두 소속
             → 브리지 역할 가능

  container1 → container2  ✅ (같은 network-A)
  container1 → container3  ❌ (다른 네트워크)
  container2 → container3  ✅ (container2 가 network-B 에도 있음)
```

---

---

#  네트워크 별칭 — 로드 밸런싱 ⭐️

```bash
# 같은 alias 로 여러 컨테이너 등록
docker run -d --network service-network \
  --network-alias myservice \
  --name service1 nginx

docker run -d --network service-network \
  --network-alias myservice \
  --name service2 nginx

# DNS 조회 → 두 IP 모두 반환
docker run --rm --network service-network appropriate/curl \
  nslookup myservice
# Address 1: 172.20.0.3 service2.service-network
# Address 2: 172.20.0.2 service1.service-network

# 라운드 로빈 확인
for i in {1..4}; do ping -c 1 myservice; done
# 172.20.0.2 / 172.20.0.3 번갈아 응답
```

```
--network-alias:
  여러 컨테이너에 동일한 DNS 이름 부여
  Docker 내장 DNS 가 자동 로드 밸런싱 (라운드 로빈)

활용:
  같은 서비스를 여러 컨테이너로 스케일 아웃
  클라이언트는 myservice 하나만 알면 됨
  실제 IP 모르고도 분산 접속 가능
```

---

---
