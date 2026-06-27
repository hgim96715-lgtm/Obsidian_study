---
aliases:
  - 방화벽
  - ufw
  - iptables
tags:
  - Linux
related:
  - "[[00_Linux_HomePage]]"
  - "[[Linux_Network_Check]]"
  - "[[Linux_SSH_Basics]]"
---

# Linux_Firewall — 방화벽 설정

## 한 줄 요약

```
ufw      = Ubuntu 기본 방화벽 (간단)
iptables = 저수준 방화벽 (복잡하지만 강력)
firewalld = RHEL/CentOS 방화벽

서버 열기 전 반드시 SSH(22) 먼저 허용 후 enable
```

---

---

# ① ufw — Ubuntu 방화벽 ⭐️

```
ufw = Uncomplicated Firewall
  복잡한 iptables 를 쉽게 다루는 래퍼
  Ubuntu 기본 내장
```

## 기본 명령어

```bash
# 방화벽 상태 확인
sudo ufw status
sudo ufw status verbose   # 상세 정보

# 방화벽 활성화 / 비활성화
sudo ufw enable
sudo ufw disable

# 방화벽 초기화
sudo ufw reset
```

---

---

# ② 포트 허용 / 차단 ⭐️

```bash
# 포트 번호로 허용
sudo ufw allow 8000       # 8000번 포트 허용
sudo ufw allow 80         # HTTP
sudo ufw allow 443        # HTTPS
sudo ufw allow 22         # SSH (숫자)
sudo ufw allow ssh        # SSH (서비스명)

# 포트 번호로 차단
sudo ufw deny 8080

# 규칙 삭제
sudo ufw delete allow 8000
sudo ufw delete deny 8080

# 확인
sudo ufw status
```

---

---

# ③ ufw 설정 순서 ⚠️ 중요

```
⚠️ 가장 많이 하는 실수:
  방화벽 enable 전에 SSH 포트를 안 열면
  → enable 하는 순간 SSH 차단
  → 서버 원격 접속 불가 (대참사)

올바른 순서:
  1. SSH 허용 먼저
  2. 필요한 포트 허용
  3. 그 다음 enable
```

```bash
# ✅ 올바른 순서
sudo ufw allow ssh        # 1. SSH 먼저 허용
sudo ufw allow 8000       # 2. 앱 포트 허용
sudo ufw enable           # 3. 활성화
sudo ufw status           # 4. 확인
```

## ufw status 출력 해석

```
Status: active

To          Action    From
--          ------    ----
22/tcp      ALLOW     Anywhere
8000        ALLOW     Anywhere
22/tcp (v6) ALLOW     Anywhere (v6)
8000 (v6)   ALLOW     Anywhere (v6)
```

---

---

# ④ 특정 IP / 서브넷 제한

```bash
# 특정 IP 에서만 허용
sudo ufw allow from 192.168.1.100
sudo ufw allow from 192.168.1.100 to any port 22

# 특정 서브넷 허용
sudo ufw allow from 192.168.1.0/24

# 특정 IP 차단
sudo ufw deny from 203.0.113.100
```

---

---

# ⑤ 실전 패턴 — 서버 오픈 체크리스트

```bash
# 웹 서비스 오픈 전 방화벽 설정

# 1. 현재 상태 확인
sudo ufw status

# 2. SSH 허용 (반드시 먼저!)
sudo ufw allow ssh

# 3. 서비스 포트 허용
sudo ufw allow 80    # HTTP
sudo ufw allow 443   # HTTPS
sudo ufw allow 8000  # 앱 포트

# 4. 방화벽 활성화
sudo ufw enable

# 5. 최종 확인
sudo ufw status verbose
```

## Docker 사용 시 주의

```
Docker 는 ufw 를 우회해서 포트를 열 수 있음
docker run -p 8080:8080 이미지
→ ufw 에서 8080 차단해도 외부 접근 가능한 경우 있음

→ Docker 방화벽은 iptables 로 직접 제어하거나
  /etc/docker/daemon.json 에서 iptables: false 설정
```

---

---

# ⑥ 트러블슈팅 패턴

```bash
# 포트 열었는데 접속 안 될 때 확인 순서
# 1. 포트가 실제로 열려있는지
ss -tlnp | grep :8000

# 2. 방화벽 상태
sudo ufw status

# 3. 방화벽 비활성화 후 테스트
sudo ufw disable
curl http://서버IP:8000   # 접속 되면 방화벽 문제
sudo ufw enable

# 4. 특정 포트 허용 추가
sudo ufw allow 8000
```

---

---

# 명령어 한눈에

|명령어|역할|
|---|---|
|`sudo ufw status`|방화벽 상태 확인|
|`sudo ufw enable`|방화벽 활성화|
|`sudo ufw disable`|방화벽 비활성화|
|`sudo ufw allow ssh`|SSH 허용|
|`sudo ufw allow 포트`|포트 허용|
|`sudo ufw deny 포트`|포트 차단|
|`sudo ufw delete allow 포트`|규칙 삭제|
|`sudo ufw reset`|방화벽 초기화|

---

---

# 자주 하는 실수

|실수|원인|해결|
|---|---|---|
|SSH 끊겨서 서버 접근 불가|enable 전에 SSH 허용 안 함|콘솔(VNC) 접속 후 `ufw allow ssh`|
|포트 열었는데 접속 안 됨|Docker 가 ufw 우회|iptables 직접 확인|
|ufw 비활성화인데 차단됨|클라우드 보안 그룹|AWS/GCP 콘솔에서 인바운드 규칙 확인|