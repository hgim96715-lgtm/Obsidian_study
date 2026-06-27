---
aliases:
  - in operator
  - not in
  - 멤버십 연산자
  - 포함 여부 확인
  - in
  - ==
tags:
  - Python
related:
  - "[[Python_Lists_Tuples]]"
  - "[[Python_Dictionaries]]"
  - "[[Python_String_Methods]]"
  - "[[00_Python_HomePage]]"
---
## 개념 한 줄 요약 (Concept Summary)

**"데이터 꾸러미(리스트, 문자열, 딕셔너리 등) 안에 특정 값이 '있는지(in)', '없는 지(not in)' 확인하여 True/False를 반환하는 연산자."**

---
## 기본 문법 (Basic Syntax)

직관적인 영어 문장처럼 읽힙니다. 
결과는 항상 **불리언(`True` / `False`)** 입니다.

```python
# 1. in (있니?)
print("a" in "apple")  # True

# 2. not in (없니?)
print("z" not in "apple")  # True
```

---
## 자료형별 동작 방식 (How it Works)

`in` 연산자는 자료형에 따라 확인하는 대상이 다릅니다. 
특히 **딕셔너리**를 조심해야 합니다.

### ① 문자열 (String) - 부분 문자열 확인

문장 안에 특정 단어가 포함되어 있는지 확인합니다.
연속된 부분 문자열 포함 여부를 한 줄로 확인가능 

```python
text = "Hello Python World"
if "Python" in text:
    print("파이썬 관련 글이군요!")
```

#### 💡 실전 팁: 대소문자 무시하고 1/0 리턴하기 (한 줄 코딩)

코딩 테스트에서 **"문자열 A 안에 문자열 B가 있으면 1, 없으면 0을 반환하라(대소문자 무시)"** 
문제가 나올 때 `if-else`를 쓸 필요가 없습니다.

**`in` 연산자**는 기본적으로 **연속된 문자열**을 확인하며, **`int()`** 를 씌우면 True는 `1`, False는 `0`으로 변합니다.

```python
def solution(myString, pat):
    # 1. lower(): 둘 다 소문자로 변환 (대소문자 무시)
    # 2. in: pat이 myString 안에 연속으로 들어있는지 확인 (True/False)
    # 3. int(): True면 1, False면 0으로 변환
    return int(pat.lower() in myString.lower())
```

> lower은 [[Python_String_Methods#대소문자 변환 (`upper`, `lower`)|lower_upper]] 참고 


### ⚠️ 주의: 부분 포함(`in`) vs 완전 일치(`{text}==`)

데이터 엔지니어링에서 파일명이나 날짜 형식을 검사할 때, `in`을 쓰면 **치명적인 버그**가 생길 수 있습니다.

**상황:** "이 파일이 `.csv` 파일인지 확인하고 싶다."

- 파일명: `data.csv.backup` (백업 파일이라 처리하면 안 됨!)

```python
filename = "data.csv.backup"

# 1. [위험] in 사용 (부분 포함)
# ".csv"가 문자열 중간에 들어있기만 하면 통과시켜버림 -> 버그 발생!
if ".csv" in filename:
    print("CSV 파일입니다.")  # 출력됨 (오작동)

# 2. [안전] 슬라이싱 + == 사용 (완전 일치)
# 뒤에서 4글자를 잘라낸 뒤, 그 조각이 ".csv"와 똑같은지 비교해야 함.
ext = filename[-4:]  # ".ckup"
if ext == ".csv":
    print("CSV 파일입니다.")  # 출력 안 됨 (정상)
```


#### 💡 [심화] 더 좋은 방법: `startswith`, `endswith`

슬라이싱해서 `{text}==`로 비교하는 것도 좋지만, 파이썬에는 더 직관적인 전용 함수가 있습니다.

>[[Python_String_Methods#특정 파일만 골라내기 (`endswith`, `startswith`)|endswith&startswith]] 내용 참고 

```python
# 파일 끝부분 검사 (Slicing + == 보다 가독성이 좋음)
if filename.endswith(".csv"):
    pass

# 날짜 연도 검사 (text[:4] == "2024" 대신)
if date_text.startswith("2024"):
    pass
```

### ② 리스트/튜플 (List/Tuple) - 요소 확인

리스트 안에 해당 아이템이 있는지 순차적으로(하나씩) 검사합니다.

```python
users = ["user1", "user2", "admin"]
if "admin" in users:
    print("관리자 계정이 존재합니다.")
```

### ③ 딕셔너리 (Dictionary) -  키(Key) 확인

**가장 많이 하는 실수!** 딕셔너리에 `in`을 쓰면 **값(Value)** 이 아니라 **키(Key)** 를 찾습니다.

```python
data = {"name": "Kim", "age": 20}

# (O) 키를 찾음
print("name" in data)  # True

# (X) 값을 찾지 않음
print("Kim" in data)   # False

# 💡 값을 찾고 싶다면? .values()를 써야 함
print("Kim" in data.values()) # True
```

## 성능 최적화 팁 (Performance Tip for DE)

데이터 엔지니어링에서 대용량 데이터를 다룰 때 `in` 연산자의 속도는 자료구조에 따라 천지차이입니다.

### 리스트 (List): O(N) - 느림

데이터가 100만 개면, 최악의 경우 100만 번을 다 뒤져봐야 합니다.

```python
big_list = [i for i in range(1000000)]
# 999999를 찾으려면 100만 번 비교해야 함
if 999999 in big_list: 
    pass
```

### 세트/딕셔너리 (Set/Dict): O(1) - 매우 빠름

해시(Hash) 구조를 사용하므로 데이터가 100만 개여도 **한 번에** 찾습니다. 
**검색(Search)이 목적이라면 리스트를 반드시 세트로 변환(`set()`)해서 찾으세요.**

```python
big_set = set(big_list)
# 데이터가 아무리 많아도 즉시 찾음 (검색 속도 최강)
if 999999 in big_set:
    pass
```

---
## 실전 활용 예제 (Data Filtering)

Airflow나 데이터 파이프라인에서 특정 조건의 데이터를 걸러낼 때 필수적입니다.

```python
allowed_extensions = {".csv", ".json", ".parquet"}
file_name = "data_2024.xlsx"

# 확장자 추출 후 허용 목록에 있는지 확인
ext = "." + file_name.split(".")[-1]

if ext not in allowed_extensions:
    print(f"Error: {ext} 파일은 지원하지 않습니다.")
```
