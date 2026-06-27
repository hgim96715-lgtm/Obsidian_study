---
aliases:
  - os 모듈
  - 환경변수
  - 파일삭제
  - 디렉토리 관리
  - os.path
tags:
  - Python
related:
  - "[[Python_Pathlib]]"
  - "[[Python_File_IO]]"
  - "[[Python_Sys_Module]]"
  - "[[00_Python_HomePage]]"
---


# Python_OS_Module

## 개념 한 줄 요약

> **파이썬이 운영체제(OS)의 파일, 폴더, 환경 변수까지 직접 다룰 수 있게 해주는 만능 도구.**

---

---

# import

```python
import os
```

---

---

# 왜 필요한가

```
pathlib 가 나온 이후에도 os 모듈을 버릴 수 없는 이유:

① 환경 변수 (os.environ / os.getenv)
   DB 비밀번호, API 키 같은 민감한 정보는
   코드에 직접 쓰지 않고 환경 변수로 숨겨서 가져옴
   → 데이터 엔지니어 필수

② 레거시 코드 호환
   pathlib 이전 코드는 전부 os.path 로 작성됨
   읽고 유지보수할 수 있어야 함

③ 시스템 제어
   프로세스 ID, 현재 작업 디렉토리, 시스템 명령 실행 등
   os 모듈에만 있는 기능이 있음
```

---

---

# ① 환경 변수 — 데이터 엔지니어 필수 ⭐️

## os.environ.get() vs os.getenv() — 차이

```python
import os

# 방법 1: os.environ.get()
api_key = os.environ.get("TRAIN_API_KEY")
api_key = os.environ.get("TRAIN_API_KEY", "기본값")  # 없을 때 기본값 지정

# 방법 2: os.getenv()  ← os.environ.get() 의 단축 버전
api_key = os.getenv("TRAIN_API_KEY")
api_key = os.getenv("TRAIN_API_KEY", "기본값")       # 없을 때 기본값 지정
```

```
os.environ.get() 과 os.getenv() 는 동작이 완전히 동일함
os.getenv() 가 더 짧아서 실무에서 더 많이 씀

두 방법 모두:
  환경 변수가 있으면  → 값 반환
  없으면             → None 반환 (기본값 지정 안 했을 때)
  없고 기본값 있으면 → 기본값 반환
```

## `os.environ[] `— 직접 접근 (주의)

```python
# 딕셔너리처럼 직접 접근
api_key = os.environ["TRAIN_API_KEY"]
# 환경 변수가 없으면 KeyError 발생 ← 위험

# .get() 이나 getenv() 를 써야 안전
```

## 환경 변수 설정 (코드 안에서 임시로)

```python
# 코드 안에서 임시로 환경 변수 설정
os.environ["KAFKA_TOPIC"] = "train-realtime"
print(os.environ["KAFKA_TOPIC"])  # train-realtime

# 주의: 이 프로세스 안에서만 유효함. 터미널에는 영향 없음
```

## 실전 패턴 — .env 파일과 함께

```python
import os
from dotenv import load_dotenv  # pip install python-dotenv

load_dotenv()  # .env 파일을 읽어서 환경 변수로 등록

# 이제 os.getenv 로 꺼내 씀
api_key  = os.getenv("TRAIN_API_KEY")
db_host  = os.getenv("DB_HOST", "localhost")    # 없으면 localhost
db_port  = int(os.getenv("DB_PORT", "5432"))    # 문자열 → int 변환 필수

if not api_key:
    raise ValueError("TRAIN_API_KEY 환경 변수가 설정되지 않았습니다.")
```

```
os.getenv() 는 항상 문자열(str) 로 반환
숫자가 필요하면 int() / float() 으로 변환해야 함
```

---

---

# ② 파일 / 폴더 존재 확인

```python
import os

# 파일 또는 폴더가 존재하는지 (True / False)
os.path.exists("data/train.csv")      # 있으면 True

# 파일인지만 확인
os.path.isfile("data/train.csv")      # 파일이면 True, 폴더면 False

# 폴더인지만 확인
os.path.isdir("data/2026")            # 폴더면 True, 파일이면 False
```

```python
# 실전 패턴: 파일 없으면 건너뜀
file_path = "data/train.csv"

if not os.path.exists(file_path):
    print(f"파일 없음: {file_path}")
else:
    process(file_path)
```

---

---

# ③ 폴더 생성 / 삭제

```python
# 폴더 만들기 (하위 폴더까지 한 번에)
os.makedirs("data/2026/03", exist_ok=True)
#                            ↑ 이미 있어도 에러 안 냄 ← 실무 필수 옵션

# 빈 폴더 삭제 (안에 파일 있으면 에러)
os.rmdir("empty_folder")

# 내용물 있는 폴더 삭제 → os 로는 불편, shutil 사용 권장
import shutil
shutil.rmtree("data/2026")   # 하위 내용 전부 삭제 (복구 불가!)
```

