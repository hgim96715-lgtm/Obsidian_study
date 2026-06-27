---
aliases:
  - 대소문자변환
  - 문자열치환
  - replace
  - re.sub
  - swapcase
  - 정규식치환
  - upper
  - lower
  - group
tags:
  - python
related:
  - "[[Python_String_Methods]]"
  - "[[Python_Modules_Imports]]"
  - "[[Python_Regex]]"
  - "[[00_Python_HomePage]]"
---
# Python_String_Case_Replace — 대소문자 변환 & 문자열 치환

## 한 줄 요약

```
단순 글자 바꾸기 → replace()
패턴(숫자만, 영어만 등) 바꾸기 → re.sub()
```

## 핵심 원칙 — 불변(Immutable)

```
Python 문자열은 불변(Immutable)
변환 메서드를 써도 원본은 절대 바뀌지 않음
→ 반드시 변수에 재할당해야 함
```

```python
text = "hello"
text.upper()       # ❌ 원본 안 바뀜, 반환값 버려짐

text = text.upper() # ✅ 재할당해야 반영
print(text)         # HELLO
```

---

---

# ① 대소문자 변환 5대장

|메서드|동작|예시 결과|
|---|---|---|
|`.upper()`|전체 대문자|`"PyThon"` → `"PYTHON"`|
|`.lower()`|전체 소문자|`"PyThon"` → `"python"`|
|`.swapcase()`|대↔소 반전|`"PyThon"` → `"pYtHON"`|
|`.capitalize()`|문장 맨 앞만 대문자|`"hello world"` → `"Hello world"`|
|`.title()`|각 단어 첫 글자 대문자|`"hello world"` → `"Hello World"`|

```python
s = "PyThon"

print(s.upper())      # PYTHON
print(s.lower())      # python
print(s.swapcase())   # pYtHON

text = "hello python programming"
print(text.capitalize())  # Hello python programming
print(text.title())       # Hello Python Programming
```

```
capitalize vs title:
  capitalize → 문장 전체에서 맨 앞 글자 하나만
  title      → 공백 기준 각 단어 첫 글자 전부
```

---

---

# ② 상태 확인 — isupper / islower / isalpha 등

```
True / False 로 반환 — 조건문에서 사용
```

|메서드|True 조건|
|---|---|
|`.isupper()`|문자열 전체가 대문자|
|`.islower()`|문자열 전체가 소문자|
|`.isalpha()`|알파벳만 (숫자·공백 없음)|
|`.isdigit()`|숫자만|
|`.isalnum()`|알파벳 + 숫자만 (공백 없음)|
|`.isspace()`|공백만|

```python
print("A".isupper())       # True
print("a".islower())       # True
print("hello".isalpha())   # True
print("123".isdigit())     # True
print("abc123".isalnum())  # True
print("hello world".isalpha())  # False  (공백 포함이라 False)
```

```
코딩테스트 패턴:
  for c in s:
      if c.isupper(): ...   한 글자씩 판별
      if c.isdigit(): ...   숫자인지 판별
```

---

---

# ③ replace() — 단순 문자열 치환

```
replace(찾을값, 바꿀값)
replace(찾을값, 바꿀값, 횟수)  ← 몇 번만 바꿀지 제한 가능

특징:
  정확한 텍스트 매칭 — 패턴(정규식) 인식 안 함
  import 없이 사용
  위치(Index)가 아니라 값(Value)을 찾아서 바꿈
```

```python
text = "Hello World World"

text.replace("World", "Python")     # "Hello Python Python"  (전체 치환)
text.replace("World", "Python", 1)  # "Hello Python World"   (1번만 치환)
```

## ⚠️ 가장 흔한 실수 — 재할당 안 하기

```python
numbers = "1 2 3 4 5"

numbers.replace(" ", ",")     # ❌ 원본 그대로, 반환값 버려짐
print(numbers)                # 1 2 3 4 5  (바뀌지 않음)

numbers = numbers.replace(" ", ",")  # ✅ 재할당 필수
print(numbers)                       # 1,2,3,4,5
```

