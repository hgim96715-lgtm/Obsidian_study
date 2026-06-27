---
aliases:
  - range
  - enumerate
  - zip
  - 반복문도구
  - 파이썬반복
tags:
  - Python
related:
  - "[[Python_Lists_Tuples]]"
  - "[[Python_Control_Flow]]"
  - "[[Spark_Session_Context]]"
  - "[[00_Python_HomePage]]"
---
# Python_Looping_Helpers — range / enumerate / zip

## 한 줄 요약

```
반복문(for) 을 더 똑똑하고 편하게 만드는 3가지 도구

range()      숫자 범위 생성
enumerate()  인덱스 + 값 동시에
zip()        두 리스트를 같은 인덱스끼리 묶기
```

---

---

# ① range() — 숫자 범위 생성

```python
range(끝)               # 0 ~ 끝-1
range(시작, 끝)          # 시작 ~ 끝-1
range(시작, 끝, 보폭)    # 시작 ~ 끝-1, 보폭씩 증가

# 끝 숫자는 포함 안 함!
```

```python
list(range(5))           # [0, 1, 2, 3, 4]
list(range(1, 6))        # [1, 2, 3, 4, 5]
list(range(0, 10, 2))    # [0, 2, 4, 6, 8]
list(range(10, 0, -1))   # [10, 9, 8, ... 1]  역순
```

## range 는 게으른 녀석 (Lazy)

```
range(999999999999)  → 메모리 거의 안 먹음 (숫자를 미리 안 만듦)
list(range(999999999999)) → 💥 메모리 폭발

for i in range(999999999999):  → 안전 (하나씩 꺼냄)
```

## step 으로 if 문 제거

```python
# ❌ if 로 홀수 거름
total = 0
for i in range(n + 1):
    if i % 2 == 1:
        total += i

# ✅ step=2 로 처음부터 홀수만
total = sum(range(1, n + 1, 2))
```

---

---

# ② enumerate() — 인덱스 + 값 동시에

```python
names = ["Spark", "Airflow", "Docker"]

# ❌ 카운터 변수 직접 관리
i = 0
for name in names:
    print(i, name)
    i += 1

# ✅ enumerate
for i, name in enumerate(names):
    print(f"{i}번째: {name}")
# 0번째: Spark
# 1번째: Airflow
# 2번째: Docker

# 1부터 시작
for i, name in enumerate(names, 1):
    print(f"{i}번째: {name}")
# 1번째: Spark ...
```

---

---

# ③ zip() — 두 리스트 병렬 순회

```python
names  = ["Ironman", "Hulk"]
powers = [100, 999]

# ❌ 인덱스 직접 관리
for i in range(len(names)):
    print(names[i], powers[i])

# ✅ zip
for n, p in zip(names, powers):
    print(f"{n}의 파워: {p}")
```

## zip 결과 확인 — list() / dict() 필요

```python
names  = ['A', 'B']
scores = [10, 20]

zip(names, scores)           # <zip object> — 내용 안 보임

list(zip(names, scores))     # [('A', 10), ('B', 20)]
dict(zip(names, scores))     # {'A': 10, 'B': 20}
```

## 길이 다르면 짧은 쪽에 맞춰 잘림

```python
list(zip([1, 2, 3], ['a', 'b']))  # [(1, 'a'), (2, 'b')]  ← 3은 버려짐
```

## zip + dict — 조건문을 데이터로 치환 ⭐️

```python
# ❌ if-elif 지옥
def solution(n, control):
    for c in control:
        if c == 'w':   n += 1
        elif c == 's': n -= 1
        elif c == 'd': n += 10
        elif c == 'a': n -= 10
    return n

# ✅ zip + dict 매핑 테이블
def solution(n, control):
    key = dict(zip(['w', 's', 'd', 'a'], [1, -1, 10, -10]))
    # {'w': 1, 's': -1, 'd': 10, 'a': -10}
    return n + sum(key[c] for c in control)
```

