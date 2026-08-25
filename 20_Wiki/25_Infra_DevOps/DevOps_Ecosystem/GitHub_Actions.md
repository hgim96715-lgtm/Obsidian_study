---
aliases:
  - GitHub Actions
  - workflow
  - cron
  - CI/CD
  - workflow_dispatch
  - Actions
  - 자동화
  - 워크플로우
  - set -euo pipefail
  - upload-artifact
  - retention-days
  - TZ 날짜
tags:
  - DevOps
  - GitHub
related:
  - "[[00_DevOps_Ecosystem_HomePage]]"
  - "[[Deploy_FullStack]]"
  - "[[NestJS_Scheduling]]"
  - "[[NestJS_Controller]]"
  - "[[NestJS_Excel]]"
---
# GitHub_Actions — 자동화 워크플로우

> [!info]
> GitHub 서버(runner)에서 특정 이벤트(push·schedule·수동)가 발생하면 정해진 명령을 자동 실행하는 CI/CD 도구.
> `.github/workflows/*.yml` 파일로 정의. 내 서버가 아닌 GitHub 서버에서 실행됨.

---

# 핵심 구조 ⭐️⭐️⭐️⭐️

```txt
Workflow  .yml 파일 하나 = 워크플로우 하나
  └─ on       언제 실행할지 (trigger)
  └─ jobs     무엇을 실행할지
       └─ runs-on   어떤 OS에서 (ubuntu-latest 등)
       └─ steps     실행할 명령 목록
            └─ name   단계 이름
            └─ run    실제 실행할 shell 명령
            └─ uses   미리 만들어진 Action 사용 (actions/checkout 등)
```

---

# 파일 추가 방법 ⭐️⭐️⭐️⭐️

```txt
1. 프로젝트 루트에 폴더 생성
   .github/
   └─ workflows/
        └─ my-workflow.yml   ← 여기에 파일 추가

2. main 브랜치에 push하면 GitHub이 자동 인식
   → GitHub 레포 → Actions 탭에서 확인 가능
   → workflow_dispatch 트리거가 있으면 수동 실행 버튼도 생김
```

---

# on — 트리거 종류 ⭐️⭐️⭐️⭐️

```yaml
on:
  push:                        # main에 push될 때
    branches: [main]

  schedule:
    - cron: '5 17 * * *'      # 매일 UTC 17:05 (= KST 02:05)

  workflow_dispatch:           # GitHub UI에서 수동 실행 버튼
```

```txt
cron 표현식 — 5자리 (분 시 일 월 요일):
  '5 17 * * *'   매일 17:05 UTC
  '0 0 * * 1'    매주 월요일 00:00 UTC
  '*/30 * * * *' 30분마다

  ⚠️ GitHub Actions cron은 UTC 기준
  KST(UTC+9) → UTC 변환 후 입력
  KST 02:05 → UTC 17:05 전날

  최소 간격: 5분 (그 이하는 무시됨)
  schedule은 정확한 시간 보장 안 됨 — 최대 몇 분 지연 가능
```

---

# Secrets — 민감한 값 관리 ⭐️⭐️⭐️⭐️

```txt
등록 방법:
  GitHub 레포 → Settings → Secrets and variables → Actions
  → New repository secret → 이름·값 입력

  이름 예: CINEMO_API_URL, CINEMO_CRON_SECRET
```

## CINEMO_API_URL 값 — Railway 도메인 발급 방법 ⭐️⭐️⭐️⭐️

```txt
Railway 대시보드 → 프로젝트 → API 서비스 클릭
  → Settings 탭 → Networking → Public Networking
  → Generate Domain 버튼 클릭
  → "Enter the port your app is listening on" 입력 창 뜸

PORT 값 확인:
  Variables 탭 → PORT 항목 값 확인
  없으면 Railway 자동 주입값 — 서비스 로그에서 "Listening on port xxxx" 확인
  → 숫자 입력 → Generate

발급 결과: https://<service-name>.up.railway.app
  → 이 URL 전체를 CINEMO_API_URL Secret 값으로 등록 (끝 / 없이)

⚠️ Actions curl이 계속 실패할 때 PORT 불일치 확인:
  NestJS main.ts가 app.listen(3000) 하드코딩이면
  Railway가 다른 PORT를 주입해도 앱이 그 포트로 안 뜸
  → 외부 도메인 접근 자체가 안 됨 → curl --fail-with-body 에러

  해결: app.listen(process.env.PORT ?? 3000)
  → [[Deploy_FullStack#2. Railway (API)]]
```

