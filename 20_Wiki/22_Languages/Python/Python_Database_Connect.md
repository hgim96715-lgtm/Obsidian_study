---
aliases:
  - DB연결
  - psycopg2
  - 파이썬-SQL
  - 데이터베이스 연동
  - sqlalchemy
  - create_engine
tags:
  - Python
  - SQL
related:
  - "[[00_Python_HomePage]]"
  - "[[PostgreSQL_Setup]]"
  - "[[Python_Functions]]"
  - "[[Docker_Host_vs_Internal_Network]]"
  - "[[Airflow_Hooks]]"
  - "[[Linux_OpenSSL]]"
  - "[[SQL_Subquery]]"
  - "[[Python_JSON]]"
---
# Python_Database_Connect — Python + PostgreSQL 연결

## 한 줄 요약

```
psycopg2 로 PostgreSQL 에 연결하고
SQL 을 실행해서 데이터를 읽고 쓰는 방법
```

---

---

# ① 전체 흐름

```
① DB_CONFIG 준비      → 연결 정보 (.env → os.getenv)
         │
         ▼
② psycopg2.connect()  → DB 연결 (conn 객체)
         │
         ▼
③ conn.cursor()       → 커서 생성 (cur 객체)
         │
         ▼
④ cur.execute(SQL)    → SQL 실행
         │
         ├─ 읽기 (SELECT) → fetchone() / fetchall()
         └─ 쓰기 (INSERT/UPDATE/DELETE) → rowcount → commit()
         │
         ▼
⑤ conn.close()        → 연결 닫기 (finally 블록에서 반드시)
```

---

---

# ② 연결 정보 — DB_CONFIG

```python
import os
from dotenv import load_dotenv

load_dotenv()   # .env 파일 읽기

DB_CONFIG = {
    "host":     os.getenv("POSTGRES_HOST", "localhost"),
    "port":     int(os.getenv("POSTGRES_PORT", "5435")),
    "dbname":   os.getenv("POSTGRES_DB"),       # ⚠️ psycopg2 키: dbname
    "user":     os.getenv("POSTGRES_USER"),
    "password": os.getenv("POSTGRES_PASSWORD"),
}
```

```
⚠️ psycopg2 가 인식하는 키 이름:
  "dbname" ✅   "db" ❌ → KeyError 발생
  "host"   ✅
  "user"   ✅
  "password" ✅
```

---

---

# ③ conn 과 cur — 연결과 커서

```python
import psycopg2

conn = psycopg2.connect(**DB_CONFIG)   # DB 연결
cur  = conn.cursor()                   # 커서 생성
```

```
conn = DB 와 열린 연결 자체 (전화 연결)
cur  = 연결 위에서 SQL 을 실행하는 도구 (전화기)

연결(conn) 하나에 커서(cur) 여러 개 만들 수 있음
실무에서는 보통 커서 하나로 순서대로 사용
```

```
with conn.cursor() as cur:  →  cur.close() 자동 (커서만 닫힘)
with conn:                  →  commit / rollback 관리 (커넥션 닫기 아님)
conn.close()                →  직접 호출 필요 (커넥션 자체를 닫음)
```

---

---

# ④ 안전한 연결 패턴 ⭐️

```python
conn = None   # ← 반드시 None 으로 초기화
try:
    conn = psycopg2.connect(**DB_CONFIG)
    with conn.cursor() as cur:          # with: cur.close() 자동
        cur.execute(sql, params)

        result = cur.fetchone()         # 읽기일 때
        # 또는
        updated = cur.rowcount          # 쓰기일 때

    conn.commit()                       # 쓰기일 때만

except Exception as e:
    print(f"DB 에러: {e}")
    if conn:
        conn.rollback()                 # 에러 시 되돌리기
    raise

finally:
    if conn:
        conn.close()                    # 반드시 닫기
```

```
conn = None 초기화가 필요한 이유:
  psycopg2.connect() 가 실패하면 conn 변수 자체가 만들어지지 않음
  finally 에서 conn.close() 시도 → NameError 발생
  → None 으로 초기화해두면 if conn: 으로 안전하게 체크 가능
```

---

---

# ⑤ SELECT — 읽기

