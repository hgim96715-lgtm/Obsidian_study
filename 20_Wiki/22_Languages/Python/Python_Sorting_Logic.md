---
aliases:
  - Sorted
  - Sort
  - Lambda
  - 람다
  - 딕셔너리정렬
  - key옵션
  - 튜플정렬
  - key
tags:
  - Python
related:
  - "[[Python_Lists_Tuples]]"
  - "[[Python_String_Methods]]"
  - "[[Python_Dictionaries]]"
  - "[[Python_Lambda_Map]]"
---
# Python_Sorting_Logic — 정렬

## 한 줄 요약

```
순서가 뒤죽박죽인 데이터를
내가 원하는 '기준(key)' 을 정해서 줄 세우는 기술
```

---

---

# ① sorted() vs sort()

```python
sorted(데이터, key=기준함수, reverse=True/False)
리스트.sort(key=기준함수, reverse=True/False)
```

|특징|`sorted()`|`.sort()`|
|---|---|---|
|원본 변경|❌ 안 바뀜 (안전)|✅ 바뀌어버림 (주의)|
|반환값|정렬된 새 리스트|None|
|사용 대상|모든 iterable (리스트·튜플·딕셔너리 등)|리스트만|
|체이닝|가능|불가 (None 반환이라)|

```python
numbers = [3, 1, 2]

# ❌ 흔한 실수 — sort() 결과를 변수에 담으면 None
result = numbers.sort()
print(result)   # None 😱

# ✅ sort() 는 그냥 혼자 실행
numbers.sort()
print(numbers)  # [1, 2, 3]

# ✅ 원본 유지하려면 sorted()
safe = sorted([3, 1, 2])
print(safe)     # [1, 2, 3]
```


## sorted() 결과를 저장 안 하는 실수 ⭐️

```python
strings = ["sun", "bed", "car"]

# ❌ sorted() 결과를 변수에 안 담으면 버려짐
sorted(strings)      # 정렬됐지만 버려짐!
print(strings)       # ["sun", "bed", "car"] (원본 그대로)

# ✅ 변수에 담거나 바로 return
answer = sorted(strings, key=lambda s: s[1])
return sorted(strings, key=lambda s: s[1])  # 바로 return 도 가능
```

## lambda 에서 s 가 뭔지 헷갈릴 때

```python
strings = ["sun", "bed", "car"]
n = 1

sorted(strings, key=lambda s: s[n])
#                        ↑
#  s = 리스트에서 꺼낸 문자열 하나 ("sun", "bed", "car" 각각)
#  s[1] = 'u', 'e', 'a'   ← n번째 글자

# 헷갈리면 변수명을 word 로 바꾸면 명확
sorted(strings, key=lambda word: word[n])
#                        ↑ 문자열 전체를 받는 변수
```

---

---

# ② key 옵션 — 정렬 기준 지정

```
컴퓨터는 ('A', 30) 덩어리를 주면
무조건 맨 앞 'A' 만 보고 정렬함

key 는 "앞에 있는 거 보지 말고, 뒤 숫자(30) 기준으로 봐!" 라고 지정하는 것
```

```python
data = {'A': 30, 'C': 5, 'B': 15}

# key 없으면 → 튜플 첫 번째(알파벳) 기준 정렬
sorted(data.items())
# [('A', 30), ('B', 15), ('C', 5)]  ← 알파벳 순

# key 있으면 → 값(숫자) 기준 정렬
sorted(data.items(), key=lambda item: item[1], reverse=True)
# [('A', 30), ('B', 15), ('C', 5)]  ← 숫자 큰 순
```

```
key=lambda item: item[1]
  → "원소(item) 하나가 들어오면 [1]번째를 기준으로 삼아라"
  → lambda 는 [[Python_Lambda_Map]] 참고
```

## key 에 튜플 넣기 — 다중 기준 정렬 ⭐️

```
key 에 튜플을 반환하면 다중 기준 정렬
(1순위 기준, 2순위 기준)

1순위 기준이 같으면 → 2순위 기준으로 결정
```

```python
# n번째 글자 기준으로 정렬 + 같으면 전체 문자열 기준
strings = ["sun", "bed", "car"]
n = 1

sorted(strings, key=lambda s: (s[n], s))
#                               ↑     ↑
#                           1순위    2순위 (동점이면 전체 문자열)
# s[1] = 'u', 'e', 'a'
# → 'a'(car), 'e'(bed), 'u'(sun) 순으로 정렬
# → ["car", "bed", "sun"]
```

```
lambda s: (s[n], s) 읽는 법:
  s          = 리스트에서 꺼낸 문자열 하나 (예: "sun")
  s[n]       = n번째 글자 (예: n=1 → "u")
  s          = 전체 문자열 (동점 처리용)
  (s[n], s)  = 튜플로 묶어서 반환
              → 파이썬이 튜플 사전식 정렬 적용

처음에 하기 쉬운 실수:
  lambda s: s[n]  ← 1순위만 있음
  → n번째 글자가 같은 경우 정렬 순서 보장 안 됨
  → 문제에서 "같은 경우 전체 사전순" 이라는 조건 놓침
```

## key 튜플 동작 원리

