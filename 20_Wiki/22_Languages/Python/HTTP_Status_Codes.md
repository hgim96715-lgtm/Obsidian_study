---
aliases:
  - HTTP 상태 코드
  - Status Codes
  - 404 에러
  - 500 에러
  - 응답 코드
tags:
  - Python
related:
  - "[[Python_Requests_Response]]"
  - "[[00_Python_HomePage]]"
  - "[[Python_Requests_Methods]]"
---

# HTTP_Status_Codes

## 개념 한 줄 요약

> **서버가 내 요청을 받고 "됐어/안 됐어" 를 숫자로 알려주는 코드.**

---

---

# 코드 분류 — 첫 번째 숫자가 전부다

```
1xx  정보         (거의 안 봄)
2xx  성공 ✅      요청 처리됨
3xx  리다이렉트   다른 URL 로 이동
4xx  클라이언트 오류 ❌   내가 잘못 보냄
5xx  서버 오류 ❌         서버가 잘못함
```

---

---

# 2xx — 성공

|코드|이름|설명|
|---|---|---|
|**200**|OK|가장 흔한 성공. GET 요청 완료|
|**201**|Created|새로운 리소스 생성 완료 (POST 성공)|
|**204**|No Content|성공했지만 반환할 내용 없음 (DELETE 등)|

```python
res = requests.get(url)
print(res.status_code)  # 200

# 주의: 공공데이터 API 는 200 OK 로 에러를 반환하기도 함
# body 안의 resultCode 를 별도로 확인해야 함
# resultCode: "00" = 정상 / "22" = 미등록 키 / "30" = 한도 초과
```

---

---

# 4xx — 내가 잘못 보낸 것

|코드|이름|원인|해결|
|---|---|---|---|
|**400**|Bad Request|파라미터 오류, 인코딩 문제|`res.url` 로 요청 URL 확인|
|**401**|Unauthorized|인증 실패 (API 키 없음/틀림)|키 재확인, Authorization 헤더 확인|
|**403**|Forbidden|접근 권한 없음|사용 신청 여부 확인|
|**404**|Not Found|URL 이 없음|endpoint 오타 확인|
|**429**|Too Many Requests|호출 횟수 초과 (Rate Limit)|`time.sleep()` 추가, 내일 재시도|

```python
# 400 Bad Request 디버깅
res = requests.get(url, params=params)
print(res.url)   # 실제 요청 URL 확인
# cond[run_ymd::EQ] → cond%5Brun_ymd%3A%3AEQ%5D 로 인코딩됐는지 확인

# 429 Rate Limit 대응
import time
for i in range(100):
    res = requests.get(url)
    if res.status_code == 429:
        print("호출 한도 초과, 1초 대기")
        time.sleep(1)
        continue
    ...
```

---

---

# 5xx — 서버가 잘못한 것

|코드|이름|원인|해결|
|---|---|---|---|
|**500**|Internal Server Error|서버 내부 버그|잠시 후 재시도|
|**502**|Bad Gateway|프록시/게이트웨이 문제|잠시 후 재시도|
|**503**|Service Unavailable|서버 점검 중 / 과부하|공지 확인 후 재시도|

```
5xx 는 내 코드 문제가 아님
요청을 그대로 재시도하면 됨
파이프라인에서는 retry 로직이 있어야 함
```

---

---

# raise_for_status() 와의 관계

```python
res.raise_for_status()
# 2xx → 통과 (아무 일 없음)
# 4xx → requests.exceptions.HTTPError 발생
# 5xx → requests.exceptions.HTTPError 발생

# 예외 처리에서 status_code 꺼내기
try:
    res.raise_for_status()
except requests.exceptions.HTTPError as e:
    print(f"에러 코드: {res.status_code}")  # 404, 500 등
    print(f"에러 내용: {e}")
```

> 자세한 예외 처리 → [[Python_Requests_Response]]

---

---

# 공공데이터 API resultCode (별도 주의)

```
공공데이터 API 는 HTTP 상태코드가 200 OK 여도
body 안에 자체 에러 코드를 담아 보내는 경우가 있음

{
  "response": {
    "header": {
      "resultCode": "30",           <- 여기가 진짜 에러 코드
      "resultMsg": "SERVICE ERROR"
    }
  }
}

resultCode 종류:
  "00"  정상
  "01"  어플리케이션 에러
  "22"  API 키 미등록
  "30"  일일 호출 한도 초과
  "99"  기타 에러
```

```python
# resultCode 도 함께 확인하는 패턴
data = res.json()
result_code = data.get("response", {}).get("header", {}).get("resultCode", "??")
if result_code != "00":
    print(f"API 에러: {result_code}")
    return {}
```