---
aliases:
  - api_crawling
tags:
  - Python
related:
  - "[[00_Python_HomePage]]"
  - "[[Python_API_Rate_Limit_Retry]]"
  - "[[Python_Requests_Methods]]"
  - "[[Python_Classes_Objects]]"
  - "[[Python_JSON]]"
  - "[[Python_Database_Connect]]"
---
# Python_API_Crawling


---

## HTML 크롤링 vs API 크롤링

|항목|HTML 크롤링 (BS4 / Selenium)|API 크롤링|
|---|---|---|
|수집 방식|HTML 파싱|JSON 직접 수신|
|JS 렌더링|Selenium 필요|불필요|
|수집량|화면에 보이는 것만|pageSize × pages 자유 설정|
|속도|느림|빠름|
|깨지는 조건|HTML 구조 변경|API 스펙 변경|
|발굴 방법|—|DevTools → Network → Fetch/XHR|

> 대부분의 현대 웹사이트는 React/Vue 기반이라 내부 API가 반드시 존재한다. HTML을 파싱하기 전에 먼저 API가 있는지 DevTools로 확인할 것.

---

## Step 1 — DevTools로 API 발굴

```
1. 크롤링할 페이지를 Chrome에서 열기
2. F12 → Network 탭
3. 상단 필터: [Fetch/XHR] 선택  ← 핵심 (All 말고)
4. 페이지 새로고침 또는 "더보기 / 다음 페이지" 클릭
5. 목록에서 JSON 반환하는 요청 찾기
6. Headers 탭 → Request URL, Query String Parameters 확인
7. Preview 탭 → 응답 JSON 구조 확인
```

### API 요청인지 확인하는 기준

```
✅ API 요청 (찾는 것)
  Sec-Fetch-Dest: empty
  Content-Type: application/json
  :path: /api/... 또는 /v1/...

❌ HTML 페이지 요청 (API 아님)
  Sec-Fetch-Dest: document
  :path: /contents/main
```

### 찾은 API 빠르게 검증

```python
import requests, json

response = requests.get(
    "https://example.com/api/items",
    params={"page": 1, "size": 5},
    headers={"User-Agent": "Mozilla/5.0", "Referer": "https://example.com"},
    timeout=10
)
print(response.status_code)   # 200 이면 성공
print(json.dumps(response.json(), indent=2, ensure_ascii=False)[:500])
```

---

## Step 2 — Session 재사용

`requests.get()`은 매번 TCP 연결을 새로 맺는다. `Session`을 쓰면 연결을 유지(Keep-Alive)하고 헤더를 한 번만 설정한다.

```python
import requests

session = requests.Session()
session.headers.update({
    "User-Agent": "Mozilla/5.0 ...",
    "Accept": "application/json",
    "Referer": "https://example.com",
})

# 이후 session.get() 으로 호출 — 헤더 자동 포함
response = session.get(url, params=params, timeout=10)
```

### Session vs requests.get

|항목|`requests.get()`|`Session`|
|---|---|---|
|TCP 연결|매 요청마다 새로 맺음|재사용 (Keep-Alive)|
|헤더 설정|매번 넘겨야 함|한 번만 설정|
|쿠키 유지|❌|✅ 자동 유지|
|속도|느림|빠름|
|적합한 상황|단발성 1~2회|페이지네이션, 반복 호출|

---

## Step 3 — 페이지네이션 루프

```python
import time

all_items = []
page = 1

while page <= max_pages:
    params = {
        "page": page,
        "size": 50,
        # ... 사이트별 파라미터 추가
    }

    response = session.get(API_URL, params=params, timeout=10)
    response.raise_for_status()
    data = response.json()

    # 응답 구조가 list / dict 로 사이트마다 다름 → 방어 처리
    if isinstance(data, list):
        items = data
    elif isinstance(data, dict):
        items = (
            data.get("list")                          # {"list": [...]}
            or data.get("items")                      # {"items": [...]}
            or data.get("data", {}).get("list", [])   # {"data": {"list": [...]}}
            or []
        )
    else:
        items = []

    if not items:   # 빈 페이지 → 수집 종료
        break

    all_items.extend(items)
    print(f"페이지 {page}: {len(items)}개 / 누계 {len(all_items)}개")
    page += 1
    time.sleep(0.3)   # 서버 부하 방지 → [[Python_API_Rate_Limit_Retry]]
```

