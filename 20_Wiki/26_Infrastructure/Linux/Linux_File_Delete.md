---
aliases:
  - rm
  - 파일 삭제
  - rmdir
tags:
  - Linux
related:
  - "[[00_Linux_HomePage]]"
  - "[[Linux_File_Move_Copy]]"
  - "[[Linux_Archive_Compress]]"
  - "[[Linux_Directory_Commands]]"
---

# Linux_File_Delete — 파일 & 디렉토리 삭제

## 한 줄 요약

```
rm      = 파일 삭제
rm -rf  = 디렉토리 + 내용 전부 삭제 (⚠️ 위험)
rmdir   = 빈 디렉토리만 삭제

삭제 전 반드시 ls 로 확인하는 습관
```

---

---

# ① rm — 파일 삭제 ⭐️

```bash
# 기본 삭제
rm file.txt

# 여러 파일 동시
rm file1.txt file2.txt file3.txt

# 삭제 전 확인 물어보기 (-i)
rm -i file.txt
# rm: remove regular file 'file.txt'? y

# 상세 출력하며 삭제 (-v)
rm -v file.txt
# removed 'file.txt'
```

---

---

# ② rm -rf — 디렉토리 전체 삭제 ⭐️⚠️

```bash
# 디렉토리와 내부 전체 삭제
rm -rf 폴더명/
rm -rf old_logs/
rm -rf ./temp/

# -r = recursive (재귀적 / 내부까지)
# -f = force (강제 / 확인 없이)
```

```
⚠️⚠️ rm -rf 는 취소 불가 ⚠️⚠️
  휴지통 없음 / 복구 거의 불가
  rm -rf / → 시스템 전체 삭제 (절대 금지)
  rm -rf ./ → 현재 폴더 전체 삭제 (매우 위험)

  실수 방지 원칙:
  1. 삭제 전 ls 로 파일 목록 확인
  2. rm -i 로 하나씩 확인하며 삭제
  3. 중요 파일은 rm 전에 백업
```

## rm -rf 안전하게 쓰기

```bash
# ❌ 위험 패턴
rm -rf *          # 현재 폴더 전부 삭제 (실수하기 쉬움)
rm -rf /path/     # 경로 실수하면 대참사

# ✅ 안전 패턴 — 삭제 전 ls 로 확인
ls old_logs/       # 삭제할 폴더 내용 먼저 확인
rm -rf old_logs/   # 확인 후 삭제

# ✅ 삭제 전 에코 확인
echo rm -rf old_logs/   # 실제 삭제 안 하고 명령어만 출력
rm -rf old_logs/        # 명령어 확인 후 실행
```

---

---

# ③ 와일드카드 패턴 삭제 ⭐️

```bash
# 특정 확장자 전체 삭제
rm *.log           # 현재 폴더의 모든 .log 파일
rm *.tmp           # 임시 파일 전부
rm *.bak           # 백업 파일 전부

# 특정 패턴 삭제
rm *_2023-*.log    # 2023 이 포함된 log 파일
rm error_*.txt     # error_ 로 시작하는 txt

# 삭제 전 ls 로 패턴 확인 (필수!)
ls *_2023-*.log    # 어떤 파일이 삭제될지 먼저 확인
rm *_2023-*.log    # 확인 후 삭제
```

```
와일드카드 실수 방지:
  rm *.log 치기 전에
  ls *.log 로 먼저 확인
  → 예상한 파일만 나오는지 검증
  → 그 다음 rm
```

---

---

# ④ rmdir — 빈 디렉토리 삭제

```bash
# 빈 폴더만 삭제 가능
rmdir empty_folder/

# 안에 파일 있으면 에러
rmdir non_empty/
# rmdir: failed to remove 'non_empty/': Directory not empty

# 중첩 빈 폴더 삭제
rmdir -p a/b/c/   # a/b/c → a/b → a 순으로 삭제 (전부 빈 경우)
```

```
rmdir vs rm -rf:
  rmdir  비어있어야 삭제 (안전)
  rm -rf 내용 상관없이 전부 삭제 (위험)

  확실히 비어있는 폴더 삭제 → rmdir (안전)
  내용 포함해서 삭제 → rm -rf (신중하게)
```

---

---

# ⑤ find + rm — 조건부 삭제 ⭐️

```bash
# 오래된 파일 삭제 (find + rm)
find /path -name "*.log" -mtime +30 -delete
# -mtime +30 = 30일 이상 된 파일
# -delete    = 찾으면서 바로 삭제

# 삭제 전 확인 (find 만 먼저 실행)
find /path -name "*.log" -mtime +30   # 삭제될 파일 목록 확인
find /path -name "*.log" -mtime +30 -delete  # 확인 후 삭제

# 용량 큰 파일 찾아서 삭제
find /var/log -name "*.gz" -size +100M -delete
```

---

---

# ⑥ 로그 정리 실전 패턴

```bash
cd /var/log/myapp

# 1. 파일 목록 확인
ls -lh *.log

# 2. 오래된 파일 압축
tar -czf old_logs_$(date +%Y%m%d).tar.gz *_2023-*.log

# 3. 압축 확인
tar -tzf old_logs_*.tar.gz

# 4. 원본 삭제
rm *_2023-*.log

# 5. 결과 확인
ls -lh
df -h .   # 디스크 확보 확인
```

---

---

# 명령어 한눈에

|명령어|역할|위험도|
|---|---|---|
|`rm 파일`|파일 삭제|보통|
|`rm -i 파일`|확인하며 삭제|낮음 (안전)|
|`rm *.확장자`|패턴 파일 삭제|높음 (ls 로 먼저 확인)|
|`rm -rf 폴더/`|폴더 전체 삭제|매우 높음 ⚠️|
|`rmdir 폴더/`|빈 폴더만 삭제|낮음 (안전)|
|`find -delete`|조건부 삭제|보통|

---

---

# 자주 하는 실수

|실수|원인|해결|
|---|---|---|
|`rm *` 로 전부 삭제|와일드카드 범위 착각|`ls *` 먼저 확인|
|`rm -rf ./`|현재 폴더 전체 날림|경로 두 번 확인|
|삭제 후 복구 시도|리눅스는 휴지통 없음|삭제 전 백업 필수|
|`rmdir` 로 내용 있는 폴더 삭제|에러 발생|`rm -rf` 또는 내용 먼저 삭제|