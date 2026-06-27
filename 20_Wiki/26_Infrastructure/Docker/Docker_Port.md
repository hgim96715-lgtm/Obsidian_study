---
aliases:
  - 포트
  - 포트바인딩
  - port binding
tags:
  - Docker
related:
  - "[[00_Docker_HomePage]]"
  - "[[Docker_Container]]"
  - "[[Docker_Dockerfile]]"
  - "[[Docker_Network]]"
---


# 한 줄 요약

```
EXPOSE = "이 포트 씁니다" 선언 (문서화)
-p     = 실제로 호스트 포트에 연결 (바인딩)
둘은 다르다 ⭐️
```

---

---

# EXPOSE vs -p 차이 ⭐️

```
EXPOSE 80 (Dockerfile)
  컨테이너가 80번 포트를 사용한다는 선언
  실제로 외부에서 접근 가능하게 열어주지는 않음
  → 문서화 / 이미지 메타데이터 역할

-p 8080:80 (docker run)
  호스트 8080 ← 컨테이너 80 실제로 연결
  이게 있어야 외부에서 접근 가능
  → 실제 포트 바인딩

결론:
  EXPOSE 없이 -p 써도 동작함
  EXPOSE 만 쓰고 -p 없으면 외부 접근 안 됨
  실무에서는 둘 다 쓰는 게 관례 (가독성)
```

---

---

# -p 문법 ⭐

```
-p 구조:
  -p 컨테이너포트                       (1단 — 호스트 포트 랜덤)
  -p 호스트포트:컨테이너포트             (2단 — IP 생략)
  -p 호스트IP:호스트포트:컨테이너포트    (3단 — IP 명시)

docker run -p 8080 nginx
               ↑
           컨테이너 포트만 지정 → 호스트 포트는 Docker 가 랜덤 배정

docker run -p 0.0.0.0:8080:80 nginx
               ↑       ↑     ↑
           바인딩 IP  호스트  컨테이너
```

```
1단 — 컨테이너 포트만 지정:
  -p 8080
  → 컨테이너 8080 포트를 호스트의 임의 포트에 연결
  → 어떤 포트가 배정됐는지 docker port 로 확인

  docker run -d --name my-nginx3 -p 8080 nginx
  docker port my-nginx3
  # 8080/tcp -> 0.0.0.0:32768   ← 이런 식으로 랜덤 배정

  언제 쓰나:
    여러 컨테이너 띄울 때 포트 충돌 피하고 싶을 때
    호스트 포트 번호가 중요하지 않은 테스트 환경
    
3단 각 자리 의미:
  0.0.0.0  → 호스트의 모든 네트워크 인터페이스에서 수신
             = 외부에서도 접근 가능
  8080     → 호스트(내 PC / 서버)의 포트
  80       → 컨테이너 안의 포트
```

```bash
# 기본 (IP 생략 → 0.0.0.0 자동)
docker run -p 8080:80 nginx

# IP 명시 — 외부 전체 허용 (기본과 동일)
docker run -p 0.0.0.0:8080:80 nginx

# IP 명시 — 로컬호스트만 허용 (외부 차단)
docker run -p 127.0.0.1:8080:80 nginx

# 여러 포트
docker run -p 8080:80 -p 443:443 nginx

# -P : EXPOSE 된 포트 전부 랜덤 호스트 포트에 자동 바인딩
docker run -P nginx
```

```
0.0.0.0 vs 127.0.0.1:
  -p 8080:80           → 0.0.0.0:8080 (모든 인터페이스, 외부 접근 가능)
  -p 127.0.0.1:8080:80 → 로컬호스트만 (외부 차단)

  서버 배포 시 보안상 127.0.0.1 바인딩 후
  리버스 프록시(Nginx) 앞에 두는 패턴 多
```

---

---

# docker inspect — ExposedPorts vs Ports ⭐️

```json
"Config": {
  "ExposedPorts": {
    "80/tcp": {}
  }
},
"NetworkSettings": {
  "Ports": {
    "80/tcp": [
      {
        "HostIp": "0.0.0.0",
        "HostPort": "8080"
      }
    ]
  }
}
```

```
ExposedPorts (Config 안):
  Dockerfile EXPOSE 또는 --expose 로 선언한 포트
  "이 포트 사용 예정" 메타데이터
  실제 바인딩 여부와 무관

Ports (NetworkSettings 안):
  실제로 호스트에 바인딩된 포트
  -p 로 연결한 것만 여기 나타남
  null 이면 바인딩 안 된 것

한 줄 정리:
  ExposedPorts = 선언 / Ports = 실제 연결
```

---

---

# docker port 명령어

```bash
# 컨테이너의 포트 바인딩 현황 한눈에 확인
docker port 컨테이너명

# 예시 출력
# 80/tcp -> 0.0.0.0:8080
```

```
docker inspect 보다 빠르게 포트 확인할 때 사용
```

---

---

# 실전 패턴

```bash
# NestJS 앱
docker run -d -p 3000:3000 my-nestjs-app

# PostgreSQL (외부 접근 허용)
docker run -d -p 5432:5432 postgres

# PostgreSQL (로컬만 허용, 보안 강화)
docker run -d -p 127.0.0.1:5432:5432 postgres

# Nginx 리버스 프록시 앞에 앱 두기
docker run -d -p 127.0.0.1:3000:3000 my-app   # 앱은 로컬만
docker run -d -p 80:80 -p 443:443 nginx        # Nginx 만 외부 노출
```

---

---

# 한눈에

|구분|위치|역할|
|---|---|---|
|`EXPOSE 80`|Dockerfile|포트 선언 (메타데이터)|
|`-p 8080:80`|docker run|실제 바인딩|
|`-P`|docker run|EXPOSE 포트 자동 랜덤 바인딩|
|`ExposedPorts`|docker inspect Config|선언된 포트 목록|
|`Ports`|docker inspect NetworkSettings|실제 바인딩 현황|