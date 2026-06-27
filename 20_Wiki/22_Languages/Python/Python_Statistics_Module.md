---
aliases:
  - Python Statistics
  - 평균
  - 중앙값
  - 최빈값
  - 통계 함수
tags:
  - Python
related:
  - "[[Python_Math_Module|산술 연산 모듈 (math.prod)]]"
  - "[[Python_Builtin_Functions|내장 함수 (sum, max, min)]]"
  - "[[00_Python_HomePage]]"
---
## 데이터 집합의 특징 파악 (`statistics`) 

수집된 데이터 리스트에서 평균, 중앙값, 분산 등을 계산하여 데이터의 경향성을 파악합니다.

- **문법**: `{python}import statistics`

---
## 주요 함수 (대표값 찾기)

### ① 산술 평균 (`mean`)

모든 값을 더해 개수로 나눈 값입니다. 전체적인 흐름을 볼 때 가장 많이 쓰입니다.

- **문법**: `{python}statistics.mean( <데이터_리스트> )`

### ② 중앙값 (`median`)

데이터를 크기 순으로 세웠을 때 정중앙에 위치한 값입니다.

- **특징**: **이상치(Outlier)** 의 영향을 적게 받습니다. 
- (예: 연봉 데이터 분석 시 연봉 100억인 사람이 1명 섞여 있어도 평균보다 더 정확한 중간을 보여줌)

- **문법**: `{python}statistics.median( data )`

### ③ 최빈값 (`mode`)

가장 자주 등장하는 데이터입니다.

- **활용**: 로그 데이터에서 가장 많이 발생한 에러 코드 등을 찾을 때 유용합니다.
- **문법**: `{python}statistics.mode( data )`

---
## 주요 함수 (흩어진 정도 파악)

### ④ 표준 편차 (`stdev`) & 분산 (`variance`)

데이터들이 평균에서 얼마나 멀리 떨어져 있는지(변동성)를 보여줍니다.

- **활용**: 데이터 파이프라인에서 갑자기 수치가 튀는 경우(표준 편차가 급격히 커짐)를 감지하여 알람을 보낼 때 씁니다.

```python
import statistics

data = [1, 2, 2, 3, 3, 3, 4, 4, 5]

print(statistics.mean(data))     # 평균: 3.0
print(statistics.median(data))   # 중앙값: 3
print(statistics.mode(data))     # 최빈값: 3 (가장 많음)
print(statistics.stdev(data))    # 표준편차: 약 1.22
```

---
## 요약 표 (Cheat Sheet)

| **함수**           | **설명** | **한 줄 비유**       |
| ---------------- | ------ | ---------------- |
| **`mean()`**     | 산술 평균  | **전체의 평균점수**     |
| **`median()`**   | 중앙값    | **딱 중간에 선 사람**   |
| **`mode()`**     | 최빈값    | **제일 인기 많은 놈**   |
| **`stdev()`**    | 표준 편차  | **평균에서 얼마나 멀어?** |
| **`variance()`** | 분산     | **얼마나 퍼져있어?**    |