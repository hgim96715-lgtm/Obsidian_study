---
aliases:
  - cut
  - sort
  - uniq
  - wc
  - awk
  - sed
  - join
  - paste
  - col
  - tr
tags:
  - Linux
related:
  - "[[00_Linux_HomePage]]"
  - "[[Linux_Search_Filter]]"
  - "[[Linux_Text_Commands]]"
  - "[[Linux_Redirect]]"
---
# Linux_Text_Processing — 텍스트 처리

# 한 줄 요약

```txt
cut   = 컬럼 추출
sort  = 정렬
uniq  = 중복 제거
wc    = 줄/단어/글자 수 세기
파이프 | 로 조합해서 강력한 데이터 처리
```

---

---

# cut — 컬럼 추출 ⭐️

```txt
cut [옵션] 파일
  -d 구분자   필드 구분 문자 지정
  -f 번호     추출할 필드 번호
  -c 범위     문자 위치로 추출 (고정 폭 데이터)
```

```bash
# /etc/passwd 에서 사용자명(1번째)과 홈디렉토리(6번째) 추출
cut -d: -f1,6 /etc/passwd
# 결과: root:/root
#       labex:/home/labex

# 처음 5줄만 (파이프 조합)
cut -d: -f1,6 /etc/passwd | head -n 5

# CSV 파일 특정 컬럼 추출
cut -d, -f1,3 data.csv    # 쉼표 구분, 1번째 3번째 컬럼
```

```txt
-d 뒤에 구분자:
  -d:   콜론(:) 구분
  -d,   쉼표(,) 구분
  -d' ' 공백 구분

-f 뒤에 필드 번호:
  -f1      1번째
  -f1,3    1번째, 3번째
  -f1-3    1~3번째
  -f3-     3번째부터 끝까지
```

## -c — 문자 위치로 추출 (고정 폭 데이터)

```bash
# 고정 폭 데이터 예시:
# John  25 USA
# Alice 30 KOR

cut -c1-5 file.txt        # 1~5번째 문자 → 이름
cut -c7-8 file.txt        # 7~8번째 문자 → 나이
cut -c10- file.txt        # 10번째부터 끝까지
cut -c1-5,10-12 file.txt  # 여러 범위
```

```txt
-c vs -f:
  -f  구분자로 나뉜 필드 기준  (CSV, /etc/passwd 등)
  -c  문자 위치 기준           (고정 폭 리포트, 로그 파일 등)
```

## grep 과 조합 ⭐️

```bash
# 가격이 20달러 이상인 도서 제목만 추출
grep -E ',[2-9][0-9]\.[0-9]{2}$' books.txt | cut -d ',' -f 2
# grep 으로 조건 필터 → cut 으로 원하는 컬럼 추출
```

## 추가 옵션

```bash
# -s — 구분자 없는 줄 억제
cut -d, -f2 -s data.csv

# --complement — 선택한 것 제외하고 나머지 출력
cut -d, -f2 --complement data.csv

# --output-delimiter — 출력 구분자 변경
cut -d: -f1,6 /etc/passwd --output-delimiter=','
```

---

---

# sort — 정렬 ⭐️

```bash
sort file.txt                    # 알파벳 순 정렬 (기본)
sort -r file.txt                 # 역순 정렬
sort -n numbers.txt              # 숫자 순 정렬
sort -t: -k3 -n /etc/passwd      # : 구분자 / 3번째 필드 기준 / 숫자순
du -h ~ | sort -rh               # 크기 단위 정렬 (1K, 10M, 2G)

sort -u file.txt                 # 정렬 + 중복 제거
sort -f file.txt                 # 대소문자 구분 없이 정렬
sort -b file.txt                 # 앞 공백 무시하고 정렬
sort -c file.txt                 # 이미 정렬됐는지 확인
sort -o output.txt file.txt      # 결과를 파일로 저장
```