### 종료 조건 3가지

|조건|처리|
|---|---|
|빈 페이지 반환|`if not items: break`|
|최대 페이지 도달|`while page <= max_pages`|
|에러 발생|`except` 에서 `break` 또는 `continue`|

### 사이트마다 다른 응답 구조

```python
# 사이트 A: 바로 리스트
[{"id": 1}, {"id": 2}]

# 사이트 B: dict 안에 list
{"list": [...], "total": 100}

# 사이트 C: 중첩 dict
{"data": {"list": [...], "page": 1}}

# 사이트 D: 커서 기반 (오프셋 아닌 토큰)
{"items": [...], "nextCursor": "abc123"}
```

> 첫 페이지 응답을 `print(json.dumps(data, indent=2))` 로 구조 먼저 확인할 것

---

## Step 4 — 중복 제거

```python
# 방법 1 — seen set (순서 보존) ← 권장
seen = set()
unique = []
for item in all_items:
    key = item["id"]   # 사이트마다 PK 필드명 다름
    if key not in seen:
        seen.add(key)
        unique.append(item)

# 방법 2 — dict comprehension (간결, 순서 불보장)
unique = list({item["id"]: item for item in all_items}.values())
```

> 방법 1 추천: 수집 순서(랭킹 순서 등)를 보존해야 할 때

---

## Step 5 — dataclass 매핑

API 응답 dict를 바로 쓰면 오타·누락·타입 오류가 생기기 쉽다. `dataclass`로 구조를 명시해두면 안전하고 IDE 자동완성도 된다.

```python
from dataclasses import dataclass, asdict
from datetime import datetime

@dataclass
class Item:
    item_id:    str            # 기본값 없음 → 필수 인자
    name:       str            # 기본값 없음 → 필수 인자
    price:      int  = None    # 기본값 None → 선택 인자
    category:   str  = None    # 기본값 None → 선택 인자
    created_at: str  = None    # 기본값 None → 선택 인자

    def __post_init__(self):
        if self.created_at is None:
            self.created_at = datetime.now().isoformat()

    def to_dict(self) -> dict:
        return asdict(self)
```

---

### `= None` 이 의미하는 것

dataclass 필드에서 `{text}= None` 은 세 가지 의미를 동시에 가진다.

```python
@dataclass
class Item:
    item_id: str          # 기본값 없음 → 반드시 전달 (필수)
    name:    str          # 기본값 없음 → 반드시 전달 (필수)
    price:   int = None   # 기본값 None → 안 넘겨도 됨 (선택)
    url:     str = None   # 기본값 None → 안 넘겨도 됨 (선택)
```

**① Optional — 있을 수도 없을 수도 있는 필드**

```python
# API 응답에 없는 경우 → None 유지 → DB에 NULL로 저장
item = Item(item_id="001", name="상품A")
print(item.price)   # None (값이 없다는 의미)
print(item.url)     # None
```

**② 기본값 — 인스턴스 생성 시 생략 가능**

```python
# price, url 없이도 생성 가능
item = Item(item_id="001", name="상품A")           # ✅
item = Item(item_id="001", name="상품A", price=15000)  # ✅ 넘길 수도 있음
```

**③ 미확정 상태 — 나중에 다른 소스로 채울 예정인 필드**

```python
# 목록 API에서 기본값 채우고
item = Item(item_id="001", name="상품A")

# 상세 API / Selenium 에서 추가 필드 채움
item.price = get_price_from_detail(item.item_id)
item.url   = build_url(item.item_id)
```

**필드 선언 규칙 — 순서 주의**

```python
@dataclass
class Item:
    # ✅ 올바른 순서: 기본값 없는 필드 → 기본값 있는 필드
    item_id: str
    name:    str
    price:   int = None
    url:     str = None

    # ❌ 오류: 기본값 있는 필드 뒤에 기본값 없는 필드 불가
    # price:   int = None
    # name:    str          ← TypeError 발생
```

**None 체크 패턴**

```python
if item.price is None:
    print("가격 정보 없음")
else:
    print(f"가격: {item.price}원")

# DB 저장 시 None → NULL 자동 처리 (psycopg2, SQLAlchemy 모두)
```

