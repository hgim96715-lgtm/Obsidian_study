---
aliases:
  - 디스크
  - df
  - du
  - mount
  - 파티션
tags:
  - Linux
related:
  - "[[00_Linux_HomePage]]"
  - "[[Linux_Archive_Compress]]"
  - "[[Linux_System_Info]]"
---

# Linux_Disk_Memory — 디스크 & 메모리

## 한 줄 요약

```txt
df  = 디스크 전체 사용량 확인
du  = 특정 폴더/파일 크기 확인
mount = 디스크를 특정 폴더에 연결
```

---

---

# ① 개념 먼저 — 파티션 / 파일시스템 / 마운트 ⭐️

## 쉽게 이해하기

```txt
비유: 건물 비유로 이해

파티션 (Partition) = 건물을 층/호수로 나누는 것
  하나의 디스크를 여러 구역으로 나눔
  예: /dev/sda1 (OS 영역), /dev/sda2 (데이터 영역)

파일시스템 (Filesystem) = 각 방의 서류 정리 방식
  파티션 안에 파일/폴더를 어떻게 저장할지 결정
  ext4 (리눅스 기본), XFS, btrfs 등
  → 포맷(format) = 서류 정리 시스템 설치

마운트 (Mount) = 방에 문을 달아서 들어갈 수 있게 하는 것
  파일시스템을 특정 디렉토리에 연결
  연결 전: 그 디스크에 접근 불가
  연결 후: /mnt/disk/ 로 접근 가능

정리:
  디스크 구매 → 파티션 나누기 → 파일시스템 포맷 → 마운트
  (큰 방)        (방 나누기)       (서류 정리법 설치)   (문 달기)
```

## USB 드라이브 비유

```txt
USB 꽂기        = mount (마운트)
USB 빼기        = umount (언마운트)
USB 포맷하기    = mkfs.ext4 (파일시스템 생성)
USB 인식되는 곳 = 마운트 포인트 (/mnt/usb/)

→ USB 꽂으면 자동으로 /media/usb 에 마운트됨
   우리가 수동으로 하는 것과 동일한 동작
```

---

---

# ② df — 디스크 전체 사용량 ⭐️

```bash
# 기본 (bytes 단위 — 읽기 어려움)
df

# 사람이 읽기 좋은 단위 (KB/MB/GB)
df -h
# Filesystem      Size  Used Avail Use% Mounted on
# overlay          20G  126M   20G   1% /
# /dev/nvme1n1    100G   56G   45G  56% /etc/hosts

# 특정 경로의 디스크만
df -h /home
df -h /etc/hosts
```

## df 출력 읽는 법

```txt
Filesystem   어떤 디스크/파티션
Size         전체 크기
Used         사용 중
Avail        남은 공간
Use%         사용률 (이게 90% 넘으면 위험!)
Mounted on   어디에 마운트됐는지

Use% 80% 이상 → 주의
Use% 90% 이상 → 위험 (서버 장애 가능)
Use% 100%     → 쓰기 불가 → 서버 다운
```

---

---

# ③ du — 특정 폴더/파일 크기 ⭐️

```bash
# 홈 디렉토리 전체 (너무 많이 출력됨)
du ~

# 읽기 좋은 단위
du -h ~

# 합계만 (total)
du -sh ~
# 4.2G  /home/labex

# 한 단계 하위 폴더만
du -h --max-depth=1 ~

# 홈 디렉토리 바로 아래 파일/폴더 크기
du -sh ~/*

# 가장 큰 것 top 10
du -h ~ | sort -rh | head -n 10
```

```txt
df vs du:
  df   디스크 전체 상태 확인 (Use% 보기)
  du   어느 폴더가 큰지 찾기 (범인 찾기)

  디스크 꽉 찼을 때:
    1. df -h → 어느 파티션이 꽉 찼는지
    2. du -sh /* | sort -rh | head -10 → 어느 폴더가 큰지
    3. 해당 폴더 정리
```

---

---

# ④ 가상 디스크 실습 — 개념 확인용

```txt
실제 하드디스크 없이 파일로 가상 디스크를 만들어서
포맷 / 마운트 / 사용 / 언마운트 흐름을 실습
```

## 1. 가상 디스크 파일 생성

```bash
# dd = 파일 복사/변환 유틸리티
dd if=/dev/zero of=virtual.img bs=1M count=256
# if=/dev/zero  → 0으로 채운 데이터 (입력)
# of=virtual.img → 만들 파일명 (출력)
# bs=1M        → 한번에 1MB씩
# count=256    → 256번 = 256MB 파일 생성

ls -lh virtual.img
# -rw-rw-r-- 1 labex labex 256M ... virtual.img
```

