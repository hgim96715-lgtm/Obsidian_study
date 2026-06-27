---
aliases:
  - Python Counter
  - 컬렉션
  - 빈도수 세기
  - 데이터 등수
  - deque
tags:
  - Python
related:
  - "[[Python_Statistics_Module|통계 모듈 (statistics.mode)]]"
  - "[[Python_Builtin_Functions|기본 내장 함수 (sum, len)]]"
  - "[[Python_Dictionaries|딕셔너리 기초 (key-value)]]"
  - "[[00_Python_HomePage]]"
---
# Python_Collections — collections 모듈

## 한 줄 요약

```
파이썬 내장 모듈 collections
딕셔너리 / 리스트의 한계를 극복한 특화 자료구조 모음
import 없이 못 쓰는 것들
```

```python
from collections import Counter, deque, defaultdict, namedtuple, OrderedDict
```

---

---

# ① Counter — 빈도 계산 ⭐️

```
리스트 / 문자열 요소의 빈도를 딕셔너리 형태로 순식간에 계산
직접 for 문으로 세는 것보다 훨씬 빠르고 코드도 짧음
```

## 기본 사용

```python
from collections import Counter

# 리스트
data = ['apple', 'banana', 'apple', 'orange', 'banana', 'apple']
counts = Counter(data)
print(counts)
# Counter({'apple': 3, 'banana': 2, 'orange': 1})

# 문자열
Counter("abracadabra")
# Counter({'a': 5, 'b': 2, 'r': 2, 'c': 1, 'd': 1})
```

## most_common() — 상위 N개 추출 ⭐️

```python
counts = Counter(['apple', 'banana', 'apple', 'orange', 'banana', 'apple'])

counts.most_common(2)    # [('apple', 3), ('banana', 2)]
counts.most_common()     # 전체 내림차순 리스트
```

## 딕셔너리처럼 사용

```python
counts['apple']          # 3
counts['없는것']         # 0  ← KeyError 없음 (기본값 0)

# 키 / 값 / 항목
counts.keys()            # 항목 목록
counts.values()          # 빈도 목록
counts.items()           # (항목, 빈도) 튜플 목록
```

## 연산

```python
a = Counter(['a', 'b', 'a'])   # Counter({'a': 2, 'b': 1})
b = Counter(['a', 'c'])        # Counter({'a': 1, 'c': 1})

a + b   # Counter({'a': 3, 'b': 1, 'c': 1})  더하기
a - b   # Counter({'a': 1, 'b': 1})           빼기 (0 이하 제거)
a & b   # Counter({'a': 1})                   교집합 (최솟값)
a | b   # Counter({'a': 2, 'b': 1, 'c': 1})  합집합 (최댓값)
```

## Counter vs sorted 비교

```python
# 빈도 순으로 정렬 (코테 단골)
data = ['a', 'b', 'a', 'c', 'b', 'a']

# Counter 방식 (권장)
Counter(data).most_common()
# [('a', 3), ('b', 2), ('c', 1)]

# sorted 방식
freq = {}
for x in data:
    freq[x] = freq.get(x, 0) + 1
sorted(freq.items(), key=lambda x: x[1], reverse=True)
```

```
코딩테스트: Counter  ← 간결
실무/가독성: Counter  ← 의미 명확
```

