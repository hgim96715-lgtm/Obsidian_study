---
aliases:
  - GitHub Actions
  - workflow
  - CI/CD 자동화
tags:
  - Git
  - Deploy
related:
  - "[[00_Deploy_HomePage]]"
  - "[[AWS_ElasticBeanstalk]]"
  - "[[AWS_S3]]"
  - "[[AWS_IAM]]"
  - "[[CICD_Concept]]"
  - "[[Git_GitHubActions]]"
---

# GitHub_Actions — GitHub Actions & EB 자동 배포

```
GitHub Actions = GitHub 에 내장된 CI/CD 도구
코드 push 시 자동으로 빌드 → S3 업로드 → EB 배포 실행
```

---

---

# 사전 준비 ⭐️

## IAM 사용자 생성 (CI/CD 전용)

```
IAM → 사용자 → 사용자 생성
  이름:  nestjs-github-cicd
  권한:  직접 정책 연결
    AdministratorAccess  또는
    AmazonS3FullAccess + ElasticBeanstalkFullAccess

→ 액세스 키 만들기
  사용 사례: AWS 외부에서 실행되는 애플리케이션
  → Access Key ID / Secret Access Key 발급 후 저장
```

## 인라인 정책으로 S3 + EB 권한 추가 ⭐️

```
사용자 생성 후 권한이 부족하면 직접 인라인 정책을 추가해야 함

IAM → 사용자 → nestjs-github-cicd 클릭
→ 권한(Permissions) 탭
→ 권한 추가 ▼ 버튼 클릭
→ 인라인 정책 생성 선택
→ JSON 탭 클릭
→ 아래 JSON 붙여넣기
→ 정책 이름 입력 후 생성
```

```json
{
  "Version": "2012-10-17",
  "Statement": [
    {
      "Sid": "S3DeploymentArtifact",
      "Effect": "Allow",
      "Action": [
        "s3:PutObject",
        "s3:GetObject",
        "s3:AbortMultipartUpload",
        "s3:ListMultipartUploadParts"
      ],
      "Resource": "arn:aws:s3:::nestjs-bucket-gong/*"
      //                                              ↑ /* = 버킷 안의 모든 객체
    },
    {
      "Sid": "S3ListBucket",
      "Effect": "Allow",
      "Action": "s3:ListBucket",
      "Resource": "arn:aws:s3:::nestjs-bucket-gong"
      //                                            ↑ /* 없음 = 버킷 자체에 대한 권한
    },
    {
      "Sid": "EBDeployPermissions",
      "Effect": "Allow",
      "Action": [
        "elasticbeanstalk:CreateApplicationVersion",
        "elasticbeanstalk:UpdateEnvironment",
        "elasticbeanstalk:DescribeEnvironments",
        "elasticbeanstalk:DescribeEvents"
      ],
      "Resource": "*"
    }
  ]
}
```

```
정책 설명:
  S3DeploymentArtifact  버킷 안 파일 업로드 / 다운로드 / 멀티파트 업로드
  S3ListBucket          버킷 목록 조회 (aws s3 cp 내부적으로 필요)
  EBDeployPermissions   EB 에 새 버전 등록 + 환경 배포 + 상태 확인

⚠️ Resource 의 버킷 이름을 실제 버킷 이름으로 변경
   nestjs-bucket-gong → 내 버킷 이름

AdministratorAccess 대신 이 인라인 정책을 쓰는 이유:
  최소 권한 원칙 — 필요한 권한만 부여
  CI/CD 용 키가 탈취돼도 피해 최소화
```

## GitHub Secrets 등록 ⭐️️

```
GitHub 레포지토리 → Settings
→ Secrets and variables → Actions
→ New repository secret
```

