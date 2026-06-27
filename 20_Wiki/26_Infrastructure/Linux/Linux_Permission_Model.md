---
aliases:
  - 권한
  - chmod
  - chown
  - rwx
  - 파일 권한
tags:
  - Linux
related:
  - "[[00_Linux_HomePage]]"
  - "[[Linux_Directory_Commands]]"
---
# Linux_Permission_Model — 파일 권한

## 한 줄 요약

```txt
리눅스 모든 파일/폴더에는 "누가 무엇을 할 수 있는가" 가 정해져 있음
rwx = 읽기(r) / 쓰기(w) / 실행(x)
chmod = 권한 변경 / chown = 소유자 변경
```

---

---

# ① 권한 구조 ⭐️

```bash
ls -l 파일명
# -rwxr-xr-- 1 ubuntu ubuntu 1234 Apr 20 10:00 script.sh
#  ↑↑↑↑↑↑↑↑↑
#  │└──┬──┘└──┬──┘└──┬──┘
#  │  소유자   그룹    기타
#  파일타입
```

## 파일 타입 (첫 글자)

```txt
-  일반 파일
d  디렉토리
l  심볼릭 링크
```

## rwx 의미

```txt
r (read)     = 4  읽기
w (write)    = 2  쓰기
x (execute)  = 1  실행

파일:
  r = 파일 내용 읽기
  w = 파일 수정
  x = 프로그램/스크립트 실행

디렉토리:
  r = 디렉토리 목록 보기 (ls)
  w = 파일 생성/삭제
  x = 디렉토리 안으로 이동 (cd)
```

## 3그룹 구조

```txt
-  rwx  r-x  r--
↑   ↑    ↑    ↑
타입 소유자 그룹  기타(others)

소유자 (user)   = 파일을 만든 사람
그룹  (group)   = 소유자가 속한 그룹
기타  (others)  = 나머지 전부
```

---

---

# ② 권한 읽는 법 ⭐️

```bash
ls -l
# -rwxr-xr-- 1 ubuntu ubuntu 1234 ... script.sh
#  rwx        소유자: 읽기+쓰기+실행
#     r-x     그룹:   읽기+실행
#        r--  기타:   읽기만
```

## 숫자 표기법

```txt
r = 4 / w = 2 / x = 1

rwx = 4+2+1 = 7
rw- = 4+2+0 = 6
r-x = 4+0+1 = 5
r-- = 4+0+0 = 4
--- = 0+0+0 = 0

예시:
  chmod 755 → rwxr-xr-x
  chmod 644 → rw-r--r--
  chmod 600 → rw-------
  chmod 700 → rwx------
```

## 자주 쓰는 권한 패턴

|숫자|기호|의미|언제|
|---|---|---|---|
|755|rwxr-xr-x|소유자 전체 / 나머지 읽기+실행|실행 파일 / 디렉토리|
|644|rw-r--r--|소유자 읽기쓰기 / 나머지 읽기|일반 파일|
|600|rw-------|소유자만 읽기쓰기|SSH 개인키 / .env|
|700|rwx------|소유자만 전체|.ssh 디렉토리|
|750|rwxr-x---|소유자 전체 / 그룹 읽기+진입 / 기타 차단|협업 프로젝트 최상위|
|777|rwxrwxrwx|모두 전체|⚠️ 보안 위험|

## chmod 600 — .env / SSH 키 필수 설정 ⭐️

```bash
touch .env
chmod 600 .env          # -rw------- (소유자만 읽기/쓰기)
chmod 600 ~/.ssh/id_rsa # SSH 개인키
```

```txt
chmod 600 이 필요한 이유:
  .env 파일 = DB 비밀번호 / API 키 등 민감 정보
  id_rsa    = SSH 개인키
  → 권한이 너무 열려있으면 (예: 644, 777)
    SSH / Git 등이 보안 취약점으로 판단해 사용 거부

  WARNING: UNPROTECTED PRIVATE KEY FILE!
  → chmod 600 ~/.ssh/id_rsa 로 해결
```

---

---

# ③ chmod — 권한 변경 ⭐️

```bash
# 숫자 방식
chmod 755 script.sh
chmod 644 config.txt
chmod 600 ~/.ssh/id_rsa      # SSH 개인키

# 기호 방식
chmod +x script.sh           # 실행 권한 추가
chmod -x script.sh           # 실행 권한 제거
chmod u+w file.txt           # 소유자(u)에 쓰기(w) 추가
chmod g-w file.txt           # 그룹(g)에서 쓰기(w) 제거
chmod o-r file.txt           # 기타(o)에서 읽기(r) 제거
chmod a+x script.sh          # 전체(a)에 실행 권한 추가

# 디렉토리 하위 전체 적용
chmod -R 755 /opt/myapp/
```

