---
aliases:
  - requests
  - GET POST
  - requests.get
  - requests.post
  - requests.Session
  - Session
  - params
  - headers
  - timeout
  - Authorization
  - RequestException
  - JSONDecodeError
  - 멱등성
  - Idempotent
  - params 자동 변환
tags:
  - Python
related:
  - "[[Python_Requests_Response]]"
  - "[[Pandas_Json_Normalize]]"
  - "[[HTTP_Status_Codes]]"
  - "[[00_Python_HomePage]]"
  - "[[Python_Error_Handling]]"
  - "[[Python_URL_Parsing]]"
  - "[[Python_Selenium]]"
---


# Python_Requests_Methods — HTTP 요청

## 한 줄 요약

```
requests.get()   → 데이터 조회 (Read)
requests.post()  → 데이터 생성/전송 (Write)
둘 다 Response 객체 반환
```

---

---

# ① GET vs POST

```
GET   → "보여줘" — 데이터 조회
POST  → "처리해줘" — 데이터 생성 / 로그인 / 결제

GET:
  데이터가 URL 뒤에 붙음 (?key=value)
  눈에 보임 → 인증 토큰 절대 GET 으로 보내면 안 됨
  몇 번 호출해도 서버 변화 없음 (멱등성 O)

POST:
  데이터가 Body 안에 숨겨짐
  로그인 / 결제 / 파일 업로드 등
  호출할 때마다 서버 변화 가능 (멱등성 X)
```

---

---

# ② 기본 사용법

## GET — params

```python
import requests

r = requests.get(
    "https://api.example.com/trains",
    params={
        "serviceKey": "API_KEY",
        "run_ymd": "20260308",
        "pageNo": 1,          # int 그대로 → requests 가 자동으로 "1" 변환
    },
    timeout=10
)
# 실제 URL: ?serviceKey=API_KEY&run_ymd=20260308&pageNo=1
```

## POST — data / json

```python
# Form 데이터 (HTML form 처럼)
r = requests.post(
    "https://example.com/login",
    data={"username": "myid", "password": "1234"},
    timeout=10
)

# JSON 데이터 (API 에 데이터 보낼 때)
r = requests.post(
    "https://api.example.com/data",
    json={"name": "gong", "age": 28},  # Content-Type 자동으로 application/json
    timeout=10
)
```

## 파라미터 4가지 정리

|파라미터|용도|위치|
|---|---|---|
|`params`|GET 쿼리스트링|URL 뒤|
|`data`|POST Form 데이터|Body|
|`json`|POST JSON 데이터|Body (Content-Type 자동)|
|`headers`|인증 / 메타데이터|요청 헤더|
|`timeout`|최대 대기 시간(초)|—|

---

---

# ③ headers — 인증 & 메타데이터 ⭐️

```
headers = 요청에 붙이는 메타데이터
  누가 요청하는지 (인증)
  어떤 형식으로 보내는지 / 받고 싶은지
  서버가 요청자를 식별하는 정보
```

## 자주 쓰는 헤더

```python
headers = {
    # 인증 (가장 중요)
    "Authorization": f"Bearer {API_TOKEN}",

    # 내가 보내는 데이터 형식
    "Content-Type": "application/json",

    # 받고 싶은 데이터 형식
    "Accept": "application/json",

    # 브라우저인 척 (봇 차단 우회)
    "User-Agent": "Mozilla/5.0 (Windows NT 10.0; Win64; x64)",
}

r = requests.get(url, headers=headers, timeout=10)
```

## Authorization 형식

```python
# Bearer 토큰 — OAuth / JWT (가장 많이 씀)
"Authorization": f"Bearer {token}"

# API Key 방식 (서비스마다 다름)
"Authorization": f"token {github_token}"   # GitHub
"x-api-key": api_key                       # AWS API Gateway 등

# Basic 인증 — ID:PW 방식
import base64
credentials = base64.b64encode(b"id:password").decode()
"Authorization": f"Basic {credentials}"
```

```
⚠️ 인증 토큰은 반드시 headers 에 넣기
   params 에 넣으면 URL 에 그대로 노출됨
   서버 로그 / 브라우저 기록에 남아서 보안 위험
```