```txt
-r   역순
-n   숫자 순
-h   human-readable 단위 인식 (1K, 10M)
-t:  : 를 구분자로
-k3  3번째 필드 기준
-u   중복 제거하며 정렬 (unique)
-f   대소문자 구분 안 함
-b   앞부분 공백 무시
-c   정렬 여부 확인만
-o   결과를 파일로 저장

-o 가 유용한 이유:
  sort file.txt > file.txt   ❌ 원본 파일이 먼저 비워짐 → 데이터 손실
  sort -o file.txt file.txt  ✅ 안전하게 같은 파일에 덮어쓰기

-u 는 sort | uniq 를 한 번에:
  sort file.txt | uniq  →  sort -u file.txt  (동일한 결과)
```

## -k 필드 정렬 이해하기 ⭐️

```bash
# colors.txt:
# 1 red
# 2 yellow
# 3 red

sort -k2 colors.txt
# 1 red       ← 2번째 필드 r
# 3 red       ← 2번째 필드 r (같은 red 끼리는 원본 순서 유지)
# 2 yellow    ← 2번째 필드 y
```

```txt
-k2 = 2번째 필드(공백 기준) 알파벳 순 정렬
1번째 필드(숫자)는 무시됨

헷갈리는 포인트:
  숫자(1,2,3) 가 정렬 기준이 아님
  2번째 컬럼(단어) 만 보고 정렬
  그래서 3 red 가 2 yellow 보다 앞에 오는 것
```

## -k 여러 필드 기준 정렬 ⭐️

```bash
# 2번째 필드 숫자 역순, 같으면 3번째 필드 숫자 역순
sort -k2n -k3nr student_records.txt

# -k 옵션에 정렬 방식을 붙여서 씀
#   -k2n    2번째 필드 숫자(n) 오름차순
#   -k2nr   2번째 필드 숫자(n) 역순(r)
```

---

---

# uniq — 중복 제거 ⭐️

```bash
sort file.txt | uniq           # 중복 제거 (sort 먼저 필수)
sort file.txt | uniq -c        # 발생 횟수 포함
sort file.txt | uniq -d        # 중복된 것만 표시
sort file.txt | uniq -u        # 고유한 행만 (딱 1번만 등장)
sort file.txt | uniq -i        # 대소문자 무시하고 비교
sort file.txt | uniq -f 1      # 첫 번째 필드 건너뛰고 비교
sort file.txt | uniq -s 3      # 처음 3글자 건너뛰고 비교
```

```bash
# 실전 — 장르별 구매 횟수 집계 후 많은 순 정렬
sort purchases.txt | uniq -c | sort -rn

# /etc/passwd 에서 셸별 사용자 수
cut -d: -f7 /etc/passwd | sort | uniq -c
```

```txt
⚠️ uniq 주의:
  인접한 중복만 제거
  → 반드시 sort 먼저 한 후 uniq

입력/출력 파일 직접 지정:
  uniq -c sorted.txt output.txt
           ↑ 입력       ↑ 출력
```

---

---

# wc — 개수 세기 ⭐️

```bash
wc -l /etc/passwd          # 줄 수
wc -w file.txt             # 단어 수
wc -c file.txt             # 바이트 수
wc -m file.txt             # 문자 수 (멀티바이트 정확히 셈)
wc -L file.txt             # 가장 긴 줄의 길이

grep "ERROR" app.log | wc -l   # 에러 줄 수
ls /etc | wc -l                # /etc 파일 수
```

```txt
-c vs -m:
  echo "안녕" | wc -c  → 7  (한글 1글자 = 3바이트 × 2 + 줄바꿈)
  echo "안녕" | wc -m  → 3  (한글 2글자 + 줄바꿈)
  영어만 있으면 결과 동일

-L 활용:
  wc -L *.py  → 파일별로 가장 긴 줄 길이 확인
                코딩 컨벤션(80자 제한 등) 체크할 때 유용
```

---

---

# 파이프 조합 읽는 법 ⭐️