→ [[Python_Sorting_Logic#⑥ 문자열 정렬로 비교 — 순서 무시 비교 ⭐️]] 참고

---

---

# ② deque — 양방향 큐 ⭐️

```
deque (Double-Ended Queue)
리스트와 달리 앞/뒤 모두 O(1) 으로 삽입/삭제
리스트 pop(0) → O(n) / deque popleft() → O(1)
```

## 기본 사용

```python
from collections import deque

dq = deque([1, 2, 3])

# 뒤에 추가 / 삭제
dq.append(4)       # deque([1, 2, 3, 4])
dq.pop()           # 4 반환 / deque([1, 2, 3])

# 앞에 추가 / 삭제
dq.appendleft(0)   # deque([0, 1, 2, 3])
dq.popleft()       # 0 반환 / deque([1, 2, 3])
```

## rotate() — 회전 ⭐️

```python
dq = deque([1, 2, 3, 4, 5])

dq.rotate(1)    # 오른쪽으로 1칸 회전
# deque([5, 1, 2, 3, 4])  ← 맨 뒤가 맨 앞으로

dq.rotate(-1)   # 왼쪽으로 1칸 회전
# deque([1, 2, 3, 4, 5])  ← 원래대로

dq.rotate(2)    # 오른쪽으로 2칸
# deque([4, 5, 1, 2, 3])

# 문자열도 가능
dq = deque("hello")
dq.rotate(1)
# deque(['o', 'h', 'e', 'l', 'l'])
"".join(dq)     # 'ohell'
```

```
rotate(n):
  n 양수 → 오른쪽으로 n칸 (뒤 → 앞)
  n 음수 → 왼쪽으로 n칸 (앞 → 뒤)

활용:
  암호화 / 패턴 매칭 / 슬라이딩 윈도우
```

## maxlen — 크기 제한

```python
# 최근 N개만 유지
dq = deque(maxlen=3)
dq.append(1)   # deque([1], maxlen=3)
dq.append(2)   # deque([1, 2], maxlen=3)
dq.append(3)   # deque([1, 2, 3], maxlen=3)
dq.append(4)   # deque([2, 3, 4], maxlen=3)  ← 1 자동 삭제

# 최근 로그 N개 유지 / 슬라이딩 윈도우에 활용
```

## list vs deque

|작업|list|deque|
|---|---|---|
|뒤 추가 `append`|O(1)|O(1)|
|뒤 삭제 `pop`|O(1)|O(1)|
|앞 추가 `appendleft`|O(n) ❌|O(1) ✅|
|앞 삭제 `popleft`|O(n) ❌|O(1) ✅|

```
앞/뒤 모두 빈번하게 삽입/삭제 → deque
단순 순회 / 인덱스 접근 많음  → list
```

---

---

# ③ defaultdict — 기본값 딕셔너리

```
없는 키에 접근해도 KeyError 없이 기본값 자동 생성
그룹화 / 빈도 계산에 자주 사용
```

```python
from collections import defaultdict

# 기본값이 list
d = defaultdict(list)
d['a'].append(1)   # 'a' 없어도 자동으로 빈 리스트 생성 후 추가
d['a'].append(2)
print(d)           # defaultdict(<class 'list'>, {'a': [1, 2]})

# 기본값이 int (빈도 계산)
d = defaultdict(int)
for x in ['a', 'b', 'a', 'c', 'a']:
    d[x] += 1      # 없는 키는 자동으로 0 으로 시작
print(d)           # defaultdict(<class 'int'>, {'a': 3, 'b': 1, 'c': 1})
```

```python
# 실전 — 그룹화
words = ["apple", "banana", "avocado", "blueberry", "cherry"]
groups = defaultdict(list)
for w in words:
    groups[w[0]].append(w)   # 첫 글자로 그룹화
# {'a': ['apple', 'avocado'], 'b': ['banana', 'blueberry'], 'c': ['cherry']}
```

```
일반 딕셔너리:
  d['없는키']          → KeyError
  d.get('없는키', [])  → 빈 리스트 반환 (매번 기본값 지정 필요)

defaultdict:
  d['없는키']          → 자동으로 기본값 생성 (기본값 지정 불필요)
```

---

---

# ④ namedtuple — 이름 있는 튜플

```
튜플인데 인덱스 대신 이름으로 접근 가능
딕셔너리보다 메모리 효율적
클래스보다 가볍게 데이터 구조 정의
```

```python
from collections import namedtuple

# 정의
Point = namedtuple('Point', ['x', 'y'])
Hospital = namedtuple('Hospital', ['hpid', 'hpname', 'hvec', 'region'])

# 생성
p = Point(3, 5)
h = Hospital('A001', '서울대병원', 15, '서울')

# 이름으로 접근
p.x         # 3
p.y         # 5
h.hpname    # '서울대병원'
h.hvec      # 15

# 인덱스로도 접근 가능 (튜플이므로)
p[0]        # 3
h[1]        # '서울대병원'

# 언패킹도 가능
x, y = p
hpid, hpname, hvec, region = h
```

```
언제 쓰나:
  함수가 여러 값을 반환할 때
  DB 조회 결과 행을 표현할 때
  좌표 / 설정 등 고정 구조 데이터
```

---

---

# ⑤ OrderedDict — 순서 보장 딕셔너리

```
Python 3.7+ 부터 일반 dict 도 삽입 순서 보장
→ 현재는 OrderedDict 쓸 이유가 거의 없음
레거시 코드에서 볼 수 있으므로 읽을 줄만 알면 됨
```

```python
from collections import OrderedDict

od = OrderedDict()
od['a'] = 1
od['b'] = 2
od['c'] = 3
# OrderedDict([('a', 1), ('b', 2), ('c', 3)])
```

---

---

# 한눈에 정리

|자료구조|용도|핵심 메서드|
|---|---|---|
|`Counter`|빈도 계산|`most_common(n)`|
|`deque`|양방향 큐|`appendleft / popleft / rotate`|
|`defaultdict`|기본값 딕셔너리|자동 기본값 생성|
|`namedtuple`|이름 있는 튜플|`.필드명` 으로 접근|

```python
from collections import Counter, deque, defaultdict, namedtuple

# 빈도 계산
Counter(data).most_common(3)

# 앞/뒤 O(1) 삽입/삭제
dq = deque([1, 2, 3])
dq.appendleft(0) / dq.popleft()
dq.rotate(1)    # 오른쪽 / rotate(-1) 왼쪽

# 기본값 딕셔너리
d = defaultdict(list)
d['key'].append(값)   # 없는 키도 자동으로 빈 리스트 생성

# 이름 있는 튜플
Point = namedtuple('Point', ['x', 'y'])
p = Point(3, 5)
p.x   # 3
```