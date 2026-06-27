---
aliases:
  - Lightsail Database
  - Lightsail PostgreSQL
tags:
  - AWS
  - Deploy
related:
  - "[[00_Deploy_HomePage]]"
  - "[[AWS_Concept]]"
  - "[[AWS_Lightsail]]"
  - "[[NestJS_PostgreSQL]]"
---

# AWS_Lightsail_Database — 라이트세일 DB 연결

```txt
Lightsail 에서 제공하는 관리형 PostgreSQL 데이터베이스
서버(인스턴스) 와 별도로 DB 를 생성해서 연결
```

---

---

# DB 생성 순서

```txt
1. Lightsail 콘솔 → Databases 탭
2. Create database 클릭
3. 엔진 선택 → PostgreSQL 선택 (현재 18.4버전)
4. 플랜 선택
     Standard       단일 인스턴스 (개발/테스트)
     High Availability  이중화 구성 (운영 서비스)
     → 지금은 Standard
5. DB 이름 지정 (원하는 이름)
6. Create database
```

---

---

# 연결 정보 확인 ⭐️

```txt
DB 생성 완료 후 → Connection details 탭에서 확인

  User name    dbmasteruser
  Endpoint     ls-xxxx.chm0u6awysck.ap-northeast-2.rds.amazonaws.com
  Port         5432
  Password     Show password 클릭해서 확인
```

---

---

# .env 설정 ⭐️

```properties
DB_TYPE=postgres
DB_HOST="ls-028863206993985a121765ccd752e065295876eb.chm0u6awysck.ap-northeast-2.rds.amazonaws.com"
DB_PORT=5432
DB_USER="dbmasteruser"
DB_PASSWORD="비밀번호 붙여넣기"
DB_DATABASE=postgres
```

```txt
DB_HOST   Lightsail 에서 제공한 Endpoint 그대로 붙여넣기
DB_USER   기본값 dbmasteruser
DB_DATABASE  기본 DB 이름 postgres
             나중에 직접 DB 생성 후 이름 변경 가능
```

---

---

# TypeORM SSL 설정 ⭐️

`app.module.ts` 의 `TypeOrmModule.forRootAsync` 에 SSL 옵션 추가.

```typescript
TypeOrmModule.forRootAsync({
  inject: [ConfigService],
  useFactory: (configService: ConfigService) => ({
    type:     configService.get(envVariableKeys.dbType) as 'postgres',
    host:     configService.get(envVariableKeys.dbHost),
    port:     configService.get<number>(envVariableKeys.dbPort),
    username: configService.get(envVariableKeys.dbUsername),
    password: configService.get(envVariableKeys.dbPassword),
    database: configService.get(envVariableKeys.dbDatabase),
    entities:    [...],
    synchronize: false,

    ssl: {
      rejectUnauthorized: false,   // ← 이 옵션 추가
    },
  }),
}),
```

## ssl.rejectUnauthorized: false 란?

```txt
Lightsail DB 는 외부에서 접속할 때 SSL(암호화 통신) 을 요구함
→ SSL 없이 접속하면 연결 거부

rejectUnauthorized: true  (기본값)
  SSL 인증서가 공식 CA 에서 발급된 것인지 엄격하게 검증
  → Lightsail 자체 서명 인증서는 검증 실패 → 연결 거부

rejectUnauthorized: false
  인증서 유효성 검사를 건너뜀
  → Lightsail 자체 서명 인증서도 허용 → 연결 성공

개발·테스트 환경에서는 false 로 설정
운영 환경에서는 공식 인증서 발급 또는 Lightsail 내부 통신 권장
```

---

---

# 연결 확인

```bash
# 로컬에서 .env 에 Lightsail DB 정보 넣고 실행
pnpm start:dev
```

```txt
연결 성공 시:
  TypeORM 이 Lightsail DB 에 연결됨
  로그에 에러 없이 서버 시작

연결 실패 시:
  ECONNREFUSED  → Networking 에서 포트 확인 필요 (아래 참고)
  SSL 에러      → ssl.rejectUnauthorized: false 추가됐는지 확인
```

---

---

# Networking — 외부 접속 허용 ⭐️

```txt
Lightsail DB 는 기본적으로 외부 접속이 막혀 있음
로컬 / DataGrip 에서 접속하려면 Public mode 를 열어줘야 함
```

```txt
Lightsail 콘솔 → Databases → 해당 DB 클릭
→ Networking 탭
→ Public mode → Enable
```

```txt
⚠️ Public mode 는 개발·테스트 용도로만 사용
   운영 환경에서는 VPC 내부에서만 접속하도록 설정 권장
   (외부에 DB 포트가 열리면 보안 위험)
```

---

---

# DataGrip 연결

```txt
DataGrip → New Connection → PostgreSQL

  Host      Lightsail Endpoint 붙여넣기
  Port      5432
  User      dbmasteruser
  Password  Lightsail 에서 확인한 비밀번호
  Database  postgres

Advanced 탭 → sslmode = require 또는 disable
  → 연결 안 되면 disable 로 변경
```

---

---

# 핵심 흐름 정리

```txt
Lightsail → Databases → PostgreSQL 생성 (Standard)
    ↓
Connection details → Endpoint / User / Password 확인
    ↓
.env 에 DB_HOST / DB_USER / DB_PASSWORD 입력
    ↓
TypeORM ssl: { rejectUnauthorized: false } 추가
    ↓
pnpm start:dev → 연결 확인
    ↓
Networking → Public mode Enable (DataGrip 연결용)
    ↓
DataGrip → PostgreSQL 연결
```