## cur.execute() 기본

```python
# SQL 만 실행 (결과는 아직 없음)
cur.execute("SELECT COUNT(*) FROM raw_exhibitions WHERE is_active = TRUE")

# fetch 로 결과 가져오기
row = cur.fetchone()   # ← 이때 결과를 가져옴
```

## fetchone() — 1행만

```python
cur.execute("SELECT COUNT(*) FROM raw_exhibitions")
row = cur.fetchone()
# row = (47,)  ← 결과가 1행, 1컬럼 → 값 하나짜리 튜플

count = row[0]   # 47

# 두 줄을 한 줄로 자주 씀
count = cur.fetchone()[0]
```

```
fetchone() 은 항상 튜플을 반환

SELECT COUNT(*)        → (47,)
SELECT AVG(a), AVG(b)  → (18347.0, 13160.0)
SELECT name, age       → ('홍길동', 25)

row[0] = 첫 번째 컬럼
row[1] = 두 번째 컬럼
```

```python
# 여러 컬럼 + None 방어 처리
cur.execute("SELECT AVG(sales_price), MAX(sales_price) FROM raw_exhibition_prices")
row = cur.fetchone()           # (18347.0, 30000)

avg_price = int(row[0]) if row and row[0] else 0
max_price = int(row[1]) if row and row[1] else 0
# 집계 대상이 없으면 AVG → None 반환 → int(None) 은 에러
```

## fetchall() — 전체 행

```python
cur.execute("""
    SELECT location, COUNT(*) AS cnt
    FROM raw_exhibitions
    WHERE is_active = TRUE
    GROUP BY location
    ORDER BY cnt DESC
    LIMIT 5
""")
rows = cur.fetchall()
# rows = [('서울', 30), ('부산', 5), ('대구', 3)]
#           ↑ 튜플들의 리스트

for loc, cnt in rows:   # 튜플 언패킹
    print(f"{loc}: {cnt}개")
```

```
fetchone()   → 튜플 1개          ('서울', 30)
fetchall()   → 튜플 리스트       [('서울', 30), ('부산', 5)]
fetchmany(n) → 튜플 n개 리스트   (거의 안 씀)
```

---

---

# ⑥ 파라미터 — %s 플레이스홀더

SQL 안에 값을 직접 문자열로 넣지 않고 `%s` 자리를 만들어두고 따로 넘긴다.

```python
# ❌ f-string 으로 SQL 만들기 → SQL Injection 위험
user_input = "'; DROP TABLE users; --"
cur.execute(f"SELECT * FROM users WHERE name = '{user_input}'")

# ✅ %s 플레이스홀더 → psycopg2 가 안전하게 이스케이프 처리
cur.execute("SELECT * FROM users WHERE name = %s", (user_input,))
```

## 기본 문법

```python
cur.execute(sql, params)
#                ↑
#    반드시 튜플 또는 리스트로 감싸야 함
```

## 값 1개 — 쉼표 필수 ⚠️

```python
# ("001") → 그냥 문자열 "001" (괄호는 우선순위일 뿐)
# ("001",) → 원소 1개짜리 튜플

cur.execute("SELECT * FROM raw_exhibitions WHERE exhibition_id = %s", ("001",))  # ✅
cur.execute("SELECT * FROM raw_exhibitions WHERE exhibition_id = %s", ("001"))   # ❌
```

## 값 여러 개 — 순서대로 매핑

```python
cur.execute(
    "INSERT INTO 테이블 (a, b, c) VALUES (%s, %s, %s)",
    ("001", "페르난도 보테로展", 15000)
    #  ↑           ↑           ↑
    # 첫%s        둘째%s       셋째%s  순서대로
)
```

## 배열 통째로 — ALL / ANY

```python
active_ids = ["001", "002", "003"]

cur.execute(
    "UPDATE raw_exhibitions SET is_active = FALSE WHERE exhibition_id != ALL(%s)",
    (active_ids,)
#    ↑                                                                   ↑
#  리스트 배열 연산자                                          "리스트 1개를 파라미터 1개로"
)
# psycopg2 가 리스트 → PostgreSQL 배열로 자동 변환
# → ARRAY['001', '002', '003']
```

