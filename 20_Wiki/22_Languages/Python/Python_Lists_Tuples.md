---
aliases:
  - List
  - Tuple
  - 리스트
  - 튜플
  - Slicing
  - 슬라이싱
  - 배열
  - 사전식비교
  - insert
  - pop
  - append
  - 슬라이싱으로 회전
tags:
  - Python
related:
  - "[[Python_Control_Flow]]"
  - "[[]]"
  - "[[Python_Dictionaries]]"
  - "[[Python_Sorting_Logic]]"
  - "[[Python_Variables_Types]]"
  - "[[00_Python_HomePage]]"
  - "[[Python_String_Indexing_Slicing]]"
  - "[[Python_Unpacking]]"
  - "[[Python_Membership_In]]"
  - "[[Python_Looping_Helpers]]"
  - "[[Python_Type_Checking]]"
  - "[[Python_Collections_Modules]]"
---
# Python_Lists_Tuples

## 개념 한 줄 요약

> **"데이터를 순서대로 담는 기차 같은 자료구조."**

|구분|List `[]`|Tuple `()`|
|---|---|---|
|**비유**|화물 기차 (짐을 뺐다 꼈다 가능)|밀봉된 기차 (출발하면 수정 불가)|
|**특징**|Mutable (수정 가능)|Immutable (수정 불가)|
|**용도**|데이터 수집, 가공, 변환|설정값, 좌표, 함수 리턴값|

---

---

# ① 생성과 조회

```python
tasks = ["extract", "transform", "load"]

tasks[0]    # "extract"   <- 앞에서 첫 번째 (0부터 시작!)
tasks[1]    # "transform" <- 앞에서 두 번째
tasks[-1]   # "load"      <- 뒤에서 첫 번째
tasks[-2]   # "transform" <- 뒤에서 두 번째
len(tasks)  # 3
```

```
인덱스 구조:
  ["extract",  "transform",  "load"]
       0            1           2      <- 앞에서
      -3           -2          -1      <- 뒤에서
```

---

---

# ② 추가 — append / insert / extend / +

```python
a = [1, 2, 3]

# append: 맨 뒤에 추가
a.append(4)
print(a)  # [1, 2, 3, 4]

# append: 리스트를 넣으면 통째로 들어감
a.append([5, 6])
print(a)  # [1, 2, 3, 4, [5, 6]]  <- 리스트 안에 리스트!

# insert: 원하는 위치에 삽입
a = [1, 2, 3]
a.insert(0, 99)   # 0번 자리에 99 삽입 (맨 앞에 추가)
print(a)  # [99, 1, 2, 3]

a.insert(2, 88)   # 2번 자리에 88 삽입
print(a)  # [99, 1, 88, 2, 3]

# extend: 리스트의 알맹이만 풀어서 합치기
a = [1, 2]
a.extend([3, 4])
print(a)  # [1, 2, 3, 4]

# +: 원본 유지, 새 리스트 반환
a = [1, 2]
c = a + [3, 4]
print(a)  # [1, 2]       <- 원본 그대로
print(c)  # [1, 2, 3, 4] <- 새 리스트
```

|방식|위치|동작|원본 변경|반환값|
|---|---|---|:-:|---|
|`append(x)`|맨 뒤|x 를 통째로 추가|O|None|
|`insert(i, x)`|i 번 자리|x 를 i 위치에 삽입|O|None|
|`extend(b)`|맨 뒤|b 의 알맹이만 풀어서|O|None|
|`a + b`|—|알맹이 합쳐서 새것 생성|X|새 리스트|

> `mix = a.extend(b)` 하면 `mix` 는 `None` extend 는 원본을 고치는 것이지 반환하지 않는다.

---

---

# ③ 삭제 / 꺼내기 — pop / del / remove / clear / [:-n]

|구분|`pop()`|`pop(0)`|`del`|`remove()`|`clear()`|`[:-n]`|
|---|:-:|:-:|:-:|:-:|:-:|:-:|
|**기능**|맨 뒤 꺼내기|맨 앞 꺼내기|위치로 삭제|값으로 삭제|전체 비우기|뒤에서 n개 제거|
|**반환값**|꺼낸 값|꺼낸 값|없음|없음|없음|새 리스트|

