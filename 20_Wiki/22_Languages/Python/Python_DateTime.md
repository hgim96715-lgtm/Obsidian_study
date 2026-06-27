---
aliases:
  - Python 날짜
  - 시간
  - time 모듈
  - datetime
  - sleep
  - timedelta
  - isoformat
  - date
  - today
  - isocalendar
  - weekday
tags:
  - Python
related:
  - "[[CS_Blocking_vs_NonBlocking]]"
  - "[[Airflow_Queues_Scaling]]"
  - "[[Python_Modules_Imports]]"
  - "[[Airflow_Sensors]]"
  - "[[Python_Mock_Data_Generator]]"
  - "[[Python_Calendar_Module]]"
  - "[[00_Python_HomePage]]"
  - "[[SQL_Date_Functions]]"
---
# Python_DateTime — 날짜와 시간

## 한 줄 요약

```
시점(언제?)    datetime
기간(얼마나?)  timedelta
대기(잠깐!)    time.sleep
측정(얼마 걸렸나?) time.time
```

---

---

# 왜 문자열 말고 datetime 을 쓰나

```
"2024-01-01" 은 파이썬 입장에서 그냥 글자(String)
"2024-01-01" + 1  → TypeError (글자에 숫자 더하기 불가)

datetime 객체로 만들어야
날짜 더하기/빼기 / 요일 계산 / 차이 계산 가능
```

---

---

# 핵심 개념

|개념|비유|역할|
|---|---|---|
|`datetime`|시계 + 달력|특정 시점 (년월일 시분초)|
|`date`|달력만|날짜만 (시간 제외)|
|`timedelta`|스톱워치|시간 간격 (3일 / 10분 차이)|
|`time.sleep`|일시정지|프로그램 대기|
|`time.time`|초시계|경과 시간 측정|

---

---

# ① datetime — 시계 + 달력

```python
from datetime import datetime, timezone

# 현재 날짜 + 시간
now = datetime.now()         # 로컬 시간
now_utc = datetime.now(timezone.utc)  # UTC 시간

# 특정 날짜 직접 만들기
d = datetime(2026, 12, 25, 9, 0, 0)  # 2026-12-25 09:00:00
```

```
datetime.now()   timezone 인자 가능 → 권장
datetime.today() timezone 인자 없음 (항상 로컬)
→ now() 쓰는 습관 들이기
```

---

---

# ② date — 날짜만

```python
from datetime import date

today = date.today()        # 2026-02-24
d_day = date(2026, 12, 25)

# 요일 (월=0 ~ 일=6)
today.weekday()   # 1 = 화요일

# 올해 몇 번째 주 (주간 통계에 필수)
year, week, weekday = today.isocalendar()
print(f"{year}년 {week}번째 주")
```

---

---

# ③ timedelta — 시간 간격

```
공식: 시점(datetime) + 기간(timedelta) = 새로운 시점
```

```python
from datetime import datetime, timedelta

now = datetime.now()

yesterday = now - timedelta(days=1)    # 어제 ← Airflow 필수
tomorrow  = now + timedelta(days=1)    # 내일
last_week = now - timedelta(weeks=1)   # 일주일 전
ago_30min = now - timedelta(minutes=30)
```

```python
# DE 실전 패턴 — 어제 날짜로 API 파라미터 만들기
yesterday_str = (datetime.now() - timedelta(days=1)).strftime("%Y%m%d")
# "20260317"
```

---

---

# ④ time.sleep — 프로그램 대기

```python
import time

time.sleep(3)     # 3초 대기
time.sleep(0.2)   # 0.2초 대기
```

```python
# API Rate Limit 대응 (1초에 5번 제한 → 0.2초 간격)
for i in range(10):
    call_api()
    time.sleep(0.2)

# 폴링 구조 — 파일이 올 때까지 대기
while True:
    if file_exists():
        process()
        break
    time.sleep(60)   # 60초 대기 후 재확인
```

```
⚠️ time.sleep() 은 CPU 를 완전히 멈추는 Blocking 함수
   비동기(async) 환경에서는 asyncio.sleep() 사용
```

>[[CS_Blocking_vs_NonBlocking]] 참고 

---

---

# ⑤ time.time — 경과 시간 측정

```python
import time

start = time.time()        # 시작 시각 (Unix timestamp, 초 단위 float)

# 오래 걸리는 작업
for i in range(1000000):
    pass

end = time.time()
elapsed = end - start

print(f"소요 시간: {elapsed:.3f}초")   # 소요 시간: 0.042초
```

