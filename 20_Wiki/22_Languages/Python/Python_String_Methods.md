---
aliases:
  - split
  - join
  - strip
  - replace
  - 문자열함수
  - 파싱
  - Parsing
  - 확장자 추출
  - isdigit
  - isalpha
  - translate
  - maketrans
  - chr
  - swapcase
  - readlines
  - splitlines
tags:
  - Python
related:
  - "[[Python_Variables_Types]]"
  - "[[Python_Lists_Tuples]]"
  - "[[00_Python_HomePage]]"
  - "[[Spark_General_Transformations]]"
  - "[[Python_String_Indexing_Slicing]]"
  - "[[Python_Membership_In]]"
  - "[[SQL_Regular_Expression]]"
---

# Python_String_Methods

## 개념 한 줄 요약

> **"문자열을 쪼개고, 붙이고, 교체하고, 검증하는 도구 모음."**

|도구|역할|
|---|---|
|`split()`|문자열을 쪼개서 리스트로|
|`join()`|리스트를 문자열로 합치기|
|`replace()`|특정 문자 찾아서 교체|
|`strip()`|양끝 공백·쓰레기 제거|
|`translate()`|여러 문자를 한 번에 교체|
|`count()`|특정 문자·문자열 등장 횟수 세기|
|`isdigit()`|숫자인지 문자인지 확인|
|`ord() / chr()`|문자 ↔ 숫자 변환|
|`swapcase()`|대↔소 뒤집기|


---

---

# ① split() — 쪼개기

> 문자열을 특정 기준(구분자)으로 잘라서 **리스트로 반환**한다.

```python
# 공백 기준 (스페이스·탭·엔터 전부 처리)
text = "Spark is   very    fast"
text.split()
# ['Spark', 'is', 'very', 'fast']  <- 공백 여러 개도 알아서 처리

# 특정 문자 기준
"apple,banana,grape".split(",")
# ['apple', 'banana', 'grape']

# maxsplit: 딱 n번만 자르기
log = "ERROR 2026-01-27 서버가 터졌습니다"
log.split(" ", 1)
# ['ERROR', '2026-01-27 서버가 터졌습니다']  <- 로그 레벨 분리
```

## rsplit() — 오른쪽에서 자르기

```python
# 파일명·확장자 분리
filename = "data.v1.final.csv"
name, ext = filename.rsplit(".", 1)
# name = "data.v1.final"
# ext  = "csv"
# 앞에서 자르면 "data" 만 분리됨 -> 오른쪽에서 잘라야 정확
```

## 빈 문자열 함정

```python
"".split(",")
# ['']  <- 빈 리스트 [] 가 아님!

len("".split(","))  # 1  <- 0이 아니라 1 -> 개수 셀 때 버그 유발
```

---

---

# ② join() — 합치기

> **`"구분자".join(리스트)`** 리스트 요소 사이에 구분자를 끼워 하나의 문자열로 합친다.

```python
words = ['Spark', 'is', 'cool']

" ".join(words)    # 'Spark is cool'
" -> ".join(words) # 'Spark -> is -> cool'
"".join(words)     # 'Sparkiscool'
```

## 이스케이프 문자 구분자

```python
answer = ['apple', 'banana', 'grape']

print('\n'.join(answer))
# apple
# banana
# grape

print('\t'.join(answer))
# apple    banana    grape
```

---

---

# ③ strip() — 공백·쓰레기 제거

> 파일을 읽어오면 눈에 보이지 않는 공백·줄바꿈이 섞여 에러를 유발한다. **split 하기 전에 무조건 strip 습관화.**

```python
text = "   \n\t  Hello, Airflow!  \n  "

text.strip()    # 'Hello, Airflow!'           <- 양쪽 전부
text.lstrip()   # 'Hello, Airflow!  \n  '     <- 왼쪽만
text.rstrip()   # '   \n\t  Hello, Airflow!'  <- 오른쪽만
```

## 특정 문자 제거

