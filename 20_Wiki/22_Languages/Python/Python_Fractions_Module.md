---
aliases:
  - Python
  - Fraction
  - 분수
  - 유리수
  - 부동소수점 오차방지
tags:
  - Python
related:
  - "[[00_Python_HomePage]]"
  - "[[Python_Math_Module]]"
---
# Python_Fractions_Module — fractions 모듈

## 한 줄 요약

```
부동소수점 오차 없이 분수(유리수)를 정확하게 계산하는 내장 모듈
```

---

---

# ① 왜 필요한가 — float 의 한계

```python
# float 의 부동소수점 오차
0.1 + 0.2 == 0.3   # False !!
print(0.1 + 0.2)   # 0.30000000000000004

# 이진수 표현 한계:
# 0.1 은 컴퓨터에서 정확히 표현 불가
# → 미세한 오차 누적
```

```
해결:
  Fraction  → 분수 형태로 정확하게 저장 / 계산
  Decimal   → 금융 / 화폐 계산 (소수점 자릿수 정밀 제어)
  Fraction  → 비율 / 수학적 정확도가 필요할 때
```

---

---

# ② 분수 생성

```python
from fractions import Fraction

# 정수로 생성 (분자, 분모)
Fraction(3, 4)      # 3/4
Fraction(6, 9)      # 2/3  ← 자동 약분

# 문자열로 생성 ← 권장 (오차 없음)
Fraction('0.1')     # 1/10  ← 정확
Fraction('3/4')     # 3/4
Fraction('-5/2')    # -5/2
Fraction('1.5')     # 3/2

# float 로 생성 ← 비권장 (오차 포함)
Fraction(0.1)       # 3602879701896397/36028797018963970  ← 이상한 값!
```

```
⚠️ float 를 직접 넣으면 안 되는 이유:
  0.1 은 이미 메모리에 오차가 있는 이진수
  Fraction 이 그 오차까지 그대로 분수로 변환
  → 반드시 문자열 Fraction('0.1') 로 넣어야 정확
```

---

---

# ③ 사칙연산 — 자동 통분 + 자동 약분

```python
from fractions import Fraction

a = Fraction(1, 6)   # 1/6
b = Fraction(1, 3)   # 1/3

print(a + b)    # 1/2  (1/6 + 2/6 = 3/6 → 1/2)
print(a - b)    # -1/6
print(a * b)    # 1/18
print(a / b)    # 1/2

# 정수와 연산
print(a * 2)    # 1/3
print(a + 1)    # 7/6
```

```
내부 동작:
  통분 → 계산 → 자동 약분 (기약분수)
  math.gcd 와 같은 원리
```

```python
# Fraction 내부 약분 원리 (math.gcd 활용)
import math

n, d = 6, 9
g = math.gcd(n, d)   # 3
print(n // g, d // g)  # 2 3  → 2/3
```

---

---

# ④ 비교 연산

```python
from fractions import Fraction

Fraction(1, 3) == Fraction(2, 6)   # True  (자동 약분 후 비교)
Fraction(1, 2) > Fraction(1, 3)    # True
Fraction(1, 4) < 0.3               # True  (float 와도 비교 가능)
```

---

---

# ⑤ 속성 확인

```python
f = Fraction(3, 4)

f.numerator    # 3  (분자)
f.denominator  # 4  (분모)

# 실수로 변환
float(f)       # 0.75
int(f)         # 0  (소수부 버림)
```

---

---

# ⑥ limit_denominator() — float 를 분수로 근사

```
float 나 무리수를 분수로 변환하면 분모가 매우 커짐
limit_denominator(n) = 분모를 최대 n 이하로 제한한 근삿값
```

```python
import math
from fractions import Fraction

# 원주율 π
f = Fraction(math.pi)
print(f)
# 884279719003555/281474976710656  ← 분모가 엄청 큼

# 분모 최대 100으로 제한
print(f.limit_denominator(100))
# 311/99  ← 3.141414...  (π 에 가장 가까운 근삿값)

print(f.limit_denominator(10))
# 22/7   ← 3.142857...  (흔히 쓰는 π 근사)
```

---

---

# ⑦ Fraction vs float vs Decimal

|항목|float|Fraction|Decimal|
|---|---|---|---|
|오차|있음|없음|없음|
|형태|소수|분수|고정 소수점|
|속도|빠름|느림|중간|
|적합한 상황|일반 계산|비율 / 수학|금융 / 화폐|

```python
from decimal import Decimal

# 금융 계산 → Decimal
Decimal('0.1') + Decimal('0.2')   # 0.3  ← 정확

# 비율 계산 → Fraction
Fraction(1, 10) + Fraction(2, 10)  # 3/10  ← 정확
```

---

---

# 자주 하는 실수

```python
# ❌ float 직접 넣기
Fraction(0.1)   # 3602879701896397/36028797018963970
                # 0.1 의 부동소수점 오차까지 그대로 변환

# ✅ 문자열로 넣기
Fraction('0.1')   # 1/10  ← 정확

# ✅ 정수로 넣기
Fraction(1, 10)   # 1/10  ← 정확
```