---
aliases:
  - Python Generator
  - yield 키워드
  - 제너레이터
  - Lazy Evaluation
tags:
  - Python
related:
  - "[[Python_Functions]]"
  - "[[Python_Looping_Helpers]]"
  - "[[00_Python_HomePage]]"
  - "[[00_Python_HomePage]]"
  - "[[Python_Iterables_Iterators]]"
---
## Concept Summary

**"필요할 때 딱 하나만 꺼내 쓰는 '크리넥스 티슈' 같은 녀석."** 
미리 다 뽑아놓은 티슈 더미(List)가 아니라, 내가 손을 뻗을 때마다(Call) 박스 안에서 한 장씩 톡 뽑아주는(Yield) **지연 평가(Lazy Evaluation)** 방식의 이터레이터입니다.

---
## 왜 필요한가? (Why)

### 1. 메모리 폭발 방지 (OOM 방어)

- **List:** 데이터가 1억 개면, 1억 개를 전부 메모리에 올린 뒤에야 for문을 돌릴 수 있습니다. (RAM 부족으로 사망)
- **Generator:** 데이터가 1억 개든 무한대든, **"지금 처리할 1개"** 의 메모리만 씁니다.

### 2. 응답 속도 향상 (Latency)

- **List:** 1만 개의 데이터를 가공할 때, 1만 개가 다 끝날 때까지 기다려야 첫 번째 결과를 볼 수 있습니다.
- **Generator:** 첫 번째 데이터 가공이 끝나자마자 즉시(`yield`) 결과를 던져줍니다. (유저 입장에서 빠르다고 느꼇짐)

### 3. 무한 데이터 표현 (Infinite Stream)

- 끝이 없는 데이터(센서 데이터, 로그 스트림)는 리스트로 담을 수 없습니다. 
- 제너레이터는 **"끝이 없는 수열"** 을 코드로 구현할 수 있는 유일한 방법입니다.

---
## Practical Context (실무 활용)

### ① 대용량 파일 처리 (Log Parsing)

50GB짜리 로그 파일을 분석할 때 절대 `read()`나 `readlines()`를 쓰지 않습니다.

```python
# [실무 패턴] 파일이 아무리 커도 메모리는 몇 KB만 씀
def read_huge_file(file_path):
    with open(file_path, 'r') as f:
        for line in f:
            yield line.strip() # 한 줄 읽고 던지고, 메모리에서 지움
```

### ② 파이프라인 구축 (Chaining)

데이터 엔지니어링에서는 여러 제너레이터를 연결해서 **파이프라인**을 만듭니다. (마치 공장 컨베이어 벨트처럼)
`Log Read` -> `Parsing` -> `Filtering` -> `DB Insert` 과정을 제너레이터로 연결하면 중간 데이터 저장 없이 물 흐르듯 처리됩니다.


---
## Code Core Points: 동작 원리 해부

### ① `yield`는 "일시 정지(Pause)" 버튼이다

함수가 실행되다가 `yield`를 만나면 **값을 던져주고(Return), 그 자리에서 얼음(Pause)** 상태가 됩니다. 
그리고 다음에 호출하면 **얼음 땡(Resume)** 하고, **변수 값들을 그대로 기억한 채** 바로 다음 줄부터 실행합니다.

```python
def my_gen():
    print("첫 번째 작업 시작")
    n = 1
    yield n  # 1을 주고 여기서 멈춤! (n=1 기억함)

    print("두 번째 작업 시작")
    n += 1
    yield n  # 2를 주고 여기서 멈춤! (n=2 기억함)

# 사용
g = my_gen()
print(next(g)) # 출력: "첫 번째 작업 시작" -> 1
print(next(g)) # 출력: "두 번째 작업 시작" -> 2
```

### ② 제너레이터 표현식 (Generator Expression) ⭐️

리스트 컴프리헨션에서 대괄호 `[]`를 소괄호 `()`로 바꾸면 바로 제너레이터가 됩니다. (실무 꿀팁)

