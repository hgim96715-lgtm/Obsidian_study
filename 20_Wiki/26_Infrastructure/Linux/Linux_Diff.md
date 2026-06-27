---
aliases:
  - diff
  - 파일비교
  - 설정차이
  - 환경불일치
tags:
  - Linux
related:
  - "[[00_Linux_HomePage]]"
  - "[[Linux_Search_Filter]]"
  - "[[Linux_Text_Commands]]"
  - "[[Linux_File_Move_Copy]]"
---

# Linux_Diff — 파일 & 디렉토리 비교

## 한 줄 요약

```txt
diff  = 두 파일을 줄 단위로 비교
diff -r = 두 디렉토리를 재귀적으로 비교

"스테이징에서는 되는데 프로덕션에서 안 됨"
→ 즉시 diff 로 설정 파일 비교
```

---

---

#  diff 기본 사용법 ⭐️

```bash
# 기본: diff 파일1 파일2
diff file1.txt file2.txt

# 결과를 파일로 저장
diff staging.conf production.conf > config_diff.txt

# 확인
cat config_diff.txt
```

```txt
⚠️ 파일 순서가 중요:
  diff [기준] [비교대상]
  diff staging production  → "staging 기준으로 production 은 어떻게 다른가"
  diff production staging  → 반대 방향으로 해석됨
```

---

---

#  diff 출력 기호 해석 ⭐️

```bash
diff staging/app.conf production/app.conf

# 출력 예시:
# 5c5
# < worker_processes 4;
# ---
# > worker_processes 8;
#
# 12d11
# < debug_mode=true;
```

## 기호 의미

```txt
<   = 첫 번째 파일(file1)에만 있는 내용
>   = 두 번째 파일(file2)에만 있는 내용
--- = 구분선

숫자 코드:
  5c5  → 5번 줄이 변경됨 (change)
  12d11 → 12번 줄이 삭제됨 (delete)
  3a5  → 3번 줄 이후에 추가됨 (add)
```

```txt
읽는 법:
  < worker_processes 4;   ← staging 에서는 4
  > worker_processes 8;   ← production 에서는 8

  < debug_mode=true;      ← staging 에만 있음 (production 에는 없음)
```

---

---

#  diff -r — 디렉토리 비교 ⭐️

```bash
# 두 디렉토리 전체 비교 (재귀)
diff -r server1_files/ server2_files/

# 결과 저장
diff -r staging/ production/ > missing_files.txt
cat missing_files.txt
```

## diff -r 출력 해석

```bash
diff -r server1_files/ server2_files/

# Only in /server1_files: asset2.js
# → server1 에만 있고 server2 에는 없음 (누락!)

# Only in /server2_files: new_feature.js
# → server2 에만 있고 server1 에는 없음

# diff -r /server1/app.conf /server2/app.conf
# < timeout=30
# > timeout=60
# → 내용이 다른 파일
```

```txt
"Only in" 의미:
  한쪽에만 있는 파일 = 배포 누락 또는 추가

실무 활용:
  배포 시 특정 파일이 누락되어 웹페이지 깨지는 경우
  diff -r 로 즉시 누락 파일 식별
```

---

---

# diff 옵션

## -u — unified 형식 (git diff 와 유사)

```bash
diff -u staging.conf production.conf

# 출력:
# --- staging.conf    2026-04-20
# +++ production.conf 2026-04-20
# @@ -3,7 +3,7 @@
#  server_name example.com;
# -worker_processes 4;
# +worker_processes 8;
```

```txt
-u 형식이 더 읽기 쉬움:
  - (마이너스) = 첫 번째 파일 내용
  + (플러스)  = 두 번째 파일 내용
  (공백)      = 양쪽에 동일한 내용 (컨텍스트)

git diff 도 이 형식 사용
```

## -i — 대소문자 무시

```bash
diff -i file1.txt file2.txt
```

## -w — 공백 무시

```bash
diff -w file1.txt file2.txt   # 들여쓰기 차이 무시
```

## -q — 다른지 같은지만

```bash
diff -q file1.txt file2.txt
# Files file1.txt and file2.txt differ
# 또는 아무 출력 없음 (동일)
```

---

---

#  실전 패턴 ⭐️

## 스테이징 vs 프로덕션 설정 비교

```bash
# 설정 파일 비교 후 보고서 저장
diff ~/project/config/staging/app.conf \
     ~/project/config/production/app.conf > ~/project/config_diff.txt

cat ~/project/config_diff.txt
```

## 디렉토리 전체 비교

```bash
# 누락 파일 찾기
diff -r /staging/assets/ /production/assets/ > missing_files.txt

# 결과 확인
cat missing_files.txt
# Only in /staging/assets: asset2.js  ← 프로덕션에 없음!
```

## 배포 전 체크리스트

```bash
# 1. 설정 파일 차이 확인
diff -u staging/nginx.conf production/nginx.conf

# 2. 디렉토리 파일 누락 확인
diff -r staging/ production/ | grep "Only in"

# 3. 차이점 보고서 저장
diff -r staging/ production/ > deployment_diff.txt
echo "비교 완료: $(date)" >> deployment_diff.txt
```

## 백업 vs 현재 비교 (변경 추적)

```bash
# 수정 전 백업해뒀던 파일과 현재 비교
diff /etc/nginx/nginx.conf.bak /etc/nginx/nginx.conf

# 내가 바꾼 것만 확인
diff -u /etc/ssh/sshd_config.orig /etc/ssh/sshd_config
```

---

---

# 명령어 한눈에

|명령어|역할|
|---|---|
|`diff file1 file2`|두 파일 비교|
|`diff -u file1 file2`|unified 형식 (읽기 쉬움)|
|`diff -r dir1/ dir2/`|디렉토리 재귀 비교|
|`diff -q file1 file2`|다른지 같은지만 확인|
|`diff -i file1 file2`|대소문자 무시|
|`diff -w file1 file2`|공백 무시|
|`diff file1 file2 > diff.txt`|결과 파일 저장|
|`diff -r d1/ d2/ \| grep "Only in"`|누락 파일만|

## 출력 기호

|기호|의미|
|---|---|
|`<`|첫 번째 파일(file1)에만 있음|
|`>`|두 번째 파일(file2)에만 있음|
|`-` (unified)|첫 번째 파일 내용|
|`+` (unified)|두 번째 파일 내용|
|`Only in /경로: 파일`|해당 경로에만 있는 파일|

---

---

# 자주 하는 실수

|실수|원인|해결|
|---|---|---|
|파일 순서 반대로|방향 혼동|`diff [기준] [비교대상]` 순서|
|디렉토리 비교 에러|`-r` 없음|`diff -r dir1/ dir2/`|
|출력이 너무 많음|전체 출력|`\| grep "Only in"` 으로 필터|
|공백 차이로 false positive|들여쓰기 차이|`-w` 옵션 추가|