---
aliases:
  - PostgreSQL Docker
  - docker-compose postgres
  - DataGrip
tags:
  - NestJS
  - SQL
related:
  - "[[00_NestJS_Ecosystem_HomePage]]"
  - "[[NestJS_Prisma]]"
  - "[[NestJS_TypeORM]]"
  - "[[Docker_Compose]]"
  - "[[NestJS_Env_Config]]"
---
# NestJS_PostgreSQL — PostgreSQL 실행과 연결 확인

# 한 줄 요약

```
Docker Compose 로 PostgreSQL 띄우고, GUI 툴로 연결까지 확인하는 단계
Prisma 든 TypeORM 이든 ORM 선택과 무관 — 이 단계는 항상 먼저, ORM 코드는 그 다음
```

---

---

# ① Docker Compose 로 실행 ⭐️

```yaml
# docker-compose.yml
services:
  db:
    image: postgres:17-alpine
    container_name: my-project-db
    ports:
      - '5433:5432'
    environment:
      POSTGRES_USER: postgres
      POSTGRES_PASSWORD: postgres
      POSTGRES_DB: postgres
    volumes:
      - db_data:/var/lib/postgresql/data
    healthcheck:
      test: ['CMD-SHELL', 'pg_isready -U "$$POSTGRES_USER" -d "$$POSTGRES_DB"']
      interval: 10s
      timeout: 5s
      retries: 5
      start_period: 40s

volumes:
  db_data:
```

```bash
docker compose up -d      # 실행
docker compose down       # 중지
docker compose down -v    # 데이터까지 삭제 (volumes 도 제거)
```

|항목|역할|
|---|---|
|`ports: '5433:5432'`|내 컴퓨터 5433 → 컨테이너 5432 (PostgreSQL 기본 포트). 호스트 쪽 포트만 바꾸면 됨|
|`volumes: db_data:/var/...`|컨테이너를 지워도 데이터가 남게 함 (볼륨에 저장) — `down -v` 해야 진짜 삭제됨|
|`healthcheck`|컨테이너가 "떴다" 와 "DB 가 실제로 접속 가능하다" 는 다른 상태임을 구분 — 아래에서 설명|

## healthcheck — 왜 필요한가 ⭐️⭐️

```
컨테이너가 Running 상태인 것과, PostgreSQL 이 실제로 연결을 받을 준비가 된 것은 시점이 다름
(컨테이너는 떴지만 그 안의 Postgres 프로세스가 초기화 중일 수 있음)
→ healthcheck 로 "진짜 준비됐는지" 를 주기적으로 확인해서, 다른 서비스(API 등)가
  "DB가 healthy 상태일 때만 시작" 하도록 의존 관계를 걸 수 있음 (depends_on: condition: service_healthy)
```

|옵션|의미|
|---|---|
|`test`|이 명령이 0(성공)을 반환하면 healthy — `pg_isready` 는 Postgres 가 공식 제공하는 접속 확인 명령|
|`interval`|얼마나 자주 검사할지|
|`retries`|연속 몇 번 실패해야 unhealthy 로 판정할지|
|`start_period`|시작 직후 이 기간 동안의 실패는 카운트하지 않음 (초기 부팅 시간 배려)|

```
$$POSTGRES_USER 처럼 $ 를 두 번 쓰는 이유:
  docker-compose.yml 자체도 환경변수 치환을 하는 파일이라, $VAR 를 한 번만 쓰면
  compose 가 자기 파일을 파싱할 때 먼저 치환해버리려 함
  → $$ 로 escape 해서 "이건 compose 가 아니라 컨테이너 내부 셸이 해석해야 하는 변수" 라고 알려줌
```

## networks — 언제 필요한가

```
서비스가 PostgreSQL 하나뿐이면 사실 커스텀 네트워크가 꼭 필요하지는 않음(기본 네트워크로 충분)
API 서버 등 다른 서비스를 같은 compose 파일에 추가해서 "서로 이름으로 통신" 시키고 싶을 때 의미가 생김
```

```yaml
services:
  db:
    networks: [my-network]
  api:
    networks: [my-network]   # 같은 네트워크에 있으면 api 컨테이너에서 db 라는 호스트명으로 db 에 접근 가능

networks:
  my-network:
    driver: bridge
```

```
→ 자세한 멀티 서비스 구성과 서비스 간 통신은 [[Docker_Compose]] 참고
```

---

---

# ② .env 로 접속 정보 분리하기 ⭐️⭐️⭐️

```
container_name/포트/계정 정보를 compose 파일에 그대로 박아두지 않고 .env 로 분리하는 두 가지 방식이 있음
```

## 방식 A — environment: 에서 ${변수} 로 참조 (이름을 자유롭게 짓고 싶을 때)

```yaml
services:
  db:
    environment:
      POSTGRES_USER: ${DB_USER}
      POSTGRES_PASSWORD: ${DB_PASSWORD}
      POSTGRES_DB: ${DB_DATABASE}
```

```bash
# .env
DB_USER=postgres
DB_PASSWORD=postgres
DB_DATABASE=postgres
```

```
.env 의 키 이름(DB_USER 등)을 자유롭게 지을 수 있음 — compose 가 ${DB_USER} 를 .env 에서 찾아 치환
대신 PostgreSQL 이 실제로 요구하는 이름(POSTGRES_USER 등)으로 한 번 더 매핑해줘야 함
```

## 방식 B — env_file: 로 파일 전체를 그대로 주입

```yaml
services:
  db:
    env_file:
      - apps/api/.env
```

