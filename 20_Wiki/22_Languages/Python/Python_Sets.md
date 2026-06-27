---
aliases:
  - Set
  - 셋
  - 집합
  - 중복제거
  - len(set)
tags:
  - Python
related:
  - "[[Python_Lists_Tuples]]"
  - "[[00_Python_HomePage]]"
---
# Python Sets — 중복을 허용하지 않는 주머니

## 한 줄 요약

> **순서 없음, 중복 없음. 중복 제거 + 집합 연산(교/차/합집합) 에 특화된 자료형.**

---

---

# ① 핵심 특징

```
1. 중복 없음   같은 값을 백 번 넣어도 하나만 남음
2. 순서 없음   인덱싱([0]) 불가
3. 변경 가능   add / remove 로 값 추가·삭제 가능
4. 해시 기반   검색 속도가 리스트보다 빠름 (O(1) vs O(n))
```

```python
s = {1, 2, 2, 3, 3, 3}
print(s)   # {1, 2, 3} — 중복 자동 제거
print(type(s))  # <class 'set'>
```

---

---

# ② 생성 방법

```python
# 방법 1 — 중괄호 리터럴
s = {1, 2, 3}

# 방법 2 — set() 생성자 (리스트 → set 변환)
s = set([1, 2, 2, 3])   # {1, 2, 3}

# 방법 3 — 빈 set 생성 ⭐️
s = set()   # 반드시 set() 으로 만들어야 함
# ❌ s = {}  → 이건 빈 dict (딕셔너리) 임!

# 문자열도 변환 가능
s = set("hello")   # {'h', 'e', 'l', 'o'} — 중복 l 제거됨
```

> `{}` 는 빈 딕셔너리. 빈 집합은 반드시 `set()` 으로 만들 것.

---

---

# ③ 값 추가 / 삭제

## add() — 값 하나 추가

```python
answer = set()   # 빈 set 생성
answer.add(1)
answer.add(2)
answer.add(2)    # 중복 → 무시됨
print(answer)    # {1, 2}
```

```
알고리즘 문제에서 자주 쓰는 패턴:
  answer = set()
  for x in data:
      if 조건:
          answer.add(x)   # 중복 걱정 없이 그냥 넣으면 됨
```

## update() — 여러 값 한 번에 추가

```python
s = {1, 2}
s.update([3, 4, 4])   # 리스트, set, 튜플 모두 가능
print(s)              # {1, 2, 3, 4}
```

## remove() vs discard() — 값 삭제

```python
s = {1, 2, 3}

s.remove(2)    # 2 삭제
print(s)       # {1, 3}

s.remove(99)   # ❌ KeyError — 없는 값이면 에러 발생

s.discard(99)  # ✅ 없어도 에러 없이 그냥 넘어감
```

```
remove   → 확실히 있는 값 삭제할 때
discard  → 있는지 없는지 모를 때 (안전한 삭제)
```

## pop() — 임의의 값 하나 꺼내고 삭제

```python
s = {1, 2, 3}
picked = s.pop()   # 뭐가 나올지 모름 (순서 없으니까)
print(picked)      # 1, 2, 3 중 하나
print(s)           # 나머지 2개 남음
```

```
로또 추첨처럼 "통에서 공 하나 꺼내서 없애버릴 때" 사용
원본이 변경되므로 주의
```

## clear() — 전체 비우기

```python
s = {1, 2, 3}
s.clear()
print(s)   # set()
```

---

---

# ④ 집합 연산 ⭐️

## 연산자 방식

```python
A = {1, 2, 3}
B = {2, 3, 4}

A & B    # {2, 3}      교집합  — 둘 다 있는 것
A | B    # {1, 2, 3, 4} 합집합 — 전부 합치기 (중복 제거)
A - B    # {1}         차집합  — A 에만 있는 것
A ^ B    # {1, 4}      대칭차  — 어느 한쪽에만 있는 것
```

## 메서드 방식 (의미가 더 명확)

```python
A.intersection(B)        # A & B  교집합
A.union(B)               # A | B  합집합
A.difference(B)          # A - B  차집합
A.symmetric_difference(B) # A ^ B 대칭차
```

## ⭐️ 리스트 · 문자열에도 집합 연산 가능 — set() 변환

```
집합 연산자는 set 타입에만 동작
리스트, 문자열은 바로 & 쓰면 에러

→ set() 으로 감싸면 어떤 iterable 이든 집합 연산 가능
```

```python
# 리스트끼리 공통 원소 찾기
s1 = [1, 2, 3, 4]
s2 = [3, 4, 5, 6]

set(s1) & set(s2)          # {3, 4}
len(set(s1) & set(s2))     # 2  ← 공통 원소 개수

# 코딩테스트 패턴 — 한 줄로
def solution(s1, s2):
    return len(set(s1) & set(s2))

# 문자열도 동일 (문자 단위로 분해됨)
set("hello") & set("world")   # {'l', 'o'}
set("apple") | set("grape")   # {'a', 'p', 'l', 'e', 'g', 'r'}

# 두 리스트의 차집합 (s1 에만 있는 것)
set(s1) - set(s2)              # {1, 2}

# 두 리스트 합치되 중복 제거
list(set(s1) | set(s2))        # [1, 2, 3, 4, 5, 6] (순서 보장 안 됨)
sorted(set(s1) | set(s2))      # [1, 2, 3, 4, 5, 6] (정렬까지)
```

```
set() 변환 흐름:
  리스트 → set() → 집합 연산 → 결과(set) → 필요하면 list() 로 다시 변환

  list → set(list) → & | - ^ → set → list(결과) or len(결과)

⚠️ set 은 순서가 없음
   결과를 리스트로 다시 쓸 때 순서가 필요하면 sorted() 사용
```

