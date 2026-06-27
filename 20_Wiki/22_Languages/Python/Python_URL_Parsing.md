---
aliases:
  - urlib.parse
  - URL 파싱
  - urlencode
  - quote
  - urlparse
  - urljoin
  - URL 인코딩
  - 쿼리스트링
  - URL 조립
  - URL 분해
tags:
  - Python
related:
  - "[[00_Python_HomePage]]"
  - "[[HTTP_Status_Codes]]"
  - "[[Python_Requests_Methods]]"
  - "[[Python_Requests_Response]]"
  - "[[Encoding_Decoding_Concept]]"
---

# Python_URL_Parsing

## 개념 한 줄 요약

> **"URL 을 조립하고, 쪼개고, 인코딩하는 표준 라이브러리."**
>  `requests` 가 내부적으로도 사용하지만, 직접 URL 을 다룰 때 필수.

```python
from urllib.parse import urlencode, quote, urlparse, urljoin, parse_qs
```

---

---

# ① URL 구조 이해

```
https://apis.data.go.kr/1613000/TrainInfoService/getStrtpntAlocFndTrainInfo?serviceKey=ABC&run_ymd=20260308
```

## 컴퓨터가 인식하는 구조 (urlparse 기준)

```
scheme  : https
netloc : apis.data.go.kr (네트워크 위치, 통상적으로 host/도메인이라고 부름)
path    : /1613000/TrainInfoService/getStrtpntAlocFndTrainInfo
query   : serviceKey=ABC&run_ymd=20260308
```

## 개발자들이 실무에서 소통하는 API 구조

```text
Base URL : https://apis.data.go.kr/1613000/TrainInfoService 
		   (API를 제공하는 기관의 공통/기본 주소)  
Endpoint : /getStrtpntAlocFndTrainInfo 
		   (Base URL 뒤에 붙어 '특정 기능'을 수행하는 세부 목적지)
```

>파이썬의 `urlparse()`는 Base URL과 Endpoint를 구분하지 않고 도메인 뒤의 주소 덩어리를 통째로 `path` 하나로 반환한다. 
>하지만 개발 실무에서는 코드의 재사용성을 위해, 변하지 않는 `Base URL`은 상단에 빼두고, 요청할 때마다 `Endpoint`만 교체해가며 사용한다.

## ⚠️ 주의: 공공데이터포털 공식 문서와의 용어 차이

>공공데이터포털(데이터.go.kr)의 활용 가이드 문서는 일반적인 개발 실무와 **용어를 다르게** 사용하므로 코드를 짤 때 헷갈리지 않도록 주의해야 한다.

>공공데이터 가이드 문서에 `End Point`라고 적힌 주소를 보았다면, 파이썬 코드에서는 이를 `BASE_URL`이라는 변수에 담아서 공통 뼈대로 사용하는 것이 올바른 방법이다!

| **일반 개발자 실무 / 파이썬 코드**         | **공공데이터포털 가이드 문서**                               |
| ------------------------------ | ------------------------------------------------ |
| **Base URL** (공통/기본 주소)        | **End Point** (예: `.../run/v2`)                  |
| **Endpoint / Path** (세부 기능 경로) | **오퍼레이션(Operation) / 상세기능** (예: `/getTrainList`) |

---

---

# ② urlparse — URL 분해

> URL 을 구성요소별로 쪼개서 분석할 때 사용.

```python
from urllib.parse import urlparse

url = "https://apis.data.go.kr/1613000/TrainInfoService?serviceKey=ABC&run_ymd=20260308"

result = urlparse(url)

print(result.scheme)    # https
print(result.netloc)    # apis.data.go.kr
print(result.path)      # /1613000/TrainInfoService
print(result.query)     # serviceKey=ABC&run_ymd=20260308
```

---

---

# ③ urlencode — 파라미터 딕셔너리 → 쿼리스트링

> 딕셔너리를 URL 쿼리스트링 형태로 변환한다. 
> `requests.get(url, params=...)` 이 내부적으로 이것을 사용함.

```python
from urllib.parse import urlencode

params = {
    "serviceKey" : "ABC123",
    "pageNo"     : 1,
    "numOfRows"  : 100,
    "dataType"   : "JSON",
    "run_ymd"    : "20260308",
}

query_string = urlencode(params)
print(query_string)
# serviceKey=ABC123&pageNo=1&numOfRows=100&dataType=JSON&run_ymd=20260308

# URL 에 직접 붙이기
base_url = "https://apis.data.go.kr/1613000/TrainInfoService"
full_url = f"{base_url}?{query_string}"
print(full_url)
```

---
## urlencode 의 함정 — 파라미터 이름에 특수문자가 있을 때 ⭐️

>`urlencode` (그리고 `requests` 의 `params={}`) 는 **key 도 인코딩 대상**으로 처리한다.