```
장점:
  기능 추가 시 리스트에 데이터만 추가
  if-elif 를 뜯어고칠 필요 없음 (Data-Driven)
  sum + 제너레이터로 한 줄 처리
```

## zip 중첩 — 행렬 연산 ⭐️

```
zip 을 이중으로 써서 행렬의 같은 위치 값 처리
[[a,b],[c,d]] + [[e,f],[g,h]] → 같은 위치끼리 더하기
```


```python
# 기본 풀이 — 이중 for 루프
def solution(arr1, arr2):
    answer = []
    for i in range(len(arr1)):
        row = []
        for j in range(len(arr1[0])):
            row.append(arr1[i][j] + arr2[i][j])
        answer.append(row)
    return answer

# Pythonic 풀이 — zip 중첩 + 리스트 컴프리헨션 ⭐️
def solution(arr1, arr2):
    return [[c + d for c, d in zip(a, b)] for a, b in zip(arr1, arr2)]
```

```
동작 원리:
  zip(arr1, arr2)      → 같은 행끼리 묶음
    [(행1A, 행1B), (행2A, 행2B), ...]

  zip(a, b)            → 같은 열끼리 묶음 (행 안에서)
    [(a[0], b[0]), (a[1], b[1]), ...]

  c + d                → 같은 위치 값 더하기

단계별로 보면:
  A = [[1, 2], [3, 4]]
  B = [[5, 6], [7, 8]]

  zip(A, B) → ([1,2],[5,6]), ([3,4],[7,8])   ← 행 단위 묶음
  zip(a, b) → (1,5), (2,6)                   ← 열 단위 묶음
  c + d     → 6, 8                            ← 더하기
  결과:  [[6, 8], [10, 12]]
```

```
행렬 문제 패턴:
  "같은 위치" 처리 → 인덱스 또는 zip
  이중 반복문 → zip 중첩으로 압축 가능

  기본 풀이: range(len) 이중 반복
  Pythonic: zip 중첩 + 리스트 컴프리헨션
```

---

---

# ④ continue 의 함정 — 루프 끝 처리

```
continue 는 결론을 내리지 않고 다음 바퀴로 넘김
모든 바퀴가 continue 로만 끝나면 → 함수가 None 반환

→ 루프 바깥에 반드시 최종 결론 작성
```

```python
# zip + continue — 날짜 비교
def solution(date1, date2):
    for a, b in zip(date1, date2):
        if a < b:
            return 1    # date1 이 더 빠름
        elif a > b:
            return 0    # date1 이 더 느림
        else:
            continue    # 같으면 다음 단계(월/일) 비교

    # ⚠️ 이 줄 없으면 완전히 같은 날짜일 때 None 반환
    return 0            # 끝까지 다 돌았는데 승부 안 났으면 = 같은 날짜
```

---

---

# ⑤ 초기화 위치 — 밖 vs 안

```
변수를 for 루프 밖에서 초기화 → 누적 (이전 바퀴 결과 유지)
변수를 for 루프 안에서 초기화 → 리셋 (바퀴마다 새로 시작)
```

## 밖 — 누적할 때

```python
# 전체에서 조건에 맞는 것 모으기
c_arr = [1, 5, 10, 20]
res = []              # ← 루프 밖 → 끝까지 유지

for x in c_arr:
    if x > 3:
        res.append(x)

print(res)   # [5, 10, 20]
```

## 안 — 바퀴마다 리셋할 때

```python
# 각 단어를 문자 단위로 분해
words = ["Hi", "Bye"]

for w in words:
    char_list = []    # ← 루프 안 → 단어 바뀔 때마다 리셋

    for c in w:
        char_list.append(c)

    print(f"{w}: {char_list}")
# Hi: ['H', 'i']
# Bye: ['B', 'y', 'e']   ← 앞 단어 안 섞임
```

## 판단 기준

|질문|위치|예시|
|---|---|---|
|이전 바퀴 결과가 다음 바퀴에 영향?|밖|합계, 전체 목록|
|각 바퀴가 독립적?|안|행별 계산, 단어별 분해|