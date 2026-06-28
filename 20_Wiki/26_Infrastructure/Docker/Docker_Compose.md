---
aliases:
  - Docker Compose
  - docker-compose
  - compose
related:
  - "[[00_Docker_HomePage]]"
  - "[[Docker_Dockerfile]]"
  - "[[Docker_Network]]"
  - "[[Docker_Volume]]"
  - "[[NestJS_Prisma]]"
tags:
  - Docker
---
# Docker_Compose — Docker Compose

# 한 줄 요약

```txt
여러 컨테이너를 하나의 파일로 정의하고 한번에 실행
docker-compose.yml = 컨테이너 설계도
```

---

---

# Docker Compose 란 ⭐️

```txt
여러 컨테이너를 매번 docker run 으로 하나씩 실행하면:
  docker run -d --name db -e POSTGRES_PASSWORD=... postgres
  docker run -d --name app --network ... -p 8080:8080 my-app
  → 명령어 길고 복잡 / 실수 가능

Docker Compose:
  docker-compose.yml 에 모든 컨테이너 설정을 정의
  docker compose up -d 한 줄로 전부 실행
  docker compose down 한 줄로 전부 종료
```

---

---

# 설치

```bash
# 최신 Docker Desktop / Docker Engine 에는 내장됨
docker compose version   # 공백 (플러그인 방식, 권장)
docker-compose version   # 하이픈 (독립형 구버전 방식)
```

---

---

# docker-compose.yml 기본 구조 ⭐️

```yaml
services:             # 컨테이너 목록
  서비스명:
    image: 이미지명
    container_name: 컨테이너명
    ports:
      - "호스트포트:컨테이너포트"
    environment:
      - KEY=VALUE
    volumes:
      - 볼륨명:/컨테이너경로
    networks:
      - 네트워크명
    depends_on:
      - 다른서비스명

volumes:              # named volume 선언
  볼륨명:

networks:             # 네트워크 선언
  네트워크명:
```

---

---

# 포트 매핑 — "호스트포트:컨테이너포트" 의 의미 ⭐️⭐️

```txt
"5433:5432" 처럼 콜론으로 나뉜 두 숫자는 서로 완전히 다른 것을 가리킴

  호스트포트(왼쪽)    내 컴퓨터(Mac)에서 이 컨테이너에 접속할 때 쓰는 포트
                      → 자유롭게 바꿀 수 있음 (다른 프로그램과 안 겹치게)
  컨테이너포트(오른쪽)  컨테이너 "안에서" 그 프로그램이 실제로 듣고 있는 포트
                      → Postgres 는 항상 내부적으로 5432 를 씀 (이건 안 바뀜)

"5433:5432" 의 뜻:
  컨테이너 안의 Postgres 는 평소처럼 5432 에서 그대로 동작하고 있음
  내 컴퓨터에서는 5433 으로 접속하면 그 5432 로 연결해주겠다는 "통로"만 만든 것
  → 컨테이너 내부 설정은 안 바꾸고, 바깥에서 접속하는 입구 번호만 바꾼 것
```

```txt
이게 왜 유용한가:
  내 컴퓨터에 이미 같은 포트(5432)를 쓰는 다른 프로그램이 있어도
  Postgres 자체 설정은 그대로 두고, 호스트 쪽 입구만 다른 번호로 열어서 충돌을 피할 수 있음
```

---

---

# ⚠️ 실전 트러블슈팅 — 로컬 Postgres 와 충돌해서 엉뚱한 DB 에 연결됨 (P1010) ⭐️⭐️⭐️

## 증상

```txt
Prisma 에러: P1010 — User 'music_user' was denied access on the database
```

## 원인 파악

