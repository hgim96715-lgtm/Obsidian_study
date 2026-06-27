---
aliases:
  - 로깅
  - 로그관리
  - print대신 logging
tags:
  - Python
related:
  - "[[Python_Builtin_Functions]]"
  - "[[00_Python_HomePage]]"
  - "[[Python_Sys_Module]]"
  - "[[Python_Error_Handling]]"
---
# Python_Logging — logging 모듈

## 한 줄 요약

```
print() 대신 logging 을 써야 하는 이유:
  레벨 분류 (DEBUG / INFO / WARNING / ERROR / CRITICAL)
  출력 위치 지정 (파일 / 터미널 / 둘 다)
  포맷 지정 (시각 / 레벨 / 파일명 / 줄 번호)
  실무 파이프라인에서 print() 는 쓰지 않음
```

---

---

# ① logging 레벨 ⭐️

```
레벨 높을수록 심각
기본값은 WARNING → WARNING 이상만 출력됨

DEBUG    (10) = 개발 중 상세 추적 정보
INFO     (20) = 정상 동작 확인 메시지
WARNING  (30) = 주의 필요 (기본값)
ERROR    (40) = 에러 발생 (처리는 가능)
CRITICAL (50) = 치명적 오류 (프로그램 중단 수준)
```

```python
import logging

logging.debug("변수값 확인용")         # DEBUG
logging.info("파이프라인 시작")         # INFO
logging.warning("디스크 용량 80%")     # WARNING
logging.error("DB 연결 실패")          # ERROR
logging.critical("시스템 중단 위기")   # CRITICAL
```

---

---

# ② basicConfig — 기본 설정 ⭐️

```python
import logging

logging.basicConfig(
    level=logging.INFO,                           # INFO 이상만 출력
    format='%(asctime)s %(levelname)s %(message)s'  # 출력 형식
)

logging.info("시작")
# 2026-04-20 10:30:00,123 INFO 시작
```

## format 포맷 옵션

```python
# 자주 쓰는 포맷 속성들
'%(asctime)s'    # 2026-04-20 10:30:00,123
'%(levelname)s'  # INFO / ERROR / WARNING
'%(message)s'    # 실제 로그 메시지
'%(name)s'       # 로거 이름 (getLogger 의 이름)
'%(filename)s'   # 파일명 (producer.py)
'%(lineno)d'     # 줄 번호 (42)
'%(funcName)s'   # 함수명 (run)

# 실전에서 자주 쓰는 포맷
format = '%(asctime)s [%(levelname)s] %(name)s - %(message)s'
# 2026-04-20 10:30:00,123 [INFO] producer - Producer 시작
```

## stream — 출력 위치 지정 ⭐️

```python
import sys
import logging

# 터미널(stdout)으로 출력 — Docker 환경에서 필수
logging.basicConfig(
    level=logging.INFO,
    format='%(asctime)s %(levelname)s %(message)s',
    stream=sys.stdout    # ← 파일 아닌 터미널로
)
```

```
stream=sys.stdout 이 필요한 이유:
  기본값은 stderr 로 출력
  Docker 는 stdout 을 캡처해서 docker logs 로 보여줌
  stream=sys.stdout 없으면 docker logs 에 안 보일 수 있음

  stream=sys.stdout 있으면:
  docker logs 컨테이너명  → 실시간 로그 확인 가능 ✅
```

---

---

# ③ getLogger — 로거 객체 생성 ⭐️

```
basicConfig = 전역 설정 (모듈 전체에 적용)
getLogger   = 특정 이름의 로거 객체 (모듈별 분리 관리)

실무에서는 getLogger(__name__) 패턴 사용
```

```python
import logging

# 이름 있는 로거 생성
logger = logging.getLogger(__name__)
# __name__ = 현재 모듈 이름 (예: 'producer', 'main', '__main__')

logger.info("Producer 시작")
logger.error("연결 실패")
logger.warning("재시도 중")
```

## **name** 의 의미

```python
# producer.py 파일에서
logger = logging.getLogger(__name__)
# → __name__ = 'producer'

# main.py 에서 실행하면
# → __name__ = '__main__'

# airflow/dags/my_dag.py 에서
# → __name__ = 'airflow.dags.my_dag'
```