```
등록할 값 전체:

AWS 관련:
  AWS_ACCESS_KEY_ID      IAM 액세스 키 ID
  AWS_SECRET_ACCESS_KEY  IAM 시크릿 키
  AWS_REGION             ap-northeast-2
  AWS_S3_BUCKET          S3 버킷 이름

DB 관련:
  DB_TYPE                postgres
  DB_HOST                RDS 엔드포인트
  DB_PORT                5432
  DB_USERNAME            postgres
  DB_PASSWORD            RDS 마스터 암호
  DB_DATABASE            DB 이름

  DATABASE_URL ⭐️        Prisma 전용 — 아래 형식으로 직접 조합
                         postgresql://${DB_USERNAME}:${DB_PASSWORD}@${DB_HOST}:${DB_PORT}/${DB_DATABASE}?sslmode=no-verify

Redis 관련 ⭐️:
  REDIS_HOST             ElastiCache Primary Endpoint
                         ⚠️ localhost / 127.0.0.1 아님
                         예: my-redis.xxxxx.apn2.cache.amazonaws.com
  REDIS_PORT             6379
  REDIS_INSIGHT_PORT     5540

앱 환경변수:
  ENV                    prod
  SALT_ROUNDS            10
  ACCESS_TOKEN_SECRET    JWT 액세스 토큰 시크릿
  REFRESH_TOKEN_SECRET   JWT 리프레시 토큰 시크릿
  SESSION_SECRET         세션 서명 키

⚠️ Secrets 는 등록 후 값 확인 불가
   잘못 입력하면 삭제 후 재등록
```

## DATABASE_URL 형식 ⭐️

```
postgresql://${DB_USERNAME}:${DB_PASSWORD}@${DB_HOST}:${DB_PORT}/${DB_DATABASE}?sslmode=no-verify

sslmode=no-verify 가 중요:
  RDS 는 SSL 연결 강제
  no-verify = SSL 은 사용하되 인증서 검증 스킵
  없으면 → Prisma P1011 self-signed certificate 에러

DB_PASSWORD 에 특수문자(:, @, / 등) 있을 때 주의:
  URL 에 넣으면 파싱 오류 (P1013)
  → deploy.yml 에서 printf 로 안전하게 .env 에 기록
```

## REDIS_HOST 주의 ⭐️

```
❌ localhost / 127.0.0.1
   EB 인스턴스 안에 Redis 없음 → ECONNREFUSED

✅ ElastiCache Primary Endpoint
   AWS → ElastiCache → 클러스터 → 기본 엔드포인트 복사
   예: my-redis.xxxxx.apn2.cache.amazonaws.com
```

---

---

---

# 워크플로우 파일 구조

```
프로젝트 루트/
└── .github/
    └── workflows/
        └── deploy.yml     ← 이 파일이 Actions 트리거
```

---

---

# 워크플로우 전체 코드 ⭐️