```python
dirty = "000000Data_Pipeline_001.csv.bak"
dirty.lstrip("0").rstrip(".bak")
# 'Data_Pipeline_001.csv'
```

## 실무 패턴 — readlines() 줄바꿈 제거

```python
for line in open('file.txt').readlines():
    line = line.rstrip('\n')  # 줄 끝 \n 제거 후 처리
```

---

---

# ④ replace() — 갈아끼우기

> "위치(Index)" 가 아니라 "값(Value)" 을 찾아서 바꾼다.

```python
text = "I like Java. Java is heavy."
text.replace("Java", "Python")
# 'I like Python. Python is heavy.'  <- 전부 다 바뀜

# 삭제: 빈 문자열로 교체
"1,000,000원".replace(",", "").replace("원", "")
# '1000000'  <- int() 변환 가능
```

## 원본은 안 바뀐다

```python
s = "Hello"
s.replace("H", "J")  # 이렇게만 쓰면 아무 일도 안 일어남!
print(s)  # 여전히 'Hello'

# 반드시 변수에 다시 담아야 함
s = s.replace("H", "J")
print(s)  # 'Jello'
```

## 반복 치환 패턴 — 딕셔너리 순회 ⭐️

```
여러 단어를 순서대로 치환할 때
딕셔너리를 만들고 for 문으로 순회하며 replace 반복 적용

핵심:
  s = s.replace(k, v)   ← s 에 다시 담는 것 필수
  매번 치환된 결과를 s 에 업데이트하면서 순차 진행
```

```python
# 영단어 → 숫자 치환 문제
def solution(s):
    en_word = {
        "zero": "0", "one": "1", "two": "2",
        "three": "3", "four": "4", "five": "5",
        "six": "6", "seven": "7", "eight": "8", "nine": "9"
    }
    for k, v in en_word.items():
        s = s.replace(k, v)   # 치환 결과를 s 에 다시 저장
    return int(s)

# "one4seveneight" → "1" + "4" + "7" + "8" → "1478" → 1478
```

```
처음에 하기 쉬운 실수:
  if k in s: s = s.replace(k, v)
  → 순서 보존 안 됨 / 불필요한 조건 체크

  올바른 방법:
  for k, v in en_word.items():
      s = s.replace(k, v)    ← 조건 없이 바로 replace
  replace 는 없는 단어면 그냥 원본 반환 → if 없어도 안전
```

## enumerate 로 더 간결하게 ⭐️


```python
# 딕셔너리 없이 enumerate 활용
def solution(s):
    numbers = ["zero", "one", "two", "three", "four",
               "five", "six", "seven", "eight", "nine"]

    for i, word in enumerate(numbers):
        s = s.replace(word, str(i))
        # i=0, word="zero"  → s.replace("zero", "0")
        # i=1, word="one"   → s.replace("one", "1")
        # ...

    return int(s)
```

```
enumerate 패턴이 왜 더 나은가:
  딕셔너리 선언 불필요
  인덱스(i) 가 곧 치환할 숫자
  numbers[i] = word, i = 그 숫자 → 자연스러운 대응

  for i, word in enumerate(numbers):
      s = s.replace(word, str(i))
               ↑           ↑
           "zero"          "0"
```

## replace 반복 vs if k in s 비교

```python
# ❌ if 조건 넣으면 코드만 길어지고 안전하지도 않음
for k, v in en_word.items():
    if k in s:                 # 불필요
        s = s.replace(k, v)

# ✅ 조건 없이 바로 replace
for k, v in en_word.items():
    s = s.replace(k, v)        # 없는 단어는 그냥 원본 반환 → 안전
```


---

---

# ⑤ translate() + maketrans() — 한 번에 여러 개 바꾸기

> **replace 는 순차 실행이라 A→B, B→A 를 동시에 못 한다.** 변환 테이블을 만들어서 한 번에 적용해야 한다.

## replace 두 번의 함정

