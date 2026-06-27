---
aliases:
  - JSON 처리
  - JSON 모듈
  - 직렬화
  - 역직렬화
  - json.dumps
  - json.dump
  - json.loads
  - json.load
tags:
  - Python
  - JSON
related:
  - "[[Python_Dictionaries]]"
  - "[[Python_File_IO]]"
  - "[[Airflow_Hooks]]"
  - "[[Airflow_XComs]]"
  - "[[Serialization_JSON_XML]]"
  - "[[00_Python_HomePage]]"
  - "[[Kafka_Python_Serialization]]"
---
# Python_JSON

## 개념 한 줄 요약

> **JSON = 데이터를 교환할 때 쓰는 가장 표준적인 텍스트 포맷.** 파이썬 딕셔너리 `{}` 와 생김새는 거의 똑같지만 타입은 다르다. `json` 모듈로 둘 사이를 자유롭게 변환할 수 있다.

---

---

# 왜 필요한가

```
파이썬 딕셔너리 {"name": "서울역"} 은 메모리 안의 객체
파일 저장 / 네트워크 전송을 하려면 반드시 문자열(String) 로 바꿔야 함

반대로 API 응답이나 파일로 받은 데이터는 그냥 텍스트
텍스트 상태로는 data["name"] 처럼 꺼내 쓸 수 없음
→ 파이썬 딕셔너리로 변환해야 함

직렬화   (Serialization)   : 파이썬 객체 → JSON 문자열  (저장 / 전송용)
역직렬화 (Deserialization) : JSON 문자열 → 파이썬 객체  (코드 사용용)
```

---

---

# 핵심 공식 4가지 — s 유무가 전부다

```
s 가 붙으면 → String (문자열) 을 다룸  (메모리 안에서 변환)
s 가 없으면 → File (파일) 을 다룸      (하드디스크에 저장/읽기)
```

|방향|문자열 (String)|파일 (File)|
|---|---|---|
|파이썬 → JSON|`json.dumps(data)`|`json.dump(data, f)`|
|JSON → 파이썬|`json.loads(text)`|`json.load(f)`|

---

---

# import

```python
import json
```

---

---

# ① json.dumps — 파이썬 → JSON 문자열

```python
import json

data = {
    "trn_no"  : "00051",
    "stn_nm"  : "서울",
    "is_active": True,
    "delay"   : None,
    "stops"   : ["서울", "수원", "대전", "부산"],
}

# 기본 변환
result = json.dumps(data)
print(result)
# {"trn_no": "00051", "stn_nm": "\uc11c\uc6b8", ...}
#                               ↑ 한글이 유니코드로 깨짐

# ensure_ascii=False : 한글 그대로 유지 ⭐️
result = json.dumps(data, ensure_ascii=False)
print(result)
# {"trn_no": "00051", "stn_nm": "서울", "is_active": true, "delay": null, ...}

# indent : 들여쓰기 (로그 출력 / 파일 저장 시 가독성)
print(json.dumps(data, ensure_ascii=False, indent=2))
# {
#   "trn_no": "00051",
#   "stn_nm": "서울",
#   "is_active": true,
#   "delay": null,
#   "stops": [
#     "서울",
#     "수원",
#     ...
#   ]
# }
```

```
Python → JSON 타입 변환:
  dict        →  {}      (object)
  list        →  []      (array)
  str         →  ""      (string)
  int / float →  숫자
  True        →  true
  False       →  false
  None        →  null
```

---

---

# ② json.loads — JSON 문자열 → 파이썬

```python
import json

json_str = '{"trn_no": "00051", "stn_nm": "서울", "is_active": true, "delay": null}'

data = json.loads(json_str)

print(type(data))          # <class 'dict'>
print(data["trn_no"])      # 00051
print(data["is_active"])   # True   ← JSON true → Python True 자동 변환
print(data["delay"])       # None   ← JSON null → Python None 자동 변환
```

```
JSON → Python 타입 변환:
  {}          → dict
  []          → list
  ""          → str
  숫자         → int / float
  true        → True
  false       → False
  null        → None
```

---

---

# ③ json.dump — 파이썬 → 파일로 저장

```python
import json

data = {"trn_no": "00051", "stn_nm": "서울"}

# 파일에 JSON 형태로 저장
with open("train.json", "w", encoding="utf-8") as f:
    json.dump(data, f, ensure_ascii=False, indent=2)

# train.json 결과:
# {
#   "trn_no": "00051",
#   "stn_nm": "서울"
# }
```

```
주의: dumps 아님 → dump (s 없음)
      파일 열 때 encoding="utf-8" 필수 (한글 깨짐 방지)
```

---

---

