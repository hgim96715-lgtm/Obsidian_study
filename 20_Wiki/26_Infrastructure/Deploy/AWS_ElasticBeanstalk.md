---
aliases:
  - Elastic Beanstalk
  - EB
  - AWS EB
tags:
  - AWS
  - Deploy
related:
  - "[[00_Deploy_HomePage]]"
  - "[[AWS_Concept]]"
  - "[[AWS_IAM]]"
  - "[[AWS_RDS]]"
---

# AWS_ElasticBeanstalk — Elastic Beanstalk 배포

```txt
Elastic Beanstalk = 코드만 올리면 인프라 자동 구성
EC2 / 로드밸런서 / 오토스케일링 / 배포 자동 처리
개발자는 인프라 몰라도 배포 가능
```

---

---

# Lightsail vs Elastic Beanstalk ⭐️

```txt
Lightsail
  단일 서버 / 고정 요금 / 간단한 설정
  직접 SSH 접속해서 배포

Elastic Beanstalk
  코드 업로드만 하면 자동 배포
  오토스케일링 / 로드밸런서 자동 구성
  RDS / VPC 등 AWS 서비스와 통합 쉬움
  → 실무 서비스에 적합
```

---

---

# 사전 준비 — IAM 역할 확인 ⭐️

```bash
EB 생성 전 IAM 역할 2개가 있어야 함

① aws-elasticbeanstalk-service-role   ← EB 자체 서비스 역할
② aws-elasticbeanstalk-ec2-role       ← EC2 인스턴스 프로파일

# → [[AWS_IAM]] 에서 생성한 역할 확인
```

---

---

# 애플리케이션 생성 순서 ⭐️

## 1단계 — 애플리케이션 생성

```txt
AWS 콘솔 → Elastic Beanstalk
→ 애플리케이션 생성

  애플리케이션 이름: NestJS-Netflix-EB
                     EB = Elastic Beanstalk (구분용)
```

## 2단계 — 환경 생성

```txt
애플리케이션 생성 후 → 환경 생성

  환경 이름: NestJS-Netflix (자동 생성 또는 직접 입력)
```

## 3단계 — 플랫폼 선택

```txt
플랫폼:         Node.js
플랫폼 브랜치:  Node.js 최신 버전
                ↑ NestJS 는 Node.js 서버이므로 Node.js 선택
```

## 4단계 — 사전 설정 선택

```txt
고가용성    로드밸런서 + 오토스케일링 (운영 서비스)
단일 인스턴스  서버 1대 (개발·학습용) ← 지금은 이것 선택

⚠️ 고가용성은 로드밸런서 비용 추가 발생
```

## 5단계 — 서비스 액세스 구성 ⭐️ (중요)

```txt
서비스 역할:
  기존 역할 사용 → aws-elasticbeanstalk-service-role
  없으면 역할 생성 버튼 클릭 → 자동 생성

EC2 인스턴스 프로파일: ⭐️ 반드시 확인
  aws-elasticbeanstalk-ec2-role 선택됐는지 꼭 확인
  → 안 되어 있으면 배포 실패
```

## 6단계 — VPC 설정

```txt
VPC:        기본 VPC 선택
인스턴스 서브넷: 전체 선택 (가용 영역 다중화)
```

## 7단계 — 인스턴스 트래픽 및 크기 조정 ⭐️

```txt
인스턴스 유형 선택:

  t3.micro    프리 티어 / 무료
              ⚠️ 메모리 1GB → NestJS 빌드 중 메모리 부족으로 실패할 수 있음

  t3.small    메모리 2GB / 약간의 비용
              NestJS 배포 시 권장

  t3.medium   메모리 4GB / 중규모 서비스
```

```txt
오토스케일링 그룹이란:
  트래픽이 늘어나면 인스턴스를 자동으로 늘림
  트래픽이 줄어들면 인스턴스를 자동으로 줄임
  단일 인스턴스 모드에서는 오토스케일링 없음

단일 인스턴스 (Single Instance):
  서버 1대로 운영
  비용 낮음 / 학습·개발용
  다운타임 발생 가능 (서버 1대이므로)

고가용성 (High Availability):
  로드밸런서 + 최소 2대 인스턴스
  한 서버 장애 시 다른 서버로 트래픽 이동
  운영 서비스 권장 / 비용 높음
```

## 8단계 — 검토 & 생성

```txt
설정 검토 후 제출
→ 환경 생성 완료까지 약 5~10분 소요
→ 상태가 Ok (초록) 되면 배포 준비 완료
```

---

---

# ⚠️ 자주 하는 실수

```txt
EC2 인스턴스 프로파일 미선택:
  aws-elasticbeanstalk-ec2-role 이 선택 안 되어 있으면
  EC2 가 S3 / CloudWatch 접근 불가 → 배포 실패

t3.micro 메모리 부족:
  NestJS pnpm build 실행 중 메모리 초과
  → 배포 실패 / 앱 미실행
  → t3.small 이상 권장

서브넷 미선택:
  인스턴스 서브넷을 전체 선택해야
  가용 영역 분산 가능
```