```python
text = "AB"
text.replace("A", "B").replace("B", "A")
# 'AA'  <- 원래 의도는 'BA' 였는데!
# A->B 로 먼저 바꿔서 "BB" 가 된 다음, B->A 로 바꿔서 "AA"
```

## translate 로 해결

```python
text = "Hello Python World"
table = str.maketrans("ABo", "BA0")  # 길이 같아야 함
text.translate(table)
# 'Hell0 Pyth0n W0rld'  <- 한 번에 동시 적용
```

## 실전 활용 — 숫자를 문자로 변환

```python
def solution(age):
    aa = str(age)
    return aa.translate(str.maketrans('0123456789', 'abcdefghij'))

solution(25)  # 'cf'  (2->c, 5->f)
```

```
str.maketrans('0123456789', 'abcdefghij')
  0->a, 1->b, 2->c, 3->d, 4->e
  5->f, 6->g, 7->h, 8->i, 9->j
```

|방식|코드|
|---|---|
|`chr` 방식|`"".join(chr(97 + int(i)) for i in str(age))`|
|`translate` 방식|`str(age).translate(str.maketrans('0123456789', 'abcdefghij'))`|

> 둘 다 같은 결과. translate 가 루프 없이 더 간결하다.

---

---

# ⑥ 문자 ↔ 숫자 변환 — ord() / chr()

> **`ord()` : 문자 → 아스키(유니코드) 번호** **`chr()` : 아스키 번호 → 문자**

```python
ord('a')  # 97
ord('A')  # 65
ord('0')  # 48
ord('z')  # 122

chr(97)   # 'a'
chr(65)   # 'A'
chr(48)   # '0'
```

## 아스키 번호 체계

```
숫자:   '0'=48  ~  '9'=57   (총 10개)
대문자: 'A'=65  ~  'Z'=90   (총 26개)
소문자: 'a'=97  ~  'z'=122  (총 26개)

대문자 -> 소문자: +32
```

## 활용 패턴

```python
# 숫자 자리 -> 알파벳 변환 (chr 방식)
"".join(chr(97 + int(i)) for i in "259")
# 'cfj'  (2->c, 5->f, 9->j)

# 알파벳 순서 계산
ord('c') - ord('a')  # 2  <- 'c' 는 0부터 세면 2번째
ord('z') - ord('a')  # 25

# 대문자 -> 소문자 변환
chr(ord('A') + 32)   # 'a'

# 알파벳 순환 (z 다음은 a)
chr((ord('z') - ord('a') + 1) % 26 + ord('a'))  # 'a'
```

## 시저 암호 — 알파벳 순환 이동 ⭐️

```
시저 암호:
  각 알파벳을 n칸씩 밀기
  z 에서 n칸 밀면 a 부터 다시 시작 (순환)
  대문자 / 소문자 기준이 다름
```

## 핵심 공식

```python
# 대문자 기준
(ord(ch) - ord('A') + n) % 26 + ord('A')
#          ↑ 0~25 범위로 정규화   ↑ 다시 'A' 기준으로 복원
# +n 이동 후 % 26 = 26을 넘으면 처음으로 돌아옴

# 소문자 기준
(ord(ch) - ord('a') + n) % 26 + ord('a')
```

```
왜 ord(ch) - ord('A') 하나:
  'A' = 65, 'B' = 66, ..., 'Z' = 90
  -ord('A') 하면 A=0, B=1, ..., Z=25 로 정규화
  → 0~25 범위에서 계산 후 % 26 으로 순환
  → +ord('A') 로 원래 아스키 범위로 복원

예시: 'Y' 를 3칸 이동
  ord('Y') = 89
  89 - 65 = 24   (Y는 0부터 24번째)
  24 + 3  = 27
  27 % 26 = 1    (순환 → B)
  1 + 65  = 66
  chr(66) = 'B'  ✅
```

## 전체 풀이 — 단계별 방식

