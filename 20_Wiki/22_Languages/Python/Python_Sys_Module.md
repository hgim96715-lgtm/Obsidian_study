---
aliases:
  - sys모듈
  - 시스템제어
  - 표준출력
tags:
  - Python
related:
  - "[[Python_Logging]]"
  - "[[Python_Builtin_Functions]]"
  - "[[00_Python_HomePage]]"
  - "[[Python_Modules_Imports]]"
  - "[[Airflow_Operators]]"
  - "[[Python_Logging]]"
---


# Python_Sys_Module — sys 모듈

## 한 줄 요약

```
파이썬 인터프리터(실행기) 를 직접 제어하는 도구

시스템 정보 조회, 스크립트 강제 종료,
입출력 방향 변경, 모듈 경로 조작 등에 사용
```

---

---

# ① sys.stdout — 표준 출력

```
Standard Output (표준 출력)
print() 함수가 글자를 내보내는 기본 통로 (터미널/모니터)

왜 직접 지정하나:
  Docker 환경에서는 로그를 파일에 숨기지 않고
  stdout 으로 쏴줘야 docker logs 로 밖에서 볼 수 있음
```

```python
import sys
import logging

# 로그를 stdout(터미널) 으로 출력 — Docker 에서 필수
logging.basicConfig(
    level  = logging.INFO,
    format = '%(asctime)s %(levelname)s %(message)s',
    stream = sys.stdout    # ← 파일 아닌 터미널로
)

logger = logging.getLogger(__name__)
logger.info("Producer 시작")
```

```
stream=sys.stdout 없으면:
  로그가 stderr 로 나가거나 파일에 숨겨짐
  docker logs 컨테이너명 으로 안 보일 수 있음

stream=sys.stdout 있으면:
  docker logs train-producer 로 실시간 확인 가능
```

---

---

# ② sys.argv — 실행 인자

```
Argument Vector
터미널에서 스크립트 실행할 때 뒤에 붙인 단어들을 리스트로 받아옴

데이터 엔지니어링에서 가장 많이 씀
"dev 환경으로 돌릴지 prod 환경으로 돌릴지 터미널에서 결정"
```

```bash
# 터미널에서 이렇게 실행
python my_job.py prod 2026-03-15
```

```python
import sys

# sys.argv = ['my_job.py', 'prod', '2026-03-15']
print(sys.argv[0])   # my_job.py    (항상 파일명)
print(sys.argv[1])   # prod         (첫 번째 인자)
print(sys.argv[2])   # 2026-03-15   (두 번째 인자)
```

```python
# 실전 패턴 — 환경 분기
import sys

env = sys.argv[1] if len(sys.argv) > 1 else 'dev'   # 기본값 dev

if env == 'prod':
    DB_HOST = 'prod-db.server.com'
    print("운영 서버 접속")
else:
    DB_HOST = 'localhost'
    print("개발 서버 접속")
```

```
⚠️ 인자 없이 실행하면 sys.argv = ['my_job.py'] (파일명만)
   sys.argv[1] 접근하면 IndexError
   → len(sys.argv) > 1 체크 or try/except 필수
```

---

---

# ③ sys.exit() — 강제 종료

```
프로그램 즉시 종료
중요 설정 없거나 DB 연결 끊겼을 때 "더 이상 못 해!" 하고 멈출 때

종료 코드:
  sys.exit(0)   정상 종료 (성공)
  sys.exit(1)   에러로 인한 종료 (실패)
  → 0 이외는 전부 에러, 숫자 자체는 에러 코드 구분용
```

```python
import sys

try:
    connect_database()
except Exception as e:
    print(f"DB 연결 실패: {e}")
    sys.exit(1)          # 여기서 프로그램 종료

print("이 줄은 절대 실행 안 됨")   # sys.exit() 이후 코드는 실행되지 않음
```

```python
# 실전 패턴 — 필수 인자 체크
import sys

if len(sys.argv) < 2:
    print("사용법: python job.py [dev|prod]")
    sys.exit(1)

env = sys.argv[1]
if env not in ('dev', 'prod'):
    print(f"알 수 없는 환경: {env}")
    sys.exit(1)
```

```
Airflow / Docker 에서의 의미:
  sys.exit(0)  → Airflow 가 태스크 성공으로 인식
  sys.exit(1)  → Airflow 가 태스크 실패로 인식 → 재시도 or 알림
```

---

---

# ④ sys.path — 모듈 경로 ⭐️

```
파이썬이 import 할 때 파일을 찾는 폴더 목록 (리스트)
"Module Not Found" 에러 뜰 때 여기를 확인

import my_module 실행 시:
  sys.path 의 각 경로를 순서대로 뒤짐
  → 찾으면 import 성공 / 못 찾으면 ModuleNotFoundError
```

