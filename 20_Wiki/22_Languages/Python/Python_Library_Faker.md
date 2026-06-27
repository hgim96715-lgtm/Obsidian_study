---
aliases:
  - Faker
  - 가짜데이터
  - MockData
  - 파이썬_페이커
tags:
  - Python
related:
  - "[[01_Faker_data]]"
  - "[[00_Python_HomePage]]"
  - "[[Python_Mock_Data_Generator]]"
---
## 개념 한 줄 요약 (Concept Summary)

**"테스트를 위한 그럴싸한 가짜 데이터(이름, 주소, 이메일, 텍스트 등)를 무작위로 생성해주는 Python 라이브러리."**
실제 운영 데이터를 쓸 수 없는 개발/테스트 환경에서 필수적으로 사용된다.

---
## 왜 필요한가? (Why)

**문제점:**
- 테스트를 위해 `test_user_1`, `test_user_2` 같은 데이터를 수동으로 넣으면, 데이터의 **다양성**이 부족해 실제 상황을 대변하지 못한다.
- 실명이나 실제 주소를 쓰면 **개인정보보호법(PII)** 위반 위험이 있다.

**해결책:**
- `Faker`를 사용하면 "김철수", "서울특별시 강남구...", "192.168.0.1" 처럼 **패턴은 맞지만 내용은 가짜인 데이터**를 무한대로 생성할 수 있다.
- 데이터 분석 파이프라인(Kafka, Spark) 테스트 시 **데이터 분포(Skew)** 나 **엣지 케이스**를 실험하기 좋다.

---

## 실전 맥락 (Practical Context)

**데이터 엔지니어링 활용:**
1.  **스트리밍 로그 생성:** Kafka 프로듀서에서 1초마다 사용자 행동 로그를 생성해 쏠 때 사용. 
2.  **대용량 적재 테스트:** DB나 Data Warehouse에 100만 건 이상의 데이터를 넣고 성능 테스트를 할 때.
3.  **익명화(Anonymization):** 실제 데이터를 분석할 때 이름을 가짜 이름으로 치환하여 마스킹할 때.

---
## Code Core Points (사용법)

### 라이브러리 설치 

- 로컬 맥북 터미널에 `Faker`를 설치
- **Kafka 연동:** 생성된 데이터를 Kafka로 보낼 때 `kafka-python` 추가 

```bash
# 1. 기본 설치 (Faker만 필요할 때)
pip install faker 

# 2. Kafka 연동 시
pip install faker kafka-python
```

### 기본 사용법 및 한글 설정

`Faker('ko_KR')`을 사용하면 한국어에 특화된 데이터(한국 이름, 한국 주소, 010 번호 등)를 생성한다.

```python
from faker import Faker

# 1. 인스턴스 생성 (Locale 설정: ko_KR)
fake = Faker('ko_KR')

# 2. 데이터 생성
print(fake.name())       # 예: 이영희
print(fake.address())    # 예: 세종특별자치시 강남대로
print(fake.email())      # 예: minsu@example.com
print(fake.job())        # 예: 데이터 엔지니어
```


### 프로젝트 실무 패턴 (Dictionary 형태 생성)

보통 JSON 형태로 데이터를 쏘기 때문에 `dict`를 반환하는 함수로 만들어 쓴다.

```python
import random

def generate_fake_log():
    data = {
        'user_id': fake.uuid4(),         # 고유 ID
        'user_name': fake.name(),        # 이름
        'ip': fake.ipv4(),               # IP 주소
        'event_time': fake.date_time_this_year().isoformat(), # 날짜
        'os': random.choice(['iOS', 'Android', 'Win10'])      # 랜덤 선택
    }
    return data
```

---
## 상세 분석: 주요 Provider (Detailed Analysis)

Faker는 데이터 종류별로 다양한 `Provider`를 제공한다.

|**카테고리**|**메소드**|**예시 데이터**|
|---|---|---|
|**인물**|`name()`|김철수|
|**주소**|`address()`, `city()`, `administrative_unit()`|서울특별시, 경기도...|
|**인터넷**|`ipv4()`, `email()`, `url()`|192.168.1.5|
|**날짜**|`date_time_this_year()`|2024-02-13 14:00:00|
|**텍스트**|`text()`, `sentence()`|임의의 문장 생성|
|**기타**|`uuid4()`, `random_int()`|ID 및 숫자|

>**코드 의미:** `fake.administrative_unit()`
>Faker의 `ko_KR` 설정에서 한국의 행정 구역을 랜덤으로 뽑아줍니다.
> **예시:** `서울특별시`, `경기도`, `부산광역시`, `강원특별자치도`, `제주특별자치도` 등

---
## 초보자가 자주 하는 실수 (Common Misconceptions)

### ① "매번 같은 데이터가 나와요."

- **이유:** `Faker`에도 시드(Seed) 기능이 있다. 
- `Faker.seed(4321)` 처럼 시드를 고정하면 실행할 때마다 똑같은 가짜 데이터가 나온다. 
- (재현 가능한 테스트를 할 때 유용하지만, 무작위가 필요하면 시드를 설정하면 안 된다.)


### ② "속도가 너무 느려요."

- **해결:** `Faker` 객체를 반복문 안에서 계속 생성하면(`fake = Faker()`) 매우 느려진다. 
- **반드시 반복문 밖에서 한 번만 생성(초기화)** 하고, 안에서는 메소드(`fake.name()`)만 호출해야 한다.

### ③ "원하는 데이터 형식이 없어요."

- **해결:** `fake.random_element(elements=['a', 'b', 'c'])` 처럼 리스트에서 뽑거나, 직접 `Provider`를 커스텀해서 만들 수 있다.