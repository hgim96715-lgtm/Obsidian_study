---
aliases:
  - Lightsail
  - AWS Lightsail
tags:
  - AWS
  - Deploy
related:
  - "[[00_Deploy_HomePage]]"
  - "[[AWS_Concept]]"
  - "[[Git_SSH]]"
---

# AWS_Lightsail — 라이트세일

```txt
소규모 프로젝트 / 간단한 웹 앱을 빠르게 시작할 수 있는 서비스
가상 서버 + 스토리지 + 데이터 전송을 하나의 패키지로 제공
월별 고정 요금 → 비용 예측 쉬움
```

---

---

# EC2 vs Lightsail ⭐️

```txt
EC2
  세부 설정 가능 / 복잡 / 유연 / 대규모 서비스 적합

Lightsail
  간단 / 고정 요금 / 소규모·개인 프로젝트 적합
  복잡한 VPC / 보안그룹 설정 없이 빠르게 시작 가능
```

---

---

# Lightsail 인스턴스 생성 순서

```txt
1. AWS 콘솔 → Lightsail 검색 → 접속
2. 인스턴스 생성 클릭
3. 리전 선택 (서울: ap-northeast-2)
4. 플랫폼 선택 → Linux/Unix
5. 블루프린트 선택 → OS Only → Amazon Linux 2
6. 플랜 선택 → 월 고정 요금 (5달러~ 시작)
7. 인스턴스 이름 지정 → 생성
```

---

---

# Git 설치 ⭐️

## Lightsail 인스턴스 접속 후 Git 설치.

```bash
sudo yum install git -y
```

```bash
# 설치 확인
git --version
```

---

---

# SSH 키 생성 → GitHub 등록 ⭐️

```txt
Lightsail 에서 GitHub 에 SSH 로 접속하려면
Lightsail 서버의 공개키(public key) 를 GitHub 에 등록해야 함
```

## 0. 기존 키 확인 먼저


```bash
ls ~/.ssh/
```

```txt
id_rsa      → 개인키 (비밀 / 절대 공유 X)
id_rsa.pub  → 공개키 (GitHub 에 등록하는 것)

파일이 있으면 → 새로 생성 안 해도 됨
파일이 없으면 → 아래에서 새로 생성
```

## 1. SSH 키 생성 (기존 키 없을 때)


```bash
ssh-keygen
```

```txt
Generating public/private rsa key pair.
Enter file in which to save the key (/home/ec2-user/.ssh/id_rsa):
→ 엔터 (기본 경로 사용)

Enter passphrase (empty for no passphrase):
→ 엔터 (비밀번호 없이 사용)

Enter same passphrase again:
→ 엔터
```

```txt
passphrase 란:
  키를 사용할 때마다 입력하는 비밀번호
  설정하면 보안이 높아지지만 git pull / push 할 때마다 입력해야 함
  서버 자동 배포 환경에서는 불편 → 엔터로 건너뛰는 경우가 많음

  엔터 (비밀번호 없음)  → git 사용 시 비밀번호 입력 불필요
  비밀번호 설정        → git 사용할 때마다 비밀번호 입력 필요
```

## 2. 공개키 확인

```bash
cat /home/ec2-user/.ssh/id_rsa.pub
```

```txt
ssh-rsa AAAAB3NzaC1yc2EAAA... ec2-user@ip-xxx
↑ 이 전체 내용을 복사
```

## 3. GitHub 에 SSH 키 등록

```txt
GitHub → Settings → SSH and GPG keys
→ New SSH key 클릭
→ Title: aws_lightsail  (알아보기 쉬운 이름)
→ Key: 복사한 공개키 붙여넣기
→ Add SSH key
```

## 4. 연결 확인

```bash
ssh -T git@github.com
# Hi {username}! You've successfully authenticated 이 나오면 성공
```

---

---

# Git 클론 & 배포

```bash
# SSH 방식으로 클론
git clone git@github.com:{username}/{repo}.git

# 이후 업데이트
cd {repo}
git pull origin main
```

```txt
HTTPS 방식은 매번 토큰 입력 필요
SSH 방식은 키 한 번 등록하면 이후 인증 없이 pull / push 가능
```

---

---

# NestJS 배포 준비 ⭐️

```txt
Lightsail 은 Amazon Linux 기반
Node.js / pnpm / PM2 를 직접 설치해야 함
```

```bash
# 1. Node.js 설치
sudo yum install nodejs -y

# 설치 확인
node -v

# 2. pnpm 설치
sudo npm i -g pnpm

# 3. 프로젝트 의존성 설치
cd {repo}
pnpm i

# 4. PM2 설치 (프로세스 매니저)
sudo npm install pm2 -g
```

```txt
PM2 란:
  Node.js 앱을 백그라운드에서 계속 실행해주는 프로세스 매니저
  서버가 꺼져도 자동 재시작 / 크래시 시 자동 복구
  → 단순히 node main.js 로 실행하면 터미널 종료 시 앱도 종료됨
  → PM2 로 실행하면 터미널 닫아도 계속 실행됨

node main.js       터미널 닫으면 종료 ❌
pm2 start main.js  터미널 닫아도 계속 실행 ✅
```

---
---
# env 파일 서버에 만들기 ⭐️

