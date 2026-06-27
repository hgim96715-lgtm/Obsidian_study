---
aliases:
  - Requests Response 객체
  - Response Object
  - Python 응답 객체
tags:
  - Python
related:
  - "[[Python_JSON]]"
  - "[[Pandas_Json_Normalize]]"
  - "[[Airflow_Hooks]]"
  - "[[Python_Requests_Methods]]"
  - "[[00_Python_HomePage]]"
  - "[[Encoding_Decoding_Concept]]"
---
# Python_Requests_Response

## 개념 한 줄 요약

> **`requests.get()` 이 반환하는 `Response` 객체 = 서버가 편지 봉투에 담아 돌려준 답장 그 자체** 데이터만 들어있는 게 아니라 상태 코드, 헤더, 인코딩 정보까지 다 갖고 있다.

---

---

# Response 객체 — 봉투를 열기 전에

```python
res = requests.get(url, timeout=10)
# 여기서 res 는 Response 클래스의 인스턴스 (객체)
# print(res) 하면 <Response [200]> 만 나옴 <- 데이터 아님!
# 실제 내용은 .text / .json() / .content 안에 있음
```

## 꺼낼 수 있는 것들

|속성/메서드|반환 타입|설명|
|---|---|---|
|`.status_code`|`int`|응답 코드 (200, 404, 500 ...)|
|`.ok`|`bool`|200~299 이면 True|
|`.text`|`str`|응답 본문 (문자열)|
|`.json()`|`dict`/`list`|응답 본문 → 파이썬 객체 파싱|
|`.content`|`bytes`|응답 본문 (바이너리, 이미지/파일)|
|`.headers`|`dict`|응답 헤더|
|`.url`|`str`|실제 요청된 최종 URL|
|`.encoding`|`str`|인코딩 (utf-8, cp949 ...)|

---

---

# ① status_code / .ok — 봉투 상태 확인

```python
res = requests.get(url, timeout=10)

print(res.status_code)   # 200

# .ok: 200~299 이면 True, 나머지는 False
if res.ok:
    print("성공!")
else:
    print(f"실패: {res.status_code}")
    print(res.text)  # 에러 내용 확인
```

```
주의: 공공데이터 API 는 status_code 200 OK 로 에러를 보내기도 함
      body 안의 resultCode 를 별도로 확인해야 함

  resultCode "00"  → 정상
  resultCode "22"  → API 키 미등록
  resultCode "30"  → 일일 호출 한도 초과
```

> 전체 상태 코드 해석 → [[HTTP_Status_Codes]]

---

---

# ② raise_for_status() — 에러 자동 감지 ⭐️

```python
res = requests.get(url, timeout=10)
res.raise_for_status()   # 4xx, 5xx 이면 HTTPError 즉시 발생
data = res.json()        # 200 이면 여기까지 도달
```

```
호출 안 하면:
  서버가 404, 500 을 반환해도 파이썬은 모름
  res.json() 까지 그냥 진행하다가 엉뚱한 에러 발생
  "편지가 왔는데 내용물도 안 보고 봉투부터 찢는 격"

호출하면:
  2xx  → 통과 (아무 일 없음)
  4xx, 5xx → HTTPError 즉시 발생 → 어디서 에러났는지 바로 앎
```

```python
# 실전 패턴 (02_API_Producer 에서 사용)
try:
    res = self.session.get(url, timeout=10)
    res.raise_for_status()       # ← 여기서 4xx/5xx 잡힘

    return res.json()

except JSONDecodeError:
    # 200 OK 인데 XML/HTML 반환 (API 키 오류, 트래픽 초과)
    print(f"응답 내용: {res.text[:100]}")
    return {}

except RequestException as e:
    # ConnectionError, HTTPError, Timeout 등 전부 잡힘
    print(e)
    return {}
```

---

---

# ③ Content-Type — 데이터의 신분증

> 서버가 "내가 보낸 데이터는 이런 형식이니까 이렇게 해석해!" 라고 알려주는 것

```python
content_type = res.headers.get("Content-Type", "")
print(content_type)
# application/json; charset=utf-8
#                  ↑ 세미콜론 뒤에 인코딩 정보가 붙어서 오기도 함
```

```
주요 Content-Type 종류:
  application/json   → .json() 으로 파싱
  text/html          → .text 로 받아서 BeautifulSoup 등으로 파싱
  text/plain         → .text 로 받음
  image/jpeg, png    → .content 로 바이너리 저장
  application/octet-stream → 알 수 없는 이진 데이터
```

```python
# Content-Type 에 따른 처리 분기
if "application/json" in content_type:    # in 으로 확인 (세미콜론 뒤 문자열 방어)
    data = res.json()

elif "text/html" in content_type:
    html = res.text

elif "image" in content_type:
    with open("image.jpg", "wb") as f:
        f.write(res.content)
```