```txt
ls -l /etc | grep '^d' | wc -l

왼쪽 → 오른쪽 순서로 읽기:
  ls -l /etc      /etc 목록 전체
    ↓ |
  grep '^d'       d 로 시작하는 줄만 (디렉토리)
    ↓ |
  wc -l           남은 줄 수 = 디렉토리 개수
```

## 실전 조합 패턴 ⭐️

```bash
# /etc 디렉토리 개수
ls -l /etc | grep '^d' | wc -l

# 에러 종류별 개수 (많은 순)
grep "ERROR" app.log | sort | uniq -c | sort -rn | head -10

# 가장 큰 폴더 top 10
du -h ~ | sort -rh | head -n 10

# /etc/passwd 에서 셸 종류별 사용자 수
cut -d: -f7 /etc/passwd | sort | uniq -c | sort -rn
```

---

---

# awk — 고급 텍스트 처리 ⭐️

## 기본 구조

```txt
awk '조건 { 동작 }' 파일

  조건 없으면 모든 줄 처리
  동작 없으면 조건 맞는 줄 출력
```

## $0 $1 $2 — 필드 변수 ⭐️

```bash
awk '{print $1}' file.txt       # 1번째 컬럼
awk '{print $2}' file.txt       # 2번째 컬럼
awk '{print $0}' file.txt       # 줄 전체
awk '{print $1, $2}' file.txt   # 1번째, 2번째

# 구분자 변경 -F
awk -F: '{print $1}' /etc/passwd   # : 기준
awk -F, '{print $2}' data.csv      # , 기준 (CSV)
```

```txt
$0  줄 전체
$1  공백 기준 1번째 단어
$2  2번째 단어
...
```

## 조건 — 특정 줄만 처리 ⭐️

```bash
awk '$2 > 28 {print $1 " is over 28"}' file.txt
#   ↑ 조건        ↑ 동작

awk '$3 == "USA" {print $1}' file.txt   # 문자열 조건
awk '/Alice/ {print $0}' file.txt       # 패턴 매칭
awk '$4 == "POST" && $6 >= 400 {print $0}' file.txt
```

## NR — 줄 번호 ⭐️

```bash
awk 'NR > 1 {print $1}' file.txt        # 1번째 줄(헤더) 제외
awk 'NR == 2 {print $0}' file.txt       # 정확히 2번째 줄
awk 'NR >= 2 && NR <= 4 {print}' file.txt  # 2~4번째 줄
```

## END — 모든 줄 처리 후 실행 ⭐️

```bash
awk 'NR > 1 {sum += $2} END {print "평균:", sum/(NR-1)}' file.txt
awk '{count[$6]++} END {for (code in count) print code, count[code]}' file.txt | sort -n
```

```txt
블록 종류:
  BEGIN { ... }  파일 읽기 전 실행 (초기화)
  { ... }        각 줄마다 실행 (메인 로직)
  END { ... }    파일 다 읽은 후 실행 (집계 출력)
```

## 연관 배열 — count[$6]++ 패턴 ⭐️

```txt
awk 의 배열은 key → value 로 저장하는 구조 (JS 객체 / Python dict 와 같음)
key 를 처음 쓰면 자동으로 0 초기화 → 미리 선언 불필요

count[$6]++  의 의미:
  $6 의 값(예: HTTP 상태코드 "200", "404")을 key 로 사용
  처음 등장하면 0 → 1, 다음번엔 2, 3... 쌓임
  → 줄을 다 읽고 나면 count["200"] = 142, count["404"] = 7 처럼 채워짐
```

```bash
# access.log — HTTP 상태코드($6) 별 요청 횟수 집계
awk '{count[$6]++} END {for (code in count) print code, count[code]}' access.log | sort -n
# → 200 142
# → 401 3
# → 404 7
# → 500 1

# IP 별 요청 횟수 (많은 순)
awk '{count[$1]++} END {for (ip in count) print ip, count[ip]}' access.log | sort -k2 -rn

# 두 필드 조합 key — "이 IP 가 이 상태코드를 몇 번?"
awk '{count[$1"→"$6]++} END {for (k in count) print k, count[k]}' access.log
```

