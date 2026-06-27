---
aliases:
  - argparse
  - 명령줄인자
  - --옵션
tags:
  - Python
related:
  - "[[00_Python_HomePage]]"
  - "[[Python_Sys_Module]]"
---

# Python_Argparse — 명령줄 인자 파싱

## 한 줄 요약

```
python3 main.py --mode test --pages 5
→ --옵션 스타일 인자를 깔끔하게 받는 표준 방법
```

---

---

# ① sys.argv vs argparse

```python
# python3 main.py --mode test --pages 5

# sys.argv 방식 — 원시적
import sys
print(sys.argv)
# ['main.py', '--mode', 'test', '--pages', '5']
mode = sys.argv[2]   # 위치로 가져옴 → 순서 바뀌면 망함

# argparse 방식 — 표준 ✅
import argparse
parser = argparse.ArgumentParser()
parser.add_argument("--mode")
parser.add_argument("--pages", type=int)
args = parser.parse_args()

print(args.mode)    # "test"
print(args.pages)   # 5
```

```
sys.argv:
  순서 의존 → --mode 와 --pages 순서 바뀌면 틀림
  타입 변환 없음 → 전부 문자열

argparse:
  이름으로 받음 → 순서 상관없음
  type 자동 변환 → --pages 5 → int 5
  --help 자동 생성
```

---

---

# ② 기본 구조

```python
import argparse

# 1. 파서 생성
parser = argparse.ArgumentParser(description="크롤러 실행 스크립트")

# 2. 인자 추가
parser.add_argument("--mode",  type=str, help="실행 모드 (test / crawl)")
parser.add_argument("--pages", type=int, help="수집할 페이지 수")

# 3. 파싱
args = parser.parse_args()

# 4. 사용
print(args.mode)
print(args.pages)
```

```bash
python3 main.py --mode crawl --pages 5
python3 main.py --pages 5 --mode crawl   # 순서 달라도 동일
python3 main.py --help                    # 자동 도움말 출력
```

---

---

# ③ add_argument 옵션 ⭐️

```python
parser.add_argument(
    "--mode",
    type=str,              # 타입 변환 (str / int / float / bool)
    default="test",        # 기본값 (없을 때)
    required=True,         # 필수 여부
    choices=["test", "crawl"],  # 허용 값 제한
    help="실행 모드"        # --help 에 표시될 설명
)
```

## 자주 쓰는 패턴

```python
# 문자열
parser.add_argument("--mode", type=str, default="test")

# 정수
parser.add_argument("--pages", type=int, default=1)

# 실수
parser.add_argument("--rate", type=float, default=0.5)

# True/False 플래그 (--verbose 있으면 True)
parser.add_argument("--verbose", action="store_true")

# 선택지 제한
parser.add_argument("--env", choices=["dev", "prod"], default="dev")

# 필수 인자
parser.add_argument("--output", required=True, help="출력 파일 경로")

# 단축형 (-m 과 --mode 동시 지원)
parser.add_argument("-m", "--mode", type=str)
```

---

---

# ④ 실전 예시

## 크롤러 스크립트

```python
import argparse
import time

def crawl(pages):
    print(f"{pages} 페이지 수집 시작")

def test():
    print("테스트 모드")

if __name__ == "__main__":
    parser = argparse.ArgumentParser(description="웹 크롤러")
    parser.add_argument("--mode",  type=str, choices=["test", "crawl"], default="test")
    parser.add_argument("--pages", type=int, default=1)
    parser.add_argument("--delay", type=float, default=1.0, help="페이지 간 딜레이(초)")
    parser.add_argument("--verbose", action="store_true", help="상세 로그 출력")

    args = parser.parse_args()

    if args.verbose:
        print(f"mode={args.mode} / pages={args.pages} / delay={args.delay}")

    if args.mode == "test":
        test()
    elif args.mode == "crawl":
        crawl(args.pages)
```

```bash
python3 main.py --mode test
python3 main.py --mode crawl --pages 5
python3 main.py --mode crawl --pages 5 --delay 0.5 --verbose
python3 main.py --help
```

## Airflow BashOperator 에서 활용

```python
# DAG 안에서 스크립트에 인자 전달
from airflow.operators.bash import BashOperator

crawl_task = BashOperator(
    task_id="crawl",
    bash_command="python3 /opt/scripts/main.py --mode crawl --pages 10"
)
```

---

---

# ⑤ --help 자동 생성

```bash
python3 main.py --help

# 출력:
# usage: main.py [-h] [--mode {test,crawl}] [--pages PAGES] [--delay DELAY] [--verbose]
#
# 웹 크롤러
#
# options:
#   -h, --help            show this help message and exit
#   --mode {test,crawl}   실행 모드
#   --pages PAGES         수집할 페이지 수
#   --delay DELAY         페이지 간 딜레이(초)
#   --verbose             상세 로그 출력
```

---

---

# 자주 하는 실수

|실수|원인|해결|
|---|---|---|
|`--pages 5` 인데 타입이 str|type 지정 안 함|`type=int` 추가|
|인자 없이 실행했는데 에러|required=True|`default` 값 지정|
|`--verbose True` 로 전달|store_true 의미 오해|`--verbose` 만 붙이면 True|
|스크립트 직접 실행 시 작동, 모듈 import 시 parse_args 실행|if **name** 없음|`if __name__ == "__main__":` 안에 작성|