```python
data = ["A", "B", "C", "D"]

# pop(): 맨 뒤 꺼내기 (Stack 패턴)
last = data.pop()
print(last)  # "D"
print(data)  # ["A", "B", "C"]

# pop(0): 맨 앞 꺼내기 (Queue 패턴)
first = data.pop(0)
print(first)  # "A"
print(data)   # ["B", "C"]
# 뒤 데이터가 전부 한 칸씩 당겨지므로 데이터 많을 때 느림 -> deque 권장

# pop(i): 특정 위치 꺼내기
data = ["A", "B", "C", "D"]
mid = data.pop(1)
print(mid)   # "B"
print(data)  # ["A", "C", "D"]

# del: 위치로 그냥 삭제 (반환값 없음)
data = ["A", "B", "C"]
del data[0]
print(data)  # ["B", "C"]

del data[0:2]  # 슬라이싱으로 여러 개 한 번에 삭제
print(data)    # []

# remove: 값으로 찾아서 삭제
data = ["Red", "Green", "Blue"]
data.remove("Green")
print(data)  # ["Red", "Blue"]
# 없는 값 remove 하면 ValueError 발생! -> if 로 먼저 확인

# clear: 전체 비우기 (빈 리스트로 만들기)
data = [1, 2, 3]
data.clear()
print(data)  # []
# del data[:] 와 동일하지만 clear() 가 더 명확

# [:-n] 슬라이싱: 뒤에서 n개 한 번에 제거
answer = [1, 2, 3, 4, 5]
v = 2
if v > 0:
    answer = answer[:-v]
print(answer)  # [1, 2, 3]
# v = 0 이면 answer[:0] = [] 가 되므로 조건 분기 필요
```

## pop 패턴 정리

```
data.pop()   <- 맨 뒤 꺼내기 = Stack (후입선출 LIFO)
data.pop(0)  <- 맨 앞 꺼내기 = Queue (선입선출 FIFO)
data.pop(i)  <- i 번째 꺼내기

Stack 예: 실행 취소(Undo), 괄호 짝 검사
Queue 예: 프린터 대기열, 메시지 큐
```


## 상위 k개 유지 패턴 ⭐️

```
"지금까지 나온 점수 중 상위 k개를 유지하면서
 매일 최하위 점수를 구하라"

핵심 발상:
  score[i:i+k] 처럼 앞으로 나올 것을 자르면 안 됨
  → 지금까지 나온 점수 중 상위 k개를 누적 관리해야 함

패턴:
  1. 새 점수 추가 (append)
  2. 내림차순 정렬
  3. k 초과 시 마지막(최솟값) 제거 (pop)
  4. 현재 최솟값 기록 ([-1])
```

```python
# 방법 1 — sort + pop (직관적)
def solution(k, score):
    answer = []
    hall = []
    for s in score:
        hall.append(s)           # 추가
        hall.sort(reverse=True)  # 내림차순 정렬
        if len(hall) > k:
            hall.pop()           # 맨 뒤(최솟값) 제거
        answer.append(hall[-1])  # 현재 최하위 점수 기록
    return answer

# 방법 2 — min + remove (더 간결)
def solution(k, score):
    q = []
    answer = []
    for s in score:
        q.append(s)
        if len(q) > k:
            q.remove(min(q))     # 최솟값 찾아서 제거
        answer.append(min(q))    # 현재 최솟값 기록
    return answer
```

```
sort + pop vs min + remove:

  sort + pop:
    정렬 후 맨 뒤 = 최솟값
    hall[-1] 로 최솟값 바로 접근
    매번 정렬 → O(n log n)

  min + remove:
    min() 으로 최솟값 찾기
    remove(min()) 으로 값으로 삭제
    min() 두 번 호출 → O(n) × 2
    코드는 더 짧고 직관적

  데이터 적으면 두 방법 차이 없음
  → 가독성 좋은 것 선택
```

---

---

# ④ 탐색 — count / in / isinstance / index / enumerate

## count — 몇 개인지 세기

```python
votes = ["Apple", "Banana", "Apple", "Kiwi", "Apple"]

votes.count("Apple")   # 3
votes.count("Orange")  # 0  <- 없어도 에러 없이 0 반환
```

> 단순히 "있냐 없냐" 확인은 `in` 연산자가 더 빠름. `count` 는 끝까지 전부 훑기 때문.

```python
"Apple" in votes   # True  <- 있냐 없냐만 볼 땐 이게 빠름
```

---

## isinstance — 타입 확인 ⭐️

> `type(x) == list` 보다 `isinstance(x, list)` 를 쓰는 것이 권장된다. 상속 관계까지 고려하기 때문.

