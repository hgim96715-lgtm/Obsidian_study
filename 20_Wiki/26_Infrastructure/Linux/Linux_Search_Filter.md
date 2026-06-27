---
aliases:
  - grep
  - 로그검색
  - 파일 검색
  - find
  - which
  - whereis
tags:
  - Linux
related:
  - "[[00_Linux_HomePage]]"
  - "[[Linux_Redirect]]"
  - "[[Linux_File_Delete]]"
  - "[[Linux_Process_Monitor]]"
---
# Linux_Search_Filter — grep & 로그 검색

```
grep = 파일이나 출력에서 특정 패턴이 포함된 줄만 걸러내기
로그 분석 / 에러 추출 / 설정 파일 검색에 매일 사용
```


---

---

# 와일드카드 (Glob 패턴) ⭐️

```
와일드카드 = 파일 이름 패턴 매칭에 쓰는 특수 문자
쉘이 명령 실행 전에 패턴을 실제 파일 목록으로 자동 확장

ls *.txt  →  ls note1.txt note2.txt readme.txt  (쉘이 확장)
```

## 종류

|기호|의미|예시|
|---|---|---|
|`*`|모든 문자 0개 이상|`*.log` → 모든 .log 파일|
|`?`|문자 정확히 1개|`file?.txt` → file1.txt, fileA.txt|
|`[abc]`|괄호 안 문자 중 하나|`file[123].txt` → file1/2/3.txt|
|`[a-z]`|범위|`[0-9]*.log` → 숫자로 시작하는 .log|
|`[!abc]`|괄호 안 문자 제외|`[!0-9]*.log` → 숫자 외로 시작하는 .log|

```bash
ls *.txt           # .txt 로 끝나는 모든 파일
ls log*            # log 로 시작하는 모든 파일
ls file?.txt       # file1.txt, file2.txt (딱 1글자)
ls [abc]*.sh       # a, b, c 로 시작하는 .sh 파일
ls *.{py,sh,txt}   # py, sh, txt 전체 (중괄호 확장)
ls *               # 숨김 파일 제외
ls .*              # 숨김 파일만
```

```
* 와 ? 차이:
  *  없어도 됨 (0개 이상)  log* → log, logs, log.txt
  ?  반드시 1개             log? → logs, logA  (log 는 미해당)

와일드카드 vs 정규표현식:
  와일드카드   ls *.txt         파일 이름 패턴  (쉘 레벨)
  정규표현식   grep "^err" 파일  텍스트 내용 패턴 (명령어 레벨)
  → * 의 의미도 다름
    와일드카드 * = 모든 문자
    정규식     * = 앞 문자 0개 이상
```

---

---

# grep 기본 ⭐️

```
grep [옵션] "찾을패턴" [파일경로]
```

```bash
# 파일에서 직접 검색
grep "ERROR" app.log
grep "ERROR" *.log
grep "ERROR" /var/log/nginx/access.log

# 파이프로 받은 출력에서 검색 (파일 경로 없음)
cat app.log  | grep "ERROR"
dmesg        | grep "error"
ps aux       | grep "python"

# 결과를 파일로 저장
grep "ERROR" app.log > error_report.txt

# AND 검색 — 두 패턴을 모두 포함하는 행 ️
grep -i "warning" application.log | grep -i "query"
# → "WARNING" 과 "query" 를 모두 포함하는 행만 출력

# 결과 저장까지
grep -i "warning" application.log | grep -i "query" > output.txt
```

```
grep 구조:
  grep "패턴" 파일     → 파일에서 검색
  명령어 | grep "패턴" → 명령어 출력에서 검색

패턴 = 찾고 싶은 문자열 (기본은 대소문자 구분)
경로 없으면 → stdin (파이프로 받은 것) 에서 검색

AND 검색 패턴:
  grep 은 기본적으로 OR 나 AND 가 없음
  파이프로 grep 을 여러 번 연결 → AND 효과
  grep "A" file | grep "B"  → A 와 B 를 모두 포함하는 줄만

  OR 검색은 -E 옵션:
  grep -E "A|B" file  → A 또는 B 를 포함하는 줄
```

---

---

# grep 주요 옵션 ⭐️

## 옵션 한눈에

|옵션|의미|예시|
|---|---|---|
|`-i`|대소문자 무시|`grep -i "error"`|
|`-r`|하위 디렉토리 재귀 검색|`grep -r "error" /var/log/`|
|`-n`|줄 번호 표시|`grep -n "error" file`|
|`-v`|패턴 제외 (반전)|`grep -v "DEBUG"`|
|`-c`|매칭 줄 수만 출력|`grep -c "ERROR"`|
|`-l`|파일명만 출력|`grep -rl "error" /dir/`|
|`-E`|정규식 / 여러 패턴 OR|`grep -E 'fail\|error'`|
|`-F`|정규식 아닌 일반 문자열로 해석|`grep -F "1.2.3.4"`|
|`-w`|단어 전체 일치만 매칭|`grep -w "error"`|
|`-o`|매칭된 부분만 출력|`grep -oE '패턴'`|
|`-A N`|매칭 줄 + 이후 N줄|`grep -A 3 "ERROR"`|
|`-B N`|매칭 줄 + 이전 N줄|`grep -B 3 "ERROR"`|
|`-C N`|매칭 줄 + 전후 N줄|`grep -C 3 "ERROR"`|


