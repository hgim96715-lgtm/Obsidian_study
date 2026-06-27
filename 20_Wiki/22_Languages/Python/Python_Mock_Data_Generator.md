---
aliases:
  - MockData
  - 가짜데이터
  - 데이터시뮬레이션
  - 더미데이터
tags:
  - Python
related:
  - "[[Python_Logging]]"
  - "[[Python_Random_Seed]]"
  - "[[Python_DateTime]]"
  - "[[Python_JSON]]"
---
---
## Concept Summary

**"붕어빵 틀(Logic)은 그대로 두고, 앙금(Data)만 바꿔 끼우는 기술."** 
`while True` (무한 반복) + `time.sleep` (주기) 구조는 그대로 유지하고, `random` 설정만 바꾸면 세상의 모든 데이터를 흉내 낼 수 있습니다.

```python
import time
import random
import json

# ==========================================
# [설정] 1. 재료 준비 (여기만 바꾸세요!)
# ==========================================
def generate_one_data():
    # 예시: 쇼핑몰 주문 데이터 만들기
    users = ["user_1", "user_2", "guest", "admin"]
    items = ["MacBook", "Mouse", "Keyboard", "Monitor"]
    
    data = {
        # 1. 범주형 (누가, 무엇을?)
        "user_id": random.choice(users),
        "item_name": random.choice(items),
        
        # 2. 수치형 (얼마나, 몇 개?)
        "price": round(random.uniform(10000, 2000000), -2), # 100원 단위 절사
        "quantity": random.randint(1, 5),
        
        # 3. 시간 (언제?)
        "timestamp": time.strftime("%Y-%m-%d %H:%M:%S")
    }
    return data

# ==========================================
# [엔진] 2. 공장 가동 (건드릴 필요 없음)
# ==========================================
def run_generator(interval=1.0):
    print("🚀 데이터 생성 시작 (Ctrl+C로 종료)")
    try:
        while True:
            # 데이터 1개 생산
            fake_data = generate_one_data()
            
            # (선택) JSON으로 변환해서 출력 or Kafka 전송
            print(json.dumps(fake_data, ensure_ascii=False))
            
            # 대기
            time.sleep(interval)
            
    except KeyboardInterrupt:
        print("\n🛑 생성 종료!")

# 실행
if __name__ == "__main__":
    run_generator(interval=0.5) # 0.5초마다 생성
```

---
## 활용예시

### 상황 1: 주식 가격 시뮬레이션

```python
def generate_one_data():
    stocks = ["AAPL", "GOOGL", "TSLA", "AMZN"]
    return {
        "symbol": random.choice(stocks),
        "price": round(random.uniform(100.0, 900.0), 2),
        "volume": random.randint(10, 1000)
    }
```

### 상황 2: 웹사이트 접속 로그

```python
def generate_one_data():
    methods = ["GET", "POST", "DELETE"]
    urls = ["/home", "/login", "/cart", "/pay"]
    status_codes = [200, 200, 200, 404, 500] # 200이 많이 나오게 확률 조작
    
    return {
        "ip": f"192.168.0.{random.randint(1, 255)}", # 가짜 IP 생성
        "method": random.choice(methods),
        "url": random.choice(urls),
        "status": random.choice(status_codes)
    }
```

---
## `Faker` 라이브러리 (치트키)

`random` 만으로는 "진짜 같은 이름(김철수, John Doe)"이나 "주소", "이메일"을 만들기 어렵습니다. 
실무에서는 **`Faker`** 라는 외부 라이브러리를 쓰면 마법처럼 해결됩니다.

```bash
pip3 install Faker
```

```python
from faker import Faker
fake = Faker('ko_KR') # 한국어 설정

print(fake.name())    # "김영희"
print(fake.address()) # "서울특별시 강남구 테헤란로..."
print(fake.email())   # "test@example.com"
```



