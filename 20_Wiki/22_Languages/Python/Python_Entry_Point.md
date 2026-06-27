---
aliases:
  - if __name__ == "__main__"
  - 파이썬 진입점
  - 메인 함수
  - 실행 제어
tags:
  - Python
related:
  - "[[Python_Modules_Imports]]"
---
#  파이썬의 시동키: `if __name__ == '__main__':`

## 개념 한 줄 요약 (Concept Summary)

**"이 파일이 주인공(Main)으로 실행될 때만 작동하는 안전장치."**
C나 Java의 `main()` 함수처럼, 파이썬 스크립트가 시작되는 **진입점(Entry Point)** 을 명시하는 문법이다.

---
## 왜 필요한가 (Why)

* **모듈 재사용:** 내가 짠 코드를 다른 파일에서 `import` 해서 부품처럼 쓰고 싶을 때가 있다.
* **자동 실행 방지:** 이 구문이 없으면, `import`만 했는데도 그 파일 안의 코드가 **제멋대로 실행**되어 버린다. (프린트가 찍히거나 함수가 돌아감)
* **명확성:** "아, 이 파일은 여기서부터 실행되는구나"라고 누가 봐도 알 수 있다.

---
## Practical Context (실무 활용)

* **Flink/Spark 작업:** 드라이버 프로그램이 시작되는 곳을 Flink 클러스터에게 알려줄 때 사용한다.
* **테스트 코드:** 모듈 파일 안에 `test()` 함수를 만들어놓고, 직접 실행할 때만 테스트가 돌게 하고, import 할 땐 안 돌게 할 수 있다.

---
##  Code Core Points

파이썬은 파일을 실행할 때 내부적으로 `__name__`이라는 숨겨진 변수에 이름을 붙여준다.

```python
def my_function():
    print("함수가 호출되었습니다.")

print(f"현재 이 파일의 이름(__name__)은: {__name__}")

# 👇 여기가 핵심!
if __name__ == "__main__":
    print("🎬 주인공으로 실행됨! (python file.py)")
    my_function()
else:
    print("🧩 부품으로 불려옴! (import file)")
```

### 동작원리

- **직접 실행 시 (`python file.py`):** `__name__` 변수에 **`"__main__"`** 이라는 값이 들어감. -> **조건문 참(True)** -> 실행 O
- **임포트 시 (`import file`):** `__name__` 변수에 **`"파일명"`** 이 들어감. -> **조건문 거짓(False)** -> 실행 X

---
## Common Beginner Misconceptions (실수)

- **오타 주의:** `__main__` 처럼 앞뒤로 **밑줄 두 개(`__`)** 이다. 하나만 쓰면 안 돌아간다.
- **들여쓰기:** `if` 문 아래에 있는 코드들만 보호받는다. `if` 문 밖에 쓴 코드는 import 할 때 무조건 실행된다.