```python
isinstance(대상, 타입)   # True / False 반환

# 기본 사용
isinstance([1, 2, 3], list)    # True
isinstance("hello", str)       # True
isinstance(3.14, float)        # True
isinstance(42, int)            # True

# 여러 타입 한 번에 확인 (튜플로 묶기)
isinstance(42, (int, float))   # True  <- int 또는 float 이면 True
isinstance("hi", (int, float)) # False
```

## 실전 패턴 — API 응답이 딕셔너리일 때 리스트로 통일 ⭐️

```
공공데이터 API 의 함정:
결과가 1개일 때    -> item 을 딕셔너리 하나로 반환  {"trn_no": "KTX001"}
결과가 여러 개일 때 -> item 을 리스트로 반환        [{"trn_no": "KTX001"}, ...]

코드에서 항상 리스트로 다루려면
isinstance 로 타입 확인 후 통일해야 함
```

```python
item = data.get("response", {}).get("body", {}).get("items", {}).get("item", [])

# isinstance 로 타입 확인 후 항상 리스트로 통일
items = item if isinstance(item, list) else [item]
# isinstance(item, list) == True  -> 이미 리스트 -> 그대로
# isinstance(item, list) == False -> 딕셔너리 1개 -> [item] 으로 감싸기

for train in items:
    print(train["trn_no"])  # 항상 리스트로 다룰 수 있음
```

> 딕셔너리 .get() 체이닝 패턴은 [[Python_Dictionaries]] 참고

---

```python
data = ["A", "B", "C", "A"]

data.index("A")  # 0  <- 맨 앞 하나만 반환
data.index("C")  # 2
# data.index("Z") -> ValueError!
# 반드시 in 으로 존재 확인 후 사용
```

### 핵심 패턴 — 정렬 후 index 로 순위 찾기

```
문제 상황:
  응급도를 내림차순으로 정렬하는 것까지는 생각했지만,
  정렬된 리스트에서 현재 값의 순위를 찾는 방법을 떠올리지 못했다.

해결:
  sorted() 로 정렬된 복사본을 만들고
  원본 값을 .index() 로 정렬본에서 찾으면 -> 그 위치가 순위
```

```python
scores = [40, 10, 30, 20]

# 1. sorted 로 내림차순 정렬 복사본 생성 (원본 유지)
sorted_scores = sorted(scores, reverse=True)
# -> [40, 30, 20, 10]

# 2. 원본을 순회하면서 index 로 순위 조회
for score in scores:
    rank = sorted_scores.index(score) + 1  # 0부터 시작 -> +1
    print(f"{score} -> {rank}등")

# 40 -> 1등
# 10 -> 4등
# 30 -> 2등
# 20 -> 3등
```

```
흐름 정리:
sorted_scores = [40, 30, 20, 10]
인덱스 번호        0   1   2   3

scores 의 40 -> sorted_scores.index(40) = 0 -> 0+1 = 1등
scores 의 10 -> sorted_scores.index(10) = 3 -> 3+1 = 4등
scores 의 30 -> sorted_scores.index(30) = 1 -> 1+1 = 2등
```

> sorted() 자세한 옵션은 [[Python_Sorting_Logic]] 참고

> 동점자 주의: 같은 값이 여러 개면 `index()` 는 가장 앞 위치만 반환. 동점자를 같은 순위로 처리할 때는 이 패턴으로 충분. 동점자를 각각 다른 순위로 처리해야 하면 `enumerate` 패턴 사용.

### 뒤에서부터 찾기

```python
data = ["A", "B", "C", "A"]

# 마지막 "A" 의 원래 인덱스 구하기
last_idx = len(data) - 1 - data[::-1].index("A")
# -> 3
```

---

## enumerate — 모든 위치 찾기

> `index()` 는 맨 처음 하나만 반환. 전부 찾으려면 `enumerate` + 리스트 컴프리헨션 조합.

```python
data = ["A", "B", "C", "A", "B", "A"]

locations = [i for i, val in enumerate(data) if val == "A"]
# -> [0, 3, 5]
```

> 리스트 컴프리헨션 자세한 내용 -> [[Python_List_Comprehension]] 참고

---

---

# ⑤ 정렬 — sort / sorted

```python
nums = [3, 1, 2]

# sort(): 원본 파괴
nums.sort()
print(nums)  # [1, 2, 3]

# sorted(): 원본 유지, 새 리스트 반환
nums = [3, 1, 2]
new_nums = sorted(nums)
print(nums)      # [3, 1, 2]  <- 원본 그대로
print(new_nums)  # [1, 2, 3]
```

> 자세한 정렬 옵션 -> [[Python_Sorting_Logic]] 참고

---

---

# ⑤-2 뒤집기 — reversed / [:​:-1] / .reverse()

