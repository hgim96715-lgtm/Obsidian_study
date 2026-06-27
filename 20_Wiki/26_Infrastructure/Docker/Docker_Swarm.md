---
aliases:
  - Docker Swarm
  - Swarm
  - 컨테이너 오케스트레이션
tags:
  - Docker
related:
  - "[[00_Docker_HomePage]]"
  - "[[Docker_Compose]]"
---

# Docker_Swarm — 클러스터 배포

```txt
Docker Swarm = 여러 Docker 호스트를 하나의 클러스터로 묶어 관리
컨테이너를 여러 서버에 분산 배포 / 자동 복구 / 수평 확장
```

---

---

# Docker Compose vs Docker Swarm ⭐️

```txt
Docker Compose
  단일 서버에서 여러 컨테이너 관리
  개발 / 테스트 환경

Docker Swarm
  여러 서버(노드)를 클러스터로 묶어서 관리
  트래픽에 따라 자동 확장 / 장애 시 자동 복구
  운영 환경 / 고가용성 필요할 때
```

---

---

# 클러스터 설정 ⭐️

## Swarm 초기화

```bash
# 현재 서버를 Manager 노드로 초기화
docker swarm init

# 출력:
# Swarm initialized: current node (xxx) is now a manager.
# To add a worker to this swarm, run:
#   docker swarm join --token SWMTKN-... IP:2377
```

```txt
Manager 노드:
  클러스터 전체를 관리
  서비스 배포 / 스케일링 명령 처리

Worker 노드:
  Manager 의 지시를 받아 컨테이너 실행
  docker swarm join 으로 클러스터 합류
```

## 노드 목록 확인

```bash
docker node ls

# ID                   HOSTNAME              STATUS    AVAILABILITY
# iZrj97znp7rovie7...  my-server             Ready     Active
#                      ↑ 이게 <swarm-node의-ip-주소> 자리에 들어갈 이름
```

---

---

# 애플리케이션 배포 ⭐️

## stack deploy — Compose 파일로 배포

```bash
# docker-compose.yml 을 Swarm 에 배포
docker stack deploy -c docker-compose.yml mystack
#                                          ↑ 스택 이름
```

```txt
docker stack = Swarm 에서 여러 서비스를 묶는 단위
docker-compose.yml 그대로 사용 가능
```

## 배포 상태 확인

```bash
docker stack ls                  # 스택 목록
docker stack services mystack    # 스택 안의 서비스 목록
docker stack ps mystack          # 각 태스크(컨테이너) 실행 상태
```

## 접속 확인

```bash
# docker node ls 에서 나온 호스트명으로 접속
curl http://<swarm-node의-ip-주소>:80

# 또는 브라우저에서
http://<swarm-node의-ip-주소>:80
```

---

---

# 수평 확장 (Scale Out) ⭐️

```txt
Docker Swarm 의 핵심 장점
트래픽 증가 시 서비스 복제본 수를 늘려서 처리량 확장
```

```bash
# nginx 서비스를 5개 복제본으로 확장
docker service scale mystack_nginx=5

# 확인
docker service ls
# ID             NAME           MODE         REPLICAS   IMAGE         PORTS
# rcmdqydphe99   mystack_nginx  replicated   5/5        nginx:latest  *:80->80/tcp
#                                            ↑ 5/5 = 5개 모두 실행 중
```

```txt
REPLICAS 5/5:
  앞 숫자  현재 실행 중인 복제본 수
  뒤 숫자  목표 복제본 수
  5/5 → 정상 / 3/5 → 2개 아직 시작 중 또는 장애

서비스 이름 규칙:
  {스택이름}_{서비스이름}
  mystack_nginx = mystack 스택의 nginx 서비스
```

---

---

# 주요 명령어 한눈에

|명령어|역할|
|---|---|
|`docker swarm init`|현재 노드를 Manager 로 Swarm 초기화|
|`docker swarm join`|Worker 노드로 클러스터 합류|
|`docker node ls`|클러스터 노드 목록 확인|
|`docker stack deploy -c yml 이름`|Compose 파일로 스택 배포|
|`docker stack ls`|스택 목록|
|`docker stack services 스택명`|스택의 서비스 목록|
|`docker stack ps 스택명`|태스크 실행 상태|
|`docker service scale 서비스=N`|복제본 수 조정|
|`docker service ls`|서비스 목록 및 상태|
|`docker stack rm 스택명`|스택 삭제|