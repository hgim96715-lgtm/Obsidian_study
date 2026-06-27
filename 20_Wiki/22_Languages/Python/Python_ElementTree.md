---
aliases:
  - ElementTree
  - XML파싱
  - xml.etree
tags:
  - Python
related:
  - "[[00_Python_HomePage]]"
  - "[[Python_Requests_Methods]]"
  - "[[Python_Requests_Response]]"
  - "[[Python_JSON]]"
  - "[[Serialization_JSON_XML]]"
---

# Python_ElementTree — XML 파싱

## 한 줄 요약

```
XML 데이터를 파이썬에서 읽고 탐색하는 내장 모듈
공공데이터 API 는 JSON 이 아닌 XML 응답이 많음
pip 설치 불필요 — 파이썬 내장
```

```python
import xml.etree.ElementTree as ET
```

---

---

# ① XML 구조 이해

```xml
<!-- XML 은 태그(tag) 로 데이터를 감싸는 형식 -->
<response>
    <header>
        <resultCode>00</resultCode>
        <resultMsg>NORMAL_SERVICE</resultMsg>
    </header>
    <body>
        <items>
            <item>
                <hpid>A1100001</hpid>
                <hpname>서울대학교병원</hpname>
                <hvec>5</hvec>
                <dutyAddr>서울특별시 종로구 대학로 101</dutyAddr>
            </item>
            <item>
                <hpid>A1100002</hpid>
                <hpname>세브란스병원</hpname>
                <hvec>3</hvec>
                <dutyAddr>서울특별시 서대문구 연세로 50-1</dutyAddr>
            </item>
        </items>
    </body>
</response>
```

```
용어 정리:
  태그(tag)     <hpid> 에서 hpid 부분
  텍스트(text)  태그 사이의 값  A1100001
  속성(attrib)  <item id="1"> 에서 id="1"
  루트(root)    최상위 태그 → <response>
  자식(child)   바로 아래 태그 → <header>, <body>
```

---

---

# ② 파싱 시작 — fromstring vs parse

```python
import xml.etree.ElementTree as ET

# API 응답 (bytes) → XML
root = ET.fromstring(response.content)   # ← API 응답에 사용

# 문자열 → XML
root = ET.fromstring("<response><item><id>1</id></item></response>")

# 파일 → XML
tree = ET.parse("data.xml")
root = tree.getroot()
```

```
response.content   bytes 타입 → fromstring 에 바로 사용
response.text      str 타입   → fromstring(response.text.encode()) 필요
→ 공공데이터 API 에는 response.content 사용 권장
```

---

---

# ③ 태그 탐색 — find / findall / findtext

```python
# find — 첫 번째 매칭 Element 반환 (없으면 None)
header = root.find("header")
result_code = root.find("header/resultCode")

# findall — 매칭되는 모든 Element 리스트 반환
items = root.findall("body/items/item")

# .// — 하위 전체 재귀 검색 (위치 몰라도 됨)
items = root.findall(".//item")      # 어디 있든 item 태그 전부
hpids = root.findall(".//hpid")     # 모든 hpid 태그

# findtext — 태그 텍스트 바로 반환 ← 가장 많이 씀
result_msg = root.findtext("header/resultMsg")   # "NORMAL_SERVICE"
hpid = item.findtext("hpid")                     # "A1100001"
```

## find vs findtext — 핵심 차이 ⭐️

```python
item = root.find(".//item")

# find → Element 객체 반환 → .text 로 접근 필요
hpid_elem = item.find("hpid")
hpid = hpid_elem.text   # "A1100001"

# 태그 없으면 None → .text 접근 시 AttributeError!
missing = item.find("없는태그")
missing.text   # AttributeError: 'NoneType' has no attribute 'text'

# findtext → 텍스트 바로 반환 + None-safe
hpid = item.findtext("hpid")        # "A1100001"
missing = item.findtext("없는태그")  # None (에러 안 남) ✅

# findtext 기본값 설정
missing = item.findtext("없는태그", "")   # None 대신 "" 반환
hvec = item.findtext("hvec", "0")         # 없으면 "0"
```

```
결론:
  find().text   → 태그 확실히 있을 때만
  findtext()    → 항상 안전 (권장) ← 실무에서 이걸 써야 함
```

---

---

# ④ 텍스트 추출 및 타입 변환

```python
for item in root.findall(".//item"):
    hpid   = item.findtext("hpid", "")
    hpname = item.findtext("hpname", "")

    # XML 은 모든 값이 문자열 → 숫자 변환 필요
    hvec_str = item.findtext("hvec", "0")
    hvec = int(hvec_str) if hvec_str else 0

    # 또는 한 줄로
    hvec = int(item.findtext("hvec") or 0)

    # 위도/경도 float 변환
    lat = float(item.findtext("wgs84Lat") or 0)
    lon = float(item.findtext("wgs84Lon") or 0)

    print(hpid, hpname, hvec, lat, lon)
```

---

---

# ⑤ 속성(attrib) 접근

```python
# <item id="001" type="emergency">
item = root.find(".//item")

item.attrib              # {'id': '001', 'type': 'emergency'}
item.attrib.get("id")    # "001"
item.get("id")           # "001" (단축)
item.get("없는속성", "")  # "" (기본값)
```

---

---

# ⑥ 실전 패턴 — 공공데이터 API 파싱

