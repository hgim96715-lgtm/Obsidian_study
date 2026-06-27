---
aliases:
  - cat
  - less
  - head
  - tail
  - 텍스트 명령어
  - nl
tags:
  - Linux
related:
  - "[[00_Linux_HomePage]]"
  - "[[Linux_Search_Filter]]"
  - "[[Linux_Redirect]]"
  - "[[Linux_Log]]"
---
# Linux_Text_Commands — 파일 내용 보기

## 한 줄 요약

```txt
cat   = 파일 전체 출력 (짧은 파일) "concatenate(연결하다)"의 약자
nl   = 줄 번호 포함 출력
less  = 페이지 단위로 보기 (긴 파일)
more  = 페이지 단위로 보기 (less 의 구버전)
head  = 앞 N줄
tail  = 뒤 N줄 / tail -f 실시간 모니터링
```

---

---

#  cat — 파일 전체 출력 ⭐️

```bash
# 기본 — 파일 내용 출력
cat file.txt
cat config.json
cat /etc/hosts

# 여러 파일 연결해서 출력
cat file1.txt file2.txt

# 줄 번호 포함
cat -n file.txt

# 빈 줄 제거해서 출력
cat -s file.txt     # squeeze: 연속 빈 줄 → 하나로

# 파일 내용 보면서 새 파일 만들기 (> 덮어쓰기)
cat file.txt > copy.txt

# 파일 내용 이어붙이기 (>>)
cat extra.txt >> main.txt
```

## 특수 문자 표시 옵션 

```bash
# -E : 각 줄 끝에 $ 표시
cat -E daily_report.txt
# Daily Report: August 8, 2024$
#                              ↑ 줄 끝 위치 확인 가능

# -T : 탭 문자를 ^I 로 표시
cat -T file.txt
# Hello^IWorld    ← 탭이 ^I 로 보임

# -A : -v -E -T 전부 합친 것 (모든 비출력 문자 표시)
cat -A file.txt
# Hello^IWorld$
```

```txt
언제 쓰나:
  -E  줄 끝 확인 → Windows(CRLF) vs Linux(LF) 개행 문자 디버깅
  -T  들여쓰기가 탭인지 스페이스인지 구분
  -A  서식 문제 전체 확인 (탭 + 줄끝 한번에)

옵션 요약:
  -n  모든 줄에 번호
  -b  비어있지 않은 줄만 번호
  -s  연속 빈 줄 → 하나로
  -E  줄 끝에 $ 표시
  -T  탭 → ^I 표시
  -v  비출력 문자 표시 (줄바꿈·탭 제외)
  -A  -v -E -T 동시에 (모든 비출력 문자)
```

## cat 실전 패턴

```bash
# 보고서 내용 확인
cat system_report.txt
cat ~/project/error_report.txt
cat ~/project/config_diff.txt

# 설정 파일 내용 확인
cat /etc/hosts
cat /etc/passwd | head -5

# 로그 파일 (짧은 경우)
cat app.log
```

```txt
cat 을 쓸 때:
  파일이 짧을 때 → cat (전부 한 번에)
  파일이 길 때   → less (페이지 단위로)
  로그 끝만 볼 때 → tail
  에러 찾을 때   → cat + grep
```

---
---
# nl — 줄 번호 포함 출력

```bash
nl file.txt
# 줄 번호와 함께 내용 출력
#      1  Line 1: Hello
#      2  Line 2: World

# cat -n 과 동일한 결과 (빈 줄 포함 번호)
cat -n file.txt

# nl 은 기본적으로 빈 줄 번호 생략
# cat -n 은 빈 줄도 번호 매김
```

## -b 옵션 — 번호 매길 줄 지정 

```bash
# 빈 줄 포함 모든 줄에 번호
nl -b a config.txt
#  a = all (모든 줄)

# 패턴 매칭 줄에만 번호
nl -b p'^[^#]' config.txt
#  b p = pattern / '^[^#]' = # 로 시작하지 않는 줄
#  → 주석(#) 줄 제외하고 실제 내용 있는 줄에만 번호
```

```txt
-b a   모든 줄 (빈 줄 포함)
-b n   번호 없음
-b p'패턴'  정규식 패턴과 일치하는 줄만

'^[^#]' 패턴 해석:
  ^     줄의 시작
  [^#]  # 이 아닌 모든 문자
  → # 로 시작하지 않는 줄만 번호 매김
```