```
getLogger(__name__) 패턴의 장점:
  어느 모듈에서 로그가 찍혔는지 이름으로 구분
  모듈별 레벨 다르게 설정 가능
  로그 필터링 시 편함
```

---

---

# ④ basicConfig + getLogger 조합 ⭐️

```python
import sys
import logging

# 1. 전역 설정 (한 번만)
logging.basicConfig(
    level=logging.INFO,
    format='%(asctime)s [%(levelname)s] %(name)s - %(message)s',
    stream=sys.stdout
)

# 2. 모듈별 로거 생성
logger = logging.getLogger(__name__)

# 3. 사용
logger.info("Producer 시작")
logger.error("Kafka 연결 실패: %s", str(e))
logger.warning("재시도 %d / 3", retry_count)
```

## 포맷 문자열 방식

```python
# f-string 보다 % 포맷 권장 (lazy evaluation)
logger.info("처리 건수: %d", count)            # ✅ 권장
logger.info(f"처리 건수: {count}")             # ❌ 레벨 미달해도 문자열 생성

# 왜 % 가 나은가:
# DEBUG 레벨 로그인데 level=INFO 로 설정 시
# f-string = 출력 안 해도 문자열 생성 (낭비)
# %s 포맷 = 출력할 때만 문자열 생성 (효율)
```

---

---

# ⑤ 파일 + 터미널 동시 출력 (Handler)

```python
import sys
import logging

logger = logging.getLogger(__name__)
logger.setLevel(logging.DEBUG)   # 로거 레벨 설정

# 핸들러 1 — 터미널 출력
stream_handler = logging.StreamHandler(sys.stdout)
stream_handler.setLevel(logging.INFO)

# 핸들러 2 — 파일 저장
file_handler = logging.FileHandler('app.log', encoding='utf-8')
file_handler.setLevel(logging.DEBUG)   # 파일엔 DEBUG 부터

# 포맷 설정
fmt = logging.Formatter('%(asctime)s [%(levelname)s] %(name)s - %(message)s')
stream_handler.setFormatter(fmt)
file_handler.setFormatter(fmt)

# 로거에 핸들러 추가
logger.addHandler(stream_handler)
logger.addHandler(file_handler)

logger.info("실행 시작")   # 터미널 + 파일 둘 다 기록
logger.debug("변수 확인") # 파일만 기록 (터미널은 INFO 이상만)
```

---

---

# ⑥ 실전 패턴 — 데이터 파이프라인

## Kafka Producer 로거

```python
import sys
import logging

# 파일 상단에 한 번만 설정
logging.basicConfig(
    level=logging.INFO,
    format='%(asctime)s %(levelname)s %(message)s',
    stream=sys.stdout   # Docker 필수
)

logger = logging.getLogger(__name__)


class TrainProducer:
    def __init__(self):
        logger.info("TrainProducer 초기화 시작")

    def produce(self, data):
        try:
            # 처리 로직
            logger.info("데이터 전송 완료: %d건", len(data))
        except Exception as e:
            logger.error("전송 실패: %s", str(e))
            raise
```

## Airflow DAG 로거

```python
import logging

# Airflow 에서는 기본 로거 사용
logger = logging.getLogger('airflow.task')

def my_task(**kwargs):
    logger.info("태스크 시작: %s", kwargs['ds'])
    # 처리...
    logger.info("태스크 완료")
```

---

---

# print() vs logging 비교 ⭐️

|구분|print()|logging|
|---|---|---|
|레벨 구분|❌ 없음|✅ DEBUG~CRITICAL|
|출력 위치|터미널만|파일 / 터미널 / 둘 다|
|시각 기록|❌|✅ 자동|
|레벨별 필터|❌|✅ level 설정|
|실무 파이프라인|❌ 비권장|✅ 표준|

---

---

# 자주 하는 실수

|실수|원인|해결|
|---|---|---|
|로그가 안 나옴|레벨 설정 문제|`basicConfig(level=logging.DEBUG)`|
|docker logs 에 안 보임|stderr 출력|`stream=sys.stdout` 추가|
|중복 로그 출력|핸들러 중복 추가|핸들러 추가 전 `logger.handlers` 확인|
|f-string 과 % 혼용|습관|% 포맷 통일 권장|
|basicConfig 효과 없음|이미 핸들러 있음|`force=True` 추가 또는 `logger.handlers.clear()`|