```txt
Mac 에 로컬 Postgres(Postgres.app 등) 가 이미 떠 있어서 localhost:5432 를 점유 중이었음
Docker 의 Postgres 컨테이너도 5432 로 접속하게 설정돼 있었음
→ Prisma 가 localhost:5432 로 연결했는데, 그게 Docker 컨테이너가 아니라
  "이미 그 포트를 쓰고 있던 Mac 의 로컬 Postgres" 로 가버림
→ 거기엔 music_user 라는 계정이 없으니 "접근 거부"(P1010) 가 뜸

핵심: 네트워크 연결 자체는 성공했음 (그래서 "연결 안 됨" 에러가 아니라
      "계정이 없다" 는 에러가 뜬 것) — 그냥 "다른" Postgres 에 연결된 것뿐
```

```txt
localhost:5432  → Mac 의 로컬 Postgres (music_user 없음)  ← Prisma 가 실수로 여기로 감
Docker DB        → 5433 으로 옮겨야 호스트에서 명확하게 접근 가능
```

## 해결 — Docker 쪽 호스트 포트를 다른 번호로

```yaml
# docker-compose.yml
services:
  db:
    image: postgres:17-alpine
    ports:
      - "5433:5432"   # ← 호스트 쪽만 5433 으로 (컨테이너 내부는 그대로 5432)
```

```properties
# apps/api/.env — Prisma 가 접속할 주소도 5433 으로 맞춰줌
DATABASE_URL=postgresql://music_user:music_password@localhost:5433/music_community_db?schema=public&sslmode=disable
```

```txt
이 사고에서 배울 점:
  포트 충돌은 항상 "에러" 로 친절하게 알려주는 게 아니라
  "전혀 다른 서버에 조용히 연결되는" 형태로 나타날 수도 있음
  → 연결은 됐는데 데이터가 이상하거나 계정 에러가 나면
    "내가 진짜 의도한 서버에 붙어있는 게 맞나?" 를 의심해볼 것

sslmode=disable 인 이유:
  로컬 Docker Postgres 는 SSL 인증서가 기본으로 설정돼 있지 않음
  → SSL 을 요구(require)하면 오히려 연결 실패
  → 로컬 개발 환경(localhost) 에서는 disable 이 맞고
    Neon 같은 클라우드 DB 는 반대로 sslmode=require 가 필요함 (암호화 필수)
```

---

---

# env_file — 컨테이너 환경변수를 앱의 .env 와 공유 ⭐️

```yaml
services:
  db:
    image: postgres:17-alpine
    env_file:
      - apps/api/.env   # ← NestJS 앱이 쓰는 .env 를 그대로 Postgres 컨테이너에도 전달
```

```txt
Postgres 공식 이미지는 처음 뜰 때 다음 환경변수를 보고 초기 계정/DB 를 만듦:
  POSTGRES_USER / POSTGRES_PASSWORD / POSTGRES_DB

apps/api/.env 를 그대로 가리키는 이유:
  DATABASE_URL 안의 music_user / music_password / music_community_db 값과
  Postgres 컨테이너가 실제로 만드는 계정/DB 이름이 "항상 정확히 같은 값" 이도록 보장
  → 두 군데(.env, docker-compose.yml) 에 같은 값을 따로 적어두면
    한쪽만 바꾸고 다른 쪽을 깜빡해서 또 P1010 같은 에러가 날 수 있음
  → 한 파일만 보고 가도록 묶어두는 것
```

---

---

# healthcheck — "떠 있다" 와 "준비됐다" 의 차이 ⭐️⭐️

```yaml
services:
  db:
    image: postgres:17-alpine
    healthcheck:
      test: ["CMD-SHELL", 'pg_isready -U "$$POSTGRES_USER" -d "$$POSTGRES_DB"']
      interval: 10s
      timeout: 5s
      retries: 5
      start_period: 40s
```