## -n 옵션 — 번호 형식 

```bash
# 기본: 오른쪽 정렬 (빈칸 채움)
nl config.txt
#      1  Hello

# 오른쪽 정렬 + 앞을 0 으로 채움
nl -n rz config.txt
#  000001  Hello

# 왼쪽 정렬
nl -n ln config.txt
#  1       Hello
```

```txt
-n rn   오른쪽 정렬 (기본값)
-n rz   오른쪽 정렬 + 앞자리 0 채움 (000001, 000002)
-n ln   왼쪽 정렬
```

## 옵션 조합 ️

```bash
# 모든 줄 번호 + 0 채움 + 구분자 ': ' + 필드 너비 3
nl -b a -n rz -s ': ' -w 3 config.txt
#  001: Hello
#  002: (빈 줄)
#  003: World
```

```txt
-s '구분자'  번호와 내용 사이 구분자 지정 (기본값: 탭)
-w N        번호 필드 너비 지정 (기본값: 6)
-v N        시작 번호 지정 (기본값: 1)
-i N        번호 증가 폭 (기본값: 1)
```

## 언제 nl vs cat -n ️

```txt
cat -n   빠르게 / 빈 줄 포함 전체 번호
nl       빈 줄 제외 / 특정 줄만 / 형식 커스텀 필요할 때
```

---

---

# less — 페이지 단위로 보기 ⭐️

```bash
# 기본
less 파일명
less /var/log/syslog
less /etc/nginx/nginx.conf

# 파이프와 함께
ps aux | less
dmesg | less

# 옵션
less -N server_log.txt          # 행 번호 표시
less -i server_log.txt          # 검색 시 대소문자 무시
less -S server_log.txt          # 긴 줄 잘라서 표시 (줄바꿈 안 함)
less -F server_log.txt          # 한 화면에 다 들어오면 즉시 종료
less +F server_log.txt          # 파일 끝에서 실시간 추적 (tail -f 유사)

# 특정 패턴부터 시작
less +/ERROR server_log.txt     # 첫 번째 ERROR 위치부터 열기
```

## less 단축키

```txt
방향키 ↑↓  줄 이동
Space       한 페이지 아래
b           한 페이지 위 (back)
/검색어      앞으로 검색
?검색어      뒤로 검색
n           다음 검색 결과
N           이전 검색 결과
g           맨 처음으로
G           맨 끝으로
q           종료 ← 가장 중요!
```

## 옵션 정리

```txt
-N   행 번호 표시
     ⚠️ 행 번호는 /검색어 로 검색 불가 (내용만 검색됨)

-i   검색 시 대소문자 무시
     /error → ERROR / Error / error 모두 찾음

-S   긴 줄 잘라서 표시
     줄바꿈 없이 옆으로 늘어나는 로그에 유용
     방향키 ←→ 로 좌우 스크롤

-F   내용이 한 화면에 다 들어오면 즉시 종료
     짧은 파일 빠르게 확인할 때 유용

+F   파일 끝에서 실시간 추적
     tail -f 와 유사 / 로그 실시간 모니터링
     Ctrl+C 로 추적 중단 후 less 모드로 돌아옴

+/패턴   파일 열 때 첫 번째 패턴 위치부터 시작
     less +/ERROR server_log.txt
     → 파일 열자마자 ERROR 있는 줄로 이동
```

```txt
⚠️ less 에서 나오려면 q 키
  모르면 갇힌 것처럼 느껴짐
```

---
---
# more — 페이지 단위로 보기 (less 의 구버전)

```bash
# 기본 사용
more weather_data.txt
more /var/log/syslog
```

## more 단축키

```txt
Space     다음 페이지
Enter     한 줄 아래
b         이전 페이지
=         현재 라인 번호 확인
/검색어    패턴 검색 (대소문자 구분)
/         마지막 검색 반복
q         종료
```

## 옵션

```bash
# 특정 라인부터 시작 (+숫자 사이 공백 없음)
more +100 weather_data.txt    # 100번째 줄부터

# 한 번에 N줄씩 표시
more -10 weather_data.txt     # 10줄씩

# 패턴이 처음 나오는 위치부터 시작
more +/"2023-07-15" weather_data.txt
```