## 실전 예시

```bash
grep -i "error" app.log               # Error / ERROR / error 전부
grep -r "worker_processes" /etc/nginx/ # 하위 폴더까지 전부
grep -n "ERROR" app.log               # 15:ERROR: connection failed
grep -v "ERROR" app.log               # ERROR 없는 줄만
grep -c "ERROR" app.log               # 15  (에러 줄 수)
grep -rl "ERROR" /var/log/            # 에러 있는 파일명만

grep -E 'fail|error' app.log          # fail 또는 error
grep -iE 'fail|error|warn' app.log    # 대소문자 무시 + OR

grep -A 3 "ERROR" app.log             # 에러 + 이후 3줄 (원인 파악)
grep -C 3 "ERROR" app.log             # 에러 + 전후 3줄

# -F — 특수문자 그대로 검색 ⭐️
grep -F "1.2.3.4" access.log             # . 을 점으로 해석 (정규식 X)
grep -F "Error: [object Object]" app.log # [] 특수문자 그대로 검색

# -w — 단어 전체 일치만 매칭
grep -w "error" app.log     # "error" 만 / "errors" "prerror" 는 제외
grep -w "GET" access.log    # "GET" 단어만 / "GETAWAY" 는 제외

# grep 자체 프로세스 제외
ps aux | grep python | grep -v grep
```


---

---

# 정규표현식 (regex) ⭐️

```
정규표현식 = 텍스트에서 특정 패턴을 찾기 위한 표현식
grep 의 패턴 자리에 사용  (-E 옵션으로 확장 정규식 활성화)
```

### 주요 기호

|기호|의미|예시|
|---|---|---|
|`^`|줄 시작|`^ERROR` → ERROR 로 시작|
|`$`|줄 끝|`\.py$` → .py 로 끝남|
|`.`|임의의 문자 1개|`l.b` → lab, lob, l1b|
|`*`|앞 문자 0개 이상|`lab*` → la, lab, labb|
|`+`|앞 문자 1개 이상 (`-E` 필요)|`[0-9]+` → 숫자 1개 이상|
|`\.`|점(.) 자체|`\.py$` → 진짜 .py|
|`[abc]`|a 또는 b 또는 c|`[Ll]ab` → Lab 또는 lab|
|`[^abc]`|a, b, c 제외|`[^0-9]` → 숫자 아닌 것|
|`[a-z]`|범위|`[a-zA-Z]` → 영문자|

### 패턴 읽는 법

```bash
# ^ — 줄 시작
grep "^ERROR" app.log         # ERROR 로 시작하는 줄만
grep "^#" nginx.conf          # 주석 줄만

# $ — 줄 끝
grep "\.py$" file_list.txt    # .py 로 끝나는 줄

# ^$ — 빈 줄
grep "^$" file.txt            # ^ 시작 바로 $ 끝 = 빈 줄

# [abc] — 문자 집합
grep "[Ll]ab" file.txt        # Lab 또는 lab
grep "lab[ex]*" file.txt      # lab 뒤에 e 또는 x 가 0개 이상
#    [ex]*  → lab, labe, labx, labex 전부 매칭

# -v + ^ 조합 — 주석 + 빈 줄 제외
grep -v "^#" nginx.conf | grep -v "^$"

# -E 여러 패턴 OR
grep -E 'ERROR|WARN|FATAL' app.log
grep -iE 'error|fail|warn' app.log
```

### 이메일 패턴 예시 ⭐️

```bash
# 이메일 포함 줄 전체 출력
grep -E '^[a-zA-Z0-9._-]+@[a-zA-Z0-9-]+(\.[a-zA-Z0-9-]+)+$' emails.txt

# -o : 매칭된 부분만 출력 (로그에서 이메일만 뽑기)
grep -oE '[a-zA-Z0-9._-]+@[a-zA-Z0-9.-]+\.[a-zA-Z]{2,}' app.log
```

```
패턴 분석:
  [a-zA-Z0-9._-]+   @ 앞부분 (1개 이상)
  @                  @ 기호
  [a-zA-Z0-9-]+     도메인 이름
  (\.[a-zA-Z0-9-]+)+ .com / .co.kr (점+문자 1번 이상 반복)

-o 옵션:
  줄 전체가 아닌 매칭된 부분만 출력
  → 로그에서 이메일·IP 주소만 깔끔하게 추출할 때 사용
```

