---
aliases:
  - 달력_모듈
  - monthrange
  - isleap
tags:
  - Python
related:
  - "[[Python_DateTime]]"
  - "[[Python_Unpacking]]"
  - "[[00_Python_HomePage]]"
---
### Concept Summary(한줄 요약)

**`calendar`** 모듈은 윤년 계산이나 월별 마지막 날짜 같은 복잡한 달력 규칙을 파이썬이 대신 계산해 주는 **기본 내장 도구**입니다.

---

### Why is it needed

- **문제점:** "4로 나누어 떨어지면 윤년..." 같은 달력 규칙을 직접 `if`문으로 짜면 코드가 길어지고 버그가 폭발합니다.
- **💡 해결책:** 파이썬에 이미 내장된 `calendar`를 쓰면, **단 한 줄**로 예외 없이 정확한 날짜 수치를 얻을 수 있습니다.

---

### Practical Context

- **언제 쓰나요?** 데이터 엔지니어가 **월말 정산 배치(Batch) 스크립트**를 짤 때 무조건 씁니다.
- **어떻게 쓰나요?** 매월 1일에 "지난달 말일이 28일인지 31일인지"를 DB에 쿼리하기 전, 파이썬으로 동적 계산할 때 찰떡입니다.
- (macOS 터미널에서 `pip install` 없이 바로 `import` 가능)

---
## Code Core Points

### 윤년 확인하기 

- **문법:** `calendar.isleap(연도)`
- **동작 원리:** 2월 29일이 있는 해(윤년)면 `True`, 아니면 `False`를 돌려줍니다.
- **코드 해석:** `calendar.isleap(2026)` ➡️ `False` 반환 

```python
import calendar

target_year = 2026
is_leap = calendar.isleap(target_year)

print(f"윤년인가요? {is_leap}") # False
```


### 특정 연월의 말일 구하기 

- **문법:** `calendar.monthrange(연도, 월)`
- **동작 원리:** `(1일의 시작 요일 인덱스, 그 달의 마지막 날짜)`라는 두 개의 데이터를 세트(튜플)로 줍니다.
- **코드 해석 (Unpacking 활용):** 반환된 튜플에서 요일 인덱스는 안 쓰므로 언더스코어(`_`)를 써서 버리고, 두 번째 값인 마지막 날짜만 변수에 담습니다.

```python
import calendar

target_year = 2026
target_month = 2

# 요일 정보는 버리고(_) 말일만(last_day) 가져오기
_, last_day = calendar.monthrange(target_year, target_month)

print(f"마지막 날은 {last_day}일입니다.") # 결과: 28일입니다.
```

---
## 달력을 주차별 리스트로 쪼개기 (monthcalendar)

- **문법:** `calendar.monthcalendar(연도, 월)`
-  **동작 원리:** 달력을 **주(Week) 단위의 리스트**로 쪼개서, 커다란 2차원 리스트(배열)로 돌려줍니다. (월요일부터 시작)
-  **코드 해석:** 해당 월에 포함되지 않는 빈 칸(예: 1일 전의 월~토요일)은 숫자 `0`으로 채워집니다. 특정 주차의 특정 요일을 배열 인덱싱으로 쉽게 찾을 수 있습니다.

```python
import calendar

cal_matrix = calendar.monthcalendar(2026, 2)

for week in cal_matrix:
    print(week)

# 출력 결과 (월, 화, 수, 목, 금, 토, 일 순서)
# [0, 0, 0, 0, 0, 0, 1]  <- 1주차 (일요일이 1일)
# [2, 3, 4, 5, 6, 7, 8]  <- 2주차
# [9, 10, 11, 12, 13, 14, 15] <- 3주차
# [16, 17, 18, 19, 20, 21, 22] <- 4주차
# [23, 24, 25, 26, 27, 28, 0] <- 5주차 (말일은 28일, 일요일은 0)
```


---
## 흔한 실수

- "날짜 다루는 건 `datetime` 모듈 아닌가요?"
	- `datetime` = "지금 몇 시지?" 같은 **시간의 흐름** (타이머 역할)
	- `calendar` = "올해 2월은 며칠까지?" 같은 **달력의 규칙** (달력 역할)
	-  실무에서는 이 두 개를 같이 씁니다!

- "`last_day = calendar.monthrange(2026, 2)` 이렇게 쓰면 안 되나요?"
	- 절대 안 됩니다! 저렇게 쓰면 `last_day`에 숫자 하나가 아니라 `(6, 28)`이라는 덩어리가 통째로 들어갑니다. 
	- 나중에 더하기 빼기 할 때 에러 나니까, 반드시 **`_, last_day`** 로 쪼개서(Unpacking) 숫자 알맹이만 쏙 빼야 합니다.