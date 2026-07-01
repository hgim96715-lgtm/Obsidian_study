---
aliases:
  - NestJS
  - PostgreSQL
  - Docker
  - DataGrip
  - Database
tags:
  - NestJS
  - SQL
related:
  - "[[00_NestJS_Ecosystem_HomePage]]"
  - "[[00_DB_HomePage]]"
  - "[[NestJS_Prisma]]"
  - "[[NestJS_StatsBucket]]"
  - "[[Deploy_CloudMVP]]"
---
# NestJS_PostgreSQL — 로컬 개발 환경 & 기초

> [!info]
>  NestJS 백엔드에서 PostgreSQL을 사용할 때의 로컬 환경 구성(Docker Compose), DataGrip 연결, timestamp 타입 선택, KST 집계 패턴까지

---

# 로컬 개발 환경 — Docker Compose ⭐️⭐️⭐️⭐️

## docker-compose.yml 기본 구성

```yaml
services:
  db:
    image: postgres:17-alpine
    container_name: app-db
    env_file:
      - apps/api/.env          # POSTGRES_USER · POSTGRES_PASSWORD · POSTGRES_DB 읽어옴
    ports:
      - '5433:5432'            # 호스트 5433 → 컨테이너 5432
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

## 포트 5433:5432를 쓰는 이유

```txt
형식: "호스트포트:컨테이너포트"
  컨테이너 내부 PostgreSQL은 항상 5432 사용
  5432:5432로 하면 로컬에 PostgreSQL이 이미 설치된 경우 포트 충돌
  → 호스트 쪽을 5433으로 올려서 회피

접속 위치에 따라 포트가 달라짐:
  로컬(호스트)에서 접속    → localhost:5433   ← DataGrip, .env
  Docker 네트워크 내부     → db:5432          ← 같은 compose의 다른 서비스
```

```txt
DATABASE_URL 예시:
  로컬 .env   → postgresql://user:pass@localhost:5433/mydb
  운영(Neon)  → postgresql://user:pass@xxxx.neon.tech/mydb?sslmode=require
```

## env_file에 필요한 변수

|변수|설명|예시|
|---|---|---|
|`POSTGRES_USER`|DB 접속 유저|`myuser`|
|`POSTGRES_PASSWORD`|비밀번호|`mypassword`|
|`POSTGRES_DB`|생성할 DB명|`mydb`|

```txt
PostgreSQL 공식 이미지는 컨테이너 첫 실행 시 위 세 변수를 읽어서 DB를 자동 생성함
volume이 이미 존재하면 무시됨 — 이미 초기화된 상태로 판단

env_file과 DATABASE_URL의 관계:
  env_file  → Docker가 PostgreSQL 컨테이너를 초기화할 때 사용
  DATABASE_URL → NestJS(Prisma)가 DB에 접속할 때 사용
  두 값이 서로 맞아야 함 (같은 user/password/dbname)
```

## healthcheck 상세

```txt
test: pg_isready -U "$POSTGRES_USER" -d "$POSTGRES_DB"
  → PostgreSQL 공식 CLI 도구 — DB가 연결 요청을 받을 준비가 됐는지 확인
  컨테이너가 올라왔다(up) ≠ PostgreSQL 프로세스가 준비됐다(ready)
  → healthcheck 없으면 NestJS가 너무 빨리 뜨면서 DB 연결 실패 가능

$$POSTGRES_USER (달러 두 개):
  Docker Compose YAML에서 $는 변수 보간 기호
  $$로 이스케이프해야 쉘 변수 치환으로 전달됨 → 런타임에 $POSTGRES_USER로 해석

start_period: 40s
  → 컨테이너 기동 후 첫 40초는 healthcheck 실패해도 unhealthy로 집계 안 함
  → PostgreSQL 기동 시간을 넉넉히 허용

interval / timeout / retries:
  10초마다 확인 · 5초 내 응답 없으면 한 번 실패 · 5번 실패하면 unhealthy
