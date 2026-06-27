---
aliases:
  - tar
  - 압축
  - 아카이브
  - 로그 로테이션
tags:
  - Linux
related:
  - "[[00_Linux_HomePage]]"
  - "[[Linux_File_Move_Copy]]"
  - "[[Linux_File_Delete]]"
  - "[[Linux_Background_Jobs]]"
  - "[[Linux_Shell_Cron_Job]]"
  - "[[Linux_Directory_Commands]]"
  - "[[Linux_Directory_Commands]]"
---
# Linux_Archive_Compress — 압축 & 아카이브

## 한 줄 요약

```
tar    = 여러 파일을 하나로 묶기 (크기 그대로)
gzip   = 파일 크기 줄이기 (압축)
tar.gz = 묶기 + 압축 동시에 (가장 많이 씀)
```

---

---

# ① 아카이빙 vs 압축 — 개념부터 ⭐️

```
헷갈리는 이유:
  tar 와 gzip 이 하는 일이 다름
  하지만 보통 같이 씀 → tar.gz 형식

아카이빙 (Archiving) = tar:
  여러 파일/폴더를 하나의 파일로 묶기
  크기는 줄어들지 않음 (오히려 메타데이터 때문에 약간 증가)
  디렉토리 구조 / 권한 / 타임스탬프 정보 유지

압축 (Compression) = gzip:
  파일 크기를 줄이기
  데이터 중복 제거 알고리즘 사용
  묶지 않음 (파일 하나만 처리)
```

```
예시로 이해하기:
  logs/ 폴더 (100MB)
    access.log  70MB
    error.log   30MB

  tar 만 적용:   logs.tar    → 100MB + 약간 (묶기만)
  gzip 만 적용:  불가 (폴더 직접 불가)
  tar + gzip:   logs.tar.gz → 약 10~30MB (묶기 + 압축)

텍스트 파일은 압축 효과 매우 좋음 (반복 패턴 많으므로)
```

## tar 를 언제 쓰나 ⭐️

```
1. 로그 파일 보관
   Airflow / Kafka / 애플리케이션 로그 → 날짜별 압축 보관
   logs/2024-01/ → 2024-01-logs.tar.gz

2. 서버 배포
   소스코드 묶어서 서버로 전송
   tar -czf app.tar.gz src/ config/

3. 백업
   디렉토리 통째로 백업
   tar -czf backup_$(date +%Y%m%d).tar.gz /opt/myapp/

4. 개발 환경 전달
   프로젝트 통째로 팀원에게 전달
   tar -czf project.tar.gz ./project/

5. 데이터 파이프라인
   Airflow DAG 에서 cron 으로 자동 압축
```

---

---

# ② tar 단계별 이해

## 단계 1 — tar 만 (묶기만)

```bash
# 생성 (압축 없이 묶기만)
tar -cvf test_archive.tar test_dir
#    ↑↑↑
#    c = create (생성)
#    v = verbose (진행상황 출력)
#    f = file (파일명 지정)

# 내용 확인 (압축 해제 없이)
tar -tvf test_archive.tar
#    t = list (목록 확인)

# 압축 해제
tar -xvf test_archive.tar -C extracted_dir/
#    x = extract (추출)
#    -C = 디렉토리 지정
```

## 단계 2 — gzip 따로 압축

```bash
# tar 파일을 gzip 으로 압축
gzip test_archive.tar
# → test_archive.tar.gz 생성 (원본 .tar 사라짐)

# 압축 파일 크기 확인
ls -lh test_archive.tar.gz
# -rw-rw-r-- 1 labex labex 281 May 15 07:50 test_archive.tar.gz
#                           ↑   ↑
#                         크기  사람이 읽기 좋은 단위 (K, M, G)

ls -lh test_dir/         # 디렉토리 내 파일 크기
ls -lh *.tar.gz          # 모든 tar.gz 크기 한눈에
ls -lhS *.tar.gz         # 크기 순 정렬 (-S)
```

```
ls -lh 읽는 법:
  -rw-rw-r-- 1 labex labex 281 May 15 07:50 test_archive.tar.gz
  ↑권한       ↑링크수 ↑소유자  ↑크기 ↑날짜            ↑파일명

  크기 단위: K(킬로), M(메가), G(기가)
  -h = human-readable (281 → 281, 2048 → 2.0K, 1048576 → 1.0M)
  -h 없으면 bytes 로 표시 → 읽기 어려움

.tar.gz 와 .tgz 는 같은 것:
  test_archive.tar.gz  = test_archive.tgz  (완전히 동일)
  tar + gzip 압축 파일을 .tgz 로 줄여 쓰기도 함
  → 어느 쪽을 써도 tar 명령어로 동일하게 처리됨

  # 둘 다 동일하게 동작
  tar -xzvf archive.tar.gz
  tar -xzvf archive.tgz
```