```txt
depends_on 만으로는 부족한 이유 (이미 알고 있던 내용):
  depends_on 은 "컨테이너가 시작은 됐다" 만 보장함
  Postgres 는 컨테이너가 뜬 직후에도 내부 초기화(데이터 디렉토리 생성 등) 가
  몇 초~몇십 초 더 걸릴 수 있어서, "시작됨" ≠ "연결 받을 준비 끝남"

healthcheck 가 채워주는 부분:
  pg_isready  Postgres 가 기본 제공하는 CLI — "지금 연결을 받을 준비가 됐는지" 확인하는 명령
  interval    몇 초마다 이 체크를 반복할지 (여기선 10초마다)
  timeout     한 번 체크할 때 몇 초까지 기다릴지
  retries     몇 번 연속 실패해야 "unhealthy" 로 판정할지
  start_period 컨테이너가 막 시작된 직후의 유예 기간 — 이 동안의 실패는 retries 에 안 셈
              (초기 부팅 중엔 당연히 몇 번 실패할 수 있으니 봐주는 기간)

  → 이렇게 만들어둔 healthcheck 를 다른 서비스의
    depends_on: { condition: service_healthy } 와 같이 쓰면
    "Postgres 가 진짜 준비된 다음에" 앱 컨테이너를 시작시킬 수 있음
```

## ⚠️ `$$` 두 번 쓰는 이유 — 변수 이스케이프

```txt
test: ["CMD-SHELL", 'pg_isready -U "$$POSTGRES_USER" -d "$$POSTGRES_DB"']
```

```txt
docker-compose.yml 자체도 $VAR / ${VAR} 문법으로 변수를 치환하는 기능이 있음
→ $POSTGRES_USER 라고만 쓰면 compose 가 "이 파일을 읽는 시점에" 자기가 먼저 치환하려고 시도함
  (이 경우 .env 변수가 yaml 파싱 시점엔 의도와 다르게 비거나 엉뚱하게 해석될 수 있음)

$$POSTGRES_USER 처럼 두 번 쓰면:
  compose 에게 "이건 너가 치환할 거 아니고, 그냥 글자 $ 야" 라고 알려주는 것
  → compose 는 $$ 를 글자 그대로의 $ 하나로 바꿔서 컨테이너 안으로 그대로 전달
  → 그 결과 컨테이너 "내부" 셸이 실행될 때 $POSTGRES_USER 가 진짜로 해석됨
    (컨테이너 안에는 POSTGRES_USER 환경변수가 실제로 있으니 거기서 정상적으로 치환됨)

한 줄 정리: $$ = "compose 야, 너는 건드리지 말고 컨테이너한테 그대로 넘겨"
```

---

---

# -f — 파일 위치 직접 지정 ⭐️

```bash
# 기본: 현재 폴더에서 아래 파일만 자동 탐색
#   docker-compose.yml / compose.yml / compose.yaml

# 파일이 다른 경로에 있을 때 -f 로 지정
docker compose -f docker/docker-compose.yml up -d
```

```txt
no configuration file provided: not found 에러:
  docker compose up -d 는 현재 폴더에서 파일을 찾음
  docker-compose.yml 이 다른 폴더에 있으면 못 찾음

  해결:
    1. docker-compose.yml 이 있는 폴더로 이동 후 실행 → cd docker && docker compose up -d
    2. -f 로 경로 직접 지정 → docker compose -f docker/docker-compose.yml up -d
```

---

---

# --env-file — 환경변수 파일 명시 ⭐️

```bash
# .env 가 자동으로 안 읽힐 때 직접 지정
docker compose --env-file .env -f docker/docker-compose.yml up -d
```

```txt
기본 동작:
  docker compose up 은 "현재 폴더" 의 .env 를 자동으로 읽음
  → 현재 폴더가 docker-compose.yml 과 다르면 .env 를 못 찾음

--env-file 이 필요한 경우:
  .env 가 프로젝트 루트에 있고 docker-compose.yml 이 하위 폴더(docker/) 에 있을 때
  docker compose --env-file .env -f docker/docker-compose.yml up -d
  → .env 위치를 명시해서 확실하게 읽힘

옵션 순서: --env-file 과 -f 는 up 앞에 위치
```

---

---

# depends_on — 서비스 의존 순서

```yaml
services:
  db:
    image: postgres
    environment:
      - POSTGRES_PASSWORD=secret
  app:
    image: my-app
    depends_on:
      - db    # db 가 먼저 시작된 후 app 시작
    ports:
      - "3000:3000"
```

