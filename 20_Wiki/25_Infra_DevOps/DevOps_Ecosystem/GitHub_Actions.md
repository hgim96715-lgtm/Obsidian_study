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
tags:
  - DevOps
  - GitHub
related:
  - "[[00_DevOps_Ecosystem_HomePage]]"
  - "[[Deploy_CloudMVP]]"
  - "[[NestJS_Scheduling]]"
  - "[[NestJS_Controller]]"
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
