
```
인메모리 데이터 저장소
캐싱 / 세션 / 메시지 큐 / Rate Limiting
```

---

---

## Level 0. 개념 잡기

```
Redis 가 뭔지, 왜 쓰는지
```

| 노트                      | 핵심 개념                                            |
| ----------------------- | ------------------------------------------------ |
| [[Redis_Concept]]       | Redis 란 / 인메모리 / Key-Value / 언제 쓰나 / DB vs Redis |
| [[Redis_Install_Setup]] | Docker 설치 / redis-cli / ping / 기본 명령어            |

---

---

## Level 1. 데이터 타입

```
Redis 는 String 만 있는 게 아니다
```

|노트|핵심 개념|
|---|---|
|[[Redis_String]]|SET / GET / INCR / DECR / EXPIRE / TTL / 카운터 패턴|
|[[Redis_Hash]]|HSET / HGET / HGETALL / 객체 저장 패턴|
|[[Redis_List]]|LPUSH / RPUSH / LPOP / RPOP / LRANGE / 큐·스택 패턴|
|[[Redis_Set]]|SADD / SMEMBERS / SISMEMBER / 중복 제거|
|[[Redis_Sorted_Set]] ⭐|ZADD / ZRANGE / ZRANK / ZSCORE / 랭킹 시스템 패턴|

---

---

## Level 2. 핵심 기능

|노트|핵심 개념|
|---|---|
|[[Redis_Expiry]] ⭐|EXPIRE / TTL / EXPIREAT / 자동 만료 / 세션 만료 패턴|
|[[Redis_Keys]] ⭐|키 네이밍 : 구분자 관례 / KEYS 패턴 / SCAN / DEL / EXISTS|
|[[Redis_Pub_Sub]]|PUBLISH / SUBSCRIBE / 채널 / 실시간 알림|
|[[Redis_Transaction]]|MULTI / EXEC / DISCARD / 원자성|

---

---

## Level 3. NestJS 연동

|노트|핵심 개념|
|---|---|
|[[NestJS_Caching]] ⭐|cache-manager-redis-store / CACHE_MANAGER / TTL=ms / 토큰 블랙리스트|
|[[Redis_NestJS_Direct]]|ioredis / 직접 연결 / 커스텀 Redis 클라이언트|

---

---

## Level 4. 실전 패턴

|노트|핵심 개념|
|---|---|
|[[Redis_Pattern_Cache]]|Look-Aside 패턴 / Write-Through / TTL 전략|
|[[Redis_Pattern_Session]]|세션 저장 / JWT 블랙리스트 / 토큰 캐싱|
|[[Redis_Pattern_RateLimit]]|요청 횟수 카운팅 / 슬라이딩 윈도우 / 고정 윈도우|
|[[Redis_Pattern_Ranking]]|Sorted Set 랭킹 / 실시간 순위 업데이트|

---

---

## Level 5. 운영

|노트|핵심 개념|
|---|---|
|[[Redis_Persistence]]|RDB / AOF / 영속성 옵션|
|[[Redis_Docker]] ⭐|Docker Compose / Redis + RedisInsight / AOF / healthcheck / .env|
|[[Redis_RedisInsight]] ⭐|연결 등록 / Host=redis 이유 / 패스워드 설정 / Browser / Workbench|
|[[Redis_CLI]] ⭐|redis-cli 명령어 / monitor / info / FLUSHDB|
