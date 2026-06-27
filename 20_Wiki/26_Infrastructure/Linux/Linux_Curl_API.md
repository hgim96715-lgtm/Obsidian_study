---
aliases:
  - curl
  - API 테스트
  - curl 명령어
tags:
  - Linux
related:
  - "[[00_Linux_HomePage]]"
  - "[[Linux_Network_Check]]"
  - "[[Linux_Environment_Variables]]"
---

# Linux_Curl — curl & API 테스트

```txt
curl = 터미널에서 URL 로 데이터를 주고받는 명령어
브라우저 주소창에 붙여 넣는 것과 같은 동작
→ XML / JSON 텍스트가 터미널에 출력됨
```

---

---

# 기본 옵션 ⭐️

```bash
curl https://example.com              # 기본 GET 요청
curl -s https://example.com           # 진행 표시 없이 (silent)
curl -S https://example.com           # 에러 나면 메시지 표시
curl -sS https://example.com          # 보통 -sS 같이 씀
```

```txt
-s  silent — 진행 표시 줄임 (스크립트 / 파이프에서 유용)
-S  에러 나면 메시지 표시 (-s 와 함께 사용)
-sS 조용히 실행하되 에러는 보여줌 → API 테스트 표준 패턴
```

---

---

# .env 변수를 터미널에서 쓰기 ⭐️

```bash
# .env 파일에 키가 있을 때
# API_KEY=abc123
# API_URL=https://apis.example.com

# 터미널에 export (한 번만 실행)
export $(grep -E '^API_' .env | xargs)

# 이후 변수 사용 가능
echo ${API_KEY}
curl -sS "${API_URL}/endpoint?serviceKey=${API_KEY}"
```

```txt
grep -E '^API_' .env:
  .env 에서 API_ 로 시작하는 줄만 추출

xargs:
  여러 줄을 한 줄 인자로 변환

export $(...):
  추출한 KEY=VALUE 쌍을 환경변수로 등록

왜 쓰나:
  curl 에 키를 직접 입력하면 터미널 히스토리에 남음
  → export 방식으로 변수에 넣어서 사용
```

---

---

# 공공 API 테스트 패턴 ⭐️

```bash
# 목록 조회
curl -sS "${API_BASE_URL}/period2?serviceKey=${API_KEY}&PageNo=1&numOfrows=10"

# 특정 조건 추가
curl -sS "${API_BASE_URL}/period2?serviceKey=${API_KEY}&PageNo=1&numOfrows=10&serviceTp=A"

# 상세 조회
curl -sS "${API_BASE_URL}/detail2?serviceKey=${API_KEY}&seq=315928"
```

```txt
성공 여부 확인:
  XML 안에 이게 있으면 OK
  <resultCode>00</resultCode>
  <resultMsg>정상입니다.</resultMsg>

실패 케이스:
  API not found          → URL 오타
  resultCode 가 00 아님  → 파라미터 / 키 확인
```

---

---

# POST / 헤더 추가

```bash
# POST 요청
curl -sS -X POST https://api.example.com/data \
  -H "Content-Type: application/json" \
  -d '{"title": "테스트", "year": 2026}'

# 인증 헤더 추가
curl -sS https://api.example.com/movies \
  -H "Authorization: Bearer eyJhb..."

# 응답 헤더까지 보기
curl -sS -i https://api.example.com/movies
```

```txt
-X POST     HTTP 메서드 지정
-H          헤더 추가 (여러 번 사용 가능)
-d          요청 body 데이터
-i          응답 헤더 포함 출력
```

---
---
# 쿠키 — 저장 & 전송 ⭐️

```txt
-c  쿠키 저장 (cookie write)   → 로그인 응답 쿠키를 파일에 저장
-b  쿠키 전송 (cookie bring)   → 저장된 쿠키 파일을 요청에 포함

-d 는 request body 데이터 — 쿠키와 다름 ⚠️
```

