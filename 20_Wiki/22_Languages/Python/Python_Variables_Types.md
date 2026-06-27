---
aliases:
  - 변수
  - 자료형
  - String
  - Integer
  - Boolean
  - TypeCasting
  - f-string
  - type
  - repr
tags:
  - Python
related:
  - "[[Python_Control_Flow]]"
  - "[[Python_String_Methods]]"
  - "[[00_Python_HomePage]]"
---
# Python_Variables_Types — 변수와 자료형

## 한 줄 요약

```
변수  = 데이터를 저장하는 이름표          name = "Spark"
자료형 = 데이터의 생김새                  문자 / 정수 / 소수 / 참거짓

Dynamic Typing:
  파이썬은 타입을 직접 선언 안 해도 알아서 판단
  C / Java:  int count = 0;   ← 타입 직접 선언 필수
  Python:    count = 0        ← 알아서 int 로 인식
```

---

---

# ① 기본 자료형 4대장

## str — 문자열

```python
tool    = "Airflow"
version = '2.5.1'    # 숫자처럼 생겼어도 따옴표 있으면 문자열!

# f-string — 변수 섞어 쓰기
msg = f"현재 {tool} 버전은 {version} 입니다."
# → "현재 Airflow 버전은 2.5.1 입니다."
```

## int — 정수

```python
count   = 100
workers = 4
total   = count * workers   # 400

# 인덱스 / 개수 세기에 사용
```

## float — 실수

```python
accuracy = 98.5
pi       = 3.14159

# ⚠️ 부동소수점 오차
0.1 + 0.2   # 0.30000000000000004 (컴퓨터 이진수 계산 한계)

# 돈 계산 등 정밀도 필요하면 decimal 모듈 사용
from decimal import Decimal
Decimal('0.1') + Decimal('0.2')   # 0.3 ✅
```

## bool — 불린

```python
is_active = True    # 반드시 대문자 T
has_error = False   # 반드시 대문자 F

# true / false → SyntaxError
# True / False → ✅

if is_active:
    print("서버 가동 중!")
```

## None — 값 없음

```python
result = None   # 0 이나 False 와 다름 — "값 자체가 없음"

if result is None:       # None 비교는 == 아니라 is 사용
    print("데이터가 아직 안 왔어요.")

# None 인지 확인
result is None      # True
result is not None  # False
result == None      # 동작은 하지만 is 권장
```

---

---

# ② 형 변환 (Type Casting)

```
웹 / 파일 / API 에서 읽어오면 무조건 문자열(str) 로 옴
→ 숫자처럼 생겨도 str 이라서 연산하면 TypeError

→ 직접 타입 변환 필요
```

```python
port = "8080"           # API 에서 읽어옴 → str

# ❌ 문자열에 숫자 더하기 → TypeError
next_port = port + 1

# ✅ int() 로 변환 후 연산
next_port = int(port) + 1   # 8081
```

## 자주 쓰는 변환

```python
int("100")      # 100
int(3.9)        # 3    ← 버림 (round 아님)
int("3.5")      # ❌ ValueError — 소수 문자열은 바로 int 불가
                # → float("3.5") 먼저, 그다음 int()

float("3.14")   # 3.14
float(3)        # 3.0

str(100)        # "100"
str([1, 2])     # "[1, 2]"

bool(0)         # False
bool("")        # False
bool([])        # False
bool(None)      # False
bool(1)         # True
bool("a")       # True
bool([0])       # True  ← 빈 리스트 아님! 주의
```

```
TypeError 뜨면:
  print(type(변수명)) 먼저 찍어볼 것
  눈엔 숫자로 보여도 str 인 경우가 많음
```

## int(문자열, 진수) — 진수 변환 ⭐️

```
int(문자열, 밑)
  두 번째 인자 = 몇 진수인지 명시
  이걸 빼면 10진수로만 해석 → 의도와 다른 결과 or 에러
```

```python
# 2진수 문자열 → 10진수 정수
int("1010", 2)    # 10   ← "1010" 을 2진수로 해석
int("0b1010", 2)  # 10   ← 0b 접두사 있어도 동일

# 8진수 → 10진수
int("17", 8)      # 15

# 16진수 → 10진수
int("ff", 16)     # 255
int("0xff", 16)   # 255
```