```python
def solution(s, n):
    answer = ''
    for s1 in s:
        if s1 == ' ':
            answer += ' '
        elif 'A' <= s1 <= 'Z':
            p     = ord(s1) - ord('A')   # 0~25 정규화
            new_p = (p + n) % 26          # n 이동 + 순환
            answer += chr(new_p + ord('A'))  # 복원
        else:
            p     = ord(s1) - ord('a')
            new_p = (p + n) % 26
            answer += chr(new_p + ord('a'))
    return answer
```

## Pythonic 버전 — base 변수로 통합 ⭐️

```python
def solution(s, n):
    answer = ''
    for ch in s:
        if ch == ' ':
            answer += ' '
        else:
            base = ord('A') if ch.isupper() else ord('a')
            answer += chr((ord(ch) - base + n) % 26 + base)
    return answer
```

```
base = ord('A') if ch.isupper() else ord('a')
  대문자이면 base = 65 ('A')
  소문자이면 base = 97 ('a')
  → 대소문자 분기를 한 줄로 처리

핵심 공식 통합:
  (ord(ch) - base + n) % 26 + base
  ↑ 정규화     ↑ 이동  ↑ 순환  ↑ 복원
```

## 단계별 vs Pythonic 비교

```
단계별 방식:
  elif 'A' <= s1 <= 'Z':  대문자 분기
  else:                    소문자 분기
  → 코드 길지만 각 단계 이해하기 쉬움

Pythonic 방식:
  base = ord('A') if ch.isupper() else ord('a')
  → 공통 공식 하나로 통합
  → 코드 짧고 수식 구조 명확

처음에 짤 때 → 단계별 방식으로 로직 확인
리팩토링    → Pythonic 방식으로 통합
```

## 자주 하는 실수 translate vs chr/ord

```python
# ❌ chr 로 변환 안 하고 숫자 그대로 더함
answer += ord(s1) - ord('A')   # int + str → TypeError

# ✅ chr() 로 반드시 문자로 변환
answer += chr((ord(s1) - ord('A') + n) % 26 + ord('A'))
```

```
translate: 변환 규칙이 고정적이고 루프 없이 처리하고 싶을 때
chr/ord:   변환 규칙이 수식으로 표현 가능할 때 (순환, 암호화 등)
```

---

---

# ⑦upper() / lower() / swapcase() — 대소문자 변환

```python
text = "Python Is Easy"
text.upper()     # 'PYTHON IS EASY'   전부 대문자
text.lower()     # 'python is easy'   전부 소문자
text.swapcase()  # 'pYTHON iS eASY'   대↔소 뒤집기

# 대소문자 무시하고 비교
user_input = "Yes"
if user_input.lower() == "yes":
    print("통과!")
```

```text
swapcase():
  대문자 → 소문자
  소문자 → 대문자
  동시에 뒤집음
```

---

---

# ⑧ startswith() / endswith() — 접두사·접미사 검사

```python
filename = "data_2024.csv"

filename.startswith("data_")             # True
filename.endswith(".txt")                # False
filename.endswith((".csv", ".txt"))      # True  <- 튜플로 여러 조건 OR
filename.lower().endswith(".csv")        # True  <- 대소문자 무시
```

## 리스트 컴프리헨션과 결합

```python
files = ["data.csv", "README.md", "data_backup.txt", "result.csv"]

csv_files  = [f for f in files if f.endswith(".csv")]
data_files = [f for f in files if f.startswith("data_")]

count = sum(f.endswith(".csv") for f in files)  # 2
```

## 불리언 → 정수 변환

```python
def check_prefix(my_string, prefix):
    return int(my_string.startswith(prefix))
# True -> 1, False -> 0
```

---
---
# ⑨ count() — 등장 횟수 세기

> 문자열 안에서 특정 문자 또는 문자열이 몇 번 나오는지 센다.


```python
# 문자열에 바로 .count() 가능 — 리스트 변환 불필요
order = "369369"
order.count('3')   # 2
order.count('6')   # 2
order.count('9')   # 2

# 여러 개 동시에 셀 때
order.count('3') + order.count('6') + order.count('9')   # 6
```

