---
aliases:
  - xargs
  - 표준입력 인자변환
tags:
  - Linux
related:
  - "[[00_Linux_HomePage]]"
  - "[[Linux_Search_Filter]]"
  - "[[Linux_Text_Processing]]"
  - "[[Linux_File_Delete]]"
---


# Linux_Xargs — 표준입력을 명령어 인자로 변환

# 한 줄 요약

```
파이프(|)로 넘어온 결과를 다른 명령어의 "인자"로 바꿔서 실행
find . -name "*.log" | xargs rm
```

---

---

# 왜 필요한가 ⭐️

```
많은 명령어가 표준입력(stdin)을 직접 못 읽음:

  find . -name "*.log" | rm
  ❌ rm 은 인자(argument)로 파일명을 받아야 하는데
     파이프로 넘어온 건 표준입력(stdin)이라 rm 이 인식 못함

  find . -name "*.log" | xargs rm
  ✅ xargs 가 stdin 으로 받은 값을 rm 의 "인자"로 바꿔서 실행
     → 실제로는 rm file1.log file2.log file3.log 처럼 실행됨

xargs 가 하는 일 한 줄:
  "파이프 왼쪽 결과" → "명령어 뒤에 붙는 인자" 로 변환
```

---

---

# 기본 사용법 ⭐️

```bash
# 기본 형태
명령어1 | xargs 명령어2

echo "file1.txt file2.txt" | xargs rm
# 실행됨: rm file1.txt file2.txt

ls *.txt | xargs wc -l
# 실행됨: wc -l file1.txt file2.txt file3.txt ...

find . -name "*.tmp" | xargs rm
# .tmp 파일 전부 찾아서 한 번에 삭제
```

```
xargs 없이 직접 쓰면:
  for file in $(find . -name "*.tmp"); do rm "$file"; done
  → 한 줄로 가능한 걸 반복문으로 풀어 쓰는 격

xargs 쓰면:
  find . -name "*.tmp" | xargs rm
  → 훨씬 간결, 실무에서 압도적으로 많이 쓰는 패턴
```

---

---

# -I — 위치 지정해서 넣기 ⭐️

```bash
# 기본 xargs는 항상 "명령어 뒤"에 인자를 붙임
# 명령어 중간/여러 곳에 넣고 싶을 때 -I 사용

find . -name "*.txt" | xargs -I {} mv {} {}.bak
#                            ↑    ↑     ↑
#                       자리표시자  원본  변경

# {} 자리에 각 파일명이 하나씩 들어가서 실행됨
# mv file1.txt file1.txt.bak
# mv file2.txt file2.txt.bak
```

```bash
# 여러 번 같은 값을 써야 할 때
echo "report" | xargs -I {} cp {}.txt {}_backup.txt
# cp report.txt report_backup.txt
```

```
-I {} 의 {} 는 단순 표시 기호 (다른 문자도 가능: -I % 등)
  명령어 안에서 입력값이 들어갈 위치를 직접 지정
  기본 xargs(뒤에만 붙임) 로는 안 되는 패턴을 가능하게 함
```

---

---

# -n — 한 번에 넘길 개수 제한

```bash
echo "1 2 3 4 5 6" | xargs -n 2 echo
# echo 1 2
# echo 3 4
# echo 5 6
#  ↑ 2개씩 끊어서 명령어 여러 번 실행
```

```
언제 쓰나:
  명령어 인자 개수 제한이 있는 경우
  대량의 파일을 한 번에 처리하면 "Argument list too long" 에러 날 때
  → -n 으로 끊어서 여러 번 나눠 실행
```

---

---

# -0 / find -print0 — 공백·특수문자 파일명 안전 처리 ⭐️

```bash
# ❌ 파일명에 공백이 있으면 깨짐
find . -name "*.txt" | xargs rm
# "my file.txt" → my 와 file.txt 로 잘못 분리됨 ⚠️

# ✅ -print0 + xargs -0 조합 — null 문자로 구분
find . -name "*.txt" -print0 | xargs -0 rm
# 공백/특수문자 포함된 파일명도 안전하게 처리
```

```
왜 이렇게 해야 하나:
  기본적으로 xargs 는 공백/줄바꿈 기준으로 항목을 나눔
  파일명에 공백이 있으면 → 하나의 파일명이 여러 개로 잘못 쪼개짐

  -print0  find 결과를 공백 대신 null 문자(\0)로 구분해서 출력
  -0       xargs 가 그 null 문자 기준으로 항목을 나눠 받음
  → 공백이 포함된 파일명도 통째로 하나로 인식

  실무 권장: find + xargs 조합 시 항상 -print0 / -0 같이 쓰는 게 안전
```

---

---

# -P — 병렬 실행 ⭐️

