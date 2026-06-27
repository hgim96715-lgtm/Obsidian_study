---
aliases:
  - itertools
  - combinations
  - permutations
  - product
  - goupby
  - accumulate
  - chain
tags:
  - Python
related:
  - "[[00_Python_HomePage]]"
  - "[[Python_Collections_Modules]]"
  - "[[Python_Looping_Helpers]]"
  - "[[Python_Math_Module]]"
---

# Python_Itertools — 반복자 도구 모음

## 한 줄 요약

```
반복문으로 직접 짜면 복잡한 것들을
한 줄로 처리해주는 표준 라이브러리

조합 / 순열 / 중복 조합 / 데카르트 곱
연속 구간 / 무한 반복 등
```

```python
from itertools import combinations, permutations, product
from itertools import combinations_with_replacement
from itertools import chain, groupby, accumulate
```

---

---

# ① combinations — 조합 (순서 무관) ⭐️

```
nCr: n개에서 r개를 순서 없이 뽑기
같은 원소 중복 선택 없음
```

```python
from itertools import combinations

# 기본 사용
list(combinations([1, 2, 3, 4], 2))
# [(1,2), (1,3), (1,4), (2,3), (2,4), (3,4)]

list(combinations("ABCD", 3))
# [('A','B','C'), ('A','B','D'), ('A','C','D'), ('B','C','D')]
```

## range(i+1) 패턴과 비교 ⭐️

```python
arr = [-2, 3, 0, 2, -5]

# 방법 1 — range(i+1) 3중 반복문
answer = 0
for n1 in range(len(arr)):
    for n2 in range(n1 + 1, len(arr)):
        for n3 in range(n2 + 1, len(arr)):
            if arr[n1] + arr[n2] + arr[n3] == 0:
                answer += 1

# 방법 2 — combinations (더 간결)
from itertools import combinations
answer = sum(1 for combo in combinations(arr, 3) if sum(combo) == 0)

# 결과 동일
```

```
언제 어느 걸 쓰나:
  range(i+1) → 인덱스가 직접 필요할 때 / combinations 모르는 환경
  combinations → 조합 자체만 필요할 때 / 코드 간결하게
```

## 실전 패턴

```python
# 3명의 합이 0인 조합 수
def solution(number):
    from itertools import combinations
    return sum(1 for combo in combinations(number, 3) if sum(combo) == 0)

# 모든 2개 쌍 비교
from itertools import combinations
scores = [85, 90, 78, 95]

for a, b in combinations(scores, 2):
    print(f"{a} vs {b} → 차이: {abs(a-b)}")

# 합이 특정 값인 조합 찾기
target = 10
nums = [1, 3, 5, 7, 2]
result = [c for c in combinations(nums, 2) if sum(c) == target]
# [(3, 7), (5, 5는없음)]
```

---

---

# ② permutations — 순열 (순서 있음) ⭐️

```
nPr: n개에서 r개를 순서 있게 뽑기
(A,B) 와 (B,A) 를 다른 것으로 봄
```

```python
from itertools import permutations

list(permutations([1, 2, 3], 2))
# [(1,2), (1,3), (2,1), (2,3), (3,1), (3,2)]
# ↑ combinations 과 달리 (1,2) 와 (2,1) 둘 다 있음

list(permutations("ABC"))   # r 생략 = 전체 순열
# [('A','B','C'), ('A','C','B'), ('B','A','C'), ...]  6개
```

## combinations vs permutations 비교

```python
from itertools import combinations, permutations

data = [1, 2, 3]

print(list(combinations(data, 2)))
# [(1,2), (1,3), (2,3)]  ← 3개 (순서 무관)

print(list(permutations(data, 2)))
# [(1,2), (1,3), (2,1), (2,3), (3,1), (3,2)]  ← 6개 (순서 있음)
```

```
조합 (combinations):
  {A,B} 와 {B,A} 를 같은 것으로 봄
  수학의 nCr = n! / (r! × (n-r)!)

순열 (permutations):
  (A,B) 와 (B,A) 를 다른 것으로 봄
  수학의 nPr = n! / (n-r)!

언제 어느 걸:
  "자리 배치 / 순서 중요" → permutations
  "팀 구성 / 선택 조합"   → combinations
```

---

---

# ③ product — 데카르트 곱 (중복 허용 전체 조합)

