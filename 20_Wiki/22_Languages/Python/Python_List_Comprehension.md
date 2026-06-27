---
aliases:
  - 파이썬 필터링
  - List Comprehension,
  - 리스트 컴프리헨션
tags:
  - Python
related:
  - "[[Python_Lists_Tuples]]"
  - "[[00_Python_HomePage]]"
  - "[[Python_Lambda_Map]]"
---
# 코드를 한 줄로 압축하는 마법 (List Comprehension)

## Concept Summary

리스트 컴프리헨션(List Comprehension)은 기존의 리스트나 반복 가능한 데이터에서 특정 조건에 맞는 데이터만 뽑아내거나 조작하여, **새로운 리스트를 단 한 줄의 코드로 뚝딱 만들어내는 파이썬 특유의 우아한 문법**이다.

---
## Why (왜 필요한가)?

**문제점** 
빈 리스트를 만들고(`result = []`), `for`문을 돌리고, `if`문으로 조건을 걸고, `.append()`로 하나씩 집어넣는 고전적인 방식은 코드를 불필요하게 길고 뚱뚱하게 만든다.

**해결책**
리스트 컴프리헨션을 쓰면 3~4줄의 코드가 단 1줄로 줄어든다. 게다가 파이썬 엔진 내부(C언어 레벨)에서 최적화되어 작동하기 때문에, 일반적인 `for`문 + `append()` 조합보다 실행 속도도 훨씬 빠르다!

---
## 실무 맥락

실무 데이터 엔지니어링 환경에서 숨 쉬듯이 쓴다. DB에서 긁어온 API JSON 응답(딕셔너리 리스트)에서 특정 컬럼(예: `user_id`)만 쏙 뽑아내어 리스트로 만들 때(`[row['user_id'] for row in api_data]`)나, 지저분한 텍스트 데이터 리스트를 한 바퀴 돌면서 공백을 제거할 때(`[text.strip() for text in raw_logs]`) 무조건 이 문법을 사용한다.

---
## 핵심 로직 & 순서 완전 정복

**기본 구조**

```python
[ 결과물  for 변수 in 바구니  if 조건 ]
  "뭘 뱉냐" "어디서 꺼내냐"  "필터"
```

>**영어로 읽어라:** "Give me `n**2` → for each `n` → in `numbers` → only if `n % 2 == 0`" 한국어 어순이랑 달라서 헷갈리는 거다. 
>영어 말하는 순서 그대로다.


---
## Code Core Points (기본 → 조건 필터 → 중첩)

**"반복문 돌면서 새로운 리스트를 뚝딱 만들기"**


```python
numbers = [1, 2, 3, 4, 5]

# 1. [고전적인 방식] 초보자의 코드 (4줄)
squared = []
for n in numbers:
    squared.append(n ** 2)


# 2. 💡 [1단계: 기본] 고인물의 한 줄 코드 (1줄)
# 해석: numbers에서 n을 하나씩 꺼내서, n**2를 한 다음 새 리스트에 담아라!
squared_comp = [n ** 2 for n in numbers]


# 3. 💡 [2단계: 조건 필터] if 추가 (짝수만 제곱해라!)
even_squared = [n ** 2 for n in numbers if n % 2 == 0]


# 4. 💡 [3단계: 값 변경] if~else 추가 (양수는 그대로, 음수는 0으로!)
raw_data = [10, -5, 3, -1]
clean_data = [n if n > 0 else 0 for n in raw_data]
```

---
##  가장 많이 틀리는 순서 실수 (if 위치)

`if`만 쓸 때 vs `if~else` 쓸 때, **위치가 완전히 다르다.**

```python
numbers = [1, 2, 3, 4, 5]

# ✅ if만 쓸 때 (필터) → if는 뒤에
even = [n for n in numbers if n % 2 == 0]
#                          ^^^^^^^^^^^^ 뒤!

# ✅ if~else 쓸 때 (값 선택) → if~else는 앞에
result = [n if n > 0 else 0 for n in numbers]
#         ^^^^^^^^^^^^^^^^^ 앞!

# ❌ 가장 흔한 오류 패턴
broken = [n for n in numbers if n > 0 else 0]  # SyntaxError!
```