```
뒤집는 방법 3가지 — 상황마다 다름
```

```python
data = [1, 2, 3, 4, 5]

# ① [::-1]  새 리스트 생성 (원본 유지)
data[::-1]              # [5, 4, 3, 2, 1]

# ② reversed()  이터레이터 반환 (for 문에 바로 사용)
list(reversed(data))    # [5, 4, 3, 2, 1]
for x in reversed(data):
    print(x)            # 5 4 3 2 1

# ③ .reverse()  원본 in-place 뒤집기 (반환값 없음!)
data.reverse()
print(data)             # [5, 4, 3, 2, 1]
mix = data.reverse()    # mix = None ← 실수 주의
```

|방법|원본 변경|반환값|문자열 가능|
|---|---|---|---|
|`[::-1]`|❌|새 리스트/문자열|✅|
|`reversed()`|❌|이터레이터|✅ (단, list/str 변환 필요)|
|`.reverse()`|✅|None|❌|

```python
# 문자열 뒤집기
"hello"[::-1]              # 'olleh'
"".join(reversed("hello")) # 'olleh'
```

## reversed() 실전 패턴 ⭐️

```python
# 숫자 → 각 자리 뒤집기
n = 12345
list(reversed(str(n)))              # ['5', '4', '3', '2', '1']
list(map(int, reversed(str(n))))    # [5, 4, 3, 2, 1]
```

```
단계별 분해:
  str(12345)            → '12345'       (문자열로 변환)
  reversed('12345')     → 이터레이터    (뒤집기)
  map(int, ...)         → 각 문자 int 변환
  list(...)             → [5, 4, 3, 2, 1]
```

```python
# 활용 예시

# 자릿수 합계 (뒤집은 결과 합산)
n = 12345
sum(map(int, reversed(str(n))))   # 15 (1+2+3+4+5)

# 팰린드롬 확인 (앞뒤가 같은지)
s = "racecar"
s == "".join(reversed(s))         # True
s == s[::-1]                      # True (더 간결)

# 역순 누적
data = [1, 2, 3, 4, 5]
result = []
for x in reversed(data):
    result.append(x * 2)
# [10, 8, 6, 4, 2]
```

```
언제 뭘 쓰나:
  새 리스트/문자열 바로 필요   → [::-1]  (가장 간결)
  for 문에서 역순 순회         → reversed()  (메모리 효율)
  원본 자체를 뒤집어야 할 때   → .reverse()
  각 자리 숫자로 변환 필요     → list(map(int, reversed(str(n))))
```

---

---

# ⑥ 슬라이싱 — [시작:끝:간격]

> **끝 번호는 포함하지 않는다.** (End Index is Exclusive)

```python
data = [0, 1, 2, 3, 4, 5]

data[:3]    # [0, 1, 2]           <- 처음부터 3개
data[3:]    # [3, 4, 5]           <- 3번 인덱스부터 끝
data[-2:]   # [4, 5]              <- 뒤에서 2개
data[:]     # [0, 1, 2, 3, 4, 5] <- 전체 복사 (원본 독립)
data[::-1]  # [5, 4, 3, 2, 1, 0] <- 전체 뒤집기
```

## 길이로 자르기

```python
text = "20240210_Data_Log"
s, l = 0, 4
print(text[s : s + l])  # "2024"
```

## 뒤에서 n개 가져오기

```python
text = "abcde"
n = 3

# 흔한 실수
print(text[-1 : -n-1])
# -> "" (빈 문자열)
# 이유: 시작(-1=e) 에서 step 기본값 +1(오른쪽) 인데
#       도착점(-n-1=b) 은 왼쪽 -> 도달 불가

# 해결: 시작점을 앞으로 당기기 (추천)
print(text[-n:])  # "cde"
```

## 특정 구간만 뒤집기

```python
def solution(my_string, s, e):
    return my_string[:s] + my_string[s:e+1][::-1] + my_string[e+1:]
# 앞부분 그대로 + 구간만 뒤집기 + 뒷부분 그대로
```

##  [시작:​:간격] — n번째마다 건너뛰기

```
[start:end:step] 에서 end 를 생략하면 끝까지라는 뜻
[start::step]  →  start 인덱스부터 끝까지, step 간격으로

:: 가 두 개 붙어있어서 헷갈리지만
앞 : 는 start / end 구분자
뒤 : 는 step 구분자
end 자리가 그냥 비어있는 것
```

