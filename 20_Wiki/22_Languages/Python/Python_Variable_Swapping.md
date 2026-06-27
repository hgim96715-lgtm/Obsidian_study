---
aliases:
  - 파이썬 변수 교환
  - Swap
  - 튜플 언패킹
  - a,b=b,a
tags:
  - Python
related:
  - "[[00_Python_HomePage]]"
  - "[[Python_Lists_Tuples]]"
  - "[[Python_Unpacking]]"
  - "[[Python_Sorting_Logic]]"
---
# Python_Variable_Swapping — 변수 교환

## 한 줄 요약

```
a, b = b, a
임시 변수 없이 한 줄로 두 값을 맞바꿈
내부적으로 튜플이 생성됐다가 언패킹되는 것
```

---

---

# ① 기본 — temp 없이 한 줄로

```python
# C / Java 방식 — temp 필요
temp = a
a = b
b = temp   # 3줄 + 쓸데없는 변수

# Python 방식 — 한 줄
a, b = b, a
```

## 동작 원리

```python
a, b = 10, 20

a, b = b, a
# 1. 우변 평가 → (b, a) = (20, 10) 튜플 생성
# 2. 좌변 언패킹 → a = 20 / b = 10

print(a, b)   # 20 10
```

```
눈에는 한 줄이지만 내부적으로:
  우변이 먼저 튜플 (b, a) 로 평가됨
  → 기존 값이 고정된 뒤 좌변에 언패킹
  → 임시 변수 없어도 덮어쓰기 충돌 없음
```

---

---

# ② 리스트 원소 교환

```python
arr = [10, 20, 30, 40]
i, j = 0, 3

arr[i], arr[j] = arr[j], arr[i]
print(arr)   # [40, 20, 30, 10]
```

```python
# 첫 번째 ↔ 마지막
arr[0], arr[-1] = arr[-1], arr[0]

# 인접한 두 원소 교환
arr[i], arr[i+1] = arr[i+1], arr[i]
```

---

---

# ③ 실전 패턴

## 범위 정렬 보장 — a > b 이면 교환

```python
def adder(a, b):
    if a > b:
        a, b = b, a   # 항상 a <= b 보장
    return sum(range(a, b + 1))

adder(5, 1)   # a=1, b=5 로 교환 후 1+2+3+4+5 = 15
adder(1, 5)   # 그대로 1+2+3+4+5 = 15
```

```
a, b 순서가 뒤집혀서 들어와도 안전하게 처리
입력 순서 상관없이 항상 작은 수 → 큰 수 방향으로 계산
```

## 피보나치 수열

```python
a, b = 0, 1
for _ in range(10):
    print(a, end=" ")
    a, b = b, a + b
# 0 1 1 2 3 5 8 13 21 34

# a, b = b, a+b 분해:
# 우변 먼저 평가: (b, a+b) = (1, 0+1) = (1, 1)
# a = 1 / b = 1
# 다음 단계: a=1, b=1 → (1, 2) → a=1, b=2 ...
```

## 정렬 알고리즘 — 버블 정렬

```python
arr = [5, 3, 1, 4, 2]

for i in range(len(arr)):
    for j in range(len(arr) - i - 1):
        if arr[j] > arr[j+1]:
            arr[j], arr[j+1] = arr[j+1], arr[j]   # 핵심

print(arr)   # [1, 2, 3, 4, 5]
```

## 문자열 뒤집기 (양 끝에서 좁혀오기)

```python
s = list("hello")
left, right = 0, len(s) - 1

while left < right:
    s[left], s[right] = s[right], s[left]
    left += 1
    right -= 1

"".join(s)   # 'olleh'
```

## 여러 변수 동시 교환

```python
# 2개
a, b = b, a

# 3개 순환
a, b, c = b, c, a

# 예시
a, b, c = 1, 2, 3
a, b, c = b, c, a
print(a, b, c)   # 2 3 1
```

## 함수 반환값으로 순서 정렬

```python
# 두 값 중 항상 (작은 값, 큰 값) 반환
def sorted_pair(a, b):
    if a > b:
        a, b = b, a
    return a, b

sorted_pair(9, 3)   # (3, 9)
sorted_pair(1, 5)   # (1, 5)

# 언패킹으로 받기
lo, hi = sorted_pair(9, 3)
```

---

---

# 자주 하는 실수

```python
# ❌ 이렇게 하면 a 가 이미 바뀐 값으로 b 에 들어감
a = b        # a = 20
b = a        # b = 20  ← 원래 a 값(10) 이 아닌 바뀐 값!

# ✅ 우변을 먼저 튜플로 평가
a, b = b, a  # 동시에 교환
```

```python
# ❌ 루프에서 착각
for i in range(len(arr) // 2):
    arr[i] = arr[-(i+1)]       # 덮어씀!
    arr[-(i+1)] = arr[i]       # 이미 바뀐 값 저장

# ✅ 한 줄 swap
for i in range(len(arr) // 2):
    arr[i], arr[-(i+1)] = arr[-(i+1)], arr[i]
```