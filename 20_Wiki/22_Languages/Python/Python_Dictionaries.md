---
aliases:
  - Dictionary
  - Dict
  - 딕셔너리
  - 해시맵
  - HashMap
  - Key-Value
  - JSON객체
  - fromkeys
  - get()
  - 체이닝
  - 중첩 딕셔너리
tags:
  - Python
related:
  - "[[00_Python_HomePage]]"
  - "[[Python_Lists_Tuples]]"
  - "[[Python_Sorting_Logic]]"
---
# Python_Dictionaries — 딕셔너리

## 한 줄 요약

```
키(Key) 로 값(Value) 을 찾는 이름표 저장소
Hash Table 구조 → 데이터 100만 개여도 즉시 조회
```

---

---

# ① 기본 CRUD

```python
user = {"name": "airflow", "age": 10}

# 조회
user["name"]        # "airflow"
user.get("email")   # None  (없는 키 → 에러 없음)

# 추가 / 수정
user["email"] = "admin@test.com"  # 없으면 추가
user["age"] = 11                  # 있으면 수정

# 삭제
del user["age"]
user.pop("email", None)   # 없어도 에러 없음 (None 반환)
```

---

---

# ② .get() — 안전한 조회 ⭐️

```python
config = {"env": "prod"}

# ❌ 대괄호 — 없는 키 → KeyError
config["retries"]

# ✅ .get() — 없는 키 → 기본값 반환
config.get("retries", 3)    # 3
config.get("retries")       # None (기본값 생략)
```

## 중첩 딕셔너리 체이닝

```python
# 공공데이터 API 응답 파싱 — 중간에 키 없어도 안전
items = (
    data
    .get("response", {})
    .get("body", {})
    .get("items", {})
    .get("item", [])
)
```

```
동작 원리:
  "body" 가 없으면 → {} 반환
  {} 에서 다음 .get() → 역시 없음
  → 최종적으로 [] 반환 (KeyError 없음)

대괄호 방식 data["response"]["body"]["items"]["item"]
  → 중간에 하나라도 없으면 KeyError 대참사
```

---

---

# ③ in 연산자 — 키 존재 확인

```python
user = {"name": "airflow", "age": 10}

"name" in user      # True  ← Key 기준
"airflow" in user   # False ← Value 는 Key 가 아님

# Value 존재 확인
"airflow" in user.values()   # True
```

---

---

# ④ items / keys / values

```python
config = {"owner": "team_A", "schedule": "@daily"}

config.keys()    # dict_keys(["owner", "schedule"])
config.values()  # dict_values(["team_A", "@daily"])
config.items()   # dict_items([("owner", "team_A"), ("schedule", "@daily")])
```

```python
# items() — 키와 값 동시에 순회
for k, v in config.items():
    print(f"{k}: {v}")

# ⚠️ 인덱싱 불가 — list() 변환 필요
config.items()[0]          # TypeError!
list(config.items())[0]    # ("owner", "team_A") ✅
```

```
착각 주의:
  for k, v in config.items() 에서
  k = 인덱스(0,1,2)로 착각하기 쉬움
  k = Key 이름 / v = Value 값
```

---

---
# ⑤ 정렬 — sorted() + items()

## key= 가 하는 일

```
sorted(리스트, key=함수)

key= 는 "각 요소를 어떤 값으로 비교할지" 정하는 것
함수가 반환한 값을 기준으로 정렬
반환값이 숫자 / 문자 / 튜플 모두 가능

x[0], x[1] 같은 인덱싱만 되는 게 아님
abs() / len() / 직접 계산식 전부 가능
```


```python
# key 없으면 요소 자체 비교
sorted([3, 1, 2])                              # [1, 2, 3]

# key 있으면 함수 반환값 비교
sorted(["banana", "apple", "cherry"], key=len) # ["apple", "banana", "cherry"]
# len 값 → 5, 6, 6  → 이 값으로 비교
```

## 값 기준 정렬