**암기법:**

- `if`만 → **"거를 거면 뒤로"** (필터니까 마지막에 판단)
- `if~else` → **"고를 거면 앞으로"** (결과물 자리니까 맨 앞에서 결정)


---
## 패턴 한눈에 비교

```python
data = [1, -2, 3, -4, 5]

# ① 기본 (전체 변환)
[n * 2 for n in data]
# → [2, -4, 6, -8, 10]

# ② if 필터 (양수만 추출)
[n for n in data if n > 0]
# → [1, 3, 5]

# ③ if~else (양수는 그대로, 음수는 0)
[n if n > 0 else 0 for n in data]
# → [1, 0, 3, 0, 5]

# ④ 변환 + 필터 동시에 (양수만 뽑아서 제곱)
[n ** 2 for n in data if n > 0]
# → [1, 9, 25]

# ⑤ 중첩 (2D 리스트 펼치기)
matrix = [[1, 2], [3, 4], [5, 6]]
[num for row in matrix for num in row]
# → [1, 2, 3, 4, 5, 6]
# 읽는 순서: 바깥 for → 안쪽 for → 결과물 (왼→오로 그냥 읽으면 됨)
```

---
## lambda랑 같이 쓰는 법(헷갈려하는 포인트)

lambda는 **함수 자체**고, 컴프리헨션은 **반복 문법**이다. 섞어 쓰려면 역할 분리를 명확히 해야 한다.

```python
numbers = [1, 2, 3, 4, 5]

# ❌ 이렇게 쓰려다 막히는 패턴
result = [lambda x: x**2 for n in numbers]
# → 숫자가 아니라 "함수 객체" 리스트가 만들어진다! 호출을 안 했으니까.


# ✅ 올바른 패턴 1: lambda를 변수에 담고 호출해라
square = lambda x: x ** 2
result = [square(n) for n in numbers]   # 함수를 호출(괄호!)해야 값이 나온다
# → [1, 4, 9, 16, 25]


# ✅ 올바른 패턴 2: 즉시 호출 (가독성이 나빠져서 실무에서 잘 안 씀)
result = [(lambda x: x ** 2)(n) for n in numbers]
#          ^^^^^^^^^^^^^^^^^ ^
#          lambda 정의        호출


# ✅ 올바른 패턴 3: map + lambda 조합 (이게 lambda의 진짜 짝꿍)
result = list(map(lambda x: x ** 2, numbers))
# map이랑 쓸 때는 자연스럽다. 컴프리헨션이랑은 굳이 섞을 필요 없다!
```

>**결론:** lambda를 컴프리헨션 안에 직접 넣고 싶으면 반드시 `(lambda x: ...)(변수)` 형태로 즉시 호출해야 한다. 
>근데 솔직히 복잡한 로직은 `def` 함수로 빼고, `lambda`는 `map()`이랑 쓰는 게 맞다.
>📎 더 자세한 내용은 [[Python_Lambda_Map]] 참고

---
## 언제 lambda 쓰고 언제 컴프리헨션 쓸지 기준

| 상황                               | 추천                |
| -------------------------------- | ----------------- |
| 단순 변환/필터                         | 리스트 컴프리헨션         |
| 함수를 인자로 넘길 때 (sort, map, filter) | lambda            |
| 로직이 복잡해질 때                       | def 함수 정의 후 컴프리헨션 |

```python
# 복잡한 로직은 def로 빼고, 컴프리헨션은 깔끔하게 유지
def clean_text(text):
    return text.strip().lower().replace(" ", "_")

raw = ["  Hello World  ", " Foo Bar "]
result = [clean_text(t) for t in raw]
# → ['hello_world', 'foo_bar']
```

