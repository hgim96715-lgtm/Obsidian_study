
```txt
개발된 소프트웨어를 사용자나 클라이언트가 사용할 수 있도록 환경에 올리는 것
안정적으로 운영 + 빠르게 업데이트 + 비용 효율적으로 확장
```

---

---

# 배포란 ⭐️

```txt
단순히 서버에 올리는 것이 아님
사용자가 언제든 안정적으로 접근할 수 있는 환경을 만드는 전체 과정

핵심 요소:
  안정성   시스템이 멈추지 않고 운영되는 것
  확장성   사용자가 늘어도 버틸 수 있는 것
  효율성   자원을 낭비 없이 사용하는 것
```

# 배포 전략이 필요한 이유

```txt
버그 없는 프로그램은 없다
  → 발견 즉시 수정해서 배포할 수 있어야 함
  → 느린 배포 = 사용자 이탈

사용자의 니즈는 계속 변한다
  → 빠른 업데이트로 반영해야 경쟁력 유지
  → 배포가 느리고 불안정하면 아무리 좋은 기능도 늦게 도달
```

---

---

# On-Premise vs Cloud ⭐️

| |On-Premise|Cloud|
|---|---|---|
|서버|직접 하드웨어 구매|AWS / Azure / GCP 서비스 이용|
|초기 비용|높음|낮음 (사용한 만큼)|
|확장|하드웨어 추가 구매·설치 필요|클릭 몇 번으로 즉시 확장|
|유지보수|직접 관리|클라우드가 대부분 담당|
|유연성|낮음|높음 (온디맨드)|
|잉여 자원|발생 가능성 있음|사용한 만큼만 비용|
|고가용성|직접 구축|다양한 지역 데이터 분산 기본 제공|

```txt
온디맨드(On-Demand):
  필요한 만큼만 자원을 즉시 사용하고 필요 없으면 반납
  서버를 미리 사 두지 않아도 됨
```

---

---

# 로컬 개발 환경 vs 클라우드 배포 환경 분리 ⭐️

```txt
왜 분리하나:
  로컬에서 잘 되던 게 서버에선 안 되는 상황 방지
  개발 중인 코드가 실제 운영 DB 에 영향 주지 않도록
  환경별로 설정(DB / API Key / 포트) 을 다르게 가져감
```

# 분리 배포의 장점

```txt
확장성
  서비스 구성 요소별로 독립적으로 스케일링 가능
  사용량 많은 부분만 서버를 늘릴 수 있음
  → 자원 낭비 없이 비용 효율 상승

관리 용이
  문제 발생 시 어느 서비스인지 바로 특정 가능
  한 서비스 문제가 전체로 번지지 않음

보안
  네트워크 계층에서 접근 제어 가능
  민감한 데이터(DB / 결제) 는 별도 네트워크로 분리
  필요한 서비스만 외부에 노출

마이크로서비스 아키텍처 기반
  도메인(영화 / 유저 / 결제)별로 독립 개발 + 독립 배포
  → 한 팀이 배포해도 다른 팀 서비스에 영향 없음
```

---

---

## AWS

| 노트                         | 핵심 개념                                                                         |
| -------------------------- | ----------------------------------------------------------------------------- |
| [[AWS_Concept]] ⭐          | EC2 / RDS / VPC / ELB / Route 53 / Beanstalk / Lightsail / IAM / S3           |
| [[AWS_Lightsail]]          | 인스턴스 생성 / NVM / ssh-keygen / GitHub SSH 키 / nano .env / Firewall              |
| [[AWS_Lightsail_Database]] | PostgreSQL 생성 / .env 설정 / SSL rejectUnauthorized / Networking / DataGrip      |
| [[AWS_Lightsail_Deploy]] ⭐ | PM2 실행·명령어 / nginx 프록시 / 재부팅 자동시작 / 리소스 삭제                                    |
| [[AWS_EC2]]                | 인스턴스 생성 / 보안그룹 / SSH 접속 / 배포                                                  |
| [[AWS_S3]]                 | IAM 사용자·액세스 키 / 버킷 생성 / Presigned URL / PUT 업로드 / GET 다운로드 / Access Denied    |
| [[AWS_RDS]] ⭐              | PostgreSQL 생성 / DB 식별자 / 엔드포인트 / 보안그룹 5432 / DataGrip / ssl 설정                |
| [[AWS_ElasticBeanstalk]] ⭐ | 앱·환경 생성 / Node.js 플랫폼 / 서비스 역할 / EC2 프로파일 / t3.small / 단일 vs 고가용성             |
| [[AWS_IAM]] ⭐              | 사용자·그룹·역할·정책 개념 / EC2 역할 생성 / Beanstalk 권한 3종 / aws-elasticbeanstalk-ec2-role |




```embed
title: "Amazon Web Services Sign-In"
image: "https://d1.awsstatic.com/onedam/marketing-channels/website/aws/en_US/homepage/console-sign-in/ai-implementation-2026.ded0f707cdac0eb150ecb0a172bc25918e792af6.png"
description: ""
url: "https://us-east-1.console.aws.amazon.com/console/home?nc2=h_si&region=us-east-1&src=header-signin#"
favicon: ""
aspectRatio: "78.94736842105263"
```


---

---

## CI_CD

| 노트                                 | 핵심 개념                                                                                    |
| ---------------------------------- | ---------------------------------------------------------------------------------------- |
| [[CICD_Concept]] ⭐                 | CI/CD 란 / 수동 배포 문제 / CI 자동 빌드·테스트 / CD 자동 배포 / 파이프라인 흐름                                  |
| [[GitHub_Deploy]] ⭐                | IAM 사용자 / GitHub Secrets / workflow yaml / services postgres / migration / S3 / EB 자동 배포 |
| [[GitHub_Actions_Troubleshooting]] | TS5011 / TS2564 / no encryption SSL / peer dependency / .env 없음 / public 폴더 없음           |
| [[Docker_Deploy]]                  | Dockerfile / 이미지 빌드 / 컨테이너 배포                                                            |