## count() vs Counter — 언제 뭘 쓰나

```python
from collections import Counter

text = "369369"

# count() — 특정 문자 1~2개만 셀 때
text.count('3')   # 2  ← 간단, import 불필요

# Counter — 전체 문자 빈도를 한 번에 볼 때
Counter(text)
# Counter({'3': 2, '6': 2, '9': 2})  ← 딕셔너리처럼 사용 가능
Counter(text)['3']   # 2
```

```
count()   특정 문자 몇 개인지만 궁금할 때  → 짧고 간단
Counter   전체 문자 빈도를 한꺼번에 볼 때  → 딕셔너리처럼 사용 가능
          없는 키 조회해도 KeyError 없이 0 반환
```

```python
# Counter 의 추가 기능
c = Counter("aabbbcc")
c.most_common(2)   # [('b', 3), ('a', 2)]  ← 빈도 높은 순 TOP N
```

>더 자세한 내용은 [[Python_Collections_Modules]] 참고 

## 부분 문자열도 셀 수 있음


```python
text = "banana"
text.count("an")   # 2  ('an', 'an')
text.count("na")   # 2  ('na', 'na')

# 단, 겹치는 구간은 세지 않음
"aaa".count("aa")  # 1  ('aa' + 'a', 앞의 aa 소비 후 'a' 만 남는다.)
```

---

---

# ⑩ isdigit() / isalpha() / isalnum() — 데이터 검증

|함수|True 조건|
|---|---|
|`.isdigit()`|`0~9` 만 있을 때|
|`.isalpha()`|한글·영어만 있을 때|
|`.isalnum()`|숫자+문자 조합 (특수문자·공백 없을 때)|

```python
"25".isdigit()     # True
"Alice".isalpha()  # True
"1000원".isdigit() # False  <- '원' 때문에
"-5".isdigit()     # False  <- '-' 때문에
"3.14".isdigit()   # False  <- '.' 때문에
```

## 음수·소수점은 False — float 으로 우회

```python
def is_number(s):
    try:
        float(s)
        return True
    except ValueError:
        return False

is_number("3.14")   # True
is_number("-10")    # True
is_number("1,000")  # False  <- 쉼표 때문에
```

---
---
# ⑪ splitlines() — 줄바꿈으로 쪼개기 ⭐️

```
HTML / 파일 텍스트 처리 시 줄바꿈 기준으로 쪼갤 때 사용
split("\n") 과 다르게 \r\n / \r 도 전부 자동 처리
```

```python
text = "line1\nline2\r\nline3\rline4"

text.split("\n")    # ['line1', 'line2\r', 'line3\rline4']  ← \r 남음!
text.splitlines()   # ['line1', 'line2', 'line3', 'line4']   ← 전부 처리 ✅
```

## split vs splitlines 차이

```
split("\n"):
  \n 만 기준으로 자름
  Windows 줄바꿈 \r\n 이면 \r 이 남음
  HTML / API 응답 처리 시 버그 유발 가능

splitlines():
  \n / \r\n / \r 전부 자동 처리
  파일 / HTML / API 응답 처리에 안전
```

## 실전 패턴 — HTML 텍스트 정제 ⭐️

```python
import re

template = "<h1>제목</h1><br/>내용1<br>내용2<p>단락</p>"

# 1. <br> 태그를 줄바꿈으로 변환
clean = re.sub(r"<br\s*/?>", "\n", template)

# 2. 나머지 HTML 태그 전부 제거
clean = re.sub(r"<[^>]+>", "", clean)

# 3. 줄 단위로 쪼개고 공백 제거 + 빈 줄 제거
lines = [l.strip() for l in clean.splitlines() if l.strip()]
# ['제목', '내용1', '내용2', '단락']
```

```
각 단계 역할:
  re.sub(<br>, \n)          → 줄바꿈 기준점 만들기
  re.sub(<[^>]+>, "")        → 태그 전부 제거
  splitlines()               → 줄 단위로 쪼개기
  l.strip()                  → 각 줄 앞뒤 공백 제거
  if l.strip()               → 빈 줄 제거 (strip 후 빈 문자열이면 False)
```

