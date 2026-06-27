---
aliases:
  - Python 문자열 위치 찾기
  - find
  - index
  - rfind
  - rindex
tags:
  - Python
related:
  - "[[Python_String_Methods]]"
  - "[[Python_Lists_Tuples]]"
  - "[[00_Python_HomePage]]"
---
# Python_String_Search — 문자열 위치 탐색

## 한 줄 요약

```
문자열 안에서 "내가 찾는 글자"가 몇 번째 칸에 있는지 찾아내는 함수
위치(인덱스) 를 숫자로 반환 → 슬라이싱과 조합해서 원하는 부분만 추출
```

---

---

# ① 인덱스는 0부터 시작 ⭐️

```
파이썬 문자열 인덱스는 무조건 0부터

  h  e  l  l  o     w  o  r  l  d
  0  1  2  3  4  5  6  7  8  9  10

find("h")  → 0
find("e")  → 1
find("o")  → 4  (hello 의 o)
find("w")  → 6
find("z")  → -1  (없으면 -1)
```

```python
text = "hello world"

print(text.find("h"))   # 0   ← 0부터 시작!
print(text.find("e"))   # 1
print(text.find("o"))   # 4   (앞에서 처음 나오는 o)
print(text.find("z"))   # -1  (없으면 -1)

# 숫자 문자열에서도 동일
num = 12345
result = str(num).find("3")   # str 로 변환 후 find
print(result)                  # 2  (0부터 세면 1→0, 2→1, 3→2)

# str(k) 패턴 — 숫자를 문자로 변환 후 위치 탐색
num = 98734
k = 7
print(str(num).find(str(k)))   # 2  (9→0, 8→1, 7→2)
```

---

---

# ② find vs index — 없을 때 차이

```
find   → 없으면 -1 반환  (프로그램 안 죽음, 안전)
index  → 없으면 ValueError 에러 발생 (프로그램 멈춤)
```

```python
text = "hello world"

# find — 없어도 안전
text.find("o")    # 4
text.find("z")    # -1  ← 에러 없이 -1 반환

# index — 없으면 에러
text.index("o")   # 4
text.index("z")   # ValueError: substring not found ← 프로그램 멈춤
```

```
언제 뭘 쓰나:
  find   → "있으면 처리하고, 없으면 말지" → if result != -1: 패턴
  index  → "이 값이 무조건 있어야 한다"   → 없으면 에러로 알고 싶을 때
```

```python
# find 실전 패턴
email = "user@gmail.com"
at_pos = email.find("@")

if at_pos != -1:
    user_id = email[:at_pos]   # "user"
    print(user_id)
else:
    print("@ 없는 이메일")
```

---

---

# ③ find vs rfind — 찾는 방향 차이

```
find   → 왼쪽(앞) 에서부터 탐색 → 처음 나오는 위치
rfind  → 오른쪽(뒤) 에서부터 탐색 → 마지막으로 나오는 위치

⚠️ rfind 라고 인덱스가 음수로 나오는 게 아님
   찾는 방향만 반대일 뿐 반환값은 항상 0부터 시작하는 양수
```

```python
path = "/usr/local/bin/python"
#      0123456789...

print(path.find("/"))    # 0   ← 앞에서 첫 번째 /
print(path.rfind("/"))   # 14  ← 뒤에서 마지막 / (양수!)

# 마지막 / 뒤 = 파일명
last_slash = path.rfind("/")
filename = path[last_slash + 1:]
print(filename)   # python
```

```python
# 여러 번 나올 때 비교
data = "apple apple apple"

data.find("apple")    # 0   ← 첫 번째 apple
data.rfind("apple")   # 12  ← 마지막 apple
```

---

---

# ④ 범위 지정 — find(문자열, 시작, 끝)

```
find(찾을값, 시작위치, 끝위치)
  시작위치 이후부터만 탐색
  두 번째, 세 번째 등 특정 순서의 위치를 찾을 때 사용
```

```python
data = "apple apple apple"

data.find("apple")        # 0   ← 첫 번째
data.find("apple", 1)     # 6   ← 1번 이후부터 → 두 번째 apple
data.find("apple", 7)     # 12  ← 7번 이후부터 → 세 번째 apple
```

---

---

# ⑤ 리스트에서는 find 못 씀

```
find / rfind → 문자열(str) 전용
리스트(list) 에는 index() 만 사용 가능
```

```python
# ✅ 문자열
"hello".find("l")           # 2

# ✅ 리스트 → index() 사용
['a', 'b', 'c'].index('b')  # 1

# ❌ 리스트에 find 쓰면 에러
['a', 'b', 'c'].find('b')   # AttributeError: 'list' object has no attribute 'find'
```

---

---

# 실전 패턴 모음

```python
# ① 이메일 아이디 추출
email = "user@gmail.com"
user_id = email[:email.find("@")]    # "user"

# ② 파일 확장자 추출
filename = "report.final.pdf"
ext = filename[filename.rfind(".") + 1:]    # "pdf"  (마지막 . 이후)

# ③ 경로에서 파일명만 추출
path = "/home/user/data/result.csv"
name = path[path.rfind("/") + 1:]    # "result.csv"

# ④ 숫자에서 특정 자리 위치 찾기
num = 98734
k = 7
pos = str(num).find(str(k))    # 2  (0부터 세면 세 번째 자리)

# ⑤ 두 번째로 나오는 위치 찾기
text = "banana"
first = text.find("a")               # 1
second = text.find("a", first + 1)   # 3
```

---

---

# 한눈에 비교

|함수|방향|없을 때|용도|
|---|---|---|---|
|`find(s)`|앞 → 뒤|`-1`|처음 나오는 위치|
|`rfind(s)`|뒤 → 앞|`-1`|마지막으로 나오는 위치|
|`index(s)`|앞 → 뒤|`ValueError`|처음 위치, 반드시 있어야 할 때|
|`rindex(s)`|뒤 → 앞|`ValueError`|마지막 위치, 반드시 있어야 할 때|

```
공통:
  반환값은 항상 0부터 시작하는 양수 인덱스
  rfind 라고 음수 나오는 게 아님
  범위 지정 가능: find(s, start, end)
```