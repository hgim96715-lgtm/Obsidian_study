---
aliases:
  - subprocess
  - 외부 명령어
  - 쉘 실행
tags:
  - Python
related:
  - "[[Python_OS_Module]]"
  - "[[Python_Pathlib]]"
  - "[[Python_Logging]]"
  - "[[Python_Error_Handling]]"
---
# Python_Subprocess — 외부 명령어 실행

## 한 줄 요약

```
파이썬에서 외부 프로그램(dbt / git / bash / pip 등) 을 실행하는 표준 라이브러리
returncode 로 성공/실패 판단
Airflow BashOperator 대신 Python 안에서 직접 쉘 명령어 실행할 때 사용
```

---

---

# ① subprocess.run() 기본 ⭐️

```python
import subprocess

# 리스트 형식으로 명령어 전달 (권장)
result = subprocess.run(["ls", "-l"])

# 문자열 형식 (shell=True 필요, 보안상 비권장)
result = subprocess.run("ls -l", shell=True)
```

```
리스트 vs 문자열:
  ["ls", "-l"]     ✅ 권장 (쉘 인젝션 방지)
  "ls -l"          ⚠️ shell=True 필요 / 보안 취약

명령어 + 옵션 분리:
  ["dbt", "run", "--project-dir", "/opt/dbt"]
  ↑ 명령어  ↑옵션    ↑옵션값          ↑경로
  각 토큰을 리스트 요소로 분리
```

---

---

# ② 주요 파라미터 ⭐️

```python
result = subprocess.run(
    cmd,                  # 실행할 명령어 (리스트)
    capture_output=True,  # stdout/stderr 캡처 (화면 출력 안 함)
    text=True,            # 결과를 str 로 받음 (bytes 아닌)
    cwd="/path/to/dir",   # 실행 디렉토리 지정
    timeout=300,          # N초 초과 시 TimeoutExpired 에러
    env={"KEY": "VAL"},   # 환경변수 지정
)
```

## capture_output vs stdout/stderr

```python
# capture_output=True → stdout + stderr 둘 다 캡처 (Python 3.7+)
result = subprocess.run(cmd, capture_output=True, text=True)

# 동일한 표현 (구버전 방식)
result = subprocess.run(
    cmd,
    stdout=subprocess.PIPE,
    stderr=subprocess.PIPE,
    text=True,
)
```

---

---

# ③ 결과 처리 ⭐️

```python
result = subprocess.run(cmd, capture_output=True, text=True)

# 종료 코드
result.returncode   # 0 = 성공 / 0이 아니면 실패

# 표준 출력
result.stdout       # 명령어가 출력한 텍스트

# 표준 에러
result.stderr       # 에러 메시지
```

## returncode 의미

```
0   성공 (Success)
1   일반 에러
2   잘못된 사용법
127 명령어를 찾을 수 없음 (not found)
130 Ctrl+C 로 중단
기타 각 프로그램마다 다름 (dbt 는 1 = 실패)
```

---

---

# ④ 실전 패턴 — dbt run ⭐️

```python
import subprocess
import logging
from pathlib import Path

log = logging.getLogger(__name__)

DBT_PATH = "/opt/airflow/dbt"


def dbt_run():
    """dbt run 실행"""
    dbt_project_dir = Path(DBT_PATH)

    # 1. 경로 존재 확인
    if not dbt_project_dir.exists():
        raise FileNotFoundError(f"dbt 프로젝트 디렉토리 없음: {dbt_project_dir}")

    # 2. 명령어 구성
    cmd = [
        "dbt", "run",
        "--project-dir", str(dbt_project_dir),
        "--profiles-dir", str(dbt_project_dir),
        "--target", "dev",
    ]

    log.info(f"dbt run 시작 :{' '.join(cmd)}")

    # 3. 실행
    result = subprocess.run(
        cmd,
        capture_output=True,       # 출력 캡처
        text=True,                 # str 로 받기
        cwd=str(dbt_project_dir),  # 실행 위치
    )

    # 4. 로그 출력
    if result.stdout:
        log.info("dbt stdout:\n%s", result.stdout)
    if result.stderr:
        log.warning("dbt stderr:\n%s", result.stderr)

    # 5. 실패 처리
    if result.returncode != 0:
        raise RuntimeError(
            f"dbt run 실패 (returncode={result.returncode})\n{result.stderr}"
        )

    log.info("dbt run 완료")
```

## 명령어 구조 상세 ⭐️

```
터미널에서 직접 치는 것:
  dbt run --project-dir /opt/airflow/dbt --profiles-dir /opt/airflow/dbt --target dev

리스트로 표현:
  ["dbt", "run", "--project-dir", "/opt/dbt", "--profiles-dir", "/opt/dbt", "--target", "dev"]

규칙:
  터미널 띄어쓰기 → 리스트의 각 요소로 분리
  옵션 이름과 값은 별도 요소
  "--project-dir /opt/dbt" (X) → "--project-dir", "/opt/dbt" (O)
```

## 각 옵션 의미

```
"dbt"
  실행할 프로그램

"run"
  dbt 서브 명령어
  dbt run    = 모든 모델 실행
  dbt test   = 테스트 실행
  dbt build  = run + test 동시

"--project-dir", str(dbt_project_dir)
  dbt_project.yml 이 있는 프로젝트 폴더 경로
  지정 안 하면 현재 디렉토리에서 찾음
  Docker 환경에서는 컨테이너 내부 경로 지정

"--profiles-dir", str(dbt_project_dir)
  profiles.yml 이 있는 폴더 경로
  profiles.yml = DB 연결 정보 (host / port / user / password)
  보통 project-dir 과 같은 폴더에 넣음

"--target", "dev"
  profiles.yml 안에서 어느 환경을 사용할지
  dev   = 개발 DB
  prod  = 운영 DB
```

