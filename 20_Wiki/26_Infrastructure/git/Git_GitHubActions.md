---
aliases:
  - GitHub Actions
  - CI/CD
  - 워크플로우
  - workflows
  - YAML
  - workflow_dispatch
  - env
  - secrets
  - Node.js CI 패턴
  - 빌드 & 테스트
  - Matrix
tags:
  - Git
related:
  - "[[00_Git_HomePage]]"
  - "[[Git_Remote]]"
  - "[[Git_Commands]]"
---
# Git_GitHubActions — GitHub Actions

## 한 줄 요약

```txt
GitHub Actions = 코드 변경 시 자동으로 빌드 / 테스트 / 배포
CI (지속적 통합) + CD (지속적 배포) 플랫폼
.github/workflows/*.yml 파일로 정의
```

---

---

#  GitHub Actions 란 ⭐️

```txt
CI (Continuous Integration):
  PR(Pull Request) 생성 시 자동으로 빌드 + 테스트 실행
  → 코드 품질 자동 검증

CD (Continuous Delivery/Deployment):
  main 브랜치에 병합되면 자동으로 배포
  → 수동 배포 없이 자동화

활용 예시:
  PR(Pull Request) 올리면 자동 테스트 실행
  main merge 시 서버 자동 배포
  스케줄로 데이터 파이프라인 실행
  패키지 자동 빌드 및 배포

공개 리포지토리: 무료
비공개 리포지토리: 월 2000분 무료
```

---

---

#  시작하기 — 디렉토리 구조

```bash
# 저장소 안에 .github/workflows 디렉토리 생성
mkdir -p .github/workflows
#         ↑ .github 숨김 폴더 / workflows 하위폴더

# 워크플로 파일 생성
touch .github/workflows/main.yml

# 구조 확인
ls -R .github
# .github:
# workflows
#
# .github/workflows:
# main.yml
```

```txt
⚠️ 규칙:
  .github/workflows/ 아래 .yml 파일만 GitHub 이 워크플로로 인식
  다른 위치에 있으면 무시됨

왜 YAML 인가:
  사람이 읽기 쉬운 형식
  Kubernetes / Docker Compose 등 업계 표준
  계층 구조 표현에 적합
  들여쓰기로 구조 명확히 표현
```

---

---

#  워크플로 파일 구조 ⭐️

```yaml
# .github/workflows/hello-world.yml

name: Hello World Workflow   # 워크플로 이름

on: [push]                   # 모든 브랜치 push 시 실행 (단축형)

jobs:
  build:                     # job ID
    runs-on: ubuntu-latest   # 실행 환경

    steps:
      - name: Say Hello      # step 이름
        run: echo "Hello, World!"   # 실행할 명령어
```

```txt
주요 키워드:
  name      워크플로 / step 이름
  on        트리거 (언제 실행할지)
  jobs      실행할 job 목록
  runs-on   실행 환경 (ubuntu-latest / windows / macos)
  steps     job 안의 단계들
  uses      미리 만들어진 Action 사용
  run       직접 명령어 실행
```

## ⚠️ YAML 작성 주의사항

```yaml
# run 뒤에 띄어쓰기 없어야 함
run: echo "Hello"   ← ✅ 올바름
run : echo "Hello"  ← ❌ 에러 (run 뒤 공백)

# 들여쓰기 = 2칸 또는 4칸 통일
# 탭 사용 불가 → 스페이스만 사용
```

## 여러 steps 추가 — 실전 예시 ⭐️

```yaml
name: Simple Commands

on: [push]

jobs:
  build:
    runs-on: ubuntu-latest   # 실행 환경 (GitHub 제공 Ubuntu)

    steps:
      - name: Echo Hello
        run: echo "Hello, World!"

      - name: Show Date
        run: date             # 러너의 현재 날짜/시간

      - name: List Files
        run: ls -la           # 현재 디렉토리 목록
```

```txt
runs-on 선택:
  ubuntu-latest  Linux (가장 많이 씀 / 무료)
  windows-latest Windows
  macos-latest   macOS

step 구조:
  - name: 단계 이름    ← 로그에서 보이는 이름
    run: 명령어        ← 실행할 shell 명령어

  name 뒤 run 들여쓰기:
    - name: Echo Hello
      run: echo "Hello"   ← name 과 같은 들여쓰기 ✅
        run: echo "Hello" ← 더 깊으면 에러 ❌
```

