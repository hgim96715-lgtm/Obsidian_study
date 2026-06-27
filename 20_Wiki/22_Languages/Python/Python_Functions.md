---
aliases:
  - 함수
  - def
  - args
  - kwargs
  - 가변인자
  - 언패킹
  - 딕셔너리
  - 튜플
tags:
  - Python
related:
  - "[[Airflow_Operators]]"
  - "[[Airflow_Hooks]]"
  - "[[Python_Database_Connect]]"
  - "[[Python_Dictionaries]]"
  - "[[Python_Type_Checking]]"
  - "[[00_Python_HomePage]]"
---
# Python_Functions — 함수

## 한 줄 요약

```
자주 쓰는 코드를 부품처럼 포장해서 이름을 붙인 것
입력 → 처리 → 출력
```

---

---

# ① 기본 구조

```python
def make_coffee(bean_type, water):   # 입력 (파라미터)
    result = f"{bean_type} 커피 + {water}ml"  # 처리
    return result                    # 출력 (반환값)

coffee = make_coffee("에티오피아", 200)
print(coffee)   # "에티오피아 커피 + 200ml"
```

---

---

# ② return vs print — 가장 큰 착각 ⭐️

```
print  화면에 보여주고 끝 (영수증 보여주기)
return 값을 변수에 저장할 수 있게 줌 (실제 물건 건네주기)
```

```python
def print_sum(a, b):
    print(a + b)    # 화면에 5 보임, 끝

def return_sum(a, b):
    return a + b    # 값 5 를 줌

x = print_sum(2, 3)
print(x)   # None  ← 받은 게 없음!

y = return_sum(2, 3)
print(y)   # 5     ← 값을 받아서 y 에 저장
```

---

---

# ③ 기본값 파라미터 (Default Parameter)

```python
# tax_rate 안 넣으면 자동으로 0.1 적용
def calc_price(price, tax_rate=0.1):
    return price * (1 + tax_rate)

calc_price(10000)        # 11000.0  (기본값)
calc_price(10000, 0.2)   # 12000.0  (직접 지정)
```

```
⚠️ 기본값 파라미터는 반드시 맨 뒤에
  def func(a=1, b)    ❌ 에러
  def func(a, b=1)    ✅
```

---

---

# ④ **args — 개수 모를 때 (튜플)

```
입력값이 몇 개인지 모를 때
받은 값들을 튜플(Tuple) 로 묶어서 처리
```

```python
def add_all(*args):
    print(args)      # (1, 2, 3, 4, 5) ← 튜플
    return sum(args)

add_all(1, 2)          # 3
add_all(1, 2, 3, 4, 5) # 15
```

---

---

# ⑤ **kwargs — 키=값 형태로 받기 (딕셔너리)

```
키=값 형태로 입력하면 딕셔너리로 받음
Keyword Arguments 의 약자
```

```python
def introduce(**kwargs):
    print(kwargs)   # {'name': 'Gong', 'age': 20} ← 딕셔너리
    if 'name' in kwargs:
        print(f"이름: {kwargs['name']}")

introduce(name="Gong", age=20, city="Seoul")
```

## ** 딕셔너리 언패킹 — 실전 활용 ⭐️

```
이미 만들어진 딕셔너리를 함수에 한번에 넣을 때
```

```python
def save_user(name, age, city):
    print(f"{name}({age}세) / {city}")

user_data = {"name": "Kim", "age": 30, "city": "Busan"}

# ❌ 하나씩 꺼내기 (귀찮음)
save_user(user_data["name"], user_data["age"], user_data["city"])

# ✅ 딕셔너리 언패킹 (한방에)
save_user(**user_data)
```

```python
# DE 실전: API params / DB 설정을 딕셔너리로 관리
params = {
    "serviceKey": API_KEY,
    "numOfRows":  "100",
    "pageNo":     "1"
}
requests.get(url, params=params)   # **params 와 동일
```

---

---

# ⑥ Airflow 에서 **kwargs 쓰는 이유

```
Airflow 는 태스크 실행 시 수십 개의 정보를 함수에 던져줌
  실행 날짜 (ds)
  태스크 인스턴스 (ti)
  파라미터 (params)
  ...

개발자가 이걸 일일이 다 파라미터로 적을 수 없음
→ **kwargs 로 한번에 받아서 필요한 것만 꺼내 씀
```

```python
# Airflow DAG 태스크 함수
def run_task(**kwargs):
    ds     = kwargs["ds"]           # 실행 날짜
    params = kwargs["params"]       # 파라미터
    ti     = kwargs["ti"]           # 태스크 인스턴스
    print(f"실행 날짜: {ds}")
```

---

---

# ⑦ 타입 힌트 (Type Hint)

```
파라미터와 반환값의 타입을 명시
강제성 없음 → 코드 가독성 + IDE 자동완성 지원
```

```python
def greeting(name: str, age: int) -> str:
    return f"{name} is {age} years old."

# 반환값이 없으면 -> None
def log(msg: str) -> None:
    print(msg)

# Optional (None 일 수도 있을 때)
from typing import Optional
def find_user(user_id: int) -> Optional[str]:
    ...

# 여러 타입 (Python 3.10+)
def fetch(endpoint: str) -> dict | None:
    ...
```

> 타입 힌트 문법 → [[Python_Type_Checking#typing 모듈]] 참고

---

---

# 인자 순서 규칙

```
def func(일반변수, 기본값변수, *args, **kwargs)
          ①       ②        ③       ④

반드시 이 순서
**kwargs 가 항상 맨 마지막
```

```python
def func(a, b=10, *args, **kwargs):
    print(a, b, args, kwargs)

func(1, 2, 3, 4, x=5, y=6)
# a=1, b=2, args=(3,4), kwargs={'x':5,'y':6}
```

---

---

# 한눈에 정리

|문법|의미|내부 자료형|
|---|---|---|
|`def func(a, b)`|일반 파라미터|변수|
|`def func(a=10)`|기본값 파라미터|변수|
|`def func(*args)`|개수 무제한|튜플|
|`def func(**kwargs)`|키=값 무제한|딕셔너리|
|`func(**my_dict)`|딕셔너리 언패킹|-|