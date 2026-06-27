---
aliases:
  - 터미널 기초
  - echo
  - date
  - expr
  - clear
  - --help
  - man
tags:
  - Linux
related:
  - "[[00_Linux_HomePage]]"
  - "[[Linux_Concept_Overview]]"
  - "[[Linux_System_Info]]"
  - "[[Linux_Directory_Commands]]"
---

# Linux_Terminal_Basics — 터미널 첫 만남

## 한 줄 요약

```
터미널 = 리눅스와 대화하는 창구
명령어 입력 → 엔터 → 결과 출력
가장 기본이 되는 명령어들부터 익히자
```

---

---

# ① echo — 문자열 출력

```bash
# 화면에 텍스트 출력
echo "Hello Linux"
# Hello Linux

echo "백업 시작..."
echo "작업 완료"

# 변수 출력
NAME="ubuntu"
echo "사용자: $NAME"
# 사용자: ubuntu

# 줄바꿈 없이 출력 (-n)
echo -n "Enter password: "

# 이스케이프 문자 해석 (-e)
echo -e "줄1\n줄2\n줄3"
# 줄1
# 줄2
# 줄3
```

```
언제 쓰나:
  쉘 스크립트에서 진행 상황 출력
  echo "백업 시작..." → 사용자에게 안내
  echo "완료" >> log.txt → 로그 파일에 기록

  파일에 텍스트 쓰기
  echo "설정값" > config.txt
  echo "추가내용" >> config.txt
```

---

---

# ② date — 날짜 & 시간

```bash
# 현재 날짜/시간 전체 출력
date
# Fri Apr 20 10:30:00 KST 2026

# 포맷 지정 출력 ⭐️
date +%Y-%m-%d          # 2026-04-20
date +%Y%m%d            # 20260420  (파일명에 자주 씀)
date +%H:%M:%S          # 10:30:00
date +"%Y-%m-%d %H:%M"  # 2026-04-20 10:30
```

## date 포맷 코드

|코드|의미|예시|
|---|---|---|
|`%Y`|4자리 연도|2026|
|`%m`|월 (01~12)|04|
|`%d`|일 (01~31)|20|
|`%H`|시 (00~23)|10|
|`%M`|분 (00~59)|30|
|`%S`|초 (00~59)|00|
|`%A`|요일 영문|Friday|
|`%u`|요일 숫자 (1=월~7=일)|5|

## 실전 패턴 — 파일명에 날짜 넣기 ⭐️

```bash
# 날짜 포함 백업 파일명
tar -czf backup_$(date +%Y%m%d).tar.gz /data/
# → backup_20260420.tar.gz

# 날짜+시간 포함
tar -czf log_$(date +%Y%m%d_%H%M%S).tar.gz /var/log/
# → log_20260420_103000.tar.gz

# cron 에서는 % → \% 이스케이프
0 2 * * * tar -czf /backup/$(date +\%Y\%m\%d).tar.gz /data/
```

```
서버 시간 확인이 중요한 이유:
  로그 기록 시간이 꼬임
  cron 예약 작업이 엉뚱한 시간에 실행
  → 작업 전 date 로 서버 시간 확인 습관
```

---

---

# ③ cal — 달력

```bash
# 이번 달 달력
cal

# 특정 연도 전체 달력
cal 2026

# 특정 월 달력
cal 4 2026    # 2026년 4월
```

---

---

# ④ expr — 터미널 계산기

```bash
# 기본 사칙연산 (공백 필수!)
expr 5 + 3    # 8
expr 10 - 4   # 6
expr 6 \* 7   # 42  ← * 는 \* 로 이스케이프
expr 15 / 4   # 3   ← 정수 나눗셈 (소수점 없음)
expr 15 % 4   # 3   ← 나머지

# 변수 연산
A=10
B=3
expr $A + $B  # 13
```

```
⚠️ 공백 필수:
  expr 5+3   → 에러 (5+3 을 문자열로 인식)
  expr 5 + 3 → 8 ✅

* 는 \* 로:
  expr 6 * 7  → 에러 (* 가 와일드카드로 해석)
  expr 6 '*' 7✅
  expr 6 \* 7 → 42 ✅

소수점 계산이 필요하면:
  python3 -c "print(15 / 4)"  → 3.75
  echo "15 / 4" | bc          → 3
  echo "scale=2; 15 / 4" | bc → 3.75
```

## 쉘 스크립트에서 산술 연산 — $(( ))

```bash
# expr 보다 더 자주 쓰는 방식
A=10
B=3

echo $((A + B))    # 13
echo $((A * B))    # 30  ← 이스케이프 불필요
echo $((A / B))    # 3
echo $((A % B))    # 1

# 변수에 저장
RESULT=$((A + B))
echo $RESULT       # 13
```

```
expr vs $(( )):
  expr 5 + 3     → 8  (외부 명령어)
  $((5 + 3))     → 8  (쉘 내장 / 더 빠름)

실무에서는 $(( )) 를 더 많이 씀
expr 는 구버전 스크립트에서 자주 보임
```

---

---

# ⑤ clear — 화면 지우기

```bash
# 화면 정리
clear

# 단축키 (더 자주 씀)
Ctrl + L
```

```
실제 데이터 삭제 아님
화면을 위로 밀어서 시야 정리
스크롤 올리면 이전 내용 그대로
```

---

---