---

---

# 핵심 흐름 정리

```txt
IAM → aws-elasticbeanstalk-ec2-role 생성 확인
    ↓
EB → 애플리케이션 생성 (NestJS-Netflix-EB)
    ↓
환경 생성 → Node.js / 단일 인스턴스
    ↓
서비스 역할: aws-elasticbeanstalk-service-role
EC2 프로파일: aws-elasticbeanstalk-ec2-role ← 반드시 확인 ⭐️
    ↓
VPC: 기본 / 서브넷: 전체 선택
    ↓
인스턴스: t3.small (메모리 부족 방지)
    ↓
생성 완료 → 상태 Ok 확인
    ↓
코드 업로드 → 배포
```

---
---
# 코드 배포 ⭐️

## 1. main.ts — PORT 환경변수 설정

```typescript
// ✅ EB 는 PORT 환경변수로 포트를 전달함
// process.env.PORT 없으면 EB 가 앱을 찾지 못해 502 발생
await app.listen(process.env.PORT || 3001);
//               ↑ EB 가 주입하는 포트 / 없으면 로컬용 3001
```

```txt
EB 내부 동작:
  EB 가 nginx 를 통해 특정 포트로 트래픽 전달
  앱이 그 포트에서 실행되지 않으면 → 502 Bad Gateway
  → process.env.PORT 반드시 포함
```

## 2. Procfile — 실행 커맨드 지정

```txt
Procfile = EB 가 앱을 어떻게 실행할지 알려주는 파일
프로젝트 최상위 루트에 위치 (확장자 없음)
```

```txt
web: node dist/main.js
```

```txt
web:       웹 서버 프로세스임을 선언
node dist/main.js  빌드된 파일 실행

⚠️ Procfile 없으면 EB 가 기본 실행 커맨드로 시도
   NestJS 는 dist/main.js 로 실행해야 하므로 명시 필수

실행 커맨드를 바꾸고 싶을 때:
  Procfile 의 node dist/main.js 수정
```

## 3. 빌드


```bash
pnpm build
# → dist/ 폴더 생성
```

## 4. 압축 — node_modules 절대 포함 금지 ⭐️

bash

```bash
zip -r deploy.zip \
  package.json \
  pnpm-lock.yaml \
  dist \
  Procfile \
  -x "*.DS_Store" \
  -x "__MACOSX/*"
```

```txt
포함할 것:
  package.json      의존성 목록
  pnpm-lock.yaml    정확한 버전 고정
  dist/             빌드 결과물
  Procfile          실행 커맨드

⚠️ 절대 포함하면 안 되는 것:
  node_modules/     용량 너무 큼 → 업로드 실패 / 배포 실패
                    EB 가 package.json 보고 직접 설치함

압축 파일 내용 확인:
  unzip -l deploy.zip | head -30
```

## 5. 환경 속성 설정 (환경변수)

```txt
EB 콘솔 → 환경 클릭 → 구성(Configuration)
→ 업데이트, 모니터링 및 로깅 → 편집
→ 환경 속성

로컬 .env 파일의 내용을 여기에 입력
  DB_HOST     = ... RDS에서 생성한 엔드포인트 
  DB_PORT     = 5432
  DB_USER     = ...
  DB_PASSWORD = ...
  DB_DATABASE = ...
  JWT_SECRET  = ...

저장 후 환경 재시작
```

## 6. 업로드 & 배포

```txt
EB 콘솔 → 환경 클릭
→ 업로드 및 배포
→ deploy.zip 첨부
→ 배포 클릭

배포 완료 후 상태 확인:
  Ok (초록)  → 정상
  Degraded   → 앱 실행 오류 → 로그 확인
```

---

---

# ⚠️ 502 Bad Gateway 원인 & 해결 ⭐️

```txt
502 Bad Gateway = nginx 가 앱으로 요청을 전달했는데 응답 없음

원인 1 — process.env.PORT 누락
  앱이 EB 가 기대하는 포트에서 실행 안 됨
  → main.ts 에 process.env.PORT 추가

원인 2 — Procfile 누락 또는 오류
  EB 가 앱 실행 커맨드를 모름
  → Procfile 에 web: node dist/main.js 작성

원인 3 — node_modules 포함
  압축 파일이 너무 커서 배포 실패
  → zip 에서 node_modules 제외

원인 4 — 환경 속성 누락
  DB 연결 정보 없음 → 앱 시작 중 에러
  → EB 구성에서 환경 속성 입력

원인 5 — dist/ 폴더 없음
  pnpm build 안 하고 압축
  → 빌드 후 압축

로그 확인:
  EB 콘솔 → 로그 → 최근 100줄 요청
  → 에러 메시지 확인
```