```

## 자주 쓰는 명령어

|명령어|설명|
|---|---|
|`docker compose up -d`|백그라운드로 컨테이너 시작|
|`docker compose down`|컨테이너 종료 (volume 유지, 데이터 안 날아감)|
|`docker compose down -v`|컨테이너 + volume 삭제 → DB 완전 초기화|
|`docker compose logs db`|DB 컨테이너 로그 확인|
|`docker compose ps`|컨테이너 상태 및 healthcheck 결과 확인|
|`docker compose restart db`|DB 컨테이너만 재시작|

---

# DataGrip 연결 ⭐️⭐️⭐️

## 연결 설정 순서

```txt
1. 왼쪽 Database 패널 → + 버튼 → Data Source → PostgreSQL
2. 드라이버 설치 알림 → Download 클릭 (JDBC 드라이버 자동 다운로드)
3. 연결 정보 입력:
     Host     : localhost
     Port     : 5433          ← docker-compose 호스트 포트
     Database : (POSTGRES_DB 값)
     User     : (POSTGRES_USER 값)
     Password : (POSTGRES_PASSWORD 값)
4. Test Connection → 초록 체크 확인 → OK
```

```txt
스키마가 안 보일 때:
  연결 후 Schemas 탭에서 보고 싶은 스키마 체크 (기본 public)
  또는 Database 패널에서 해당 연결 우클릭 → Refresh

Neon(운영 DB) 연결 시:
  Host: xxxx.neon.tech, Port: 5432
  SSL: require (Neon은 SSL 필수)
  DataGrip → Advanced 탭 → sslmode=require
```

## 표시 시간대 주의 (UTC vs KST) ⭐️⭐️⭐️

```txt
증상: 7/1 오전 10:00 KST에 삽입했는데 DataGrip에서 7/1 01:00으로 보임
원인: DataGrip/JVM이 UTC 기준으로 timestamptz를 표시하기 때문
      → 저장값 자체는 올바름 (UTC instant로 정확히 저장됨)

DataGrip 설정으로 KST 표시:
  File → Settings → Database → Data Views → Timezone → Asia/Seoul

쿼리로 직접 KST 확인:
  SELECT created_at AT TIME ZONE 'Asia/Seoul' FROM "User" LIMIT 5;

타입이 timestamp(timezone 없음)일 때는 더 혼란스러움:
  timezone 정보 없이 naive하게 저장 → 세션 설정에 따라 해석이 달라짐
  → timestamptz를 써야 이 문제가 없어짐 (아래 섹션 참고)
```

---

# timestamp vs timestamptz ⭐️⭐️⭐️⭐

>️각 타입의 상세 비교(JSONB · UUID · ARRAY · ENUM 포함) 
>→ [[PG_Types]]

## PostgreSQL 타입 차이

|
|`timestamp`|`timestamptz`|
|---|---|---|
|정식 명칭|`timestamp without time zone`|`timestamp with time zone`|
|저장 방식|naive — TZ 정보 없이 "날짜+시간 문자열"처럼 저장|UTC epoch(ms)로 저장|
|세션 TZ 영향|받음 — 환경마다 해석이 달라질 수 있음|받지 않음 — 항상 UTC로 변환|
|꺼낼 때|저장한 그대로 반환|클라이언트 TZ 설정에 맞게 변환|
|일관성|위험 — 서버/DB/앱 TZ가 다르면 값이 달라짐|안전 — 어디서 넣어도 같은 instant|
|Prisma|`DateTime` (기본)|`DateTime @db.Timestamptz(3)`|

```txt
timestamptz 내부 동작:
  INSERT할 때: 현재 세션 TZ 기준으로 받은 값을 UTC로 변환해서 저장
  SELECT할 때: UTC 저장값을 세션 TZ에 맞게 변환해서 반환

  → 어떤 TZ 환경에서 넣어도 동일한 UTC instant로 저장
  → KST 서버에서 넣든 UTC 서버에서 넣든 같은 값

precision (3):
  소수점 이하 자릿수 = 밀리초 단위 (.000)
  생략하면 PostgreSQL 기본값 6 (마이크로초) — 실용적으로 3이면 충분
  timestamp(3) / timestamptz(3) 모두 동일하게 적용
```

## Prisma 스키마 적용

```prisma
model User {
  createdAt    DateTime  @default(now()) @db.Timestamptz(3)
  updatedAt    DateTime  @updatedAt      @db.Timestamptz(3)
  lastActiveAt DateTime?                 @db.Timestamptz(3)
}
```

```txt
@default(now())  → DB 서버 시각(UTC)으로 자동 삽입
@updatedAt       → Prisma가 UPDATE 시 자동으로 현재 시각 세팅
lastActiveAt     → nullable: 한 번도 활동 안 한 유저는 null

