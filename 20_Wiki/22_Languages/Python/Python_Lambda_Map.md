---
aliases:
  - 람다함수
  - Lambda
  - Map_Filter_Reduce
  - 익명함수
  - 고차함수
tags:
  - Python
related:
  - "[[RDD_Concept]]"
  - "[[Spark_Session_Context]]"
  - "[[00_Python_HomePage]]"
  - "[[Python_List_Comprehension]]"
  - "[[Python_Lists_Tuples]]"
---
# Python_Lambda_Map — Lambda & Map/Filter/Reduce

## 한 줄 요약

```
Lambda  이름 짓기 귀찮은 일회용 함수
Map     데이터 하나하나를 변환
Filter  조건 True 인 것만 통과
Reduce  전체를 하나로 압축
```

---

---

# ① Lambda — 일회용 함수

```
def 로 정의해서 이름을 붙이기엔 너무 간단하고
딱 한 번만 쓰고 버릴 함수가 필요할 때 사용
```

## def vs lambda 문법 비교

```
def  함수이름 (입력변수) :
         return  반환값

lambda   입력변수   :   반환값
```

```python
# def 방식 → 별도 줄에 return
def multiply(x, y):
    return x * y

# lambda 방식 → 콜론(:) 뒤가 곧 return, return 키워드 없음
multiply = lambda x, y: x * y
```

```
핵심 암기:
  lambda 에서는 콜론(:) 뒤에 오는 것이 곧 return 값
  return 키워드를 쓰는 순간 SyntaxError
```

## 자주 하는 문법 실수

```python
# ❌ = 쓰면 안 됨
lambda x, y = x * y       # SyntaxError

# ❌ return 쓰면 안 됨
lambda x, y: return x * y # SyntaxError

# ✅ 콜론 뒤가 곧 반환값
lambda x, y: x * y        # 정상
```

## 기본 사용 패턴

```python
# 입력 1개
square = lambda x: x ** 2
square(5)   # 25

# 입력 2개
mul = lambda x, y: x * y
mul(3, 4)   # 12

# 삼항 연산자도 가능
check = lambda x: "양수" if x > 0 else "음수"
check(10)   # '양수'
```

## "정의" vs "실행" 헷갈리지 않기

```python
# ❌ 함수 객체만 리스트에 들어간 것 (실행 안 됨)
answer = [lambda x: sum(x), [1, 2, 3]]
# → [<function <lambda>...>, [1, 2, 3]]

# ✅ 방법 1: 변수에 담고 호출
f = lambda x: sum(x) ** 2
f([1, 2, 3])            # 36

# ✅ 방법 2: 즉시 호출 — (함수)(인자)
(lambda x: sum(x) ** 2)([1, 2, 3])   # 36

# ✅ 방법 3: map/filter 에 태우기 (가장 자연스러움)
list(map(lambda x: x + 10, [1, 2, 3]))  # [11, 12, 13]
```

---

---

# ② Map / Filter / Reduce — 데이터 3대장

## 한눈에 비교

|구분|Map|Filter|Reduce|
|---|---|---|---|
|역할|변환|걸러내기|하나로 압축|
|입력 개수|n개|n개|n개|
|출력 개수|**n개**|**n개 이하**|**1개**|
|lambda 반환값|변환된 값|**True / False**|누적값|
|lambda 인자 수|1개|1개|**2개 (누적값, 현재값)**|

---

## A. Map — 변환 (개수 그대로)

```
데이터 하나하나를 변환
흰 자동차들이 공장에 들어가서 빨간색으로 도색되어 나오는 것
```

```python
prices = [100, 200, 300]

list(map(lambda x: x * 1.1, prices))
# [110.0, 220.0, 330.0]   3개 → 3개 (개수 그대로)
```

---

## B. Filter — 조건 통과 (개수 줄어듦)

```
조건이 True 인 것만 통과
공항 보안 검색대처럼 기준 미달이면 걸러냄
```

```python
ages = [5, 12, 17, 20, 25]

list(filter(lambda x: x >= 20, ages))
# [20, 25]   5개 → 2개 (줄어듦)
```

---

## C. Reduce — 압축 (결과 1개)

```
두 개씩 씹어먹으며 하나로 합침
파이썬 내장이 아니라 import 필요
```

```python
from functools import reduce

nums = [1, 2, 3, 4, 5]

reduce(lambda x, y: x + y, nums)
# 1+2=3 → 3+3=6 → 6+4=10 → 10+5=15
# → 15  (리스트 아님, 숫자 하나)
```

---

---

# ③ list() 는 왜 씌우나 — Lazy Evaluation