```bash
# 순차 실행 (하나씩 처리)
find . -name "*.jpg" | xargs -I {} convert {} {}.png

# 병렬 실행 — 동시에 4개씩 처리
find . -name "*.jpg" | xargs -P 4 -I {} convert {} {}.png
```

```
-P 숫자:
  동시에 실행할 프로세스 개수
  대량 파일 처리(이미지 변환, 압축 등) 속도를 크게 높일 수 있음
  CPU 코어 수에 맞춰 설정 (너무 크면 오히려 느려질 수 있음)
```

---

---

# -p — 실행 전 확인 (dry run 처럼 사용)

```bash
find . -name "*.log" | xargs -p rm
# rm file1.log file2.log ?...
# y 입력해야 실제 실행됨
```

```
언제 쓰나:
  rm 같은 위험한 명령어를 xargs 와 같이 쓸 때
  실제로 어떤 명령이 실행될지 미리 확인하고 싶을 때
  → 대량 삭제 전에 안전장치로 사용
```

---

---

# -r (--no-run-if-empty) — 빈 입력 처리

```bash
# 입력이 없으면 xargs 가 명령어를 한 번이라도 실행하려고 시도함
find . -name "*.tmp"   # 결과 없음

# ❌ -r 없으면 "rm" 만 인자 없이 실행되려다 에러 날 수 있음
find . -name "*.tmp" | xargs rm

# ✅ -r 추가 — 입력이 비어있으면 명령어 자체를 실행 안 함
find . -name "*.tmp" | xargs -r rm
```


---
---
# 추가 옵션 — -L / -s / -a / -E ⭐️

## -L — 줄 단위로 묶어서 처리

```bash
# books.txt 내용:
#   book1
#   book2
#   book3
#   book4

cat books.txt | xargs -L 2 echo "묶음:"
# 묶음: book1 book2
# 묶음: book3 book4
```

```
-L 숫자 vs -n 숫자 차이:
  -n 2   "인자 개수" 기준으로 2개씩 (한 줄에 여러 단어가 있어도 단어 단위로 셈)
  -L 2   "줄(line)" 기준으로 2줄씩 (한 줄에 단어가 몇 개든 그 줄 전체를 하나로 취급)

  한 줄에 단어가 1개씩이면 -n 과 -L 결과가 같아 보이지만
  한 줄에 여러 단어가 있을 때 차이가 드러남:

  echo -e "apple banana\ncherry" | xargs -n 2 echo
  → apple banana   (이미 2개라 한 묶음)
  → cherry          (1개 남아서 따로)

  echo -e "apple banana\ncherry" | xargs -L 1 echo
  → apple banana   (줄 1)
  → cherry          (줄 2)
  → -L 은 줄 수 기준이라 "apple banana" 가 한 줄이면 그대로 한 묶음
```

## -s — 명령줄 길이(문자 수) 제한

```bash
# 한 번에 넘기는 인자들의 총 글자 수를 제한
cat long_list.txt | xargs -s 100 echo
# 합쳐서 100자가 넘지 않는 만큼씩 끊어서 echo 실행
```

```
언제 쓰나:
  파일 개수가 아니라 "명령줄 전체 글자 수"가 시스템 한도를 넘을 때
  대량의 긴 파일 경로를 한 번에 넘기다가
  "Argument list too long" 에러가 날 때 -n 대신 글자 수로 제한하고 싶을 때

-n vs -s:
  -n  "몇 개" 기준
  -s  "몇 글자" 기준 (개수가 적어도 경로가 길면 글자 수로 초과될 수 있음)
```

## -a — 표준입력 대신 파일에서 읽기

```bash
# 평소 패턴: 파이프로 입력 전달
cat books.txt | xargs echo

# -a 사용: 파이프 없이 파일을 직접 지정
xargs -a books.txt echo
# cat books.txt | xargs echo 와 결과 동일
```

```
-a 파일명 의 의미:
  표준입력(stdin) 대신 지정한 파일을 직접 읽어서 처리
  cat 으로 파일을 먼저 출력하고 파이프로 넘기는 과정을 한 단계 줄임

  cat file | xargs cmd     ← 2단계 (cat 실행 → 파이프 → xargs)
  xargs -a file cmd        ← 1단계 (xargs 가 파일을 직접 읽음)

  결과는 동일하지만 -a 가 프로세스 하나(cat) 를 덜 띄움
```

## -E — EOF(종료) 문자열 지정

```bash
# 입력 중간에 특정 문자열이 나오면 그 지점에서 입력을 끝낸 것으로 처리
echo -e "book1\nbook2\nSTOP\nbook3" | xargs -E STOP echo
# book1 book2
# (STOP 이후의 book3 은 무시됨)
```