## 2. 파일시스템 포맷

```bash
# ext4 파일시스템으로 포맷 (USB 포맷과 동일 개념)
sudo mkfs.ext4 virtual.img
```

```txt
ext4 란:
  리눅스에서 가장 많이 쓰는 파일시스템
  안정적 / 빠름 / 대부분 리눅스 배포판 기본값
  파일/폴더를 어떻게 저장할지 규칙을 정하는 것
```

## 3. 마운트 포인트 생성

```bash
# 마운트할 빈 디렉토리 생성 (문 달 자리)
sudo mkdir /mnt/virtualdisk

# /mnt = 임시 마운트에 쓰는 관례적인 위치
```

## 4. 마운트 (연결)

```bash
# 가상 디스크를 /mnt/virtualdisk 에 연결
sudo mount -o loop virtual.img /mnt/virtualdisk
# -o loop = 파일을 실제 디스크처럼 취급 (루프 장치)

# 마운트 확인
mount | grep virtualdisk
# /home/labex/project/virtual.img on /mnt/virtualdisk type ext4

# 이제 /mnt/virtualdisk 로 접근 가능
sudo touch /mnt/virtualdisk/testfile
ls /mnt/virtualdisk
```

## 5. 언마운트 (연결 해제)

```bash
# 사용 끝나면 반드시 언마운트
sudo umount /mnt/virtualdisk

# ⚠️ 언마운트 안 하면:
#   읽기/쓰기 작업 완료 보장 안 됨
#   → 데이터 손상 가능
```

## 전체 흐름 정리

```txt
dd → 빈 파일 생성 (디스크 역할할 파일)
mkfs.ext4 → 포맷 (서류 정리 시스템 설치)
mkdir /mnt/virtualdisk → 마운트 포인트 생성 (문 달 자리)
mount → 마운트 (연결)
사용...
umount → 언마운트 (안전하게 분리)
```

---

---

# ⑤ fdisk — 파티션 확인

```bash
# 시스템 전체 디스크/파티션 목록
sudo fdisk -l

# 특정 디스크 파티션 확인
sudo fdisk -l virtual.img

# 파티션 수정 (실제 디스크 — 주의!)
sudo fdisk /dev/sda
# n = 새 파티션 생성
# d = 파티션 삭제
# w = 저장 후 종료
# q = 저장 안 하고 종료
```

```txt
⚠️ fdisk 주의:
  파티션 수정하면 데이터 손실 가능
  수정 전 반드시 백업
  학습/확인 용도로만 -l 옵션 사용
```

---

---

# ⑥ free — 메모리 확인

```bash
free -h
#               total    used    free    available
# Mem:           7.7G    3.2G    1.1G    4.0G
# Swap:          2.0G    0.0B    2.0G

# 1초마다 갱신
free -h -s 1
```

```txt
Mem: 실제 RAM
Swap: 디스크를 RAM처럼 쓰는 임시 공간 (느림)
available: 실제로 쓸 수 있는 메모리

available 이 total 의 20% 미만이면 위험
```

---

---

# 디스크 장애 대응 패턴 ⭐️

```bash
# 1. 어느 파티션이 꽉 찼는지
df -h

# 2. 어느 폴더가 큰지
du -sh /* 2>/dev/null | sort -rh | head -10

# 3. 로그 파일이 주범인 경우
ls -lhS /var/log/*.log | head -5
sudo tar -czf /backup/logs.tar.gz /var/log/*.log
sudo rm /var/log/*.log

# 4. 디스크 여유 확인
df -h
```

---

---

# 명령어 한눈에

|명령어|역할|
|---|---|
|`df -h`|디스크 전체 사용량|
|`du -sh 경로`|특정 경로 크기 합계|
|`du -h --max-depth=1`|한 단계 하위 폴더 크기|
|`du -h \| sort -rh \| head -10`|큰 것 top 10|
|`dd if=/dev/zero of=파일 bs=1M count=N`|N MB 빈 파일 생성|
|`mkfs.ext4 파일`|ext4 파일시스템 포맷|
|`mount -o loop 파일 /마운트포인트`|가상 디스크 마운트|
|`umount /마운트포인트`|언마운트|
|`fdisk -l`|파티션 목록 확인|
|`free -h`|메모리 사용량|