## 단계 3 — tar + gzip 한번에 ⭐️ (실무)

```bash
# -z 옵션 추가 = gzip 압축 동시에
tar -czvf test_combined.tar.gz test_dir
#     z = gzip 압축

# 내용 확인
tar -tzvf test_combined.tar.gz

# 압축 해제
tar -xzvf test_combined.tar.gz -C extracted/
```

---

---

# ③ tar 핵심 옵션

```bash
tar [옵션] 아카이브명 파일들

옵션:
  c  = Create  (새 아카이브 생성)
  x  = eXtract (압축 해제)
  t  = lisT    (목록만 보기)
  z  = gZip    (.tar.gz 형식)
  f  = File    (파일명 지정, 항상 마지막)
  v  = Verbose (처리 과정 출력)
```

---

---

# ② 압축 생성 — tar -czf ⭐️

## 구조 먼저 이해하기

```
tar -czf  [저장할 파일명]  [압축할 대상]
           ↑               ↑
           결과물 이름      무엇을 압축할지
           (어디에 저장)    (어디서 가져올지)
```

```bash
tar -czf  backup.tar.gz  /var/log/
#         ↑               ↑
#         결과 파일        압축할 대상 (폴더)

tar -czf  backup.tar.gz  파일1.txt 파일2.txt
#         ↑               ↑
#         결과 파일        압축할 파일 여러 개

tar -czf  /home/labex/project/backup.tar.gz  /var/log/
#         ↑ 저장 위치 포함한 결과 경로              ↑ 압축 대상
```

```
핵심:
  f 다음에 오는 첫 번째 = 만들 파일 이름 (결과물)
  그 다음 = 압축할 파일/폴더 (재료)

  tar -czf  결과.tar.gz  재료
  저장할 곳  ↑           ↑ 가져올 곳
```

## 기본 예시

```bash
# 폴더 전체 압축
tar -czf backup.tar.gz src/
tar -czf project.tar.gz ~/project/

# 특정 파일만
tar -czf archive.tar.gz 파일1.log 파일2.log

# 와일드카드 패턴
tar -czf old_logs.tar.gz *_2023-*.log

# 저장 위치 + 날짜 포함 (sudo 로 /var/log 접근)
sudo tar -czf /home/labex/project/$(date +%Y-%m-%d).tar.gz /var/log/
#             ↑ 저장할 경로/파일명                          ↑ 압축할 대상
#             /home/.../2026-05-15.tar.gz 로 저장
```

```
sudo tar -czf /home/labex/project/$(date +%Y-%m-%d).tar.gz /var/log/ 분석:

  sudo                                    → root 권한 (로그 폴더 접근)
  tar -czf                                → 압축 생성
  /home/labex/project/$(date +%Y-%m-%d).tar.gz → 저장할 파일명 (날짜 포함)
  /var/log/                               → 압축할 대상 (시스템 로그 폴더)

  $(date +%Y-%m-%d) = 오늘 날짜 자동 입력
  → 결과: /home/labex/project/2026-05-15.tar.gz
```

## -czvf 각 옵션 상세 ⭐️

```
c  (Create)  = 새 아카이브 파일 생성
z  (Zip)     = gzip 으로 압축 (.tar.gz 형식)
v  (Verbose) = 처리 중인 파일명을 화면에 출력
f  (File)    = 저장할 아카이브 파일명 지정 → 반드시 마지막!

tar -czvf archive.tar.gz data/
     ↑↑↑↑ ↑             ↑
     옵션  아카이브이름   압축대상
```

```
v 옵션:
  있으면 → 압축 중인 파일들이 화면에 쭉 출력됨
  없으면 → 조용히 압축만 진행

  nohup tar -czvf backup.tar.gz data/ > backup.log 2>&1 &
  tail -f backup.log   ← 실시간 진행 확인
```

```
f 옵션 위치가 왜 마지막인가:
  f 는 "다음 인자가 파일명" 이라는 의미
  f 뒤에 아카이브 이름이 바로 와야 함

  ✅ tar -czvf archive.tar.gz data/
               ↑ f 다음 = 파일명

  ❌ tar -fczv archive.tar.gz data/
         ↑ f 가 앞에 오면 의미 혼동

  tar -czvf 파일들 archive.tar.gz  ← ❌ 순서 바꾸면 에러
  tar -czvf archive.tar.gz 파일들  ← ✅ 올바른 순서
```

## -czf vs -czvf 선택 기준

```
-czf  (v 없음)  조용히 압축 / cron 자동화 / 빠른 실행
-czvf (v 있음)  진행 상황 확인 / 수동 실행 / 디버깅
```

---

---

# ③ 목록 확인 — tar -tzf ⭐️