```python
# 코레일 V2 API 는 이런 형태의 파라미터 이름을 사용
params = {"cond[run_ymd::EQ]": "20260308"}

query_string = urlencode(params)
print(query_string)
# cond%5Brun_ymd%3A%3AEQ%5D=20260308
#      ↑ [=%5B  ]=%5D  :=%3A 로 인코딩됨
# -> 서버가 알아보지 못하고 400 Bad Request 반환
```

```python
# 해결: f-string 으로 직접 조립 (인코딩 없이 그대로 전송)
query_str = f"pageNo=1&numOfRows=100&cond[run_ymd::EQ]=20260308"
url = f"{base_url}?serviceKey={api_key}&returnType=JSON&{query_str}"
# cond[run_ymd::EQ] 가 그대로 유지됨 -> 서버 정상 처리
```

```text
언제 urlencode/params={} 를 쓰고 언제 직접 조립하나?

파라미터 이름이 평범한 영문/숫자  -> urlencode / params={} 사용
파라미터 이름에 [] :: 등 특수문자 -> f-string 직접 조립
```

>[[Python_Requests_Methods#params 의 함정 — 특수문자 인코딩 ⭐|params 함정]]  / [[02_API_Producer]] 실제 사례 참고


---

---

# ④ quote / unquote — 한글 / 특수문자 URL 인코딩

> URL 에 한글이나 특수문자가 포함되면 깨지거나 에러가 발생한다. 
> `quote()` 로 퍼센트 인코딩(%) 처리를 해줘야 한다. 
> 인코딩/디코딩 개념 자체가 궁금하면 → [[Encoding_Decoding_Concept]] 참고

## quote — 인코딩

```python
from urllib.parse import quote, unquote

# 한글 인코딩
quote("서울역")              # %EC%84%9C%EC%9A%B8%EC%97%AD
quote("서울역", safe="")     # %EC%84%9C%EC%9A%B8%EC%97%AD (safe 문자 없음)
```

## safe 파라미터

>**"이 문자는 인코딩하지 말고 그대로 둬"** 라고 예외를 지정하는 파라미터. 
>`quote()` 는 기본적으로 특수문자를 전부 인코딩하지만, `safe` 에 지정한 문자는 건드리지 않는다.

```python
# safe: 인코딩하지 않을 문자 지정 (기본값: '/')
# 기본값 safe="/"  -> / 는 인코딩 안 함
quote("/path/to/resource")          # /path/to/resource       <- / 그대로
quote("/path/to/resource", safe="") # %2Fpath%2Fto%2Fresource <- / 도 인코딩
```

```text
왜 / 가 기본값으로 safe 에 들어있냐?

URL 에서 / 는 경로 구분자다.
https://example.com/api/v1/users
                   ↑    ↑  ↑
이 / 들을 인코딩해버리면 경로 자체가 망가진다.
그래서 quote() 는 / 를 기본으로 건드리지 않는다.
```

```text
언제 safe="" 를 쓰냐?

경로가 아니라 쿼리스트링 값 안에 / 가 포함됐을 때.
예: 날짜 "2026/03/09" 를 파라미터 값으로 보낼 때
    safe="" 로 / 까지 인코딩해야 서버가 정확히 받는다.
```

>실무에서는 `urlencode()` 나 `requests params=` 가 자동으로 처리해줘서 `safe` 를 직접 건드릴 일은 많지 않다.

## unquote — 디코딩⭐

>️인코딩된 문자열을 **원래의 특수문자 상태로 복원**하는 함수.

```python
# 디코딩 (원래대로 복원)
unquote("%EC%84%9C%EC%9A%B8%EC%97%AD")  # 서울역
unquote("abc%2Bdef%3D")                 # abc+def=

# 인코딩 기호(%) 가 없으면 아무것도 건드리지 않고 그대로 통과
unquote("hello")   # hello  <- 변화 없음
```

## 공공데이터 API 키 이중 인코딩 문제

>공공데이터 API 키에는 `+`, `{text}=` 같은 특수문자가 포함된다. 
>여기서 실수하면 **"SERVICE KEY IS NOT REGISTERED ERROR"** 가 발생한다.

```text
인코딩 키 (복사한 상태): abc%2Bdef%3D   <- % 포함, 이미 인코딩된 상태
디코딩 키 (원본 상태):   abc+def=       <- 순수 원본
```

### 문제 — 이중 인코딩 발생

```python
import requests

# 인코딩 키를 그대로 params 에 넣으면?
params = {"serviceKey": "abc%2Bdef%3D"}  # 이미 인코딩된 키

requests.get(url, params=params)
# requests 가 자동으로 한 번 더 인코딩
# % -> %25 로 변환
# 결과: serviceKey=abc%252Bdef%253D  <- 이중 인코딩 대참사!
# 서버는 전혀 다른 키를 받아서 에러 발생
```

### 해결 — unquote 로 먼저 풀어주기

```python
from urllib.parse import unquote
import requests
import os
from dotenv import load_dotenv

load_dotenv()

# .env 에서 읽어온 키를 한 번 풀어준 뒤 넘기기
api_key = unquote(os.getenv("TRAIN_API_KEY"))
# "abc%2Bdef%3D" -> "abc+def=" (원본으로 복원)
# 이미 디코딩된 키라면 그대로 통과 (안전장치 역할)

params = {"serviceKey": api_key}
requests.get(url, params=params)
# requests 가 딱 한 번만 예쁘게 인코딩해서 전송 ✅
```

```text
흐름 정리:
.env 의 키 (상태 불명)
    ↓ unquote()
원본 키 (디코딩 확정)     <- 인코딩 기호 없으면 그대로 통과
    ↓ requests (자동 인코딩)
서버로 정상 전송 ✅

unquote 없이:
.env 의 키 (이미 인코딩된 상태)
    ↓ requests (한 번 더 인코딩)
이중 인코딩 키 전송 ❌ -> SERVICE KEY IS NOT REGISTERED ERROR
```

## quote vs quote_plus

```python
from urllib.parse import quote_plus

# quote:       공백 -> %20
# quote_plus:  공백 -> +  (HTML form 전송 방식)

quote("a b+c")       # a%20b%2Bc
quote_plus("a b+c")  # a+b%2Bc
```

---

---

# ⑤ parse_qs — 쿼리스트링 → 딕셔너리

> 반대 방향. 쿼리스트링을 파싱해서 딕셔너리로 만든다.

```python
from urllib.parse import parse_qs

query = "serviceKey=ABC&run_ymd=20260308&pageNo=1"

result = parse_qs(query)
print(result)
# {'serviceKey': ['ABC'], 'run_ymd': ['20260308'], 'pageNo': ['1']}
# 값이 리스트로 감싸짐 (같은 키가 여러 번 올 수 있어서)

# 첫 번째 값만 꺼내기
print(result["run_ymd"][0])  # '20260308'
```

---

---

# ⑥ urljoin — Base URL + 상대경로 합치기

> base URL 에 경로를 안전하게 합칠 때 사용. 문자열 단순 합치기(f-string) 는 `/` 중복 등 버그가 생길 수 있다.

```python
from urllib.parse import urljoin

base = "https://apis.data.go.kr/1613000/TrainInfoService/"
endpoint = "getStrtpntAlocFndTrainInfo"

url = urljoin(base, endpoint)
print(url)
# https://apis.data.go.kr/1613000/TrainInfoService/getStrtpntAlocFndTrainInfo

# 주의: base 에 / 가 없으면 마지막 경로가 교체됨
base2 = "https://apis.data.go.kr/1613000/TrainInfoService"
print(urljoin(base2, endpoint))
# https://apis.data.go.kr/1613000/getStrtpntAlocFndTrainInfo  <- 틀림!
# base 끝에 / 를 반드시 붙여야 한다
```

---

---

# 실전 패턴 — 공공데이터 API URL 직접 조립

```python
from urllib.parse import urlencode, urljoin
import os
from dotenv import load_dotenv

load_dotenv()
API_KEY = os.getenv("TRAIN_API_KEY")

BASE_URL = "https://apis.data.go.kr/1613000/TrainInfoService/"

def build_url(endpoint: str, params: dict) -> str:
    url = urljoin(BASE_URL, endpoint)
    query = urlencode(params)
    return f"{url}?{query}"

params = {
    "serviceKey" : API_KEY,
    "pageNo"     : 1,
    "numOfRows"  : 100,
    "dataType"   : "JSON",
    "run_ymd"    : "20260308",
    "dptre_stn_cd": "0001",
}

full_url = build_url("getStrtpntAlocFndTrainInfo", params)
print(full_url)
```

> 보통 `requests.get(url, params=dict)` 쓰면 자동으로 처리해줘서 직접 조립할 일은 많지 않다. 
> 하지만 URL 을 로그로 남기거나 디버깅할 때 이 패턴이 유용하다.

---

---

# 전체 치트시트

|함수|방향|용도|
|---|---|---|
|`urlparse(url)`|URL → 구성요소|URL 분해 분석|
|`urlencode(dict)`|dict → 쿼리스트링|파라미터 조립|
|`parse_qs(query)`|쿼리스트링 → dict|쿼리스트링 파싱|
|`quote(str)`|문자열 → 인코딩|한글/특수문자 처리|
|`unquote(str)`|인코딩 → 문자열|디코딩|
|`quote_plus(str)`|문자열 → 인코딩|폼 데이터 방식 인코딩|
|`urljoin(base, url)`|base + 경로 합치기|안전한 URL 조립|