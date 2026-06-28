---
aliases:
  - NestJS 개념
  - NestJS 설치
  - nest new
tags:
  - NestJS
related:
  - "[[00_NestJS_Ecosystem_HomePage]]"
  - "[[NestJS_Env_Config]]"
  - "[[Monorepo_PNPM]]"
---
# NestJS_Concept — 개념 & 설치

# 한 줄 요약

```txt
NestJS = Node.js 위에서 동작하는 TypeScript 백엔드 프레임워크
Express 보다 구조적 / 의존성 주입 / 데코레이터 기반
```

---

---

# NestJS 란 ⭐️

```txt
Express 의 문제: 코드 구조가 정해져 있지 않음, 프로젝트가 커질수록 파일이 뒤죽박죽, 팀마다 구조가 달라 협업 어려움

NestJS 가 해결: Angular 에서 영감받은 구조화된 아키텍처
  Module / Controller / Service 역할 분리, TypeScript 기본, 의존성 주입(DI) 내장
  데코레이터(@Get, @Post, @Injectable) 로 선언적 코딩
```

|구성요소|역할|비유|
|---|---|---|
|`main.ts`|앱 시작점|전원 버튼|
|Module|관련 기능을 묶는 단위|조립 설명서|
|Controller|요청 받기 (URL + HTTP 메서드)|접수 직원|
|Service|실제 비즈니스 로직 (DB 조회·계산·가공)|실제 일하는 사람|

## ⚠️ NestJS vs Next.js

|구분|NestJS|Next.js|
|---|---|---|
|역할|Node.js 서버 백엔드 프레임워크|React 기반 풀스택 프론트엔드 프레임워크|
|담당|API 서버 / DB 연동 / 인증 처리|UI 렌더링 / SSR / 라우팅|

```txt
이름이 비슷해서 자주 혼동 — NestJS ≠ Next.js
```

---

---

# 설치 — 전역 설치 없이 시작하기 (권장) ⭐️⭐️

```bash
node --version   # 없으면 https://nodejs.org LTS 설치
sudo npm install -g pnpm
pnpm --version
```

```bash
# 전역으로 nest CLI 를 설치하지 않고 바로 프로젝트 생성
pnpm dlx @nestjs/cli new api      # NestJS 백엔드
pnpm create next-app web          # Next.js 프론트엔드
```

|명령|의미|
|---|---|
|`pnpm dlx @nestjs/cli`|`npx @nestjs/cli` 와 동의어 (npm 진영 표현)|
|`pnpm create next-app`|`pnpm dlx create-next-app` 의 축약 별칭|

```txt
dlx/create 가 하는 일: 패키지를 전역에 설치하지 않고, "한 번 실행하고 버리는" 임시 환경에서 받아와 실행
→ sudo 권한 문제 / PATH 충돌 / 전역 버전과 프로젝트 버전 불일치 문제를 구조적으로 피함

반대로 sudo npm install -g 로 전역 CLI 를 까는 방식은 이런 문제를 자주 일으킴
```

## 전역 `nest` 가 꼭 필요하다면

```txt
CLI 자동완성 / nest g 스캐폴딩을 자주 쓴다면 전역 설치가 편리할 수 있음

  1. sudo 없이 pnpm/npm 전역 경로 정리 (sudo npm install -g 와 pnpm 전역 경로가 섞이면 권한·경로 충돌의 흔한 원인)
  2. pnpm setup 실행 후 터미널 재시작 (PATH 에 pnpm 전역 bin 경로 등록)
  3. 그래도 안 되면 전역 설치를 포기하고 pnpm dlx @nestjs/cli 사용 (전역 설치가 깨져 있어도 영향 안 받음)
```

---

---

# 프로젝트 생성 ⭐️

```bash
pnpm dlx @nestjs/cli new api      # 권장 — 전역 설치 없이

nest new hello-world               # 전역 nest 가 있는 경우, 새 폴더로 시작
nest new .                         # 전역 nest + 이미 만든 Git 폴더에 바로 생성
```

> 모노레포(apps/api, apps/web 구조)로 시작한다면 → [[Monorepo_PNPM]] 먼저 참고

---

---

# 프로젝트 폴더 구조