---

---

# ④ 파일 삭제 / 이름 변경

```python
# 파일 삭제 (휴지통 없이 즉시 삭제, 복구 불가!)
os.remove("useless_file.txt")

# 안전하게 삭제 (없어도 에러 안 나게)
if os.path.exists("useless_file.txt"):
    os.remove("useless_file.txt")

# 파일 이름 변경 / 이동
os.rename("old_name.txt", "new_name.txt")
os.rename("data/file.txt", "backup/file.txt")   # 경로 이동도 가능
```

---

---

# ⑤ 경로 다루기 (os.path)

```python
full_path = "data/uploads/train_2026.csv"

# 파일명 추출 (경로에서 마지막 부분)
os.path.basename(full_path)     # "train_2026.csv"

# 폴더 경로 추출 (파일명 빼고)
os.path.dirname(full_path)      # "data/uploads"

# 파일명 + 확장자 분리
name, ext = os.path.splitext("train_2026.csv")
# name = "train_2026" / ext = ".csv"

# 경로 합치기 (OS 에 맞게 자동으로 구분자 사용)
joined = os.path.join("data", "uploads", "train.csv")
# Windows: data\uploads\train.csv
# Mac/Linux: data/uploads/train.csv
```

```
경로 합칠 때 os.path.join 쓰는 이유:
  "data" + "/" + "uploads"  → Windows 에서 깨질 수 있음
  os.path.join("data", "uploads")  → 운영체제에 맞게 자동 처리
```

---

---

# ⑥ 디렉토리 탐색

```python
# 현재 작업 디렉토리 확인 (Current Working Directory)
cwd = os.getcwd()
print(cwd)   # /home/user/project

# 디렉토리 안 파일/폴더 목록 (그냥 이름만)
files = os.listdir("data")
print(files)  # ['train.csv', 'schedule.csv', '2026']

# 하위 폴더까지 전부 탐색 (os.walk)
for dirpath, dirnames, filenames in os.walk("data"):
    for fname in filenames:
        full = os.path.join(dirpath, fname)
        print(full)
# data/train.csv
# data/2026/03/schedule.csv
# ...
```

---

---

# os vs pathlib — 언제 뭘 쓰나

```
os.getenv / os.environ    → 무조건 os (pathlib 대체 불가)
환경 변수는 os 의 영역

파일/폴더 경로 다루기     → pathlib 권장 (더 직관적)
  pathlib: Path("data") / "uploads" / "train.csv"
  os    : os.path.join("data", "uploads", "train.csv")

레거시 코드 읽기          → os.path 읽을 줄 알아야 함

결론:
  환경 변수 → os.getenv
  경로 조작 → pathlib
  레거시    → os.path 읽기 능력 필요
```

---

---

# 초보자 실수 포인트

```
① os.environ["KEY"] 로 접근 → 없으면 KeyError
   os.getenv("KEY") 또는 os.environ.get("KEY") 를 써야 안전

② os.getenv 는 항상 문자열 반환
   int(os.getenv("DB_PORT", "5432"))  ← 숫자는 형변환 필수

③ os.remove 는 휴지통 없이 즉시 삭제
   복구 불가 → 삭제 전 os.path.exists 로 확인하는 습관

④ os.makedirs vs os.mkdir 차이
   os.mkdir   : 한 단계만 생성 (부모 폴더 없으면 에러)
   os.makedirs: 중간 폴더 전부 자동 생성 ← 이걸 쓸 것
   exist_ok=True 항상 붙이기
```

---

---

# 치트시트

|작업|코드|
|---|---|
|환경 변수 읽기 (안전)|`os.getenv("KEY", "기본값")`|
|환경 변수 읽기 (동일)|`os.environ.get("KEY", "기본값")`|
|파일/폴더 존재 확인|`os.path.exists("path")`|
|파일인지 확인|`os.path.isfile("path")`|
|폴더인지 확인|`os.path.isdir("path")`|
|폴더 생성 (하위 포함)|`os.makedirs("a/b/c", exist_ok=True)`|
|파일 삭제|`os.remove("file.txt")`|
|파일명 추출|`os.path.basename("data/file.csv")`|
|폴더 경로 추출|`os.path.dirname("data/file.csv")`|
|경로 합치기|`os.path.join("data", "uploads", "file.csv")`|
|파일명/확장자 분리|`os.path.splitext("file.csv")`|
|현재 작업 디렉토리|`os.getcwd()`|
|폴더 안 목록|`os.listdir("data")`|