## YAML 들여쓰기 에러 ⭐️

```yaml
# ❌ 에러 — run 이 너무 깊게 들여써짐
steps:
  - name: Echo Hello
      run: echo "Hello"   ← name 보다 더 들여써짐 → 에러

# ✅ 올바름 — name 과 run 동일 깊이
steps:
  - name: Echo Hello
    run: echo "Hello"     ← name 과 같은 깊이
```

```txt
에러 메시지:
  "run" is not a valid step property at this indentation level

해결:
  name 과 run 의 들여쓰기를 같게 (2칸 또는 4칸)
  - 으로 시작하는 것이 하나의 step
  해당 step 의 키들은 모두 같은 들여쓰기
```

---

---

#  트리거(on) 종류 ⭐️

```yaml
# 단축형 — 모든 브랜치 push 시
on: [push]

# 상세 설정 — 특정 브랜치만
on:
  push:
    branches: [ main, develop ]

  pull_request:
    branches: [ main ]

  # 스케줄 (cron)
  schedule:
    - cron: '0 9 * * 1'  # 매주 월요일 09:00

  # 수동 실행 (Actions 탭에서 버튼 클릭)
  workflow_dispatch:

  # 릴리스 게시 시
  release:
    types: [published]
```

```txt
on: [push]        모든 브랜치 push
on: [push, pull_request]  push + PR(Pull Request) 둘 다
branches: [main]  특정 브랜치만

자주 쓰는 조합:
  개발 중: on: [push]         모든 push 마다 확인
  운영용:  push branches:main  main 에 merge 될 때만
  배포용:  release             릴리스 태그 생성 시
```

---

---

# 첫 워크플로 만들기 실습

```bash
# 1. 저장소 복제
git clone https://github.com/유저명/저장소명.git
cd 저장소명

# 2. 디렉토리 생성
mkdir -p .github/workflows

# 3. 워크플로 파일 작성
cat > .github/workflows/hello-world.yml << 'EOF'
name: Hello World Workflow

on: [push]

jobs:
  build:
    runs-on: ubuntu-latest
    steps:
      - name: Say Hello
        run: echo "Hello, World!"
EOF

# 4. 커밋 & 푸시
git add .
git commit -m "ci: Hello World workflow 추가"
git push origin main
```

## GitHub Actions 탭에서 확인 ⭐️

```txt
push 후 GitHub 저장소 → Actions 탭 이동
"Hello World Workflow" 실행 중 또는 완료 확인

워크플로 실행 클릭 → 상세 로그:
  워크플로 이름:  Hello World Workflow  (상단)
  job 이름:      build                 (왼쪽 사이드바)
  step 확인:     "Say Hello" 클릭
  출력:          Hello, World!

여러 워크플로 있을 때:
  왼쪽 사이드바에서 워크플로 이름으로 필터
```

---

---

#  환경변수 (env) ⭐️

## 워크플로 레벨 환경변수

```yaml
name: Environment Variable Demo

on: [push]

env:
  GREETING: "Hello"    # 모든 jobs / steps 에서 사용 가능

jobs:
  print-greeting:
    runs-on: ubuntu-latest
    steps:
      - name: Print Greeting
        run: echo "${{ env.GREETING }}, World!"
        # 출력: Hello, World!
```

```txt
env: 키워드:
  워크플로 최상위 레벨에 선언
  아래 모든 jobs / steps 에서 ${{ env.변수명 }} 으로 참조

${{ env.변수명 }}:
  GitHub Actions 표현식 문법
  {{ }} 안에 컨텍스트.변수명 형태
  env      = 환경변수 컨텍스트
  secrets  = 비밀값 컨텍스트
  github   = GitHub 메타데이터 컨텍스트
```

## 레벨별 환경변수 범위 ⭐️

```yaml
env:
  WORKFLOW_VAR: "전체"    # 워크플로 전체에 적용

jobs:
  my-job:
    env:
      JOB_VAR: "이 job만"  # 이 job 안에서만 적용

    steps:
      - name: step1
        env:
          STEP_VAR: "이 step만"  # 이 step에서만 적용
        run: |
          echo "${{ env.WORKFLOW_VAR }}"
          echo "${{ env.JOB_VAR }}"
          echo "${{ env.STEP_VAR }}"
```