## 기호 방식 기호 정리

```txt
대상:
  u = user (소유자)
  g = group (그룹)
  o = others (기타)
  a = all (전체)

동작:
  + = 권한 추가
  - = 권한 제거
  = = 권한 설정 (나머지 제거)
```

---

---

# ④ chown — 소유자 변경 ⭐️

```txt
기본: chown [소유자]:[그룹] 파일

패턴 1 — 소유자만 변경:      chown 사용자명 파일
패턴 2 — 소유자 + 그룹:      chown 사용자:그룹 파일
패턴 3 — 그룹만 변경:        chown :그룹명 파일  (또는 chgrp)
```

```bash
# 패턴 1 — 소유자만 변경 (그룹은 그대로)
sudo chown labex /etc/apt/sources.list
sudo chown ubuntu file.txt

# 패턴 2 — 소유자 + 그룹 동시 변경
sudo chown ubuntu:ubuntu file.txt
sudo chown labex:root file.txt     # 소유자=labex, 그룹=root 유지

# 패턴 3 — 현재 사용자로 (sudo 로 만든 파일 권한 가져올 때)
sudo chown $USER:$USER file.txt

# 디렉토리 하위 전체 (-R 재귀)
sudo chown -R ubuntu:ubuntu /opt/myapp/
```

## ls -l 출력으로 소유자 확인 ⭐️

```bash
ls -l /etc/apt/sources.list
# -rw------- 1 labex root 2557 Jul 18  2024 /etc/apt/sources.list
#              ↑     ↑
#           소유자  소유그룹
```

```txt
ls -l 출력 읽는 법:
  -rw------- 1 labex root 2557 ...
              ↑소유자 ↑소유그룹

  소유자 = labex  → labex 가 읽기/쓰기 가능
  소유그룹 = root → root 그룹 멤버는 권한 없음 (--- 이므로)
  기타 = ---      → 그 외 사람도 권한 없음

chown 전:
  -rw------- 1 root root  → root 만 읽기/쓰기
  → 일반 사용자 수정 불가

chown labex 후:
  -rw------- 1 labex root → labex 가 읽기/쓰기
  → labex 로 수정 가능
```

## 소유자만 vs 소유자+그룹 구분 ⭐️

```bash
# 소유자만 변경 (그룹은 현재 상태 유지)
sudo chown labex file.txt
# 결과: labex:root (그룹 root 그대로)

# 소유자 + 그룹 둘 다 변경
sudo chown labex:labex file.txt
# 결과: labex:labex (그룹도 labex 로)

# 왜 구분이 중요한가:
#   파일이 특정 그룹 권한으로 공유될 때
#   그룹을 바꾸면 다른 멤버의 접근 권한도 바뀜
```

```txt
sudo 로 만든 파일 → root 소유
→ 일반 사용자가 수정 못 함
→ chown $USER:$USER 로 소유권 가져오기
→ 또는 소유자만 chown $USER 으로 변경
```

---

---

# ⑤ id / whoami — 내 권한 확인

```bash
whoami          # 현재 사용자명
id              # uid / gid / 소속 그룹 전체
id ubuntu       # 특정 사용자 정보

# 출력 예시
# uid=1000(ubuntu) gid=1000(ubuntu) groups=1000(ubuntu),27(sudo)
#                                                          ↑ sudo 그룹 = 관리자 권한
```

---

---

# ⑥ sudo — 관리자 권한 실행

```bash
sudo 명령어          # root 권한으로 실행
sudo -i             # root 쉘 전환
sudo su -           # root 로 전환

# sudo 없이 실패하는 경우
cat /etc/shadow     # Permission denied
sudo cat /etc/shadow # ✅ root 로 읽기
```

```txt
sudo 를 쓸 수 있는 조건:
  /etc/sudoers 에 등록된 사용자
  또는 sudo 그룹 (id 에서 27(sudo) 확인)
```

---

---

# ⑦ 특수 권한

## SUID (Set User ID)

```bash
# 실행 시 파일 소유자 권한으로 실행
chmod u+s 파일
# rws 처럼 x 자리에 s 표시

예시: /usr/bin/passwd
  -rwsr-xr-x root root ...
  일반 사용자가 실행해도 root 권한으로 동작
  → 자기 비밀번호 변경 가능
```

## SGID (Set Group ID)

