---
aliases:
  - api
  - 서버 과부하 방지
tags:
  - Python
related:
  - "[[00_Python_HomePage]]"
  - "[[Python_Database_Connect]]"
---


> API를 반복 호출할 때 서버 과부하를 방지하고, 실패했을 때 안전하게 재시도하는 패턴 관련 노트 

---

## 왜 필요한가

API를 루프로 빠르게 반복 호출하면 두 가지 문제가 생긴다.

```
문제 1 — 서버 측 차단
  요청이 너무 빠르면 서버가 봇으로 인식
  → 429 Too Many Requests 반환
  → 심하면 IP 차단

문제 2 — 일시적 오류
  네트워크 불안정, 서버 순간 과부하
  → 500 / 503 / ConnectionError 발생
  → 루프 전체가 중단됨
```

해결책 → **적절한 딜레이** + **재시도(retry) 패턴**

---

## time.sleep — 기본 딜레이

```python
import time

page = 1
while page <= max_pages:
    response = session.get(url, params=params, timeout=10)
    # ... 데이터 처리 ...
    page += 1
    time.sleep(0.3)   # 다음 요청 전 0.3초 대기
```

### 딜레이 기준

|상황|권장 딜레이|이유|
|---|---|---|
|공개 API (공식 제공)|0.1 ~ 0.3초|Rate Limit 명시된 경우 문서 확인|
|내부 API (DevTools 발굴)|0.3 ~ 1.0초|서버 부하 불확실 → 보수적으로|
|Selenium 페이지 전환|1.0 ~ 2.0초|JS 렌더링 대기 포함|
|429 받은 직후|5.0초 이상|Retry-After 헤더 확인|

> 인터파크 프로젝트: API 목록 수집 `0.3초` / Selenium 상세 `2.0초`

---

## 429 Too Many Requests 처리

### 기본 처리

```python
response = session.get(url, params=params, timeout=10)

if response.status_code == 429:
    # Retry-After 헤더가 있으면 그 시간만큼 대기
    retry_after = int(response.headers.get("Retry-After", 5))
    print(f"429 수신 — {retry_after}초 후 재시도")
    time.sleep(retry_after)
    # 재시도
    response = session.get(url, params=params, timeout=10)
```

### HTTP 상태코드별 대응

|코드|의미|대응|
|---|---|---|
|`200`|정상|데이터 처리|
|`429`|요청 너무 많음|Retry-After 확인 후 대기 → 재시도|
|`500`|서버 내부 오류|잠시 후 재시도|
|`503`|서버 일시 불가|잠시 후 재시도|
|`403`|접근 거부|User-Agent / Referer 헤더 확인|
|`404`|없는 URL|재시도 의미 없음 → skip|

---

## 지수 백오프 (Exponential Backoff)

단순 고정 딜레이보다 정교한 재시도 전략. **실패할수록 대기 시간을 지수적으로 늘린다.**

```
1회 실패 → 2^1 = 2초 대기
2회 실패 → 2^2 = 4초 대기
3회 실패 → 2^3 = 8초 대기
4회 실패 → 포기
```

### 구현

```python
import time
import random

def fetch_with_backoff(session, url, params, max_retries=3):
    for attempt in range(max_retries):
        try:
            response = session.get(url, params=params, timeout=10)

            if response.status_code == 200:
                return response

            elif response.status_code == 429:
                retry_after = int(response.headers.get("Retry-After", 5))
                wait = max(retry_after, 2 ** attempt)
                print(f"429 수신 — {wait}초 대기 (시도 {attempt + 1}/{max_retries})")
                time.sleep(wait)

            elif response.status_code in (500, 503):
                wait = 2 ** attempt + random.uniform(0, 1)   # 지터 추가
                print(f"{response.status_code} — {wait:.1f}초 대기 (시도 {attempt + 1}/{max_retries})")
                time.sleep(wait)

            else:
                # 400, 404 등 재시도해도 의미 없는 오류
                print(f"재시도 불가 오류: {response.status_code}")
                return None

        except requests.exceptions.Timeout:
            wait = 2 ** attempt
            print(f"타임아웃 — {wait}초 대기 (시도 {attempt + 1}/{max_retries})")
            time.sleep(wait)

        except requests.exceptions.ConnectionError:
            wait = 2 ** attempt
            print(f"연결 오류 — {wait}초 대기 (시도 {attempt + 1}/{max_retries})")
            time.sleep(wait)

    print(f"최대 재시도 {max_retries}회 초과 — 포기")
    return None
```

### 지터(Jitter)란

```python
wait = 2 ** attempt + random.uniform(0, 1)
```

