---
aliases:
  - Prisma 모노레포
  - pnpm Prisma
tags:
  - NestJS
related:
  - "[[00_NestJS_Ecosystem_HomePage]]"
  - "[[NestJS_Prisma]]"
---
# NestJS_Prisma_Monorepo — pnpm 모노레포에서 Prisma

# 명령 실행 — 두 가지 동등한 방법 ⭐️⭐️⭐️

|방법|예시|
|---|---|
|cd 후 직접|`cd apps/api && pnpm prisma migrate dev --name x`|
|`--filter` (루트에서)|`pnpm --filter api exec prisma migrate dev --name x`|

```txt
둘 다 결과 동일 — pnpm 이 그 워크스페이스의 로컬 prisma 바이너리를 찾아 실행하는 것은 같음
선택 기준은 "지금 어디 있는가" 뿐:
  이미 apps/api 안에 있다 → 그냥 cd 후 실행
  루트에서 그대로(스크립트/CI 등) → --filter exec 가 편리
```

---

---

# 진짜 주의할 시점 — 워크스페이스 인식 "전" 설치 ⭐️⭐️⭐️

```txt
명령 실행(migrate/generate)은 위치 상관없이 안전
위험한 건 "pnpm-workspace.yaml 으로 루트 워크스페이스가 아직 안 잡힌 상태" 에서의 설치/초기화
```

|증상|원인|
|---|---|
|`apps/api` 에 `pnpm-lock.yaml` 이 따로 생김|워크스페이스 인식 전에 그 폴더에서 설치|
|`apps/api` 에 `.git` 폴더가 따로 있음|`nest new` 를 루트 세팅 전에 실행|

```txt
해결: 중복 lockfile/.git 삭제 후 루트에서 재설치
결론: pnpm-workspace.yaml 이 루트에 있고 apps/api 가 정상 인식된 상태라면
      이후 설치/명령 실행은 cd 든 --filter 든 자유롭게 선택 가능
```

---

---

# Prisma 7 + NestJS 통합 이슈

## moduleFormat 불일치 (ESM/CJS)

```txt
에러: ReferenceError: exports is not defined in ES module scope
원인: Prisma 7 클라이언트 기본 출력은 ESM, NestJS 빌드는 CJS — 형식 불일치
해결: schema.prisma generator 블록에 moduleFormat = "cjs" 추가, output 은 src/ 하위로 통일
```

## 폴더 위치 혼동

```txt
schema.prisma        → prisma/schema.prisma (프로젝트 루트 기준)
PrismaService/Module  → src/prisma/ (NestJS 소스 코드 위치)
→ 둘은 서로 다른 폴더, 같은 "prisma" 라는 이름 때문에 혼동하기 쉬움
```

---

---

# 이 프로젝트 DB 연결 (참고용)

```bash
DATABASE_URL=postgresql://music_user:music_password@localhost:5433/music_community_db?schema=public&sslmode=disable
```

```txt
Docker Compose 의 계정/포트 설정과 일치해야 함 → [[Docker_Compose]] 참고
prisma init 으로 자동 생성된 URL 은 Prisma 클라우드 주소이므로 반드시 위처럼 교체
TypeORM 과 같은 DB 쓰면 충돌 → Prisma 전용 DB 따로 생성
```

---

---

# Next.js(Web) 에서 API 의 Prisma Client 재사용

```txt
Web(Next.js, Auth.js)과 API(NestJS) 가 같은 모노레포에 있고 같은 DB 를 쓴다면
Web 쪽에 Prisma Client 를 새로 만들지 않고, API 가 generate 한 결과물을 그대로 import 가능
([[Next_Auth]] 의 "백엔드와 DB 를 공유한다면" 참고)
```

```typescript
// apps/web/lib/prisma.ts — 모노레포 상대 경로 예시
import { PrismaClient } from "../../api/src/generated/prisma/client";
```

```txt
경로는 실제 폴더 구조(apps/web, apps/api)에 맞춰야 함 — output 경로를 바꾸면 이 import 도 같이 변경
schema.prisma 자체는 API 쪽에만 두고, Web 은 생성된 Client 코드만 가져다 씀 (schema 직접 소유 안 함)
```