```
-E 문자열:
  입력 스트림에서 그 문자열을 만나면 이후 내용을 전부 무시
  옛 POSIX xargs 의 동작 방식 (지금은 거의 안 씀)

  ⚠️ GNU xargs(리눅스 기본) 에서는 기본적으로 비활성화되어 있고
     명시적으로 -E 를 줘야 동작
  실무에서는 거의 쓰이지 않는 옵션 — 알아두면 충분
```
---

---

# 실전 조합 패턴 ⭐️

```bash
# 1. 특정 패턴 파일 전부 삭제 (공백 안전)
find . -name "*.tmp" -print0 | xargs -0 rm

# 2. 여러 텍스트 파일에서 동시에 문자열 검색
find . -name "*.txt" | xargs grep "ERROR"

# 3. 권한 한 번에 변경
find . -name "*.sh" | xargs chmod +x

# 4. 디렉토리 크기 한 번에 확인
find . -maxdepth 1 -type d | xargs du -sh

# 5. 이미지 대량 변환 (병렬)
find . -name "*.png" | xargs -P 4 -I {} convert {} {}.webp

# 6. grep 으로 찾은 파일들만 백업
grep -l "TODO" *.js | xargs -I {} cp {} {}.bak
```

```
find + xargs 가 가장 흔한 조합인 이유:
  find 는 "조건에 맞는 파일을 찾는" 역할만 함 (찾은 파일에 직접 명령 실행 못함)
  xargs 가 그 결과를 받아서 다른 명령어의 인자로 넘겨줌
  → 둘이 합쳐져야 "찾아서 + 처리" 가 완성됨

find 자체의 -exec 옵션과 비교:
  find . -name "*.tmp" -exec rm {} \;
  → xargs 없이도 같은 결과 가능하지만
  → xargs 가 더 빠름 (find -exec 은 파일마다 명령어 새로 실행 / xargs 는 묶어서 한 번에 실행)
```

---

---
# 실습 예제 모음 ⭐️

## 1. 기본 — 파일 내용을 인자로

```bash
cat ~/project/fruits.txt | xargs echo
```

```
fruits.txt 내용이 이렇다면:
  apple
  banana
  cherry

실행되는 것:
  echo apple banana cherry
  → apple banana cherry  (한 줄로 출력)

cat 으로 파일 내용을 꺼내고
xargs 가 그 내용(여러 줄)을 echo 의 "인자들"로 바꿔서 한 번에 실행
```

## 2. -I {} — 파일명마다 새 파일 만들기

```bash
cat ~/project/books.txt | xargs -I {} touch ~/project/{}.txt
```

```
books.txt 내용:
  harry_potter
  lord_of_rings

실행되는 것 (한 줄씩 반복):
  touch ~/project/harry_potter.txt
  touch ~/project/lord_of_rings.txt

{} 자리에 books.txt 의 각 줄(책 이름)이 하나씩 들어가서
touch 명령어가 줄 수만큼 반복 실행됨
→ harry_potter.txt, lord_of_rings.txt 두 개의 빈 파일이 새로 생성됨
```

## 3. -n 2 — 묶어서 처리

```bash
cat ~/project/more_books.txt | xargs -n 2 echo "Processing books:"
```

```
more_books.txt 내용:
  book1
  book2
  book3
  book4

-n 2 는 "한 번에 2개씩 묶어서 명령어 실행"

실행되는 것:
  echo "Processing books:" book1 book2
  echo "Processing books:" book3 book4

출력:
  Processing books: book1 book2
  Processing books: book3 book4
```

## 4. -P 3 -I {} — 병렬로 스크립트 실행 ⭐️

```bash
cat ~/project/more_books.txt | xargs -P 3 -I {} ~/project/process_book.sh {}
```

```
more_books.txt 내용:
  book1
  book2
  book3

-P 3  → 최대 3개까지 동시에(병렬) 실행
-I {} → 각 책 이름이 {} 자리에 들어가서 스크립트 실행

실행되는 것 (동시에):
  ~/project/process_book.sh book1
  ~/project/process_book.sh book2
  ~/project/process_book.sh book3
```

```bash
# process_book.sh 안에 이런 내용이 있다고 가정하면:
#!/bin/bash
# $1 = xargs 가 넘겨준 책 이름 (book1, book2 ...)
echo "처리 중: $1"
cp ~/project/"$1".txt ~/project/processed_"$1".txt
#  ↑ 원본을 읽어서 "processed_" 를 붙인 새 이름으로 복사/가공
```

