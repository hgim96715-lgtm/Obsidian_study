---
aliases:
  - Python Random Sampling
  - 가중치 랜덤
  - random.choices
  - 확률 조작
tags:
  - Python
related:
  - "[[Python_Random_Seed]]"
  - "[[Python_Lists_Tuples]]"
---
## 개념 한 줄 요약 (Concept Summary)

**"모든 선택지의 확률이 똑같지 않을 때(불공평한 뽑기), 특정 항목이 더 자주 나오도록 '가중치(Weight)'를 주어 뽑는 방법."**
마치 룰렛의 칸 크기를 다르게 설정하는 것과 같다.

---

## 왜 필요한가? (Why)

**현실 세계의 데이터는 평등하지 않기 때문이다.**

- **동전 던지기:** 앞/뒤가 50:50이다. (`random.choice` 사용)
- **실제 쇼핑몰 주문:**
    - '배송 완료'가 60%로 압도적으로 많고,
    - '주문 취소'나 '반품'은 5% 정도로 적다.

- **문제:** 그냥 `random.choice`를 쓰면 '반품'이 '배송 완료'만큼 자주 나와서 현실성이 떨어진다.
- **해결:** `random.choices`에 **가중치(weights)** 를 줘서 현실을 반영해야 한다.

---

## 실전 문법 분석 (Syntax Breakdown)

```python
import random

status = ['pending', 'approved', 'delivered']
weights = [10, 10, 80]  # 합쳐서 100이 되면 이해하기 편함 (10%, 10%, 80%)

# ⭐️ 핵심 코드
result = random.choices(status, weights=weights, k=1)[0]
```

### `weights=[...]`의 의미

- 각 항목이 뽑힐 **확률의 비중**이다.
- `[10, 10, 80]`이라고 쓰면, 전체 100번 중 `delivered`가 약 80번 나온다는 뜻이다.

### `k=1`의 의미

- "몇 개를 뽑을래?"를 묻는 것이다.
- `k=5`라고 하면 5개를 뽑아준다.
- **주의:** `random.choices`는 무조건 **리스트(List)** 형태로 결과를 준다.
    - `k=1`일 때 결과: `['delivered']` (상자 안에 담김)

### 뒤에 붙은 `[0]`의 의미 ⭐️ (가장 많이 틀리는 곳)

- `random.choices`는 결과를 리스트 `['delivered']`로 주기 때문에, 알맹이인 문자열 `'delivered'`만 꺼내기 위해 인덱싱을 하는 것이다.
- `[0]`을 안 쓰면 나중에 DB에 넣을 때 에러(`Type Error`)가 난다.

---
## 비교: `choice` vs `choices`

|**함수**|**random.choice()**|**random.choices()**|
|---|---|---|
|**특징**|**공평하다** (1/N 확률)|**불공평하다** (가중치 적용 가능)|
|**설정**|`weights` 옵션 없음|`weights` 설정 필수|
|**반환값**|값 1개 (`str`, `int` 등)|**리스트 (`list`)**|
|**비고**|단순 랜덤 뽑기용|현실 데이터 시뮬레이션용|


---
## 예제 코드 (Practical Example)

```python
import random

options = ["당첨", "꽝"]
prob = [1, 99] # 당첨 1%, 꽝 99%

# 100번 뽑아보기
for _ in range(5):
    pick = random.choices(options, weights=prob, k=1)[0]
    print(pick)

# 결과 예상: 대부분 "꽝"이 나오고, 운 좋아야 "당첨"이 나옴.
```

### 💡 팁: 

왜 `choice`가 아니라 `choices`인가요? 
- 영어 문법 차이. 
- -`choice`: (단수) 하나만 고름. 
- `choices`: (복수) 여러 개를 고를 수 있음 (`k`개). 
- 결과도 **리스트**로 나옵니다.