여러 클라이언트가 동시에 실패하면 동시에 재시도해서 서버가 또 과부하가 된다. `random.uniform(0, 1)` 를 더해서 재시도 타이밍을 분산시킨다.

---

## urllib3 Retry — requests 내장 재시도

직접 루프를 짜는 대신, `requests`의 `HTTPAdapter`에 Retry 전략을 붙이는 방법.

```python
import requests
from requests.adapters import HTTPAdapter
from urllib3.util.retry import Retry

def create_session_with_retry() -> requests.Session:
    session = requests.Session()

    retry_strategy = Retry(
        total=3,                              # 최대 3회 재시도
        backoff_factor=1,                     # 대기: 1, 2, 4초
        status_forcelist=[429, 500, 503],     # 재시도할 상태코드
        allowed_methods=["GET"],              # GET 요청만 재시도
        raise_on_status=False                 # 예외 대신 응답 반환
    )

    adapter = HTTPAdapter(max_retries=retry_strategy)
    session.mount("https://", adapter)
    session.mount("http://", adapter)

    session.headers.update({
        "User-Agent": "Mozilla/5.0 ...",
        "Referer": "https://tickets.interpark.com"
    })

    return session

# 사용
session = create_session_with_retry()
response = session.get(url, params=params, timeout=10)
```

### backoff_factor 계산

```
backoff_factor = 1 이면:
  1회 재시도 → 1 * (2^1 - 1) = 1초
  2회 재시도 → 1 * (2^2 - 1) = 3초
  3회 재시도 → 1 * (2^3 - 1) = 7초
```

---

## 페이지네이션에 적용한 전체 패턴

```python
import time
import requests
from requests.adapters import HTTPAdapter
from urllib3.util.retry import Retry

API_URL = "https://tickets.interpark.com/contents/api/goods/genre"

def create_session() -> requests.Session:
    session = requests.Session()
    retry = Retry(total=3, backoff_factor=1, status_forcelist=[429, 500, 503])
    session.mount("https://", HTTPAdapter(max_retries=retry))
    session.headers.update({
        "User-Agent": "Mozilla/5.0 (Macintosh; Intel Mac OS X 10_15_7) AppleWebKit/537.36",
        "Accept": "application/json",
        "Referer": "https://tickets.interpark.com/contents/genre/exhibition"
    })
    return session


def fetch_page(session, page: int) -> list:
    params = {
        "genre": "EXHIBIT",
        "page": page,
        "pageSize": 50,
        "sort": "DAILY_RANKING",
        "subCategory": ["08002", "08016"]
    }
    try:
        response = session.get(API_URL, params=params, timeout=10)
        response.raise_for_status()
        data = response.json()

        if isinstance(data, list):
            return data
        elif isinstance(data, dict):
            return data.get("list") or data.get("data", {}).get("list", [])
        return []

    except requests.RequestException as e:
        print(f"페이지 {page} 요청 실패: {e}")
        return []           # 실패해도 루프 계속


def crawl_all(max_pages: int = 10) -> list:
    session = create_session()
    all_items = []

    for page in range(1, max_pages + 1):
        print(f"[{page}/{max_pages}] 수집 중...")
        items = fetch_page(session, page)

        if not items:
            print("빈 페이지 — 종료")
            break

        all_items.extend(items)
        time.sleep(0.3)     # 서버 부하 방지

    # 중복 제거
    seen = set()
    unique = []
    for item in all_items:
        key = str(item.get("goodsCode"))
        if key not in seen:
            seen.add(key)
            unique.append(item)

    print(f"총 {len(unique)}개 수집 완료")
    return unique
```

---

## 직접 구현 vs urllib3 Retry 비교

|항목|직접 구현 (fetch_with_backoff)|urllib3 Retry|
|---|---|---|
|코드 양|많음|적음|
|상태코드별 세부 제어|✅ 자유롭게 가능|제한적|
|지터 추가|✅ 직접 추가|❌ 기본 없음|
|Retry-After 헤더 사용|✅ 직접 읽어야 함|✅ 자동 처리|
|학습 난이도|높음|낮음|
|추천 상황|복잡한 오류 처리 필요|빠르게 적용할 때|

---

## 요약

```
API 반복 호출 시 지켜야 할 3가지

1. time.sleep()으로 요청 간격 유지
   └─ 내부 API: 0.3~1.0초

2. 429 수신 시 즉시 재시도 금지
   └─ Retry-After 헤더 or 지수 백오프 적용

3. 실패해도 루프 전체가 죽지 않도록
   └─ try/except로 오류를 잡고 continue
   └─ 빈 결과만 반환하고 다음 페이지로
```

---