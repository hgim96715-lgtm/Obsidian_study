---
aliases:
  - IAM
  - AWS 권한
  - IAM 역할
tags:
  - AWS
  - Deploy
related:
  - "[[00_Deploy_HomePage]]"
  - "[[AWS_Concept]]"
---

# AWS_IAM — IAM 권한 관리

```txt
IAM = Identity and Access Management
누가 / 어떤 AWS 리소스에 / 어떤 행동을 할 수 있는지 제어
```

---

---

# IAM 핵심 개념 ⭐️

```txt
사용자 (User)
  AWS 콘솔에 로그인하는 개별 계정
  사람 또는 애플리케이션

그룹 (Group)
  사용자를 묶어서 한 번에 권한 부여

역할 (Role) ⭐️
  EC2 / Lambda 같은 AWS 서비스에 부여하는 권한
  사람이 아니라 서비스가 다른 서비스를 사용할 때 필요

정책 (Policy)
  어떤 행동을 허용/거부하는지 정의한 JSON 문서
  사용자 / 그룹 / 역할에 연결

최소 권한 원칙:
  필요한 권한만 부여
  루트 계정은 일상적으로 사용하지 않음
```

---

---

# Elastic Beanstalk EC2 역할 생성 ⭐️

```txt
Elastic Beanstalk 로 앱을 배포하면
EC2 인스턴스가 S3 / CloudWatch 등 AWS 서비스에 접근해야 함
→ EC2 에 역할(Role) 을 붙여줘야 권한이 생김
```

## 역할 생성 순서

```txt
IAM 콘솔 → 역할(Roles) → 역할 생성
```

## 1단계 — 신뢰할 수 있는 엔터티 선택

```txt
엔터티 유형: AWS 서비스
사용 사례:   EC2
           ↑ EC2 인스턴스가 이 역할을 사용
```

## 2단계 — 권한 정책 추가 ⭐️

```txt
검색해서 아래 3가지 정책 선택:

  AWSElasticBeanstalkMulticontainerDocker
    Docker 컨테이너 기반 Beanstalk 환경 권한

  AWSElasticBeanstalkWebTier
    웹 서버 계층 (HTTP 요청 처리) 권한
    → S3 로그 업로드 / CloudWatch 모니터링

  AWSElasticBeanstalkWorkerTier
    워커 계층 (백그라운드 작업) 권한
    → SQS 메시지 처리
```

## 3단계 — 역할 이름 지정 & 생성

```txt
역할 이름: aws-elasticbeanstalk-ec2-role
            ↑ Beanstalk 환경 생성 시 자동으로 찾는 기본 이름
              다른 이름 쓰면 수동으로 연결해야 함

검토 및 생성 클릭 → 역할 생성 완료
```

## 생성 후 권한 정책 확인

|정책 이름|유형|역할|
|---|---|---|
|`AWSElasticBeanstalkMulticontainerDocker`|AWS 관리형|Docker 환경|
|`AWSElasticBeanstalkWebTier`|AWS 관리형|웹 서버 계층|
|`AWSElasticBeanstalkWorkerTier`|AWS 관리형|워커 계층|

---

---

# 역할이 필요한 이유 ⭐️

```txt
Beanstalk 으로 앱 배포 시:
  EC2 인스턴스가 자동으로 생성됨
  이 EC2 가 S3 에 로그를 올리거나 CloudWatch 에 지표를 보내야 함
  → 권한 없으면 접근 거부 → 배포 실패 또는 로그 없음

역할을 만들어두면:
  Beanstalk 환경 생성 시 aws-elasticbeanstalk-ec2-role 자동 연결
  EC2 가 필요한 AWS 서비스에 자유롭게 접근 가능
```

---

---

# IAM 정책 종류

|종류|설명|
|---|---|
|AWS 관리형|AWS 가 만들어둔 정책 / 권장 / 업데이트 자동|
|고객 관리형|직접 만든 정책 / 세밀한 제어 가능|
|인라인 정책|사용자·역할에 직접 붙이는 정책 / 재사용 불가|

```txt
실무에서는:
  AWS 관리형 정책으로 시작
  세밀한 제어가 필요하면 고객 관리형 정책 생성
```

---

---

# 핵심 흐름 정리

```txt
IAM → 역할 → 역할 생성
    ↓
엔터티: AWS 서비스 → EC2
    ↓
정책 3개 추가:
  AWSElasticBeanstalkMulticontainerDocker
  AWSElasticBeanstalkWebTier
  AWSElasticBeanstalkWorkerTier
    ↓
역할 이름: aws-elasticbeanstalk-ec2-role
    ↓
생성 완료 → Beanstalk 환경 생성 시 자동 연결
```