```
replace() 는 새 문자열을 반환할 뿐
원본을 수정하지 않음 → 반드시 = 로 받아야 함
```

## replace() 실전 패턴

```python
# 여러 개 연속 치환 (체이닝)
text = "Hello, World! 123"
text = text.replace(",", "").replace("!", "").replace(" ", "_")
# Hello_World_123

# 문자 반복 확장 (코딩테스트 패턴)
k = 2
row = ".x."
expanded = row.replace('.', '.' * k).replace('x', 'x' * k)
# ..xx..

# 공백 제거
text = "  hello  world  "
text = text.replace(" ", "")  # helloworld
# 양쪽 공백만 제거하려면 strip() 사용
```

## ⚠️ replace() 로 정규식 패턴 쓰면 안 됨

```python
text = "abc123"

text.replace("[a-z]", "X")   # ❌ "[a-z]" 라는 5글자를 문자 그대로 찾음
                              # 매칭 없으면 원본 그대로 반환

# ✅ 패턴이 필요하면 re.sub() 사용
import re
re.sub(r"[a-z]", "X", text)  # XXX123
```

---

---

# ④ re.sub() — 패턴 기반 치환

```
re.sub(패턴, 바꿀값, 원본텍스트)

언제 씀:
  "모든 소문자"를 X로
  "모든 숫자"를 제거
  "특수문자 전부"를 공백으로
  → 범위·종류 단위로 바꿀 때
```

```python
import re

text = "abc123def"

re.sub(r"[a-z]", "X", text)   # XXX123XXX  (소문자 → X)
re.sub(r"\d", "", text)        # abcdef     (숫자 제거)
re.sub(r"\s+", " ", text)      # 연속 공백 → 공백 1개
re.sub(r"[^가-힣]", "", text)  # 한글만 남기기
```

## 자주 쓰는 패턴

|패턴|의미|
|---|---|
|`[a-z]`|소문자 알파벳|
|`[A-Z]`|대문자 알파벳|
|`[a-zA-Z]`|모든 알파벳|
|`\d`|숫자 (= `[0-9]`)|
|`\D`|숫자 아닌 것|
|`\s`|공백·탭·줄바꿈|
|`\w`|알파벳+숫자+밑줄|
|`[^가-힣]`|한글이 아닌 것|

## re.sub() 함수 매핑 — 람다 연결

```python
# 매칭된 각 문자에 함수 적용 가능
import re

s = "PyThon"

# 알파벳을 잡아서 대↔소 반전 (swapcase 와 동일한 효과)
result = re.sub(
    r"[a-zA-Z]",
    lambda x: x.group(0).lower() if x.group(0).isupper() else x.group(0).upper(),
    s
)
# pYtHON

# x.group(0): 정규식에 매칭된 문자 하나하나
# 복잡한 조건(숫자는 그대로 두고 알파벳만 바꾸기 등)에 유용
```

---

---

# 실전 선택 기준

```
replace()       정확한 문자열을 찾아 바꿀 때
re.sub()        범위·종류 단위 패턴으로 바꿀 때

판단 방법:
  "사과" → "배"         replace
  "모든 숫자" → ""       re.sub
  "[a-z]" 넣고 싶다면   re.sub (절대 replace 아님)
```

---

---

# 코딩테스트 패턴 3단계

```python
s = "PyThon123"

# [하수] for + if 로 한 글자씩
res = ""
for c in s:
    if c.isupper(): res += c.lower()
    elif c.islower(): res += c.upper()
    else: res += c

# [고수] swapcase (숫자·공백은 그대로)
res = s.swapcase()   # pYtHON123

# [초고수] re.sub 람다 (조건이 복잡할 때만)
res = re.sub(r"[a-zA-Z]",
    lambda x: x.group(0).lower() if x.group(0).isupper() else x.group(0).upper(), s)
# pYtHON123
```

```
swapcase 와 re.sub 람다의 차이:
  swapcase  → 모든 문자(숫자 포함) 에 적용, 숫자·공백은 변화 없음
  re.sub    → [a-zA-Z] 만 잡아서 적용
              "숫자는 건드리지 않고 알파벳만" 같은 조건 분기 가능
```