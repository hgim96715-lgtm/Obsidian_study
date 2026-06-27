---
aliases:
  - 네트워크 확인
  - ip addr
  - ping
  - ss
  - ifconfig
  - lsof
tags:
  - Linux
related:
  - "[[00_Linux_HomePage]]"
  - "[[Linux_Firewall]]"
  - "[[Linux_SSH_Basics]]"
---
# Linux_Network_Check — 네트워크 진단

## 한 줄 요약

```txt
ip addr   = 내 IP / 인터페이스 상태 확인
ping      = 외부 통신 가능한지 테스트
ss -tlnp  = 어떤 포트가 열려있는지 확인
lsof      = 포트 & 프로세스 확인
```

---

---

# ① ip addr — 네트워크 인터페이스 상태 ⭐️

```bash
ip addr
ip a      # 축약형

# 특정 인터페이스만
ip addr show eth0
```

## 출력 해석

```txt
1: lo: <LOOPBACK,UP,LOWER_UP> mtu 65536
    inet 127.0.0.1/8              ← 루프백 (자기 자신)

2: eth0: <BROADCAST,MULTICAST,UP,LOWER_UP> mtu 1500
    link/ether 02:42:ac:10:32:03  ← MAC 주소
    inet 172.16.50.3/20           ← 내 IP 주소
         ↑ UP = 인터페이스 활성화
```

```txt
확인 포인트:
  <UP>     = 네트워크 카드 정상 작동
  <DOWN>   = 비활성화 → 드라이버/물리 연결 문제
  inet     = IPv4 주소
  inet6    = IPv6 주소
  lo       = 루프백 (127.0.0.1 / 자기 자신)
  eth0     = 메인 네트워크 카드 (외부 통신)
```

## IP만 깔끔하게 추출

```bash
hostname -I          # 모든 IP 한 줄로
# 172.16.50.3 172.17.0.1

ip addr | grep "inet " | awk '{print $2}'
# 127.0.0.1/8
# 172.16.50.3/20
```

---

---

# ② ifconfig — 구버전 네트워크 확인

```bash
# 설치 (없으면)
sudo apt-get install net-tools

ifconfig
ifconfig eth0   # 특정 인터페이스만
```

```txt
ip addr vs ifconfig:
  ip addr   현대 표준 (권장)
  ifconfig  레거시 환경에서 여전히 사용
  → 둘 다 알아두는 것이 안전

inet 필드 = IPv4 주소 (실제 통신에 쓰는 IP)
```

---

---

# ③ ping — 외부 통신 테스트 ⭐️

```bash
# 기본 (Linux = 무한 반복 → Ctrl+C 로 종료)
ping 8.8.8.8
ping google.com

# 횟수 제한 (-c 필수 ⭐️)
ping -c 3 8.8.8.8     # 3번만 보내고 종료
ping -c 4 google.com
```

```txt
-c 옵션이 필요한 이유:
  Linux ping 은 Ctrl+C 전까지 무한 반복
  스크립트/헬스체크에서 -c 없이 쓰면
  → 백그라운드에서 영원히 돌며 리소스 낭비

Health Check 패턴:
  ping -c 1 서버IP > /dev/null 2>&1
  if [ $? -eq 0 ]; then echo "살아있음"; fi
```

## ping 출력 해석

```txt
PING 8.8.8.8 (8.8.8.8) 56 bytes of data.
64 bytes from 8.8.8.8: icmp_seq=1 ttl=117 time=1.23 ms
                                              ↑ 응답 시간

--- 8.8.8.8 ping statistics ---
3 packets transmitted, 3 received, 0% packet loss
                                    ↑ 손실률 0% = 정상
rtt min/avg/max = 1.1/1.2/1.4 ms
```

```txt
실패 유형:
  Request timeout       → 응답 없음 (방화벽 차단 or 서버 다운)
  Destination unreachable → 경로 없음 (라우팅 문제)
  100% packet loss      → 완전 차단
```

---

---

# ④ ss — 열린 포트 확인 ⭐️

```bash
# 리스닝 포트 전체
ss -tlnp
# -t TCP / -l Listening / -n 숫자 / -p 프로세스명

# 특정 포트 확인
ss -tlnp | grep :8000
ss -tlnp | grep :22     # SSH
ss -tlnp | grep :5432   # PostgreSQL
ss -tlnp | grep :9092   # Kafka
```

## 출력 해석