```txt
for (code in count)
  count 배열의 key 들을 순서 없이 하나씩 꺼냄
  → 순서 보장 없음 → | sort -n 으로 정렬 필수

sort 옵션:
  | sort -n       첫 번째 컬럼(key) 숫자 오름차순
  | sort -k2 -rn  두 번째 컬럼(횟수) 숫자 내림차순 (많은 것 먼저)
```

## NF — 필드 개수

```bash
awk '{print NF}' file.txt               # 각 줄의 필드 수 출력
awk '{print $NF}' file.txt              # 마지막 필드 출력 ($NF = $마지막번호)
awk '{print $(NF-1)}' file.txt          # 끝에서 두 번째 필드
awk 'NF == 3 {print $0}' file.txt       # 필드가 정확히 3개인 줄만
awk 'NF > 0 {print $0}' file.txt        # 빈 줄 제거 (필드 1개 이상인 줄만)
```

```txt
NF  현재 줄의 필드 수 (Number of Fields)
NR  현재 줄 번호     (Number of Records)

$NF    NF 가 숫자라서 $NF = $마지막필드번호 → 마지막 필드 값이 됨
$(NF-1) → 끝에서 두 번째 필드
```

## -v — 외부 변수 전달

```bash
# 스크립트 안에서 쉘 변수를 직접 쓰면 확장이 안 됨 → -v 로 전달
threshold=100
awk -v limit=$threshold '$2 > limit {print $1}' file.txt
#       ↑ awk 변수명=쉘변수

# 여러 변수
awk -v min=10 -v max=100 '$2 >= min && $2 <= max {print}' file.txt

# 구분자도 변수로
awk -v OFS=',' '{print $1, $2, $3}' file.txt   # 출력 구분자 변경
#      ↑ OFS = Output Field Separator
```

```txt
-v 없이 쉘 변수를 따옴표 안에 넣으면:
  awk '$2 > $threshold'   ← ❌ $threshold 가 awk 변수로 해석됨 (값 없음 → 0)
  awk -v t=$threshold '$2 > t'   ← ✅
```

## -f — 스크립트 파일 실행

```bash
# 긴 awk 코드는 파일로 저장해서 실행
cat report.awk
# BEGIN { print "=== 리포트 ===" }
# $3 == "ERROR" { count++ }
# END { print "에러 수:", count }

awk -f report.awk app.log
# → === 리포트 ===
# → 에러 수: 42
```

```txt
-f 가 유용한 경우:
  한 줄에 다 쓰기 너무 긴 복잡한 처리
  여러 파일에 반복해서 쓰는 공통 분석 스크립트
  버전 관리(git)에 넣어야 하는 경우
```

## 내장 함수

```bash
# 문자열 함수
awk '{print length($1)}' file.txt          # 문자열 길이
awk '{print toupper($1)}' file.txt         # 대문자 변환
awk '{print tolower($1)}' file.txt         # 소문자 변환
awk '{print substr($1, 2, 3)}' file.txt    # 부분 문자열 (2번째부터 3글자)
awk '{gsub(/foo/, "bar"); print}' file.txt # 전체 치환 (sed 's/foo/bar/g' 와 동일)
awk '{sub(/foo/, "bar"); print}' file.txt  # 첫 번째만 치환

# 수학 함수
awk '{print int($1)}'   file.txt   # 정수 변환 (소수점 버림)
awk '{print sqrt($1)}'  file.txt   # 제곱근
awk '{print $1 ^ 2}'    file.txt   # 거듭제곱
awk 'BEGIN {print sin(3.14/2)}'    # sin (라디안)
awk 'BEGIN {srand(); print rand()}' # 0~1 난수 (srand 로 시드 초기화)

# split — 필드를 다시 쪼개기
awk '{n = split($3, parts, "-"); print parts[1], parts[2]}' file.txt
#    ↑ $3 을 '-' 기준으로 쪼개서 parts 배열에 저장, 개수 반환
```

