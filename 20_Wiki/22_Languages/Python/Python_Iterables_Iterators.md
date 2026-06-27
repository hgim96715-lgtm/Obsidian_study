---
aliases:
  - 반복자
  - Iterable
  - Iterator
  - 반복 가능한 객체
tags:
  - Python
related:
  - "[[00_Python_HomePage]]"
  - "[[Python_Generators_Yield]]"
  - "[[Python_Lists_Tuples]]"
  - "[[Python_Control_Flow]]"
  - "[[Python_Sets]]"
---
# Python_Iterables_Iterators — 이터러블과 이터레이터

## 한 줄 요약

```
Iterable  = for 문에 넣을 수 있는 객체 (데이터 보따리)
Iterator  = 데이터를 하나씩 꺼내주는 포인터 (손가락)
```

---

---

# ① 핵심 개념

```
Iterable (이터러블):
  for 문에 넣을 수 있는 모든 객체
  리스트 / 튜플 / 문자열 / 딕셔너리 / range() 등
  데이터를 가지고 있는 "보따리"

Iterator (이터레이터):
  next() 로 하나씩 꺼낼 수 있는 객체
  현재 위치를 기억하는 "손가락(포인터)"
  일방통행 → 한 번 소진되면 재사용 불가

관계:
  Iterable → iter() → Iterator
  Iterator → next() → 값 하나씩
```

## 도시락 비유

```
Iterable = 도시락통 (내용물 전체 보관)
Iterator = 젓가락  (하나씩 꺼내는 동작)

도시락통(Iterable) 이 있어도
젓가락(Iterator) 없이는 꺼낼 수 없음
for 문 = 내부적으로 젓가락을 자동으로 만들어서 사용
```

---

---

# ② 기본 사용법

```python
# Iterable → for 문에 바로 사용 가능
my_list = [1, 2, 3]

for item in my_list:
    print(item)    # 1 / 2 / 3

# 문자열도 Iterable
for char in "Hello":
    print(char)    # H / e / l / l / o
```

## iter() / next() 직접 사용

```python
lunch_box = ["Sandwich", "Apple", "Juice"]

# iter() → Iterable 에서 Iterator 생성
waiter = iter(lunch_box)
print(waiter)         # <list_iterator object at 0x...>
                      # 내용물이 아니라 "포인터" 객체

# next() → 하나씩 꺼내기
print(next(waiter))   # Sandwich
print(next(waiter))   # Apple
print(next(waiter))   # Juice
print(next(waiter))   # StopIteration 예외 발생!
```

```
for 문 내부 동작:
  for item in my_list: 는 사실 아래와 동일

  _iter = iter(my_list)
  while True:
      try:
          item = next(_iter)
          # 본문 실행
      except StopIteration:
          break
```

---

---

# ③ Iterable vs Iterator 차이 ⭐️

```python
my_list = [1, 2, 3]

# Iterable
iter(my_list)         # ✅ 가능 → Iterator 반환
my_list[0]            # ✅ 인덱스 접근 가능
for x in my_list:     # ✅ 반복 가능 (여러 번 해도 됨)
    pass
for x in my_list:     # ✅ 다시 반복 가능
    pass

# Iterator
it = iter(my_list)
next(it)              # ✅ 1
it[0]                 # ❌ TypeError: 인덱스 접근 불가
for x in it:          # ✅ 남은 것만 (2, 3)
    pass
for x in it:          # ❌ 아무것도 안 나옴 (이미 소진)
    pass
```

|구분|Iterable|Iterator|
|---|---|---|
|인덱스 접근|✅|❌|
|재사용|✅ (몇 번이든)|❌ (일회용)|
|`iter()` 호출|→ Iterator 반환|→ 자기 자신 반환|
|`next()` 호출|❌|✅|
|메모리|전체 보관|현재 위치만|

---

---

# ④ 판별법

```python
# Iterable 판별
def is_iterable(obj):
    try:
        iter(obj)
        return True
    except TypeError:
        return False

is_iterable([1, 2, 3])   # True
is_iterable("hello")     # True
is_iterable(123)          # False (정수는 Iterable 아님)

# hasattr 로 확인
hasattr([1,2,3], '__iter__')   # True
hasattr(123, '__iter__')       # False
```

---

---

# ⑤ **iter** / **next** — 내부 동작 원리

```python
# Iterable 은 __iter__ 메서드를 가짐
# Iterator 는 __iter__ + __next__ 둘 다 가짐

class MyIterator:
    def __init__(self, data):
        self.data = data
        self.index = 0

    def __iter__(self):
        return self          # 자기 자신 반환

    def __next__(self):
        if self.index >= len(self.data):
            raise StopIteration   # 끝나면 예외
        value = self.data[self.index]
        self.index += 1
        return value

it = MyIterator([10, 20, 30])
print(next(it))   # 10
print(next(it))   # 20
for x in it:      # 나머지 30
    print(x)
```

---

---

# ⑥ 일회용성 주의 ⭐️

```python
my_list = [1, 2, 3]
it = iter(my_list)

# 일부 소비
next(it)   # 1 꺼냄
next(it)   # 2 꺼냄

# 나머지만 남음
list(it)   # [3]  ← 1, 2 는 이미 소진

# 다시 시도
list(it)   # []   ← 완전히 빈 껍데기
```

```python
# ⚠️ list(elements) 함정
# Iterator 를 list() 로 변환하면 내부적으로 next() 를 끝까지 호출
# → Iterator 소진됨

def process(elements):
    result = list(elements)   # ← 여기서 elements 소진!
    total = sum(elements)     # ← elements 가 비어서 0 나옴!
    return result, total

# ✅ 해결
def process(elements):
    result = list(elements)   # 리스트로 변환
    total = sum(result)       # result 사용
    return result, total
```

---

---

# ⑦ 왜 필요한가 — 메모리 효율 ⭐️

```
리스트 방식:
  [1, 2, 3, ... 1억] 전체를 메모리에 올림
  → 1억 개 × 8바이트 = 800MB 필요 → 서버 폭발

Iterator 방식:
  현재 위치(포인터) 만 메모리에 유지
  next() 호출할 때마다 하나씩 계산
  → 메모리 거의 0에 가까움

Lazy Evaluation (게으른 평가):
  미리 다 만들지 않고
  필요할 때만 계산해서 가져옴
  → 끝을 알 수 없는 실시간 스트림에 필수
```

## Flink 실무 연결

```python
# PyFlink ProcessWindowFunction 에서
# elements 는 리스트가 아닌 Iterator 로 전달됨

def process(key, context, elements, out):
    # ❌ 이렇게 하면 elements 소진 후 total = 0
    result = list(elements)
    total = sum(elements)

    # ✅ list 로 한 번만 변환 후 재사용
    element_list = list(elements)
    total = sum(e.value for e in element_list)
    count = len(element_list)

# Flink 가 리스트 대신 Iterator 를 주는 이유:
#   윈도우 데이터가 수백만 건이 될 수 있음
#   리스트로 주면 메모리 즉사
#   → Iterator 로 안전하게 전달
```

---

---

# 자주 하는 실수

|실수|원인|해결|
|---|---|---|
|소진된 Iterator 재사용|일회용|`iter()` 로 새로 생성|
|`list(elements)` 후 `sum(elements)`|elements 소진|`result = list(elements)` 후 `sum(result)`|
|Iterator 에 인덱스 접근|Iterator 는 인덱스 없음|`list(it)` 로 변환 후 접근|
|`for` 문 두 번 돌렸는데 두 번째 빈 값|Iterator 소진|Iterable 로 `for` / Iterator 재생성|