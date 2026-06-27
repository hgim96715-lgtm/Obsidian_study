---
aliases:
  - Python_Type_Checking
  - isinstance
  - type
  - 자료형_검사
  - repr
  - 값의 진짜 모습 보기
  - typing
tags:
  - Python
related:
  - "[[Python_Classes_Objects]]"
  - "[[00_Python_HomePage]]"
  - "[[Python_Variables_Types]]"
  - "[[Python_Lists_Tuples]]"
  - "[[Python_Dictionaries]]"
  - "[[Python_Functions]]"
---
# Python_Type_Checking — type vs isinstance

## 한 줄 요약

```
"이 변수, 내가 생각하는 그 데이터 타입 맞아?"
데이터의 자료형을 확인하는 내장 함수
```

## 왜 필요한가?

```
외부 API / 파일에서 읽어온 데이터는 타입을 확신할 수 없음
→ 에러 터지기 전에 안전장치(방어적 프로그래밍) 필요

"리스트가 맞을 때만 반복문 돌아라"
"dict 가 맞을 때만 .get() 써라"
```

---

---

# ① isinstance — 타입 검사 (권장)

```python
isinstance(검사할_값, 확인하고_싶은_타입)
# 맞으면 True / 틀리면 False
```

```python
data = [1, 2, 3]

isinstance(data, list)   # True
isinstance(data, dict)   # False
```

## 여러 타입 동시 검사 — 튜플로 묶기

```python
val = 3.14

isinstance(val, (int, float))   # True  ← int 또는 float
isinstance(val, (str, bool))    # False
```

```python
# 실전: API 응답 데이터 방어적 처리
response = fetch_data()

if isinstance(response, list):
    for item in response:        # 리스트일 때만 반복
        process(item)
elif isinstance(response, dict):
    process(response)            # 딕셔너리면 바로 처리
else:
    print(f"예상치 못한 타입: {type(response)}")
```

---

---

# ② type — 타입 확인

```python
type(3)          # <class 'int'>
type(3.14)       # <class 'float'>
type("hello")    # <class 'str'>
type([1, 2])     # <class 'list'>

# 타입 비교
type(3) == int   # True
```

---

---

# ③ type vs isinstance — 핵심 차이

```
type()       깐깐함 → "정확히 그 클래스" 여야만 True
isinstance() 유연함 → "그 클래스의 자식(상속)" 도 True
```

```python
class Animal:
    pass

class Dog(Animal):   # Animal 을 상속받은 Dog
    pass

baduk = Dog()

# type() — 상속 무시
type(baduk) == Dog      # True
type(baduk) == Animal   # False  ← 개는 동물 아니라고?? 🚨

# isinstance() — 상속 포함
isinstance(baduk, Dog)     # True
isinstance(baduk, Animal)  # True  ← 개도 동물의 한 종류 ✅
```

```
결론:
  파이썬 공식 문서에서도 isinstance() 권장
  상속 관계 고려 → 더 안전하고 유연함

  type() 쓸 때:
  "정확히 이 클래스 그 자체인지" 확인할 때만 사용
```

## 한눈에 비교

|항목|`type()`|`isinstance()`|
|---|:-:|:-:|
|상속 인정|❌|✅|
|여러 타입 동시 검사|❌|✅ `(int, float)`|
|공식 권장|-|✅|
|용도|정확한 클래스 확인|타입 안전성 검사|

---

---

# ④ repr() — 값의 진짜 모습 보기

```
type() / isinstance() 로 타입은 알았는데
값 안에 공백 / 특수문자가 숨어있을 수 있음
→ repr() 로 실제 값 확인

print() 는 보기 좋게 출력
repr() 는 있는 그대로 출력 (공백/특수문자 포함)
```

```python
val = "Y         "   # API 응답 (공백 포함)

type(val)            # <class 'str'>   ← 타입은 str 맞음
isinstance(val, str) # True

print(val)           # Y              ← 공백 안 보임
print(repr(val))     # 'Y         '  ← 공백 눈에 보임 ✅

val == "Y"           # False (공백 때문)
val.strip() == "Y"   # True ✅
```

```python
# 타입 디버깅 세트
print(type(val))     # 타입 확인
print(repr(val))     # 값 확인 (공백/특수문자)

# 실전: API 필드 디버깅
for child in item:
    print(f"{child.tag}: {type(child.text)} | {repr(child.text)}")
# hvctayn: <class 'str'> | 'Y         '  ← 공백 발견
# MKioskTy1: <class 'str'> | '정보미제공' ← Y/N 아님
```

```
type()       → "어떤 타입이야?"
isinstance() → "이 타입 맞아?"
repr()       → "실제 값이 뭐야? (숨겨진 문자 포함)"

셋 다 함께 쓰면 타입 관련 에러 대부분 잡을 수 있음
```

---

---

# ⑤ typing 모듈 — 정적 타입 힌트

## 런타임 확인 vs 정적 힌트

```
지금까지 배운 type() / isinstance()
→ 런타임 (실행 중에) 타입을 확인하는 것

typing 모듈
→ 정적 힌트 (실행 전에 IDE / mypy 가 "이거 맞아?" 확인)
→ 실행 자체에는 영향 없음. 힌트일 뿐.
```