# ⑥ figlet — ASCII 아트 (보조 도구)

```bash
# 설치
sudo apt install figlet

# 기본 사용
figlet "Hello"

# 폰트 지정
figlet -f slant "Linux"

# 사용 가능한 폰트 목록
ls /usr/share/figlet/
```

```
실무 활용:
  서버 접속 시 경고문을 크게 표시
  "PRODUCTION SERVER" 같은 안내
  /etc/motd 파일에 넣어두면
  SSH 접속할 때마다 자동 출력
```

---

---

# ⑦ 터미널 단축키 & 생산성 도구 ⭐️

## Tab — 자동 완성

```bash
# 일부만 입력 후 Tab
cat h[Tab]       # → cat hello.txt  자동 완성
cd lin[Tab]      # → cd linux_practice/

# 후보 여러 개면 Tab 두 번 → 목록 출력
ls /etc/n[Tab][Tab]
# network  nginx  ...
```

```
Tab 은 가장 중요한 단축키
오타 방지 + 긴 경로 빠르게 입력
습관처럼 쓸 것
```

## ↑↓ — 이전 명령어 탐색

```bash
↑   이전 명령어
↓   다음 명령어
```

## history — 명령어 이력 ⭐️

```bash
history           # 전체 이력
history 20        # 최근 20개

# 특정 명령어 검색
history | grep git
history | grep apt

# !번호 로 재실행
!125              # 125번 명령어 재실행
sudo !!           # 마지막 명령어에 sudo 붙여 재실행
```

---

---

# ⑧ 도움말 & 명령어 탐색 ⭐️

## type — 명령어 종류 확인 ⭐️

```bash
type cd       # cd is a shell builtin   ← 내장 명령어
type ls       # ls is aliased to 'ls --color=tty'  ← 별칭
type cp       # cp is /usr/bin/cp       ← 외부 명령어 (경로 출력)
type grep     # grep is /usr/bin/grep
```

```
명령어 3가지 종류:
  내장 명령어 (built-in): 쉘 프로그램 안에 내장
    cd / echo / exit / pwd 등
    쉘이 직접 처리 → 외부 파일 없음

  외부 명령어 (external): 별도 실행 파일로 존재
    /usr/bin / /bin 디렉토리에 위치
    cp / grep / ls / tar 등

  별칭 (alias): 다른 명령어의 단축키
    ls → ls --color=tty 처럼 실제 실행 명령어가 따로 있음

type 이 유용한 때:
  명령어가 이상하게 동작할 때 실제로 뭐가 실행되는지 확인
  alias 로 덮어씌워져 있는지 확인
```

## --help — 빠른 옵션 확인

```bash
ls --help
cp --help
tar --help

# 길면 less 로 페이지 단위
ls --help | less
```

## --help 출력 읽는 법

```
ls [OPTION]... [FILE]...
    ↑             ↑
  선택 사항     선택 사항

[] = 선택 사항 (없어도 됨)
... = 여러 개 가능
→ ls -la /home /tmp  처럼 여러 옵션 + 여러 경로 가능
```

## man — 공식 매뉴얼

```bash
man ls
man cp
man grep
```

## man 내부 단축키 ⭐️

|키|동작|
|---|---|
|`↑` / `↓`|한 줄씩 스크롤|
|`Space`|한 페이지 앞으로|
|`b`|한 페이지 뒤로|
|`/검색어`|문서 안에서 검색|
|`n`|다음 검색 결과|
|`N`|이전 검색 결과|
|`q`|종료 ← 반드시!|

```
--help vs man:
  --help  간단한 옵션 목록 → 빠르게 확인
  man     상세 공식 설명서 → 깊게 이해
  둘 다 q 로 종료
```

## apropos — 키워드로 명령어 검색 ⭐️

```bash
# "무엇을 하고 싶은지" 알지만 명령어를 모를 때
apropos password        # 패스워드 관련 명령어 전체
apropos file            # 파일 관련 명령어 전체

# grep 조합으로 결과 필터링
apropos file | grep create   # 파일 + create 관련만
apropos copy | grep -i dir   # 복사 + 디렉토리 관련만
```

```
apropos 동작 원리:
  man 페이지의 이름 + 설명에서 키워드 검색
  관련 명령어 목록 출력

언제 쓰나:
  "파일 압축하는 명령어 뭐였지?" → apropos compress
  "프로세스 보는 명령어?" → apropos process
  결과가 너무 많으면 | grep 으로 좁히기
```

```
정리:
  type 명령어     → 이 명령어가 뭔지 확인
  --help         → 이 명령어 어떻게 쓰나 (빠른 참고)
  man 명령어      → 이 명령어 자세히 알고 싶다
  apropos 키워드  → 이 작업에 필요한 명령어가 뭔지 모르겠다
```

---
---


# 명령어 한눈에

|명령어|역할|예시|
|---|---|---|
|`echo "텍스트"`|화면 출력|`echo "시작"`|
|`date`|현재 날짜/시간|`date +%Y-%m-%d`|
|`cal`|달력 출력|`cal 2026`|
|`expr A + B`|계산 (공백 필수)|`expr 5 + 3`|
|`$((A + B))`|쉘 산술 연산|`echo $((10 * 3))`|
|`clear`|화면 정리|또는 `Ctrl+L`|
|`figlet "텍스트"`|ASCII 아트|`figlet -f slant "Hi"`|