```txt
우선순위 (좁을수록 높음):
  step env > job env > workflow env

secrets 와 차이:
  env      공개 환경변수 (로그에 노출될 수 있음)
  secrets  암호화된 비밀값 (로그에서 마스킹됨)
  → API 키 / 비밀번호 → secrets 사용
  → 일반 설정값 → env 사용
```

---
---
#  GitHub Actions Secrets ⭐️

```txt
Secrets = 리포지토리에 저장하는 암호화된 환경변수
API 키 / DB 비밀번호 / 토큰 등 코드에 노출 안 할 값

저장소 Secrets 추가 방법:
  1. 리포지토리 → Settings 탭
  2. 왼쪽 Security → Secrets and variables → Actions
  3. New repository secret 클릭
  4. Name: MY_SECRET / Secret: 실제값 입력
  5. Add secret

워크플로에서 사용:
  ${{ secrets.MY_SECRET }}
```

## Secrets 자동 마스킹 ⭐️

```yaml
jobs:
  use-secret:
    runs-on: ubuntu-latest
    steps:
      - name: Print Secret
        env:
          MY_SECRET_VAL: ${{ secrets.MY_SECRET }}  # step 환경변수로 전달
        run: |
          echo "직접 출력: ${{ secrets.MY_SECRET }}"
          echo "env 로 출력: $MY_SECRET_VAL"
```

```txt
실행 결과:
  Printing secret directly (masked): ***
  Printing secret from env (masked) : ***

  → echo 로 출력해도 실제 값 대신 *** 로 마스킹
  → GitHub 가 자동으로 로그에서 숨겨줌

secrets vs env:
  env:   공개 환경변수 (로그에 값 그대로 출력)
  secrets: 암호화 + 로그 마스킹 → 비밀값에 사용

secrets 주의사항:
  워크플로 외부로 출력 불가 (마스킹)
  fork PR 에서는 secrets 접근 제한 (보안)
  Organization / Repository / Environment 단위로 관리 가능
```

---
---
#  actions/checkout — 코드 체크아웃 ⭐️

```txt
Runner(가상 환경)는 빈 서버로 시작
→ 리포지토리 코드가 없음
→ actions/checkout 으로 코드를 Runner 에 가져와야 함

actions/checkout 없이 ls -la:
  .git / .github 폴더만 있음 (실제 코드 없음)

actions/checkout 후 ls -la:
  README.md / index.js 등 실제 파일 보임
```

```yaml
steps:
  - name: Checkout code
    uses: actions/checkout@v4    # 리포지토리 코드 가져오기

  - name: List files
    run: ls -la                  # 이제 index.js 등 파일 보임
```

```txt
uses vs run:
  uses  미리 만들어진 Action 사용 (actions/checkout 같은)
  run   직접 shell 명령어 실행

actions/checkout@v4:
  @ 뒤 = 버전 (v5 = 4번째 메이저 버전)
```


---

---

#  Node.js CI 패턴 — 빌드 & 테스트 ⭐️

```yaml
# .github/workflows/node-ci.yml
name: Node.js CI

on: [push]

jobs:
  build:
    runs-on: ubuntu-latest

    steps:
      # 1. 리포지토리 코드 가져오기
      - uses: actions/checkout@v4

      # 2. Node.js 설치
      - name: Use Node.js
        uses: actions/setup-node@v4
        with:
          node-version: "22"   # 로컬 버전과 맞추기 (node -v 로 확인)

      # 3. 의존성 설치
      - name: Install dependencies
        run: npm install

      # 4. 테스트 실행
      - name: Run tests
        run: npm test
```

```txt
actions/setup-node:
  Runner(빈 서버) 에 Node.js 를 설치하는 공식 액션
  actions/checkout 이 코드를 가져온 후 Node.js 설정

node-version:
  로컬 개발 환경과 맞추는 게 좋음
  node -v → v22.22.2 → "22" 로 지정
  버전 다르면 로컬에서 되는데 CI 에서 안 되는 문제 발생 가능

npm test 실패 조건:
  package.json 의 "test" 스크립트 종료 코드 != 0
  → 워크플로 실패로 표시됨
```

## 최소한의 package.json