```txt
옵션 요약:
  +N        N번째 라인부터 시작
  -N        한 번에 N줄씩 표시
  +/패턴    패턴 첫 등장 위치부터
  -d        도움말 프롬프트 표시
  -s        연속 빈 줄 하나로 합침
  -p        페이지 표시 전 화면 지움
```

## more vs less 차이

```txt
more  구버전 / 앞으로만 이동 (b 키로 이전 가능하지만 제한적)
less  신버전 / 앞뒤 자유 이동 / 검색 기능 강력 / 대용량 파일 빠름

→ 실무에서는 less 를 더 많이 씀
  more 는 파이프 결과를 간단히 볼 때 사용하기도 함
```

---

---

# # head — 앞 N줄 보기

```bash
# 기본: 앞 10줄
head file.txt

# N줄 지정
head -n 20 file.txt    # 앞 20줄
head -5 file.txt       # 앞 5줄 (단축)
head -n 1 file.txt     # 첫 번째 줄만

# 실전 활용
head -5 /etc/passwd    # 계정 정보 상위 5개
head -10 error.log     # 로그 첫 10줄
cat big_file.csv | head -3   # CSV 헤더 확인
```

## 여러 파일 동시 확인 ⭐️

```bash
# 여러 파일 시작 부분을 한번에 확인
head access.log error.log
# → 각 파일 구분 헤더와 함께 출력
# ==> access.log <==
# ...
# ==> error.log <==
# ...
```

## 옵션

```txt
-n N   앞 N줄 출력 (기본값 10)
-c N   앞 N바이트 출력 (줄 단위 아님)
-q     여러 파일 확인 시 파일 이름 헤더 숨기기
-v     여러 파일 확인 시 항상 파일 이름 헤더 표시
```

## grep | head 조합 ⭐️

```bash
# 특정 패턴 검색 결과 상위 N개만 확인
grep "/admin" access.log | head -n 3
# access.log 에서 /admin 포함된 줄 중 첫 3개만

grep "ERROR" app.log | head -20
# ERROR 로그 최신 20개 확인
```

```txt
grep 으로 필터링 → head 로 개수 제한
로그가 수만 줄이어도 필요한 만큼만 빠르게 확인
```

## head + grep 조합 ⭐️

```bash
# grep 으로 필터링 후 head 로 상위 N개만
grep "/admin" access.log | head -n 3
# → /admin 경로 접근 로그 중 첫 3줄만 확인

grep "ERROR" app.log | head -20
# → 에러 로그 중 처음 20개만 빠르게 확인
```

```txt
head + grep 조합 왜 쓰나:
  grep 만 쓰면 결과가 수백 줄 나올 수 있음
  head 로 앞부분만 잘라서 빠르게 확인
  대용량 로그 파일 조사할 때 유용
```

---

---

#  tail — 뒤 N줄 보기 ⭐️

```bash
# 기본: 뒤 10줄
tail file.txt

# N줄 지정
tail -n 20 file.txt
tail -20 file.txt      # 단축형
tail -n 1 file.txt     # 마지막 줄만

# 실전 활용
tail -20 /var/log/syslog     # 최근 로그 20줄
tail -50 app.log             # 최근 50줄
```

## -n +N — 특정 줄부터 끝까지 출력 ⭐️

```bash
# 50번째 줄부터 파일 끝까지 출력
tail -n +50 /home/labex/project/system.log
#       ↑ + 기호: "이 줄 번호부터 시작"

# 3번째 줄부터 끝까지 (첫 2줄 건너뜀)
tail -n +3 file.txt
```

```txt
-n N   파일 끝에서 N줄 (기본 방향)
-n +N  N번째 줄부터 끝까지 (+ 기호가 방향 바꿈)

⚠️ 전체 줄 수보다 큰 숫자 지정 시
   아무것도 출력 안 됨 또는 파일 끝부분만 표시
```


## tail -f — 실시간 모니터링 ⭐️

```bash
# -f : follow (파일에 내용 추가될 때마다 자동 출력)
tail -f /var/log/syslog
tail -f /opt/airflow/logs/scheduler/*.log
tail -f /var/log/nginx/access.log

# 실시간 + grep 조합
tail -f app.log | grep "ERROR"      # 에러만 실시간
tail -f app.log | grep -i "warn"    # 경고만

# -n 과 -f 조합 — 마지막 N줄부터 시작해서 실시간 추적
tail -n 3 -f /home/labex/project/system.log
# 마지막 3줄 먼저 출력 → 이후 추가되는 내용 실시간 출력
```

