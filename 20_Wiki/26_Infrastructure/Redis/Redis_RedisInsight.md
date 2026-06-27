---
aliases:
  - RedisInsight
  - Redis GUI
tags:
  - Redis
related:
  - "[[00_Redis_HomePage]]"
  - "[[Redis_Docker]]"
---

# Redis_RedisInsight — GUI 관리 도구

```txt
RedisInsight = Redis 데이터를 GUI 로 확인하는 도구
DataGrip 의 Redis 버전

접속: http://localhost:5540
```

---

---

# 연결 등록 ⭐️

```txt
"Database 생성" 이 아니라
이미 실행 중인 Redis 서버를 RedisInsight 에 "연결 등록" 하는 것
```

```txt
http://localhost:5540 접속
→ Add Redis Database (또는 Add database) 클릭
→ 아래 입력

  Database alias: nest-redis  (알아보기 쉬운 이름)
  Host:           redis        ← ⚠️ localhost 아님
  Port:           6379
  Username:       비움
  Password:       비움
  Database index: 0

→ Test Connection → Add Redis Database
```

---

---

# Host 가 redis 인 이유 ⭐️

```txt
RedisInsight 도 Docker Compose 로 실행 중
→ Redis 와 RedisInsight 는 같은 Docker 네트워크 안

Docker 네트워크 안에서는:
  컨테이너 이름 = 호스트명 으로 자동 DNS 해석
  compose 의 서비스 이름이 redis → Host 에 redis 입력

localhost 를 쓰면:
  RedisInsight 컨테이너 자기 자신을 가리킴
  Redis 컨테이너와 다른 곳 → 연결 실패

정리:
  로컬(브라우저)에서 Redis 접속   → localhost:6379
  Docker 컨테이너에서 Redis 접속  → redis:6379 (서비스 이름)
```

---

---

# 아이디 / 패스워드 ⭐️

```txt
기본 설정에서는 Redis 에 인증 없음
→ Username / Password 비워두면 됨

패스워드를 설정하려면:
  docker-compose.yml 의 command 에 추가
  command: redis-server --appendonly yes --requirepass 비밀번호

  .env
  REDIS_PASSWORD=your_password

  RedisInsight 연결 시 Password 입력
  NestJS 연결 시도 password 옵션 추가 필요
```

---

---

# 연결 후 확인 ⭐️

```txt
연결된 DB 클릭 → Browser 또는 Workbench 메뉴
```

## Workbench 에서 명령어 직접 실행

```txt
SET test:hello "hello redis"
GET test:hello
```

```txt
결과:
  "hello redis"   → 연결 성공
```

## Browser 에서 확인

```txt
저장된 Key 목록 시각적으로 확인
Key 클릭 → Value / TTL / 타입 확인
```

---

---

# 주요 기능

```txt
Browser    저장된 Key / Value 목록 조회
Workbench  CLI 명령어 직접 실행
Profiler   실시간 명령어 모니터링
Analysis   메모리 사용량 / Key 통계
```