## keepends 옵션

```python
text = "line1\nline2\nline3"
text.splitlines()           # ['line1', 'line2', 'line3']
text.splitlines(keepends=True)  # ['line1\n', 'line2\n', 'line3']  ← 줄바꿈 유지
```

---

---

# ⑫ 문자열 반복 — * 연산자 ⭐️

```
문자열 * n  = 문자열을 n번 반복
리스트 * n  = 리스트를 n번 반복

반복 후 슬라이싱으로 원하는 길이만 잘라내기
→ for 루프 없이 한 줄로 처리
```

## 기본 사용

python

```python
"수박" * 3       # '수박수박수박'
"ha" * 4        # 'hahahaha'
"-" * 10        # '----------'
[1, 2] * 3     # [1, 2, 1, 2, 1, 2]
```

## 반복 + 슬라이싱 패턴 ⭐️

python

```python
# 길이 n 만큼 "수박수박..." 만들기

# 방법 1 — for 루프 (처음에 짜기 쉬운 방식)
def solution(n):
    answer = ''
    for s in '수박' * n:
        if len(answer) < n:
            answer += s
    return answer

# 방법 2 — 반복 + 슬라이싱 (더 간결) ⭐️
def water_melon(n):
    return ("수박" * n)[:n]
```

```
왜 "수박" * n 으로 충분한가:
  n = 5 → "수박" * 5 = "수박수박수박수박수박" (10글자)
  [:5] → "수박수" (5글자)

  n 이 홀수 → 마지막이 "수" 로 끊김 → 슬라이싱이 알아서 처리
  for 루프 + if 없이 한 줄로 해결
```

## 다른 활용 패턴

python

```python
# 구분선 만들기
print("=" * 30)
# ==============================

# 특정 패턴 반복
("AB" * 5)[:7]   # 'ABABABA'

# 리스트 초기화
[0] * 10         # [0, 0, 0, 0, 0, 0, 0, 0, 0, 0]
[[]] * 3         # [[], [], []]  ← 주의: 같은 객체 참조

# 2D 리스트 초기화 (독립적으로)
[[0] * 5 for _ in range(3)]  # 안전한 방법
```

```
for 루프 방식 vs 반복+슬라이싱:
  for 루프   → 의도 명확 / 코드 길다
  * + 슬라이싱 → 한 줄 / 간결 / 속도 빠름

  코테에서 둘 다 정답이지만
  * + 슬라이싱 패턴 알아두면 더 빠르게 풀 수 있음
```

---

---

# 전체 치트시트

|함수|동작|반환|
|---|---|---|
|`split(sep)`|구분자로 쪼개기|리스트|
|`rsplit(sep, n)`|오른쪽에서 n번 쪼개기|리스트|
|`join(list)`|리스트 합치기|문자열|
|`strip()`|양끝 공백 제거|문자열|
|`replace(a, b)`|a 를 b 로 교체 (원본 유지)|문자열|
|`translate(table)`|변환 테이블로 한 번에 교체|문자열|
|`count(sub)`|sub 등장 횟수|int|
|`upper() / lower()`|대소문자 변환|문자열|
|`swapcase()`|대↔소 뒤집기|문자열|
|`startswith() / endswith()`|접두·접미사 확인|bool|
|`isdigit()`|전부 숫자?|bool|
|`isalpha()`|전부 문자?|bool|
|`ord(c)`|문자 -> 아스키 번호|int|
|`chr(n)`|아스키 번호 -> 문자|str|
|`"str" * n`|문자열 n번 반복|문자열|
|`("str" * n)[:k]`|n번 반복 후 k 길이로 자르기|문자열|
|`"문자열" * n`|문자열 n 번 반복|문자열|
|`("패턴" * n)[:n]`|n 글자만 반복 생성|문자열|