## 공공데이터 API 헤더 패턴

```python
# 공공데이터포털은 serviceKey 를 params 에 넣음 (서비스마다 다름)
params = {"serviceKey": API_KEY, "dataType": "JSON", ...}

# 일반 REST API 는 headers 에 넣음
headers = {"Authorization": f"Bearer {API_TOKEN}"}
```

---

---

# ④ timeout — 필수 안전장치 ⭐️

```
timeout 없으면 → 서버 응답 때까지 영원히 대기
→ 파이프라인 전체가 멈추는 대참사

항상 설정 필수
```

```python
# 10초 안에 응답 없으면 Timeout 예외
r = requests.get(url, timeout=10)

# 연결 대기 / 응답 대기 각각 설정
r = requests.get(url, timeout=(5, 30))
#                               ↑  ↑
#                             연결  응답
```

---

---

# ⑤ Session — 반복 호출 최적화 ⭐️

```
매번 requests.get() → 매번 새로운 TCP 연결 맺음 (느림)
Session 사용 → 연결 재사용 + 공통 설정 한 번만 (빠름)

Session 장점:
  TCP 연결 재사용 → 속도 향상
  headers / auth 한 번만 설정 → 반복 코드 제거
  쿠키 자동 유지 → 로그인 세션 유지
```

## 기본 사용

```python
import requests

# Session 없이 — headers 매번 반복
r1 = requests.get(url1, headers=headers, timeout=10)
r2 = requests.get(url2, headers=headers, timeout=10)   # 반복!
r3 = requests.get(url3, headers=headers, timeout=10)   # 반복!

# Session 사용 — 한 번만 설정
session = requests.Session()
session.headers.update({
    "Authorization": f"Bearer {API_TOKEN}",
    "Accept": "application/json",
})

r1 = session.get(url1, timeout=10)                     # 깔끔
r2 = session.get(url2, params={"page": 2}, timeout=10) # 파라미터만 추가
r3 = session.get(url3, timeout=5)                      # 이 요청만 timeout 다르게
```

## with 문 사용 (권장)

```python
# with 문 → 블록 끝나면 자동으로 연결 종료
with requests.Session() as session:
    session.headers.update({
        "Authorization": f"Bearer {API_TOKEN}",
        "Accept": "application/json",
    })

    for date in dates:
        r = session.get(url, params={"run_ymd": date}, timeout=10)
        r.raise_for_status()
        data = r.json()
```

## Session 헤더 + 개별 헤더 병합

```python
session = requests.Session()
session.headers.update({"Authorization": "Bearer TOKEN"})

# 개별 요청에 헤더 추가 → Session 헤더에 병합됨
r = session.get(url, headers={"X-Custom": "value"})
# 실제 전송 헤더: Authorization + X-Custom 둘 다
```

## 클래스에서 Session 쓰는 패턴

```python
class TrainAPI:
    def __init__(self):
        self.base_url = "https://api.example.com"
        self.session = requests.Session()
        self.session.headers.update({
            "Authorization": f"Bearer {os.getenv('API_TOKEN')}",
            "Accept": "application/json",
        })

    def get_schedule(self, date: str):
        r = self.session.get(
            f"{self.base_url}/schedule",
            params={"run_ymd": date},
            timeout=10
        )
        r.raise_for_status()
        return r.json()

    def get_realtime(self, date: str, train_no: str):
        r = self.session.get(
            f"{self.base_url}/realtime",
            params={"run_ymd": date, "trn_no": train_no},
            timeout=10
        )
        r.raise_for_status()
        return r.json()

# 사용
api = TrainAPI()
schedule = api.get_schedule("20260308")   # Session 재사용
realtime = api.get_realtime("20260308", "00051")   # 연결 그대로 사용
```

---

---

# ⑥ params 특수문자 함정 ⭐️

```
params={} 는 key 도 URL 인코딩함
[] :: 같은 특수문자가 key 에 있으면 서버가 못 알아봄
```