```python
# isinstance — 실행 중에 검사 (런타임)
if isinstance(data, list):
    ...   # 틀리면 이 줄에서 잡힘

# typing — 실행 전에 IDE 가 경고 (정적)
def process(data: list) -> int:
    ...   # 틀려도 실행은 됨. IDE 에서 빨간 줄만 표시.
```

## 기본 문법

```python
def 함수명(파라미터: 타입) -> 반환타입:
    ...
```

```python
def greet(name: str) -> str:
    return f"안녕, {name}"

def add(a: int, b: int) -> int:
    return a + b

def process() -> None:   # 반환값 없을 때
    print("완료")
```

## 임포트 — 3.9+ 기준

```python
# 3.9+ : list / dict / tuple / set 는 소문자로 바로 사용 (임포트 불필요)
# 단, 아래 세 가지는 여전히 typing 에서 임포트 필요
from typing import Any, Optional, Union
```

```
3.9 이전 문법    3.9+ 문법       의미
List[str]    →  list[str]      문자열 리스트
Dict[str, Any] → dict[str, Any] 딕셔너리
Tuple[int, str] → tuple[int, str] 튜플
Set[int]     →  set[int]       집합

Any / Optional / Union 은 버전 무관하게 항상 typing 에서 임포트
```

## list / dict / tuple

```python
# list — 원소 타입 지정
def get_ids(exhibitions: list[str]) -> list[int]:
    ...
# list[str] → 문자열 원소를 담은 리스트
# list[int] → 정수 원소를 담은 리스트

# dict — 키 타입, 값 타입 지정
def get_stats() -> dict[str, Any]:
    ...
# dict[str, Any] → 키=str, 값=아무 타입

# tuple — 원소 개수와 각 타입 고정
def get_coord() -> tuple[float, float]:
    return (37.5, 127.0)
# tuple[float, float] → 정확히 float 2개짜리 튜플
```

## Any — 아무 타입이나

```python
from typing import Any

# 어떤 타입이든 허용 (타입 힌트를 포기하는 것)
def process(data: Any) -> None:
    ...

# dict 값이 타입이 제각각일 때 자주 씀
stats: dict[str, Any] = {
    "total":       47,           # int
    "by_location": [(...), ...], # list
    "active":      True,         # bool
}
```

## Optional — None 이 될 수 있을 때

```python
from typing import Optional

# Optional[str] = str 또는 None
def get_location(address: str) -> Optional[str]:
    if not address:
        return None          # None 반환 가능
    return parse(address)    # str 반환 가능
```

```
Optional[str] 은 str | None 의 줄임말
3.10+ 에서는 Optional 없이 str | None 으로 직접 쓸 수 있음
3.9 에서는 Optional 을 쓰는 게 더 명확
```

## Union — 여러 타입 중 하나

```python
from typing import Union

# int 또는 str 둘 다 받을 수 있을 때
def process_id(id: Union[int, str]) -> str:
    return str(id)
```

## 실전 패턴 — load_to_postgres.py

```python
from typing import Any, Optional, Union
import psycopg2

class PostgresLoader:

    def get_connection(self) -> psycopg2.extensions.connection:
        # 반환 타입: psycopg2 커넥션 객체
        return psycopg2.connect(**self.db_config)

    def upsert_exhibitions(self, exhibitions: list[dict[str, Any]]) -> int:
        # exhibitions: 딕셔너리들의 리스트
        # 반환:        적재된 행 수 (int)
        ...

    def get_stats(self) -> dict[str, Any]:
        # 반환: 키=str, 값=여러 타입 섞인 딕셔너리
        stats: dict[str, Any] = {}
        ...
        return stats
```

```
list[dict[str, Any]]
  list      → 리스트
  dict      → 딕셔너리
  str       → 키는 문자열
  Any       → 값은 아무 타입 (int / str / list / bool 다 가능)

→ "딕셔너리들의 리스트"
→ exhibitions = [{"exhibition_id": "001", "title": "...", "is_active": True}, ...]
```

## 타입 힌트가 있을 때 vs 없을 때

```python
# 힌트 없는 버전 — 파라미터가 뭔지 모름
def load(data, config):
    ...

# 힌트 있는 버전 — 호출하기 전에 뭘 넣어야 할지 명확
def load(data: List[Dict[str, Any]], config: Dict[str, str]) -> int:
    ...
```

```
타입 힌트의 효과:
  IDE 자동완성 정확도 올라감
  잘못된 타입 전달 시 IDE 경고
  코드 리뷰 / 협업 시 의도 명확하게 전달
  실행에는 영향 없음 (강제 아님)
```

## 한눈에 정리

| 표현 (3.9+)         | 의미             | 임포트            |
| ----------------- | -------------- | -------------- |
| `str`             | 문자열            | 불필요            |
| `int`             | 정수             | 불필요            |
| `float`           | 소수             | 불필요            |
| `bool`            | 참/거짓           | 불필요            |
| `list[str]`       | 문자열 리스트        | 불필요            |
| `dict[str, Any]`  | 키=str, 값=아무 타입 | `Any` 만 임포트    |
| `tuple[int, str]` | (int, str) 튜플  | 불필요            |
| `set[int]`        | 정수 집합          | 불필요            |
| `Optional[str]`   | str 또는 None    | `Optional` 임포트 |
| `Union[int, str]` | int 또는 str     | `Union` 임포트    |
| `Any`             | 아무 타입          | `Any` 임포트      |
| `-> None`         | 반환값 없음         | 불필요            |