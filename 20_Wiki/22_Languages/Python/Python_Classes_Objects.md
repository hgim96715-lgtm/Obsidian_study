---
aliases:
  - Class
  - Object
  - Instance
  - 클래스
  - 객체
  - 인스턴스
  - 붕어빵
tags:
  - Python
related:
  - "[[Python_Dictionaries]]"
  - "[[Spark_Core_Broadcast]]"
  - "[[00_Python_HomePage]]"
  - "[[Python_Lists_Tuples]]"
  - "[[Python_DateTime]]"
  - "[[Python_Decorators]]"
  - "[[Python_Inheritance]]"
---
# Python_Classes_Objects

## 개념 한 줄 요약

> **"관련 있는 데이터(변수)와 행동(함수)을 한 박스에 묶어놓은 것."**

---

---

# 핵심 용어

|용어|비유|설명|
|---|---|---|
|**클래스 (Class)**|붕어빵 **틀**|설계도. 아직 실체 없음|
|**객체 / 인스턴스 (Object)**|틀로 찍어낸 **붕어빵**|실제로 만들어진 것|
|**속성 (Attribute)**|붕어빵 안에 들어있는 **팥**|객체 안의 데이터 (`.value` 의 정체)|
|**메서드 (Method)**|붕어빵이 할 수 있는 **행동**|객체 안의 함수|

---

---

# 클래스 기본 구조

```python
# 1. 설계도 (Class) 정의
class Bungeoppang:

    # ① __init__: 객체가 만들어질 때 자동으로 실행되는 초기 세팅
    def __init__(self, contents, price):
        self.contents = contents  # 속성 (데이터)
        self.price    = price     # 속성 (데이터)

    # ② 메서드: 이 객체가 할 수 있는 행동
    def eat(self):
        print(f"{self.contents} 맛 붕어빵을 먹어요! 냠냠.")


# 2. 객체 생성 (설계도로 실체를 만드는 것 = "인스턴스화")
fish1 = Bungeoppang("팥", 1000)      # fish1 탄생
fish2 = Bungeoppang("슈크림", 1500)  # fish2 탄생

# 3. 속성 꺼내기 (점 사용)
print(fish1.contents)  # 팥
print(fish2.price)     # 1500

# 4. 메서드 호출
fish1.eat()  # 팥 맛 붕어빵을 먹어요! 냠냠.
fish2.eat()  # 슈크림 맛 붕어빵을 먹어요! 냠냠.
```

---

---

# **init** 이 뭐야?

```
"Initialize (초기화)" 의 약자

객체가 생성되는 순간 자동으로 실행됨
생성할 때 뭘 세팅할지 여기서 정함

fish1 = Bungeoppang("팥", 1000)
           ↑
     여기서 __init__ 이 자동 실행됨
     self.contents = "팥"
     self.price    = 1000
     이 두 줄이 실행됨
```

---

---

# self 는 왜 쓰나?

```
"자기 자신" 을 가리키는 손가락

fish1 과 fish2 는 같은 설계도에서 나왔지만 서로 다른 존재
컴퓨터에게 "지금 말하는 contents 는 fish1 꺼야!" 라고 알려줘야 함

fish1.eat() 실행   ->  self = fish1
fish2.eat() 실행   ->  self = fish2

self 가 없으면 컴퓨터는 "누구 꺼?" 를 모름
```

---

---

# 점(`.`) vs 대괄호(`[]`) — 절대 규칙

```
중괄호 {} 로 만들었으면   ->  ['키'] 로 꺼낸다    (딕셔너리)
대문자 Class() 로 만들었으면  ->  .속성 으로 꺼낸다  (객체)
```

|구분|딕셔너리|객체|
|---|---|---|
|만드는 법|`{"name": "A"}`|`ClassName()`|
|데이터 꺼낼 때|`data['name']`|`obj.name`|
|행동(함수)|없음|`.method()`|
|비유|서랍장|로봇|