```txt
자주 쓰는 것만:
  length(s)         문자열 길이
  substr(s, start, len)  부분 문자열
  toupper(s) / tolower(s)  대소문자 변환
  gsub(패턴, 교체)  전체 치환 (g = global)
  sub(패턴, 교체)   첫 번째만 치환
  split(s, arr, sep)  문자열을 배열로 분리
  int(n)            정수 변환
```

---

---

# sed — 텍스트 치환 ⭐️

## 기본 구조

```txt
sed 's/패턴/교체/플래그' 파일

s    = substitute (치환)
g    = global (줄 전체 적용, 없으면 첫 번째만)
i    = 대소문자 무시
```

```bash
sed 's/Hello/Hi/' file.txt         # 첫 번째만 치환
sed 's/Hello/Hi/g' file.txt        # 전체 치환
sed -i 's/Hello/Hi/g' file.txt     # 파일 직접 수정
sed -e 's/a/b/g' -e 's/c/d/g' f   # 여러 치환 동시에
```

## 경로 포함 시 구분자 변경 ⭐️

```bash
# / 가 포함되면 구분자와 겹침
sed 's/\/home\/user/\/home\/new/g' paths.txt   # 복잡

# # 또는 | 로 변경 (권장)
sed 's#/home/user#/home/new#g' paths.txt       # 깔끔
```

## 줄 삭제 / 삽입 / 추가

```bash
sed '2d' file.txt           # 2번째 줄 삭제
sed '/ERROR/d' file.txt     # ERROR 포함 줄 삭제
sed '$d' file.txt           # 마지막 줄 삭제
sed '1i\First line' file.txt   # 1번째 줄 앞에 삽입
sed '$a\Last line' file.txt    # 마지막 줄 뒤에 추가
```

---

---

# tr — 문자 치환 & 삭제 ⭐️

```txt
tr = translate
문자를 하나씩 대응시켜 치환 / 삭제 / 압축
파일 직접 못 받음 → 파이프로 입력 받음
```

## 기본 치환

```bash
# 소문자 → 대문자
echo 'hello world' | tr 'a-z' 'A-Z'
echo 'hello world' | tr '[:lower:]' '[:upper:]'
# HELLO WORLD

# 특정 문자 치환 (한 문자씩 대응)
echo 'hello world' | tr 'ol' 'OL'
# heLLO wOrLd  ← o→O, l→L
```

## 범위 조합 — 문자 순환 / 암호화 ⭐️

```bash
# tr 'b-za-a' 'a-z'
# b-z (25개) + a (1개)  →  a-z (26개)

echo 'hello' | tr 'b-za-a' 'a-z'
# gdkkn  ← 각 문자를 알파벳 한 칸 앞으로 이동

# 반대 방향 (한 칸 뒤로)
echo 'hello' | tr 'a-z' 'b-za'
# ifmmp
```

```txt
범위 조합 원리:

  tr 'b-za-a' 'a-z'
  b-z = b,c,...,z (25개)  →  a-y
  a   = a         (1개)   →  z
  합계 26개

  대응: b→a, c→b, ... z→y, a→z
  → 알파벳을 한 칸 앞으로 순환 (ROT-25)

범위를 쪼개서 이어 붙이면 순환(wrap-around) 효과
앞뒤 글자 수만 맞으면 어떤 조합이든 가능
```

```bash
# ROT13 — 13칸 이동 (두 번 적용하면 원래대로)
echo 'hello' | tr 'A-Za-z' 'N-ZA-Mn-za-m'
# uryyb

echo 'uryyb' | tr 'A-Za-z' 'N-ZA-Mn-za-m'
# hello  ← 같은 명령 두 번 = 원래대로

# 구조:
# A-Z → N-ZA-M  (N부터 Z까지 13개 + A부터 M까지 13개)
# a-z → n-za-m  (소문자도 동일)
```

