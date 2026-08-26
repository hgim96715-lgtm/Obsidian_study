---
aliases:
  - CLI
  - Command Line Interface
  - 터미널
  - 쉘
  - stdin
  - stdout
  - stderr
  - 종료 코드
  - exit code
tags:
  - DevOps
  - CS
related:
  - "[[00_DevOps_Ecosystem_HomePage]]"
  - "[[OpenSSL]]"
  - "[[NestJS_Seed]]"
---
# CLI_Concept — CLI · 터미널 · 쉘 기초 개념

> [!info]
> CLI(Command Line Interface) = 텍스트 명령어로 프로그램을 조작하는 방식.
> GUI(그래픽 인터페이스)와 반대 개념.
> 터미널 → 쉘 → CLI 도구 순서로 요청이 전달됨.
> 개발 도구(git, pnpm, nest, prisma, openssl 등)는 전부 CLI로 동작함.

---

# CLI란 — GUI와 비교 ⭐️⭐️⭐️⭐️

```txt
GUI (Graphical User Interface):
  마우스로 클릭하고 버튼을 눌러 조작
  눈에 보이는 화면(창, 버튼, 메뉴)으로 상호작용
  예: Finder, 브라우저, DataGrip UI

CLI (Command Line Interface):
  텍스트 명령어를 입력해서 조작
  화면에 글자만 표시 — 마우스 필요 없음
  예: git commit -m "메시지", pnpm install, openssl rand -base64 32

왜 개발자가 CLI를 쓰는가:
  자동화 가능 — 명령어를 파일에 적어두면 반복 실행 가능 (스크립트)
  빠름 — 마우스 클릭 없이 키보드만으로 조작
  원격 서버 — GUI 없이 텍스트만 존재하는 환경에서 유일한 수단
  정확함 — 명령어는 항상 동일한 결과 보장 (클릭은 실수 가능)
```

---

# 터미널 · 쉘 · CLI 도구 — 세 가지 구분 ⭐️⭐️⭐️⭐️

```txt
터미널 (Terminal):
  텍스트를 입력하고 출력받는 "창"
  macOS: Terminal.app, iTerm2
  → 단순히 텍스트를 보여주는 화면

쉘 (Shell):
  터미널 안에서 실행되는 "명령어 해석기"
  입력한 텍스트를 해석해서 OS에 전달
  macOS 기본: zsh (Z Shell)
  Linux 기본: bash (Bash Shell)
  → 명령어를 읽고, 찾고, 실행하는 프로그램

CLI 도구:
  쉘을 통해 실행되는 "프로그램"
  git, pnpm, node, nest, prisma, openssl 등
  → 실제로 일을 하는 프로그램

흐름:
  내가 터미널에 "git commit -m '메시지'" 입력
  → 쉘(zsh)이 "git이라는 프로그램을 commit 인수와 함께 실행해" 라고 OS에 전달
  → git 프로그램이 실행되어 커밋 처리
  → 결과 텍스트를 터미널에 출력
```

```txt
비유:
  터미널 = 전화기 (입출력 장치)
  쉘     = 교환원 (요청을 받아서 연결해줌)
  CLI 도구= 실제 통화 상대방 (일을 처리하는 프로그램)
```

---

# 입력 · 출력 · 에러 ⭐️⭐️⭐️

```txt
CLI 프로그램의 세 가지 스트림:

  stdin  (표준 입력, 0번):
    프로그램이 읽어들이는 입력
    키보드 입력 또는 다른 프로그램의 출력을 파이프(|)로 받음

  stdout (표준 출력, 1번):
    프로그램의 정상 출력 — 터미널에 출력됨
    console.log() = stdout으로 출력
    > 파일명 으로 파일에 저장 가능

  stderr (표준 에러, 2번):
    에러 메시지 전용 출력 채널
    console.error() = stderr로 출력
    stdout과 분리되어 있어 파이프라인에서 에러만 필터 가능
```

```bash
# stdout → 파일로 저장
node script.js > output.txt

# stderr → 파일로 저장
node script.js 2> error.txt

# stdout + stderr 모두 파일로
node script.js > output.txt 2>&1

# 파이프: A의 stdout → B의 stdin
openssl rand -base64 32 | pbcopy   # 생성한 키를 클립보드에 복사
cat file.txt | grep "error"        # 파일에서 error가 있는 줄만 출력
```

---

# 종료 코드 (Exit Code) ⭐️⭐️⭐️⭐️

