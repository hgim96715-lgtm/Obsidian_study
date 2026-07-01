---
aliases:
  - Docker
  - services
  - DockerCompose
tags:
  - Docker
  - DevOps
related:
  - "[[00_DevOps_HomePage]]"
  - "[[NestJS_PostgreSQL]]"
  - "[[Docker_Dockerfile]]"
---


# Docker_Compose — 멀티 컨테이너 환경 구성

> [!info] 
> `docker-compose.yml` 하나로 여러 컨테이너(앱 서버, DB, 캐시 등)를 한 번에 정의하고 실행
>  로컬 개발 환경을 팀원 누구나 `docker compose up` 한 줄로 동일하게 재현할 수 있게 해주는 도구

---

# 전체 구조 한눈에

```yaml
services:       # 실행할 컨테이너 목록 (필수)
  서비스명:
    image:      # 사용할 이미지 (또는 build:)
    build:      # Dockerfile로 직접 빌드
    ports:      # 포트 매핑 (호스트:컨테이너)
    volumes:    # 파일/디렉토리 마운트
    environment:# 환경 변수 직접 작성
    env_file:   # .env 파일에서 환경 변수 읽기
    depends_on: # 다른 서비스가 먼저 실행되어야 함
    networks:   # 참여할 네트워크
    healthcheck:# 컨테이너 준비 상태 확인
    restart:    # 재시작 정책

volumes:        # named volume 정의 (선택)
  볼륨명:

networks:       # 네트워크 정의 (선택)
  네트워크명:
    driver: bridge
```

---

# services ⭐️⭐️⭐️⭐️

## image vs build

```yaml
services:
  db:
    image: postgres:17-alpine      # Docker Hub에서 이미지 그대로 사용

  api:
    build:
      context: .                   # Dockerfile이 있는 디렉토리
      dockerfile: Dockerfile       # 파일명 (기본값: Dockerfile)
```

```txt
image: 이미 만들어진 이미지를 그대로 쓸 때
build: 내 Dockerfile로 이미지를 직접 빌드할 때

로컬 개발:
  DB, Redis 같은 서드파티 → image 사용
  내가 만든 앱(API, Worker) → build 사용
```

## ports — 포트 매핑

```yaml
ports:
  - '3000:3000'   # 호스트:컨테이너
  - '5433:5432'   # 호스트 5433 → 컨테이너 5432 (로컬 충돌 회피)
```

```txt
형식: "호스트포트:컨테이너포트"

컨테이너 내부는 항상 기본 포트 사용 (PostgreSQL = 5432)
호스트 쪽 포트만 바꿔서 로컬에 깔린 DB와 충돌 회피
→ localhost:5433으로 접속 (DataGrip, .env에서)

Docker 네트워크 내부에서는:
  서비스명:컨테이너포트 로 접근 (예: db:5432)
  포트 매핑과 무관 — 내부끼리는 기본 포트로 통신
```

## restart — 재시작 정책

|값|동작|
|---|---|
|`no`|재시작 안 함 (기본값)|
|`always`|항상 재시작 (수동 stop 제외)|
|`on-failure`|오류 종료 시에만 재시작|
|`unless-stopped`|수동으로 stop하지 않는 한 항상 재시작|

```yaml
services:
  api:
    restart: unless-stopped   # 운영 환경에서 권장
```

---

# environment vs env_file ⭐️⭐️⭐️

```yaml
services:
  api:
    # 방법 1: 직접 작성 (값이 docker-compose.yml에 노출됨)
    environment:
      NODE_ENV: production
      PORT: 3000

    # 방법 2: .env 파일에서 읽기 (값이 파일에 분리됨)
    env_file:
      - .env
      - apps/api/.env    # 모노레포에서 특정 앱의 .env
```

```txt
env_file 사용 이유:
  비밀 값(DB 비밀번호, JWT Secret)을 docker-compose.yml에 하드코딩하지 않아도 됨
  .env 파일을 .gitignore에 추가 → Git에 올라가지 않음
  팀원마다 로컬 .env 파일을 따로 관리 가능

env_file에서 읽은 변수는 컨테이너의 환경 변수로 주입됨
PostgreSQL 공식 이미지의 경우 POSTGRES_USER · POSTGRES_PASSWORD · POSTGRES_DB를
env_file에서 읽어서 DB를 자동 초기화함 ([[NestJS_PostgreSQL]] 참고)
```