```json
{
  "name": "my-project",
  "version": "1.0.0",
  "scripts": {
    "test": "echo \"Running tests...\" && exit 0"
  }
}
```

```txt
CI 워크플로 실행 순서:
  1. Checkout code   리포지토리 코드 가져옴
  2. Use Node.js     Node.js v22 설치
  3. Install deps    npm install 실행
  4. Run tests       npm test 실행 → 성공/실패 결정
```

---
---
#  빌드 아티팩트 — 업로드 & 다운로드 ⭐️

## 빌드 / 아티팩트 개념

```txt
빌드 (Build):
  소스코드를 실행 가능한 형태로 변환하는 과정
  TypeScript → JavaScript 컴파일 (tsc)
  React → 정적 파일 번들링 (npm run build)
  결과물이 dist/ 또는 build/ 폴더에 생성됨

아티팩트 (Artifact):
  빌드 결과물 = "만들어진 산출물"
  dist/app.js / build/index.html 등
  테스트 결과 파일 / 커버리지 리포트도 아티팩트

왜 GitHub Actions 에서 아티팩트를 업로드하나:
  Runner 는 임시 서버 → 워크플로 끝나면 전부 사라짐
  빌드 결과물을 보존하려면 업로드 필요
  → GitHub UI 에서 다운로드 가능
  → 다음 Job 에서 이어서 사용 가능 (테스트 후 배포 등)
```

## actions/upload-artifact

```yaml
name: Upload Artifacts

on: [push]

jobs:
  build:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4

      - name: Use Node.js
        uses: actions/setup-node@v4
        with:
          node-version: "22"

      - name: Install dependencies
        run: npm install

      - name: Build project
        run: |
          mkdir dist
          echo "빌드 결과물" > dist/build.txt
          # 실제 프로젝트: npm run build → dist/ 생성

      - name: Run tests
        run: npm test

      - name: Upload build artifact
        uses: actions/upload-artifact@v4
        with:
          name: build-assets   # GitHub UI 에 표시될 이름
          path: dist           # 업로드할 폴더 (또는 파일)
```

```txt
with 옵션:
  name  GitHub UI 의 Artifacts 섹션에 표시될 이름
  path  업로드할 경로 (폴더 전체 / 특정 파일)

  path: dist               → dist 폴더 전체
  path: dist/app.js        → 특정 파일만
  path: |                  → 여러 경로
    dist/
    coverage/
```

## GitHub UI 에서 확인

```txt
Actions 탭 → 워크플로 실행 클릭
→ 요약 페이지 하단 Artifacts 섹션
→ build-assets 클릭 → zip 파일 다운로드
```


---
---
# Matrix Build — 여러 환경 동시 테스트 

```txt
Matrix Strategy = 변수 조합으로 Job 을 자동 생성
Node.js v18 / v20 / v22 에서 동시에 테스트
별도 Job 3개를 직접 쓰는 대신 matrix 로 자동화
```

```yaml
name: Matrix Build

on: [push]

jobs:
  build:
    runs-on: ubuntu-latest

    strategy:
      matrix:
        node-version: [18, 20, 22]   # 세 개의 Job 자동 생성
    #                  ↑ 배열 원소 수 = 생성될 Job 수

    steps:
      - uses: actions/checkout@v4

      - name: Use Node.js
        uses: actions/setup-node@v4
        with:
          node-version: ${{ matrix.node-version }}
          #              ↑ 현재 matrix 값 참조

      - name: Install dependencies
        run: npm install

      - name: Run tests
        run: npm test

      - name: Upload artifact
        uses: actions/upload-artifact@v4
        with:
          name: build-assets-${{ matrix.node-version }}
          #                   ↑ 이름 겹치면 덮어씌워짐 → 반드시 구분
          path: dist
```

```txt
strategy.matrix:
  변수 이름(node-version)과 값 배열 정의
  GitHub Actions 가 각 값마다 Job 자동 생성

  [18, 20, 22] → Job 3개 병렬 실행:
    build (18)
    build (20)
    build (22)

${{ matrix.node-version }}:
  현재 실행 중인 Job 의 matrix 값 참조
  setup-node 의 node-version / artifact name 등에 사용

아티팩트 이름 주의 ⭐️:
  name: build-assets              → 3개 Job 이 같은 이름 → 덮어씌움 ❌
  name: build-assets-${{ matrix.node-version }}
    → build-assets-18 / build-assets-20 / build-assets-22 → 구분됨 ✅

GitHub Actions 탭에서 확인:
  build (18) / build (20) / build (22) 병렬 실행
  각 Job 클릭 → Use Node.js 단계에서 버전 확인
```