```python
strings = ["sun", "bed", "car"]
n = 1

# key 함수가 반환하는 튜플:
# "sun" → ('u', 'sun')
# "bed" → ('e', 'bed')
# "car" → ('a', 'car')

# 튜플 정렬:
# ('a', 'car') < ('e', 'bed') < ('u', 'sun')
# → ["car", "bed", "sun"]

# 만약 n번째 글자가 같다면:
# "abc" → ('b', 'abc')
# "xbc" → ('b', 'xbc')   ← n=1 이면 둘 다 'b'
# ('b', 'abc') < ('b', 'xbc')  → 'abc' 가 먼저 (전체 문자열 비교)
```

---
---

# ③ reverse 옵션

```
기본값(False) = 오름차순  1, 2, 3... / 가, 나, 다...
reverse=True  = 내림차순  3, 2, 1...
```

```python
scores = [50, 90, 70]

sorted(scores)               # [50, 70, 90]  오름차순
sorted(scores, reverse=True) # [90, 70, 50]  내림차순 (랭킹, TOP N)
```

---

---

# ④ 튜플 정렬 — 사전식(Lexicographic) 정렬

```
파이썬은 튜플을 만나면
맨 앞(0번째) 값부터 차례대로 비교하는 '본능' 이 있음

1. 첫 번째 값으로 줄 세우기
2. 첫 번째 값이 같으면(동점) → 두 번째 값으로 마저 줄 세우기
3. 두 번째도 같으면 → 세 번째 값으로...
```

```python
data = [(1, 'C'), (3, 'A'), (1, 'B')]

sorted(data)
# [(1, 'B'), (1, 'C'), (3, 'A')]
#   ↑ 1이 두 개라 동점 → 두 번째 값 B, C 로 결정
```

### 튜플 본능 활용 — key=lambda 없이 정렬

```
튜플 만들 때 정렬하고 싶은 기준을 맨 앞에 두면
lambda 없이도 알아서 정렬됨

(가장 중요한 정렬 기준, 두 번째 기준, 그냥 들고 갈 데이터)
```

```python
# 예: 출석한 학생만 랭크 순서로 뽑기
rank       = [3, 1, 2]
attendance = [True, False, True]

# 애초에 (랭크, 인덱스) 순서로 튜플 만들기
arr = [(x, i) for i, x in enumerate(rank) if attendance[i]]
# [(3, 0), (2, 2)]

arr = sorted(arr)
# [(2, 2), (3, 0)]  ← 랭크 기준 자동 정렬, lambda 불필요
```

---

---

# ⑤ 딕셔너리 정렬 — 실전 패턴

```python
data = {'A': 30, 'C': 5, 'B': 15}

# 값(value) 기준 내림차순
sorted(data.items(), key=lambda x: x[1], reverse=True)
# [('A', 30), ('B', 15), ('C', 5)]

# 키(key) 기준 오름차순
sorted(data.items(), key=lambda x: x[0])
# [('A', 30), ('B', 15), ('C', 5)]

# 값 기준 정렬 후 딕셔너리로 복원
dict(sorted(data.items(), key=lambda x: x[1], reverse=True))
# {'A': 30, 'B': 15, 'C': 5}
```

---

---

# ⑥ 문자열 정렬로 비교 — 순서 무시 비교 ⭐️

```
두 문자열이 같은 문자로 이루어졌는지 비교할 때
sorted() 로 정렬하면 순서 무시하고 구성만 비교 가능

"abc" vs "bca" → 순서 다르지만 같은 문자 구성
sorted("abc") = ['a', 'b', 'c']
sorted("bca") = ['a', 'b', 'c']  ← 정렬하면 동일!
```

```python
# 아나그램 확인 (같은 문자로 이루어진지)
sorted("listen") == sorted("silent")   # True
sorted("hello")  == sorted("world")    # False

# 문자 구성 비교
word1, word2 = "abc", "bca"
sorted(word1) == sorted(word2)   # True  ← 순서 달라도 구성 같음
```

## 코딩테스트 실전 — 아나그램 그룹 묶기

```python
# 같은 문자 구성끼리 그룹으로 묶기
words = ["eat", "tea", "tan", "ate", "nat", "bat"]

groups = {}
for word in words:
    key = "".join(sorted(word))   # 정렬된 문자열을 키로
    groups.setdefault(key, []).append(word)

# {'aet': ['eat', 'tea', 'ate'], 'ant': ['tan', 'nat'], 'abt': ['bat']}
```

## sorted() vs Counter — 어떤 걸 쓸까?

```python
from collections import Counter

s1 = "listen"
s2 = "silent"

# sorted() 방식
sorted(s1) == sorted(s2)         # True

# Counter 방식
Counter(s1) == Counter(s2)       # True
```

```
코딩테스트:  sorted()  ← 간단 + 충분히 빠름
실무/가독성: Counter() ← "문자 빈도 비교" 의미 명확
```

>

---

---

# 핵심 정리

```
sorted()   원본 유지, 새 리스트 반환  → 거의 항상 이걸 쓸 것
.sort()    원본 변경, None 반환       → 변수에 담으면 None 함정

key        정렬 기준 지정
           단순 숫자/문자  → key 생략
           튜플·딕셔너리   → key=lambda x: x[1]
           튜플 맨 앞이 기준이면 → lambda 없이 sorted() 그대로

reverse=True   내림차순 (랭킹, TOP N 뽑을 때)

순서 무시 비교:
  sorted(str1) == sorted(str2)   ← 같은 문자 구성인지 확인
  Counter(str1) == Counter(str2) ← 동일 (더 명확)

데이터 엔지니어 반사 공식:
  "빈도수 많은 순서대로" → sorted(..., key=lambda x: x[1], reverse=True)
```