## 문자 삭제 — -d ⭐️

```bash
echo 'hello labex' | tr -d 'olh'
# e abex  ← o, l, h 전부 삭제

echo 'abc123def' | tr -d '0-9'
# abcdef  ← 숫자만 삭제

# 숫자만 남기기 (-c 보수)
echo 'abc123def456' | tr -cd '0-9'
# 123456  ← 0-9 외 나머지 삭제
```

```txt
-d  지정한 문자 삭제
-c  보수 (complement) — 지정한 문자 외 나머지 선택
-cd 조합 → 지정한 문자만 남기고 나머지 삭제
```

## 연속 중복 압축 — -s ⭐️

```bash
echo 'hello' | tr -s 'l'
# helo  ← ll → l

echo 'hello   world' | tr -s ' '
# hello world  ← 공백 여러 개 → 하나로
```

## 문자 클래스

```txt
[:lower:]   소문자 전체 (a-z)
[:upper:]   대문자 전체 (A-Z)
[:digit:]   숫자 전체   (0-9)
[:space:]   공백 문자
[:alpha:]   영문자 전체
[:punct:]   모든 문장 부호
```

---

---

# col — 탭 → 공백 변환

```bash
cat /etc/protocols | col -x    # 탭 → 공백 변환
```

---

---

# join — 두 파일 공통 필드 기준 결합 ⭐️

```bash
SQL JOIN 과 같은 개념 — 공통 키 기준으로 두 파일 합치기
기본: 첫 번째 필드 기준 / 정렬 필수
```

```bash
# employees.txt        salaries.txt
# 1 Alice HR           1 50000
# 2 Bob  IT           2 60000
# 3 Carol HR          3 55000

join employees.txt salaries.txt
# 1 Alice HR 50000
# 2 Bob IT 60000
# 3 Carol HR 55000
```

```txt
⚠️ join 주의:
  결합 기준 필드로 정렬되어 있어야 함
  → sort 먼저 후 join
  sort -k1 employees.txt > emp_sorted.txt
  sort -k1 salaries.txt  > sal_sorted.txt
  join emp_sorted.txt sal_sorted.txt
```

## 주요 옵션 ⭐️

```bash
# -1 -2 — 결합 기준 필드 지정
join -1 1 -2 1 employees.txt salaries.txt
# employees 의 1번째 = salaries 의 1번째 기준

# -j — -1 FIELD -2 FIELD 와 동일 (간단 표기)
join -j 1 employees.txt salaries.txt   # 둘 다 1번째 필드 기준
# → join -1 1 -2 1 과 같음

# -t — 구분자 지정
join -t, employees.csv salaries.csv    # CSV 파일
join -t: file1.txt file2.txt           # : 구분자

# -i — 대소문자 무시하고 비교
join -i employees.txt salaries.txt
# Alice 와 alice 를 같은 키로 인식

# -e — 누락된 필드를 대체 문자열로 채우기
join -a 1 -e 'N/A' employees.txt salaries.txt
# 매칭 안 된 필드 → N/A 로 출력
# -a 와 함께 써야 의미 있음 (매칭 없는 줄이 있어야 채울 곳이 생김)

# -a — 매칭 안 된 줄도 출력 (SQL LEFT/RIGHT JOIN)
join -a 1 employees.txt salaries.txt    # 파일1 기준 (LEFT JOIN)
join -a 2 employees.txt salaries.txt    # 파일2 기준 (RIGHT JOIN)
join -a 1 -a 2 employees.txt salaries.txt  # 전체 (FULL JOIN)

# -v — 짝 없는 줄만 출력 (-a 와 반대)
join -v 1 employees.txt salaries.txt    # 파일1 에만 있는 줄
join -v 2 employees.txt salaries.txt    # 파일2 에만 있는 줄
join -v 1 -v 2 employees.txt salaries.txt  # 양쪽 다 짝 없는 줄
```