```txt
CLI 프로그램이 종료될 때 숫자 하나를 OS에 전달 = 종료 코드

  0     = 성공 (정상 종료)
  0이 아님 = 실패 (관례적으로 1을 많이 씀)

왜 중요한가:
  쉘 스크립트나 CI/CD(GitHub Actions 등)가 이 숫자로 성공·실패를 판단
  종료 코드 0 → "이 명령어 성공, 다음 단계 진행"
  종료 코드 1 → "이 명령어 실패, 파이프라인 중단"
```

```bash
# 직전 명령어의 종료 코드 확인
echo $?    # 0이면 성공, 1이면 실패

# 예시
git status
echo $?    # 0 (성공)

git commit   # 변경사항 없으면 실패
echo $?    # 1 (실패)
```

```typescript
// Node.js 스크립트에서 종료 코드 설정
process.exitCode = 1;   // 종료 코드 예약 (자연 종료 시 적용)
process.exit(1);        // 즉시 강제 종료 (종료 코드 1)
process.exit(0);        // 즉시 정상 종료

// 아무것도 설정 안 하면 → 기본값 0 (성공)
// → main().catch에서 exitCode = 1 을 설정해야 CI가 실패로 인식
// → [[NestJS_Seed]] 스크립트 종료 패턴 참고
```

---

# 자주 쓰는 CLI 도구 모음

```txt
패키지 관리:
  npm install        Node.js 패키지 설치
  pnpm add <pkg>     pnpm으로 패키지 추가
  pnpm install       lock 파일 기준으로 설치

버전 관리:
  git add .          변경 파일 스테이징
  git commit -m ""   커밋
  git push           원격 저장소에 푸시

NestJS:
  nest new <name>    새 프로젝트 생성
  nest g resource    모듈+컨트롤러+서비스 한 번에 생성
  nest build         빌드

Prisma:
  prisma migrate dev      개발 환경 마이그레이션
  prisma migrate deploy   프로덕션 마이그레이션
  prisma generate         Prisma Client 타입 재생성
  prisma studio           DB GUI 열기

보안·암호화:
  openssl rand -base64 32  시크릿 키 생성 → [[OpenSSL]]

Node.js 실행:
  node script.js           JS 파일 실행
  npx ts-node src/seed.ts  TypeScript 파일 직접 실행
  tsx src/seed.ts          ts-node 대안 (빠름)
```

---

# 인수(Argument) · 옵션(Option) · 플래그(Flag) ⭐️⭐️⭐️

```txt
CLI 명령어 구조:
  프로그램  인수      옵션(--키 값)    플래그(--불리언)
  git       commit    -m "메시지"      --no-verify
  pnpm      add       bcrypt           --save-dev
  openssl   rand                       -base64     32
  nest      g         resource         --no-spec

인수 (Argument):
  프로그램에 전달하는 위치 기반 값
  git commit → "commit"이 인수 (어떤 동작을 할지)
  pnpm add bcrypt → "bcrypt"가 인수 (무엇을 추가할지)

옵션 (Option):
  --키 값 또는 -단축키 값 형태
  git commit -m "메시지" → -m은 메시지 옵션
  값을 받는 옵션

플래그 (Flag):
  --키 형태, 값 없이 켜고 끄는 용도
  nest g resource --no-spec → --no-spec은 테스트 파일 생성 안 함
  true/false 스위치
```

---

# CLI 스크립트 작성 — Node.js/TypeScript

```typescript
// src/cli/my-script.ts
// TypeScript로 CLI 스크립트 작성하는 패턴

async function main() {
  // 명령줄 인수 읽기
  const args = process.argv.slice(2);
  // process.argv = ['node', 'script.js', '인수1', '인수2', ...]
  //  [0] = node 실행파일 경로
  //  [1] = 스크립트 파일 경로
  //  [2~] = 실제 인수들

  const name = args[0] ?? 'World';
  console.log(`Hello, ${name}!`);

  // 환경변수 읽기
  const apiKey = process.env.API_KEY;
  if (!apiKey) throw new Error('API_KEY 환경변수가 없습니다');
}

main().catch((error) => {
  console.error('[my-script] 실패:', error.message);
  process.exitCode = 1;
});
```

```bash
# 실행 방법
npx ts-node src/cli/my-script.ts 인수1
tsx src/cli/my-script.ts 인수1

# package.json scripts에 등록
"scripts": {
  "seed": "tsx src/cli/seed.ts",
  "test-user": "tsx src/cli/test-user.ts"
}
pnpm seed
pnpm test-user
```