```python
# 헷갈리는 케이스 비교
ids = ["001", "002", "003"]

cur.execute("WHERE id != ALL(%s)",    (ids,))          # ✅ 배열로 변환
cur.execute("WHERE id NOT IN %s",     (tuple(ids),))   # ✅ IN 절용 튜플
cur.execute("WHERE id != ALL(%s)",    ids)             # ❌ 파라미터 3개로 해석
cur.execute("WHERE id = %s",          (ids[0],))       # ✅ 첫 원소만
```

| 상황         | 전달 방법            | SQL 변환 결과                          |
| ---------- | ---------------- | ---------------------------------- |
| 단일 값       | `("001",)`       | `{text}= '001'`                    |
| IN 절       | `(tuple(ids),)`  | `IN ('001','002','003')`           |
| ALL/ANY 배열 | `(ids,)`         | `!= ALL(ARRAY['001','002','003'])` |
| 여러 값 순서대로  | `("a", 1, True)` | 각 `%s` 에 순서대로                      |

---

---

# ⑦ 쓰기 SQL — commit / rollback / rowcount

```python
# INSERT / UPDATE / DELETE 는 반드시 commit() 해야 저장됨
cur.execute("UPDATE raw_exhibitions SET is_active = FALSE WHERE ...")
# ← 여기까지는 임시 상태 (DB 에 반영 안 됨)

conn.commit()    # ← 이때 비로소 DB 에 영구 저장
conn.rollback()  # commit 전 변경사항 취소 (에러 시)
```

## cur.rowcount — 영향받은 행 수

```python
cur.execute("""
    UPDATE raw_exhibitions
    SET is_active = FALSE
    WHERE exhibition_id != ALL(%s) AND is_active = TRUE
""", (active_ids,))

updated = cur.rowcount   # ← execute() 바로 다음에 읽기 (commit 전)
conn.commit()

if updated > 0:
    print(f"  {updated}개 비활성화")
```

```
cur.rowcount 반환값:
  INSERT → 삽입된 행 수
  UPDATE → 변경된 행 수
  DELETE → 삭제된 행 수
  SELECT → -1 (psycopg2 에서는 행 수 미보장)

⚠️ commit() 전에 읽어야 함
```

---

---

# ⑧ execute_values — 대량 INSERT ⭐️

`execute()` 를 반복하는 것보다 훨씬 빠름. 1000건도 SQL 1번으로 처리.

## cur.execute vs execute_values — 인자 개수가 다른 이유

```python
# cur.execute — 커서 메서드 (cur. 뒤에 붙여서 호출)
cur.execute(sql, params)
# cur 은 이미 앞에 있으니까 인자로 안 넘김
# sql + params = 2개

# execute_values — 독립 함수 (cur 을 직접 넘겨야 함)
execute_values(cur, sql, values)
# 함수 자체는 커서를 모름 → 첫 번째 인자로 직접 넘겨야 함
# cur + sql + values = 3개
```

```
비유:
  cur.execute()    → 전화기(cur)에 달린 버튼을 누름
                     전화기가 누구인지 이미 알고 있음

  execute_values() → 외부 기계에 "이 전화기(cur) 로 보내줘" 하고 건네줌
                     어떤 전화기로 보낼지 명시해야 함
```


```python
# 헷갈리는 이유 — 둘 다 SQL 을 실행하는데 생김새가 다름
cur.execute(
    "INSERT INTO t (a, b) VALUES (%s, %s)",
    ("001", "전시A")          # 단건 파라미터 튜플
)

execute_values(
    cur,                      # ← 여기가 추가됨
    "INSERT INTO t (a, b) VALUES %s",
    [("001", "전시A"), ("002", "전시B")]   # 여러 행 리스트
)
```

| |`cur.execute()`|`execute_values()`|
|---|---|---|
|종류|커서 메서드|psycopg2.extras 함수|
|호출 방식|`cur.execute(sql, params)`|`execute_values(cur, sql, values)`|
|커서 위치|`cur.` 앞에|첫 번째 인자|
|용도|단건 SQL|대량 INSERT|
|VALUES 형태|`(%s, %s)`|`%s` (행 묶음 전체)|