```txt
hello-world/
├── src/
│   ├── main.ts              진입점 — 서버 시작
│   ├── app.module.ts        루트 모듈 — 전체 앱 조립
│   ├── app.controller.ts    컨트롤러 — 요청 처리
│   └── app.service.ts       서비스 — 비즈니스 로직
├── test/                    E2E 테스트
├── package.json             의존성 목록 / 스크립트
├── tsconfig.json            TypeScript 설정
└── nest-cli.json            NestJS CLI 설정
```

## src/ 각 파일 역할 ⭐️

```typescript
// main.ts — 앱 시작점
import { NestFactory } from '@nestjs/core';
import { AppModule } from './app.module';

async function bootstrap() {
  const app = await NestFactory.create(AppModule);
  await app.listen(process.env.PORT || 3000);
}
bootstrap();
```

|파일|역할|
|---|---|
|`main.ts`|`NestFactory.create(AppModule)` / 포트 지정|
|`app.module.ts`|루트 모듈 / 컨트롤러·서비스 등록 / `imports` 에 다른 모듈 추가|
|`*.controller.ts`|URL + HTTP 메서드 처리 / 비즈니스 로직은 직접 안 함|
|`*.service.ts`|실제 비즈니스 로직 / DB 조회·계산·데이터 가공|

```txt
⚠️ 실무 관행: app.controller.ts / app.service.ts 는 삭제하고 기능별 모듈로 분리하는 것이 일반적
   AppModule 은 중앙 조립 역할만 담당
```

## PORT — 포트 충돌 처리 ⭐️⭐️

```typescript
await app.listen(process.env.PORT || 3000);   // 환경변수에 PORT 있으면 그 값, 없으면 3000
```

```txt
컴퓨터 한 대에서 여러 프로젝트를 동시에 띄우는 경우가 흔함 — 포트를 코드에 고정하면 그 프로젝트를
손대지 않고는 바꿀 방법이 없음 → process.env.PORT 로 열어두면 실행할 때마다 바꿀 수 있음
```

### ⚠️ .env 에 적기만 하면 안 됨 — 로드 방법이 따로 필요함 ⭐️⭐️

```txt
Node.js 는 .env 파일을 "알아서" 읽어주지 않음 — PORT=3030 이라고 적어놔도
process.env.PORT 는 여전히 undefined 라서, 그 값을 실제로 process.env 에 채워주는 무언가가 필요
```

|방법|적용 위치|
|---|---|
|`nest start --env-file .env`|`package.json` 스크립트에 직접 추가 (Node 20.6+ 내장 옵션, NestJS CLI v11+ 에서 지원)|
|`ConfigModule.forRoot()`|내부적으로 dotenv 사용 — `AppModule` 안에서 처리되어 스크립트 수정 불필요|

```json
{
  "scripts": {
    "start:dev": "nest start --watch --env-file .env"
  }
}
```

```bash
# 또는 실행 시점에 1회성으로
PORT=3030 pnpm run start:dev

# 지금 그 포트를 누가 쓰고 있는지 확인
lsof -i :3000
npx kill-port 3000   # 강제 종료
```

```txt
⚠️ .env 로 포트를 바꿨다면, 프론트엔드(Next.js)에서 이 API 를 호출하는 baseURL 도 같이 맞춰야 함
   하나만 바꾸고 다른 쪽을 그대로 두면 "연결이 안 된다" 는 에러로 이어짐
```

---

---

# 서버 실행

```bash
pnpm run start:dev   # 개발 모드 (파일 변경 감지 자동 재시작)
pnpm run start       # 기본 실행
pnpm run build && pnpm run start:prod  # 프로덕션
curl http://localhost:3000   # → "Hello World!"
```

---

---

# package.json 스크립트 ⭐️

```json
{
  "scripts": {
    "start":       "nest start",
    "start:dev":   "nest start --watch",
    "start:debug": "nest start --debug --watch",
    "start:prod":  "node dist/main",
    "build":       "nest build",
    "test":        "jest"
  }
}
```

|스크립트|용도|
|---|---|
|`start:dev`|개발 중 — 파일 변경 시 자동 재시작|
|`start:prod`|배포 후 — 빌드된 `dist/main.js` 실행|
|`build`|TypeScript → JavaScript 컴파일 (`dist/` 생성)|

```txt
⚠️ 이건 가장 기본형 — .env 값을 실제로 로드하려면 위 "PORT" 섹션처럼 --env-file 추가 필요
```

