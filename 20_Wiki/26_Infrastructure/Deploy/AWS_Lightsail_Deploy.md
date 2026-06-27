---
aliases:
  - Lightsail 배포
  - PM2
  - nginx
tags:
  - Deploy
  - AWS
related:
  - "[[00_Deploy_HomePage]]"
  - "[[AWS_Lightsail]]"
  - "[[AWS_Lightsail_Database]]"
---

# AWS_Lightsail_Deploy — PM2 & nginx 배포

```txt
PM2   → NestJS 앱을 백그라운드에서 안정적으로 실행
nginx → 80 포트로 들어온 요청을 3001 로 프록시
```

---

---

# PM2 — 백그라운드 프로세스 실행 ⭐️

```txt
PM2 없이 실행하면:
  node dist/main.js  → 터미널 닫으면 앱 종료
  → 사용자가 접속 불가

PM2 로 실행하면:
  터미널 닫아도 계속 실행
  서버 재시작 시 자동으로 앱 재시작
  크래시 시 자동 복구
```

## 앱 빌드 & 실행

```bash
# 1. TypeScript 빌드
pnpm build
# → dist/ 폴더 생성

# 2. PM2 로 실행
pm2 start dist/main.js --name netflix
#                              ↑ 앱 이름 지정 (관리용)
```

## PM2 주요 명령어

```bash
pm2 list                       # 실행 중인 앱 목록 + 상태
pm2 logs netflix               # 앱 로그 확인
pm2 logs netflix --lines 50    # 최근 50줄 로그

pm2 restart netflix            # 앱 재시작 (코드 업데이트 후)
pm2 stop netflix               # 앱 중지
pm2 delete netflix             # 앱 PM2 목록에서 제거

pm2 save                       # 현재 실행 목록 저장
pm2 startup                    # 서버 재부팅 시 자동 시작 등록

pm2 monit                      # 실시간 CPU / 메모리 / 로그 모니터링 ⭐️
```

## pm2 monit — 실시간 모니터링 ⭐️

```txt
pm2 monit 실행 시 보이는 것:
  CPU 사용률       앱이 CPU 를 얼마나 쓰는지
  Memory 사용량    앱이 메모리를 얼마나 쓰는지
  실시간 로그      앱 출력 로그 실시간 확인
  프로세스 상태    online / stopped / errored

종료: q 또는 Ctrl+C
```

## 이름 변경 — restart --name ⭐️

```bash
# id(숫자) 로 재시작하면서 이름 변경 , pm2 list에서 id 확인 
pm2 restart 0 --name nestjs-project-pipeline
#           ↑             ↑
#          앱 id        새로운 이름

# 변경 후 저장 (재부팅 시에도 유지)
pm2 save

# 확인
pm2 list
```

```txt
pm2 list 에서 id 확인:
  id  name     status  cpu  mem
  0   netflix  online  0%   50mb

pm2 restart netflix --name new-name 처럼 이름으로도 가능
하지만 id 로 하는 게 더 확실함

이름 변경 후 pm2 save 필수:
  안 하면 재부팅 시 이전 이름으로 돌아옴
```

```txt
코드 업데이트 후 재배포 순서:
  git pull origin main
  pnpm build
  pm2 restart netflix
```

## 서버 재부팅 시 자동 시작 ⭐️

```bash
# 현재 pm2 목록 저장
pm2 save

# 재부팅 시 자동 시작 스크립트 생성
pm2 startup

# → 출력된 명령어를 복사해서 실행 (sudo ... 형태)
```

```txt
pm2 startup 실행 후 출력되는 명령어를 그대로 복사·실행
그래야 서버 재부팅 후에도 앱이 자동으로 올라옴
```

---

---

# nginx — 리버스 프록시 설정 ⭐️

```txt
왜 nginx 를 쓰나:
  NestJS 는 3001 포트에서 실행
  사용자는 http://IP (80 포트) 로 접속
  → nginx 가 80 으로 받은 요청을 3001 로 전달

  직접 :3001 붙이지 않아도 됨
  나중에 HTTPS / 도메인 연결도 nginx 에서 처리
```