```yaml
# yml에서 사용
env:
  API_URL: ${{ secrets.CINEMO_API_URL }}
  CRON_SECRET: ${{ secrets.CINEMO_CRON_SECRET }}

steps:
  - name: API 호출
    env:
      API_URL: ${{ secrets.CINEMO_API_URL }}
    run: |
      curl --request POST "${API_URL}/v1/something"
```

```txt
secrets 특징:
  로그에 자동으로 *** 마스킹됨
  fork된 PR에서는 접근 불가 (보안)
  환경변수로 주입 → run 블록 안에서 $API_URL 로 사용
```

---

# curl로 API 호출 — 스케줄 워크플로우 패턴 ⭐️⭐️⭐️⭐️

```yaml
# .github/workflows/movie-pool-seed.yml
name: MoviePool Seed
on:
  schedule:
    - cron: '5 17 * * *'   # KST 02:05
  workflow_dispatch:         # 수동 실행 가능

jobs:
  seed:
    runs-on: ubuntu-latest
    timeout-minutes: 30
    steps:
      - name: Trigger MoviePool seed
        env:
          API_URL: ${{ secrets.CINEMO_API_URL }}
          CRON_SECRET: ${{ secrets.CINEMO_CRON_SECRET }}
        run: |
          curl --fail-with-body --silent --show-error \
            --request POST \
            --url "${API_URL}/v1/tmdb/seed-pool/cron?pages=3" \
            --header "x-cron-secret: ${CRON_SECRET}"
```

```txt
curl 옵션:
  --fail-with-body   4xx·5xx 응답 시 종료 코드 1 반환 (워크플로우 실패 처리)
  --silent           진행 상태 출력 숨김
  --show-error       에러는 출력 (--silent와 같이 씀)
  --request POST     HTTP 메서드
  --header           헤더 추가
```

```txt
Actions 실행 중 curl 에러 원인 판단:

  exit code 22 + {"statusCode":401}
    → JWT Guard가 막음 → @Public() 누락
    → [[NestJS_Controller#외부 시스템 cron 엔드포인트 — x-cron-secret 패턴]]

  exit code 22 + {"statusCode":500, "message":"데이터베이스 오류"}
    → API는 정상 기동, DB 테이블이 없음 → migration 미적용
    → prisma migrate deploy 필요 → [[Deploy_FullStack#prisma generate vs prisma migrate deploy]]

  exit code 6 또는 curl: (6) Could not resolve host
    → API_URL Secret 값 오타 또는 Railway 도메인 미발급

  exit code 22 + {"statusCode":500} (응답 없이)
    → PORT 불일치 → app.listen(process.env.PORT ?? 3000) 확인
    → [[Deploy_FullStack#2. Railway (API)]]
```

---

# NestJS — cron secret 검증 패턴 ⭐️⭐️⭐️⭐️

```txt
GitHub Actions → 내 NestJS 서버 POST 요청
→ 아무나 호출하면 안 되니까 secret으로 검증
→ x-cron-secret 헤더에 secret 값을 담아서 보냄
→ 서버에서 환경변수와 비교
```

```typescript
@Post('seed-pool/cron')
seedPoolCron(
  @Headers('x-cron-secret') secret: string | undefined,
  @Query('pages', new DefaultValuePipe(3), ParseIntPipe) pages: number,
) {
  // CRON_SECRET 없거나 불일치 → 401
  if (!process.env.CRON_SECRET || secret !== process.env.CRON_SECRET) {
    throw new UnauthorizedException('잘못된 cron secret입니다.');
  }

  return this.someService.execute(pages);
}
```

```txt
패턴 요약:
  GitHub Secrets  CINEMO_CRON_SECRET = "랜덤 긴 문자열"
  NestJS .env     CRON_SECRET = "동일한 값"
  요청 헤더       x-cron-secret: ${CRON_SECRET}
  서버 검증       secret !== process.env.CRON_SECRET → 401

  JWT Guard 대신 이 패턴을 쓰는 이유:
  → GitHub Actions는 로그인 유저가 아님
  → 단순 secret 비교가 cron 전용으로 충분
```