---

## Step 6 — API 응답 → dataclass 변환 패턴

실전에서 자주 나오는 변환 처리:

```python
for data in items:
    # PK 없으면 skip
    pk = data.get("id") or data.get("goodsCode") or data.get("itemNo")
    if not pk:
        continue

    # 날짜 포맷: YYYYMMDD → YYYY-MM-DD
    raw_date = data.get("startDate", "")
    start_date = (
        f"{raw_date[:4]}-{raw_date[4:6]}-{raw_date[6:]}"
        if raw_date and len(raw_date) == 8 else None
    )

    # URL 보정: // → https://
    image_url = data.get("imageUrl", "")
    if image_url.startswith("//"):
        image_url = "https:" + image_url

    # 문자열 숫자 → int ("15,000" → 15000)
    price_raw = data.get("price", "")
    price = int(price_raw.replace(",", "")) if price_raw else None

    # 빈 문자열도 None 처리 (DB에 빈 문자열 저장 방지)
    category = data.get("category") or None

    item = Item(
        item_id    = str(pk),
        name       = data.get("name", ""),
        price      = price,
        start_date = start_date,
        image_url  = image_url,
        category   = category,
    )
    results.append(item)
```

### dict vs dataclass

|항목|dict|dataclass|
|---|---|---|
|오타 감지|❌ 런타임에서야 오류|✅ IDE 자동완성|
|기본값|매번 `.get("key", default)`|선언부에 한 번만|
|직렬화|그대로 사용|`asdict(obj)`|
|`__post_init__`|❌|✅ 초기화 후처리 가능|
|None 필드 명시|❌ 암묵적|✅ 구조가 명확함|

---

## 전체 흐름 정리

```
DevTools → Fetch/XHR 에서 API URL + 파라미터 확인
       │
       ▼
Session 생성 — headers 한 번만 설정
       │
       ▼
페이지네이션 루프 ─────────────────────┐
  params page 증가                     │
  session.get() 호출                   │
  응답 구조 방어 파싱                  │
  items 없으면 break                   │
  time.sleep(0.3) ─────────────────────┘
       │
       ▼
중복 제거 (seen set, 순서 보존)
       │
       ▼
dataclass 매핑 (날짜 / URL / 타입 / None 처리)
       │
       ▼
DB 적재 / 파일 저장
```

---

## 자주 쓰는 파라미터 패턴

```python
# 오프셋 페이지네이션
params = {"page": 1, "size": 50}
params = {"page": 1, "pageSize": 50}
params = {"offset": 0, "limit": 50}

# 배열 파라미터 (requests가 ?a=1&a=2 로 자동 변환)
params = {"category": ["001", "002"]}

# 정렬
params = {"sort": "WEEKLY_RANKING"}
params = {"orderBy": "createdAt", "direction": "DESC"}

# 날짜 범위
params = {"startDate": "20260101", "endDate": "20261231"}
```

---

## 목록 API + 상세 API 분리 패턴

많은 사이트가 목록과 상세를 분리된 API로 제공한다. 목록 API에서 ID를 수집하고, 상세 API로 필드를 보완하는 전략이 일반적이다.

```python
LIST_API   = "https://example.com/api/items"          # 목록
DETAIL_API = "https://example.com/api/items/{id}"     # 단건 상세

# 전략
items = get_list(max_pages=10)          # ID + 기본 필드 수집
for item in items:
    detail = get_detail(item["id"])     # 주소, 시간, 이미지 등 보완
    price  = get_price_selenium(url)    # Selenium 필요한 것만 최소화
    time.sleep(0.3)
```

> 상세 API가 있으면 Selenium 사용을 최소화할 수 있다. DevTools에서 상세 페이지를 열었을 때 호출되는 API를 추가로 찾아볼 것.

---

## 프로젝트 적용 사례

|프로젝트|목록 API|상세 API|Selenium 용도|
|---|---|---|---|
|인터파크 전시|`/contents/api/goods/genre`|`/v1/goods/{id}/summary`|가격 팝업만|
|(다음 프로젝트)|—|—|—|

> 인터파크 전시 상세 → [[01_docker_init.sql]]

---
