---
aliases: [모노레포, allowBuilds, monorepo, pnpm workspace]
tags:
  - 모노레포
  - pnpm
related:
  - "[[00_JS_Ecosystem_HomePage]]"
---
# Monorepo_PNPM — pnpm 워크스페이스로 모노레포 구성하기

# 한 줄 요약

```txt
모노레포 = 여러 독립적인 패키지(앱/라이브러리)를 하나의 Git 저장소에서 관리하는 방식
pnpm workspace = pnpm 이 그 여러 패키지를 하나의 의존성 트리(lockfile 1개)로 묶어 관리해주는 기능

이 노트는 NestJS/Next.js 둘 다보다 "한 단계 앞서는" 저장소 구조 자체의 이야기
```

---

---

# 모노레포 vs 멀티레포(폴리레포)

|구분|멀티레포(저장소 여러 개)|모노레포(저장소 하나)|
|---|---|---|
|코드 공유|패키지 publish 해서 import|로컬 폴더 참조 — 즉시 반영|
|타입/스키마 공유|버전 맞춰 배포해야 반영됨|수정 즉시 다른 패키지에서 바로 보임|
|PR/리뷰|프론트·백엔드 변경이 따로 흩어짐|관련 변경을 한 PR로 같이 리뷰 가능|
|설정 관리(eslint, tsconfig)|각 저장소마다 따로|루트에서 한 번 관리|
|저장소 크기/빌드 시간|작음|커질수록 빌드/CI 시간 관리 필요|

```txt
이 프로젝트(apps/web=Next.js, apps/api=NestJS)처럼 프론트와 백엔드가 같은 Prisma 스키마/타입을
공유해야 하는 구조라면, 모노레포가 "타입을 항상 최신으로 동기화" 하는 데 특히 유리함
```

---

---

# 기본 디렉토리 구조

```txt
my-project/
├── apps/
│   ├── web/              Next.js
│   └── api/              NestJS
├── packages/             (선택) 여러 앱이 같이 쓰는 공유 라이브러리/타입
├── pnpm-workspace.yaml
└── package.json          루트 — 보통 비워두거나 공통 스크립트만
```

---

---

# pnpm-workspace.yaml — packages 필드 ⭐️⭐️⭐️

```yaml
packages:
  - 'apps/*'
  - 'packages/*'
```

```txt
packages: 에 적힌 glob 패턴에 해당하는 폴더들을 "워크스페이스 멤버" 로 인식
각 멤버는 자기만의 package.json 을 가지지만, 루트의 pnpm-lock.yaml 하나로 의존성을 통합 관리
→ 이게 "진짜" 모노레포 설정 — 워크스페이스가 어디로 구성되는지를 정의하는 부분
```

## --filter — 특정 패키지만 대상으로 명령 실행

```bash
pnpm add @prisma/client --filter api    # api 패키지에만 설치
pnpm --filter api exec prisma generate  # api 패키지 안에서만 명령 실행
```

```txt
cd 로 그 폴더에 들어가서 직접 실행해도 결과는 동일 — --filter 는 루트에서 안 옮기고 실행하고 싶을 때 편의
(자세한 비교는 [[NestJS_Prisma_Monorepo]] 참고)
```

## ⚠️ 중첩 Git 저장소 — `nest new`/`create-next-app` 을 모노레포 안에서 실행할 때 ⭐️⭐️

```txt
nest new / create-next-app 은 기본적으로 그 폴더에서 git init 까지 자동으로 함
모노레포는 이미 루트에 .git 이 하나 있는 구조인데, 그 하위 폴더(apps/api 등) 안에 또 .git 이 생기면
"저장소 안에 또 다른 저장소"(nested repo) 가 되어버림
→ 루트에서 git add 해도 그 폴더 내용이 통째로 하나의 파일처럼만 잡히고
  실제 변경 내역이 추적 안 되는 등 git 이 이상하게 동작함
```

```bash
# apps/api, apps/web 등 — 모노레포 하위 폴더에 .git 이 생겼다면 삭제
rm -rf apps/api/.git
rm -rf apps/web/.git
```

```txt
이미 루트에 .git 이 있는 모노레포라면, 하위 폴더에는 별도 .git 이 있으면 안 됨 — 항상 확인할 것
```

---

---

# 루트 package.json — 무엇을 적는가 ⭐️⭐️⭐️

```json
{
  "name": "my-project",
  "private": true,
  "scripts": {
    "dev:api": "pnpm --filter api start:dev",
    "dev:web": "pnpm --filter web dev"
  }
}
```

|필드|뭘 적나|
|---|---|
|`name`|프로젝트/저장소 이름 — 자유롭게 지어도 됨 (npm 에 publish 될 일이 없어서 이름 충돌 걱정 없음). 보통 저장소 폴더명과 맞춤|
|`private: true`|⚠️ 거의 항상 필수 — 루트는 실제 배포되는 패키지가 아니므로, 실수로 npm 에 publish 되는 걸 막는 안전장치|
|`scripts`|개별 워크스페이스를 `--filter` 로 대신 호출해주는 진입점 — 매번 `cd apps/api` 안 하고 루트에서 바로 실행|

