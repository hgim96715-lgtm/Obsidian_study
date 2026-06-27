---
aliases:
  - Python Math
  - math.prod
  - 수학 함수
tags:
  - Python
related:
  - "[[Python_Random_Seed|랜덤 수 생성 (random)]]"
  - "[[Python_Builtin_Functions|기본 내장 함수 (sum, max, min)]]"
  - "[[00_Python_HomePage]]"
  - "[[Python_Fractions_Module]]"
---
# Python_Math_Module

## 개념 한 줄 요약

> *_"기본 연산자(+, -, _, /)만으로 부족한 정밀한 계산이나 누적 연산을 처리하는 표준 라이브러리."__

```python
import math  # 먼저 선언 필수
```

---

---

# ① 누적 곱 — math.prod ✖️

> 리스트나 튜플 안에 있는 모든 숫자를 곱한다. (Python 3.8+)

```python
math.prod(리스트)
```

```python
import math

rates = [0.9, 0.8, 0.95]
total_efficiency = math.prod(rates)
# 결과: 0.684  (0.9 * 0.8 * 0.95)
```

```txt
활용:
- 장비 가동률 누적 계산
- 확률 누적 계산
```

---

---

# ② 올림 / 내림 / 버림 — ceil, floor, trunc

> 소수점 데이터를 정수로 변환할 때 사용. 데이터 전처리 필수.

```python
math.ceil(x)   # 올림 — x 보다 크거나 같은 최소 정수
math.floor(x)  # 내림 — x 보다 작거나 같은 최대 정수
math.trunc(x)  # 버림 — 소수점 제거, 0 쪽으로 붙임
```

```python
x = 3.7

math.ceil(x)   # → 4   (무조건 올림)
math.floor(x)  # → 3   (무조건 내림)
math.trunc(x)  # → 3   (소수점 버림)

x = -3.7
math.ceil(x)   # → -3  (0 방향으로 올림)
math.floor(x)  # → -4  (0 반대 방향으로 내림)
math.trunc(x)  # → -3  (0 쪽으로 버림)
```

```txt
ceil  vs  trunc:
양수일 땐 동일하게 보임
음수일 때 차이가 생김 → -3.7 기준
  ceil  = -3  (0 방향)
  trunc = -3  (동일)
  floor = -4  (0 반대 방향)
```

---

---

# ③ 최대공약수 / 최소공배수 — gcd, lcm

> 배치 작업 주기 맞추기, 데이터를 균등하게 나눌 때 활용.

```python
math.gcd(a, b)  # 최대공약수 (Greatest Common Divisor)
math.lcm(a, b)  # 최소공배수 (Least Common Multiple, Python 3.9+)
```

```python
math.gcd(12, 8)  # → 4
math.lcm(4, 6)   # → 12
```

## 버전 주의 ⭐️

```txt
math.gcd  → Python 3.5+ (거의 어디서나 사용 가능)
math.lcm  → Python 3.9+ (구버전 환경에서 없을 수 있음)

코딩 테스트 플랫폼에 따라 Python 3.8 이하일 수 있음
→ math.lcm 못 쓰는 상황에서 직접 구현 필요
```

## math.lcm 없을 때 — 직접 구현 ⭐️

```python
import math

def solution(n, m):
    # 최대공약수
    ma_gc = math.gcd(n, m)

    # 최소공배수 = (n * m) // 최대공약수
    ma_lc = (n * m) // ma_gc

    return [ma_gc, ma_lc]
```

```txt
왜 (n * m) // gcd 인가:
  두 수의 최소공배수 공식:
  LCM(a, b) = (a × b) / GCD(a, b)

  예시: n=4, m=6
    gcd(4, 6) = 2
    lcm = (4 × 6) // 2 = 24 // 2 = 12 ✅
```

## math 도 못 쓸 때 — 유클리드 호제법으로 직접 구현

```python
# 최대공약수 — 유클리드 호제법
def gcd(a, b):
    while b:
        a, b = b, a % b
    return a

# 최소공배수
def lcm(a, b):
    return (a * b) // gcd(a, b)

# 사용
print(gcd(12, 8))   # 4
print(lcm(4, 6))    # 12
```

