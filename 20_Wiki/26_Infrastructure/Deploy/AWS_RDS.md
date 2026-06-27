---
aliases:
  - RDS
  - AWS RDS
  - PostgreSQL RDS
tags:
  - AWS
  - Deploy
related:
  - "[[00_Deploy_HomePage]]"
  - "[[AWS_Concept]]"
  - "[[AWS_IAM]]"
  - "[[AWS_Lightsail_Database]]"
---

# AWS_RDS — RDS PostgreSQL 생성 & 연결

```txt
RDS = Relational Database Service
AWS 가 관리하는 PostgreSQL 서버
백업 / 패치 / 복구 자동화 → DB 서버 직접 관리 불필요
```

---

---

# Lightsail DB vs RDS ⭐️

```txt
Lightsail Database
  간단 / 고정 요금 / 소규모 프로젝트
  Lightsail 인스턴스와 같은 환경에서 쓰기 좋음

RDS
  세밀한 설정 가능 / 다양한 인스턴스 유형
  Elastic Beanstalk / EC2 와 조합할 때 사용
  멀티 AZ / 읽기 복제본 등 고급 기능 지원
```

---

---

# RDS PostgreSQL 생성 순서 ⭐️

## 1단계 — RDS 접속

```txt
AWS 콘솔 → RDS 검색 → 접속
→ 왼쪽 메뉴 데이터베이스 클릭
→ 데이터베이스 생성 클릭
```

## 2단계 — 엔진 선택

```txt
엔진 유형: PostgreSQL
           ↑ MySQL / MariaDB 등 선택 가능
버전:      최신 LTS 버전 선택
```

## 3단계 — 템플릿 선택

```txt
프로덕션    멀티 AZ / 고가용성 (비용 높음)
개발/테스트  단일 인스턴스 (비용 낮음) ← 실습 시 선택
프리 티어    무료 사용 한도 내
```

## 4단계 — DB 식별자 & 자격 증명

```txt
DB 인스턴스 식별자: db-nestjs-postgres
                    ↑ RDS 목록에서 보이는 이름

마스터 사용자 이름: postgres (기본값)
마스터 암호:       직접 입력 → 안전하게 보관 ⭐️
                   → .env 의 DB_PASSWORD 에 사용
```

## 5단계 — 인스턴스 구성

```txt
인스턴스 클래스:
  db.t3.micro   프리 티어 / 개발용
  db.t3.small   소규모 서비스
  db.m5.large   중규모 서비스
```

## 6단계 — 연결 설정 ⭐️

```txt
퍼블릭 액세스:
  예   → 외부(로컬 / DataGrip) 에서 접속 가능
  아니요 → VPC 내부에서만 접속 (운영 환경 권장)

보안 그룹:
  새로 생성 또는 기존 선택
  5432 포트 인바운드 규칙 열려 있어야 외부 접속 가능

가용 영역: 기본값
```

## 7단계 — 추가 구성

```txt
초기 데이터베이스 이름: postgres (비워두면 기본 DB 없음)
백업 보존 기간: 7일 (기본값)
```

## 8단계 — 생성

```txt
데이터베이스 생성 클릭
→ 생성 완료까지 약 5~10분 소요
→ 상태가 available 되면 사용 가능
```

---

---

# 연결 정보 확인 ⭐️

```txt
RDS → 데이터베이스 → nestjs-netflix-db 클릭
→ 연결 & 보안 탭

  엔드포인트: nestjs-netflix-db.xxxxxxx.ap-northeast-2.rds.amazonaws.com
  포트:       5432
  사용자:     postgres
```

---

---

# .env 설정

```properties
DB_TYPE=postgres
DB_HOST=nestjs-netflix-db.xxxxxxx.ap-northeast-2.rds.amazonaws.com
DB_PORT=5432
DB_USER=postgres
DB_PASSWORD=설정한_마스터_암호
DB_DATABASE=postgres
```

---

---

# TypeORM 연결 설정

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
    entities: [...],
    synchronize: false,   // 운영에서는 반드시 false ⭐️

    ssl: {
      rejectUnauthorized: false,   // RDS 외부 접속 시 필요
    },
  }),
}),
```

---

---

# DataGrip 연결

```txt
New Connection → PostgreSQL

  Host:     RDS 엔드포인트 붙여넣기
  Port:     5432
  Database: postgres
  User:     postgres
  Password: 마스터 암호

Advanced → sslmode = require
           연결 안 되면 disable 로 변경
```

---

---

# 보안 그룹 — 외부 접속 허용 ⭐️

```txt
RDS 생성 후 DataGrip / 로컬에서 접속이 안 되면
보안 그룹 인바운드 규칙 확인

RDS → 인스턴스 → 연결 & 보안 → 보안 그룹 클릭
→ 인바운드 규칙 편집 → 규칙 추가

  유형:   PostgreSQL
  포트:   5432
  소스:   내 IP (로컬 접속) 또는 0.0.0.0/0 (전체 허용)

⚠️ 0.0.0.0/0 은 개발용으로만 사용
   운영에서는 Beanstalk / EC2 보안 그룹 IP 만 허용
```

---

---

# 리소스 삭제 ⭐️

```txt
RDS → 데이터베이스 → 인스턴스 선택 → 작업 → 삭제

⚠️ 삭제 시 주의:
  최종 스냅샷 생성 여부 선택
  데이터 보관 필요하면 스냅샷 생성 후 삭제
  스냅샷도 비용 발생 → 불필요하면 같이 삭제
```

---

---

# 핵심 흐름 정리

```txt
RDS → 데이터베이스 생성 → PostgreSQL 선택
    ↓
DB 식별자: nestjs-netflix-db
마스터 암호 설정
    ↓
퍼블릭 액세스: 예 (개발) / 아니요 (운영)
    ↓
생성 완료 → 엔드포인트 확인
    ↓
.env 에 DB_HOST / DB_PASSWORD 입력
    ↓
TypeORM ssl: { rejectUnauthorized: false } 추가
    ↓
보안 그룹 → 5432 인바운드 열기
    ↓
DataGrip / pnpm start:dev 로 연결 확인
```