```python
data = [0, 1, 2, 3, 4, 5, 6, 7, 8]

data[::2]    # [0, 2, 4, 6, 8]  ← 처음부터 2칸씩
data[1::2]   # [1, 3, 5, 7]     ← 1번 인덱스부터 2칸씩
data[2::3]   # [2, 5, 8]        ← 2번 인덱스부터 3칸씩
```

```python
# 실전 예시 — 암호 해독 (code 번째마다 글자 뽑기)
cipher = "dfjkdlsoakjfosldjaklsfj"
code = 4

# ❌ 굳이 리스트로 안 바꿔도 됨
arr = list(cipher)
result = arr[code-1::code]   # 리스트 반환

# ✅ 문자열에 바로 슬라이싱 가능
result = cipher[code-1::code]   # 문자열 반환 (더 간결)
```

```
cipher[code-1::code] 분해:
  code-1  → 시작 인덱스 (3번째 = index 3)
  (비어있음) → 끝까지
  code    → step 간격 (4칸씩)
  → index 3, 7, 11, 15... 위치의 문자만 뽑음
```

## 문자열은 리스트로 안 바꿔도 슬라이싱 가능

```
파이썬에서 문자열은 '읽기 전용 리스트' 처럼 동작
인덱싱 / 슬라이싱 / for 루프 모두 리스트 변환 없이 바로 됨
```

```python
text = "Hello"

text[0]      # 'H'       인덱싱 ✅
text[1:3]    # 'el'      슬라이싱 ✅
text[::-1]   # 'olleH'   뒤집기 ✅
text[::2]    # 'Hlo'     step ✅

for ch in text:          # 반복 ✅
    print(ch)

# 단, 수정은 불가 (문자열은 immutable)
text[0] = 'J'   # ❌ TypeError
# 수정이 필요하면 list() 로 변환 후 ''.join() 으로 복원
arr = list(text)
arr[0] = 'J'
''.join(arr)    # 'Jello'
```

```
list() 변환이 필요한 경우:
  문자 수정이 필요할 때만

list() 변환 불필요:
  인덱싱, 슬라이싱, for 루프, step 건너뛰기
  → 문자열에 바로 쓰면 됨
```

## 슬라이싱으로 회전 — rotate 패턴 ⭐️

```
deque.rotate() 없이 슬라이싱으로 회전 구현
문자열 / 리스트 모두 동일한 방식
```

|방향|코드|결과|
|---|---|---|
|오른쪽 (뒤 → 앞)|`A[-1] + A[:-1]`|`"hello"` → `"ohell"`|
|왼쪽 (앞 → 뒤)|`A[1:] + A[0]`|`"hello"` → `"elloh"`|

```python
A = "hello"

# 오른쪽으로 1칸 회전 — 맨 뒤가 맨 앞으로
A[-1] + A[:-1]
# 'o' + 'hell' = 'ohell'

# 왼쪽으로 1칸 회전 — 맨 앞이 맨 뒤로
A[1:] + A[0]
# 'ello' + 'h' = 'elloh'
```

```python
# 리스트도 동일
A = [1, 2, 3, 4, 5]

[A[-1]] + A[:-1]   # [5, 1, 2, 3, 4]  오른쪽
A[1:] + [A[0]]     # [2, 3, 4, 5, 1]  왼쪽
```

```
슬라이싱 회전 vs deque.rotate():
  슬라이싱    새 문자열/리스트 생성 → 원본 유지
  deque.rotate()  원본 in-place 회전 → 더 효율적

  코딩테스트 문자열 조작 → 슬라이싱 패턴
  대량 데이터 회전 → deque.rotate()
```

## 회전 결과인지 확인 — b*2 패턴 ⭐️

```
"a 가 b 를 회전한 결과인지" 확인할 때
b 를 두 번 이어붙이면 모든 회전 경우가 포함됨

b = "hello"
b*2 = "hellohello"
  → "hello" 의 모든 회전 결과가 b*2 안에 있음
  → a 가 b*2 의 부분 문자열이면 a 는 b 의 회전 결과
```

```python
# a 가 b 의 회전 결과인지 확인
solution = lambda a, b: (b * 2).find(a)
# a 가 있으면 시작 인덱스 반환
# a 가 없으면 -1 반환

solution("ohell", "hello")   # 4  ← 있음 (회전 결과)
solution("world", "hello")   # -1 ← 없음 (회전 결과 아님)

# 있는지 없는지만 판단할 때
a in (b * 2)   # True / False
```