## -o — 출력 필드 선택 ⭐️

```bash
# 출력할 필드를 직접 지정
# 형식: 파일번호.필드번호

join -o 1.2,1.3,2.2,1.1 employees.txt salaries.txt
#        ↑   ↑   ↑   ↑
#     파일1  파일1 파일2 파일1
#     2번째 3번째 2번째 1번째
```

```txt
# employees.txt    salaries.txt
# 1 Alice HR       1 50000
# 2 Bob   IT       2 60000

join -o 1.2,1.3,2.2,1.1 employees.txt salaries.txt

출력:
  Alice HR 50000 1
  Bob   IT 60000 2

  1.2 → employees 의 2번째 필드 (Alice, Bob)
  1.3 → employees 의 3번째 필드 (HR, IT)
  2.2 → salaries  의 2번째 필드 (50000, 60000)
  1.1 → employees 의 1번째 필드 (1, 2)

원하는 컬럼만 원하는 순서로 조합
→ SQL 의 SELECT 컬럼 지정과 동일한 개념
```

## SQL JOIN 대응 ⭐️

```bash
# INNER JOIN (기본)
join employees.txt salaries.txt

# LEFT JOIN (파일1 기준, 매칭 없으면 포함)
join -a 1 employees.txt salaries.txt

# RIGHT JOIN (파일2 기준)
join -a 2 employees.txt salaries.txt

# FULL OUTER JOIN
join -a 1 -a 2 employees.txt salaries.txt

# 특정 컬럼만 SELECT
join -o 1.2,2.2 employees.txt salaries.txt
# → 이름 + 급여만 출력
```

```txt
join 은 정렬된 텍스트 파일에서만 동작
→ DB 없이 CSV / 로그 파일 분석할 때 유용
→ 대용량 파일은 awk 가 더 유연
```

---

---

# paste — 파일 옆에 붙이기

```bash
paste fruits.txt colors.txt          # 탭으로 구분
paste -d ':' fruits.txt colors.txt   # : 로 구분
paste -s fruits.txt                  # 한 줄로 직렬화
```

---

---

# 한눈에

| 명령어                     | 용도            | 자주 쓰는 옵션               |
| ----------------------- | ------------- | ---------------------- |
| `cut -d: -f1,6`         | 컬럼 추출         | -d 구분자 / -f 번호         |
| `sort -rn`              | 정렬            | -r 역순 / -n 숫자 / -h 단위  |
| `sort -k2`              | 특정 필드 기준 정렬   | -k번호 / -t 구분자          |
| `sort \| uniq -c`       | 중복 제거 + 개수    | -c 개수 / -d 중복만         |
| `wc -l`                 | 줄 수           | -l 줄 / -w 단어 / -m 문자   |
| `awk '{print $1}'`      | 컬럼 출력 / 계산    | -F 구분자                 |
| `awk '{count[$6]++}'`   | 필드 값별 빈도 집계   | `\| sort -k2 -rn` 과 조합 |
| `awk -v x=$val`         | 쉘 변수 → awk 전달 | -v 변수=값                |
| `awk '{print NF, $NF}'` | 필드 수 / 마지막 필드 | NF 내장 변수               |
| `awk -f script.awk`     | 스크립트 파일 실행    | -f 파일명                 |
| `sed 's/a/b/g'`         | 텍스트 치환        | -i 파일직접수정              |
| `tr 'a-z' 'A-Z'`        | 문자 치환         | -d 삭제 / -s 압축 / -c 보수  |
| `col -x`                | 탭 → 공백        | -x                     |
| `join 파일1 파일2`          | 공통 필드 결합      | -1 -2 필드번호             |
| `paste 파일1 파일2`         | 파일 옆에 붙이기     | -d 구분자 / -s 직렬화        |