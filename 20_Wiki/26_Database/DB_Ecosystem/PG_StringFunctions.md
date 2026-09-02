---
aliases:
  - btrim
  - ILIKE
  - lower
  - ltrim
  - PostgreSQL 문자열 함수
  - rtrim
  - upper
tags:
  - PostgreSQL
  - SQL
  - string
  - 문자열
related:
  - "[[00_DB_HomePage]]"
  - "[[PG_DML]]"
  - "[[PG_Patterns]]"
  - "[[DB_MigrationPattern]]"
---
# PG_StringFunctions — PostgreSQL 문자열 함수

> [!info]
> PostgreSQL 문자열 처리 함수 모음
> WHERE 조건·SET 절·인덱스 표현식 등 DML 전반에서 사용됨

---

## trim 계열 — 공백·문자 제거

```sql
btrim('  hello  ')          -- 'hello'    양쪽 공백 제거 (both trim)
ltrim('  hello  ')          -- 'hello  '  왼쪽만
rtrim('  hello  ')          -- '  hello'  오른쪽만

-- 특정 문자 제거
btrim('###hello###', '#')   -- 'hello'

-- 표준 SQL 동일 표현
trim(both from '  hello  ') -- 'hello'
trim(leading from '  hi')   -- 'hi'
trim(trailing from 'hi  ')  -- 'hi'
```

```txt
btrim = "both trim" — 양쪽 제거
빈 문자열 필터링 패턴 (공백만 있는 값 제외):
  AND btrim(column) <> ''
  → '   ' 같은 값도 trim 후 '' 가 되어 필터됨
  → NULL은 별도 IS NOT NULL 조건 추가 필요

실전 사례 — Backfill UPDATE:
  AND movie."title" IS NOT NULL
  AND btrim(movie."title") <> ''   ← 공백 문자열 방어
```

---

## 대소문자 변환

```sql
lower('Hello World')   -- 'hello world'
upper('hello world')   -- 'HELLO WORLD'
initcap('hello world') -- 'Hello World' (각 단어 첫 글자 대문자)
```

```txt
실전 사용:
  이메일 저장 전 lower(email) 으로 정규화
  검색 시 lower(name) = lower($1) → 대소문자 무관 비교
```

---

## 길이

```sql
length('hello')          -- 5  (문자 수)
char_length('hello')     -- 5  (표준 SQL — 동일)
octet_length('hello')    -- 5  (바이트 수)
octet_length('안녕')     -- 6  (UTF-8 한글 1자 = 3바이트)
```

---

## 이어붙이기

```sql
concat('foo', 'bar')              -- 'foobar'
concat('foo', NULL, 'bar')        -- 'foobar' (NULL 무시)
concat_ws('-', 'a', 'b', 'c')     -- 'a-b-c'  (구분자 포함)
concat_ws('-', 'a', NULL, 'c')    -- 'a-c'    (NULL 항목 건너뜀)
'foo' || 'bar'                    -- 'foobar' (연산자 방식)
'foo' || NULL                     -- NULL     (NULL 전파 — concat과 차이)
```

| 방식 | NULL 처리 |
|---|---|
| `concat()` | NULL 무시, 나머지 이어붙임 |
| `concat_ws()` | NULL 항목 건너뜀 |
| `\|\|` 연산자 | NULL 전파 → 결과 NULL |

---

## 부분 문자열

```sql
substring('hello world', 1, 5)   -- 'hello'  (1-based index)
left('hello world', 5)            -- 'hello'
right('hello world', 5)           -- 'world'

-- 정규식 추출
substring('2024-01-15' from '\d{4}')  -- '2024'
```

---

## 검색 · 위치

```sql
position('world' in 'hello world')  -- 7  (없으면 0)
strpos('hello world', 'world')      -- 7  (position과 동일)
```

---

## 교체 · 변환

```sql
replace('hello world', 'world', 'SQL')   -- 'hello SQL'

-- 정규식 교체
regexp_replace('abc123', '\d+', 'NUM')   -- 'abcNUM'
regexp_replace('aabbcc', 'b+', 'X', 'g') -- 'aaXcc' (g = 전체 치환)
```

---

## LIKE · ILIKE — 패턴 검색

```sql
-- LIKE — 대소문자 구분
WHERE name LIKE 'kim%'    -- 'kim'으로 시작
WHERE name LIKE '%kim%'   -- 'kim' 포함
WHERE name LIKE '_im'     -- 한 글자 + 'im'

-- ILIKE — 대소문자 무시 (PostgreSQL 전용)
WHERE name ILIKE '%kim%'  -- 'Kim', 'KIM', 'kim' 모두 매칭

-- 패턴 문자 이스케이프
WHERE path LIKE '/100\%' ESCAPE '\'
```

```txt
LIKE  — 표준 SQL, 대소문자 구분
ILIKE — PostgreSQL 전용, 대소문자 무시 (MySQL의 LIKE와 동일한 동작)

성능:
  LIKE 'prefix%'  → B-Tree 인덱스 활용 가능
  LIKE '%infix%'  → 풀 스캔 → 전문 검색이 필요하면 pg_trgm + GIN 인덱스
  ILIKE           → lower() 함수 인덱스 또는 pg_trgm 필요
```

---

## 한눈에

| 함수 | 설명 | 예시 결과 |
|---|---|---|
| `btrim(str)` | 양쪽 공백 제거 | `'hello'` |
| `ltrim(str)` | 왼쪽 공백 제거 | `'hello  '` |
| `rtrim(str)` | 오른쪽 공백 제거 | `'  hello'` |
| `lower(str)` | 소문자 변환 | `'hello'` |
| `upper(str)` | 대문자 변환 | `'HELLO'` |
| `length(str)` | 문자 수 | `5` |
| `concat_ws(sep, ...)` | 구분자 포함 이어붙이기 | `'a-b-c'` |
| `substring(str, start, len)` | 부분 문자열 | `'hello'` |
| `left(str, n)` | 왼쪽 n자 | `'hel'` |
| `replace(str, from, to)` | 문자열 교체 | `'hello SQL'` |
| `position(sub in str)` | 위치 검색 (없으면 0) | `7` |
| `regexp_replace(str, pat, rep)` | 정규식 교체 | `'abcNUM'` |
| `ILIKE` | 대소문자 무시 LIKE | PostgreSQL 전용 |