```
파이썬 3 부터 map / filter 는 결과를 즉시 만들지 않고
"준비만 된 상태(Iterator)" 를 반환함 → 메모리 절약

list() 로 감싸야 비로소 실제 연산이 실행됨
```

```python
result = map(lambda x: x * 2, [1, 2, 3])
print(result)        # <map object at 0x...>  안 보임
print(list(result))  # [2, 4, 6]  list() 로 감싸야 보임
```

```
Spark 연결고리:
  파이썬 list()  → 결과 강제 실행
  Spark collect() → 결과 강제 실행

  둘 다 Lazy Evaluation 원리 동일
  → [[Spark_Core_Objects]] 참고
```

---

---

# ④ map + lambda 실전 패턴 ⭐️

## 2차원 배열 순회 — x[인덱스] 접근

```
map 의 lambda 인자 x 가 리스트(또는 튜플) 일 때
x[0], x[1], x[2] 로 각 원소에 접근

commands = [[2, 5, 3], [4, 4, 1], [1, 7, 3]]
map(lambda x: ..., commands)
       ↑
  x = [2, 5, 3]  → x[0]=2, x[1]=5, x[2]=3
  x = [4, 4, 1]  → x[0]=4, x[1]=4, x[2]=1
  ...
```

## 배열 자르고 정렬 후 k번째 — 실전 문제 ⭐️

```python
# for 문 방식 (단계별로 명확)
def solution(array, commands):
    answer = []
    for command in commands:
        i, j, k = command            # 언패킹
        sliced = array[i-1:j]        # i번째~j번째 자르기 (1-indexed)
        sorted_arr = sorted(sliced)  # 정렬
        answer.append(sorted_arr[k-1])  # k번째 (1-indexed)
    return answer

# map + lambda 방식 (한 줄)
def solution(array, commands):
    return list(map(lambda x: sorted(array[x[0]-1:x[1]])[x[2]-1], commands))
```

## map + lambda 한 줄 해석

```python
lambda x: sorted(array[x[0]-1:x[1]])[x[2]-1]
#                        ↑              ↑
#          슬라이싱 + 정렬         k번째 꺼내기

# x = [2, 5, 3] 일 때:
#   x[0]-1 = 1   (i=2 → 인덱스 1)
#   x[1]   = 5   (j=5 → 인덱스 5 미포함)
#   x[2]-1 = 2   (k=3 → 인덱스 2)
#
#   array[1:5] = [5, 2, 6, 3]
#   sorted([5, 2, 6, 3]) = [2, 3, 5, 6]
#   [2, 3, 5, 6][2] = 5   ✅
```

```
for 문 vs map+lambda:
  for 문     → 단계별 명확 / 디버깅 쉬움
  map+lambda → 한 줄 / 간결 / 익숙해지면 빠르게 읽힘

  for 문 먼저 짜서 로직 확인
  → 익숙해지면 map+lambda 로 리팩토링
```

## lambda 에서 x[인덱스] 접근 패턴 정리

```python
# x 가 리스트/튜플일 때
list(map(lambda x: x[0] + x[1], [[1,2], [3,4], [5,6]]))
# [3, 7, 11]

# x 가 딕셔너리일 때
data = [{"name": "Kim", "age": 25}, {"name": "Lee", "age": 30}]
list(map(lambda x: x["name"], data))
# ['Kim', 'Lee']

# 정렬 key 로 활용
commands = [[2,5,3], [4,4,1], [1,7,3]]
sorted(commands, key=lambda x: x[2])  # k 값 기준 정렬
# [[4,4,1], [1,7,3], [2,5,3]]
```

# ⑤ 최종 암기 요약

```python
# def 는 return 있음
def f(x): return x * 2

# lambda 는 콜론 뒤가 return (return 쓰지 말 것)
f = lambda x: x * 2

# 입력 여러 개 → 콤마 구분
f = lambda x, y: x * y

# map / filter 는 list() 로 감싸야 결과 보임
list(map(lambda x: x * 2, [1,2,3]))    # [2, 4, 6]
list(filter(lambda x: x > 1, [1,2,3])) # [2, 3]

# 복잡해지면 컴프리헨션이 더 직관적
list(map(lambda x: x**2, filter(lambda x: x%2==0, data)))
[x**2 for x in data if x%2 == 0]  # 같은 결과, 더 읽기 쉬움
```

---

---

# ⑤ Python vs Spark 대응표

|개념|Python|Spark|
|---|---|---|
|변환|`map(...)`|`rdd.map(...)`|
|필터|`filter(...)`|`rdd.filter(...)`|
|집계|`reduce(...)`|`rdd.reduce(...)`|
|결과 수집|`list(...)`|`rdd.collect()`|
|공통 원리|Lazy Evaluation — 강제 실행 전까지 계산 안 함||