```python
# 딕셔너리 -> 대괄호
data = {"name": "서울역", "code": "3900023"}
print(data["name"])   # 서울역

# 객체 -> 점
fish1 = Bungeoppang("팥", 1000)
print(fish1.contents)  # 팥
```

```
Spark 의 broadcast.value 가 헷갈렸던 이유:
Broadcast 클래스 안에 self.value = ... 로 정의되어 있음
점(.) 을 찍으면 그 값을 꺼내오는 것
```

---

---

# TrainInfo 로 보는 실전 클래스

```python
# 클래스 정의 (설계도)
class TrainInfo:
    def __init__(self):                        # 생성될 때 자동 실행
        self.api_key  = "..."                  # 속성: API 키
        self.base_url = "https://..."          # 속성: base URL
        self.session  = requests.Session()     # 속성: 세션 객체

    def get_train_schedule(self, run_ymd):     # 메서드
        ...

    def get_train_realtime(self, run_ymd, trn_no):  # 메서드
        ...


# 객체 생성 (인스턴스화) -> __init__ 이 자동 실행됨
train_info = TrainInfo()
#            ↑ 이 순간 api_key, base_url, session 이 세팅됨

# 메서드 호출
schedule = train_info.get_train_schedule("20260309")
realtime = train_info.get_train_realtime("20260309", "00051")
```

```
왜 클래스로 만들었나?

함수로만 만들면:
  get_train_schedule(api_key, session, base_url, run_ymd)  <- 매번 다 넘겨야 함

클래스로 만들면:
  api_key, session, base_url 는 __init__ 에서 한 번만 세팅
  get_train_schedule(run_ymd)  <- 바뀌는 값만 넘기면 됨
  훨씬 깔끔하고 재사용하기 좋음
```

---

---

# if **name** == "**main**": 이게 뭐야?

```python
# main.py

class TrainInfo:
    ...

if __name__ == "__main__":
    train_info = TrainInfo()
    schedule = train_info.get_train_schedule("20260309")
```

```
파이썬은 파일을 실행하는 방법이 2가지야:

방법 1: 직접 실행
  $ python main.py
  -> __name__ 이 "__main__" 이 됨
  -> if 조건이 True -> 아래 코드 실행됨

방법 2: 다른 파일에서 import
  from main import TrainInfo  (다른 파일에서 가져다 쓸 때)
  -> __name__ 이 "main" (파일 이름) 이 됨
  -> if 조건이 False -> 아래 코드 실행 안 됨
```

```
왜 이걸 쓰나?

if __name__ == "__main__" 없이 그냥 쓰면:
  다른 파일에서 import 할 때
  train_info = TrainInfo() 가 자동으로 실행돼버림
  -> 불필요한 API 호출, 에러 발생 가능

if __name__ == "__main__" 안에 넣으면:
  직접 실행할 때만 돌아감
  import 해서 가져다 쓸 때는 클래스 정의만 가져오고 실행은 안 됨
```

```
정리:
  클래스, 함수 정의   -> 파일 본문에 작성 (어디서든 import 가능)
  실행 코드           -> if __name__ == "__main__": 안에 작성
```

---

---

# Airflow Operator 도 클래스다

```python
from airflow.operators.bash import BashOperator

# BashOperator 클래스로 t1 객체 생성
t1 = BashOperator(
    task_id='print_date',
    bash_command='date'
)

# 속성 꺼내기 (점 사용)
print(t1.task_id)       # print_date
print(t1.bash_command)  # date
```

---

---
# @staticmethod / @classmethod — 메서드 종류 3가지

## 메서드 종류 비교

|종류|self 필요?|언제 쓰나|호출 방법|
|---|---|---|---|
|**일반 메서드**|✅|객체 데이터(self.xxx) 를 쓸 때|`obj.method()`|
|**@staticmethod**|❌|클래스/객체 데이터 안 씀. 그냥 관련 함수|`ClassName.method()` 또는 `obj.method()`|
|**@classmethod**|❌ (cls 사용)|클래스 자체를 다루거나 대안 생성자|`ClassName.method()`|

