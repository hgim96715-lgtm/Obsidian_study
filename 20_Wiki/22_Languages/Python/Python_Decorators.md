---
aliases:
  - 데코레이터
  - Decorator
  - "@"
  - TakFlow
tags:
  - Python
related:
  - "[[Airflow_DAG_Skeleton]]"
  - "[[Python_Functions]]"
  - "[[Airflow_XComs]]"
  - "[[Airflow_TaskFlow_API]]"
  - "[[Python_Classes_Objects]]"
  - "[[Python_Inheritance]]"
---
# Python_Decorators — 데코레이터

## 한 줄 요약

```txt
선물 상자의 "포장지"

내용물(기존 함수)은 건드리지 않고
겉에 추가 기능을 덧붙여서 새로운 기능을 입히는 문법
기호: @ (골뱅이)
```

---

---

# 왜 필요한가?

```txt
문제: 함수 100개에 실행 시간 로그를 남기고 싶다
     100개 함수에 print 문을 일일이 복사? → 유지보수 지옥

해결: "시간 측정 기능" 데코레이터 하나 만들어두고
     필요한 함수 위에 @timer 만 붙이면 끝
```

---

---

# ① 기본 구조

```txt
데코레이터 = "함수를 받아서, 함수를 리턴하는 함수"

구조:
  def decorator(func):      ← 원래 함수를 인자로 받음
      def wrapper():        ← 기능을 추가한 새 함수
          # 실행 전 처리
          func()            ← 원래 함수 실행
          # 실행 후 처리
      return wrapper        ← 새 함수를 반환
```

```python
# 1. 데코레이터 정의
def my_decorator(func):
    def wrapper():
        print("함수 실행 전")
        func()              # 원래 함수 실행
        print("함수 실행 후")
    return wrapper

# 2. 적용 (@골뱅이)
@my_decorator
def say_hello():
    print("Hello!")

# 3. 실행
say_hello()
```

```txt
출력:
  함수 실행 전
  Hello!
  함수 실행 후
```

## @ 문법의 정체

```python
# @my_decorator 는 아래 코드와 완전히 동일
say_hello = my_decorator(say_hello)

# 즉, say_hello 를 my_decorator 로 포장해서 재할당
```

---

---

# ② 실용 예제 — 실행 시간 측정

```python
import time

def timer(func):
    def wrapper(*args, **kwargs):    # *args, **kwargs: 원래 함수의 인자 그대로 받기
        start  = time.time()
        result = func(*args, **kwargs)
        end    = time.time()
        print(f"{func.__name__} 실행 시간: {end - start:.4f}초")
        return result                # 원래 함수의 반환값도 그대로 전달
    return wrapper

@timer
def slow_function():
    time.sleep(1)
    return "완료"

slow_function()
# slow_function 실행 시간: 1.0012초
```

```txt
*args, **kwargs 를 쓰는 이유:
  데코레이터는 어떤 함수에든 붙을 수 있어야 함
  → 원래 함수의 인자가 몇 개든 그대로 받아서 그대로 넘겨줌
```

---

---

# ③ 여러 개 쌓기 (스택)

```python
@decorator_A
@decorator_B
@decorator_C
def my_func():
    pass
```

```txt
코드 읽는 순서:  위 → 아래  (A → B → C 순으로 씌워짐)
실제 실행 순서:  아래 → 위  (C가 가장 안쪽, A가 가장 바깥)

= decorator_A(decorator_B(decorator_C(my_func)))
```

---

---

# ④ functools.wraps — 함수 이름 보존

```txt
데코레이터를 쓰면 함수 이름이 wrapper 로 바뀌어버림
→ 디버깅할 때 헷갈림

functools.wraps 로 원본 함수 이름 보존
```

```python
from functools import wraps

def my_decorator(func):
    @wraps(func)            # ← 이 한 줄로 이름 보존
    def wrapper(*args, **kwargs):
        print("실행 전")
        return func(*args, **kwargs)
    return wrapper

@my_decorator
def say_hello():
    pass

print(say_hello.__name__)   # say_hello  ✅ (wraps 없으면 wrapper 가 나옴)
```

---

---

# ⑤ Airflow 에서의 데코레이터 3대장

```txt
Airflow 2.0 부터 데코레이터 방식이 표준
→ 데코레이터를 모르면 Airflow 코드를 읽을 수 없음
```

|데코레이터|역할|
|---|---|
|`@dag`|함수 전체를 DAG 파이프라인으로 선언|
|`@task`|함수를 PythonOperator 태스크로 변환|
|`@task_group`|태스크들을 폴더(그룹) 로 묶어서 표시|

```python
from airflow.decorators import dag, task

@dag(schedule='@daily', ...)
def my_pipeline():

    @task
    def extract():
        return {'data': 100}    # return → 자동 XCom Push

    @task
    def load(data):             # 인자로 받음 → 자동 XCom Pull
        print(data)

    load(extract())             # 순서 + 데이터 전달 한 번에

dag = my_pipeline()
```

---

---

# ⑥ 함수 위치 — DAG 안 vs 밖

```txt
Outside (DAG 밖에 정의):
  "도장을 파는 행위" — 기능만 만들어두는 것
  다른 DAG 파일에서 import 해서 재사용 가능

Inside (DAG 안에서 호출):
  "도장을 종이에 찍는 행위" — DAG 흐름에 실제 배치
```

```python
from airflow.decorators import task

# 밖에서 정의 → dag1, dag2 어디서든 재사용 가능
@task
def send_slack_alert(message):
    # 슬랙 알림 전송
    pass

@task
def preprocess(data):
    return data.strip()

with DAG(dag_id='my_dag', ...) as dag:
    # 안에서 호출 → 이 DAG 의 태스크로 등록
    send_slack_alert("파이프라인 시작")
    preprocess("  raw data  ")
```

```txt
공통 태스크 (슬랙 알림, 전처리 등) 는
utils.py 에 정의해두고 여러 DAG 에서 import 해서 쓰는 것이 고수 방식
```

---

---

# 초보자가 자주 착각하는 것들

## ① 함수 이름이 바뀐다

```python
@my_decorator
def say_hello():
    pass

print(say_hello.__name__)   # ❌ wrapper  (데코레이터 적용 후 이름 바뀜)

# ✅ functools.wraps 로 해결
```

## ② @task 함수 안 vs 밖 실행 위치

```txt
@task 함수 안의 코드  → 워커(Worker) 에서 실행
@task 함수 밖의 코드  → 스케줄러에서 실행

→ @task 밖에서 전역 변수나 외부 상태에 의존하면 안 됨
```

## ③ Airflow 2.0 에서는 선택이 아닌 필수

```txt
@task / @dag 없이도 PythonOperator 로 쓸 수 있지만
코드 길이와 복잡도가 몇 배 늘어남
→ Airflow 2.0 이상이면 TaskFlow API 가 표준
```

> 상세 사용법 → [[Airflow_TaskFlow_API]]