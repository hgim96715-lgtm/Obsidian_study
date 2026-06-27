---
aliases:
  - CI/CD
  - 지속적 통합
  - 지속적 배포
tags:
  - Git
  - Deploy
related:
  - "[[00_Deploy_HomePage]]"
  - "[[GitHub_Deploy]]"
  - "[[Git_GitHubActions]]"
---

# CICD_Concept — CI/CD 개념

```txt
CI/CD = 코드 변경 → 자동 빌드 → 자동 테스트 → 자동 배포
사람이 직접 빌드/배포하지 않아도 되는 자동화 파이프라인
```

---

---

# 기존 배포 방식의 문제

```txt
수동 배포 흐름:
  코드 작성 → 로컬 빌드 → zip 압축 → S3 업로드 → EB 배포
  → 매번 사람이 직접 해야 함
  → 실수 가능성 / 느림 / 배포 타이밍 불일치

팀 협업 시:
  여러 명이 코드 변경 → 누가 언제 배포했는지 추적 어려움
  배포 전 테스트 빠뜨릴 수 있음
```

---

---

# CI — Continuous Integration ⭐️

```txt
지속적 통합 (CI):
  코드를 자주 통합하고 자동으로 빌드 · 테스트

  개발자가 코드를 push 하면:
    → 자동으로 빌드 실행
    → 자동으로 테스트 실행
    → 문제 있으면 즉시 알림

  목적:
    버그를 빨리 발견
    "내 컴퓨터에서는 됐는데" 문제 방지
    팀 전체가 항상 동작하는 코드를 유지
```

---

---

# CD — Continuous Delivery / Deployment ⭐️

```txt
지속적 배포 (CD):
  CI 통과 후 자동으로 서버에 배포

  Continuous Delivery  → 배포 준비까지 자동 / 실제 배포는 수동 승인
  Continuous Deployment → 승인 없이 자동 배포까지

  목적:
    배포 과정 자동화 → 사람의 실수 제거
    빠른 릴리즈 사이클 → 사용자에게 빠르게 기능 전달
```

---

---

# CI/CD 파이프라인 흐름 ⭐️

```txt
개발자 코드 push (main 브랜치)
    ↓
GitHub Actions 트리거
    ↓
① Checkout    코드 가져오기
② Node.js 설치
③ 의존성 설치  npm i
④ 빌드        npm run build → dist/ 생성
⑤ 테스트      (옵션)
⑥ zip 압축    배포 파일 생성
⑦ S3 업로드   deployment.zip → S3 버킷
⑧ EB 배포     S3 → Elastic Beanstalk 환경에 새 버전 적용
    ↓
자동 배포 완료 → 서버에 새 코드 반영
```

---

---

# 도구

```bash
GitHub Actions  → CI/CD 파이프라인 실행 (이 프로젝트에서 사용)
Jenkins         → 자체 서버에 설치해서 사용
GitLab CI       → GitLab 내장 CI/CD
CircleCI        → 클라우드 기반

# → [[GitHub_Deploy]] 참고
```