```
b = "hello"
b * 2 = "hellohello"
         ↑↑↑↑↑        원본
              ↑↑↑↑↑   한 칸 회전들

모든 회전 결과:
  "hello"  → "hellohello" 에 포함 ✅
  "ohell"  → "hellohello" 에 포함 ✅ (index 4)
  "lohel"  → "hellohello" 에 포함 ✅
  "world"  → "hellohello" 에 없음  ❌
```

> `deque.rotate()` 상세 → [[Python_Collections_Counter#② deque — 양방향 큐]] 참고

---

---

# ⑦ Tuple (튜플)

```python
db_info = ("localhost", 5432)

db_info[0]    # "localhost"
# db_info[0] = "127.0.0.1"  -> TypeError 발생
```

## Unpacking

```python
numbers = 1, 2, 3        # 괄호 없어도 콤마면 튜플
a, b, c = numbers        # 풀어서 변수에 담기

a, *b = (1, 2, 3, 4)     # a=1, b=[2, 3, 4]  <- 나머지 전부 *
```

> 요소가 하나일 때는 반드시 콤마 필요 `("hello")` -> 문자열 / `("hello",)` -> 튜플

---

---

# ⑧ 사전식 비교 (Lexicographical Comparison)

> 앞에서부터 차례대로 비교, 처음으로 다른 값이 나오는 순간 결과 확정.

```python
date1 = [2024, 1, 31]
date2 = [2024, 2, 1]

# 2024 == 2024 -> 다음 / 1 < 2 -> True 확정
print(date1 < date2)  # True

def solution(date1, date2):
    return int(date1 < date2)  # True -> 1, False -> 0
```

---
---

# ⑧-2 [-1:] — 빈 리스트 안전한 마지막 원소 비교 ⭐️

```
문제:
  answer[-1] → 리스트 비어있으면 IndexError
  answer[-1:] → 비어있으면 [] 반환 → 에러 없음

[-1:] = 마지막 원소를 리스트 형태로 가져오기
  비어있으면  → []
  원소 있으면 → [마지막값]
```

## 기본 동작

```python
a = [1, 2, 3]
a[-1]     # 3          (정수)
a[-1:]    # [3]        (리스트)

b = []
b[-1]     # IndexError ← 에러!
b[-1:]    # []         ← 안전 ✅
```

## 실전 — 연속된 같은 값 제거 ⭐️

```python
# 방법 1 — len() + 인덱스 비교 (처음에 짜기 쉬운 방식)
def solution(arr):
    answer = []
    for i in range(len(arr)):
        if len(answer) == 0 or answer[-1] != arr[i]:
            answer.append(arr[i])
    return answer

# 방법 2 — [-1:] 슬라이싱 (더 Pythonic) ⭐️
def no_continuous(s):
    a = []
    for i in s:
        if a[-1:] == [i]:   # 비어있으면 [] == [i] → False → append
            continue
        a.append(i)
    return a
```

```
방법 2 핵심:
  a[-1:]      → a 가 비어있으면 []  / 있으면 [마지막값]
  [i]         → 현재 값을 리스트로 감싸서 비교
  [] == [i]   → False → continue 안 함 → append
  [3] == [3]  → True  → continue (중복이니 건너뜀)

  len() 체크 + 인덱스 비교를 한 줄로 대체
```

## 두 방법 비교

```python
arr = [1, 1, 2, 2, 3, 1, 1]

# 방법 1
if len(answer) == 0 or answer[-1] != arr[i]:
    answer.append(arr[i])

# 방법 2 (동일한 결과)
if a[-1:] != [i]:
    a.append(i)
```

```
방법 1:
  len() 로 빈 리스트 먼저 체크 → 직관적 / 의도 명확

방법 2:
  [-1:] 로 한 줄에 처리 → 간결
  비어있으면 [] → 어떤 [i] 와도 다름 → 자동으로 append

코테에서 둘 다 정답
[-1:] 패턴 알아두면 조건 분기 줄일 수 있음
```

## [-1:] 활용 패턴 정리

```python
# 마지막 원소 안전하게 비교
if a[-1:] != [x]:     # 마지막이 x 가 아니거나 비어있으면
    a.append(x)

if a[-1:] == [x]:     # 마지막이 x 와 같으면 (비어있으면 False)
    continue

# 마지막 원소 꺼내기 (빈 리스트여도 에러 없음)
last = a[-1:]         # [] 또는 [마지막값]
last = a[-1:][0] if a else None   # None 처리가 필요할 때
```

---

---

# ⑨ 튜플 이어붙이기 — 실전 패턴

```
튜플도 + 로 이어붙일 수 있음
리스트 컴프리헨션과 조합해서 대량 데이터 처리에 활용
```

## 기본 원리

```python
# 튜플 + 튜플 = 새 튜플
a = (1, 2, 3)
b = (4,)       # ← 1개짜리 튜플은 콤마 필수!
a + b          # (1, 2, 3, 4)

# 헷갈리는 포인트
(4)            # 그냥 정수 4 (괄호일 뿐)
(4,)           # 1개짜리 튜플 ✅ (콤마가 튜플임을 결정)
```

## rows_with_ts 패턴 — 대량 데이터에 컬럼 추가

```
배치 적재 시 자주 나오는 패턴:
  rows = [(값1, 값2, ...), (값1, 값2, ...), ...]
  여기에 updated_at 같은 공통 컬럼을 추가해야 할 때
  리스트 컴프리헨션 + 튜플 이어붙이기로 처리
```

```python
from datetime import datetime

rows = [
    ("A1100001", "서울대병원", "서울"),
    ("A1100002", "세브란스병원", "서울"),
]

# ① 각 튜플에 datetime.now() 이어붙이기
rows_with_ts = [r + (datetime.now(),) for r in rows]

# 결과:
# [
#   ("A1100001", "서울대병원", "서울", datetime(2026, 3, 23, ...)),
#   ("A1100002", "세브란스병원", "서울", datetime(2026, 3, 23, ...)),
# ]
```

```
단계별 분해:

  r               = ("A1100001", "서울대병원", "서울")  ← 원본 튜플
  datetime.now()  = 2026-03-23 12:00:00  ← 현재 시각
  (datetime.now(),) = (2026-03-23 12:00:00,)  ← 1개짜리 튜플 (콤마 필수!)
  r + (datetime.now(),) = ("A1100001", "서울대병원", "서울", 2026-03-23...)

  [r + (datetime.now(),) for r in rows]
  → rows 의 모든 튜플에 적용 → 새 리스트 반환
```

```python
# 공통 값 여러 개 추가 시
status = "active"
rows_extended = [r + (datetime.now(), status) for r in rows]
#                        ↑           ↑
#                   2개짜리 튜플 이어붙이기 (괄호 안에 콤마로 구분)
```

```
활용:
  Airflow DAG 에서 배치 적재 시 updated_at 추가
  execute_values 로 DB 에 한 방에 넣을 때
  → [[Airflow_Hooks#hook.get_conn() + execute_values]] 참고
```


---

---

# ⑩ a[-1:] — 빈 리스트 안전 참조 ⭐️

```
a[-1]  → 리스트가 비어있으면 IndexError
a[-1:] → 리스트가 비어있으면 [] 반환 (에러 없음)

슬라이싱은 범위를 벗어나도 에러가 나지 않음
→ 빈 리스트 체크 없이 바로 비교 가능
```

## 기본 동작

```python
a = [1, 2, 3]
a[-1]    # 3       → 마지막 원소
a[-1:]   # [3]     → 마지막 원소를 리스트로

a = []
a[-1]    # ❌ IndexError
a[-1:]   # ✅ []   → 에러 없이 빈 리스트 반환
```

## 실전 패턴 — 연속 중복 제거 ⭐️

```
문제: 연속으로 같은 값이 오면 하나만 남기기
  [1, 1, 3, 3, 0, 1, 1] → [1, 3, 0, 1]
```

```python
# 방법 1 — if len(answer)==0 or 조건 (명시적)
def solution(arr):
    answer = []
    for i in range(len(arr)):
        if len(answer) == 0 or answer[-1] != arr[i]:
            answer.append(arr[i])
    return answer

# 방법 2 — a[-1:] 슬라이싱 활용 (더 간결) ⭐️
def no_continuous(s):
    a = []
    for i in s:
        if a[-1:] == [i]:   # 빈 리스트여도 에러 없음
            continue
        a.append(i)
    return a
```

```
방법 2 동작 원리:

  a = [] 일 때:
    a[-1:]  = []
    [i]     = [1]
    [] == [1] → False → append 실행 ✅

  a = [1] 일 때 i = 1:
    a[-1:]  = [1]
    [i]     = [1]
    [1] == [1] → True → continue (중복 무시)

  a = [1] 일 때 i = 3:
    a[-1:]  = [1]
    [i]     = [3]
    [1] == [3] → False → append 실행
```

```
방법 1 vs 방법 2:

  방법 1: len(answer)==0 or answer[-1] != arr[i]
    → 조건이 길지만 의도가 명확
    → "비어있거나 마지막이 다르면 추가"

  방법 2: a[-1:] == [i] 이면 continue
    → 슬라이싱으로 빈 리스트 체크 생략
    → 조건이 간결
    → 코테에서 자주 쓰는 패턴
```

## a[-1:] 활용 패턴 정리

```python
# 빈 리스트일 때 기본값 처리
a = []
a[-1:] or [0]   # [0]  ← 빈 리스트는 Falsy

a = [5]
a[-1:] or [0]   # [5]

# 마지막 원소와 비교
stack = []
stack[-1:] == [x]   # 비어있어도 False (에러 없음)
```

---

---

# ⑩ [-1:] 슬라이싱 — 빈 리스트 안전 비교 ⭐️

```
a[-1]   → 빈 리스트면 IndexError 발생
a[-1:]  → 빈 리스트면 [] 반환 (에러 없음)

마지막 원소를 리스트 형태로 가져오는 패턴
→ 빈 리스트 체크 없이 바로 비교 가능
```

## 비교

```python
a = []

a[-1]    # IndexError ← 빈 리스트에서 에러
a[-1:]   # []         ← 에러 없이 빈 리스트 반환 ✅
```

## 실전 패턴 — 연속 중복 제거 ⭐️

```python
# 방법 1 — 직접 길이 체크 + 인덱스 비교
def solution(arr):
    answer = []
    for i in range(len(arr)):
        if len(answer) == 0 or answer[-1] != arr[i]:
            answer.append(arr[i])
    return answer

# 방법 2 — a[-1:] 슬라이싱 활용 (더 간결)
def no_continuous(s):
    a = []
    for i in s:
        if a[-1:] == [i]:   # 빈 리스트면 [] == [i] → False → append
            continue
        a.append(i)
    return a
```

```
방법 2 핵심 원리:
  a[-1:]       → 마지막 원소를 리스트로 가져옴
  [i]          → 비교 대상도 리스트로 감싸기
  == [i]       → 리스트 vs 리스트 비교

  a 가 빈 리스트일 때:
    a[-1:]  = []
    [] == [i]  → False → append 실행 ✅ (에러 없음)

  a 마지막이 같은 값일 때:
    a[-1:]  = [i]
    [i] == [i] → True → continue (중복 스킵)

  a 마지막이 다른 값일 때:
    a[-1:]  = [j]  (j ≠ i)
    [j] == [i] → False → append 실행
```

## a[-1] vs a[-1:] 정리

|표현|빈 리스트|반환 타입|용도|
|---|---|---|---|
|`a[-1]`|IndexError|원소 자체|값을 꺼낼 때|
|`a[-1:]`|`[]` 반환|리스트|안전 비교할 때 ⭐️|

```python
# 패턴 확장 — 마지막 원소가 조건 만족 시
stack = [1, 2, 3]

# 안전하게 꺼내서 비교
if stack[-1:] == [3]:
    print("마지막이 3")

# 빈 스택도 안전하게
stack = []
if stack[-1:] == [3]:   # [] == [3] → False → 에러 없음
    print("마지막이 3")
```

---

---

# 초보자 실수 체크리스트

|실수|원인|해결|
|---|---|---|
|`list_b = list_a` 후 원본이 바뀜|참조 복사 (별명 붙이기)|`list_a[:]` 또는 `.copy()`|
|`mix = a.extend(b)` 가 None|extend 는 None 반환|`a.extend(b)` 후 `a` 사용|
|`mix = data.reverse()` 가 None|reverse() 는 None 반환|`data.reverse()` 후 `data` 사용|
|`text[-1:-n-1]` 이 빈 문자열|step 기본값 +1 방향 불일치|`text[-n:]` 사용|
|`index()` 에서 ValueError|없는 값 조회|`if val in data:` 로 먼저 확인|
|정렬 후 순위 못 구함|index() 패턴을 모름|`sorted_list.index(값) + 1`|
|요소 1개 튜플이 문자열로 됨|콤마 누락|`("hello",)` 콤마 필수|
|`reversed(str(n))` 바로 출력 안 됨|이터레이터라 바로 못 씀|`list(reversed(str(n)))` 또는 `"".join(reversed(str(n)))`|
|빈 리스트에서 `a[-1]` → IndexError|인덱스 직접 접근|`a[-1:]` 슬라이싱 사용|
|마지막 원소 비교 시 빈 리스트 체크 따로 필요|`if len(a)==0 or ...` 패턴|`a[-1:] == [값]` 으로 한 번에 처리|
|빈 리스트에서 `a[-1]` 사용|IndexError|`a[-1:]` 슬라이싱 → 빈 리스트면 `[]` 반환|

---

