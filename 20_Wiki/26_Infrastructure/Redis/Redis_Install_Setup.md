---
aliases:
  - Redis 설치
  - redis-cli
  - PING
tags:
  - Redis
related:
  - "[[00_Redis_HomePage]]"
  - "[[Redis_Concept]]"
  - "[[Redis_String]]"
  - "[[Docker_Volume]]"
---

# Redis_Install_Setup — 설치 & 기본 명령어

---

---

# Docker 로 Redis 실행 ⭐️

```bash
# 기본 실행 (데이터 휘발)
docker run -d --name redis -p 6379:6379 redis

# 데이터 영속성 보장 (볼륨 마운트)
docker run -d \
  --name redis \
  -p 6379:6379 \
  -v redis-data:/data \
  redis

# 비밀번호 설정
docker run -d \
  --name redis \
  -p 6379:6379 \
  redis \
  redis-server --requirepass 비밀번호
```

```
기본 포트: 6379
-v redis-data:/data → 컨테이너 삭제해도 데이터 유지
```

---

---

#  redis-cli — 접속 ⭐️

```bash
# 실행 중인 컨테이너에 접속
docker exec -it redis redis-cli

# 비밀번호 있을 때
docker exec -it redis redis-cli -a 비밀번호

# 접속 후 프롬프트
# 127.0.0.1:6379>
```

---

---

# 기본 명령어 ⭐


## 연결 확인 — PING ⭐️

```bash
PING
# → PONG  (연결 정상)

PING "hello"
# → "hello"  (메시지 포함)
```

```
PING 의 역할:
  Redis 서버가 실제로 실행 중이고 응답하는지 확인
  헬스 체크(Health Check) 의 가장 기본적인 방법
  PONG 이 돌아오면 정상 / 응답 없으면 서버 문제

흐름:
  redis-cli 접속
    127.0.0.1:6379> PING
    PONG
    127.0.0.1:6379> quit   ← 반드시 종료

접속 전 확인:
  redis-cli 로 들어가기 전에
  Redis 서버가 실제로 실행 중인지 먼저 확인

⚠️ 실습 환경에서:
  다음 단계로 넘어가기 전에 반드시 quit 으로 redis-cli 종료
  종료 안 하면 다음 명령어가 redis-cli 안에서 실행돼 에러 발생
```

## Key-Value 기본 조작

```bash
# 값 저장
SET name "Alice"
# → OK

# 값 조회
GET name
# → "Alice"

# 키 존재 확인
EXISTS name
# → 1 (있음) / 0 (없음)

# 키 삭제
DEL name
# → 1 (삭제된 키 수)

# 모든 키 목록
KEYS *
# → 전체 키 목록 (실무에서 주의 — 많으면 느려짐)

# TTL 설정 (초 단위)
SET code "482910"
EXPIRE code 180   # 180초 후 자동 삭제

# 남은 TTL 확인
TTL code
# → 178 (남은 초) / -1 (TTL 없음) / -2 (키 없음)
```

## 종료

```bash
quit
# redis-cli 종료
```

---

---

# 자주 쓰는 명령어 한눈에

|명령어|역할|
|---|---|
|`PING`|연결 확인 → PONG|
|`SET key value`|값 저장|
|`GET key`|값 조회|
|`DEL key`|키 삭제|
|`EXISTS key`|키 존재 확인|
|`EXPIRE key 초`|TTL 설정|
|`TTL key`|남은 TTL 확인|
|`KEYS *`|전체 키 목록|
|`FLUSHDB`|현재 DB 전체 삭제 ⚠️|
|`quit`|redis-cli 종료|