```
⚠️ 진수 명시 안 하면 실수하기 쉬움
  int("1010")    # 1010  ← 10진수로 해석 (이진수 아님!)
  int("ff")      # ValueError  ← 10진수에 f 없음
  int("ff", 16)  # 255 ✅
```

## bin() — 10진수 → 2진수 문자열

```python
bin(10)       # '0b1010'  ← 문자열 반환 (0b 접두사 포함)
bin(255)      # '0b11111111'

# 0b 접두사 제거
bin(10)[2:]   # '1010'
```

## 이진수 연산 패턴 ⭐️

```
이진수 문자열끼리 더하려면:
  "1010" + "0101" → 그냥 더하면 "10100101" (문자열 이어붙임!)
  → int(문자열, 2) 로 10진수 변환 후 연산
  → bin() 으로 다시 2진수 변환
  → [2:] 로 0b 접두사 제거
```

```python
bin1 = "1010"
bin2 = "0101"

result = bin(int(bin1, 2) + int(bin2, 2))[2:]
# '1111'

# 단계별
int("1010", 2)   # 10
int("0101", 2)   # 5
10 + 5           # 15
bin(15)          # '0b1111'
bin(15)[2:]      # '1111'
```

```
흐름:
  2진수 문자열 → int(문자열, 2) → 10진수 정수
  10진수 정수  → bin(정수)      → '0b...' 2진수 문자열
  '0b...'      → [2:]           → 순수 2진수 문자열
```

```python
# 비트 연산도 동일 패턴
a, b = "1010", "0110"

bin(int(a, 2) & int(b, 2))[2:]   # '10'   AND
bin(int(a, 2) | int(b, 2))[2:]   # '1110' OR
bin(int(a, 2) ^ int(b, 2))[2:]   # '1100' XOR
```

---

---

# ③ Iterable — 이터러블이 뭔가?

```
"for 문에 넣을 수 있는 것" = Iterable (반복 가능한 객체)
내부적으로 __iter__() 메서드가 있음

쉽게: 하나씩 꺼낼 수 있는 것
```

```
✅ Iterable (for 문에 넣을 수 있음):
  str      "hello"   → h e l l o
  list     [1, 2, 3] → 1 2 3
  tuple    (1, 2, 3) → 1 2 3
  dict     {"a": 1}  → 키 a
  set      {1, 2, 3} → 순서 없이
  range    range(5)  → 0 1 2 3 4

❌ Not Iterable (for 문에 못 넣음):
  int      10        → TypeError
  float    3.14      → TypeError
  bool     True      → TypeError
  None               → TypeError
```

```python
# str 도 Iterable
for c in "hello":
    print(c)         # h e l l o

# int 는 Iterable 아님
for i in 10:
    print(i)         # TypeError: 'int' is not iterable

# → 숫자로 반복하고 싶으면 range()
for i in range(10):
    print(i)         # 0 1 2 3 4 ... 9
```

## 이터러블과 자료형 변환

```python
# list() / tuple() / set() 로 변환 가능
list("hello")     # ['h', 'e', 'l', 'l', 'o']
tuple([1, 2, 3])  # (1, 2, 3)
set([1, 1, 2, 2]) # {1, 2}  ← 중복 제거

# int / float / bool 는 변환 안 됨
list(10)          # TypeError
```

---

---

# ④ 실무 주의사항

## Airflow / 환경변수는 전부 str

```python
# Airflow Variable 에 100 저장해도 코드로 오면 "100" (str)
batch_size = Variable.get("batch_size")   # "100" (str)
batch_size = int(Variable.get("batch_size"))   # 100 (int) ✅

# os.environ 도 마찬가지
port = os.environ.get("PORT")    # "8080" (str)
port = int(os.environ.get("PORT", "8080"))   # 8080 (int) ✅
```

## 타입 확인 습관

```python
# TypeError 뜨면 먼저 타입 확인
print(type(변수))    # 타입 확인
print(repr(변수))    # 값 확인 (공백/특수문자 포함)
```

> type / isinstance / repr 상세 → [[Python_Type_Checking]]

## None 체크

```python
# ❌ None 에 연산하면 TypeError
result = None
result + 1    # TypeError

# ✅ None 체크 먼저
if result is not None:
    result + 1
```