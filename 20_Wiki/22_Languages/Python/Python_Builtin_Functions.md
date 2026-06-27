---
aliases:
  - Built-in Functions
  - max
  - min
  - sum
  - len
  - abs
  - 내장함수
  - ord
  - chr
  - pow
  - bit_length()
tags:
  - Python
related:
  - "[[Python_Sorting_Logic]]"
  - "[[Python_Variables_Types]]"
  - "[[00_Python_HomePage]]"
  - "[[Python_Math_Module]]"
  - "[[Python_Type_Checking]]"
  - "[[Python_Unpacking]]"
---
# Python_Builtin_Functions — 내장함수

## 한 줄 요약

```
import 없이 바로 쓸 수 있는 파이썬이 미리 만들어 둔 도구들
for / if 로 길게 쓸 것을 한 줄로 줄여주는 Pythonic 코딩의 기본
```

---

---

# ① 집계 — max / min / sum / len / abs

```python
numbers = [1, 5, 3, 9]

max(numbers)     # 9   ← 리스트 통째로
min(10, 20)      # 10  ← 인자 여러 개
sum(numbers)     # 18  ← for 문 쓰지 말 것
len("Hello")     # 5
abs(-100)        # 100
```

```
내부적으로 C 언어로 구현된 최적화 로직
직접 for 문 쓰는 것보다 빠름
```

## sum() + range() 조합 패턴 ⭐️

```python
# a 부터 b 까지 합계

# ❌ 리스트 컴프리헨션 불필요
sum([n for n in range(a, b+1)])
# [a, a+1, ..., b] 리스트를 메모리에 생성 후 sum → 낭비

# ✅ range 는 이미 iterable → 바로 전달
sum(range(a, b+1))
# 리스트 생성 없이 바로 합산
```

```
sum() 은 리스트뿐 아니라 iterable 이면 뭐든 받음
range() 는 iterable → [n for n in range()] 불필요

[n for n in range(a, b+1)]  → 리스트 생성 (메모리 낭비)
range(a, b+1)               → 바로 전달 (효율적)
```

```python
# a > b 케이스까지 안전하게
sum(range(min(a, b), max(a, b) + 1))
#          ↑ 항상 작은 수 시작 / 큰 수 끝

# 예시
a, b = 5, 1
sum(range(min(5,1), max(5,1)+1))  # sum(range(1, 6)) = 15
sum(range(1, 6))                  # 1+2+3+4+5 = 15

# 비교: a, b 정렬 후 사용 (Python_Variable_Swapping 패턴)
if a > b: a, b = b, a
sum(range(a, b+1))   # 동일 결과
```

## max / min 활용 — 두 값 정렬 패턴 ⭐️

```
"회전 가능 / 방향 선택 가능한 경우"
→ 항상 큰 쪽을 가로, 작은 쪽을 세로로 통일

상황 감지 키워드:
  "회전 가능", "방향 선택 가능", "가로/세로 구분 없음"
  "박스 안에 넣기", "직사각형 포함 여부"
  → 즉시 "정렬 패턴" 떠올리기
```

```python
# 방법 1 — if 로 바꾸기
for a, b in sizes:
    if a < b:
        a, b = b, a      # a 가 더 작으면 바꿔서 a 가 항상 큰 쪽
    w = max(w, a)
    h = max(h, b)

# 방법 2 — max/min 으로 한 줄에
for a, b in sizes:
    a, b = max(a, b), min(a, b)   # a = 항상 큰 값 / b = 항상 작은 값
    w = max(w, a)
    h = max(h, b)
```

```
if a < b: a, b = b, a   vs   a, b = max(a,b), min(a,b):
  결과 동일
  if 방식   → 조건 분기 / 읽기 쉬움
  max/min  → 한 줄 / 간결 (Pythonic)
```

## 명함 지갑 문제 — 전체 풀이 ⭐️

```python
# 문제: 모든 명함을 수납하는 가장 작은 지갑 크기
# 명함은 회전 가능 → 긴 쪽을 가로로 통일

def solution(sizes):
    w, h = 0, 0
    for a, b in sizes:
        if a < b:
            a, b = b, a    # a = 긴 쪽 / b = 짧은 쪽으로 통일
        w = max(w, a)      # 모든 명함 중 가장 긴 가로
        h = max(h, b)      # 모든 명함 중 가장 긴 세로
    return w * h

# Pythonic 한 줄 버전
def solution(sizes):
    return max(max(x) for x in sizes) * max(min(x) for x in sizes)
```