---

# volumes ⭐️⭐️⭐️

## Named Volume vs Bind Mount

```yaml
services:
  db:
    volumes:
      - db_data:/var/lib/postgresql/data   # Named Volume ← DB 데이터 보관

  api:
    volumes:
      - ./apps/api:/app/apps/api           # Bind Mount ← 로컬 코드 실시간 반영

volumes:
  db_data:    # Named Volume 정의 (내용 비워도 됨)
```

|종류|형식|용도|
|---|---|---|
|Named Volume|`볼륨명:/컨테이너/경로`|DB 데이터 영속 보관 — Docker가 관리|
|Bind Mount|`./로컬경로:/컨테이너/경로`|로컬 코드 → 컨테이너에 실시간 반영|

```txt
Named Volume:
  컨테이너가 삭제되어도 데이터 유지 (docker compose down)
  docker compose down -v 해야 volume도 삭제됨
  위치: Docker 내부 관리 영역 (/var/lib/docker/volumes/)

Bind Mount:
  호스트의 실제 파일을 컨테이너에 마운트
  코드를 수정하면 컨테이너 재시작 없이 즉시 반영 (핫리로드 조합)
  경로가 존재해야 함 — 없으면 에러
```

---

# networks ⭐️⭐️

```yaml
services:
  api:
    networks:
      - app-network

  db:
    networks:
      - app-network

networks:
  app-network:
    driver: bridge
```

```txt
왜 network를 명시하는가:
  같은 compose 파일 안의 서비스는 기본적으로 같은 네트워크에 있어서
  명시하지 않아도 서비스명으로 통신 가능 (db:5432 처럼)

  명시하는 이유:
    여러 compose 파일의 네트워크를 연결할 때
    특정 서비스만 격리하고 싶을 때
    네트워크 이름을 의미 있게 관리하고 싶을 때

bridge:
  기본 드라이버 — 같은 호스트 내 컨테이너끼리 통신하는 가상 네트워크
  host: 컨테이너가 호스트 네트워크를 직접 사용 (포트 매핑 없이 접근, 보안 약함)
```

---

# healthcheck ⭐️⭐️⭐️⭐️

```yaml
services:
  db:
    image: postgres:17-alpine
    healthcheck:
      test: ['CMD-SHELL', 'pg_isready -U "$$POSTGRES_USER" -d "$$POSTGRES_DB"']
      interval: 10s       # 10초마다 확인
      timeout: 5s         # 5초 내 응답 없으면 실패
      retries: 5          # 5번 실패하면 unhealthy
      start_period: 40s   # 처음 40초는 실패해도 unhealthy로 안 봄

  api:
    depends_on:
      db:
        condition: service_healthy   # db가 healthy 상태일 때만 api 시작
```

```txt
컨테이너 up ≠ 서비스 준비됨:
  PostgreSQL 컨테이너가 떴다고 바로 연결 가능한 게 아님
  DB 프로세스 기동까지 수십 초 걸릴 수 있음
  → healthcheck 없이 api가 먼저 뜨면 DB 연결 실패 에러 발생

pg_isready:
  PostgreSQL 공식 CLI 도구 — 연결 요청을 받을 준비가 됐는지 확인
  -U 유저명 / -d DB명 옵션으로 특정 DB 확인

$$POSTGRES_USER (달러 두 개):
  Docker Compose YAML에서 $는 변수 보간 기호
  $$로 이스케이프해야 런타임에 쉘 변수 $POSTGRES_USER로 전달됨
  ([[NestJS_PostgreSQL]] "healthcheck" 섹션 참고)

start_period:
  DB 기동 초기에는 당연히 healthcheck 실패 → unhealthy 오판 방지
  40s 동안은 실패해도 카운트 안 함
```

---

# depends_on ⭐️⭐️⭐️

```yaml
services:
  api:
    depends_on:
      db:
        condition: service_healthy   # healthcheck 통과 후 시작
      redis:
        condition: service_started   # 컨테이너가 시작되기만 하면 됨 (기본값)
```