```
이 .env 안의 모든 키가 컨테이너 환경변수로 그대로 들어감 — 매핑 코드가 없음
→ 단, 이 방식이 제대로 동작하려면 .env 의 키 이름이 PostgreSQL 이 인식하는 이름과
  이미 정확히 일치해야 함: POSTGRES_USER / POSTGRES_PASSWORD / POSTGRES_DB

⚠️ 흔한 오해 정정: "env_file 쓰면 무관한 키가 많아서 충돌난다" 는 사실이 아님
   PostgreSQL 이미지는 자기가 모르는 환경변수(JWT_SECRET 등)는 그냥 무시함 — 충돌 안 함
   진짜 문제가 되는 경우는 키 이름이 안 맞을 때뿐(예: DB_USER 라고만 적혀있고 POSTGRES_USER 가 없는 경우)
   → 이 경우엔 환경변수 자체가 안 보이는 거지 "충돌" 이 나는 게 아님
```

|방식|장점|단점|
|---|---|---|
|A. `environment` + `${}`|키 이름을 프로젝트 컨벤션대로 자유롭게|매핑 줄을 한 번 더 써야 함|
|B. `env_file` 그대로|매핑 코드 없음, 한 파일로 끝|`.env` 의 키 이름이 `POSTGRES_*` 와 정확히 일치해야 함|

```
NestJS(ConfigModule)/Prisma 쪽에서 쓰는 .env 와 같은 파일을 공유하고 싶다면,
그 파일에 POSTGRES_USER 같은 이름을 처음부터 맞춰서 적어두면 B 방식을 그대로 쓸 수 있음
(.env 를 누가 어떻게 읽는지 전체 그림은 [[NestJS_Env_Config]] 참고)
```

---

---

# ③ GUI 툴로 연결 확인 — ORM 코드 작성 전에 ⭐️

```
ORM(Prisma/TypeORM) 코드를 쓰기 전에, DB 자체가 살아있고 접속 가능한지부터 GUI로 확인하는 게 순서
이 단계에서 안 되면 ORM 설정 문제가 아니라 DB/Docker/포트 문제라는 뜻
```

```
DataGrip(JetBrains) 또는 pgAdmin 등으로 New Data Source → PostgreSQL 선택

Host:     localhost
Port:     5433        ← compose 의 ports 호스트 쪽 값
Database: postgres
User:     postgres
Password: postgres

Test Connection → 연결 확인
```

|툴|특징|
|---|---|
|pgAdmin|무료, PostgreSQL 전용, 가벼움|
|DataGrip|유료(학생 무료), 여러 DB 지원, 자동완성 강력|

```bash
# GUI 없이 터미널에서 빠르게 확인하고 싶다면
docker exec -it my-project-db psql -U postgres -d postgres
```

---

---

# ④ 그 다음 — ORM 연결

```
여기까지 끝났으면 DB 자체는 준비된 상태 — 이후 ORM 코드는 선택한 쪽만 보면 됨
```

|선택|어디서 이어서|
|---|---|
|Prisma|[[NestJS_Prisma]] — schema.prisma, PrismaService, adapter-pg|
|TypeORM|[[NestJS_TypeORM]] — TypeOrmModule.forRootAsync, Entity|

## 커넥션 풀 — 개념만 짧게

```
DB 연결을 미리 여러 개 만들어두고 재사용 — 매번 새로 연결/해제하는 비용을 절약
동시 요청이 많을 때 효율적, 너무 크면 DB 자체에 부하, 너무 작으면 요청이 대기함 (보통 10~20 사이에서 조정)
구체적인 설정 문법은 ORM마다 다름 — Prisma/TypeORM 각 노트 참고
```

---

---

# 자주 만나는 에러

|증상|원인|해결|
|---|---|---|
|`ECONNREFUSED`|DB 컨테이너 안 켜짐|`docker ps` 로 Running 상태 확인, 안 떠 있으면 `docker compose up -d`|
|포트 에러|호스트 쪽 포트와 ORM 설정의 포트가 다름|compose 의 `ports` 호스트 값과 `.env`/연결 문자열의 포트 일치시키기|
|비밀번호/계정 에러|`.env` 값과 compose 설정 불일치|`environment` 또는 `env_file` 에 실제로 들어간 값을 `docker exec` 로 직접 확인|
|`env_file` 썼는데 인식 안 됨|`.env` 의 키 이름이 `POSTGRES_*` 와 다름|위 "방식 B" 참고 — 이름을 맞추거나 방식 A 로 전환|
|healthcheck 가 계속 unhealthy|`$$` 이스케이프 누락, 또는 `start_period` 가 너무 짧음|`test` 의 `$VAR` → `$$VAR` 확인|

---

---

# 한눈에

```
순서: ① Docker Compose 로 DB 실행 → ② .env 로 접속정보 분리 → ③ GUI로 연결 확인 → ④ ORM 연결(선택)
이 노트는 ORM 선택과 무관 — Prisma/TypeORM 둘 다 이 단계를 먼저 거침

env_file 은 .env 의 키 이름이 POSTGRES_* 와 일치할 때만 매핑 없이 그대로 동작
  (무관한 키가 섞여 있어도 충돌 안 남 — Postgres 가 모르는 키는 그냥 무시함)

healthcheck 의 pg_isready 로 "떴다" 와 "접속 가능하다" 를 구분
$$VAR 이스케이프는 compose 파일 자체의 변수 치환과 컨테이너 내부 셸의 치환을 구분하기 위함

ORM 연결 코드/커넥션 풀 구체 설정 → [[NestJS_Prisma]] 또는 [[NestJS_TypeORM]]
```