## 데이터 엔지니어링 활용 예시

```python
어제_운행_열차 = {"KTX001", "KTX002", "ITX003"}
오늘_운행_열차 = {"KTX002", "ITX003", "SRT004"}

# 오늘 신규 투입 열차 (오늘에만 있는 것)
신규 = 오늘_운행_열차 - 어제_운행_열차   # {"SRT004"}

# 어제 운행했는데 오늘 안 한 열차 (결항/취소)
결항 = 어제_운행_열차 - 오늘_운행_열차   # {"KTX001"}

# 이틀 모두 운행한 열차
공통 = 어제_운행_열차 & 오늘_운행_열차   # {"KTX002", "ITX003"}

# DB 컬럼 비교 — 두 테이블에 공통으로 있는 컬럼 찾기
table_a_cols = ["id", "name", "age", "city"]
table_b_cols = ["id", "name", "score"]

set(table_a_cols) & set(table_b_cols)   # {'id', 'name'}
set(table_a_cols) - set(table_b_cols)   # {'age', 'city'}  A 에만 있는 컬럼
```

---

---

# ⑤ 포함 여부 확인

```python
s = {1, 2, 3}

# in 연산자 (리스트보다 훨씬 빠름)
print(1 in s)    # True
print(99 in s)   # False
print(99 not in s)  # True
```

```
리스트는 처음부터 끝까지 하나씩 확인 → O(n)
set 은 해시로 바로 찾음              → O(1)

데이터가 많을수록 set 이 압도적으로 빠름
"이 값이 있나?" 를 자주 체크한다면 리스트 대신 set 사용 권장
```

---

---

# ⑥ 값 꺼내기

```python
s = {10, 20, 30}
```

## 방법 1 — 반복문 (가장 일반적)

```python
for v in s:
    print(v)   # 순서는 보장 안 됨
```

## 방법 2 — list() 변환 후 인덱싱 (권장)

```python
lst = list(s)
print(lst[0])   # 안전하게 인덱스로 접근 가능
                # 단, 어떤 값이 [0] 에 올지는 모름
```

## 방법 3 — next(iter()) 로 하나만 꺼내기

```python
first = next(iter(s))   # 하나만 꺼냄 (원본 유지)
# iter(s)  → 줄 세우는 기계
# next()   → 맨 앞 녀석 꺼내기
```

## 방법 4 — 언패킹 (개수가 확실할 때)

```python
s = {1, 2}
a, b = s    # 개수가 딱 맞아야 함
            # 누가 a 가 될지는 모름 (순서 없으니까)
```

|방법|코드|원본 변경|추천|
|---|---|:-:|:-:|
|반복문|`for v in s`|❌|✅ 일반적|
|리스트 변환|`list(s)[0]`|❌|✅ 인덱스 필요할 때|
|next(iter)|`next(iter(s))`|❌|하나만 꺼낼 때|
|pop|`s.pop()`|✅ 삭제됨|⚠️ 원본 파괴 주의|

---

---

# ⑦ len 활용 — 조건문 단순화 ⭐️

복잡한 비교 연산자 범벅을 `len(set())` 하나로 해결하는 패턴.

```python
# ❌ Before — 조건문 지옥
if a != b and a != c and b != c:
    ...
elif a == b and b == c:
    ...
elif a != b == c or a == b != c or a == c != b:
    ...

# ✅ After — len(set()) 으로 단순화
unique_count = len(set([a, b, c]))

if unique_count == 3:    # 모두 다름
    ...
elif unique_count == 2:  # 두 개만 같음
    ...
else:                    # 모두 같음
    ...
```

```
변수가 4개로 늘어나도?
  Before: 조건문 수십 줄
  After:  len(set([a, b, c, d])) 로직 그대로 유지
```

---

---

# ⑧ frozenset — 변경 불가능한 set

```python
fs = frozenset([1, 2, 3])
fs.add(4)     # ❌ AttributeError — 변경 불가

# 언제 쓰나:
# 딕셔너리의 키(key) 로 set 을 쓰고 싶을 때
d = {frozenset([1, 2]): "값"}   # ✅
d = {{1, 2}: "값"}               # ❌ set 은 해시 불가 (unhashable)
```

---

---

# ⑨ 중복 제거 패턴

```python
# 리스트 중복 제거
data = [1, 2, 2, 3, 3, 3]
unique = list(set(data))    # [1, 2, 3] (순서 보장 안 됨)

# 순서 유지하면서 중복 제거 (Python 3.7+)
unique_ordered = list(dict.fromkeys(data))   # [1, 2, 3] 순서 유지
```

---

---

# 치트시트

|작업|코드|
|---|---|
|빈 set 생성|`s = set()`|
|값 추가|`s.add(x)`|
|여러 값 추가|`s.update([1, 2, 3])`|
|값 삭제 (에러 O)|`s.remove(x)`|
|값 삭제 (에러 X)|`s.discard(x)`|
|임의 값 꺼내고 삭제|`s.pop()`|
|전체 비우기|`s.clear()`|
|포함 확인|`x in s`|
|개수|`len(s)`|
|교집합|`A & B` / `A.intersection(B)`|
|합집합|`A \| B` / `A.union(B)`|
|차집합|`A - B` / `A.difference(B)`|
|대칭차|`A ^ B` / `A.symmetric_difference(B)`|
|중복 제거|`list(set(data))`|
|조건 단순화|`len(set([a, b, c]))`|