```bash
# 압축 풀지 않고 내용만 확인
tar -tzf old_logs.tar.gz
# app_2023-01-01.log
# app_2023-01-02.log
# error_2023-01-01.log
# ...

# 상세 정보 포함 (-v 추가)
tar -tzvf archive.tar.gz
```

```
압축 풀기 전에 반드시 확인하는 습관:
  1. tar -tzf 로 목록 확인
  2. 예상한 파일이 맞는지 검증
  3. 그 다음 압축 해제
```

---

---

# ④ 압축 해제 — tar -xzf

```bash
# 현재 디렉토리에 해제
tar -xzf archive.tar.gz

# 특정 디렉토리에 해제 (-C)
tar -xzf archive.tar.gz -C /tmp/

# 과정 출력하며 해제
tar -xzvf archive.tar.gz
```

## 부분 복원 — 특정 파일만 ⭐️

```bash
# 아카이브 전체 해제 대신 파일 하나만 복원
tar -xzvf backups/system-backup.tar.gz config/app.conf
#                                       ↑ 복원할 파일 경로 명시

# 복원 확인
ls -l config/app.conf
cat config/app.conf
```

```
장애 상황에서 수십 GB 전체 백업을 풀면 시간 낭비
→ 파일명 명시 → 해당 파일만 즉시 복원
→ 서비스 다운타임 최소화
```

## -T 옵션 — 목록 파일로 백업 ⭐️

```bash
# backup-list.txt 예시
# data/
# config/
# logs/

# -T 로 목록 파일 읽어서 백업
tar -czvf backups/system-backup.tar.gz -T backup-list.txt
#                                       ↑ 경로 목록 파일

# 백업 무결성 확인
tar -tzvf backups/system-backup.tar.gz > backup-contents.txt
cat backup-contents.txt
```

```
왜 목록 파일을 쓰나:
  명령줄에 경로 20개 나열 → 오타 유발
  backup-list.txt 한 번 만들어두면 반복 사용
  수정도 파일 하나만 편집 → 관리 편함
```

## -C 옵션 — 기준 디렉토리 지정

```bash
# -C 경로 = 해당 경로 기준으로 상대 경로 압축
tar -czf backup.tar.gz -C /home/labex/project data config logs
#                       ↑ 이 경로에서        ↑ 이 폴더들 압축

# 해제 시 -C 로 위치 지정
tar -xzf backup.tar.gz -C /restore/location/

# vs 경로 통째로 (차이)
tar -czf backup.tar.gz /home/labex/project/data
# → 아카이브 안 경로: home/labex/project/data/...

tar -czf backup.tar.gz -C /home/labex/project data
# → 아카이브 안 경로: data/...  (깔끔)
```

## cron 자동 백업 — 날짜 타임스탬프 패턴 ⭐️

```bash
# 매 분마다 날짜/시간 포함 백업 파일 생성
* * * * * tar -czf /home/labex/project/backups/backup-$(date +\%Y-\%m-\%d_\%H-\%M-\%S).tar.gz -C /home/labex/project data config logs

# cron 에서 % 는 줄바꿈으로 인식 → \% 로 이스케이프 필수!
# 터미널에서 직접 실행할 때는 % 그대로
tar -czf backup_$(date +%Y-%m-%d_%H-%M-%S).tar.gz -C /home/labex/project data config logs
```

```
cron % 이스케이프:
  터미널:  date +%Y-%m-%d    (% 그대로)
  cron:    date +\%Y-\%m-\%d (\% 이스케이프)

날짜 포맷:
  %Y = 연도 (2026)
  %m = 월   (04)
  %d = 일   (20)
  %H = 시   (10)
  %M = 분   (30)
  %S = 초   (00)

  결과: backup-2026-04-20_10-30-00.tar.gz
  → 같은 이름으로 덮어쓰기 방지
```

---

---

# ⑤ 와일드카드 패턴 — rm 과 조합 ⭐️

```bash
# 특정 패턴 파일만 압축 후 삭제 (로그 정리)
cd ~/project/logs

# 1. 2023년 로그 압축
tar -czf old_logs.tar.gz *_2023-*.log

# 2. 압축 확인
tar -tzf old_logs.tar.gz

# 3. 원본 삭제
rm *_2023-*.log

# 4. 결과 확인
ls
```

```
와일드카드 패턴:
  *        = 모든 문자 (아무거나)
  ?        = 한 문자
  [abc]    = a, b, c 중 하나

*_2023-*.log 해석:
  *        = 아무 이름
  _2023-   = 고정 문자
  *        = 아무 이름
  .log     = .log 확장자

예시 매칭:
  app_2023-01-01.log   ✅
  error_2023-12-31.log ✅
  app_2024-01-01.log   ❌ (2024는 해당 없음)
```

---

---

# ⑥ 로그 로테이션 패턴 ⭐️