```python
scores = {"Alice": 90, "Bob": 70, "Carol": 85}

# 값(점수) 기준 오름차순
sorted(scores.items(), key=lambda x: x[1])
# [("Bob", 70), ("Carol", 85), ("Alice", 90)]

# 값 기준 내림차순
sorted(scores.items(), key=lambda x: x[1], reverse=True)
# [("Alice", 90), ("Carol", 85), ("Bob", 70)]
```

## 키 기준 정렬


```python
scores = {"Carol": 85, "Alice": 90, "Bob": 70}

# 키 기준 오름차순
sorted(scores.items(), key=lambda x: x[0])
sorted(scores.items())                      # 더 간단 (기본이 키 기준)
# [("Alice", 90), ("Bob", 70), ("Carol", 85)]

# 키 기준 내림차순
sorted(scores.items(), reverse=True)
# [("Carol", 85), ("Bob", 70), ("Alice", 90)]
```

## 다중 정렬 — 튜플 키


```python
answer = {"a": 3, "b": 1, "c": 3, "d": 1}

# 값 오름차순 → 값 같으면 키 오름차순
sorted(answer.items(), key=lambda x: (x[1], x[0]))
# [("b", 1), ("d", 1), ("a", 3), ("c", 3)]

# 숫자 키 — 값 오름차순 + 키 내림차순 (-x[0])
answer = {1: 3, 2: 1, 3: 3, 4: 1}
sorted(answer.items(), key=lambda x: (x[1], -x[0]))
# [(4, 1), (2, 1), (3, 3), (1, 3)]

# 키만 추출
[k for k, v in sorted(answer.items(), key=lambda x: (x[1], -x[0]))]
# [4, 2, 3, 1]
```

```
lambda x: (x[1], -x[0]) 읽는 법:
  x       → (키, 값) 튜플
  x[0]    → 키
  x[1]    → 값
  튜플 정렬 → 첫 번째 같으면 두 번째로 비교
  -x[0]   → 숫자 키일 때만 가능 (문자열은 reverse 사용)
```

## key= 에 계산식 넣기 ⭐️

```
딕셔너리 없이 리스트를 직접 정렬할 때
key= 에 abs() 같은 계산식을 바로 넣을 수 있음
```


```python
numlist = [1, 2, 3, 4, 5, 6]
n = 4

# 방법 1: 딕셔너리 만들고 정렬 (내 코드)
answer = {a: abs(n-a) for a in numlist}   # 거리값 딕셔너리
arr = sorted(answer.items(), key=lambda x: (x[1], -x[0]))
# x = (원소, 거리값) 튜플 → x[1]=거리, x[0]=원소

# 방법 2: 리스트 직접 정렬 (더 간결)
sorted(numlist, key=lambda x: (abs(x-n), n-x))
# x = numlist 의 원소 그 자체
# abs(x-n) = n 과의 거리 (첫 번째 기준)
# n-x      = 거리 같을 때 작은 수 우선 (두 번째 기준)
```

```
n-x 가 두 번째 기준인 이유:
  n=4, x=2 → n-x = 2  (양수)
  n=4, x=6 → n-x = -2 (음수)
  양수 < 음수 → 2 가 6 보다 먼저
  → "거리 같으면 작은 수 먼저"

key= 핵심:
  x[0], x[1] 인덱싱만 가능한 게 아님
  abs(x-n) / len(x) / x*2 등 어떤 계산식도 가능
  "이 요소를 어떤 값으로 바꿔서 비교할지" 만 정하면 됨
```

## 정렬 후 딕셔너리로 변환


```python
sorted_dict = dict(sorted(scores.items(), key=lambda x: x[1], reverse=True))
# {"Alice": 90, "Carol": 85, "Bob": 70}
```

---

---

# ⑥ fromkeys() — 일괄 초기화

```python
keys = ["name", "age", "email"]

# 기본값 없이 → None
dict.fromkeys(keys)
# {"name": None, "age": None, "email": None}

# 기본값 지정 (단순 값만)
dict.fromkeys(keys, 0)
# {"name": 0, "age": 0, "email": 0}
```