```python
# [List] 메모리에 100만 개 다 만듦 (무거움)
my_list = [i * 2 for i in range(1000000)] 

# [Generator] 생성할 '준비'만 해둠 (가벼움)
my_gen = (i * 2 for i in range(1000000)) 

import sys
print(sys.getsizeof(my_list)) # 약 8MB
print(sys.getsizeof(my_gen))  # 딱 112 Byte (크기 고정)
```

---
## Detailed Analysis: PyFlink와의 관계

PyFlink의 `KeyedProcessFunction`이나 `process_element`가 왜 `return`이 아니라 `yield`를 쓸까요?

1. **Flink는 스트림입니다:** 데이터가 끊임없이 들어오는데 `return`을 해버리면 함수가 아예 종료되어 버립니다.
2. **상태 유지:** `yield`를 쓰면 함수가 끝나지 않고 대기 상태로 남아있기 때문에, 다음 데이터 처리를 위해 **Context를 유지**하는 데 유리한 구조를 가집니다. (물론 Flink는 별도의 State Backend를 쓰지만, 파이썬 레벨에서의 처리는 제너레이터 패턴이 적합합니다.)

```python
# PyFlink 내부적인 느낌 (의사 코드)
def process_element(self, value, ctx):
    # 복잡한 계산 후...
    yield result  # Flink야, 이거 가져가고 난 다음 데이터 기다릴게!
```

---
## 제네레이터랑 이터레이터랑 뭐가다르지?

>"제네레이터는 이터레이터를 만드는 가장 쉽고 간편한 방법(Syntactic Sugar)"
>모든 **제네레이터는 이터레이터의 일종**이지만, 제네레이터가 훨씬 더 코드를 짜기 쉽고(함수형), 상태 관리(State)를 자동으로 해준다

- **Iterable (반복 가능한 것):** `for` 문을 돌릴 수 있는 모든 것 (리스트, 튜플, 딕셔너리 등).
- **Iterator (반복자):** `next()` 함수를 호출할 때마다 값을 하나씩 주는 **객체(Class)**.
- **Generator (생성기):** `yield`를 사용해 만든 **특별한 형태의 이터레이터**.

| **특징**    | **Iterator (이터레이터)**                               | **Generator (제네레이터)**                |
| --------- | -------------------------------------------------- | ------------------------------------ |
| **만드는 법** | **클래스(Class)** 로 만듦                                | **함수(Function)** 로 만듦                |
| **필수 요소** | `__iter__`와 `__next__` 메서드를<br><br>직접 구현해야 함 (귀찮음) | 그냥 **`yield`** 만 쓰면 끝<br><br>(간편함)   |
| **상태 관리** | `self.idx` 처럼 변수를<br>수동으로 관리해야 함                   | 함수 내부 변수가<br><br>**자동으로 저장/복구**됨     |
| **종료 처리** | `StopIteration` 에러를<br><br>직접 발생시켜야 함              | 함수가 끝나면 알아서 종료됨                      |
| **주요 용도** | 복잡한 동작이 필요한<br><br>커스텀 자료구조                        | 데이터 파이프라인,<br><br>스트림 처리 (PyFlink 등) |



---
## Common Beginner Misconceptions (흔한 오해)

### Q1. "리스트랑 사용법이 똑같은데 그냥 리스트 쓰면 안 돼요?"

- **절대 안 됩니다.** 
- 데이터가 작으면 상관없지만, 현업 데이터(GB 단위)를 리스트로 다루면 서버가 죽습니다(OOM). 
- 습관적으로 제너레이터를 고려해야 합니다.

### Q2. "인덱싱(`gen[0]`)이나 슬라이싱(`gen[:3]`)이 안 돼요!"

- **네, 안 됩니다.** 제너레이터는 "다음에 나올 게 뭔지"만 알지, 전체 지도가 없습니다.
- **해결:** 꼭 필요하면 `itertools.islice`를 쓰거나, `list(gen)`으로 변환해야 합니다(메모리 주의).

### Q3. "for 문을 두 번 돌렸는데 두 번째엔 아무것도 안 나와요."

- **정상입니다.** 제너레이터는 **"일회용 티슈"** 입니다. 
- 한 번 다 뽑아 쓰면(Iteration이 끝나면) 빈 껍데기만 남습니다. 
- 다시 쓰려면 제너레이터를 **다시 생성(함수 재호출)** 해야 합니다.