```
processed_* 파일이 생기는 원리 ⭐️:

  xargs 자신은 파일을 만들지 않음
  → xargs 는 그냥 process_book.sh 를 book1, book2, book3 각각에 대해 실행시켜줄 뿐

  실제로 processed_book1.txt 같은 파일을 만드는 건
  process_book.sh 스크립트 내부의 로직 (위 예시의 cp 명령)
  → "원본 파일명 앞에 processed_ 를 붙여서 저장해라" 라는 코드가 스크립트 안에 있어야 함

  즉 processed_* 파일들이 보인다면:
    process_book.sh 가 book1, book2, book3 각각에 대해 실행되면서
    processed_book1.txt / processed_book2.txt / processed_book3.txt 를
    각자 만들어낸 결과물이라는 뜻
```

## 5. 결과 확인 — 와일드카드(*) 로 한꺼번에 보기

```bash
cat ~/project/processed_*
```

```
이건 xargs 명령이 아니라 그냥 쉘의 와일드카드(글로빙) 기능 ⭐️

* 는 "아무 문자열이나 매칭" 이라는 쉘의 패턴 확장 기호
실행 전에 쉘이 먼저 processed_* 를 실제 파일 목록으로 바꿔치기함

cat ~/project/processed_*
  ↓ 쉘이 먼저 펼침 (xargs 와 무관)
cat ~/project/processed_book1.txt ~/project/processed_book2.txt ~/project/processed_book3.txt
  ↓ cat 이 그 파일들을 전부 이어서 출력

즉 4번 단계에서 process_book.sh 가 만들어낸
processed_book1.txt, processed_book2.txt, processed_book3.txt 파일들을
한 번에 확인하는 명령일 뿐 — 새로운 xargs 개념이 아니라 쉘 와일드카드 복습
→ 헷갈렸던 부분은 "어떻게 그 파일이 생겼나" 인데, 답은 4번의 스크립트가 만든 것
```

## 6. -n 과 -P 동시에 — 배치 단위 병렬 처리 ⭐️

```bash
cat ~/project/classic_books.txt | xargs -n 2 -P 3 sh -c 'echo "Processing batch: $@"' _
```

```
classic_books.txt 내용:
  book1 book2 book3 book4 book5 book6

-n 2  → 2개씩 묶어서 한 묶음(batch) 구성
        [book1 book2] [book3 book4] [book5 book6]  → 총 3묶음

-P 3  → 묶음 3개를 동시에(병렬) 처리

sh -c '...' _ 의 의미:
  sh -c 'echo "Processing batch: $@"'
  → 뒤에 오는 인자들을 받아서 실행할 작은 셸 스크립트를 직접 정의

  $@  → sh -c 에 전달된 모든 인자(배치 안의 책 이름들)
  맨 끝의 _ (언더스코어):
    sh -c 의 첫 번째 인자는 항상 $0(스크립트 이름)으로 소비됨
    → 책 이름이 $0 으로 빨려 들어가 누락되는 걸 방지하기 위해
      더미 값(_)을 일부러 하나 채워 넣는 관용 표현
    → _ 가 없으면 첫 번째 책 이름이 $0 에 먹혀서 $@ 에서 빠짐 ⚠️

실행 결과 예 (3묶음이 거의 동시에):
  Processing batch: book1 book2
  Processing batch: book3 book4
  Processing batch: book5 book6
```

```
이 예제가 중요한 이유:
  xargs 의 -I {} 는 "한 개씩"만 처리 가능 (자리표시자라서)
  -n 2 처럼 "여러 개씩 묶음"으로 받아서 처리하려면
  sh -c '...' 로 직접 셸 스크립트를 인라인으로 만들어 $@ 로 한꺼번에 받아야 함
  → 묶음 단위 + 병렬 처리를 동시에 하고 싶을 때 자주 쓰는 패턴
```

---
---

# 한눈에

| 옵션       | 역할                             |
| -------- | ------------------------------ |
| (기본)     | stdin → 명령어 뒤에 인자로 붙임          |
| `-I {}`  | 자리표시자로 위치 지정                   |
| `-n 개수`  | 인자 개수 기준 묶음                    |
| `-L 개수`  | 줄(line) 개수 기준 묶음               |
| `-s 개수`  | 명령줄 총 글자 수 제한                  |
| `-0`     | null 문자 구분 (find -print0 과 세트) |
| `-P 개수`  | 병렬 실행                          |
| `-p`     | 실행 전 확인 (y/n)                  |
| `-r`     | 입력 없으면 명령어 실행 안 함              |
| `-a 파일`  | stdin 대신 파일에서 읽기               |
| `-E 문자열` | 지정 문자열에서 입력 종료 (거의 안 씀)        |

```
기본 패턴:
  find . -name "패턴" | xargs 명령어

안전한 패턴 (공백 파일명 대비):
  find . -name "패턴" -print0 | xargs -0 명령어

위험한 명령어(rm 등) 실행 전:
  -p 옵션으로 먼저 확인
```