```
핵심 아이디어:
  각 명함을 (긴 변, 짧은 변) 으로 통일
  → 모든 명함의 긴 변 중 최댓값 = 지갑 가로
  → 모든 명함의 짧은 변 중 최댓값 = 지갑 세로
```

## max / min 주의 — 문자열 vs 숫자

```python
# 숫자는 크기 비교
max(10, 9)      # 10  ✅

# 문자열은 사전순 비교 (첫 글자 아스키코드 기준)
max("10", "9")  # "9"  ← '9'(57) > '1'(49) 이라서!
                #  ⚠️ 숫자 크기 비교 아님

# 해결: int() 로 변환 후 비교
max(int("10"), int("9"))  # 10 ✅
```

```python
# 응용: 두 수를 합쳐 더 큰 수 만들기 (코테 단골)
a, b = 3, 30
max(f"{a}{b}", f"{b}{a}")   # "330"  ← "330" vs "303" 사전순 비교
```

---

---

# ② 문자 ↔ 숫자 변환 — ord / chr

```
컴퓨터는 문자를 아스키(ASCII) 코드 숫자로 기억
ord  문자 → 숫자 (Ordinal)
chr  숫자 → 문자 (Character)
```

```python
ord('a')   # 97
ord('A')   # 65  ← 대문자가 소문자보다 숫자 작음!
chr(97)    # 'a'
chr(65)    # 'A'

# 알파벳 A~C 출력
for i in range(ord('A'), ord('D')):
    print(chr(i), end=" ")
# A B C
```

```
주요 아스키 코드:
  'A' = 65     'Z' = 90
  'a' = 97     'z' = 122
  '0' = 48     '9' = 57

한글도 가능 (유니코드 지원):
  ord('가') = 44032

⚠️ ord() 는 딱 한 글자만
  ord("ABC") → TypeError
  ord("A")   → 65 ✅
```

---

---

# ③ 몫과 나머지 — divmod

```
divmod(a, b) → (몫, 나머지) 튜플 반환
// 과 % 를 따로 두 번 하는 것보다 한 번에 처리 → 성능 유리
```

```python
total_records = 10050
batch_size    = 1000

batches, remainder = divmod(total_records, batch_size)

print(f"전체 배치 수: {batches}, 남은 데이터: {remainder}")
# 전체 배치 수: 10, 남은 데이터: 50
```

```python
# 시간 변환 (초 → 시/분/초)
seconds = 3725
hours,   rest = divmod(seconds, 3600)
minutes, sec  = divmod(rest, 60)
print(f"{hours}시간 {minutes}분 {sec}초")  # 1시간 2분 5초
```

---

---

# ④ 거듭제곱 — pow

```python
pow(2, 10)          # 1024  ← 2의 10승 (2**10 과 동일)
pow(2, 10, 1000)    # 24    ← (2**10) % 1000
```

## 3번째 인자 mod — 큰 수의 나머지 계산 ⭐️

```
pow(base, exp, mod) = (base ** exp) % mod

2**10 → 1024 → % 1000 → 24

왜 pow(a, b, mod) 가 (a**b) % mod 보다 좋은가:
  a**b 를 먼저 계산하면 천문학적으로 큰 수가 메모리를 잡아먹음
  pow(a, b, mod) 는 내부적으로 빠른 거듭제곱(Fast Exponentiation) 사용
  중간 결과를 계속 mod 로 줄이며 계산 → 빠르고 메모리 효율적
```

```python
# 알고리즘 단골 패턴 — 10억의 10억승 나머지도 바로 계산
pow(10**9, 10**9, 10**9 + 7)  # 빠르고 정확하게 나머지 반환
```

---

---

# ⑤ 비트 길이 — bit_length()

```
정수를 2진수로 표현할 때 필요한 비트(자릿수) 수 반환

bin(4)  = '0b100'  → 유효 비트 3개 → (4).bit_length() = 3
bin(8)  = '0b1000' → 유효 비트 4개 → (8).bit_length() = 4
```

```python
(1).bit_length()   # 1
(4).bit_length()   # 3
(7).bit_length()   # 3   (111)
(8).bit_length()   # 4   (1000)
(0).bit_length()   # 0   ← 0 은 유효 비트 없음
```

## << 연산자 (Left Shift) — 2의 거듭제곱

