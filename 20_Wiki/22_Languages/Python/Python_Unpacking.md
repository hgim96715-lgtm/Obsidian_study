---
aliases:
  - Unpacking
  - 언패킹
  - 튜플 풀기
  - 리스트 풀기
  - 구조 분해 할당
  - Asterisk
tags:
  - Python
related:
  - "[[Python_Lists_Tuples]]"
  - "[[00_Python_HomePage]]"
  - "[[Python_Builtin_Functions]]"
---
# Python_Unpacking — 언패킹 (값 풀어서 담기)

## 한 줄 요약

```
묶여있는 데이터(리스트/튜플)를
개별 변수 여러 개로 한 번에 풀어서 담는 문법

a, b = [10, 20]   →   a=10, b=20
```

---

---

# ① 기본 언패킹 ⭐️

```python
# 리스트 풀기
data = [10, 20]
a, b = data
print(a)   # 10
print(b)   # 20

# 튜플 풀기 (괄호 생략 가능)
name, age = ("Kim", 25)
print(name)   # "Kim"
print(age)    # 25

# 직접 쓰기
x, y, z = [1, 2, 3]
```

```
⚠️ 개수 일치 필수
  a, b = [1, 2, 3]       # ValueError: too many values to unpack
  a, b, c, d = [1, 2, 3] # ValueError: not enough values to unpack
```

---

---

# ② for 문에서 바로 언패킹 ⭐️

```python
scores = [["Kim", 90], ["Lee", 80], ["Park", 95]]

# ❌ 번거로운 방식
for student in scores:
    name = student[0]
    score = student[1]

# ✅ for 문에서 바로 언패킹
for name, score in scores:
    print(f"{name} 학생: {score}점")
# Kim 학생: 90점
# Lee 학생: 80점
```

```
내부 리스트가 2개씩이면 변수도 2개
for name, score in scores    ← 안쪽 [이름, 점수] = 2개
for a, b in sizes            ← 안쪽 [가로, 세로] = 2개
for char, num in pairs       ← 안쪽 (문자, 숫자) = 2개
```

## 헷갈리는 포인트

```python
arr = [[a, b], [c, d], [e, f]]

# for q in arr → q 에 [a, b] 리스트 자체가 들어옴
for q in arr:
    one, two, three = q   # ❌ q 는 2개인데 변수 3개 → ValueError

# ✅ 바로 언패킹
for x, y in arr:          # x = a, y = b 로 각 쌍이 언패킹됨
    print(x, y)
```

---

---

# ③ 함수 반환값 언패킹

```python
def get_min_max(lst):
    return min(lst), max(lst)   # 튜플로 반환

# 언패킹으로 한 번에 받기
lo, hi = get_min_max([3, 1, 4, 1, 5])
print(lo)   # 1
print(hi)   # 5

# divmod 도 동일 패턴
quotient, remainder = divmod(17, 5)
# quotient=3, remainder=2
```

---

---

# ④ *rest — 나머지 몽땅 담기 ⭐️

```python
numbers = [1, 2, 3, 4, 5]

# 앞에서 하나, 나머지 전부
a, *rest = numbers
print(a)      # 1
print(rest)   # [2, 3, 4, 5]

# 처음과 끝만, 중간은 버리기
first, *middle, last = numbers
print(first)    # 1
print(last)     # 5
print(middle)   # [2, 3, 4]

# 앞에서 두 개, 나머지
a, b, *rest = [10, 20, 30, 40, 50]
print(a, b)     # 10 20
print(rest)     # [30, 40, 50]
```

```
규칙:
  * 는 한 식에 1개만 사용 가능
  * 앞에 오면 맨 앞부터 / 뒤에 오면 나머지 전부
  * 결과는 항상 리스트
```

---

---

# ⑤ _ — 필요 없는 값 버리기

```python
user_info = ["admin", "1234", "Seoul", "010-1234-5678"]

# ID와 지역만 필요
user_id, _, location, _ = user_info
print(user_id)    # "admin"
print(location)   # "Seoul"
# _ 에는 값이 들어있지만 사용하지 않음

# enumerate 에서 인덱스 무시
for _, value in enumerate(["a", "b", "c"]):
    print(value)

# 여러 반환값 중 일부만
_, max_val = divmod(100, 7)   # 몫은 버리고 나머지만
```

---

---

# ⑥ 두 값 정렬 패턴 — 회전 가능 문제 ⭐️

```
"회전 가능 / 방향 선택 가능" 조건이 보이면
→ 두 값을 (큰 것, 작은 것) 으로 통일

패턴:
  a, b = max(a, b), min(a, b)
  또는
  if a < b: a, b = b, a
```

```python
# 명함 지갑 문제 — 모든 명함을 수납하는 가장 작은 지갑

def solution(sizes):
    w, h = 0, 0
    for a, b in sizes:
        if a < b:
            a, b = b, a    # a = 긴 쪽, b = 짧은 쪽으로 통일
        w = max(w, a)
        h = max(h, b)
    return w * h

# 또는 max/min 으로 한 줄
def solution(sizes):
    w, h = 0, 0
    for a, b in sizes:
        a, b = max(a, b), min(a, b)
        w = max(w, a)
        h = max(h, b)
    return w * h
```

```
if a < b: a, b = b, a 의 의미:
  "a 가 더 작으면, 둘을 바꿔서 a 가 항상 더 크도록 만든다"

  (30, 70) → a=30, b=70 → a < b → 바꿔서 a=70, b=30
  (90, 50) → a=90, b=50 → a > b → 그대로

  결과: 항상 a = 긴 변 / b = 짧은 변
```

## 적용 상황

```
이 패턴이 필요한 문제 유형:
  "명함/카드를 회전 가능"
  "박스 넣기 (방향 자유)"
  "직사각형이 다른 직사각형에 들어가는가"
  → 언제나 (큰 쪽, 작은 쪽) 으로 정렬해서 비교
```

---

---

# ⑦ 중첩 언패킹

```python
# 2중 중첩
data = [[1, 2], [3, 4]]
(a, b), (c, d) = data
print(a, b, c, d)   # 1 2 3 4

# 실전: 좌표 쌍
point1, point2 = (0, 0), (3, 4)
x1, y1 = point1
x2, y2 = point2
```

---

---

# 자주 하는 실수

|실수|원인|해결|
|---|---|---|
|`a, b = [1, 2, 3]` 에러|개수 불일치|`a, *b = [1, 2, 3]` 또는 `a, b, c = ...`|
|`for x, y, z in [[a,b], ...]` 에러|내부 리스트 2개인데 변수 3개|내부 리스트 크기와 변수 수 맞추기|
|`*` 두 번 쓰기|문법 에러|`*` 는 한 식에 1개만|
|`a, b = max(a,b), min(a,b)` 순서|양쪽 동시 평가 걱정|파이썬은 우변 먼저 전부 계산 후 좌변 할당 → 안전|