Prisma DateTime만 쓰면 기본이 timestamp(3) → 반드시 @db.Timestamptz(3) 명시
([[NestJS_Prisma]] 참고 — DateTime 필드 섹션)
```

## 기존 timestamp → timestamptz 마이그레이션

```sql
-- schema.prisma만 고쳐서는 DB 타입이 바뀌지 않음 → migrate 필수
ALTER TABLE "User"
  ALTER COLUMN "createdAt" TYPE TIMESTAMPTZ(3)
  USING "createdAt" AT TIME ZONE 'UTC';
```

```txt
USING "createdAt" AT TIME ZONE 'UTC':
  기존 row를 "UTC 기준으로 넣었다"고 가정하고 타입을 변환
  만약 KST로 저장된 데이터가 섞여 있다면 'Asia/Seoul'로 맞춰야 함

적용 순서:
  1. schema.prisma 수정 (@db.Timestamptz(3) 추가)
  2. prisma migrate dev (로컬) — SQL 생성 + 실행
  3. prisma migrate deploy (운영/Neon)
  로컬과 운영 모두 migrate 실행해야 반영됨
```

---

# KST 기준 집계 쿼리 ⭐️⭐️⭐️

```sql
-- timestamptz 컬럼을 KST 기준 날짜별 집계
SELECT
  DATE(created_at AT TIME ZONE 'Asia/Seoul') AS date_kst,
  COUNT(*)                                    AS cnt
FROM "User"
GROUP BY date_kst
ORDER BY date_kst;
```

```sql
-- 월별 집계
SELECT
  DATE_TRUNC('month', created_at AT TIME ZONE 'Asia/Seoul') AS month_kst,
  COUNT(*) AS cnt
FROM "User"
GROUP BY month_kst
ORDER BY month_kst;
```

```txt
AT TIME ZONE 'Asia/Seoul':
  UTC로 저장된 timestamptz를 KST로 변환한 뒤 날짜/월 추출
  → 한국 서비스에서 "오늘 가입자 수", "이번 달 활동량" 등을 정확히 집계할 때 필수

저장 타입(timestamptz)과 집계 TZ는 별개:
  저장은 항상 UTC
  집계 기준 TZ는 쿼리에서 AT TIME ZONE으로 명시적으로 지정

빈 날짜 구간(0건인 날)이 누락되는 문제:
  GROUP BY만으로는 데이터 없는 날은 결과에 안 나옴
  → generate_series + LEFT JOIN 또는 애플리케이션 레벨에서 버킷 채우기
  ([[NestJS_StatsBucket]] 참고)
```

---

# 한눈에

```txt
Docker Compose:
  image: postgres:17-alpine
  포트: 5433:5432 (로컬 충돌 회피 — 호스트:컨테이너)
  env_file: POSTGRES_USER / POSTGRES_PASSWORD / POSTGRES_DB
  volume: down해도 데이터 유지, down -v로 초기화
  healthcheck: pg_isready — 컨테이너 up ≠ DB ready
  $$: YAML에서 $ 이스케이프 → 런타임에 $POSTGRES_USER로 해석

DataGrip:
  localhost:5433으로 연결 (로컬)
  시간 표시가 UTC처럼 보이는 건 DataGrip 설정 — 저장값은 올바름
  SELECT col AT TIME ZONE 'Asia/Seoul'로 KST 확인

timestamp vs timestamptz:
  timestamp  → naive, TZ 영향 받음 → 쓰지 말 것
  timestamptz → UTC instant, 항상 일관성 → 채택
  Prisma: @db.Timestamptz(3)
  기존 마이그레이션: ALTER COLUMN ... TYPE TIMESTAMPTZ USING ... AT TIME ZONE 'UTC'
  schema.prisma 수정만으로는 부족 → migrate deploy 필수
  PostgreSQL 타입 전체(JSONB · UUID · ARRAY · ENUM) → [[PG_Types]]

KST 집계:
  DATE(col AT TIME ZONE 'Asia/Seoul')
  DATE_TRUNC('month', col AT TIME ZONE 'Asia/Seoul')
  빈 구간 채우기는 NestJS_StatsBucket 참고
```