## 스크립트 이름 짓는 법 — `<동작>:<패키지>` 관례 ⭐️⭐️

```txt
콜론(:) 으로 "무엇을" 과 "어디서" 를 구분해서 이름 붙이는 게 흔한 관례

  dev:api    dev:web      개발 서버 실행
  build:api  build:web    빌드
  start:api  start:web    프로덕션 실행
```

```json
{
  "scripts": {
    "dev:api":   "pnpm --filter api start:dev",
    "dev:web":   "pnpm --filter web dev",
    "build:api": "pnpm --filter api build",
    "build:web": "pnpm --filter web build"
  }
}
```

```txt
이렇게 나누는 이유:
  api/web 각각의 실제 명령(start:dev, dev 등)은 패키지마다 이름이 다를 수 있음
  (NestJS 는 보통 start:dev, Next.js 는 보통 dev)
  → 루트에서는 그 차이를 감추고, dev:api / dev:web 처럼 "내가 뭘 실행하는지" 만 일관되게 노출
```

## 여러 개를 한 번에 실행하고 싶다면

|방법|예시|
|---|---|
|`concurrently` 패키지 (가장 흔함)|`"dev": "concurrently \"pnpm dev:api\" \"pnpm dev:web\""`|
|`pnpm -r --parallel`|각 패키지의 스크립트 이름이 전부 같을 때만 가능 (예: 전부 `dev` 로 통일)|

```txt
스크립트 이름이 패키지마다 다르면(start:dev vs dev) -r --parallel 로 한 번에 못 묶음
→ 이럴 때는 concurrently 로 직접 두 명령을 나열하는 게 일반적
```

---

---

# ⚠️ allowBuilds — 모노레포 설정이 아니라 별개의 pnpm 보안 기능 ⭐️⭐️⭐️

```yaml
allowBuilds:
  '@prisma/client': true
  '@prisma/engines': true
  prisma: true
```

```txt
pnpm-workspace.yaml 은 "워크스페이스 구성"(packages) 뿐 아니라, pnpm 자체의 일반 설정도 같이 들어가는 파일
(pnpm 11부터 .npmrc 의 비인증 설정들이 전부 이 파일로 옮겨졌기 때문)
→ allowBuilds 는 모노레포 여부와 무관 — 단일 패키지 프로젝트에서도 똑같이 필요한 설정
  (같은 파일에 있다 보니 모노레포 설정으로 착각하기 쉬운 지점)
```

## 왜 필요한가 — 공급망 공격(supply chain attack) 방어

```txt
pnpm 10부터 의존성의 postinstall/preinstall 같은 빌드 스크립트를 기본적으로 전부 차단함
이전엔 패키지를 설치하는 순간 그 패키지의 스크립트가 자동 실행됐음
  → 패키지가 해킹당하면(레지스트리 계정 탈취 등) 설치만으로 악성 코드가 그대로 실행되는 위험이 있었음
→ allowBuilds 에 명시된 패키지만 빌드 스크립트 실행을 허용하는 방식으로 전환
```

|값|의미|
|---|---|
|`true`|이 패키지의 빌드 스크립트 실행 허용|
|`false`|차단 (목록에 없는 패키지의 기본 동작과 동일)|

```txt
Prisma(@prisma/client, @prisma/engines, prisma)가 여기 들어가는 이유:
  설치 시 자신의 바이너리(엔진)를 내려받는 postinstall 스크립트를 실제로 사용함
  → 허용 안 해두면 그 스크립트가 막혀서 Prisma 자체가 정상 동작하지 않음

직접 안 적고 대화형으로 채우는 방법:
  pnpm approve-builds
  → 설치 중 보류된 패키지를 하나씩 보여주고 승인/거부 선택 → 결과가 allowBuilds 에 자동 기록됨
```

```txt
⚠️ 목록에 없는 패키지는 기본적으로 차단되고, strictDepBuilds(기본값 true)로 설치 자체가 에러로 멈춤
   "설치했는데 뭔가 안 된다" 싶으면 pnpm install 출력에 "ignored builds" 경고가 있는지 먼저 확인
```

---

---

# 한눈에

```txt
모노레포 = 여러 패키지를 한 저장소에서 관리, pnpm workspace = 그걸 하나의 의존성 트리로 묶음

packages: 필드     → 진짜 모노레포 설정 (워크스페이스 멤버가 어디 있는지)
allowBuilds        → 별개의 pnpm 보안 기능 (설치 스크립트 허용 목록), 같은 파일에 있을 뿐 모노레포와 무관
--filter <패키지>   → 특정 워크스페이스 멤버만 골라서 설치/명령 실행 (cd 후 직접 실행해도 결과는 동일)
루트 package.json: name 은 자유, private:true 거의 필수, scripts 는 <동작>:<패키지> 관례(dev:api, dev:web)
nest new/create-next-app 을 하위 폴더에서 실행하면 중첩 .git 이 생길 수 있음 — 바로 삭제

Prisma 가 allowBuilds 에 필요한 이유: 설치 시 자체 바이너리를 받는 postinstall 스크립트를 쓰기 때문
NestJS + Prisma 모노레포 조합의 디테일(moduleFormat 이슈 등) → [[NestJS_Prisma_Monorepo]]
```