---
## @staticmethod — "클래스와 관련 있지만 self 가 필요 없는 함수"

```python
class TrainInfo:
    def __init__(self):
        self.api_key = "..."
        self.session = requests.Session()

    # 일반 메서드: self.session 을 써야 하므로 self 필요
    def _request_api(self, endpoint, query_string):
        res = self.session.get(...)
        return res.json()

    # @staticmethod: 날짜 문자열 변환은 self.xxx 를 전혀 안 씀
    # 클래스 밖에 함수로 빼도 되지만 TrainInfo 와 관련 있으니까 안에 묶어둠
    @staticmethod
    def _format_dt(date_str: str, default_text: str) -> str:
        if not date_str or not isinstance(date_str, str) or len(date_str) < 16:
            return default_text
        try:
            dt_obj = datetime.strptime(date_str, "%Y-%m-%d %H:%M:%S.%f")
            return dt_obj.strftime("%H:%M")
        except ValueError:
            return date_str[11:16]
```

```
호출 방법 2가지 모두 가능:
  train_info._format_dt(raw, "출발 전")   # 객체로 호출
  TrainInfo._format_dt(raw, "출발 전")    # 클래스명으로 호출 ← 더 명확

if __name__ == "__main__" 에서 train_info 객체를 안 만들고 호출할 때
TrainInfo._format_dt(...) 처럼 클래스명으로 바로 쓸 수 있음
```

```
언제 @staticmethod 로 만드나?
  → 함수 안에서 self.xxx 나 cls.xxx 를 한 번도 안 쓴다면 staticmethod
  → "이 클래스와 관련 있는 유틸리티 함수" 를 클래스 안에 정리해두고 싶을 때
```

---
## @classmethod — "클래스 자체를 다루는 메서드"


```python
class TrainInfo:
    default_url = "https://apis.data.go.kr/B551457/run/v2"

    def __init__(self, api_key, base_url):
        self.api_key  = api_key
        self.base_url = base_url

    # @classmethod: cls = 클래스 자체를 받음
    # 대안 생성자 패턴: .env 에서 자동으로 api_key 읽어서 객체 생성
    @classmethod
    def from_env(cls):
        api_key = os.getenv("TRAIN_API_KEY")
        return cls(api_key, cls.default_url)  # TrainInfo(api_key, url) 와 동일

# 사용
train_info = TrainInfo.from_env()  # .env 에서 키 자동 로드
```

```
@classmethod 는 데이터 엔지니어링에서 자주 쓰는 패턴:
  설정 파일, 환경변수에서 자동으로 읽어서 객체 만들기
  "대안 생성자 (Alternative Constructor)" 패턴
```

---
## @staticmethod vs @classmethod 한 줄 정리

```
@staticmethod  : self 도 cls 도 없음. 그냥 클래스 안에 묶인 일반 함수
@classmethod   : cls (클래스 자체) 를 받음. 클래스 변수 접근하거나 대안 생성자 만들 때
```

---
---
# 매직 메서드 (Dunder Method) — 심화