```bash
chmod g+s 디렉토리
# 디렉토리 안에 만들어지는 파일이 자동으로 같은 그룹 소유
# 팀 공유 폴더에 활용
```

## setgid 실전 — 협업 디렉토리 설정 ⭐️

```txt
문제:
  개발자 A 가 파일 만들면 → A 소유
  개발자 B 가 수정하려면 → 권한 없어서 "Permission denied"

해결: setgid (chmod 2770)
  폴더에 setgid 설정
  → 그 안에서 만드는 파일이 자동으로 폴더의 그룹 소속
  → 같은 그룹이면 누구나 수정 가능
```

```bash
# 협업 디렉토리에 setgid 설정
sudo chmod 2770 ~/project/phoenix_project/src
#              ↑
#              2 = setgid 특수 권한

ls -ld ~/project/phoenix_project/src
# drwxrws---  ← 그룹 x 자리가 s 로 바뀜
#       ↑
#       s = setgid 활성화 표시
```

## chmod 2770 의미

```txt
2   setgid 활성화
7   소유자: rwx (읽기 + 쓰기 + 진입)
7   그룹:   rwx (읽기 + 쓰기 + 진입)
0   기타:   --- (접근 불가)

drwxrws---
     ↑↑↑
     그룹 rwx 인데 x 자리가 s
     → setgid 활성화 상태
```

## 특수 권한 숫자 전체

```txt
setuid  = 앞에 4  →  4xxx  (예: 4755)
setgid  = 앞에 2  →  2xxx  (예: 2770)
sticky  = 앞에 1  →  1xxx  (예: 1777 = /tmp)
복합    = 합산    →  예: setuid+setgid = 6xxx

확인:
  drwxrws---  → s 가 그룹 x 자리 = setgid
  drwxrwt---  → t 가 기타 x 자리 = sticky
  -rwsr-xr-x  → s 가 소유자 x 자리 = setuid
```

## 협업 디렉토리 전체 설정 순서

```bash
# 1. 그룹 생성
sudo groupadd developers

# 2. 사용자를 그룹에 추가
sudo usermod -aG developers dev_lead
sudo usermod -aG developers dev_member

# 3. 소유권 변경 (디렉토리 + 하위 전체)
sudo chown -R dev_lead:developers ~/project/phoenix_project
ls -ld ~/project/phoenix_project/   # 소유권 확인

# 4. 디렉토리 권한 설정
sudo chmod 750 ~/project/phoenix_project      # 최상위: 외부 차단
sudo chmod 2770 ~/project/phoenix_project/src # 협업 폴더: setgid

# 5. 확인
ls -ld ~/project/phoenix_project/src
# drwxrws---  dev_lead developers ...
```

```txt
chmod 750 vs chmod 2770:

  chmod 750 (최상위 디렉토리):
    소유자  rwx
    그룹    r-x  (읽기 + 진입만 / 쓰기 없음)
    기타    ---  (접근 불가)
    → 외부인 차단 / 그룹은 읽기만

  chmod 2770 (협업 디렉토리):
    소유자  rwx
    그룹    rws  (읽기 + 쓰기 + 진입 + setgid)
    기타    ---  (접근 불가)
    → 그룹원 전원 수정 가능 + 파일 그룹 자동 상속
```

## Sticky Bit

```bash
chmod +t 디렉토리
# /tmp 에 적용됨
# 파일 삭제는 소유자만 가능 (쓰기 권한 있어도)
ls -ld /tmp
# drwxrwxrwt ← 끝에 t
```

---

---

# ⑧ SSH 권한 체크리스트

```bash
chmod 700 ~/.ssh                   # .ssh 디렉토리
chmod 600 ~/.ssh/id_rsa            # 개인키
chmod 644 ~/.ssh/id_rsa.pub        # 공개키
chmod 600 ~/.ssh/authorized_keys   # authorized_keys
chmod 600 ~/.ssh/config            # config
```

```txt
권한이 느슨하면 SSH 접속 거부:
  WARNING: UNPROTECTED PRIVATE KEY FILE!
  → chmod 600 ~/.ssh/id_rsa 로 해결
```

---

---

# 자주 하는 실수

|실수|원인|해결|
|---|---|---|
|Permission denied|실행 권한 없음|`chmod +x 파일`|
|sudo 만든 파일 수정 못함|root 소유|`sudo chown $USER:$USER 파일`|
|SSH 접속 거부|개인키 권한 느슨|`chmod 600 ~/.ssh/id_rsa`|
|rm 못함|쓰기 권한 없음|`chmod +w 파일` 또는 `sudo rm`|
|cd 못함|실행 권한 없음|`chmod +x 디렉토리`|