```yaml
# .github/workflows/deploy.yml
name: Deploy to AWS Elastic Beanstalk

on:
  push:
    branches:
      - main   # main 브랜치 push 시만 실행

jobs:
  build-and-deploy:
    runs-on: ubuntu-latest

    # 모든 step 에서 공유할 환경변수 — Secrets 에서 주입
    env:
      ENV: ${{ secrets.ENV || 'prod' }}
      DB_TYPE: ${{ secrets.DB_TYPE || 'postgres' }}
      DB_HOST: ${{ secrets.DB_HOST }}
      DB_PORT: ${{ secrets.DB_PORT || '5432' }}
      DB_USERNAME: ${{ secrets.DB_USERNAME }}
      DB_PASSWORD: ${{ secrets.DB_PASSWORD }}
      DB_DATABASE: ${{ secrets.DB_DATABASE }}
      DATABASE_URL: ${{ secrets.DATABASE_URL }}         # Prisma 전용 (sslmode=no-verify 포함)
      SALT_ROUNDS: ${{ secrets.SALT_ROUNDS }}
      ACCESS_TOKEN_SECRET: ${{ secrets.ACCESS_TOKEN_SECRET }}
      REFRESH_TOKEN_SECRET: ${{ secrets.REFRESH_TOKEN_SECRET }}
      REDIS_HOST: ${{ secrets.REDIS_HOST }}             # ElastiCache endpoint (localhost 아님)
      REDIS_PORT: ${{ secrets.REDIS_PORT || '6379' }}
      REDIS_INSIGHT_PORT: ${{ secrets.REDIS_INSIGHT_PORT || '5540' }}
      SESSION_SECRET: ${{ secrets.SESSION_SECRET }}
      AWS_REGION: ${{ secrets.AWS_REGION }}
      AWS_S3_BUCKET: ${{ secrets.AWS_S3_BUCKET }}

    # TypeORM migration 용 CI 전용 로컬 PostgreSQL
    services:
      postgres:
        image: postgres:16-alpine
        env:
          POSTGRES_USER: ${{ secrets.DB_USERNAME }}
          POSTGRES_PASSWORD: ${{ secrets.DB_PASSWORD }}
          POSTGRES_DB: ${{ secrets.DB_DATABASE }}
        options: >-
          --health-cmd pg_isready
          --health-interval 10s
          --health-timeout 5s
          --health-retries 5
        ports:
          - 5432:5432

    steps:
      - name: Checkout code
        uses: actions/checkout@v6

      - name: Set up Node.js
        uses: actions/setup-node@v6
        with:
          node-version: '22'

      # CI 환경에는 .env 파일 없음 → Secrets 값으로 직접 생성
      # printf 사용 이유: DB_PASSWORD / DATABASE_URL 에 특수문자 있어도 안전하게 처리
      - name: Create Env File
        run: |
          {
            echo "ENV=${ENV:-prod}"
            echo "DB_TYPE=${DB_TYPE:-postgres}"
            echo "DB_HOST=${DB_HOST}"
            echo "DB_PORT=${DB_PORT:-5432}"
            echo "DB_USERNAME=${DB_USERNAME}"
            printf 'DB_PASSWORD=%s\n' "${DB_PASSWORD}"
            echo "DB_DATABASE=${DB_DATABASE}"
            printf 'DATABASE_URL=%s\n' "${DATABASE_URL}"
            echo "SALT_ROUNDS=${SALT_ROUNDS}"
            echo "ACCESS_TOKEN_SECRET=${ACCESS_TOKEN_SECRET}"
            echo "REFRESH_TOKEN_SECRET=${REFRESH_TOKEN_SECRET}"
            echo "REDIS_HOST=${REDIS_HOST}"
            echo "REDIS_PORT=${REDIS_PORT:-6379}"
            echo "REDIS_INSIGHT_PORT=${REDIS_INSIGHT_PORT:-5540}"
            echo "SESSION_SECRET=${SESSION_SECRET}"
            echo "AWS_REGION=${AWS_REGION}"
            echo "AWS_S3_BUCKET=${AWS_S3_BUCKET}"
          } > .env
          # 필수값 없으면 여기서 즉시 실패 (조기 검증)
          test -n "${DATABASE_URL}" && test -n "${DB_HOST}" && test -n "${DB_USERNAME}" && test -n "${DB_DATABASE}"

      # ServeStaticModule / FFmpeg 등 폴더 없으면 앱 시작 실패
      - name: Create Folders
        run: |
          mkdir -p ./public/movie
          mkdir -p ./public/temp

      - name: Install dependencies
        run: npm ci --legacy-peer-deps   # lock 파일 기반 / peer dependency 충돌 무시

      - name: Build Project
        run: npm run build

      # 빌드 결과물 검증 — 없으면 EB 502 원인이 됨
      - name: Verify build artifact
        run: |
          test -f dist/src/main.js
          test -f Procfile
          grep -q 'dist/src/main.js' Procfile
          test -f node_modules/@redis/client/dist/lib/commands/reply-utils.js

      # RDS 에 Prisma migration 적용
      # P3005 = 기존 TypeORM 테이블이 있는 경우 → baseline 처리 후 재시도
      - name: Run Prisma migrations (RDS)
        env:
          DATABASE_URL: ${{ secrets.DATABASE_URL }}
        run: |
          set +e   # P3005 실패해도 다음 줄 실행 (기본 set -e 무력화)
          npm run migration:deploy > migrate.log 2>&1
          status=$?
          cat migrate.log
          if [ "$status" -ne 0 ] && grep -q 'P3005' migrate.log; then
            echo "기존 스키마 감지 → Prisma baseline 등록..."
            npx prisma migrate resolve --applied "20260605052743_init"
            npm run migration:deploy
          elif [ "$status" -ne 0 ]; then
            exit "$status"
          fi

      - name: Zip Artifact For Deployment
        run: zip -r deployment.zip .   # node_modules 포함 → EB 재설치 불필요

      - name: Upload To S3
        env:
          AWS_ACCESS_KEY_ID: ${{ secrets.AWS_ACCESS_KEY_ID }}
          AWS_SECRET_ACCESS_KEY: ${{ secrets.AWS_SECRET_ACCESS_KEY }}
          AWS_REGION: ${{ secrets.AWS_REGION }}
          AWS_DEFAULT_REGION: ${{ secrets.AWS_REGION }}
          S3_BUCKET: ${{ secrets.AWS_S3_BUCKET }}
          VERSION_LABEL: ${{ github.sha }}-${{ github.run_number }}-${{ github.run_attempt }}
        run: |
          aws s3 cp deployment.zip "s3://${S3_BUCKET}/deployments/${VERSION_LABEL}.zip"

      # VERSION_LABEL = 커밋SHA + 실행번호 + 재시도번호 → 중복 방지
      - name: Deploy to AWS Elastic Beanstalk
        env:
          AWS_ACCESS_KEY_ID: ${{ secrets.AWS_ACCESS_KEY_ID }}
          AWS_SECRET_ACCESS_KEY: ${{ secrets.AWS_SECRET_ACCESS_KEY }}
          AWS_REGION: ${{ secrets.AWS_REGION }}
          AWS_DEFAULT_REGION: ${{ secrets.AWS_REGION }}
          S3_BUCKET: ${{ secrets.AWS_S3_BUCKET }}
          VERSION_LABEL: ${{ github.sha }}-${{ github.run_number }}-${{ github.run_attempt }}
        run: |
          S3_KEY="deployments/${VERSION_LABEL}.zip"
          aws elasticbeanstalk create-application-version \
            --application-name NestJS-Project-EB \
            --version-label "$VERSION_LABEL" \
            --source-bundle "S3Bucket=${S3_BUCKET},S3Key=${S3_KEY}"

          aws elasticbeanstalk update-environment \
            --application-name NestJS-Project-EB \
            --environment-name NestJS-Project-EB-env \
            --version-label "$VERSION_LABEL"
```