```python
import sys

# 현재 경로 목록 확인
print(sys.path)
# ['',                             ← 현재 작업 디렉토리
#  '/usr/lib/python310.zip',
#  '/usr/lib/python3.10',
#  '/usr/lib/python3.10/lib-dynload',
#  '/usr/local/lib/python3.10/dist-packages',  ← pip 설치 패키지들
#  ...]
```

## sys.path 기본 구성

```
sys.path 에 자동으로 들어오는 것들:
  ''(빈 문자열)      현재 작업 디렉토리
  PYTHONPATH        환경변수로 지정한 경로
  표준 라이브러리     /usr/lib/python3.x/
  site-packages     pip 로 설치한 패키지들

import 탐색 순서:
  1. 현재 디렉토리 ('')
  2. PYTHONPATH 경로들
  3. 표준 라이브러리
  4. site-packages
  → 앞에서 찾으면 즉시 반환 (뒤는 안 봄)
```

## sys.path.append vs sys.path.insert ⭐️

```python
# append — 목록 맨 끝에 추가 (우선순위 낮음)
sys.path.append('/opt/airflow/producer')

# insert(0, ...) — 목록 맨 앞에 추가 (우선순위 높음) ⭐️
sys.path.insert(0, '/opt/airflow/producer')
```

```
차이:
  append        → 기존 경로 다 탐색 후 마지막에 확인
  insert(0, ...) → 가장 먼저 확인

  같은 이름의 모듈이 여러 경로에 있을 때:
  insert(0, ...) → 내가 추가한 경로의 모듈을 우선 사용
  → 충돌 방지 목적이면 insert(0, ...) 사용
```

## ModuleNotFoundError 디버깅 패턴 ⭐️

```python
# 에러 발생 시 확인 절차
import sys

# 1. 현재 sys.path 출력
print('\n'.join(sys.path))

# 2. 찾고 싶은 모듈이 어디 있는지 확인
import os
print(os.path.exists('/opt/airflow/producer/producer.py'))

# 3. 경로 추가
sys.path.insert(0, '/opt/airflow/producer')

# 4. 다시 import
from producer import TrainProducer
```

```
자주 겪는 ModuleNotFoundError 원인:
  가상환경 활성화 안 됨    → source venv/bin/activate
  pip 설치 안 됨          → pip install 패키지명
  경로 문제               → sys.path.insert(0, 경로)
  __init__.py 없음        → 패키지 폴더에 빈 파일 생성
```

## 실전 사용 패턴

```python
# Airflow DAG 에서 producer 코드 import 할 때
# docker-compose volumes 로 /opt/airflow/producer 마운트되어 있어야 함
import sys
sys.path.insert(0, '/opt/airflow/producer')

from producer import TrainProducer   # 이제 import 가능
```

```python
# 로컬 개발 시 상위 폴더의 모듈 import
import sys
import os

# 현재 파일 기준 상위 디렉토리를 경로에 추가
sys.path.insert(0, os.path.dirname(os.path.abspath(__file__)))

# 또는 특정 상위 폴더
BASE_DIR = os.path.dirname(os.path.dirname(os.path.abspath(__file__)))
sys.path.insert(0, BASE_DIR)
```

```python
# PYTHONPATH 환경변수 확인 (sys.path 에 자동 포함)
import os
print(os.environ.get('PYTHONPATH', '없음'))

# .env 또는 docker-compose 에서 설정
# PYTHONPATH=/opt/airflow/producer:/opt/airflow/common
```

```
⚠️ 주의:
  sys.path 조작은 임시방편
  패키지 구조를 제대로 잡거나 pip install -e . 이 더 깔끔
  하지만 Airflow / Docker 환경에서는 sys.path.insert 가 현실적
```

---

---

# 치트시트

|속성/함수|용도|핵심|
|---|---|---|
|`sys.stdout`|표준 출력 통로|Docker 로그에서 `docker logs` 로 보려면 필수|
|`sys.argv`|실행 인자 리스트|`argv[0]` = 파일명, `argv[1]` 부터 인자|
|`sys.exit(0)`|정상 종료|Airflow 태스크 성공으로 인식|
|`sys.exit(1)`|에러 종료|Airflow 태스크 실패로 인식|
|`sys.path`|모듈 탐색 경로 목록|`ModuleNotFoundError` 날 때 확인|
|`sys.path.insert(0, ...)`|경로 우선 추가|Airflow DAG 에서 producer import 할 때|
|`sys.path.append(...)`|경로 끝에 추가|우선순위 낮아도 될 때|