```python
# ❌ params 에 특수문자 포함된 key
params = {"cond[run_ymd::EQ]": "20260308"}
# 실제 전송: ?cond%5Brun_ymd%3A%3AEQ%5D=20260308  ← 서버가 모름

# ✅ URL 직접 조립
query = "cond[run_ymd::EQ]=20260308&pageNo=1"
url = f"{base_url}?serviceKey={api_key}&{query}"
# 실제 전송: ?serviceKey=...&cond[run_ymd::EQ]=20260308  ← 정상
```

```
언제 params={} vs 직접 조립:
  key 가 평범한 영문/숫자  → params={} 사용
  key 에 [] :: 특수문자    → f-string 직접 조립
```

---

---

# ⑦ 예외 처리 ⭐️

## raise_for_status()

```python
r = requests.get(url, timeout=10)

# 없으면 → 404 / 500 와도 모르고 지나침
# 있으면 → 4xx / 5xx 에서 HTTPError 발생
r.raise_for_status()

# 200~299  → 통과
# 400~499  → HTTPError (내 잘못)
# 500~599  → HTTPError (서버 잘못)
```

## 실전 예외 처리 패턴

```python
from requests.exceptions import RequestException, Timeout, JSONDecodeError

def fetch(url: str, params: dict) -> dict | None:
    try:
        r = requests.get(url, params=params, timeout=10)
        r.raise_for_status()
        return r.json()

    except Timeout:
        print("응답 시간 초과")
        return None

    except JSONDecodeError:
        # 공공데이터 API 에서 자주 발생
        # API 키 오류 → 200 OK 이지만 XML 반환
        print(f"JSON 파싱 실패: {r.text[:200]}")
        return None

    except RequestException as e:
        print(f"요청 실패: {type(e).__name__}: {e}")
        return None
```

## 예외 계층

```
RequestException       ← 이것 하나로 전부 잡을 수 있음
├── ConnectionError    ← 서버 연결 실패
├── Timeout            ← timeout 초과
├── HTTPError          ← raise_for_status() 에서 발생
└── JSONDecodeError    ← r.json() 실패
```

---

---

# ⑧ API 함수 파라미터 설계 ⭐️

```
함수 파라미터 = "호출할 때마다 바뀌는 값" 만
고정값은 함수 안 params 에 하드코딩
```

```python
# ❌ 고정값까지 파라미터로 받으면
def get_schedule(self, serviceKey, pageNo, dataType, run_ymd):
    ...
# 호출할 때마다 serviceKey, pageNo 등 매번 넘겨야 함

# ✅ 바뀌는 값만 파라미터로
def get_schedule(self, run_ymd: str, stn_cd: str = "0001") -> list:
    params = {
        "serviceKey":  self.api_key,   # 항상 고정 → 안에 하드코딩
        "pageNo":      1,              # 항상 고정
        "dataType":    "JSON",         # 항상 고정
        "run_ymd":     run_ymd,        # 호출마다 다름 → 파라미터
        "dptre_stn_cd": stn_cd,        # 가끔 바꿈 → 기본값 파라미터
    }
```

|분류|넣는 곳|예시|
|---|---|---|
|항상 고정|params 딕셔너리|`pageNo=1` / `dataType="JSON"`|
|호출마다 다름|함수 파라미터 (필수)|`run_ymd`|
|가끔 바꿈|함수 파라미터 (기본값)|`stn_cd="0001"`|
|인증 키|`self.api_key` / `.env`|`serviceKey`|

---

---

# 자주 하는 실수

|실수|원인|해결|
|---|---|---|
|토큰을 `params` 에 넣음|URL 에 노출됨|`headers["Authorization"]` 에 넣기|
|`timeout` 미설정|무한 대기|항상 `timeout=10` 설정|
|`raise_for_status()` 빠짐|에러 응답 모르고 지나침|항상 호출|
|`r.json()` 에러|응답이 XML/HTML|`JSONDecodeError` 처리 + `r.text` 확인|
|반복 호출에 `requests.get()` 매번|매번 새 연결|`Session` 사용|
|params key 에 특수문자|인코딩 됨|f-string 직접 조립|