```
여러 iterable 의 모든 경우의 수
중첩 for 문을 한 줄로
```

```python
from itertools import product

# 두 리스트의 모든 쌍
list(product([1, 2], ['A', 'B']))
# [(1,'A'), (1,'B'), (2,'A'), (2,'B')]

# 주사위 2개 던지기
list(product(range(1, 7), repeat=2))
# [(1,1), (1,2), ..., (6,6)]  36가지

# 비밀번호 0~9 중 3자리
list(product(range(10), repeat=3))
# [(0,0,0), (0,0,1), ..., (9,9,9)]  1000가지
```

```python
# 중첩 for 문과 비교
# ❌ 중첩 for
for a in [1, 2]:
    for b in ['A', 'B']:
        print(a, b)

# ✅ product
for a, b in product([1, 2], ['A', 'B']):
    print(a, b)
```

---

---

# ④ combinations_with_replacement — 중복 조합

```
같은 원소를 여러 번 뽑을 수 있는 조합
nHr: 중복 조합
```

```python
from itertools import combinations_with_replacement

list(combinations_with_replacement([1, 2, 3], 2))
# [(1,1), (1,2), (1,3), (2,2), (2,3), (3,3)]
# ↑ (1,1) 처럼 같은 원소 중복 가능
```

|함수|중복|순서|수식|
|---|---|---|---|
|`combinations`|✗|✗|nCr|
|`permutations`|✗|✅|nPr|
|`product`|✅|✅|n^r|
|`combinations_with_replacement`|✅|✗|nHr|

---

---

# ⑤ chain — 여러 iterable 이어붙이기

```python
from itertools import chain

list(chain([1, 2], [3, 4], [5]))
# [1, 2, 3, 4, 5]

# 중첩 리스트 평탄화
nested = [[1, 2], [3, 4], [5, 6]]
list(chain.from_iterable(nested))
# [1, 2, 3, 4, 5, 6]
```

---

---

# ⑥ accumulate — 누적 계산

```python
from itertools import accumulate

list(accumulate([1, 2, 3, 4, 5]))
# [1, 3, 6, 10, 15]  ← 누적합 (기본값)

import operator
list(accumulate([1, 2, 3, 4, 5], operator.mul))
# [1, 2, 6, 24, 120]  ← 누적곱 (팩토리얼)
```

---

---

# ⑦ groupby — 연속 같은 값 묶기

```python
from itertools import groupby

data = [1, 1, 2, 2, 2, 3, 1, 1]

for key, group in groupby(data):
    print(key, list(group))
# 1 [1, 1]
# 2 [2, 2, 2]
# 3 [3]
# 1 [1, 1]   ← 연속 기준이라 중간에 끊기면 별도 그룹
```

```
⚠️ 정렬 후 사용해야 전체 그룹화 가능
   정렬 없이 사용하면 연속된 것만 묶음
```

```python
# 정렬 후 사용
data = [1, 2, 1, 2, 3]
for key, group in groupby(sorted(data)):
    print(key, list(group))
# 1 [1, 1]
# 2 [2, 2]
# 3 [3]
```

---

---

# 한눈에 정리

|함수|역할|예시|
|---|---|---|
|`combinations(it, r)`|순서 무관 조합|팀 구성, 쌍 비교|
|`permutations(it, r)`|순서 있는 순열|자리 배치, 비밀번호|
|`product(*its, repeat)`|데카르트 곱|주사위, 경우의 수|
|`combinations_with_replacement(it, r)`|중복 조합|중복 허용 선택|
|`chain(*its)`|iterable 이어붙이기|중첩 리스트 평탄화|
|`accumulate(it)`|누적 계산|누적합, 누적곱|
|`groupby(it)`|연속 같은 값 묶기|연속 구간 분석|

---

---

# 자주 하는 실수

|실수|원인|해결|
|---|---|---|
|`combinations(...)` 결과가 보이지 않음|이터레이터 반환|`list(...)` 로 변환|
|combinations 에서 (1,2) 와 (2,1) 둘 다 원함|순서 있는 조합|`permutations` 사용|
|groupby 결과가 이상함|정렬 없이 사용|`sorted()` 후 groupby|
|product repeat 빠트림|같은 iterable 반복|`product(range(10), repeat=3)`|