# ④ json.load — 파일에서 읽기

```python
import json

# 파일에서 딕셔너리로 바로 읽기
with open("train.json", "r", encoding="utf-8") as f:
    data = json.load(f)

print(type(data))       # <class 'dict'>
print(data["stn_nm"])   # 서울
```

```
주의: loads 아님 → load (s 없음)
      파일 열 때 encoding="utf-8" 필수
```

---

---

# 언제 뭘 써야 하나

```
json.dumps  메모리 → 문자열
  API 응답 데이터 가공 후 Kafka send 직전
  로그 출력할 때 딕셔너리를 문자열로 변환
  requests.post(data=json.dumps(payload)) 처럼 POST body 만들 때

json.loads  문자열 → 메모리
  API 응답 .text 를 dict 로 변환할 때
  Kafka message.value 를 dict 로 복원할 때 (역직렬화)
  파일에서 읽은 텍스트를 파싱할 때

json.dump   메모리 → 파일
  설정 파일 저장 (DB 접속 정보, API 설정 등)
  수집한 데이터를 JSON 파일로 로컬 저장

json.load   파일 → 메모리
  저장해둔 설정 파일 읽기
  JSON 파일로 저장된 데이터 불러오기
```

---

---

# 실전 활용 패턴

## API 응답 파싱

```python
import requests, json

res = requests.get(url)

# 방법 1: requests 의 .json() 메서드 (내부적으로 json.loads 호출)
data = res.json()

# 방법 2: 수동 (응답이 JSON 인지 확신 못할 때)
data = json.loads(res.text)

print(data["response"]["body"]["totalCount"])
```

## Kafka 직렬화 / 역직렬화

```python
from kafka import KafkaProducer, KafkaConsumer
import json

# Producer: dict → bytes (json.dumps + encode)
producer = KafkaProducer(
    value_serializer=lambda v: json.dumps(v, ensure_ascii=False).encode("utf-8")
)
producer.send("train-realtime", value={"trn_no": "00051"})

# Consumer: bytes → dict (decode + json.loads)
consumer = KafkaConsumer(
    value_deserializer=lambda x: json.loads(x.decode("utf-8"))
)
```

> 자세한 내용 → [[Kafka_Python_Serialization]]

## 설정 파일 관리

```python
import json

# 설정 저장
config = {"db_host": "localhost", "db_port": 5432, "db_name": "train_db"}
with open("config.json", "w", encoding="utf-8") as f:
    json.dump(config, f, ensure_ascii=False, indent=2)

# 설정 읽기
with open("config.json", "r", encoding="utf-8") as f:
    config = json.load(f)

print(config["db_host"])  # localhost
```

---

---

# 초보자 착각 포인트

```
① JSON 이랑 딕셔너리는 같은 거 아닌가요?

  아님. 타입이 다름

  json_str = '{"name": "서울역"}'  ← str  (문자열)
  data     = {"name": "서울역"}    ← dict (딕셔너리)

  json_str["name"]  → TypeError  (문자열 인덱싱 에러)
  data["name"]      → "서울역"    ← 이렇게 써야 함

  json.loads() 를 거쳐야 딕셔너리로 쓸 수 있음

② 따옴표 문제

  Python dict: 'key' 작은따옴표 사용 가능
  JSON 표준  : "key" 큰따옴표만 허용

  손으로 JSON 파일 만들 때 작은따옴표 쓰면
  json.load() 에서 JSONDecodeError 발생

③ 한글 깨짐

  json.dumps(data)                     → \uc11c\uc6b8 (깨짐)
  json.dumps(data, ensure_ascii=False) → 서울 (정상) ← 이걸 써야 함

④ dump vs dumps / load vs loads 헷갈리기

  s 붙음  → String 대상   dumps / loads
  s 없음  → File 대상     dump  / load

⑤ 파일 저장/읽기 시 encoding 빠트리기

  open("file.json", "w")              → 한글 깨질 수 있음
  open("file.json", "w", encoding="utf-8") ← 이걸 써야 함
```

---

---

# 치트시트

|작업|코드|
|---|---|
|dict → JSON 문자열|`json.dumps(data, ensure_ascii=False)`|
|dict → JSON 문자열 (예쁘게)|`json.dumps(data, ensure_ascii=False, indent=2)`|
|JSON 문자열 → dict|`json.loads(json_str)`|
|dict → 파일 저장|`json.dump(data, f, ensure_ascii=False, indent=2)`|
|파일 → dict|`json.load(f)`|
|dict → bytes (Kafka용)|`json.dumps(data, ensure_ascii=False).encode("utf-8")`|
|bytes → dict (Kafka용)|`json.loads(b.decode("utf-8"))`|