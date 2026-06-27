---
aliases:
  - os
  - file
  - with
  - open
  - close
  - read()
  - write
tags:
  - Python
related:
  - "[[Text_vs_Binary]]"
  - "[[Python_JSON]]"
  - "[[Python_Pathlib]]"
  - "[[00_Python_HomePage]]"
---
## 개념 한 줄 요약

**"프로그램이 꺼져도 사라지지 않는 '파일'을 만들거나 읽어오는 방법 (feat. 파일 경로 관리자 `os`)"**

---
## 기본 문법 (`open` & `with`)

파일을 열 때는 반드시 **`with` 구문(Context Manager)** 을 쓰는 습관을 들여야 합니다. 
파일을 다 쓴 뒤에 `close()`를 안 해도 알아서 닫아주기 때문입니다. (실수 방지!)

```python
# 1. 파일 쓰기 (Write)
# 'data.txt' 파일을 쓰기 모드('w')로 엽니다. (없으면 새로 만듦)
with open("data.txt", "w", encoding="utf-8") as f:
    f.write("Hello Python!\n")
    f.write("파일 쓰기 연습 중입니다.")

# 2. 파일 읽기 (Read)
# 'data.txt' 파일을 읽기 모드('r')로 엽니다.
with open("data.txt", "r", encoding="utf-8") as f:
    content = f.read()
    print(content)
```

---
## 자주 쓰는 모드 (Mode) 3대장

`open()` 함수의 두 번째 파라미터로 결정합니다.

|**모드**|**설명**|**주의사항**|
|---|---|---|
|**`'r'`** (Read)|**읽기 전용**. 파일 내용을 가져옵니다.|파일이 없으면 에러(`FileNotFoundError`) 발생|
|**`'w'`** (Write)|**쓰기 전용**. 새 내용을 씁니다.|**기존 내용이 싹 지워지고** 덮어씌워짐! (주의)|
|**`'a'`** (Append)|**이어 쓰기**. 기존 내용 뒤에 추가합니다.|로그(Log) 파일 만들 때 주로 사용|

---
## 파일 관리의 단짝, `os` 모듈 

파일을 읽고 쓰기 전에 **"그 파일이 진짜 있나?", "경로가 맞나?"** 확인하려면 `os` 모듈이 필수입니다.

```python
import os

file_path = "data.txt"

# 1. 파일 존재 여부 확인 (가장 많이 씀!)
if os.path.exists(file_path):
    print("파일이 있습니다. 읽을게요!")
    with open(file_path, "r") as f:
        print(f.read())
else:
    print("파일이 없어서 읽을 수 없습니다.")

# 2. 파일 삭제하기
os.remove(file_path)

# 3. 폴더(디렉토리) 만들기
os.makedirs("my_folder", exist_ok=True) # 이미 있어도 에러 안 나게 함
```

---
## 실전 예제 (대시보드 종료 신호)

 **"플래그 파일"** 방식이 바로 이 `File I/O`와 `os`의 완벽한 콜라보 예제입니다.

```python
import os

# [Dashboard] 종료 신호 보내기 (파일 생성)
with open(".stop_signal", "w") as f:
    f.write("stop")

# [Producer] 신호 확인하고 멈추기 (파일 체크 & 삭제)
if os.path.exists(".stop_signal"):
    print("종료 신호 감지!")
    os.remove(".stop_signal") # 다 쓴 파일은 청소
    break
```