```python
# ❌ 리스트를 기본값으로 → 대참사
d = dict.fromkeys(["a", "b"], [])
d["a"].append(1)
print(d)   # {"a": [1], "b": [1]}  ← b 도 같이 바뀜! (같은 객체 공유)

# ✅ 키마다 다른 값 → Dict Comprehension
d = {k: [] for k in ["a", "b"]}
d["a"].append(1)
print(d)   # {"a": [1], "b": []}  ← 각자 독립 ✅
```

```
fromkeys 적합:  모든 키에 같은 단순 값 (0, None, "")
Dict Comp 적합: 키마다 다른 값 / 계산식 / 리스트 등 가변 객체
```

## 순서 유지 중복 제거

```python
items = [3, 1, 2, 1, 3, 4]
list(dict.fromkeys(items))
# [3, 1, 2, 4]  ← 첫 등장 순서 유지

"".join(dict.fromkeys("banana"))
# "ban"

# set → 순서 보장 X
set([3, 1, 2, 1, 3, 4])  # {1, 2, 3, 4}  순서 제각각 ❌
```

---

---

# ⑦ Dict Comprehension — 키마다 다른 값

```
{key: value_expression for key in iterable}
```

## 기본 패턴

```python
# 리스트 → 딕셔너리
{k: 0 for k in ["a", "b", "c"]}
# {"a": 0, "b": 0, "c": 0}

# 값에 계산식
{n: n ** 2 for n in [1, 2, 3, 4, 5]}
# {1: 1, 2: 4, 3: 9, 4: 16, 5: 25}
```

## 조건 포함

```python
{n: n ** 2 for n in range(1, 6) if n % 2 == 0}
# {2: 4, 4: 16}
```

## 기존 딕셔너리 변환

```python
scores = {"Alice": 90, "Bob": 70, "Carol": 85}

# 모든 값 +10
{k: v + 10 for k, v in scores.items()}
# {"Alice": 100, "Bob": 80, "Carol": 95}

# 70점 이상만 필터링
{k: v for k, v in scores.items() if v >= 70}
# {"Alice": 90, "Bob": 70, "Carol": 85}

# 키 / 값 뒤집기
{v: k for k, v in scores.items()}
# {90: "Alice", 70: "Bob", 85: "Carol"}
```

## fromkeys vs Dict Comprehension

```python
numlist = [1, 2, 3]

# fromkeys — 모든 키에 같은 값
dict.fromkeys(numlist, 0)       # {1: 0, 2: 0, 3: 0}

# ❌ fromkeys 에 계산식 → 안 됨
# dict.fromkeys(numlist, n * 2) → NameError

# ✅ Dict Comprehension — 키마다 계산
{n: n * 2 for n in numlist}     # {1: 2, 2: 4, 3: 6}
{n: [] for n in numlist}        # {1: [], 2: [], 3: []}  각자 독립
```

---

---

# ⑧ update() — 딕셔너리 합치기

```python
a = {"x": 1, "y": 2}
b = {"y": 99, "z": 3}

# update: b 로 a 를 덮어씀 (중복 키는 b 값 우선)
a.update(b)
print(a)   # {"x": 1, "y": 99, "z": 3}

# 새 딕셔너리 생성 (원본 유지)
merged = {**a, **b}   # b 값이 a 덮어씀
```

---

---
# ⑨ 실전 패턴 — 등수 계산

```
리스트 원본 순서는 유지하면서
각 원소에 등수를 부여해야 할 때

방법 1: 딕셔너리로 등수 매핑
방법 2: .index() 로 등수 계산
```

## 방법 1 — 딕셔너리로 등수 매핑


```python
rank = [4, 1, 2, 1, 3]   # 원본 순서 유지해야 함

# ① 정렬된 리스트에서 등수 딕셔너리 생성
rank_sorted = sorted(set(rank))   # 중복 제거 + 정렬: [1, 2, 3, 4]

rank_dict = {}
for i, r in enumerate(rank_sorted):
    rank_dict[r] = i + 1          # {1: 1, 2: 2, 3: 3, 4: 4}

# ② 원본 리스트에 딕셔너리로 등수 적용
result = [rank_dict[r] for r in rank]
# [4, 1, 2, 1, 3]  ← 원본 순서 유지
```