---

# GitHub UI에서 워크플로우 실행 · 관리 ⭐️⭐️⭐️⭐️

## Actions 탭 — 워크플로우 목록 확인

```txt
GitHub 레포 → 상단 탭 → Actions
  왼쪽 사이드바: 워크플로우 목록 (yml 파일 이름 기준)
  오른쪽: 최근 실행 기록 (성공 ✅ / 실패 ❌ / 진행 중 🟡)
```

## workflow_dispatch — 수동 실행 ⭐️⭐️⭐️⭐️

```txt
수동 실행 버튼이 보이는 조건:
  yml에 workflow_dispatch: 트리거가 있어야 함
  main 브랜치에 push된 yml만 인식됨

실행 방법:
  Actions 탭 → 왼쪽 사이드바에서 워크플로우 선택
  → 오른쪽 상단 "Run workflow" 버튼 클릭
  → 브랜치 선택 (보통 main)
  → "Run workflow" 확인

  ⚠️ workflow_dispatch가 yml에 없으면 버튼 자체가 안 보임
  ⚠️ yml을 main에 push하지 않으면 목록에도 안 뜸
```

## 실행 로그 확인

```txt
Actions 탭 → 실행 항목 클릭
  → 좌측: job 목록
  → job 클릭 → step별 접기/펼치기 가능

각 step 옆 아이콘:
  ✅ 초록  성공
  ❌ 빨강  실패 (종료 코드 non-zero)
  ⏭️       스킵됨

실패 step:
  클릭하면 stderr 포함 전체 로그 펼쳐짐
  curl --fail-with-body → API 응답 body도 로그에 찍힘
  secrets 값은 *** 마스킹 처리됨
```

## 실패 시 재실행

```txt
실행 결과 페이지 → 우측 상단 "Re-run jobs" 드롭다운
  Re-run failed jobs   실패한 job만 다시 실행 (빠름)
  Re-run all jobs      전체 job 재실행

스케줄 워크플로우가 예상 시간에 안 돌았을 때:
  → "Run workflow" 수동 실행으로 즉시 트리거 가능
  → schedule은 몇 분 지연 발생 가능 (GitHub 서버 부하)
```

---

# Secrets 종류 — Repository vs Environment ⭐️⭐️⭐️⭐️

```txt
GitHub에서 Secret을 등록하는 위치가 두 군데:

  1. Repository Secret
     Settings → Secrets and variables → Actions → Repository secrets
     → 레포의 모든 워크플로우에서 ${{ secrets.NAME }} 으로 바로 사용 가능
     → yml에 environment: 설정 불필요

  2. Environment Secret
     Settings → Secrets and variables → Actions → Environments → {env 선택} → Secrets
     → 특정 환경(production, staging 등)에만 적용
     → yml의 해당 job에 environment: production 명시 필요
```

## `Context access might be invalid` 경고 ⭐️⭐️⭐️⭐️

```txt
증상:
  yml 편집 중 ${{ secrets.MY_SECRET }} 에 노란 경고
  "Context access might be invalid: MY_SECRET"

원인:
  Secret을 Environment Secret으로 등록했는데
  해당 워크플로우 job에 environment: 설정이 없음
  → GitHub이 "이 컨텍스트에서 접근 가능한 secret인지 모르겠음" 경고

해결 방법 A — Repository Secret으로 재등록 (가장 빠름):
  Settings → Secrets and variables → Actions → Repository secrets
  → 기존 Environment secret 삭제 후 Repository secret으로 새로 등록
  → yml 그대로 사용 가능, 경고 사라짐

해결 방법 B — yml에 environment: 추가:
  job에 environment: production 추가하면 그 환경의 secret에 접근 가능
  → 별도 승인 게이트(protection rule) 설정했다면 이 방식 필요
```

```yaml
# 해결 B — environment 명시
jobs:
  seed:
    runs-on: ubuntu-latest
    environment: production   # ← 이 환경의 secret을 사용
    steps:
      - name: API 호출
        env:
          API_URL: ${{ secrets.CINEMO_API_URL }}
```

```txt
실무 기준:
  단순 cron 워크플로우       → Repository Secret (yml 단순, 경고 없음)
  배포·환경 분리가 필요할 때  → Environment Secret + environment: 명시
```

---