```txt
유클리드 호제법 원리:
  gcd(12, 8):
    a=12, b=8  → a=8,  b=12%8=4
    a=8,  b=4  → a=4,  b=8%4=0
    b=0  → return a = 4  ✅

  핵심: gcd(a, b) = gcd(b, a % b)
  b 가 0 이 될 때까지 반복
  b 가 0 이면 a 가 최대공약수
```

## 상황별 선택

```python
# Python 3.9+ 환경 → 둘 다 사용 가능
import math
math.gcd(n, m)
math.lcm(n, m)

# Python 3.5~3.8 환경 → lcm 직접 계산
import math
gcd = math.gcd(n, m)
lcm = (n * m) // gcd

# math import 불가 환경 → 유클리드 호제법 직접 구현
def gcd(a, b):
    while b:
        a, b = b, a % b
    return a
lcm = (n * m) // gcd(n, m)
```

```python
import math

def add_fractions(a_num, a_den, b_num, b_den):
    # ① 공통 분모 = 두 분모의 최소공배수
    common_den = math.lcm(a_den, b_den)

    # ② 분자를 공통 분모 기준으로 변환 후 합산
    total_num = (a_num * (common_den // a_den)) + (b_num * (common_den // b_den))

    # ③ 결과 약분: 분자·분모의 최대공약수로 나눔
    g = math.gcd(total_num, common_den)
    return total_num // g, common_den // g

result = add_fractions(1, 3, 1, 4)  # 1/3 + 1/4
print(result)  # → (7, 12)  즉 7/12
```

> `fractions.Fraction` 을 쓰면 자동 처리되지만, `gcd` / `lcm` 으로 직접 구현하면 내부 동작 원리와 성능을 직접 제어할 수 있다. → [[Python_Fractions_Module]] 참고

---

---

# ④ 팩토리얼 — math.factorial

> n! = 1 × 2 × 3 × ... × n 순열/조합 계산의 기반.

```python
math.factorial(n)
```

```python
math.factorial(5)  # → 120  (5 * 4 * 3 * 2 * 1)
math.factorial(0)  # → 1    (0! = 1 by definition)
math.factorial(1)  # → 1
```

```txt
활용:
- 순열 계산: nPr = n! / (n-r)!
- 조합 계산: nCr = n! / (r! * (n-r)!)
- 확률 계산에서 경우의 수
```

---

---

# ⑤ 조합 — math.comb

> nCr = n개 중 r개를 **순서 없이** 뽑는 경우의 수. `factorial` 로 직접 계산하는 것보다 빠르고 오버플로우 안전.

```python
math.comb(n, r)
```

```python
math.comb(5, 2)   # → 10   (5개 중 2개 선택)
math.comb(10, 3)  # → 120
math.comb(5, 0)   # → 1
math.comb(5, 5)   # → 1
```

```python
# 로또 당첨 확률 분모 (45개 중 6개 선택)
math.comb(45, 6)  # → 8,145,060
```

```txt
math.comb vs math.factorial 로 직접 계산:
math.comb(10, 3)                              # 간단 ✅
math.factorial(10) // (math.factorial(3) * math.factorial(7))  # 장황 ❌
```

---

---

# ⑥ 순열 — math.perm

> nPr = n개 중 r개를 **순서 있게** 뽑는 경우의 수. (Python 3.8+)

```python
math.perm(n, r)
```

```python
math.perm(5, 2)   # → 20   (5개 중 2개를 순서 있게)
math.perm(5, 5)   # → 120  (= 5!)
math.perm(5)      # → 120  (r 생략 시 n! 과 동일)
```

```txt
조합 vs 순열:
math.comb(5, 2) = 10   → {A,B} 와 {B,A} 를 같은 것으로 봄
math.perm(5, 2) = 20   → {A,B} 와 {B,A} 를 다른 것으로 봄
```

---

---

# ⑦ 제곱근 — math.sqrt