```txt
.env 는 .gitignore 에 등록 → Git 에 올라가지 않음
→ git clone 해도 서버에 .env 없음
→ 서버에서 직접 만들어야 함
```

## Lightsail 브라우저 터미널 접속

```txt
AWS Lightsail 콘솔 → Instances → 인스턴스 클릭
→ Connect 탭 → Connect using SSH 클릭
→ 브라우저에서 터미널 바로 열림
```

## nano 로 .env 파일 생성

```bash
# 프로젝트 폴더로 이동
cd {repo}

# nano 편집기로 .env 파일 생성
nano .env
```

```txt
nano 편집기가 열리면:
  로컬 .env 파일 내용을 전체 복사 → 터미널에 붙여넣기
  (Ctrl+V 또는 마우스 오른쪽 클릭 → 붙여넣기)

저장 후 종료:
  Ctrl + O   → 저장 (파일명 확인 후 엔터)
  Ctrl + X   → 종료
```

```bash
# 잘 만들어졌는지 확인
cat .env
```

```txt
⚠️ .env 에 민감한 정보(DB 비밀번호 / JWT 시크릿) 가 포함됨
   cat .env 결과를 화면에 그대로 두지 않도록 주의
   확인 후 터미널 clear 권장
```

----
---

# 핵심 흐름 정리

```txt
Lightsail 인스턴스 생성
    ↓
sudo yum install git -y
    ↓
ssh-keygen  →  공개키 생성
    ↓
cat /home/ec2-user/.ssh/id_rsa.pub  →  공개키 복사
    ↓
GitHub → Settings → SSH keys → aws_lightsail 이름으로 등록
    ↓
git clone git@github.com:...  →  서버에 코드 클론
    ↓
sudo yum install nodejs -y
sudo npm i -g pnpm
pnpm i
sudo npm install pm2 -g
    ↓
PM2 로 NestJS 앱 실행
```
---
---
# Networking — 포트 열기 & 배포 확인 ⭐️

## Firewall 규칙 추가

```txt
Lightsail 콘솔 → Instances → 인스턴스 클릭
→ Networking 탭 → IPv4 Firewall
→ Add rule

  Application : Custom
  Protocol    : TCP
  Port        : 3001          ← NestJS 기본 포트
  Restricted to: Any IPv4 address

→ Create
```

```txt
기본으로 열려있는 포트:
  SSH   TCP 22   Lightsail 브라우저 SSH 접속용
  HTTP  TCP 80   웹 HTTP 트래픽

NestJS 앱 포트(3001) 는 기본 닫혀있음
→ 반드시 Firewall 에서 열어줘야 외부에서 접속 가능
```

## Public IP 확인

```txt
Lightsail 콘솔 → Instances → 인스턴스 클릭
→ 상단 Public IP 복사

예시: 43.202.2.225
```

## Postman 에서 확인 ⭐️

```txt
Postman → Environments → 기존 환경 선택
→ host 값 변경

  기존: http://localhost:3001
  변경: http://43.202.2.225:3001
         ↑ Lightsail Public IP

→ Save → API 요청 전송 → 응답 확인
```

```txt
⚠️ Lightsail 재시작 시 Public IP 가 바뀔 수 있음
   고정 IP 가 필요하면 Lightsail → Networking → Static IP 할당
```

---
---
# ⚠️ 트러블슈팅 — Node 버전 이슈

## crypto is not defined ⭐️

```txt
에러:
  ReferenceError: crypto is not defined

원인:
  @nestjs/typeorm 이 전역 crypto 를 사용
  Node 18 에서는 전역 crypto 가 기본 비활성화
  Node 20+ 에서는 전역 crypto 가 기본 활성화

  로컬 Node 22 → 문제 없음
  EC2 Node 18  → 에러 발생
  → 로컬에서 잘 되던 코드가 EC2 에서만 에러 나는 이유

⚠️ Lightsail 브라우저 터미널에서도 동일하게 설치해야 함
   sudo yum install nodejs 로 설치된 버전이 18 이면 같은 에러 발생
```

### 해결법 1 — NVM 으로 Node 20 설치 (강력 권장) ⭐️

```txt
NVM 사용 시 장점:
  버전 관리가 쉬움 → 나중에 버전 변경도 명령어 한 줄
  sudo 없이 npm install -g 사용 가능 → 보안상 안전
```

```bash
# 1단계 — NVM 설치
curl -o- https://raw.githubusercontent.com/nvm-sh/nvm/v0.40.1/install.sh | bash

# 2단계 — NVM 활성화 (터미널 재시작 효과)
. ~/.bashrc

# 3단계 — Node.js 20 설치
nvm install 20

# 버전 확인
node -v
npm -v
```

```txt
설치 후 확인:
  node -v  → v20.x.x 나오면 성공

which node 로 어떤 node 가 실행되는지 확인
PM2 가 예전 node 바이너리를 쓰고 있으면:
  pm2 restart all  또는  pm2 delete all → pm2 start 다시
```

###  해결법 2 — polyfill 추가 (Node 18 유지 시)

```typescript
// main.ts 맨 위에 추가
import { webcrypto } from 'crypto';

if (!globalThis.crypto) {
  (globalThis as any).crypto = webcrypto;
}
```

```txt
Node 18 을 꼭 써야 하는 경우 사용
main.ts 진입점에서 전역에 crypto 를 직접 등록
```
