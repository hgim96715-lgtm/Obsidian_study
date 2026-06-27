---
aliases:
  - 파이썬 모듈
  - 임포트 문법
  - import vs from
  - 패키지 구조
tags:
  - Python
related:
  - "[[Python_Virtual_Env]]"
  - "[[Python_Entry_Point]]"
  - "[[00_Python_HomePage]]"
---
## 개념 한 줄 요약 (Concept Summary)

**"남이 만든 바퀴(Wheel)를 가져다 쓰는 법."**
이미 잘 만들어진 코드 덩어리(모듈)를 내 파일로 가져와서(`import`) 사용하는 문법이다.

----
## 왜 필요한가 (Why)

**생산성:** 모든 기능을 내가 처음부터 다 짤 수 없다. (수학 계산, 날짜 처리, Flink 연결 등)
**구조화:** 코드가 1000줄이 넘어가면 파일 하나로 관리하기 힘들다. 기능별로 파일을 쪼개고(모듈화) 필요할 때 불러와야 한다.

---
## Practical Context (실무 활용)

- **표준 라이브러리:** 파이썬 깔면 그냥 주는 것들 (`math`, `datetime`, `json`, `os`)
- **서드파티 라이브러리:** `pip install`로 깐 것들 (`pandas`, `pyflink`, `requests`)
- **로컬 모듈:** 내가 옆에 만들어둔 파일 (`my_utils.py`)

---
## Code Core Points (3가지 방식)

### ① 통째로 가져오기 (`import`)

가장 안전하고 깔끔한 방식. 쓸 때마다 `모듈명.함수`로 써야 함.

```python
import math
print(math.sqrt(16)) # 4.0
# print(sqrt(16)) # 에러! math. 을 안 붙이면 모름.
```

### ② 필요한 것만 쏙 가져오기 (`from ... import`)

`모듈명.` 없이 바로 함수를 쓸 수 있어 편하지만, 이름이 겹칠 수 있음.

```python
from datetime import datetime

print(datetime.now()) # 편함!
```

### ③ 별명 붙이기 (`as`) ⭐️

이름이 너무 길거나 겹칠 때 줄임말을 쓴다. (Pandas, PyFlink에서 필수)

```python
import pandas as pd
import numpy as np
from pyflink.common import Types as T  # 이렇게 줄여 쓰기도 함
```

---
## Detailed Analysis (패키지 구조)

- **모듈(Module):** 파이썬 파일 하나 (`.py`)
- **패키지(Package):** 모듈들을 모아놓은 폴더. (보통 `__init__.py` 라는 파일이 들어있음)

```text
my_project/
├── main.py
└── utils/           <-- 패키지
    ├── __init__.py
    ├── string_utils.py
    └── math_utils.py
```

- `main.py`에서 쓸 때: `from utils import string_utils`

---
## Common Beginner Misconceptions (자주 하는 실수)

- **파일명 충돌 💥:** 내 파일 이름을 `random.py`나 `email.py`로 지으면 안 된다!
    - 파이썬이 진짜 표준 라이브러리(`random`) 대신 **내 파일을 먼저 읽어버려서** `ImportError`가 난다.
- **순환 참조 (Circular Import):** A파일이 B를 부르고, B파일이 다시 A를 부르면 서로 꼬여서 에러가 난다. (설계 단계에서 조심해야 함)