---

---

# 파이프 | ⭐️

```
앞 명령어 출력 → 뒤 명령어 입력으로 전달
여러 명령어를 조합해서 복잡한 처리를 한 줄로
```

```bash
ps aux | grep python
ps aux | grep python | grep -v grep

dmesg | grep -iE 'error|fail' | tail -20
cat /etc/passwd | grep "labex"
ss -tulnp | grep ":8080"
```

---

---

# && / || — 조건부 실행 ⭐️

```
|   출력을 다음 명령어 입력으로 연결
&&  앞이 성공(exit 0) 해야 뒤 실행
||  앞이 실패(exit 1) 해야 뒤 실행
```

```bash
# && — 앞이 성공해야 뒤 실행
sudo apt-get update && sudo apt-get install -y nginx
mkdir mydir && cd mydir

# || — 앞이 실패해야 뒤 실행
which cowsay || echo "cowsay 가 없습니다"
ls config.txt || touch config.txt    # 파일 없으면 생성

# && + || 조합 — if-else 처럼 ⭐️
which cowsay && cowsay "Hello" || echo "cowsay 가 없습니다"
# 패턴: 명령어 && 성공시실행 || 실패시실행
```

---

---

# 리다이렉션 > / >> ⭐️

```bash
# > — 덮어쓰기 (기존 내용 날아감 ⚠️)
grep "ERROR" app.log > error_report.txt

# >> — 이어쓰기 (기존 내용 유지하며 추가)
grep "worker_processes" nginx.conf >> error_report.txt

# > 파일만으로 내용 초기화
> system_report.txt
```

```bash
보고서 작성 시 항상 >> 사용
>  파일 처음 만들 때
>> 내용 추가할 때
# [[Linux_Redirect]] 참고 
```


---

---

# find — 파일 & 디렉토리 위치 찾기 ⭐️

```
grep = 파일 안의 내용 검색
find = 파일 자체의 위치 검색
```

```
find 검색 시작 경로:
  .          현재 폴더 (및 하위 전체)
  ~          홈 디렉토리 (/home/username)
  /          루트 전체 (시스템 전체)
  /etc       /etc 폴더부터 검색 ← 특정 폴더 지정 가능
  /var/log   /var/log 폴더부터 검색

  → . 이나 ~ 만 되는 게 아니라 어떤 경로든 가능
```

```bash
# /etc 에서 .conf 파일만 찾아서 파일로 저장
find /etc -type f -name "*.conf" > ~/project/config_files.txt

# /var/log 에서 오늘 수정된 로그
find /var/log -mtime -1 -name "*.log"

# /home 전체에서 특정 사용자 파일
find /home -user nginx
```

```bash
# 기본
find . -name "*.txt"            # 현재 폴더 하위 전체
find ~ -name "config.json"      # 홈 디렉토리
sudo find / -name "passwd"      # 루트 전체

# 이름 패턴
find . -name "*.log"
find . -iname "*.TXT"           # 대소문자 무시

# -o (OR) — 여러 조건 중 하나
find . -name "*.txt" -o -name "*.log"   # .txt 또는 .log

# 크기
find ~ -size +1M                # 1MB 초과
find ~ -size -100k              # 100KB 미만
find . -size +10M -size -1G     # 10MB ~ 1GB 사이

# 수정 시간
find ~ -mtime -1                # 최근 24시간 내 수정
find ~ -mtime +7                # 7일 이상 된 파일
find ~ -mmin -30                # 최근 30분 내

# 파일 타입
find . -type f                  # 파일만
find . -type d                  # 디렉토리만

# 소유자 / 권한
find . -user nginx              # nginx 사용자 소유 파일
find . -group www-data          # www-data 그룹 파일
find . -perm 644                # 권한이 644 인 파일
find . -perm /111               # 실행 권한 있는 파일

# 탐색 깊이
find . -maxdepth 2 -name "*.log"   # 최대 2단계 깊이까지만
find . -mindepth 2 -name "*.log"   # 2단계 이상 깊이에서만

# 기타
find . -empty                      # 비어있는 파일/디렉토리
find . -newer reference.txt        # reference.txt 보다 최근 파일

# 찾은 후 실행
find . -name "*.log" -delete            # 찾은 것 삭제
find . -name "*.txt" -exec cat {} \;    # 찾은 파일 cat 출력
```