```bash
# 1. 로그인 → 쿠키 파일에 저장
curl -sS -X POST http://localhost:3000/auth/login \
  -H "Content-Type: application/json" \
  -d '{"email":"admin@example.com","password":"비밀번호"}' \
  -c cookies.txt
#   ↑ 응답의 Set-Cookie 헤더를 cookies.txt 에 저장

# 2. 저장된 쿠키로 인증 필요한 요청
curl -sS -X POST http://localhost:3000/collector/collect \
  -b cookies.txt
#   ↑ cookies.txt 의 쿠키를 요청 헤더에 자동으로 포함

# 쿠키 저장 + 전송 동시에 (세션 유지하며 여러 요청)
curl -sS http://localhost:3000/admin/me \
  -b cookies.txt \
  -c cookies.txt
```

```txt
-c cookies.txt  로그인 응답의 Set-Cookie 를 파일에 저장
-b cookies.txt  이후 요청마다 파일의 쿠키를 요청에 포함

쿠키 파일 확인:
  cat cookies.txt
  # Netscape 형식으로 저장됨
  # 도메인 / 경로 / 만료 / 이름 / 값 등

쿠키 직접 입력 (파일 없이):
  -b "session=abc123"
  -b "session=abc123; token=xyz"

JWT Bearer 토큰 방식 vs 쿠키 방식:
  Bearer 토큰  → -H "Authorization: Bearer eyJhb..."
  쿠키         → -c 로 저장 후 -b 로 전송

  httpOnly 쿠키 기반 인증은 쿠키 방식 사용
  .env 의 ADMIN_EMAIL / ADMIN_PASSWORD 로 로그인 후 사용
```

---

---

# XML / JSON 보기 좋게 출력

```bash
# XML — xmllint (macOS 기본 내장)
curl -sS "${API_URL}/detail2?serviceKey=${API_KEY}&seq=315928" \
  | xmllint --format -

# JSON — python
curl -sS https://api.example.com/movies \
  | python3 -m json.tool

# JSON — jq (설치 필요)
curl -sS https://api.example.com/movies | jq .
curl -sS https://api.example.com/movies | jq '.data[0].title'
```

```txt
xmllint 없으면:
  긴 한 줄 XML 로 나와도
  resultCode / 원하는 태그 grep 으로 찾으면 됨

  curl -sS "..." | grep "resultCode"
```

---

---

# grep 으로 특정 값만 추출

```bash
# 응답에서 특정 태그만 뽑기
curl -sS "${API_URL}/period2?serviceKey=${API_KEY}&PageNo=1&numOfrows=5" \
  | grep -o '<title>[^<]*</title>'

# resultCode 만 확인
curl -sS "..." | grep "resultCode"
```

---

---

# 실전 팁

```bash
# 키를 직접 넣으면 히스토리에 남음 → 비추천
curl -sS "https://apis.example.com?serviceKey=여기에_실제_키"

# .env export 방식 권장
export $(grep -E '^API_' .env | xargs)
curl -sS "${API_URL}?serviceKey=${API_KEY}"

# 응답을 파일로 저장
curl -sS "${API_URL}/detail2?serviceKey=${API_KEY}&seq=315928" > response.xml

# 여러 seq 테스트 (for 루프)
for seq in 319005 249345 315928; do
  echo "=== seq: $seq ==="
  curl -sS "${API_URL}/detail2?serviceKey=${API_KEY}&seq=$seq" | grep "resultCode"
done
```

---

---

# 한눈에

| 옵션                                                       | 역할                     |
| -------------------------------------------------------- | ---------------------- |
| `-sS`                                                    | 조용히 실행 + 에러 표시 (표준 패턴) |
| `-X POST`                                                | HTTP 메서드 지정            |
| `-H "헤더"`                                                | 헤더 추가                  |
| `-d '데이터'`                                               | 요청 body                |
| `-i`                                                     | 응답 헤더 포함 출력            |
| <code>\|</code>`xmllint --format -`                      | XML 보기 좋게              |
| <code>\|</code>`python3 -m json.tool`                    | JSON 보기 좋게             |
| <code>\|</code> grep "키워드"`                              | 특정 값만 추출               |
| `export $(grep -E '^API_' .env` <code>\|</code> `xargs`) | .env 변수 터미널에 등록        |