# 파일 다운로드 워크플로우 — Excel 리포트 패턴 ⭐️⭐️⭐️⭐️

```yaml
# .github/workflows/daily-excel.yml
name: Daily Admin Excel

on:
  schedule:
    - cron: "0 17 * * *"   # KST 02:00 = UTC 17:00
  workflow_dispatch:

jobs:
  export:
    runs-on: ubuntu-latest
    timeout-minutes: 10

    steps:
      - name: Download yesterday report
        env:
          API_URL: ${{ secrets.CINEMO_API_URL }}
          CRON_SECRET: ${{ secrets.CINEMO_CRON_SECRET }}
        run: |
          set -euo pipefail

          report_date="$(TZ=Asia/Seoul date -d 'yesterday' +%F)"
          report_month="$(TZ=Asia/Seoul date -d 'yesterday' +%Y/%m)"
          report_path="reports/${report_month}/cinemo-${report_date}.xlsx"

          mkdir -p "$(dirname "$report_path")"

          curl --fail-with-body --silent --show-error \
            --request POST \
            --url "${API_URL}/v1/admin/reports/daily-excel/cron" \
            --header "x-cron-secret: ${CRON_SECRET}" \
            --output "$report_path"

          test -s "$report_path"
          echo "생성 파일: $report_path"

      - name: Upload report artifact
        uses: actions/upload-artifact@v4
        with:
          name: cinemo-daily-excel-${{ github.run_id }}
          path: reports/
          if-no-files-found: error
          retention-days: 90
```

## set -euo pipefail — bash 안전 플래그 ⭐️⭐️⭐️⭐️

```txt
set -e   에러 시 즉시 종료 (exit on error)
  명령이 0이 아닌 종료 코드를 반환하면 즉시 스크립트 종료
  이게 없으면 curl 실패해도 다음 줄을 그냥 실행

set -u   미정의 변수 사용 시 에러
  $UNDEFINED_VAR 참조하면 에러 (기본은 빈 문자열로 처리됨)
  오타 방지 — $CRON_SECERT 같은 실수를 잡아줌

set -o pipefail   파이프 중간 실패도 감지
  cmd1 | cmd2 에서 cmd1이 실패해도 기본적으로 cmd2가 성공이면 전체 성공으로 처리
  pipefail → 파이프 어디서든 실패하면 전체 실패

-euo = -e -u -o 를 합친 표기
  pipefail은 -o 옵션으로 설정하는 방식이라 별도 -o pipefail
  set -euo pipefail = 세 가지를 한 줄에 켜는 표준 관용구

Actions run 블록에서 항상 첫 줄에 쓰는 이유:
  Actions는 기본적으로 -e만 켜져 있음
  -u, pipefail은 직접 켜야 완전한 보호
```

## TZ=Asia/Seoul date — 타임존 날짜 추출 ⭐️⭐️⭐️⭐️

```bash
report_date="$(TZ=Asia/Seoul date -d 'yesterday' +%F)"
# → "2026-08-24"

report_month="$(TZ=Asia/Seoul date -d 'yesterday' +%Y/%m)"
# → "2026/08"
```

```txt
TZ=Asia/Seoul:
  date 명령 앞에 붙이는 환경변수
  해당 명령 실행 동안만 타임존을 Asia/Seoul로 변경
  GitHub Actions runner는 UTC → 그대로 쓰면 KST와 날짜가 다를 수 있음

date -d 'yesterday':
  어제 날짜를 기준으로 (today는 -d 없이 바로 사용)
  KST 기준 어제 = UTC 기준 어제와 다를 수 있어서 TZ 지정 필수

포맷 지정자:
  +%F    = +%Y-%m-%d 의 단축형 → "2026-08-24"
  +%Y/%m = 연도/월              → "2026/08"

$(...)  명령 치환:
  $()로 감싼 명령의 출력을 변수에 대입
  report_date="$(date ...)" → date 출력 결과가 변수에 들어감
```

## mkdir -p "$(dirname ...)" — 경로 자동 생성 ⭐️⭐️⭐️

```bash
report_path="reports/${report_month}/cinemo-${report_date}.xlsx"
# → "reports/2026/08/cinemo-2026-08-24.xlsx"

mkdir -p "$(dirname "$report_path")"
# → "reports/2026/08/" 폴더를 없으면 생성
```

