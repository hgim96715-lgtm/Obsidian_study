---
aliases:
  - 모노레포
  - allowBuilds
  - monorepo
  - pnpm workspace
  - 멀티레포
tags:
  - 모노레포
  - pnpm
related:
  - "[[00_JS_Ecosystem_HomePage]]"
  - "[[00_NestJS_Ecosystem_HomePage]]"
  - "[[NestJS_Migration]]"
---
# Monorepo_PNPM — pnpm 워크스페이스로 모노레포 구성하기

> [!info] 
> 모노레포 = 여러 패키지(앱/라이브러리)를 하나의 Git 저장소에서 관리하는 방식.
>  pnpm workspace = 그 여러 패키지를 하나의 의존성 트리(lockfile 1개)로 묶어 관리하는 기능.

---

# 모노레포 vs 멀티레포 ⭐️⭐️⭐️

|구분|멀티레포 (저장소 여러 개)|모노레포 (저장소 하나)|
|---|---|---|
|코드 공유|패키지 publish 해서 import|로컬 폴더 참조 — 즉시 반영|
|타입/스키마 공유|버전 맞춰 배포해야 반영됨|수정 즉시 다른 패키지에서 바로 보임|
|PR/리뷰|프론트·백엔드 변경이 따로 흩어짐|관련 변경을 한 PR로 같이 리뷰 가능|
|설정 관리 (eslint, tsconfig)|각 저장소마다 따로|루트에서 한 번 관리|
|저장소 크기/빌드 시간|작음|커질수록 CI 시간 관리 필요|

```txt
apps/web(Next.js) + apps/api(NestJS)처럼 프론트와 백엔드가 같은 Prisma 스키마/타입을
공유해야 하는 구조라면, 모노레포가 "타입을 항상 최신으로 동기화"하는 데 특히 유리함
```

---

# 기본 디렉토리 구조

```txt
my-project/
├── apps/
│   ├── web/                  Next.js
│   └── api/                  NestJS
├── packages/                 (선택) 여러 앱이 같이 쓰는 공유 라이브러리/타입
├── pnpm-workspace.yaml       워크스페이스 멤버 정의 + pnpm 설정
├── pnpm-lock.yaml            단 하나의 lockfile — 모든 패키지 의존성 통합
└── package.json              루트 — 공통 스크립트만 (보통 private: true)
```

---

# 초기 설정 순서 ⭐️⭐️⭐️⭐️

```txt
반드시 루트 워크스페이스를 먼저 잡고, 그 다음에 각 앱을 추가해야 함
순서를 지키지 않으면 lockfile이 분리되거나 .git이 중첩되는 문제 발생
```

```bash
# 1. 루트 초기화
mkdir my-project && cd my-project
git init
pnpm init

# 2. pnpm-workspace.yaml 생성 (이 순서가 중요)
# packages: ['apps/*', 'packages/*'] 내용으로 먼저 만들어둠

# 3. 각 앱 생성
mkdir -p apps
cd apps
npx create-next-app@latest web    # Next.js
nest new api                      # NestJS

# 4. 루트에서 의존성 설치
cd ../..   # 루트로 돌아와서
pnpm install
```

## ⚠️ 워크스페이스 인식 전 설치 주의 ⭐️⭐️⭐️

```txt
pnpm-workspace.yaml이 루트에 없는 상태에서 apps/api 폴더 안으로 들어가 설치하면
그 폴더가 독립 프로젝트로 인식되어 다음 문제가 발생함:
```

|증상|원인|
|---|---|
|`apps/api`에 `pnpm-lock.yaml`이 따로 생김|워크스페이스 인식 전에 그 폴더에서 설치|
|`apps/api`에 `.git` 폴더가 따로 있음|`nest new`를 루트 세팅 전에 실행|

```txt
해결: 중복 lockfile / .git 삭제 후 루트에서 pnpm install 재실행

확인 방법: 루트에서 pnpm install 실행 시 "Scope: all 2 workspace projects" 같은
워크스페이스 범위 메시지가 나오면 정상적으로 인식된 것

결론: pnpm-workspace.yaml이 루트에 있고 각 앱이 정상 인식된 상태라면
      이후 설치/명령 실행은 cd든 --filter든 자유롭게 선택 가능
```

---

# pnpm-workspace.yaml

## packages — 워크스페이스 멤버 정의 ⭐️⭐️⭐️

```yaml
packages:
  - 'apps/*'
  - 'packages/*'
```

```txt
packages:에 적힌 glob 패턴에 해당하는 폴더들을 "워크스페이스 멤버"로 인식
각 멤버는 자기만의 package.json을 가지지만, 루트의 pnpm-lock.yaml 하나로 의존성을 통합 관리
→ 이게 "진짜" 모노레포 설정 — 워크스페이스가 어디에 있는지를 정의하는 부분
```

## allowBuilds — 빌드 스크립트 허용 목록 ⭐️⭐️⭐️

```yaml
allowBuilds:
  '@prisma/client': true
  '@prisma/engines': true
  prisma: true
```

```txt
pnpm-workspace.yaml은 워크스페이스 구성(packages) 뿐 아니라,
pnpm 자체의 일반 설정도 같이 들어가는 파일
(pnpm 11부터 .npmrc의 비인증 설정들이 전부 이 파일로 옮겨졌기 때문)

→ allowBuilds는 모노레포 여부와 무관 — 단일 패키지 프로젝트에서도 동일하게 필요한 설정
  같은 파일에 있다 보니 모노레포 설정으로 착각하기 쉬운 지점
```