```python
math.sqrt(x)
```

```python
math.sqrt(16)   # → 4.0
math.sqrt(2)    # → 1.4142135623730951
```

```txt
주의: 결과가 float 로 반환됨
정수로 필요하면 int(math.sqrt(x)) 또는 x ** 0.5 사용
```

## 완전 제곱수 판별 ⭐️

```txt
완전 제곱수 = 어떤 정수의 제곱인 수
  1(1²), 4(2²), 9(3²), 16(4²), 25(5²) ...
  제곱근이 정수로 딱 떨어지면 완전 제곱수
```

## 판별 방법 3가지

```python
import math

i = 16

# 방법 1 — math.sqrt % 1
math.sqrt(i) % 1 == 0       # True  (4.0 % 1 == 0.0)

# 방법 2 — i**0.5 소수점 비교
int(i**0.5) == i**0.5       # True  (4 == 4.0)

# 방법 3 — 제곱으로 역산
int(i**0.5) ** 2 == i       # True  (4**2 == 16)
```

```txt
i = 15 일 때 (완전 제곱수 아님):
  math.sqrt(15) % 1         → 0.872... % 1 ≠ 0  → False
  int(15**0.5) == 15**0.5   → 3 == 3.872...     → False
  int(15**0.5) ** 2 == 15   → 3**2 = 9 ≠ 15    → False
```

## 실전 — 완전 제곱수면 빼고 아니면 더하기

```python
import math

def solution(left, right):
    answer = 0
    for k in range(left, right + 1):
        if math.sqrt(k) % 1 == 0:   # 완전 제곱수
            answer -= k
        else:
            answer += k
    return answer

# 또는 i**0.5 버전
def solution(left, right):
    answer = 0
    for i in range(left, right + 1):
        if int(i**0.5) == i**0.5:   # 완전 제곱수
            answer -= i
        else:
            answer += i
    return answer
```

## 세 방법 비교

|방법|코드|특징|
|---|---|---|
|`math.sqrt(i) % 1 == 0`|import 필요 / 읽기 좋음|가장 직관적|
|`int(i**0.5) == i**0.5`|import 불필요|간결|
|`int(i**0.5) ** 2 == i`|import 불필요|역산으로 검증 / 가장 안전|

## 방법 3이 왜 가장 안전한가 ⭐️

```txt
부동소수점 오차 문제:
  파이썬에서 큰 수의 제곱근을 float 로 계산하면
  미세한 오차가 생길 수 있음

  예시:
  i = 999999999999
  i**0.5           → 999999.9999995 (오차 발생)
  int(i**0.5)      → 999999
  int(i**0.5) == i**0.5  → False  ← 완전 제곱수가 아닌데 아닌 걸 잡음

  방법 3 역산:
  int(i**0.5) ** 2 == i
  → 999999 ** 2 == 999999999999
  → 999998000001 == 999999999999 → False  ← 정확하게 판별
```

```python
# 방법 1 — math.sqrt % 1 (직관적)
math.sqrt(i) % 1 == 0

# 방법 2 — i**0.5 소수점 비교 (간결)
int(i**0.5) == i**0.5

# 방법 3 — 역산 (가장 안전) ⭐️
int(i**0.5) ** 2 == i
```

## 실전 — 방법 3 버전

```python
def solution(left, right):
    answer = 0
    for i in range(left, right + 1):
        if int(i**0.5) ** 2 == i:   # 완전 제곱수 (역산 / 가장 안전)
            answer -= i
        else:
            answer += i
    return answer
```

```txt
코테 선택 기준:
  빠르게 짜기   → int(i**0.5) == i**0.5  (방법 2)
  안전하게 짜기 → int(i**0.5) ** 2 == i  (방법 3)
  가독성 우선   → math.sqrt(i) % 1 == 0  (방법 1)
```

---

---

# ⑧ 부동소수점 비교 — math.isclose

> 실수 연산에서 발생하는 미세한 오차를 무시하고 두 값이 같은지 비교.