---
---
# Job Dependencies — needs ⭐️

```txt
Job 간 순서와 의존성을 정의하는 방법
build 성공 후에만 deploy 실행
Job 은 기본적으로 병렬 실행 → needs 로 순서 제어
```

```yaml
name: Job Dependencies

on: [push]

jobs:
  build:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4
      - name: Use Node.js
        uses: actions/setup-node@v4
        with:
          node-version: "22"
      - name: Install dependencies
        run: npm install
      - name: Run tests
        run: npm test
      - name: Build project
        run: |
          mkdir dist
          echo "Build artifact at $(date)" > dist/build.txt
      - name: Upload artifact
        uses: actions/upload-artifact@v4
        with:
          name: dist-files
          path: dist

  deploy:
    runs-on: ubuntu-latest
    needs: build          # ← build 성공 후에만 실행
    steps:
      - name: Download artifact
        uses: actions/download-artifact@v4   # ← v7 부터 Node.js 24
        with:
          name: dist-files   # ← upload 때 쓴 이름과 반드시 일치
          path: dist
      - name: Deploy project
        run: |
          echo "Deploying..."
          ls -R dist
```

```txt
needs: build:
  build Job 이 성공해야 deploy Job 실행
  build 실패 → deploy 는 실행 안 됨

왜 download-artifact 가 필요한가:
  Job 은 서로 다른 가상 머신에서 실행
  → 파일 시스템을 공유하지 않음
  → build Job 에서 upload 한 아티팩트를
     deploy Job 에서 download 해서 써야 함

여러 Job 에 의존:
  needs: [build, test]   → 두 Job 모두 성공해야 실행
```

## ⚠️ 버전 경고 — download-artifact ⭐️

```txt
경고:
  Warning: Node.js 20 actions are deprecated.
  actions/download-artifact@v6 runs on Node.js 20

원인:
  actions/download-artifact@v6 → 내부 런타임 Node.js 20
  actions/upload-artifact@v6 / checkout@v6 / setup-node@v6 → Node.js 24

해결:
  actions/download-artifact@v7 부터 Node.js 24 로 변경
  → @v4 또는 @v7 이상 사용 권장

  upload-artifact 와 download-artifact 버전 맞추기:
    upload-artifact@v4 + download-artifact@v4   ✅
    upload-artifact@v6 + download-artifact@v7   ✅
    upload-artifact@v6 + download-artifact@v6   ← 경고 발생 ⚠️
```

---
---
#  여러 워크플로가 동시에 실행되는 이유 ⭐️

```txt
스크린샷에서 push 하나에 3개가 실행됨:
  Simple Commands #3
  CI #1
  Hello World Workflow #5

이유:
  .github/workflows/ 아래 yml 파일이 3개
  on: [push] 가 모두 설정됨
  → 하나의 push 이벤트 → 3개 워크플로 전부 트리거

  워크플로 파일마다 독립적으로 실행됨
  yml 파일 수 = 동시 실행 워크플로 수
```

## push 시 모두 실행되는 것 막기 ⭐️

```yaml
# ❌ 전부 push 에 반응
on: [push]

# ✅ 방법 1: 수동 실행만 (Actions 탭 버튼 클릭)
on:
  workflow_dispatch:

# ✅ 방법 2: 특정 브랜치 push 만
on:
  push:
    branches: [main]   # main 에 push 시만 실행

# ✅ 방법 3: PR 생성 시만
on:
  pull_request:
    branches: [main]
```

```txt
workflow_dispatch:
  Actions 탭에서 "Run workflow" 버튼이 생김
  자동으로 트리거 안 됨 → 원할 때만 수동으로 실행
  테스트 / 배포 등 직접 실행하고 싶은 워크플로에 적합

실무 전략:
  테스트 자동화  → on: [push] / pull_request
  수동 배포      → on: workflow_dispatch
  스케줄 작업    → on: schedule
  → 파일마다 트리거를 다르게 설정해서 충돌 방지
```


