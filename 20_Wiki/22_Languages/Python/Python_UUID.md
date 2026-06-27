---
aliases:
  - UUID
  - 고유식별자
  - Unique ID
  - 범용고유식별자
tags:
  - Python
related:
  - "[[00_Python_HomePage]]"
  - "[[Python_JSON]]"
  - "[[Python_Library_Faker]]"
---
## 개념 한 줄 요약 (Concept Summary)

**"전 우주에서 겹칠 확률이 0에 수렴하는, 128비트짜리 긴 무작위 고유 ID를 생성하는 표준 라이브러리."**

(예: `550e8400-e29b-41d4-a716-446655440000`)

---

## 왜 필요한가? (Why)

**문제점:**
- 기존의 **Auto Increment (1, 2, 3...)** 방식은 데이터베이스가 한 대일 때는 좋지만, **분산 환경(Distributed System)** 에서는 치명적이다.
- A서버에서도 1번을 만들고, B서버에서도 1번을 만들면 나중에 합칠 때 **충돌(Collision)** 이 난다.

**해결책:**
- 중앙에서 번호를 관리해주는 관리자 없이도, 각자 알아서 막 만들어도 **절대 겹치지 않는 ID**가 필요하다.
- UUID는 **시간, 랜덤, 기기 정보** 등을 섞어서 만들기 때문에 충돌 확률이 사실상 없다.

---

## 실전 맥락 (Practical Context)

**데이터 엔지니어링 활용:**
1.  **로그 추적 (Tracing ID):** Kafka로 들어오는 수억 건의 로그에 각각 고유한 이름표를 붙여서, 시스템 끝까지 추적할 때.
2.  **NoSQL Primary Key:** Cassandra, DynamoDB, MongoDB 등 분산 DB에서 기본 키(PK)로 사용.
3.  **파일 저장:** S3나 MinIO에 파일을 저장할 때, 파일명이 겹쳐서 덮어쓰기 되는 것을 방지하기 위해 파일명으로 씀. (`uuid.jpg`)

---

## Code Core Points (사용법)

### 1. 가장 많이 쓰는 `uuid4` (완전 랜덤)
실무에서는 99% 확률로 **`uuid4()`** 를 쓴다. (랜덤 기반이라 개인정보 이슈가 없음)

```python
import uuid

# 1. UUID 객체 생성
my_id = uuid.uuid4()
print(my_id) 
# 출력: a8098c1a-f86e-11da-bd1a-00112444be1e (객체 형태)

# 2. 문자열 변환 (⭐️ 중요)
# JSON으로 보내거나 DB에 넣으려면 꼭 str()로 감싸야 함!
str_id = str(my_id)
print(type(str_id)) # <class 'str'>
```

---
## 버전별 차이

|**버전**|**생성 방식**|**특징**|**비고**|
|---|---|---|---|
|**v1**|타임스탬프 + MAC 주소|언제, 누가 만들었는지 알 수 있음|**개인정보 노출 위험** ⚠️|
|**v3/v5**|네임스페이스 + 이름(Hash)|입력값이 같으면 항상 같은 UUID 나옴|특정 목적용|
|**v4**|**완전 무작위 (Random)**|유추 불가능, 개인정보 안전|**표준 (가장 많이 씀)** ✅|


---
## Import

- `uuid`는 파이썬을 설치하면 기본으로 따라오는 **내장 라이브러리(Built-in Library)** 이지만, 자동으로 로드되지는 않는다.

```python
import uuid

def get_unique_id():
    # 데이터베이스 PK로 쓸 36자짜리 문자열 반환
    return str(uuid.uuid4())
```

---

## 초보자가 자주 하는 실수 (Common Misconceptions)

### ① "그냥 `uuid.uuid4()`만 보내면 에러 나요."

- **이유:** `uuid.uuid4()`가 반환하는 건 문자열이 아니라 `UUID`라는 **특수한 객체(Class)** 다.
- **해결:** JSON 직렬화(`json.dumps`)를 하거나 Kafka로 보낼 때는 반드시 **`str(uuid.uuid4())`** 로 문자열로 바꿔줘야 한다.