```
주의: 간혹 실제 JSON 데이터인데 text/plain 으로 보내는 서버도 있음
      Content-Type 이 틀려도 .json() 으로 파싱 시도해볼 수 있음
```

---

---

# ④ .text vs .json() vs .content

|속성|반환 타입|언제 씀|
|---|---|---|
|`.json()`|`dict`/`list`|JSON API 응답 파싱|
|`.text`|`str`|XML, HTML, 에러 내용 확인|
|`.content`|`bytes`|이미지, PDF 등 바이너리 파일|

```python
# 에러 응답 원본 확인
except JSONDecodeError:
    print(res.text[:200])       # XML / HTML 원본 확인

# 이미지 다운로드
img_res = requests.get(image_url)
with open("image.jpg", "wb") as f:
    f.write(img_res.content)
```

---

---

# ⑤ .url — 실제 요청된 URL 확인 (디버깅 필수)

```python
params = {"run_ymd": "20260309", "pageNo": 1}
res = requests.get(base_url, params=params)

print(res.url)
# https://apis.data.go.kr/...?run_ymd=20260309&pageNo=1
```

```
400 Bad Request 날 때 res.url 로 확인하면
파라미터가 어떻게 인코딩됐는지 바로 보임

cond[run_ymd::EQ] → cond%5Brun_ymd%3A%3AEQ%5D 로 깨진 것도 여기서 확인
[[Python_URL_Parsing]] params 함정 참고
```

---

---

# ⑥ 인코딩 문제 — 한글이 깨질 때

```python
# 한글이 깨져서 나올 때
res = requests.get(url)
print(res.text)  # 깩뛟쀍... 깨진 글자

# 해결: 인코딩 수동 지정 후 다시 읽기
res.encoding = "utf-8"     # 또는 "cp949"
print(res.text)            # 이제 정상 출력
```

```
requests 가 Content-Type 의 charset 을 못 읽으면 잘못된 인코딩으로 해석함
공공데이터 CSV 는 cp949 가 많음 → [[Encoding_Decoding_Concept]] 참고
```

---

---

# Content-Type vs Accept — 헷갈리지 말기

```
Accept        : 내가 서버한테 "나 JSON 으로 받고 싶어" 라고 요청 (Request Header)
Content-Type  : 서버가 나한테 "이건 JSON 데이터야" 라고 알려줌 (Response Header)
```

---

---

# 전체 흐름 — 실전 패턴

```python
import requests
from requests.exceptions import RequestException, JSONDecodeError

def fetch(url, params):
    try:
        # 1. 요청 보내기
        res = requests.get(url, params=params, timeout=10)

        # 2. 상태 코드 확인 (4xx, 5xx → HTTPError)
        res.raise_for_status()

        # 3. Content-Type 확인
        content_type = res.headers.get("Content-Type", "")

        # 4. 타입에 맞게 파싱
        if "application/json" in content_type:
            return res.json()
        else:
            return res.text

    except JSONDecodeError:
        # 200 OK 인데 JSON 이 아닌 경우 (XML, HTML)
        print(f"[JSONDecodeError] 응답 내용: {res.text[:100]}")
        return {}

    except RequestException as e:
        # 네트워크 오류, HTTPError, Timeout 등
        print(f"[RequestException] {res.status_code} - {e}")
        return {}
```

---

---

# 초보자 착각 포인트

```
① print(res) 하면 데이터가 나올 줄 알았는데 <Response [200]> 만 나옴
   → 실제 내용은 .text / .json() / .content 안에 있음

② .text 와 .json() 혼동
   → .text   : 속성 (괄호 없음) - 문자열 반환
   → .json() : 메서드 (괄호 필수) - 딕셔너리/리스트 반환
   → JSON 형식이 아닌데 .json() 쓰면 JSONDecodeError

③ Content-Type 비교 시 == 사용
   → "application/json; charset=utf-8" 처럼 세미콜론 뒤에 뭐가 붙음
   → == 으로 비교하면 False 나옴
   → 반드시 in 연산자로 확인
   → "application/json" in content_type  ← 이렇게

④ status_code 200 인데 에러?
   → 공공데이터 API 는 200 OK 로 에러를 반환하기도 함
   → body 안의 resultCode 를 별도로 확인해야 함
```

---

---

# 연습용 사이트

```
jsonplaceholder : https://jsonplaceholder.typicode.com/
  API 연습용 가장 유명한 사이트. 회원가입 불필요

httpbin : https://httpbin.org/
  HTTP 상태코드, 헤더, 인코딩 등 테스트 전용
  /status/404 → 404 반환
  /status/500 → 500 반환
  /get        → GET 요청 내용 그대로 돌려줌

example.com : https://example.com
  크롤링 연습용 표준 사이트
```