## nginx 설치

```bash
sudo yum install nginx -y

# 시작
sudo systemctl start nginx

# 부팅 시 자동 시작
sudo systemctl enable nginx

# 상태 확인
sudo systemctl status nginx
```

## nginx 프록시 설정

```bash
# 설정 파일 열기
sudo nano /etc/nginx/conf.d/app.conf
```

```nginx
server {
    listen 80;              # 80 포트(HTTP) 로 들어오는 요청 수신
    server_name _;          # 도메인 없을 때 _ = 모든 요청 받기
                            # 도메인 있으면: server_name example.com;

    location / {
        proxy_pass http://localhost:3001;           # 요청을 NestJS(3001) 로 전달
        proxy_http_version 1.1;                     # HTTP/1.1 사용 (WebSocket 지원)
        proxy_set_header Upgrade $http_upgrade;     # WebSocket 업그레이드 헤더 전달
        proxy_set_header Connection 'upgrade';      # 연결 유지 / WebSocket 용
        proxy_set_header Host $host;                # 원본 Host 헤더 그대로 전달
        proxy_cache_bypass $http_upgrade;           # WebSocket 요청은 캐시 우회
    }
}
```

```bash
# 설정 문법 확인
sudo nginx -t

# nginx 재시작 (설정 적용)
sudo systemctl restart nginx
```

```txt
proxy_pass http://localhost:3001:
  80 으로 들어온 요청을 3001 로 전달

nginx 적용 후:
  http://43.202.2.225      → 접속 가능 (:80 생략)
  http://43.202.2.225:3001 → 직접 접속도 가능
  http://43.202.2.225/v1/movie -> 이렇게 가능

Firewall 에서 80 포트 열려있는지 확인
  Lightsail Networking → IPv4 Firewall → HTTP TCP 80 있어야 함
```

## nginx 설정 후 Postman 환경 변수 업데이트

```txt
host: http://43.202.2.225
     ↑ :3001 없이 80 포트로 접속
```

---

---

# 리소스 삭제 ⭐️

```txt
Lightsail 은 월 고정 요금
사용하지 않는 리소스는 삭제해야 비용 발생 안 함

삭제 순서:
  ① Static IP 해제 (인스턴스에 붙어 있으면 먼저 분리)
  ② 인스턴스 삭제
  ③ Database 삭제
  ④ Storage 삭제
```

### 인스턴스 삭제

```txt
Lightsail 콘솔 → Instances
→ 인스턴스 오른쪽 점 3개 메뉴 → Delete
→ "Delete" 입력 후 확인

⚠️ 삭제하면 서버 데이터 전부 사라짐
   .env / 업로드 파일 등 필요한 것 미리 백업
```

### Database 삭제

```txt
Lightsail 콘솔 → Databases
→ DB 클릭 → Delete database
→ 스냅샷 생성 여부 선택 후 삭제

⚠️ DB 삭제 전 데이터 백업 필요시 스냅샷 생성 선택
```

### Static IP 삭제

```txt
Lightsail 콘솔 → Networking → Static IPs
→ Static IP 가 인스턴스에 연결돼 있으면 먼저 Detach
→ Delete

⚠️ Static IP 는 인스턴스에 연결 안 된 상태에서도 요금 발생
   인스턴스 삭제 후 반드시 Static IP 도 삭제
```

---

---

# 전체 배포 흐름 정리

```txt
git clone  →  pnpm i  →  nano .env
    ↓
pnpm build
    ↓
pm2 start dist/main.js --name netflix
    ↓
pm2 save  →  pm2 startup  (재부팅 자동 시작 등록)
    ↓
sudo yum install nginx -y
sudo nano /etc/nginx/conf.d/app.conf  (proxy_pass 3001)
sudo systemctl restart nginx
    ↓
Lightsail Networking → Firewall → TCP 80 열려있는지 확인
    ↓
http://43.202.2.225 접속 확인
```