```python
# 내림차순 등수 (점수 높을수록 1등)
scores = [90, 70, 85, 70, 95]

rank_sorted = sorted(set(scores), reverse=True)  # [95, 90, 85, 70]
rank_dict = {r: i + 1 for i, r in enumerate(rank_sorted)}
# {95: 1, 90: 2, 85: 3, 70: 4}

result = [rank_dict[s] for s in scores]
# [2, 4, 3, 4, 1]  ← 원본 순서 유지, 동점 같은 등수
```

```
흐름:
  ① 정렬 + 중복 제거 → 등수 기준 만들기
  ② {값: 등수} 딕셔너리 생성
  ③ 원본 리스트 순서대로 딕셔너리에서 등수 꺼내기
```

```python
# enumerate + if 패턴 (중복 건너뛰기)
rank_dict = {}
for i, r in enumerate(sorted(set(rank))):
    if r not in rank_dict:          # 중복값은 한 번만 저장
        rank_dict[r] = i + 1

# set 으로 이미 중복 제거했으므로 if 체크 생략 가능
# 하지만 set 쓰지 않고 직접 순회할 때는 if 체크 필요
```

## 방법 2 — .index() 로 등수 계산

>[[Python_Lists_Tuples#핵심 패턴 — 정렬 후 index 로 순위 찾기]] 참고 

```python
scores = [90, 70, 85, 70, 95]

arr_sort = sorted(set(scores), reverse=True)  # [95, 90, 85, 70]

result = [arr_sort.index(v) + 1 for v in scores]
# [2, 4, 3, 4, 1]
```

```
arr_sort.index(v):
  정렬된 리스트에서 v 의 위치(0부터) 반환
  + 1 → 등수 (1부터)

  arr_sort = [95, 90, 85, 70]
  90 의 index = 1 → 1 + 1 = 2등
  70 의 index = 3 → 3 + 1 = 4등
```

## 두 방법 비교

```
딕셔너리 방법:
  등수를 한 번만 계산 → 딕셔너리에 저장 → 꺼내서 사용
  데이터 많을 때 빠름 (딕셔너리 조회 = O(1))

.index() 방법:
  코드가 더 짧음
  데이터 많을 때 느릴 수 있음 (.index 는 O(n) 선형 탐색)
  소량 데이터에서 간결하게 사용
```

---
---

# 자주 하는 실수

```python
# ① 없는 키 대괄호 조회 → KeyError
user["없는키"]             # ❌
user.get("없는키", "기본") # ✅

# ② in 이 Value 검사라고 착각
"airflow" in user          # False  (Key 검사)
"airflow" in user.values() # True   (Value 검사)

# ③ fromkeys 에 리스트 → 공유 참조
dict.fromkeys(keys, [])    # ❌ 공유됨
{k: [] for k in keys}      # ✅ 각자 독립

# ④ dict_items 직접 인덱싱
config.items()[0]          # ❌ TypeError
list(config.items())[0]    # ✅

# ⑤ 정렬에서 x[0] x[1] 혼동
sorted(d.items(), key=lambda x: x[1])
# x = (키, 값) 튜플 → x[0]=키 / x[1]=값
```

---

---

# 한눈에 정리

```
조회:    d.get("key", 기본값)           안전 조회
순회:    for k, v in d.items()
정렬:    sorted(d.items(), key=lambda x: x[1])
         sorted(d.items(), key=lambda x: (x[1], -x[0]))  다중 정렬

초기화:  dict.fromkeys(keys, 0)          모든 키 같은 값
         {k: 계산식 for k in keys}       키마다 다른 값 ✅

변환:    {k: v+10 for k, v in d.items()}
         {k: v for k, v in d.items() if 조건}
         {v: k for k, v in d.items()}   키/값 뒤집기

합치기:  {**a, **b}  또는  a.update(b)
중복제거: list(dict.fromkeys(items))     순서 유지
```