```python
# 실전: API 수집 시간 측정
start = time.time()
data = fetch_all_hospitals()
elapsed = time.time() - start
print(f"[수집 완료] {len(data)}건 / {elapsed:.1f}초")
```

```
time.time() 반환값:
  Unix timestamp (1970-01-01 00:00:00 UTC 기준 경과 초)
  float 타입 → 소수점 이하 = 밀리초

time.time() vs datetime.now():
  time.time()     숫자(float) → 경과 시간 계산용
  datetime.now()  객체 → 날짜/시간 표현용
```

---

---

# ⑥ 날짜 → 문자열 (DB / API 저장용)

## isoformat() — 국제 표준

```python
now.isoformat()
# "2026-02-24T21:06:32.123456"
#             ↑ T 로 날짜와 시간 구분

now.isoformat(sep=" ")
# "2026-02-24 21:06:32.123456"  (T 대신 공백)
```

```
언제: Kafka 타임스탬프 / 빅데이터 저장 / 시간순 정렬 보장
```

## strftime() — 원하는 포맷으로

```python
now.strftime("%Y-%m-%d")          # "2026-02-24"   DB 저장용
now.strftime("%Y%m%d")            # "20260224"     API 파라미터 run_ymd
now.strftime("%Y-%m-%d %H:%M:%S") # "2026-02-24 21:06:32"
```

```
결과는 항상 str → 계산 불가 (계산하려면 다시 strptime 으로 변환)
```

## f-string — 로그 출력용

```python
print(f"현재: {now:%Y-%m-%d %H:%M}")
# 현재: 2026-02-24 21:06
```

## 비교표

|방법|결과 예시|용도|
|---|---|---|
|`isoformat()`|`2026-02-24T21:06:32.123456`|Kafka / 빅데이터|
|`strftime("%Y-%m-%d")`|`2026-02-24`|DB 저장|
|`strftime("%Y%m%d")`|`20260224`|API 파라미터|
|`f"{now:%H:%M}"`|`21:06`|로그 출력|

---

---

# ⑦ 문자열 → 날짜 (계산용으로 변환)

```
strptime = String Parse Time
글자를 분해해서 datetime 객체로 만들기
```

```python
from datetime import datetime

date_text = "2026/01/23"
date_obj  = datetime.strptime(date_text, "%Y/%m/%d")

print(date_obj)         # 2026-01-23 00:00:00
print(date_obj.date())  # 2026-01-23  (시간 제거)
```

```python
# 실전: API 응답 날짜 파싱
raw = "2026-04-09 05:13:00.0"
try:
    dt = datetime.strptime(raw, "%Y-%m-%d %H:%M:%S.%f")
    print(dt.strftime("%H:%M"))   # "05:13"
except ValueError:
    print(raw[11:16])             # 파싱 실패 시 슬라이싱 fallback
```

---

---

# 포맷 코드

|코드|의미|예시|
|---|---|---|
|`%Y`|4자리 연도|`2026`|
|`%m`|2자리 월|`01`|
|`%d`|2자리 일|`05`|
|`%H`|24시간|`14`|
|`%M`|분|`30`|
|`%S`|초|`00`|
|`%f`|마이크로초|`000000`|
|`%Y%m%d`|API 파라미터용|`20260105`|
|`%Y-%m-%d`|DB 저장용|`2026-01-05`|

---

---

# 실전 패턴 모음

```python
from datetime import datetime, timedelta, timezone

# ① 어제 날짜 API 파라미터
yesterday = (datetime.now() - timedelta(days=1)).strftime("%Y%m%d")
# "20260317"

# ② Airflow 파티션 폴더명
partition = (datetime.now() - timedelta(days=1)).strftime("ds=%Y-%m-%d")
# "ds=2026-03-17"

# ③ UTC 저장 + KST 출력
now_utc = datetime.now(timezone.utc)
now_kst = now_utc + timedelta(hours=9)
print(f"저장: {now_utc.isoformat()}")        # UTC 로 DB 저장
print(f"표시: {now_kst:%Y-%m-%d %H:%M} KST") # 화면 출력

# ④ 수집 시간 측정
import time
start = time.time()
fetch_data()
print(f"수집 완료: {time.time() - start:.1f}초")
```

---

---

# 자주 하는 실수

```
① import 방식 차이
  import datetime           → datetime.datetime.now()  길게 써야 함
  from datetime import datetime → datetime.now()        바로 사용

② 문자열 vs 객체 구분
  "2024-01-01"           → String / 계산 불가
  datetime(2024, 1, 1)   → 객체  / 계산 가능

③ time.time() vs datetime.now()
  time.time()    숫자(float) → 경과 시간 계산용
  datetime.now() 객체       → 날짜/시간 표현용
```