```
크기 단위:
  c = 바이트 / k = 킬로바이트 / M = 메가바이트 / G = 기가바이트

+N = N 보다 큰 것
-N = N 보다 작은 것

-o 옵션:
  find 명령어 안에서 OR 조건
  find . -name "*.txt" -o -name "*.log"
  → .txt 파일 또는 .log 파일 모두 찾음

-exec {} \; 설명 ⭐️:
  {}   find 가 찾아낸 파일 이름이 들어갈 자리 (placeholder)
  \;   -exec 에 전달할 명령어의 끝을 알리는 표시
       없으면 find 가 명령어 끝을 모름

  find . -name "*.txt" -exec cat {} \;
  → 찾은 각 파일에 대해 cat 파일명 실행
```

---

---

# which — 명령어 경로 & PATH 우선순위 ⭐️

## 기본 사용

```bash
which python     # /usr/bin/python
which python3    # /usr/bin/python3
which git        # /usr/bin/git
```

## which -a — 모든 설치 경로 확인 ⭐️

```bash
which -a python
# /usr/local/bin/python
# /usr/bin/python
```

```
which      PATH 에서 처음 발견한 경로 1개만 출력
which -a   PATH 에서 일치하는 경로 전부 출력

목록의 첫 번째 항목
= 터미널에서 python 을 입력했을 때 실제로 실행되는 파일

프로젝트마다 Python 버전이 다를 때 어느 버전이 실행되는지 확인할 때 사용
```

## echo $PATH — PATH 변수 확인 ⭐️

```bash
echo $PATH
# /usr/local/bin:/usr/bin:/bin:/home/user/custom_bin
#       ↑              ↑
#  우선순위 높음    우선순위 낮음
```

```
PATH = 명령어를 찾을 디렉토리 목록 (: 로 구분)
앞에 나열된 디렉토리가 더 높은 우선순위

같은 이름의 실행 파일이 두 디렉토리에 있으면
PATH 앞쪽 디렉토리에 있는 파일이 실행됨
```

## PATH 우선순위 직접 확인 ⭐️

```bash
# 특정 경로를 PATH 앞에 추가 (현재 세션만 적용)
export PATH=$HOME/custom_bin:$PATH

# 추가 후 어느 버전이 실행되는지 확인
which prioritydemo

# PATH 내 일치하는 것 전부 확인
which -a prioritydemo
# /home/user/custom_bin/prioritydemo   ← 이게 실행됨
# /usr/bin/prioritydemo
```

```
export PATH=$HOME/custom_bin:$PATH 분석:
  $HOME/custom_bin   추가할 경로
  :                  구분자
  $PATH              기존 PATH 전체

  앞에 붙이면 → 기존보다 우선순위 높아짐
  뒤에 붙이면 → 기존보다 우선순위 낮아짐

⚠️ export PATH=... 는 현재 세션에서만 유효
   영구 적용하려면 ~/.bashrc 또는 ~/.zshrc 에 추가
```

## which vs type

```bash
which git       # /usr/bin/git  (실행 파일 경로만)
type git        # git is /usr/bin/git  (내장/외부/별칭 구분)
type ll         # ll is aliased to 'ls -l'
```

```
which   실행 파일 경로만 출력
type    내장 명령어 / 외부 명령어 / 별칭 전부 구분해서 보여줌
→ 경로만 필요하면 which
→ 종류 확인하려면 type
```

---
---
# whereis — 바이너리 · 매뉴얼 · 소스 위치 찾기

```
which 는 PATH 기준으로 실행 파일만 찾음
whereis 는 바이너리 / 매뉴얼 / 소스 파일을 한번에 찾음

실무에서 자주 쓰이지는 않지만
명령어가 어디 있는지 + 매뉴얼은 어디 있는지 한번에 확인할 때 유용
```

```bash
# 기본 — 바이너리 / 매뉴얼 / 소스 전부 출력
whereis ls
# ls: /usr/bin/ls /usr/share/man/man1/ls.1.gz
#     ↑ 실행 파일   ↑ 매뉴얼 페이지

whereis grep
whereis bash
```

## 옵션

```bash
# -b — 바이너리(실행 파일)만
whereis -b grep
# grep: /usr/bin/grep

# -m — 매뉴얼 페이지만
whereis -m ssh
# ssh: /usr/share/man/man1/ssh.1.gz

# -s — 소스 파일만
whereis -s bash
# bash:  (없으면 빈 값)

# 옵션 조합
whereis -bm python3
# python3: /usr/bin/python3 /usr/share/man/man1/python3.1.gz
```

```
출력 읽는 법:
  /usr/bin/ls           → 실행 파일 위치
  /usr/share/man/...    → 매뉴얼 페이지 위치
  아무것도 없으면       → 설치 안 됐거나 PATH 밖에 있는 것

which vs whereis:
  which     실행 파일 경로 1개만 (PATH 기준)
  whereis   바이너리 + 매뉴얼 + 소스 위치 전부 (더 넓게 탐색)

특정 명령어가 설치됐는지 확인:
  whereis -b nonexistent → 빈 값 = 설치 안 됨
```