```
문제:
  서버 장기 운영 → 로그 파일이 GB 단위로 쌓임
  디스크 100% → 시스템 장애

해결:
  오래된 로그 → 압축 보관 → 원본 삭제
  이것이 Log Rotation 의 핵심 원리
```

```bash
#!/bin/bash
# 로그 로테이션 스크립트 예시

LOG_DIR="/var/log/myapp"
ARCHIVE_DIR="/backup/logs"
DAYS_TO_KEEP=30

# 30일 이상 된 로그 압축
find $LOG_DIR -name "*.log" -mtime +$DAYS_TO_KEEP | while read f; do
    tar -czf "$ARCHIVE_DIR/$(basename $f).tar.gz" "$f"
    rm "$f"
done

echo "로그 정리 완료: $(date)"
```

---

---

# ⑦ 형식별 비교

|확장자|옵션|특징|
|---|---|---|
|`.tar`|`-cf` / `-xf`|묶기만 (압축 없음)|
|`.tar.gz` / `.tgz`|`-czf` / `-xzf`|gzip 압축 (빠름)|
|`.tar.bz2`|`-cjf` / `-xjf`|bzip2 (더 작지만 느림)|
|`.tar.xz`|`-cJf` / `-xJf`|xz 압축 (가장 작음)|

---

---

# 명령어 한눈에

|명령어|역할|
|---|---|
|`tar -czf 결과.tar.gz 파일들`|압축 생성|
|`tar -tzf 파일.tar.gz`|내용 목록 확인|
|`tar -xzf 파일.tar.gz`|압축 해제 (현재 위치)|
|`tar -xzf 파일.tar.gz -C 경로`|특정 경로에 해제|
|`rm *.log`|패턴 파일 삭제|
|`rm *_2023-*.log`|특정 패턴만 삭제|

---

---

# 자주 하는 실수

|실수|원인|해결|
|---|---|---|
|압축 후 원본 삭제 안 함|순서 잊음|`tar` → `tar -tzf` 확인 → `rm`|
|`rm *` 잘못 사용|와일드카드 범위 착각|삭제 전 `ls *_2023-*.log` 로 확인|
|`-f` 옵션 순서 틀림|`-fczf` 처럼 f 앞에 씀|`-czf` 순서 유지 (f 는 항상 마지막)|
|압축 해제 후 파일 위치 모름|경로 지정 안 함|`-C 경로` 로 명시적 지정|

---

---

# ⑧ zip / unzip — Windows 호환

```bash
# zip 생성 (-r = 하위 폴더 포함)
zip -r test_archive.zip test_dir

# 압축 해제 (-d = 경로 지정)
unzip -d unzipped_files test_archive.zip
# unzipped_files 폴더가 없으면 자동 생성
```

```
tar.gz vs zip:
  tar.gz  Linux 서버간 전송 / 배포 / 백업
  zip     Windows 사용자와 공유 / 범용 호환성

  → 팀 내 Linux 서버끼리: tar.gz
  → Windows 사용자에게 전달: zip
```

---

---

# ⑨ 디렉토리 생성 — 중괄호 확장 ⭐️

```
중괄호 확장 = 쉘이 패턴을 자동으로 펼쳐주는 기능
mkdir / touch / tar 등 모든 명령어에 적용 가능
```

## 문법 3가지

```bash
# 1. 직접 나열 {항목1,항목2,...}
mkdir -p test_dir/{subdir1,subdir2}
# → mkdir -p test_dir/subdir1 test_dir/subdir2

# 2. 숫자 범위 {시작..끝}
mkdir -p logs/{2023..2025}
# → mkdir -p logs/2023 logs/2024 logs/2025

# 3. 앞자리 0 유지 {01..12}
mkdir -p monthly/{01..12}
# → mkdir -p monthly/01 monthly/02 ... monthly/12
```

```
-p 옵션 = 중간 경로까지 자동 생성
  test_dir 없어도 → test_dir 자동 생성 후 subdir1/subdir2 생성
  -p 없으면 test_dir 없을 때 에러
```

## 실전 패턴

```bash
# 데이터 엔지니어 프로젝트 구조 한번에
mkdir -p pipeline/{src,config,logs/{2024,2025},data/{raw,processed}}
# pipeline/src/
# pipeline/config/
# pipeline/logs/2024/
# pipeline/logs/2025/
# pipeline/data/raw/
# pipeline/data/processed/

# 실행 전에 echo 로 확인
echo mkdir -p pipeline/{src,config,logs}
# mkdir -p pipeline/src pipeline/config pipeline/logs
```

```
⚠️ 주의:
  {a, b}  ❌ 공백 있으면 안 됨
  {a,b}   ✅ 쉼표 뒤 공백 없어야 함
```

→ 자세한 내용 → [[Linux_Directory_Commands]]