### 왜 필요한가 — 공급망 공격(supply chain attack) 방어

```txt
pnpm 10부터 의존성의 postinstall/preinstall 같은 빌드 스크립트를 기본 전부 차단
이전엔 패키지를 설치하는 순간 그 패키지의 스크립트가 자동 실행됐음
  → 패키지가 해킹당하면 설치만으로 악성 코드가 그대로 실행되는 위험

→ allowBuilds에 명시된 패키지만 빌드 스크립트 실행을 허용
```

```txt
Prisma(@prisma/client · @prisma/engines · prisma)가 여기 들어가는 이유:
  설치 시 자신의 바이너리(엔진)를 내려받는 postinstall 스크립트를 실제로 사용함
  → 허용 안 해두면 스크립트가 막혀서 Prisma 자체가 정상 동작하지 않음

대화형으로 채우는 방법:
  pnpm approve-builds
  → 설치 중 보류된 패키지를 하나씩 보여주고 승인/거부 선택
  → 결과가 allowBuilds에 자동 기록됨

⚠️ "설치했는데 뭔가 안 된다" 싶으면 pnpm install 출력에 "ignored builds" 경고가 있는지 확인
```

---

# 명령 실행 — cd vs --filter ⭐️⭐️⭐️

|방법|예시|
|---|---|
|cd 후 직접|`cd apps/api && pnpm prisma migrate dev --name x`|
|`--filter` (루트에서)|`pnpm --filter api exec prisma migrate dev --name x`|
|패키지 설치|`pnpm add @prisma/client --filter api`|

```txt
둘 다 결과 동일 — pnpm이 그 워크스페이스의 로컬 바이너리를 찾아 실행하는 것은 같음
선택 기준은 "지금 어디 있는가" 뿐:
  이미 apps/api 안에 있다 → cd 후 그냥 실행
  루트에서 그대로 (스크립트 / CI 등) → --filter exec 가 편리
```

## 루트 package.json에 공통 스크립트 등록

```json
{
  "private": true,
  "scripts": {
    "dev:api": "pnpm --filter api dev",
    "dev:web": "pnpm --filter web dev",
    "build:api": "pnpm --filter api build",
    "migrate": "pnpm --filter api exec prisma migrate dev"
  }
}
```

```txt
private: true — 루트 패키지 자체가 npm에 publish되는 걸 방지
각 앱의 스크립트를 루트에서 실행할 수 있게 등록해두면 CI/배포에서 편리함
```

---

# 패키지 간 코드 공유

## 로컬 패키지 참조 ⭐️⭐️

```json
// apps/web/package.json
{
  "dependencies": {
    "@my-project/shared": "workspace:*"
  }
}
```

```txt
workspace:* — 로컬 패키지를 npm 레지스트리가 아닌 로컬 폴더에서 직접 참조
  * = 버전 무관, 현재 워크스페이스에 있는 것을 그대로 씀
  수정 즉시 반영 (publish/버전 업 불필요)

packages/ 폴더에 공유 타입이나 유틸을 두고 싶을 때 이 방식으로 참조
```

## Prisma Client — Web에서 API 것 재사용 ⭐️⭐️

```txt
Web(Next.js)과 API(NestJS)가 같은 모노레포에 있고 같은 DB를 쓴다면
Web 쪽에 Prisma Client를 새로 만들지 않고, API가 generate한 결과물을 그대로 import 가능
```

```typescript
// apps/web/lib/prisma.ts
import { PrismaClient } from '../../api/src/generated/prisma/client';

const globalForPrisma = globalThis as unknown as { prisma: PrismaClient };

export const prisma =
  globalForPrisma.prisma ?? new PrismaClient();

if (process.env.NODE_ENV !== 'production') {
  globalForPrisma.prisma = prisma;
}
```

```txt
경로는 실제 폴더 구조(apps/web, apps/api)에 맞춰야 함
output 경로를 바꾸면 이 import도 같이 변경 필요

schema.prisma 자체는 API 쪽에만 두고,
Web은 생성된 Client 코드만 가져다 씀 (schema 직접 소유 안 함)

globalThis 패턴을 쓰는 이유:
  Next.js 개발 모드에서는 파일 변경 시 모듈을 재평가(hot reload)함
  그때마다 new PrismaClient()가 호출되면 connection이 계속 생성됨
  → globalThis에 저장해두고 재사용하면 개발 중 connection 과다 생성 방지
```

---

# 자주 만나는 에러

| 증상                                  | 원인                                | 해결                                          |
| ----------------------------------- | --------------------------------- | ------------------------------------------- |
| `apps/api`에 `pnpm-lock.yaml`이 따로 생김 | 워크스페이스 인식 전에 그 폴더에서 설치            | 중복 lockfile 삭제 후 루트에서 `pnpm install`        |
| `pnpm install` 후 워크스페이스 인식 안 됨      | `pnpm-workspace.yaml`이 없거나 잘못된 위치 | 루트에 파일 생성 후 재설치                             |
| Prisma 설치 후 바이너리 없음                 | `allowBuilds`에 Prisma 누락          | `pnpm-workspace.yaml`에 allowBuilds 추가 후 재설치 |
| `pnpm --filter api` 가 패키지를 못 찾음     | `package.json`의 `name` 필드가 다름     | `apps/api/package.json`의 `"name": "api"` 확인 |