```txt
⚠️ 주의: 컨테이너가 "시작된 것" 만 보장 — DB 가 "완전히 준비됐는지" 는 보장 안 함
→ 진짜로 준비될 때까지 기다리려면 위 "healthcheck" 섹션의
  depends_on: { condition: service_healthy } 조합 사용
```

---

---

# 환경변수 설정 방법

```yaml
services:
  app:
    image: my-app

    # 방법 1 — 직접 입력
    environment:
      - NODE_ENV=production
      - DB_HOST=db

    # 방법 2 — .env 파일에서 읽기 (위 "env_file" 섹션 참고)
    env_file:
      - .env
```

---

---

# named volume — 데이터 영속성

```yaml
services:
  db:
    image: postgres:17-alpine
    volumes:
      - db_data:/var/lib/postgresql/data   # 컨테이너 삭제해도 데이터는 유지

volumes:
  db_data:   # 이름만 적으면 Docker 가 자동 관리
```

```txt
volumes 없이 그냥 컨테이너만 삭제(docker compose down)하면 안의 데이터도 같이 사라짐
named volume 으로 분리해두면 컨테이너를 지우고 다시 만들어도 데이터는 살아있음
docker compose down -v 를 해야 volume 까지 진짜로 삭제됨
```

---

---

# 실전 예시 — 종합 (Postgres) ⭐️

```yaml
services:
  db:
    image: postgres:17-alpine
    container_name: music-community-db
    env_file:
      - apps/api/.env
    ports:
      - "5433:5432"
    volumes:
      - db_data:/var/lib/postgresql/data
    healthcheck:
      test: ["CMD-SHELL", 'pg_isready -U "$$POSTGRES_USER" -d "$$POSTGRES_DB"']
      interval: 10s
      timeout: 5s
      retries: 5
      start_period: 40s
    networks:
      - music-community-network

volumes:
  db_data:

networks:
  music-community-network:
    driver: bridge
```

---

---

# 주요 명령어 ⭐️

```bash
docker compose up -d              # 실행 (백그라운드)
docker compose up -d --build      # 실행 + 이미지 강제 재빌드
docker compose down               # 종료 (컨테이너만 삭제 / 볼륨 유지)
docker compose down -v            # 종료 + 볼륨도 삭제
docker compose logs -f            # 로그 확인 (전체)
docker compose logs -f 서비스명    # 로그 확인 (특정 서비스)
docker compose ps                 # 상태 확인
docker compose restart 서비스명    # 특정 서비스만 재시작
docker compose exec 서비스명 bash  # 컨테이너 안 명령 실행
```

---

---

# 추천 습관 — package.json 스크립트 등록 ⭐️

```json
// package.json
{
  "scripts": {
    "docker:up":   "docker compose --env-file .env -f docker/docker-compose.yml up -d",
    "docker:down": "docker compose -f docker/docker-compose.yml down"
  }
}
```

```bash
pnpm docker:up     # 실행
pnpm docker:down   # 종료
```

```txt
장점: 명령어 외울 필요 없음 / 팀원 모두 동일한 방식으로 실행 / --env-file 빠뜨리는 실수 방지
```

---

---

# 한눈에

```txt
포트 충돌 의심 신호:
  "연결이 안 된다" 가 아니라 "계정/DB 가 없다" 는 에러(P1010 등)가 뜨면
  → 엉뚱한(다른) 서버에 연결됐을 가능성 — 호스트 포트를 바꿔서 분리

ports: "호스트:컨테이너"  좌측만 바꿔도 컨테이너 내부 설정엔 영향 없음
env_file 로 앱의 .env 를 DB 컨테이너와 공유 → 계정 정보 불일치 방지
healthcheck + depends_on(service_healthy)  → "시작됨" 이 아니라 "준비됨" 까지 보장
$$VAR  compose 가 치환하지 않고 컨테이너 내부 셸에 그대로 넘기게 하는 이스케이프
named volume  컨테이너 지워도 데이터 보존, down -v 로만 진짜 삭제
```