## 주요 패키지

|패키지|역할|
|---|---|
|`@nestjs/common`|핵심 데코레이터 (`@Get` `@Post` `@Injectable` ...)|
|`@nestjs/core`|NestJS 프레임워크 엔진|
|`reflect-metadata`|TypeScript 데코레이터 동작에 필요|
|`rxjs`|비동기 처리 라이브러리|

---

---

# 의존성 관리

```bash
pnpm install              # package.json 기준 전체 설치
pnpm add @nestjs/config   # 패키지 추가
pnpm add -D @types/node   # 개발용 패키지 추가
rm -rf node_modules && pnpm install   # 문제 생길 때 재설치
```

|파일|Git 포함 여부|
|---|---|
|`node_modules/`|❌ `.gitignore` 포함 — `pnpm install` 로 자동 생성|
|`pnpm-lock.yaml`|✅ 의존성 버전 고정을 위해 반드시 포함|

## 빌드 스크립트 차단 — ERR_PNPM_IGNORED_BUILDS ⭐️

```txt
pnpm 은 보안상 postinstall 같은 빌드 스크립트가 있는 패키지를 기본 차단함
@nestjs/core 등 특정 패키지에서 이 경고가 뜨면 허용 목록에 추가해야 함
```

```bash
pnpm approve-builds   # 대화형으로 허용/거부 선택 (가장 간단)
```

```txt
직접 텍스트로 쓰는 것과 결과는 동일 — 이 기능 자체의 동작 원리(allowBuilds 맵)와
모노레포에서의 디테일은 [[Monorepo_PNPM]] 참고
```

---

---

# CJS vs ESM ⭐️

```txt
JS 모듈 시스템이 역사적으로 2개로 나뉘어 있고, NestJS 프로젝트는 그 둘을 "섞어서" 쓰는 애매한 위치에 있음
```

|구분|CJS (CommonJS)|ESM (ES Module)|
|---|---|---|
|문법|`require()` / `module.exports`|`import` / `export`|
|tsconfig|`"module": "commonjs"`|`"module": "nodenext"`|
|어디서 기본|Node.js 의 원래 기본 방식|브라우저 + 최신 JS 표준|

```txt
NestJS 프로젝트의 애매한 위치:
  tsconfig 는 "module": "nodenext" 로 ESM 에 가깝게 설정돼 있지만
  package.json 에는 "type": "module" 이 없음
  → 코드는 import/export(ESM 문법)로 쓰지만 실제 런타임 동작은 CJS 처럼 처리됨
  → "ESM 으로만 배포되는" 외부 라이브러리를 쓰면 이 애매한 설정과 충돌해 import 에러가 날 수 있음

실전에서 자주 마주치는 경우 — Prisma 연동:
  generated/prisma/client.ts 가 ESM 스타일로 생성되면 위 충돌 발생
  해결: generator 에 moduleFormat = "cjs" 추가 → [[NestJS_Prisma]] 참고

지금 당장 다 이해 안 돼도 됨 — "import/export 썼는데 왜 안 되지?" 에러를 만났을 때
"CJS/ESM 설정 충돌일 수도 있겠다" 라고 떠올릴 수 있으면 충분
```

---

---

# nest g — 리소스 생성 ⭐️

```bash
nest g resource movie          # 모듈 + 컨트롤러 + 서비스 + DTO 한번에
nest g module movie            # 개별 생성
nest g controller movie
nest g service movie
nest g service exhibitions --no-spec   # 테스트 파일 없이
```

|schematic|단축|생성 내용|
|---|---|---|
|`resource`|`res`|모듈+컨트롤러+서비스+DTO 한번에|
|`module`|`mo`|모듈만|
|`controller`|`co`|컨트롤러만|
|`service`|`s`|서비스만|
|`guard`|`gu`|가드|
|`middleware`|`mi`|미들웨어|
|`pipe`|`pi`|파이프|
|`interceptor`|`in`|인터셉터|
|`decorator`|`d`|데코레이터|

```txt
nest g resource 실행 시 선택:
  transport layer → REST API / GraphQL / WebSockets / Microservices
  CRUD entry points 생성? → Y(findAll/findOne/create/update/remove 자동) / N(빈 컨트롤러·서비스만)

생성되는 파일:
  src/movie/movie.module.ts, movie.controller.ts, movie.service.ts
  src/movie/dto/create-movie.dto.ts, update-movie.dto.ts
  src/movie/entities/movie.entity.ts
```

