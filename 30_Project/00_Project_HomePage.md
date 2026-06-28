---
aliases:
  - 00_Project_HomePage — 프로젝트
tags:
  - HomePage
related:
  - "[[전체 개요 (overview)]]"
  - "[[00_JS_Ecosystem_HomePage]]"
  - "[[00_NestJS_Ecosystem_HomePage]]"
---
# 00_Project_HomePage — 프로젝트

> [!info]
> `20_Wiki`는 **언어·프레임워크 개념**을, `30_Project`는 **실제로 만든 앱 하나**를 기록하는 폴더다.  
> 모노레포 구조, 인증 흐름, 단계별 구현, 트러블슈팅처럼 "이 프로젝트에서만 그렇게 했다"는 내용이 여기 해당한다.

```txt
Wiki 노트에서 [[Project_Notes]]로 링크되던 내용은 이 폴더로 옮기는 중
개념(JwtGuard, Server Actions 등)은 Wiki — 프로젝트 적용본은 30_Project
```

---

# 폴더 구성 — 지금 있는 것만

| 폴더 | 프로젝트 | 진입 노트 |
|---|---|---|
| **31_music-community/** | Next.js + NestJS 모노레포 · 음악 추천 커뮤니티 | [[전체 개요 (overview)]] ⭐ |

```txt
프로젝트가 늘어나면 32_xxx/ 폴더를 추가하고 이 표에 행을 넣으면 됨
```

---

# music-community — 한눈에

| 항목 | 내용 |
|---|---|
| Web | Next.js 16 · Tailwind 4 · `:3031` |
| API | NestJS 11 · Prisma 7 · Swagger · `:3030` |
| DB | PostgreSQL 17 · Docker · `:5433` |
| 구조 | pnpm 모노레포 — `apps/api` + `apps/web` |
| 방향 | **Nest 집중** — 인증·DB·DTO는 API, Web은 UI + Bearer fetch |

---

# 빠른 진입 — 하고 싶은 일

| 하고 싶은 일 | 먼저 볼 노트 |
|---|---|
| 프로젝트 전체 구조·스택·실행 순서 | [[전체 개요 (overview)]] |
| Nest JWT · Guard · Bearer 흐름 (개념) | [[NestJS_JwtGuard]] · [[Auth_Concept]] |
| Next fetch · API Client (개념) | [[NextJS_API_Client]] · [[NextJS_TokenStorage]] |
| DTO · ValidationPipe (개념) | [[NestJS_DTO]] |
| Prisma · PostgreSQL (개념) | [[NestJS_Prisma]] · [[NestJS_PostgreSQL]] |
| 모노레포 pnpm (개념) | [[Monorepo_PNPM]] |
| Docker Compose로 DB 띄우기 | [[Docker_Compose]] |

---

# 프로젝트 노트 작성 기준

```txt
한 프로젝트 폴더 안에서 이렇게 나누면 찾기 쉬움 — music-community overview에도 같은 패턴 언급됨

전체 개요 (overview)   큰 틀 · mermaid · 스택 · 폴더 트리
changelog              단계별 진행·완료 체크 (날짜)
routes / auth / ui …   주제별 상세 (필요할 때 추가)
```

| Wiki (`20_Wiki`) | Project (`30_Project`) |
|---|---|
| JwtGuard가 뭔지, 어떻게 동작하는지 | 이 앱에서 Guard를 어디에 붙였는지 |
| NextJS API Client 패턴 | 이 앱의 `fetchApi.ts` · env · 실제 endpoint |
| Prisma schema 문법 | 이 앱의 `schema.prisma` · migration 이력 |
| Docker Compose 개념 | 이 앱의 `docker-compose.yml` · 포트 5433 |



