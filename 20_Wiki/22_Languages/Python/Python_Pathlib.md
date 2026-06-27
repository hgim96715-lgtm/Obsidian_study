---
aliases:
  - stem
  - suffix
  - 확장자추출
  - name
  - Path
  - exists
  - mkdir
  - touch
tags:
  - Python
related:
  - "[[Python_File_IO]]"
  - "[[00_Python_HomePage]]"
  - "[[Python_OS_Module]]"
---


## 개념 한 줄 요약

**"파일 경로를 단순한 '글자(String)'가 아니라, 기능이 담긴 '객체(Object)'로 다루는 현대적인 방법"**

---
## 왜 `os` 대신 쓸까요?

기존 `os.path`는 경로를 문자열로만 취급해서 코드가 지저분했습니다. 
`pathlib`는 경로를 **객체**로 만들어서 훨씬 직관적이고 똑똑하게 작동합니다.

### ① 직관적인 경로 합치기 (`/` 연산자)

```python
from pathlib import Path

# [Old] os.path 방식: 콤마(,)와 함수 호출 범벅
import os
p = os.path.join("data", "uploads", "image.png")

# [New] pathlib 방식: 그냥 나누기(/) 기호로 연결! ⭐️
p = Path("data") / "uploads" / "image.png"
```

**핵심:** 맨 앞의 `Path("data")`만 객체로 선언해주면, 뒤따라오는 문자열들은 자동으로 경로의 일부가 됩니다.

---
## 혁신적인 파일 읽고 쓰기 (No more `with open`!)

`pathlib`를 쓰면 복잡한 `with open(...)` 구문을 **단 한 줄**로 줄일 수 있습니다. (텍스트 파일 다룰 때 최고!)

### ① 파일 쓰기 (`write_text`)

파일을 열고(w), 쓰고, 닫는 과정을 함수 하나가 알아서 다 처리합니다.

```python
# 파일 객체 생성 (아직 파일이 만들어진 건 아님)
log_file = Path("server_log.txt")

# 파일 생성 + 내용 쓰기 + 닫기 (자동)
log_file.write_text("Server Started!", encoding="utf-8")
```

### ② 파일 읽기 (`read_text`)

```python
# 파일 열기 + 전체 읽기 + 닫기 (자동)
content = log_file.read_text(encoding="utf-8")
print(content)
```

---
## 실전 예제: 대시보드 종료 신호 만들기 

### 상황 A: 명확한 메시지를 남길 때 (`write_text`)

파일 안에 "stop"이나 "pause" 같은 구체적인 명령어를 적어둘 때 씁니다.

```python
signal = Path(".stop_signal")

# 파일을 만들고 그 안에 "stop"이라고 적습니다. (가장 확실함)
signal.write_text("stop", encoding="utf-8")
```

### 상황 B: 그냥 신호만 줄 때 (`touch`)

파일 내용 따윈 필요 없고, **"파일이 거기 있다"** 는 사실 자체가 신호일 때 씁니다.

```python
signal = Path(".stop_signal")

# 내용 없는 빈 껍데기 파일만 툭(touch) 만듭니다.
# exist_ok=True: 이미 있어도 에러 내지 말라는 뜻
signal.touch(exist_ok=True)
```

---
## 파일명과 확장자 분리 (`name`,`stem`, `suffix`)

```python
from pathlib import Path

p = Path("/user/logs/2026_data_report.csv")

# 1. name: 전체 파일 이름 (이름 + 확장자)
print(p.name)    # 결과: "2026_data_report.csv"

# 2. stem: 확장자를 뺀 순수 이름 (알맹이)
print(p.stem)    # 결과: "2026_data_report"

# 3. suffix: 마지막 점(.)을 포함한 확장자 (꼬리표)
print(p.suffix)  # 결과: ".csv"
```

#### 심화 학습: 점이 여러 개인 경우 (`suffixes`)

데이터 엔지니어링에서는 `.tar.gz`나 `.csv.tmp` 같은 파일을 자주 만납니다. 
이때 `suffix`만 쓰면 마지막 하나만 가져오기 때문에 주의가 필요합니다.

|**속성**|**코드 예시 (Path("data.tar.gz"))**|**결과값**|**설명**|
|---|---|---|---|
|**`stem`**|`p.stem`|`"data.tar"`|**마지막 점** 기준 왼쪽 전부|
|**`suffix`**|`p.suffix`|`".gz"`|**마지막 점** 기준 오른쪽|
|**`suffixes`**|`p.suffixes`|`[".tar", ".gz"]`|모든 확장자를 **리스트**로 반환|

---
## 요약 표

|**기능**|**os / open (구식)**|**pathlib (신식)**|**비고**|
|---|---|---|---|
|**경로 생성**|`os.path.join(a, b)`|`Path(a) / b`|가독성 압승|
|**파일 쓰기**|`with open(...) as f: f.write()`|`path.write_text()`|한 줄 컷|
|**빈 파일 생성**|`open(p, 'a').close()`|`path.touch()`|리눅스 명령어와 동일|
|**존재 확인**|`os.path.exists(p)`|`path.exists()`||
|**폴더 생성**|`os.makedirs(p)`|`path.mkdir(parents=True)`|중간 폴더 자동 생성|
