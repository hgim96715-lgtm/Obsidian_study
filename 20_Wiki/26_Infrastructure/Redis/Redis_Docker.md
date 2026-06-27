---
aliases:
  - Redis Docker
  - Docker Redis
tags:
  - Redis
  - Docker
related:
  - "[[00_Redis_HomePage]]"
  - "[[00_Docker_HomePage]]"
  - "[[NestJS_Queue]]"
  - "[[Redis_RedisInsight]]"
---

# Redis_Docker — Docker 로 Redis 실행

```
Docker Compose 로 Redis + RedisInsight 를 한번에 실행
Redis       = 실제 데이터 저장소
RedisInsight = Redis GUI 관리 도구
```

---

---

# Docker Compose 설정 ⭐️

```yaml
redis:
  image: redis:7-alpine
  container_name: nest-redis
  restart: unless-stopped
  ports:
    - '${REDIS_PORT:-6379}:6379'
  volumes:
    - redis_data:/data
  command: redis-server --appendonly yes
  healthcheck:
    test: ['CMD', 'redis-cli', 'ping']
    interval: 10s
    timeout: 5s
    retries: 5

redisinsight:
  image: redis/redisinsight:latest
  container_name: nest-redisinsight
  restart: unless-stopped
  ports:
    - '${REDIS_INSIGHT_PORT:-5540}:5540'
  volumes:
    - redisinsight_data:/data
  depends_on:
    redis:
      condition: service_healthy

volumes:
  postgres_data:
  redis_data:
  redisinsight_data:
```

```
각 옵션 설명:

image: redis:7-alpine
  Redis 7 버전 / alpine = 경량 Linux 기반 → 용량 작음

restart: unless-stopped
  Docker 재시작 시 컨테이너 자동 재시작
  수동으로 stop 하면 재시작 안 함

command: redis-server --appendonly yes
  AOF(Append Only File) 활성화
  → 서버 재시작해도 데이터 유지 (영속성)

healthcheck:
  redis-cli ping 으로 Redis 준비 완료 여부 확인
  → redisinsight 는 Redis 가 healthy 상태일 때만 시작

depends_on: condition: service_healthy
  Redis healthcheck 통과 후에 RedisInsight 시작
  순서 보장
```

---

---

# .env 설정

```properties
# Redis
REDIS_HOST=localhost
REDIS_PORT=6379
REDIS_INSIGHT_PORT=5540
```

```
REDIS_PORT:-6379
  .env 에 REDIS_PORT 없으면 기본값 6379 사용

NestJS 에서 연결:
  host: REDIS_HOST (localhost)
  port: REDIS_PORT (6379)
```

---

---

# RedisInsight

```bash
Redis 데이터를 GUI 로 확인하는 도구
http://localhost:5540 접속

# → 연결 등록 및 사용법: [[Redis_RedisInsight]] 참고
```

---

---

# 실행

```bash
# 실행
docker compose up -d

# Redis 컨테이너만 실행
docker compose up -d redis redisinsight

# 로그 확인
docker logs nest-redis
docker logs nest-redisinsight

# 중지
docker compose down
```