```txt
dirname "$report_path":
  경로에서 파일명을 제거하고 디렉토리 부분만 반환
  "reports/2026/08/cinemo-2026-08-24.xlsx" → "reports/2026/08"

mkdir -p:
  -p = parents: 중간 경로(reports, reports/2026)가 없어도 한 번에 생성
  이미 있으면 에러 없이 그냥 넘어감

왜 이 패턴인가:
  report_path가 여러 단계 폴더를 포함 (reports/연도/월/파일)
  curl --output으로 파일을 저장하려면 상위 폴더가 이미 있어야 함
  → 먼저 mkdir로 경로 확보
```

## curl --output — 응답을 파일로 저장 ⭐️⭐️⭐️⭐️

```bash
curl --fail-with-body --silent --show-error \
  --request POST \
  --url "${API_URL}/v1/admin/reports/daily-excel/cron" \
  --header "x-cron-secret: ${CRON_SECRET}" \
  --output "$report_path"
```

```txt
--output "$report_path":
  curl 응답 body를 지정한 파일 경로에 저장
  이게 없으면 응답이 stdout으로 출력됨 (터미널에 바이너리 출력)
  xlsx처럼 바이너리 파일은 --output으로 파일에 직접 저장해야 함

--fail-with-body:
  4xx·5xx 응답 시 종료 코드 1 반환 → set -e와 함께 워크플로우 실패 처리
  응답 body도 출력해줌 (어떤 에러인지 로그에 찍힘)

--silent:
  curl 자체의 진행 상태(전송 속도 등) 출력 숨김

--show-error:
  --silent와 같이 쓰는 짝꿍 — 에러만 출력 (진행은 숨기되 에러는 보임)
  → 평상시 로그 깨끗 + 에러 시 원인 확인 가능
```

## test -s — 파일 비어있지 않은지 검증 ⭐️⭐️⭐️

```bash
test -s "$report_path"
```

```txt
test -s 파일경로:
  파일이 존재하고 크기가 0보다 크면 → 종료 코드 0 (성공)
  파일이 없거나 비어있으면 → 종료 코드 1 (실패)

set -e가 켜져 있으므로 → 실패하면 워크플로우 즉시 종료

왜 필요한가:
  curl --fail-with-body는 HTTP 에러코드는 잡지만
  API가 200을 반환하면서 빈 body를 보내는 경우는 감지 못 함
  → test -s로 이중 검증 → "진짜로 파일이 왔는지" 확인
```

## upload-artifact 옵션 ⭐️⭐️⭐️⭐️

```yaml
- name: Upload report artifact
  uses: actions/upload-artifact@v4
  with:
    name: cinemo-daily-excel-${{ github.run_id }}
    path: reports/
    if-no-files-found: error
    retention-days: 90
```

```txt
name:
  Actions UI에서 보이는 아티팩트 이름
  ${{ github.run_id }} → 실행마다 고유 ID 붙여서 이름 충돌 방지

path:
  업로드할 파일/폴더 경로
  reports/ 폴더 전체 → 폴더 구조(연도/월)까지 함께 업로드

if-no-files-found:
  warn   파일 없어도 워크플로우는 성공 처리 (기본값)
  error  파일 없으면 step 실패 → 워크플로우 실패 처리 ← 권장
  ignore 조용히 넘어감

  error로 설정하는 이유:
  curl 단계에서 --output 저장 실패가 set -e에 안 걸렸어도
  upload 단계에서 반드시 걸림 → 이중 안전장치

retention-days:
  아티팩트 보관 기간 (기본: 90일, 최대: 90일 — 무료 플랜 기준)
  기간 지나면 자동 삭제
  Actions UI → 해당 run → Artifacts 섹션에서 다운로드
```

---

# 범용 cron 워크플로우 템플릿

```yaml
name: 워크플로우 이름
on:
  schedule:
    - cron: 'UTC 크론식'
  workflow_dispatch:

jobs:
  job이름:
    runs-on: ubuntu-latest
    timeout-minutes: 10
    steps:
      - name: API 호출
        env:
          API_URL: ${{ secrets.API_URL }}
          CRON_SECRET: ${{ secrets.CRON_SECRET }}
        run: |
          curl --fail-with-body --silent --show-error \
            --request POST \
            --url "${API_URL}/경로" \
            --header "x-cron-secret: ${CRON_SECRET}"
```