```python
1 << 0   # 1   (2⁰)
1 << 1   # 2   (2¹)
1 << 2   # 4   (2²)
1 << 3   # 8   (2³)

# 1 << n  =  2ⁿ  =  pow(2, n) 과 동일
# 비트 연산이라 속도가 더 빠름
```

## 실전 패턴 — 다음 2의 거듭제곱으로 올림 (패딩)

```python
def solution(arr):
    length = len(arr)
    target = 1 << (length - 1).bit_length()
    return arr + [0] * (target - length)

# arr = [1, 2, 3, 4, 5]  length = 5
# length - 1 = 4
# (4).bit_length() = 3   (4는 이진수 '100', 3비트)
# 1 << 3 = 8             (2³ = 8)
# arr + [0] * (8 - 5)  → [1, 2, 3, 4, 5, 0, 0, 0]

print(solution([1, 2, 3, 4, 5]))  # [1, 2, 3, 4, 5, 0, 0, 0]
print(solution([1, 2, 3, 4]))     # [1, 2, 3, 4]  ← 이미 2의 거듭제곱이면 그대로
```

---

---

# ⑥ 반올림 — round

```python
round(3.14159)       # 3     ← 정수로 반올림
round(3.14159, 2)    # 3.14  ← 소수점 2자리
round(3.14159, 4)    # 3.1416

round(2.5)   # 2  ← 파이썬은 은행가 반올림 (짝수로 반올림)
round(3.5)   # 4  ← 주의! 수학적 반올림과 다름
```

```
⚠️ 파이썬 round() 는 은행가 반올림 (Banker's Rounding)
  .5 일 때 가장 가까운 짝수로 반올림
  2.5 → 2  (2가 짝수)
  3.5 → 4  (4가 짝수)
  4.5 → 4  (4가 짝수)

  수학적 반올림이 필요하면:
  from decimal import Decimal, ROUND_HALF_UP
  또는 int(x + 0.5)
```

---

---

# ⑦ 조건 검사 — any / all

```
any(iterable)  하나라도 True 면 True
all(iterable)  전부 True 여야 True
```

```python
nums = [0, 1, 2, 3]

any(nums)                    # True  ← 1 이상 하나라도 있음
all(nums)                    # False ← 0 이 False 라서

any(x > 2 for x in nums)    # True  ← 3 이 있으니까
all(x > 0 for x in nums)    # False ← 0 이 있으니까
all(x >= 0 for x in nums)   # True  ← 전부 0 이상

# 빈 리스트
any([])   # False
all([])   # True  ← 반례가 없으므로 (vacuous truth)
```

```python
# 실전: 리스트에 특정 조건 만족하는 값 있는지
data = ["정상", "지연", "정상", "취소"]

any(x == "지연" for x in data)    # True  ← 지연 있음
all(x == "정상" for x in data)    # False ← 전부 정상은 아님
```

---

---

# ⑧ 타입 확인 — type / isinstance

```python
type(3)           # <class 'int'>
type(3.14)        # <class 'float'>
type("hello")     # <class 'str'>
type([1, 2])      # <class 'list'>

# 타입 비교
type(3) == int    # True
```

```python
# isinstance — 상속 고려 (권장)
isinstance(3, int)          # True
isinstance(3.14, float)     # True
isinstance(3, (int, float)) # True  ← 여러 타입 동시 체크

# type() vs isinstance() 차이
class MyInt(int): pass
x = MyInt(3)

type(x) == int         # False  ← MyInt 는 int 가 아님
isinstance(x, int)     # True   ← 상속 관계 포함
```

---

---

# ⑨ 형변환 — int / float / str / bool

```python
int("123")        # 123
int(3.9)          # 3    ← 내림 (round 아님)

float("3.14")     # 3.14
float(3)          # 3.0

str(123)          # "123"
str([1, 2, 3])    # "[1, 2, 3]"

bool(0)           # False
bool("")          # False
bool([])          # False
bool(None)        # False
bool(1)           # True
bool("a")         # True
```

```
int() 는 내림 (truncate) — round() 와 다름
  int(3.9) = 3   (버림)
  round(3.9) = 4 (반올림)
```

## int(문자열, 진수) — 진수 변환 ⭐️

```
int(문자열, 밑)
  두 번째 인자 = 몇 진수인지 명시
  이걸 안 쓰면 10진수로만 해석해서 에러 or 잘못된 결과
```