---

---

# VSCode Debugger 설정 ⭐️

```txt
console.log() 디버깅보다 효율적 — 원하는 줄에서 코드 멈춤, 그 시점 변수 전부 확인
```

```json
// .vscode/launch.json
{
  "version": "0.2.0",
  "configurations": [
    {
      "type": "node",
      "request": "launch",
      "name": "Debug Nest FrameWork",
      "runtimeExecutable": "pnpm",
      "runtimeArgs": ["run", "start:debug", "--", "--inspect-brk"],
      "autoAttachChildProcesses": true,
      "restart": true,
      "sourceMaps": true,
      "stopOnEntry": false,
      "console": "integratedTerminal"
    }
  ]
}
```

|옵션|의미|
|---|---|
|`sourceMaps: true`|컴파일된 JS 가 아닌 원본 `.ts` 기준 브레이크포인트|
|`restart: true`|프로세스 종료 시 자동 재시작|
|`request: "launch"` vs `"attach"`|VS Code 가 직접 실행 vs 이미 실행 중인 프로세스에 붙기|

```txt
사용 방법: ① 줄 번호 왼쪽 클릭으로 중단점 ② ▶ → "Debug Nest FrameWork" ③ 요청 전송 ④ 중단점에서 변수 확인
단축키: F5 계속 실행 / F10 다음 줄 / F11 함수 안으로 / Shift+F5 종료
```

---

---

# 자주 만나는 에러

|증상|원인|해결|
|---|---|---|
|`pnpm pnpm install ...` 처럼 명령이 이중으로 보임, 스캐폴딩 직후 install 단계에서 실패|CLI 의존성 설치 단계의 알려진 버그 — 이전에 pnpm PATH/권한 문제(`pnpm setup` 실패, `sudo npm`과 `pnpm` 혼용 등)를 겪은 환경에서 더 잘 발생|파일 생성 자체는 끝나있는 경우가 많음 — 해당 폴더에서 `pnpm install --strict-peer-dependencies=false` 를 수동으로 한 번 더 실행|
|위 복구 직후 빌드 스크립트 승인 경고|pnpm 의 기본 빌드 스크립트 차단 (`ERR_PNPM_IGNORED_BUILDS`)|`pnpm approve-builds` 로 해당 패키지(`@nestjs/core` 등) 허용|
|위 이중 실행 버그를 아예 피하고 싶을 때|`pnpm dlx`가 매번 새로 받아 실행하는 방식이라 이 버그와 자주 겹침|전역 `nest` 가 있다면 `pnpm dlx` 대신 `nest new ... --skip-install` 후 `pnpm install` 을 직접 실행 — install 단계를 CLI 가 자동으로 안 하게 분리|

```txt
이 프로젝트에서 실제로 겪었던 구체적인 경로/로그는 [[Project_Notes]] 참고
```

---

---

# 명령어 한눈에

|명령어|역할|
|---|---|
|`pnpm dlx @nestjs/cli new api`|전역 설치 없이 새 프로젝트 생성 ⭐️ (권장)|
|`pnpm create next-app web`|전역 설치 없이 Next.js 프로젝트 생성|
|`nest new 프로젝트명`|(전역 nest 있을 때) 새 프로젝트 생성|
|`nest new .`|(전역 nest 있을 때) 현재 폴더에 생성|
|`nest new api --skip-install`|스캐폴딩만, install 은 수동으로|
|`PORT=3030 pnpm run start:dev`|다른 프로젝트와 포트 충돌 시 임시로 다른 포트 사용|
|`lsof -i :3000`|그 포트를 누가 쓰고 있는지 확인|
|`pnpm approve-builds`|차단된 빌드 스크립트 대화형으로 허용|
|`pnpm run start:dev`|개발 서버 실행|
|`nest g resource 이름`|모듈+컨트롤러+서비스 한번에|
|`nest g module/controller/service 이름`|개별 생성|
|`--no-spec`|테스트 파일 없이 생성|

```txt
모노레포로 시작한다면(워크스페이스 설정, 중첩 .git 등) → [[Monorepo_PNPM]]
```