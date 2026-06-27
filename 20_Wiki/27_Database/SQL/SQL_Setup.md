---
aliases:
  - SQL 설치
  - PostgreSQL 설치
  - psql
  - DB 연결
  - DataGrip
tags:
  - SQL
related:
  - "[[00_SQL_HomePage]]"
  - "[[SQL_Concept]]"
  - "[[NestJS_PostgreSQL]]"
  - "[[NodeJS_PostgreSQL]]"
---

# SQL_Setup — PostgreSQL 설치 & 연결

---

---

# ① PostgreSQL 설치 ⭐️

## 방법 1 — Docker (권장)

```bash
# PostgreSQL 컨테이너 실행
docker run -d \
  --name postgres-dev \
  -e POSTGRES_PASSWORD=mysecret \
  -e POSTGRES_USER=myuser \
  -e POSTGRES_DB=mydb \
  -p 5432:5432 \
  -v postgres-data:/var/lib/postgresql/data \
  postgres:16

# 확인
docker ps | grep postgres
```

```txt
포트: 5432 (PostgreSQL 기본 포트)
-v postgres-data  컨테이너 삭제해도 데이터 유지
-e POSTGRES_PASSWORD  반드시 설정 (없으면 에러)
```

## 방법 2 — 직접 설치 (Ubuntu)

```bash
sudo apt update
sudo apt install -y postgresql postgresql-contrib

# 서비스 시작
sudo systemctl start postgresql
sudo systemctl enable postgresql   # 부팅 시 자동 시작

# 버전 확인
psql --version
```

## 방법 3 — Mac (Homebrew)

```bash
brew install postgresql@16
brew services start postgresql@16
```

---

---

# ② psql — CLI 클라이언트 ⭐️

```bash
# 로컬 접속
psql -U postgres
psql -U myuser -d mydb

# 원격 접속
psql -h localhost -p 5432 -U myuser -d mydb

# Docker 컨테이너 접속
docker exec -it postgres-dev psql -U myuser -d mydb
```

## psql 주요 명령어

```sql
-- 메타 명령어 (\ 로 시작)
\l              -- 데이터베이스 목록
\c mydb         -- mydb 로 전환
\dt             -- 테이블 목록
\d users        -- users 테이블 구조
\du             -- 유저 목록
\q              -- 종료

-- SQL 직접 실행
SELECT version();
SELECT current_database();
\timing         -- 쿼리 실행 시간 표시
```

---

---

# ③ DataGrip — GUI 클라이언트 

```txt
DataGrip = JetBrains 의 DB GUI 툴
PostgreSQL / MySQL / SQLite 등 다양한 DB 지원
SQL 작성 / 자동완성 / 테이블 조회 / ER 다이어그램

무료 버전 사용법:
  JetBrains Toolbox 또는 학생 라이선스 (무료)
  또는 30일 평가판

다운로드: https://www.jetbrains.com/datagrip/

DBeaver 도 무료 대안이지만 느린 편
→ DataGrip 권장
```

## DataGrip 연결 설정

```txt
+ New Data Source → PostgreSQL 선택

Host:     localhost
Port:     5432
Database: mydb
User:     myuser
Password: mysecret

Test Connection → 성공 확인
```

---

---

# ④ Node.js 에서 연결 (pg 모듈) ⭐️

```bash
pnpm add pg
pnpm add -D @types/pg
```

```typescript
import { Pool } from 'pg';

const pool = new Pool({
  host:     process.env.DB_HOST     || 'localhost',
  port:     Number(process.env.DB_PORT) || 5432,
  database: process.env.DB_NAME     || 'mydb',
  user:     process.env.DB_USER     || 'myuser',
  password: process.env.DB_PASSWORD || 'mysecret',
});

// 연결 테스트
pool.query('SELECT NOW()', (err, res) => {
  if (err) console.error('연결 실패:', err);
  else     console.log('연결 성공:', res.rows[0]);
});

// 쿼리 실행
const result = await pool.query(
  'SELECT * FROM users WHERE id = $1',
  [userId]
);
console.log(result.rows);
```

```txt
→ 자세한 내용: [[NodeJS_PostgreSQL]]
```

---

---

# ⑤ NestJS 에서 연결 (TypeORM) ⭐️

```bash
pnpm add @nestjs/typeorm typeorm pg
```

```typescript
// app.module.ts
import { TypeOrmModule } from '@nestjs/typeorm';

@Module({
  imports: [
    TypeOrmModule.forRootAsync({
      useFactory: (config: ConfigService) => ({
        type:     'postgres',
        host:     config.get('DB_HOST'),
        port:     config.get<number>('DB_PORT'),
        username: config.get('DB_USER'),
        password: config.get('DB_PASSWORD'),
        database: config.get('DB_NAME'),
        entities:     [__dirname + '/**/*.entity{.ts,.js}'],
        synchronize:  false,   // 프로덕션에서 반드시 false
      }),
      inject: [ConfigService],
    }),
  ],
})
export class AppModule {}
```

```txt
.env 파일:
  DB_HOST=localhost
  DB_PORT=5432
  DB_USER=myuser
  DB_PASSWORD=mysecret
  DB_NAME=mydb

→ 자세한 내용: [[NestJS_TypeORM]]
→ NestJS PostgreSQL 연결: [[NestJS_PostgreSQL]]
```

---

---

# ⑥ MySQL 설치 & 연결

```bash
# Docker
docker run -d \
  --name mysql-dev \
  -e MYSQL_ROOT_PASSWORD=mysecret \
  -e MYSQL_DATABASE=mydb \
  -p 3306:3306 \
  mysql:8

# CLI 접속
docker exec -it mysql-dev mysql -u root -p
```

```txt
MySQL 기본 포트: 3306
PostgreSQL 기본 포트: 5432

DBeaver 연결 시 동일 방법 / 드라이버만 MySQL 선택
```

---

---

# 포트 정리

|DB|기본 포트|
|---|---|
|PostgreSQL|5432|
|MySQL|3306|
|Redis|6379|
|MongoDB|27017|