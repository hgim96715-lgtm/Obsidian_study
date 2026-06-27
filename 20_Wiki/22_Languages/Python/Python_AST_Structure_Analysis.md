---
aliases:
  - Abstract Syntax Tree
  - ast.parse
  - ast.walk
tags:
  - Python
related:
  - "[[PyFlink_Operators_Basic]]"
  - "[[00_Python_HomePage]]"
---
# Python AST: 코드의 뼈대를 보는 X-ray

> [!QUOTE] 핵심 요약
> **"우리가 짠 영어 단어(코드)를 파이썬 인터프리터가 이해하기 좋게 '트리 구조'로 바꾼 것."**
> **실무:** 문자열로 된 자료구조(`"['a','b']"`)를 안전하게 **복구**할 때 씀 (`literal_eval`).
> **고급:** 코드를 **분석**하거나, **수정**하거나, **새로 만들 때** 씀 (`parse`, `dump`, `walk`).

---
## 4대장 함수 (The Big Three) 

|**함수**|**역할**|**비유**|**추천 상황**|
|---|---|---|---|
|**`literal_eval`**|문자열 ➡ 객체 변환|**복원술**|**로그 파일 파싱, Kafka 메시지 처리 (필수!)**|
|**`parse`**|코드 ➡ AST 변환|**CT 촬영**|코드를 분석하고 싶을 때|
|**`dump`**|AST 구조 출력|**판독**|AST가 어떻게 생겼나 궁금할 때|
|**`walk`**|트리 순회|**수색**|코드 내에서 특정 패턴(함수, 변수 등)을 찾을 때|


---
## 실전: 코드를 해부해보자 (Parse & Dump)

가장 간단한 코드가 파이썬 내부에서 어떻게 보이는지 확인해 봅시다.

- **`ast.parse()`**: 코드를 AST 객체(트리)로 변환.
- **`ast.dump()`**: 트리 내부 구조를 눈으로 확인.

### 실험 대상 코드

```python
code = "x = 1 + 2"
```

### 해부 코드

```python
import ast

# 1. 코드를 트리로 변환 (Parse)
tree = ast.parse("x = 1 + 2")

# 2. 트리의 구조 출력 (Dump)(들여쓰기 적용)
print(ast.dump(tree, indent=4))
```

### 결과 (이게 AST입니다!)

```text
Module(
    body=[
        Assign(  # 할당 (x =)
            targets=[Name(id='x', ctx=Store())],  # 변수 x에 저장
            value=BinOp(  # 이항 연산 (1 + 2)
                left=Constant(value=1),  # 왼쪽: 1
                op=Add(),                # 연산자: +
                right=Constant(value=2)  # 오른쪽: 2
            )
        )
    ],
    type_ignores=[]
)
```

> **해석:**
>  파이썬은 `x = 1 + 2`를 단순히 글자로 보는 게 아니라, **`Assign`(할당)**, **`BinOp`(연산)**, **`Add`(더하기)** 라는 **"부품(Node)"들의 조립**으로 이해합니다.

---
## 활용: "이 파일에 함수가 몇 개지?" (Walk)

정규표현식(RegEx)으로 `def ...`를 찾는 건 실수. 주석 안에 있는 `def`까지 찾거든요.
**AST를 쓰면 진짜 함수 정의만 정확하게 찾아낼 수 있습니다.**

>코드 전체를 순회하면서 내가 원하는 것(함수 정의, 클래스, import 문 등)만 쏙쏙 뽑아낼 수 있습니다. 
>정규표현식보다 훨씬 정확합니다.

**코드 분석기 만들기**

```python
import ast

source_code = """
def hello():
    print("안녕")

def add(a, b):
    return a + b

# def comment(): 주석은 무시해야 함
"""

# 1. 파싱
tree = ast.parse(source_code)

# 2. 순회 (Walk) 하며 함수(FunctionDef)만 찾기
for node in ast.walk(tree):
    if isinstance(node, ast.FunctionDef):
        print(f"함수 발견! 이름: {node.name}")
        print(f" - 인자 개수: {len(node.args.args)}")
```

[출력 결과]

```text
함수 발견! 이름: hello
 - 인자 개수: 0
함수 발견! 이름: add
 - 인자 개수: 2
```

**장점:** 주석 처리된 `# def comment():`는 AST 트리에 포함되지 않거나 무시되므로, **정확도가 100%** 입니다.

---
## ast.unparse() (Python 3.9+)

코드를 분해했으면 다시 조립도 가능할까요? 
네! AST 객체를 다시 파이썬 코드로 바꿔줍니다. (리팩토링 도구 만들 때 씀)

```python
import ast

# 1. 뼈대만 있는 트리 생성
tree = ast.parse("x = 1 + 2")

# 2. (심화) 트리를 살짝 수정해볼까? (1 -> 100으로 변경)
tree.body[0].value.left.value = 100 

# 3. 코드로 복원 (Unparse)
new_code = ast.unparse(tree)
print(new_code) 
# 결과: x = 100 + 2
```

---
## 안전한 파싱 (`ast.literal_eval`)

데이터 엔지니어링에서 가장 많이 쓰는 기능입니다. **"문자열을 진짜 파이썬 객체로 변환"** 합니다.

### 왜 `split`이나 `eval` 대신 쓰는가? 

* **Split:** `("Alice", 1)` 같은 문자열은 쉼표, 괄호, 따옴표 때문에 `split(',')`으로 자르기 매우 어렵습니다. 
* **Eval:** `eval()`은 해킹 코드(`os.system('rm -rf /')`)가 들어있으면 그대로 실행해버려 위험합니다.
* **Literal Eval:** 오직 **데이터 구조(List, Tuple, Dict)** 만 해석하므로 **보안상 완벽하게 안전**합니다.

```python
import ast

# 상황: Kafka에서 튜플 모양의 '문자열'이 들어옴
raw_data = "('Alice', 100)"

# 변환: 안전하게 튜플 객체로 복원
data = ast.literal_eval(raw_data)

print(data[0]) # 'Alice' (문자열)
print(data[1]) # 100 (숫자)
```

---