|condition|의미|
|---|---|
|`service_started`|컨테이너가 시작만 되면 됨 (healthcheck 무시)|
|`service_healthy`|healthcheck 통과 후 시작|
|`service_completed_successfully`|이전 서비스가 종료(exit 0)해야 시작|

```txt
service_healthy 사용 조건:
  의존하는 서비스에 healthcheck가 정의되어 있어야 함
  없으면 "service has no healthcheck" 에러
```

---

# 자주 쓰는 명령어 ⭐️⭐️⭐️

|명령어|설명|
|---|---|
|`docker compose up`|서비스 시작 (포그라운드)|
|`docker compose up -d`|백그라운드로 시작|
|`docker compose up --build`|이미지 재빌드 후 시작|
|`docker compose down`|서비스 종료 (volume 유지)|
|`docker compose down -v`|서비스 종료 + volume 삭제|
|`docker compose logs 서비스명`|특정 서비스 로그|
|`docker compose logs -f`|로그 실시간 follow|
|`docker compose ps`|실행 중인 서비스 목록 + 상태|
|`docker compose restart 서비스명`|특정 서비스 재시작|
|`docker compose exec 서비스명 bash`|실행 중인 컨테이너 안으로 진입|
|`docker compose build`|이미지 빌드만 (실행 안 함)|

```txt
--build 언제 쓰나:
  Dockerfile이나 소스코드가 바뀌었는데 docker compose up만 하면
  캐시된 이전 이미지를 그대로 씀 → 변경사항 미반영
  → up --build로 강제 재빌드

volume 삭제 (-v) 시 주의:
  DB 데이터도 날아감 → 개발 중 초기화하고 싶을 때만 사용
```

---

# 전체 예시 — API + DB 로컬 개발 환경

```yaml
services:
  api:
    build:
      context: .
      dockerfile: apps/api/Dockerfile
    ports:
      - '3000:3000'
    env_file:
      - apps/api/.env
    volumes:
      - ./apps/api/src:/app/apps/api/src   # 소스 코드 핫리로드
    depends_on:
      db:
        condition: service_healthy
    networks:
      - app-network
    restart: unless-stopped

  db:
    image: postgres:17-alpine
    env_file:
      - apps/api/.env
    ports:
      - '5433:5432'
    volumes:
      - db_data:/var/lib/postgresql/data
    healthcheck:
      test: ['CMD-SHELL', 'pg_isready -U "$$POSTGRES_USER" -d "$$POSTGRES_DB"']
      interval: 10s
      timeout: 5s
      retries: 5
      start_period: 40s
    networks:
      - app-network

volumes:
  db_data:

networks:
  app-network:
    driver: bridge
```

---

# 한눈에

```txt
구조:
  services: 컨테이너 목록 (필수)
  volumes:  Named Volume 정의 (선택)
  networks: 네트워크 정의 (선택)

ports: "호스트:컨테이너" — 로컬 충돌 시 호스트 포트만 바꾸기
  내부 통신은 서비스명:컨테이너포트 (포트 매핑과 무관)

env_file vs environment:
  env_file → .env 파일에서 읽기 (비밀값 분리, 권장)
  environment → 직접 작성 (비밀값 노출 주의)

volumes:
  Named Volume (볼륨명:경로) → DB 데이터 영속 보관
  Bind Mount (./로컬:경로)   → 코드 핫리로드

healthcheck:
  컨테이너 up ≠ 서비스 준비됨 → pg_isready로 실제 준비 상태 확인
  $$POSTGRES_USER → YAML의 $ 이스케이프 (런타임에 $POSTGRES_USER로 해석)
  start_period → 기동 초기 실패 허용 시간

depends_on:
  service_healthy → healthcheck 통과 후 다음 서비스 시작 (권장)
  service_started → 시작만 하면 됨 (healthcheck 없을 때)

명령어:
  up -d        백그라운드 시작
  up --build   소스 변경 시 강제 재빌드
  down -v      컨테이너 + 볼륨 삭제 (DB 초기화)
  logs -f      실시간 로그
  exec 서비스 bash  컨테이너 진입
```