python

```python
from psycopg2.extras import execute_values

values = [
    ("26002594", "페르난도 보테로展", "서울"),
    ("26003427", "맥스 시덴토프 개인전", "부산"),
    ("26003180", "인상주의를 넘어", "서울"),
]

sql = """
INSERT INTO raw_exhibitions (exhibition_id, title, location)
VALUES %s
ON CONFLICT (exhibition_id) DO UPDATE SET
    title    = EXCLUDED.title,
    location = EXCLUDED.location
"""

conn = None
try:
    conn = psycopg2.connect(**DB_CONFIG)
    with conn.cursor() as cur:
        execute_values(cur, sql, values)
    conn.commit()
finally:
    if conn:
        conn.close()
```

```
execute_values(cur, sql, values)
                ↑    ↑     ↑
              커서   SQL  행 리스트 (각 행은 튜플)

VALUES %s → VALUES (%s,%s,%s), (%s,%s,%s), (%s,%s,%s) 로 자동 확장
→ 한 번의 쿼리로 전체 삽입

EXCLUDED: ON CONFLICT DO UPDATE 블록 안에서
          "방금 삽입을 시도한 새 값"을 의미
  EXCLUDED.title = 충돌이 난 행의 새 title 값
  → 충돌 시 기존 행을 이 값으로 덮어씀 (UPSERT)
```

```python
# execute() 반복 vs execute_values 속도 비교
for row in values:                                      # ❌ N건 = N번 왕복
    cur.execute("INSERT INTO t VALUES (%s, %s)", row)

execute_values(cur, "INSERT INTO t VALUES %s", values)  # ✅ 1번 왕복
```

---

---

# ⑨ Json — JSONB 컬럼에 데이터 넣기 ⭐️

## 왜 필요한가

```
DB 컬럼 타입이 JSONB 일 때:

  plain str 로 그냥 넣으면:
    psycopg2 → TEXT 타입으로 인식
    PostgreSQL → JSONB 컬럼에 TEXT 넣으려 함 → 타입 불일치 에러 가능

  Json() 으로 감싸면:
    psycopg2 → "이 값은 JSON 이다" 고 명시
    PostgreSQL → JSONB 컬럼에 올바르게 삽입 ✅
```

## 기본 사용법

```python
import json
from psycopg2.extras import Json

# prices_raw 가 JSON 문자열일 때
prices_str = '[{"name": "성인", "price": 23000}]'

# ❌ plain str 그대로 → JSONB 에 타입 불일치 위험
cur.execute("INSERT INTO t (prices_raw) VALUES (%s)", (prices_str,))

# ✅ Json() 래핑 → JSONB 에 안전하게 삽입
cur.execute("INSERT INTO t (prices_raw) VALUES (%s)", (Json(json.loads(prices_str)),))
```

## 입력 타입별 처리

```python
import json
from psycopg2.extras import Json

def to_jsonb(value):
    """str / dict / list / None → Json() 래퍼 또는 None"""
    if value is None:
        return None
    if isinstance(value, str):
        return Json(json.loads(value))   # 문자열이면 먼저 파싱
    return Json(value)                   # dict / list 는 바로 래핑
```

```python
# 사용 예시 (load_to_postgres.py 패턴)
values = [
    (
        ex["exhibition_id"],
        to_jsonb(ex.get("prices_raw")),   # str → Json() 래핑
        ex.get("is_active", True),
    )
    for ex in exhibitions
]
execute_values(cur, sql, values)
```

## JSONB 읽기 — 별도 처리 불필요

```python
cur.execute("SELECT prices_raw FROM raw_exhibitions WHERE exhibition_id = %s", ("26002594",))
row = cur.fetchone()

prices = row[0]   # psycopg2 가 JSONB → Python dict / list 로 자동 변환
# prices = [{"name": "성인", "price": 23000}, ...]

print(prices[0]["name"])    # "성인"
print(prices[0]["price"])   # 23000
```

```
쓰기: str → Json() 래핑 필요 (psycopg2 가 JSONB 로 보내도록)
읽기: 자동 변환 (JSONB → Python dict/list) → 별도 처리 불필요
```