---

---

# 워크플로우 각 단계 설명 ⭐️

```
env (job 레벨):
  steps 전체에서 공유하는 환경변수
  Secrets 값을 여기서 한 번에 받아서 각 step 에서 재사용
  || 'prod' = Secret 없으면 기본값 사용

services.postgres:
  CI 환경에서 migration 실행을 위한 임시 PostgreSQL 컨테이너
  health-cmd 로 DB 준비됐는지 확인 후 다음 step 진행
  ports: 5432:5432 → localhost:5432 로 접근 가능

Create Env File:
  CI 환경에는 .env 파일이 없음 (gitignore)
  → Secrets 값으로 직접 생성
  test -n ... = 필수 값이 비어있으면 에러로 조기 종료

Create Folders:
  ServeStaticModule 등에서 폴더 없으면 앱 시작 실패
  → 빌드 전에 미리 생성

npm install --legacy-peer-deps:
  pnpm-lock.yaml 이 있어도 CI 에서는 npm 사용
  --legacy-peer-deps: peer dependency 버전 충돌 무시

Run Migration:
  package.json 의 migration:run 스크립트 실행
  CI PostgreSQL 컨테이너에 스키마 적용
  → data-source.ts 에 SSL 조건 처리 필요 (트러블슈팅 참고)

$GITHUB_SHA:
  현재 커밋 해시 → 버전 레이블
  어떤 커밋이 EB 에 배포됐는지 추적 가능
```

---

---

# ⚠️ application-name / environment-name 주의 ⭐️

```yaml
--application-name NestJS-Project-EB      ← EB 애플리케이션 이름과 정확히 일치
--environment-name NestJS-Project-EB-env  ← EB 환경 이름과 정확히 일치
```

```
EB 콘솔에서 이름 확인:
  애플리케이션 이름: EB 콘솔 왼쪽 목록
  환경 이름:         애플리케이션 클릭 후 환경 목록

이름이 다르면 배포 실패
→ 콘솔에서 정확히 복사해서 사용
```

---

---

# 실행 확인

```
GitHub → 레포지토리 → Actions 탭
→ 워크플로우 실행 목록
→ 각 step 클릭 → 로그 확인

성공:  초록 체크 ✅
실패:  빨간 X ❌ → 해당 step 클릭해서 에러 확인
```

---

---

# 핵심 흐름 정리

```
IAM → nestjs-github-cicd 사용자 생성 → 액세스 키 발급
    ↓
GitHub Secrets 에 4개 등록
  AWS_ACCESS_KEY_ID / AWS_SECRET_ACCESS_KEY / AWS_REGION / AWS_S3_BUCKET
    ↓
.github/workflows/deploy.yml 파일 생성
    ↓
main 브랜치에 push
    ↓
GitHub Actions 자동 실행:
  checkout → .env 생성 → 폴더 생성 → npm install
  → build → migration:run → zip → S3 업로드 → EB 배포
    ↓
EB 환경에 새 버전 자동 반영
```

→ 오류 모음: [[GitHub_Actions_Troubleshooting]]