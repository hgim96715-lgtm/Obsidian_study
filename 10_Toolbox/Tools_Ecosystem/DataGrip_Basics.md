---
aliases:
  - DataGrip
  - DataGrip 기본
  - timestamptz
tags:
  - DataGrip
  - Tools
related:
  - "[[00_DB_HomePage]]"
  - "[[00_Deployment_HomePage]]"
  - "[[00_Tools_Ecosystem_HomePage]]"
  - "[[NestJS_Prisma]]"
  - "[[NestJS_StatsBucket]]"
  - "[[Snippet_date-statistics-pattern]]"
---
# DataGrip_Basics — PostgreSQL GUI · 연결 · KST 표시

```txt
Prisma/Nest가 쓰는 DB를 GUI로 열어서
테이블 · timestamptz · SQL 콘솔 확인
Postman = HTTP / DataGrip = DB
```

---

# 어디에 쓰나

| 도구                | 역할                                    |
| ----------------- | ------------------------------------- |
| **Postman**       | Nest API HTTP 요청                      |
| **DataGrip**      | Postgres **스키마·데이터·SQL**              |
| **Prisma Studio** | (선택) 간단 CRUD — 이 vault에서는 DataGrip 위주 |

```txt
연결 문자열을 apps/api/.env 에 넣지 않는다 (로컬 Docker 유지)
Neon URL → DataGrip 데이터 소스에만
```

---

# music-community 데이터 소스 (예)

| 이름 | DB | 용도 |
|------|-----|------|
| **`music-community-local`** | Docker `localhost:5433` | 로컬 개발 · `apps/api/.env`와 동일 DB |
| **`music-community-neon-prod`** | Neon (배포) | Railway API가 붙는 DB · 라이브 데이터 |

---

# 연결 설정

## 로컬 Docker

| 항목 | 값 |
|------|-----|
| Host | `localhost` |
| Port | `5433` |
| Database | `music_community_db` |
| User / Password | `apps/api/.env`의 `POSTGRES_*` |
| SSL | 없음 (`sslmode=disable`) |

## Neon (배포)

1. Neon Console → **Connect** → connection string 복사
2. DataGrip → New Data Source → PostgreSQL → **URL 붙여넣기**
3. SSL **`sslmode=require`**

```txt
postgresql://USER:PASS@ep-xxxx-pooler....neon.tech/neondb?sslmode=require
```

> 비밀번호·호스트는 vault/ git에 **실제 값 붙여넣지 않음**

---

# `timestamptz` · KST 표시 ⭐️

**DB는 UTC instant 저장** — `2026-07-01 04:07 +00` = 한국 **13:07** 과 **같은 순간**.

| 증상 | 원인 |
|------|------|
| 한국 13시인데 `04:07 +00` | UTC 표시 — **틀린 값 아님** |
| 날짜가 하루 밀림 | `timestamp`(구) 또는 세션 TZ |

개념·API 집계(KST) → [[Snippet_date-statistics-pattern]] · [[NestJS_StatsBucket]]

## ① 연결 설정 (먼저)

데이터 소스 → **Advanced** → Driver properties:

```text
options=-c TimeZone=Asia/Seoul
```

→ **Reconnect** 후 테이블 다시 열기.

## ② SQL 콘솔 — `SET TIME ZONE` (① 안 될 때)

**Query Console** 맨 위:

```sql
SET TIME ZONE 'Asia/Seoul';
SHOW TIME ZONE;   -- Asia/Seoul 이면 OK
```

```txt
세션 단위 — 새 콘솔 탭마다 다시 실행
SHOW 가 UTC면 SET 다시
```

확인 쿼리:

```sql
SELECT "lastActiveAt", "createdAt" FROM "User" LIMIT 5;
```

## ③ Init script (선택)

데이터 소스 → **Options** → **Init script / On connect**:

```sql
SET TIME ZONE 'Asia/Seoul';
```

---

# 자주 하는 일

| 하고 싶은 일 | 방법 |
|--------------|------|
| 테이블 구조 | Database 탐색기 → `User` 등 더블클릭 |
| Prisma migrate 반영 확인 | `_prisma_migrations` 테이블 |
| Admin DAU 확인 | `User.lastActiveAt` 조회 |
| 로컬 vs Neon 데이터 비교 | **다른 DB** — 동기화 안 됨 주의 |

---

# Wiki와의 경계

| DataGrip (여기) | Wiki |
|-----------------|------|
| 연결 UI · `SET TIME ZONE` | [[NestJS_Prisma]] · `@db.Timestamptz(3)` |
| 테이블 눈으로 보기 | [[00_DB_HomePage]] SQL · ER |
| UTC vs KST **표시** | [[Snippet_date-statistics-pattern]] · KST 집계 코드 |

---