## profiles.yml 과 --target 관계

```yaml
# profiles.yml 예시
my_project:
  outputs:
    dev:                    # ← --target dev
      type: postgres
      host: localhost
      port: 5432
      user: dev_user
      dbname: dev_db

    prod:                   # ← --target prod
      type: postgres
      host: prod-server.com
      port: 5432
      user: prod_user
      dbname: prod_db
```

```python
# 환경변수로 target 동적 변경
import os
target = os.environ.get("DBT_TARGET", "dev")   # 기본값 dev

cmd = [
    "dbt", "run",
    "--project-dir", str(dbt_project_dir),
    "--profiles-dir", str(dbt_project_dir),
    "--target", target,    # "dev" 또는 "prod"
]
```

## str() 변환이 필요한 이유

```python
dbt_project_dir = Path("/opt/airflow/dbt")   # Path 객체

# ❌ Path 객체 그대로 넣으면 TypeError
cmd = ["dbt", "run", "--project-dir", dbt_project_dir]

# ✅ str() 로 변환 필수
cmd = ["dbt", "run", "--project-dir", str(dbt_project_dir)]
# str(Path("/opt/airflow/dbt")) = "/opt/airflow/dbt"
```

```
subprocess.run() 은 리스트의 모든 요소가 str 이어야 함
Path 객체는 str 이 아님 → str() 필수
```

## 패턴 설명

```
① Path.exists() 로 경로 먼저 확인
  → 경로 없으면 FileNotFoundError (명확한 에러)

② 명령어를 리스트로 구성
  → " ".join(cmd) 으로 로그 출력 시 보기 좋음

③ capture_output=True + text=True
  → 화면에 안 나오고 result.stdout / result.stderr 로 받음
  → 로그로 남길 수 있음

④ returncode 체크
  → 0 이 아니면 RuntimeError 발생
  → Airflow 는 Exception = 태스크 실패로 처리
```

---

---

# ⑤ 자주 쓰는 패턴

## git 명령어

```python
# git pull
result = subprocess.run(
    ["git", "pull", "origin", "main"],
    capture_output=True, text=True,
    cwd="/opt/airflow/dbt"
)

if result.returncode != 0:
    raise RuntimeError(f"git pull 실패: {result.stderr}")

print(result.stdout)   # Already up to date.
```

## pip install

```python
result = subprocess.run(
    ["pip", "install", "-r", "requirements.txt"],
    capture_output=True, text=True
)
```

## 출력 결과 파싱

```python
# df -h 결과 파싱
result = subprocess.run(["df", "-h"], capture_output=True, text=True)
lines = result.stdout.strip().split("\n")
for line in lines[1:]:   # 헤더 제외
    print(line)
```

## 실시간 출력 (캡처 없이)

```python
# capture_output 없으면 터미널에 실시간 출력
result = subprocess.run(["dbt", "run"])
# 실행 중 터미널에 바로 출력됨
# result.stdout 은 None
```

---

---

# ⑥ 에러 처리 패턴

## returncode 체크

```python
result = subprocess.run(cmd, capture_output=True, text=True)

if result.returncode != 0:
    raise RuntimeError(f"명령어 실패: {result.stderr}")
```

## check=True (자동 에러 발생)

```python
# check=True → returncode != 0 이면 자동으로 CalledProcessError
try:
    result = subprocess.run(cmd, capture_output=True, text=True, check=True)
except subprocess.CalledProcessError as e:
    print(f"실패: {e.returncode}")
    print(f"에러: {e.stderr}")
```

## timeout 처리

```python
try:
    result = subprocess.run(cmd, capture_output=True, text=True, timeout=300)
except subprocess.TimeoutExpired:
    raise RuntimeError("명령어 실행 시간 초과 (5분)")
```

---

---

# ⑦ Airflow 에서 활용

## @task 안에서 사용

```python
from airflow.decorators import task
import subprocess
import logging

log = logging.getLogger(__name__)


@task
def run_dbt():
    result = subprocess.run(
        ["dbt", "run", "--project-dir", "/opt/airflow/dbt"],
        capture_output=True,
        text=True,
    )

    if result.stdout:
        log.info(result.stdout)
    if result.stderr:
        log.warning(result.stderr)

    if result.returncode != 0:
        raise RuntimeError(f"dbt 실패: {result.stderr}")
```

## BashOperator vs subprocess

|구분|BashOperator|subprocess|
|---|---|---|
|위치|DAG 레벨 Task|@task 함수 안|
|로그 제어|제한적|직접 가능|
|에러 처리|returncode 자동|직접 처리|
|Python 로직 결합|어려움|쉬움|
|사용 시점|단순 쉘 명령어|Python 처리 + 명령어 조합|

---

---

# 명령어 한눈에

|파라미터|의미|
|---|---|
|`capture_output=True`|stdout/stderr 캡처 (화면 안 나옴)|
|`text=True`|결과를 str (bytes 아닌)|
|`cwd="경로"`|실행 디렉토리|
|`timeout=N`|N초 초과 시 TimeoutExpired|
|`check=True`|실패 시 자동 CalledProcessError|
|`result.returncode`|0=성공 / 나머지=실패|
|`result.stdout`|표준 출력|
|`result.stderr`|에러 출력|

---

---

# 자주 하는 실수

|실수|원인|해결|
|---|---|---|
|stdout 이 None|capture_output 안 씀|`capture_output=True` 추가|
|결과가 bytes|text=True 안 씀|`text=True` 추가|
|명령어 못 찾음|PATH 문제|절대 경로 사용 `/usr/bin/dbt`|
|returncode 무시|체크 안 함|`if result.returncode != 0: raise`|
|shell=True 사용|습관|리스트 형식으로 변경|