```python
# 2진수 문자열 → 10진수 정수
int("1010", 2)    # 10   ← "1010" 을 2진수로 해석
int("0b1010", 2)  # 10   ← 0b 접두사 있어도 동일

# 3진수 문자열 → 10진수
int("1200", 3)    # 45

# 8진수 문자열 → 10진수
int("17", 8)      # 15

# 16진수 문자열 → 10진수
int("ff", 16)     # 255
int("0xff", 16)   # 255
```

```
⚠️ 진수 명시 안 하면 에러
  int("1010")    # 1010  ← 10진수로 해석 (의도와 다름!)
  int("ff")      # ValueError  ← 10진수에 f 없으니까
  int("ff", 16)  # 255 ✅

  자세한 진법 변환 → [[Python_Number_Bases]] 참고
```

## bin() — 10진수 → 2진수 문자열

```python
bin(10)       # '0b1010'   ← 문자열 반환 (0b 접두사 포함)
bin(255)      # '0b11111111'

# 0b 접두사 제거
bin(10)[2:]   # '1010'
```

## oct() / hex() — 8진수 / 16진수 변환

```python
oct(8)        # '0o10'    ← 8진수 문자열 (0o 접두사)
oct(255)      # '0o377'

hex(255)      # '0xff'    ← 16진수 문자열 (0x 접두사)
hex(16)       # '0x10'

# 접두사 제거
oct(8)[2:]    # '10'
hex(255)[2:]  # 'ff'
```

## 진수 변환 한눈에 ⭐️

```
10진수 → 다른 진수:   bin() / oct() / hex()
다른 진수 → 10진수:   int(문자열, 밑)

모든 진수 함수가 문자열 반환 (숫자 아님!)
```

```python
# 10진수 ↔ 각 진수 변환
n = 10

bin(n)          # '0b1010'  → 2진수
oct(n)          # '0o12'    → 8진수
hex(n)          # '0xa'     → 16진수
bin(n)[2:]      # '1010'    → 접두사 없이

int('1010', 2)  # 10        → 2진수 문자열 → 10진수
int('12', 8)    # 10        → 8진수 문자열 → 10진수
int('a', 16)    # 10        → 16진수 문자열 → 10진수
```

## N진법 변환 직접 구현 ⭐️

```
bin() = 2진수  /  oct() = 8진수  /  hex() = 16진수
3진수 / 5진수 / 7진수 등 임의 진수 → 직접 구현 필요

핵심 원리:
  10진수 n → N진수:   n % N (나머지) 을 반복 수집
  N진수 → 10진수:     int(문자열, N) 사용
```

```python
# 10진수 → 3진수 문자열로 변환
def to_base3(n):
    if n == 0:
        return '0'
    res = ''
    while n > 0:
        res = str(n % 3) + res    # 나머지를 앞에 붙임
        n //= 3
    return res

to_base3(10)   # '101'  (1×9 + 0×3 + 1×1 = 10)
to_base3(0)    # '0'
to_base3(27)   # '1000' (27 = 3³)
```

```
동작 원리:
  n=10:
    10 % 3 = 1  → res = '1'   / n = 3
     3 % 3 = 0  → res = '01'  / n = 1
     1 % 3 = 1  → res = '101' / n = 0
  결과: '101'

나머지를 앞에 붙이는 이유:
  % 연산은 가장 낮은 자리(1의 자리)부터 나옴
  맨 앞에 붙여야 높은 자리부터 순서대로
  → 반대로 하면 뒤집어야 함 ([::-1] 또는 res = res[::-1])
```

## 3진법 뒤집기 문제 패턴 ⭐️

```
문제 유형:
  "n 을 3진법으로 바꾸고 뒤집어서 다시 10진수로 반환"

핵심 패턴:
  1. 10진수 → 3진수 문자열 (직접 구현)
  2. 문자열 뒤집기 [::-1]
  3. 3진수 문자열 → 10진수 int(문자열, 3)
```

```python
def solution(n):
    # 1. 10진수 → 3진수 문자열
    res = ''
    while n > 0:
        res = str(n % 3) + res
        n //= 3

    # 2. 문자열 뒤집기 ([::-1] 는 이미 역순이 되어 있으니 그냥 사용)
    # res 는 이미 올바른 3진수 표현
    res = res[::-1]    # 뒤집기

    # 3. 뒤집힌 3진수 문자열 → 10진수
    answer = int(res, 3)
    return answer

# n=45:
# 45 → 3진수: '1200'
# 뒤집기:     '0021'
# int('0021', 3) = 0×27 + 0×9 + 2×3 + 1 = 7
solution(45)   # 7
```

