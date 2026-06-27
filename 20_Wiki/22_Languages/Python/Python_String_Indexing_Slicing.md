---
aliases:
  - String Indexing
  - Slicing
  - 문자열 슬라이싱
  - 문자열 인덱싱
  - 불변객체
tags:
  - Python
related:
  - "[[Python_Lists_Tuples]]"
  - "[[Python_String_Methods]]"
  - "[[00_Python_HomePage]]"
  - "[[Python_Variables_Types]]"
---
# Python_String_Indexing_Slicing — 인덱싱과 슬라이싱

## 한 줄 요약

```
인덱싱  특정 위치 한 글자를 콕 집어내기
슬라이싱 범위를 지정해서 잘라내기

문자열은 불변(Immutable) → 직접 수정 불가
슬라이싱으로 잘라서 새 문자열을 조립해야 함
```

---

---

# ① 인덱싱 — 한 글자만 콕

```
파이썬은 0 부터 셈
뒤에서부터 셀 때는 -1 사용
```

```python
text = "PYTHON"
#  P   Y   T   H   O   N
#  0   1   2   3   4   5   ← 양수 인덱스
# -6  -5  -4  -3  -2  -1  ← 음수 인덱스

text[0]    # 'P'  (맨 앞)
text[5]    # 'N'  (맨 뒤)
text[-1]   # 'N'  (맨 뒤, 음수)
text[-2]   # 'O'
```

---

---

# ② 슬라이싱 — 범위로 자르기

```
문법: [시작 : 끝 : 간격]

⚠️ 끝 인덱스는 포함하지 않음 (End is Exclusive)
   [0:5] = 0, 1, 2, 3, 4  (5 제외)
```

## 기본 자르기

```python
text = "Hello World"
#       01234 56789 10

text[0:5]   # "Hello"   (0 이상 5 미만)
text[:5]    # "Hello"   (처음부터 5 미만)
text[6:]    # "World"   (6부터 끝까지)
text[:]     # "Hello World"  (전체 복사)
```

## 간격(Step) 활용

```python
numbers = "123456789"

numbers[::2]    # "13579"   (2칸씩 건너뛰기)
numbers[::3]    # "147"     (3칸씩 건너뛰기)

# 문자열 뒤집기 ⭐️
numbers[::-1]   # "987654321"
```

---

---

# ③ 슬라이싱 연계 (Chaining) — 자르고 뒤집기

```
슬라이싱 결과도 문자열
→ 뒤에 바로 [::-1] 붙여서 연속 처리 가능
```

```python
text = "Pro Python"

# 1단계: text[:5] → "Pro P"
# 2단계: "Pro P"[::-1] → "P orP"
text[:5][::-1]   # "P orP"
```

```python
# 코테 실전 패턴 — a번째부터 b번째까지 자른 뒤 뒤집기
answer = my_string[a : b+1][::-1]

# b+1 이유:
#   인덱스가 0부터 시작
#   b번째 글자까지 포함하려면 끝 인덱스에 b+1 필요
```

---

---

# ④ 불변성 (Immutability) — 문자열 수정하기

```
문자열은 Immutable (불변 객체)
→ 특정 위치를 직접 수정 불가
→ 잘라서 새 문자열을 조립해야 함

리스트 vs 문자열:
  a = [1, 2, 3]
  a[0] = 9       # ✅ 리스트는 수정 가능

  s = "abc"
  s[0] = 'z'     # ❌ 문자열은 TypeError
```

## 특정 위치 글자 바꾸기

```python
s = "Teat"
#    0123
# 2번 인덱스 'a' 를 's' 로 바꾸고 싶다

# ❌ 직접 대입 → 에러
s[2] = 's'
# TypeError: 'str' object does not support item assignment

# ✅ 앞 + 새글자 + 뒷부분 조립
idx = 2
result = s[:idx] + 's' + s[idx+1:]

# s[:2]    → "Te"   (0, 1번)
# 's'      → "s"    (새 글자)
# s[3:]    → "t"    (3번부터 끝)
# 합치면   → "Test"
```

```
⚠️ 뒷부분 자를 때 함정:
  s[idx:]   → "at"  (기존 'a' 가 살아있음)
  s[idx+1:] → "t"   (기존 'a' 건너뜀) ✅
```

---
---
# ⑤ 실전 — 중간 글자 추출 ⭐️ㅍ

```
홀수 길이 → 가운데 1글자
짝수 길이 → 가운데 2글자
```

## 방법 1 — if/else 로 분기

```python
def solution(s):
    if len(s) % 2:                          # 홀수
        answer = s[len(s) // 2]             # 중간 1글자
    else:                                   # 짝수
        answer = s[len(s)//2-1 : len(s)//2+1]  # 중간 2글자
    return answer

# "abcde" → len=5 (홀수) → s[2] = 'c'
# "qwer"  → len=4 (짝수) → s[1:3] = 'we'
```

## 방법 2 — 슬라이싱 하나로 통합 ⭐️

```python
def string_middle(s):
    return s[(len(s)-1)//2 : len(s)//2 + 1]
```

```
핵심 원리:
  시작 = (len-1) // 2
  끝   = len // 2 + 1

홀수 예시 "abcde" (len=5):
  시작 = (5-1)//2 = 2
  끝   = 5//2 + 1 = 3
  → s[2:3] = 'c'  ← 1글자

짝수 예시 "qwer" (len=4):
  시작 = (4-1)//2 = 1
  끝   = 4//2 + 1 = 3
  → s[1:3] = 'we'  ← 2글자
```

```
왜 이게 자동으로 되나:
  홀수: (len-1)//2 와 len//2 가 같은 값
        → 시작과 끝 차이 = 1 → 1글자

  짝수: (len-1)//2 = len//2 - 1
        → 시작과 끝 차이 = 2 → 2글자

"홀짝 분기 없이 슬라이싱 범위가 알아서 1글자/2글자 반환"
```

## 두 방법 비교

```python
# 방법 1 — if/else (직관적 / 의도 명확)
def solution(s):
    mid = len(s) // 2
    return s[mid] if len(s) % 2 else s[mid-1:mid+1]

# 방법 2 — 슬라이싱 통합 (한 줄 / 고급)
def solution(s):
    return s[(len(s)-1)//2 : len(s)//2 + 1]
```

```
처음엔 방법 1 이 이해하기 쉬움
방법 2 는 범위 계산이 왜 되는지 이해하면 더 깔끔

코테에서 둘 다 정답
```

---

---

# 한눈에 정리

|표현|의미|예시 결과|
|---|---|---|
|`s[0]`|첫 번째 글자|`'P'`|
|`s[-1]`|마지막 글자|`'N'`|
|`s[1:4]`|1~3번 인덱스|`'YTH'`|
|`s[:3]`|처음~2번|`'PYT'`|
|`s[3:]`|3번~끝|`'HON'`|
|`s[::2]`|2칸씩 건너뛰기|`'PTO'`|
|`s[::-1]`|전체 뒤집기|`'NOHTYP'`|
|`s[1:4][::-1]`|잘라서 뒤집기|`'HTY'`|