```txt
State   Recv-Q  Send-Q  Local Address:Port   Peer Address:Port  Process
LISTEN  0       128     0.0.0.0:22           0.0.0.0:*          users:(("sshd",pid=1234))
LISTEN  0       5       127.0.0.1:5432       0.0.0.0:*          users:(("postgres",pid=5678))

0.0.0.0:22     = 모든 인터페이스의 22번 포트 (외부 접근 가능)
127.0.0.1:5432 = 루프백만 (로컬만 접근 가능)
```

## 트러블슈팅 흐름 ⭐️

```txt
앱 실행했는데 접속 안 될 때:

1. ss -tlnp | grep :8000
   → 포트가 안 보임 = 앱 자체가 안 켜진 것
   → 포트 보임 = 방화벽 문제

2. 포트 보이는데 외부 접속 안 됨
   → sudo ufw status 확인 → [[Linux_Firewall]]
```

---

---

# ⑤ lsof — 포트 & 프로세스 확인 ⭐️

```bash
# 특정 포트를 사용 중인 프로세스 확인
lsof -i :5555
lsof -i :8080
lsof -i :5432   # PostgreSQL

# 현재 LISTEN 중인 포트 전체 보기
lsof -nP -iTCP -sTCP:LISTEN

# sudo 필요할 때 (다른 유저 프로세스도 보려면)
sudo lsof -i :5555
sudo lsof -nP -iTCP -sTCP:LISTEN
```

## 옵션 설명

```txt
lsof -i :5555:
  -i = 네트워크 소켓 파일 보기
  :5555 = 5555 포트

lsof -nP -iTCP -sTCP:LISTEN:
  -n = IP를 호스트명으로 변환 안 함 (숫자로 표시, 빠름)
  -P = 포트 번호를 서비스명으로 변환 안 함 (5432 → postgres 아닌 숫자)
  -iTCP = TCP 소켓만
  -sTCP:LISTEN = LISTEN 상태인 것만 (현재 열려서 대기 중인 포트)
```

## 출력 해석

```bash
lsof -i :5555
# COMMAND   PID   USER   TYPE  DEVICE  SIZE  NODE  NAME
# node     1234  myuser  IPv6  12345   0t0   TCP   *:5555 (LISTEN)
# ↑ 명령어   ↑PID  ↑유저               ↑프로토콜  ↑포트   ↑상태

lsof -nP -iTCP -sTCP:LISTEN
# COMMAND     PID   USER    TYPE  NAME
# postgres    1234  postgres IPv4  *:5432 (LISTEN)
# node        5678  myuser   IPv6  *:3000 (LISTEN)
# docker-pr   9012  root     IPv4  *:8080 (LISTEN)
```

## ss vs lsof 차이

```txt
ss -tlnp:
  Linux 에서 표준
  빠름 / 간결한 출력
  포트 + 프로세스 이름

lsof -i :포트:
  Mac / Linux 모두 사용 가능
  특정 포트 하나 집중 확인 시 직관적
  PID / 유저 / 명령어 상세 정보

실무:
  특정 포트 누가 쓰고 있나  → lsof -i :포트번호
  전체 열린 포트 목록        → ss -tlnp 또는 lsof -nP -iTCP -sTCP:LISTEN
```

---

---

# ⑥ netstat — 구버전 포트 확인

```bash
sudo apt-get install net-tools

sudo netstat -tulnp
# ss -tlnp 와 동일한 역할 (구버전)
```

---

---

# ⑥ curl / wget — HTTP 통신 확인

```bash
# HTTP 응답 확인
curl http://localhost:8000
curl -I http://google.com    # 헤더만

# 파일 다운로드
wget https://example.com/file.tar.gz
curl -O https://example.com/file.tar.gz
```

---

---

# 네트워크 점검 흐름

```txt
접속 안 될 때 순서:

1. ip addr          내 IP / 인터페이스 UP 확인
2. ping -c 3 8.8.8.8   외부 통신 가능한지
3. ss -tlnp | grep :포트  포트 열려있는지
4. sudo ufw status      방화벽 차단 여부
```

---

---

# 명령어 한눈에

|명령어|역할|
|---|---|
|`ip addr`|인터페이스 / IP 확인|
|`hostname -I`|IP 주소만 출력|
|`ifconfig`|구버전 인터페이스 확인|
|`ping -c 3 대상`|외부 통신 테스트|
|`ss -tlnp`|리스닝 포트 전체|
|`ss -tlnp \| grep :포트`|특정 포트 확인|
|`curl http://주소`|HTTP 응답 확인|
|`netstat -tulnp`|구버전 포트 확인|