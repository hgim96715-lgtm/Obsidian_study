---
aliases:
  - openssl
  - openssl rand
  - openssl base64
  - openssl dgst
  - 시크릿 키 생성
tags:
  - DevOps
  - Security
related:
  - "[[00_DevOps_Ecosystem_HomePage]]"
  - "[[Auth_Concept]]"
  - "[[Web_XSS_CSRF]]"
---
# OpenSSL — 암호화 · 난수 · 인증서 CLI 도구

> [!info]
> OpenSSL = TLS/SSL 프로토콜과 범용 암호화 기능을 제공하는 오픈소스 라이브러리 + CLI 도구.
> 서버 시크릿 키 생성(`rand`), Base64 변환, 해시, SSL 인증서 생성 등에 사용.
> JWT_SECRET · NEXTAUTH_SECRET 생성 → `openssl rand`. SSL 인증서 → `openssl req/x509`.

---

# openssl rand — 난수 생성 ⭐️⭐️⭐️⭐️

```txt
암호학적으로 안전한 난수(random bytes)를 생성하는 명령어
  Math.random() 같은 일반 난수와 달리 → 예측 불가능 (CSPRNG 사용)
  JWT_SECRET, NEXTAUTH_SECRET, SESSION_SECRET 등 서버 시크릿 생성에 사용
```

```bash
# 32바이트 난수 → Base64 인코딩 출력
openssl rand -base64 32
# 출력 예: 4f3k+jP2mQr8sXvYnZwD1Lh9KpRtUeAcBgHiNqMoVwE=

# 32바이트 난수 → hex(16진수) 인코딩 출력
openssl rand -hex 32
# 출력 예: a3f7c2d1e4b8905f6a2c3d4e5f6a7b8c9d0e1f2a3b4c5d6e7f8a9b0c1d2e3f4
```

## -base64 vs -hex ⭐️⭐️⭐️⭐️

```txt
공통점:
  둘 다 32바이트(256비트)의 동일한 난수를 생성
  암호학적 강도는 완전히 동일 — 보안 차이 없음
  차이는 오직 "인코딩 방식(출력 형식)"만

-base64 32:
  출력 문자: A-Z · a-z · 0-9 · + · / · =
  출력 길이: 44자
  특수문자(+, /, =) 포함 → URL·일부 환경에서 이스케이프 필요할 수 있음
  관례적으로 더 많이 쓰임

-hex 32:
  출력 문자: 0-9 · a-f 만 사용
  출력 길이: 64자
  특수문자 없음 → 어디서나 안전하게 붙여넣기 가능
  URL·쿼리스트링·shell 환경에서도 이스케이프 불필요

어떤 걸 쓰나:
  JWT_SECRET, NEXTAUTH_SECRET → 둘 다 OK (서버 env에만 있으면 됨)
  URL·쿼리스트링에 포함되는 값 → -hex 권장
  길이 짧게 → -base64 (같은 강도, 더 짧음)
```

## 왜 32(256비트)인가

```txt
AES-256, HS256(JWT 기본) 등 현재 표준 알고리즘이 256비트 키를 사용
브루트포스로 뚫으려면 2^256가지를 시도 → 사실상 불가능
16바이트(128비트)도 안전하지만 32바이트(256비트)가 관례적 표준
```

```bash
# 실전 — .env에 복사해서 사용
JWT_SECRET=$(openssl rand -base64 32)
NEXTAUTH_SECRET=$(openssl rand -hex 32)

# 또는 출력값을 직접 복사
openssl rand -base64 32   # → 터미널에 출력 → 복사 → .env에 붙여넣기
```

---

# openssl base64 — Base64 인코딩/디코딩 ⭐️⭐️⭐️

```bash
# 문자열 → Base64 인코딩
echo -n "hello world" | openssl base64
# 출력: aGVsbG8gd29ybGQ=

# Base64 → 디코딩
echo "aGVsbG8gd29ybGQ=" | openssl base64 -d
# 출력: hello world

# 파일 → Base64
openssl base64 -in image.png -out image.b64

# Base64 파일 → 원본 복원
openssl base64 -d -in image.b64 -out image_restored.png
```

```txt
echo -n 의 -n:
  줄바꿈(\n) 없이 출력 — Base64 인코딩 시 줄바꿈이 포함되면 결과가 달라짐
  → 문자열 인코딩 시 항상 -n 붙이기
```

---

# openssl dgst — 해시 생성 ⭐️⭐️⭐️

```bash
# SHA-256 해시 (파일)
openssl dgst -sha256 file.txt
# 출력: SHA2-256(file.txt)= a3f7c2...

# SHA-256 해시 (문자열)
echo -n "hello" | openssl dgst -sha256
# 출력: SHA2-256(stdin)= 2cf24d...

# HMAC-SHA256 (비밀키로 서명)
echo -n "message" | openssl dgst -sha256 -hmac "secret_key"
# 출력: HMAC-SHA2-256(stdin)= ...

# hex 출력만 (앞의 파일명 제거)
echo -n "hello" | openssl dgst -sha256 | awk '{print $2}'
```

```txt
주요 해시 알고리즘:
  -sha256    → SHA-256 (현재 표준)
  -sha512    → SHA-512 (더 긴 해시)
  -sha1      → SHA-1 (보안 취약 — 사용 비권장)
  -md5       → MD5 (보안 취약 — 파일 무결성 간단 체크용만)
```

---

# openssl req/x509 — 자체 서명 인증서(개발용) ⭐️⭐️

```bash
# 개발용 자체 서명 인증서 생성 (HTTPS 로컬 테스트)
openssl req -x509 -newkey rsa:4096 -keyout key.pem -out cert.pem -days 365 -nodes

# 비대화형 (CN만 지정)
openssl req -x509 -newkey rsa:4096 \
  -keyout key.pem -out cert.pem \
  -days 365 -nodes \
  -subj "/CN=localhost"
```

```txt
출력 파일:
  key.pem  → 개인키 (비밀 — 절대 공개 금지)
  cert.pem → 인증서 (공개 — 서버에 설정)

옵션:
  -newkey rsa:4096  → 4096비트 RSA 키 새로 생성
  -days 365         → 인증서 유효기간 365일
  -nodes            → 개인키 암호화 안 함 (서버 자동 시작 시 필요)
  -subj "/CN=..."   → 비대화형으로 CN(Common Name) 지정

용도:
  로컬 HTTPS 개발 환경
  프로덕션은 Let's Encrypt 등 공인 CA 인증서 사용
```

---

# 자주 쓰는 한 줄 명령어 모음

```bash
# 시크릿 키 생성 (가장 자주 씀)
openssl rand -base64 32      # JWT_SECRET, NEXTAUTH_SECRET 등
openssl rand -hex 32         # hex 형식 시크릿

# 파일 SHA-256 체크섬 확인 (다운로드 파일 무결성)
openssl dgst -sha256 downloaded_file.tar.gz

# 인증서 내용 확인
openssl x509 -in cert.pem -text -noout

# 인증서 만료일 확인
openssl x509 -in cert.pem -noout -dates

# 원격 서버 인증서 확인
openssl s_client -connect example.com:443 -showcerts
```