```python
# 더 간결하게 (한 번에)
def solution(n):
    res = ''
    while n > 0:
        res = str(n % 3) + res
        n //= 3
    return int(res[::-1], 3)
```

## 임의 진수 변환 범용 함수

```python
# N진수로 변환 (2~36진수 가능)
def to_base(n, base):
    digits = "0123456789ABCDEFGHIJKLMNOPQRSTUVWXYZ"
    if n == 0:
        return '0'
    res = ''
    while n > 0:
        res = digits[n % base] + res
        n //= base
    return res

to_base(10, 2)   # '1010'  2진수
to_base(10, 3)   # '101'   3진수
to_base(255, 16) # 'FF'    16진수

# 역방향: int(문자열, 진수) 사용
int('1010', 2)   # 10
int('101', 3)    # 10
int('FF', 16)    # 255
```

## 이진수 더하기 패턴 ⭐️

```
"1010" + "0101" = ?  (이진수 문자열 덧셈)

"1010" 은 문자열 → 그냥 + 하면 "10100101" (이어붙임)
→ int(문자열, 2) 로 10진수로 변환 후 더하기
→ bin() 으로 2진수 문자열로 변환
→ [2:] 로 0b 접두사 제거
```

```python
bin1 = "1010"
bin2 = "0101"

# 한 줄로
result = bin(int(bin1, 2) + int(bin2, 2))[2:]
# '1111'

# 단계별로 풀면
a = int("1010", 2)   # 10
b = int("0101", 2)   # 5
c = a + b            # 15
result = bin(15)[2:] # '1111'
```

```
bin(int(bin1, 2) + int(bin2, 2))[2:]
      ↑                           ↑
      "1010" → 10진수 10          '0b1111' → '1111'
      "0101" → 10진수 5
      10 + 5 = 15
      bin(15) = '0b1111'

흐름:
  2진수 문자열 → int(문자열, 2) → 10진수 정수
  10진수 정수  → bin(정수)      → '0b...' 2진수 문자열
  '0b...'      → [2:]           → 순수 2진수 문자열
```

```python
# 이진수 비트 연산
a = "1010"
b = "0110"

bin(int(a, 2) & int(b, 2))[2:]   # '10'   AND
bin(int(a, 2) | int(b, 2))[2:]   # '1110' OR
bin(int(a, 2) ^ int(b, 2))[2:]   # '1100' XOR
```

---

---

# ⑩ is_integer() — 소수점 없는지 확인 (float 메서드)

```
float 객체의 메서드 (내장함수는 아니지만 자주 쓰임)
소수점 이하가 0 인지 확인
정수처럼 생긴 float 인지 체크할 때 유용
```

```python
(3.0).is_integer()    # True   ← 소수점 없음
(3.14).is_integer()   # False
(5.0).is_integer()    # True
(5.5).is_integer()    # False

# 실전: 나누어 떨어지는지 확인
def is_divisible(a, b):
    return (a / b).is_integer()

is_divisible(10, 2)   # True  (10 / 2 = 5.0)
is_divisible(10, 3)   # False (10 / 3 = 3.333...)
```

```python
# 리스트에서 정수로 표현 가능한 것만 필터링
nums = [1.0, 2.5, 3.0, 4.7, 5.0]
integers = [int(x) for x in nums if x.is_integer()]
# [1, 3, 5]
```

```
⚠️ int 에는 is_integer() 없음 → float 전용
  (3).is_integer()     # AttributeError
  (3.0).is_integer()   # True ✅
  → int 는 항상 정수이므로 체크 불필요
```

## ① for 문 안에 max/min 넣지 말 것

```python
arr = [5, 3, 9, 1]

# ❌ 하나씩 꺼내서 max 에 넣음 → TypeError
for c in arr:
    print(max(c))   # int 는 iterable 이 아님

# ✅ 리스트 통째로
print(max(arr))   # 9
```

## ② max("10", "9") 는 "9"

```python
# 문자열 비교는 첫 글자 아스키코드 기준
max("10", "9")   # "9"  ← '9'(57) > '1'(49)

# 숫자 크기 비교하려면 int() 변환
max(int("10"), int("9"))  # 10
```

## ③ ord() 는 한 글자만

```python
ord("ABC")  # TypeError
ord("A")    # 65 ✅
```