```python
import requests
import xml.etree.ElementTree as ET
import os

def fetch_er_data(api_key: str, stage: str = "서울") -> list:
    url = "http://apis.data.go.kr/B552657/ErmctInfoInqireService/getEmrrmRltmUsefulSckbdInfoInqire"
    params = {
        "serviceKey": api_key,
        "STAGE":      stage,
        "numOfRows":  "100",
        "pageNo":     "1"
    }

    r = requests.get(url, params=params, timeout=10)
    r.raise_for_status()

    root = ET.fromstring(r.content)

    # 결과 코드 확인 (공공데이터 API 특성: HTTP 200 이어도 에러일 수 있음)
    result_code = root.findtext("header/resultCode") or \
                  root.findtext(".//resultCode", "")
    if result_code != "00":
        msg = root.findtext(".//resultMsg", "알 수 없음")
        raise ValueError(f"API 에러: {result_code} - {msg}")

    result = []
    for item in root.findall(".//item"):
        result.append({
            "hpid":    item.findtext("hpid", ""),
            "hpname":  item.findtext("hpname", ""),
            "hvec":    int(item.findtext("hvec") or 0),
            "dutyAddr": item.findtext("dutyAddr", ""),
            "region":  item.findtext("dutyAddr", "")[:2],  # 앞 2자리 = 시도명
        })

    return result
```

---

---

# ⑦ JSON vs XML 비교

```
JSON                     XML
─────────────────────────────────────────
{ "hpid": "A1100001" }  <hpid>A1100001</hpid>
파이썬 dict 와 유사       태그로 감싸는 구조
r.json() 으로 바로 변환  ET.fromstring() 으로 파싱
대부분의 현대 API        공공데이터 API / 레거시 시스템
```

```python
# JSON 응답
data = r.json()
hpid = data["item"]["hpid"]

# XML 응답
root = ET.fromstring(r.content)
hpid = root.findtext(".//hpid")
```
---
---
# ⑧ XML 내부 확인 — 디버깅

```
print(root) → <Element 'response' at 0x106757f90>
→ Element 객체 주소만 나옴, 내용 안 보임
→ 아래 방법으로 내용 확인
```

```python
import xml.etree.ElementTree as ET

# 방법 1: XML 문자열로 변환 (가장 확실)
print(ET.tostring(root, encoding="unicode"))
# → <response><header>...</header><body>...</body></response>

# 방법 2: 예쁘게 들여쓰기 출력 (Python 3.9+)
ET.indent(root)
print(ET.tostring(root, encoding="unicode"))
# → 태그별로 들여쓰기 적용

# 방법 3: API 원본 응답 그대로 출력
print(r.content.decode("utf-8"))
# → API 가 실제로 뭘 보냈는지 그대로 확인

# 방법 4: 첫 번째 item 컬럼 전체 확인 (실용적)
item = root.find(".//item")
if item is not None:
    for child in item:
        print(f"{child.tag}: {child.text}")

# 방법 5: 구조 탐색 (어떤 태그가 있는지)
for child in root:
    print(child.tag)              # header / body
    for grandchild in child:
        print(f"  {grandchild.tag}")  # resultCode / items 등
```

```
상황별 추천:
  API 응답 구조 처음 확인할 때  → r.content.decode("utf-8")
  특정 item 컬럼 확인할 때      → item 루프로 tag: text 출력
  전체 XML 트리 구조 볼 때      → ET.indent + ET.tostring
```

---
---
# ⑨ 타입 힌트 — ET.Element | None

```
함수 반환값이 "ET.Element 또는 None" 임을 명시
API 호출 성공 → ET.Element 반환
API 호출 실패 → None 반환

Python 3.10+:  ET.Element | None
Python 3.9 이하: Optional[ET.Element]
```


```python
import xml.etree.ElementTree as ET
from typing import Optional

# Python 3.10+ 방식
def fetch_xml(endpoint: str, extra_params: dict = None) -> ET.Element | None:
    ...
    root = ET.fromstring(r.content)
    return root     # 성공 → ET.Element
    return None     # 실패 → None

# Python 3.9 이하 방식
def fetch_xml(endpoint: str, extra_params: dict = None) -> Optional[ET.Element]:
    ...
```


```python
# 호출하는 쪽에서 None 체크 필수
root = fetch_xml("getEmrrmRltmUsefulSckbdInfoInqire")

if root is None:          # API 실패
    return {}

items = root.findall(".//item")   # None 아닐 때만 탐색
```

```
ET.Element 가 뭔가:
  ET.fromstring() 이 반환하는 타입
  XML 트리의 태그 하나를 나타내는 객체
  find / findall / findtext 메서드를 가짐

  root = ET.fromstring(r.content)
  type(root)  → xml.etree.ElementTree.Element

타입 힌트를 쓰는 이유:
  IDE 자동완성 지원
  함수 반환값이 코드만 봐도 명확
  None 체크 안 하면 경고 표시
```

---
---

# 메서드 한눈에 정리

|메서드|반환|설명|
|---|---|---|
|`ET.fromstring(bytes)`|Element|bytes → XML 트리|
|`ET.parse("file")`|ElementTree|파일 → XML|
|`root.find("태그")`|Element or None|첫 번째 매칭|
|`root.findall(".//태그")`|list|모든 매칭 (재귀)|
|`root.findtext("태그")`|str or None|텍스트 바로 반환 ← 권장|
|`root.findtext("태그", "기본값")`|str|None 대신 기본값|
|`element.text`|str or None|태그 텍스트|
|`element.tag`|str|태그 이름|
|`element.attrib`|dict|태그 속성 전체|
|`element.get("속성")`|str or None|특정 속성 값|