```python
math.isclose(a, b, rel_tol=1e-9)
```

```python
0.1 + 0.2 == 0.3        # → False  ← 부동소수점 오차
math.isclose(0.1 + 0.2, 0.3)  # → True  ✅
```

---
---
# ⑨ 약수 구하기 — 제곱근 최적화 ⭐️

## 왜 제곱근까지만 반복하나

```python
# 약수는 항상 쌍으로 존재
# 12의 약수:  1×12  2×6  3×4
#             ↑쌍     ↑쌍   ↑쌍
# 제곱근(√12 ≈ 3.46) 이하까지만 찾으면 → 나머지 쌍은 자동으로 구해짐

# 비효율 (n 번 반복)
for i in range(1, n+1):
    if n % i == 0:
        count += 1

# 효율 (√n 번 반복) ⭐️
for i in range(1, int(n**0.5) + 1):
    if n % i == 0:
        count += 1   # i 약수
        count += 1   # n//i 약수 (쌍)
# 단, i*i == n 이면 (완전제곱수) 쌍이 하나 → count 1번만
```

## 약수 개수 구하기 패턴 ⭐️


```python
def count_divisors(n):
    count = 0
    for j in range(1, int(n**0.5) + 1):
        if n % j == 0:          # j 가 약수인지 확인
            if j * j == n:      # 완전제곱수 (j == n//j)
                count += 1      # 약수 1개만 (중복 방지)
            else:
                count += 2      # j 와 n//j 두 개
    return count

# 예시
# n=12:  j=1(1,12) j=2(2,6) j=3(3,4)  → count=6
# n=9:   j=1(1,9) j=3(3*3==9)          → count=3
```

```txt
j * j == n 이 왜 필요한가:
  n=9, j=3: 3*3=9 (완전제곱수)
  → 3의 쌍은 3 자신 (9//3=3)
  → count+2 하면 3이 두 번 카운트됨
  → count+1 만 해야 함

  n=12, j=2: 2*2=4 ≠ 12
  → 2의 쌍은 6 (12//2=6)
  → count+2 (2와 6 각각)
```

## 약수 목록 구하기

```python
def get_divisors(n):
    divisors = []
    for i in range(1, int(n**0.5) + 1):
        if n % i == 0:
            divisors.append(i)
            if i != n // i:       # 중복 방지
                divisors.append(n // i)
    return sorted(divisors)

get_divisors(12)  # [1, 2, 3, 4, 6, 12]
get_divisors(9)   # [1, 3, 9]
```

## set 활용 패턴 (간결한 버전)

```python
def count_divisors_set(n):
    a = []
    for i in range(1, int(n**0.5) + 1):
        if n % i == 0:
            a.append(i)
            a.append(n // i)    # 쌍도 추가
    return len(set(a))          # set 으로 중복 제거
# → j*j==n 체크 없이 set 으로 해결
```

## 실전 — 약수 개수 + 조건 처리

```python
def solution(number, limit, power):
    answer = 0
    for n in range(1, number + 1):
        count = 0
        for j in range(1, int(n**0.5) + 1):
            if n % j == 0:
                if j * j == n:
                    count += 1    # 완전제곱수
                else:
                    count += 2    # 약수 쌍
        # count > limit 이면 power 로 대체
        answer += power if count > limit else count
    return answer
```

---

---

# 전체 치트시트

|함수|설명|비고|
|---|---|---|
|`prod(list)`|리스트 내 모든 수의 곱|Python 3.8+|
|`ceil(x)`|올림||
|`floor(x)`|내림||
|`trunc(x)`|소수점 버림 (0 방향)||
|`gcd(a, b)`|최대공약수||
|`lcm(a, b)`|최소공배수|Python 3.9+|
|`factorial(n)`|n! 팩토리얼||
|`comb(n, r)`|조합 nCr (순서 무관)|Python 3.8+|
|`perm(n, r)`|순열 nPr (순서 있음)|Python 3.8+|
|`sqrt(x)`|제곱근|float 반환|
|`isclose(a, b)`|부동소수점 근사 비교||