> 앞뒤로 언더바 두 개 `__` 가 붙은 특수 메서드 우리가 쓰는 연산자(+, -, //) 는 사실 내부 클래스의 매직 메서드 단축키

```python
# 우리가 아는 연산
print(10 // 3)  # 3

# 파이썬 엔진이 실제로 하는 것
print(int.__floordiv__(10, 3))  # 3  <- 같은 결과

# 코딩테스트 고인물 패턴: 함수 새로 만들지 않고 이름만 바꿔 달기
# 파이썬에서 함수도 변수처럼 담을 수 있음 (일급 객체)
solution = int.__floordiv__
print(solution(10, 3))  # 3
```

```
자주 보이는 매직 메서드들:
  __init__   : 객체 생성 시 자동 실행
  __str__    : print(obj) 할 때 어떻게 출력할지
  __len__    : len(obj) 할 때 동작
  __add__    : obj1 + obj2 할 때 동작
  __floordiv__: obj1 // obj2 할 때 동작
```

---
# @dataclass — 클래스를 간결하게 ⭐️

```
__init__ / __repr__ / __eq__ 를 자동으로 만들어주는 데코레이터
반복적인 보일러플레이트 코드 제거
데이터를 담는 클래스(DTO, VO) 만들 때 특히 유용
```

## 기본 사용

```python
from dataclasses import dataclass

# 기존 클래스 방식
class User:
    def __init__(self, name: str, age: int):
        self.name = name
        self.age = age

    def __repr__(self):
        return f"User(name={self.name}, age={self.age})"

# dataclass 방식 — 훨씬 간결
@dataclass
class User:
    name: str
    age: int

# 자동으로 생성되는 것:
#   __init__(self, name, age)
#   __repr__ → "User(name='gong', age=28)"
#   __eq__   → name 과 age 가 같으면 == True

u1 = User("gong", 28)
u2 = User("gong", 28)
print(u1)        # User(name='gong', age=28)
print(u1 == u2)  # True ← __eq__ 자동 생성
```

## 기본값 설정

```python
@dataclass
class Config:
    host: str = "localhost"
    port: int = 5432
    debug: bool = False

c = Config()                    # 기본값 사용
c = Config(host="prod-db")      # 일부만 지정
c = Config("prod-db", 5435, True)  # 전부 지정
```

## asdict() — 딕셔너리로 변환 ⭐️

```python
from dataclasses import dataclass, asdict

@dataclass
class Hospital:
    hpid: str
    hpname: str
    hvec: int

h = Hospital("A1100001", "삼성서울병원", -26)

# 딕셔너리로 변환
d = asdict(h)
print(d)
# {'hpid': 'A1100001', 'hpname': '삼성서울병원', 'hvec': -26}

# JSON 직렬화할 때 바로 활용
import json
json.dumps(asdict(h))
# '{"hpid": "A1100001", "hpname": "삼성서울병원", "hvec": -26}'
```

```
asdict 활용:
  Kafka 메시지로 보낼 때
  DB INSERT 할 때 딕셔너리로 변환
  API 응답 JSON 만들 때
```

## field() — 고급 설정


```python
from dataclasses import dataclass, field

@dataclass
class Pipeline:
    name: str
    tags: list = field(default_factory=list)   # 리스트 기본값
    meta: dict = field(default_factory=dict)   # 딕셔너리 기본값
    _id: str = field(default="", repr=False)   # repr 에서 제외

# ⚠️ 리스트/딕셔너리는 default= 직접 못 씀
# @dataclass
# class Bad:
#     tags: list = []   # ❌ ValueError → field(default_factory=list) 사용
```

## frozen=True — 불변 객체

```python
@dataclass(frozen=True)
class Point:
    x: float
    y: float

p = Point(1.0, 2.0)
p.x = 3.0   # ❌ FrozenInstanceError (수정 불가)

# 해시 가능 → 딕셔너리 키 / 세트 원소로 사용 가능
d = {p: "좌표"}
```

## 일반 클래스 vs dataclass

|구분|일반 클래스|@dataclass|
|---|---|---|
|`__init__`|직접 작성|자동 생성|
|`__repr__`|직접 작성|자동 생성|
|`__eq__`|직접 작성|자동 생성|
|메서드|자유롭게|자유롭게|
|용도|복잡한 로직|데이터 담는 용도|

```
언제 dataclass 쓰나:
  데이터를 담는 클래스 (DTO / 설정값 / API 응답 파싱)
  __init__ 코드가 반복될 때
  asdict() 로 JSON 변환이 필요할 때

언제 일반 클래스 쓰나:
  복잡한 로직 / 상속 / 프로퍼티가 필요할 때
```