```txt
tail -f 는 Ctrl + C 로 종료
서버 로그 실시간 모니터링의 핵심 패턴
Airflow DAG 실행 로그 추적 시 매우 유용

-n 3 -f 조합:
  -n 3   마지막 3줄 먼저 보여줌 (컨텍스트 확인)
  -f     이후 추가 내용 실시간 출력
  → 현재 상태 파악 + 실시간 추적 동시에
```

## tail -f /dev/null — 프로세스 유지 패턴 ⭐️

```bash
tail -f /dev/null
```

```txt
/dev/null:
  리눅스의 블랙홀 장치
  어떤 내용을 써도 사라짐
  읽으면 아무것도 없음 (EOF 없이 영원히 대기)

tail -f /dev/null 의 동작:
  /dev/null 을 follow → 내용이 절대 추가되지 않음
  → 아무것도 출력 안 하면서 프로세스만 살아있음
  → 터미널 세션 무한 유지
```

## 언제 쓰나

```bash
# 1. Docker 컨테이너 종료 방지 — 가장 많이 쓰는 용도 ⭐️
# Dockerfile 또는 docker-compose.yml 에서
CMD ["tail", "-f", "/dev/null"]
# → 컨테이너가 아무 작업 없어도 종료되지 않음
# → 내부 진입 후 디버깅 가능

# 2. 터미널 세션 유지 테스트
tail -f /dev/null &   # 백그라운드로 실행
# → SSH 연결 유지 / 세션 킵얼라이브 테스트

# 3. 스크립트에서 무한 대기
#!/bin/bash
echo "서버 준비 완료"
tail -f /dev/null     # 스크립트가 종료되지 않고 대기
```

```txt
Docker 에서 자주 보이는 이유:
  컨테이너는 CMD(메인 프로세스) 가 종료되면 즉시 종료됨
  데몬이 없는 컨테이너를 띄워두고 싶을 때
  → tail -f /dev/null 로 메인 프로세스를 영원히 살려둠

  예시:
    docker run -d ubuntu tail -f /dev/null
    docker exec -it 컨테이너명 bash   ← 내부 진입
```

---

---

#  wc — 줄/단어/바이트 수 세기

```bash
wc -l file.txt          # 줄 수만
wc -w file.txt          # 단어 수
wc -c file.txt          # 바이트 수

# 파이프와 조합
grep "ERROR" app.log | wc -l    # 에러 줄 수
cat /etc/passwd | wc -l         # 계정 수
ls /var/log/ | wc -l            # 로그 파일 수
```

---

---

#  실전 패턴 — 로그 분석

```bash
# 로그 파일 확인 흐름
# 1. 파일 크기 확인
ls -lh /var/log/app.log

# 2. 최근 로그 확인
tail -50 /var/log/app.log

# 3. 에러 추출
grep "ERROR" /var/log/app.log

# 4. 실시간 모니터링
tail -f /var/log/app.log | grep "ERROR"

# 5. 에러 수 확인
grep "ERROR" /var/log/app.log | wc -l
```

## Airflow 로그 모니터링

```bash
# DAG 실행 로그 실시간 보기
tail -f ~/airflow/logs/dag_name/task_name/날짜/*.log

# 에러만 필터
tail -f ~/airflow/logs/scheduler/*.log | grep -iE "error|fail"
```

---

---

# 명령어 한눈에

|명령어|역할|언제|
|---|---|---|
|`cat 파일`|전체 출력|파일이 짧을 때|
|`less 파일`|페이지 보기|파일이 길 때|
|`head -n N 파일`|앞 N줄|파일 시작 확인|
|`tail -n N 파일`|뒤 N줄|최근 로그 확인|
|`tail -f 파일`|실시간 추적|로그 모니터링|
|`wc -l 파일`|줄 수|로그 에러 개수|

---

---

# 자주 하는 실수

|실수|원인|해결|
|---|---|---|
|`less` 에서 못 나옴|q 키 모름|`q` 로 종료|
|`cat` 으로 큰 파일|화면 가득 쏟아짐|`less` 사용|
|`tail -f` 안 꺼짐|종료 방법 모름|`Ctrl + C`|
|head/tail 줄 수 헷갈림|기본값이 10줄|`-n N` 으로 명시|