---

---

# ⑩ 커서 종류 — RealDictCursor

기본 커서는 결과를 튜플로 반환. `RealDictCursor` 는 컬럼명을 키로 갖는 dict 로 반환.

```python
from psycopg2.extras import RealDictCursor

with conn.cursor(cursor_factory=RealDictCursor) as cur:
    cur.execute("SELECT exhibition_id, title FROM raw_exhibitions LIMIT 3")
    rows = cur.fetchall()

# 기본 커서:     [('26002594', '페르난도 보테로展'), ...]
# RealDictCursor: [{'exhibition_id': '26002594', 'title': '페르난도 보테로展'}, ...]

for row in rows:
    print(row['title'])     # 컬럼명으로 접근 (인덱스 불필요)
    print(row['exhibition_id'])
```

```
기본 커서     → row[0], row[1]    (순서 알아야 함)
RealDictCursor → row['컬럼명']    (이름으로 접근, 가독성 좋음)

언제 쓰나:
  컬럼이 많아서 인덱스로 관리하기 어려울 때
  API 응답을 dict 로 바로 만들어서 반환할 때
```

---

---

# ⑪ dict 에서 값 꺼내기 — .get() vs []

```python
ex = {"exhibition_id": "001", "title": "전시A"}

ex["title"]               # "전시A"
ex["prices_raw"]          # KeyError ← 없는 키

ex.get("title")           # "전시A"
ex.get("prices_raw")      # None ← 에러 없음
ex.get("is_active", True) # True ← 없으면 기본값
```

```python
# execute_values 값 만들 때 .get() 쓰는 이유:
# 모든 전시가 모든 필드를 가지지 않음 → 없으면 None → DB 에 NULL 저장
values = [
    (
        ex["exhibition_id"],        # PK: 반드시 있어야 해서 []
        ex.get("title"),            # 없으면 None → NULL
        ex.get("prices_raw"),       # 없으면 None → NULL
        ex.get("is_active", True),  # 없으면 기본값 True
    )
    for ex in exhibitions
]
```

---

---

# ⑫ 타입 힌트

```python
from typing import Any, Dict, List
import psycopg2

def get_connection(self) -> psycopg2.extensions.connection:
    return psycopg2.connect(**self.db_config)

def upsert(self, exhibitions: List[Dict[str, Any]]) -> int:
    ...
```

```
psycopg2.extensions.connection
  → psycopg2.connect() 가 반환하는 객체의 타입
  → IDE 자동완성용. 실행에 영향 없음.

List[Dict[str, Any]]
  → 딕셔너리들의 리스트
  → Dict[str, Any]: 키=str, 값=아무 타입

Python 3.10+ 에서는 소문자도 가능
  list[dict[str, Any]]
```

---

---

# ⑬ sqlalchemy — pandas 연동

psycopg2 직접 사용 없이 pandas 와 연결할 때 사용.

```python
from sqlalchemy import create_engine
import pandas as pd

engine = create_engine(
    f"postgresql://{os.getenv('POSTGRES_USER')}:{os.getenv('POSTGRES_PASSWORD')}"
    f"@{os.getenv('POSTGRES_HOST')}:{os.getenv('POSTGRES_PORT')}/{os.getenv('POSTGRES_DB')}"
)

df = pd.read_sql("SELECT * FROM raw_exhibitions", engine)    # DB → DataFrame
df.to_sql("backup", engine, if_exists="append", index=False) # DataFrame → DB
```

|`if_exists`|동작|
|---|---|
|`replace`|테이블 삭제 후 새로 생성|
|`append`|기존 데이터에 추가|
|`fail`|이미 있으면 에러|

---

---

# 임포트 한눈에

```python
import psycopg2                              # 기본 연결
from psycopg2.extras import execute_values   # 대량 INSERT (VALUES %s)
from psycopg2.extras import Json             # JSONB 컬럼 삽입용 래퍼
from psycopg2.extras import RealDictCursor   # 결과를 dict 로

import json                                  # str ↔ dict 변환 (Json() 과 함께)
import os
from dotenv import load_dotenv               # .env 파일 로드
from typing import Any, Dict, List           # 타입 힌트
```