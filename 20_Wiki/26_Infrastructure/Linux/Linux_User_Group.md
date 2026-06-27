---
aliases:
  - 사용자 관리
  - 그룹 관리
  - useradd
  - usermod
tags:
  - Linux
related:
  - "[[00_Linux_HomePage]]"
  - "[[Linux_Permission_Model]]"
  - "[[Linux_Directory_Structure]]"
  - "[[Linux_Concept_Overview]]"
  - "[[Linux_System_Info]]"
---
# Linux_User_Group — 사용자 & 그룹 관리

## 한 줄 요약

```txt
리눅스는 다중 사용자 시스템
사용자와 그룹으로 "누가 무엇에 접근할 수 있는지" 제어
```

---

---

# ① 사용자 관련 파일 ⭐️

## /etc/passwd — 사용자 정보

```bash
cat /etc/passwd
# username:x:UID:GID:comment:home_dir:shell
# gong:x:1001:1001::/home/gong:/bin/bash
```

```txt
필드 설명:
  username   사용자명
  x          패스워드 (실제 값은 /etc/shadow 에 저장)
  UID        사용자 ID (root=0 / 일반=1000~)
  GID        기본 그룹 ID
  comment    설명 (보통 이름)
  home_dir   홈 디렉토리
  shell      기본 셸 (/bin/bash / /sbin/nologin 등)
```

## /etc/shadow — 암호화된 비밀번호

```bash
sudo cat /etc/shadow
# username:암호화된비밀번호:마지막변경일:최소사용일:최대사용일:...
```

```txt
root 만 읽을 수 있음
비밀번호는 해시로 저장 (SHA-512 등)
! 또는 * 로 시작하면 로그인 비활성화
```

## /etc/group — 그룹 정보

```bash
cat /etc/group
# groupname:x:GID:member1,member2
# docker:x:999:gong,ubuntu
```

---

---

# ② 사용자 생성 / 삭제

## useradd — 사용자 생성

```bash
# 기본 생성 (홈 디렉토리 없음!)
sudo useradd newuser

# 홈 디렉토리 포함 생성 (-m 필수)
sudo useradd -m newuser

# 기본 그룹 + 보조 그룹 + 홈 한번에 생성 ⭐️
sudo useradd -m -d /home/jack -g dev -G labex jack
#                  ↑ 홈 경로  ↑ 기본그룹 ↑ 보조그룹
```

|옵션|의미|
|---|---|
|`-m`|홈 디렉토리 생성 (필수!)|
|`-d /home/이름`|홈 디렉토리 경로 지정|
|`-g 그룹명`|기본 그룹(Primary) 지정|
|`-G 그룹1,그룹2`|보조 그룹(Secondary) 지정|
|`-s /bin/bash`|기본 셸 지정|
|`-c "설명"`|코멘트 추가|
|`-u 1500`|UID 직접 지정|

```bash
# 실전 예시
sudo useradd -m -s /bin/bash -c "데이터 엔지니어" -G docker,sudo gong
```

```txt
useradd vs adduser:
  useradd = 스크립트/자동화 (묵묵히 옵션대로 실행 / 비밀번호 프롬프트 없음)
  adduser = 대화형 (비밀번호/이름 등 입력 요구)
  → 일괄 계정 생성 스크립트 = useradd 사용

-m 옵션이 왜 필요한가:
  기본적으로 홈 디렉토리를 안 만듦
  홈 없으면 로그인 시 프롬프트 깨짐 / .bashrc 없음
  → 반드시 -m 붙이기
```

## 사용자 생성 2가지 방법 ⭐️

```txt
방법 1 — useradd (스크립트/자동화 적합)
  모든 속성을 한 번에 옵션으로 지정
  비밀번호 입력 프롬프트 없음

방법 2 — adduser → usermod (대화형 적합)
  adduser 로 생성 (비밀번호 입력 필요 / UPG 자동 생성)
  이후 usermod -g 로 기본 그룹 변경
  이후 usermod -aG 로 보조 그룹 추가
```

```bash
# ─── 방법 1 — useradd 한 번에 ────────────────────
sudo groupadd dev
sudo useradd -m -d /home/jack -g dev -G labex jack

# 또는 두 단계로 분리:
sudo useradd -m -d /home/jack -g dev jack  # 기본 그룹과 함께 생성
sudo usermod -aG labex jack                # 보조 그룹 추가

# ─── 방법 2 — adduser → usermod ──────────────────
sudo groupadd dev
sudo adduser jack              # 대화형 생성 (비밀번호 입력 요구)
                               # → UPG: jack 그룹 자동 생성 / 기본그룹 = jack
sudo usermod -g dev jack       # 기본 그룹 jack → dev 로 변경
sudo usermod -aG labex jack    # 보조 그룹 labex 추가
```

```txt
usermod -g vs usermod -aG ⭐️:
  usermod -g  그룹명  → 기본 그룹(Primary) 변경 (1개만 존재하므로 -a 불필요)
  usermod -aG 그룹명  → 보조 그룹(Secondary) 추가 (-a 없으면 기존 그룹 전부 날아감!)

두 방법 모두 유효:
  useradd   → 빠르고 자동화에 적합
  adduser   → 대화형 / 실습 환경에 적합
  결과 동일 → 두 방법 모두 이해해두기
```

## /etc/skel — 홈 디렉토리 기본 파일 ⭐️

```bash
# -m 옵션 사용 시 /etc/skel 내용이 신규 홈에 자동 복사
ls -la /etc/skel/
# .bash_logout  .bashrc  .profile

# 신규 사용자 홈에 자동 복사됨
sudo ls -la /home/jack/
# .bash_logout  .bashrc  .profile  (동일한 파일들)
```

```txt
/etc/skel = Skeleton (뼈대)
  모든 신규 사용자의 홈 디렉토리 초기 세팅 거푸집
  여기 파일 추가/수정 → 이후 생성 계정에 자동 반영
```

## passwd — 비밀번호 설정

```bash
sudo passwd newuser     # 비밀번호 설정
passwd                  # 내 비밀번호 변경
```

## userdel — 사용자 삭제

```bash
sudo userdel newuser          # 사용자만 삭제 (홈 디렉토리 유지)
sudo userdel -r newuser       # 사용자 + 홈 디렉토리 함께 삭제
```

---

---

# ③ 사용자 수정

## usermod — 사용자 정보 변경

```bash
sudo usermod -s /bin/zsh gong           # 셸 변경
sudo usermod -d /new/home gong          # 홈 디렉토리 변경
sudo usermod -aG docker gong            # 그룹 추가 ← -a 필수!
sudo usermod -G docker,sudo gong        # 그룹 전체 교체
sudo usermod -l newname oldname         # 이름 변경
sudo usermod -L gong                    # 계정 잠금 (Lock)
sudo usermod -U gong                    # 계정 잠금 해제 (Unlock)
```

```txt
계정 잠금 작동 원리:
  usermod -L → /etc/shadow 비밀번호 앞에 ! 추가
  sudo grep "^gong:" /etc/shadow
  # gong:!$6$해시...  ← ! 가 붙으면 로그인 거부

왜 삭제 대신 잠금인가:
  퇴사자 계정 바로 삭제 시
  → 그 사람 파일들이 "고아 파일" 로 변함 (UID 가 숫자만 남음)
  → 파일 소유권 관리 꼬임

  잠금 처리:
  → 로그인 불가 (보안)
  → 파일 소유권 유지 (데이터 보존)
  → 나중에 복구 가능
```

```txt
⚠️ -aG vs -G 차이:
  -aG docker  → 기존 그룹 유지 + docker 추가  ✅
  -G docker   → 기존 그룹 전부 제거 + docker 만 남김  ❌ 위험!

  그룹 추가할 땐 반드시 -aG (append + Group)
```

---

---

# ④ 그룹 생성 / 삭제 / 수정

## groupadd — 그룹 생성

```bash
sudo groupadd research             # 그룹 생성
sudo groupadd -g 2000 mygroup      # GID 지정하며 생성

# 생성 확인 → /etc/group 에서 확인
grep research /etc/group
# research:x:5003:
#   ↑이름   ↑비밀번호 ↑GID ↑멤버(비어있음)
```

```txt
/etc/group 형식:
  그룹명 : 비밀번호 : GID : 멤버목록
  research:x:5003:labex,gong
```

```txt
⚠️ sudo 없으면 permission denied
   그룹 관리는 반드시 관리자 권한 필요
```

## usermod -aG — 보조 그룹에 사용자 추가 ⭐️

```bash
# 그룹에 사용자 추가
sudo usermod -aG developers jack

# sudo 권한 부여 — sudo 그룹에 추가
sudo usermod -aG sudo jack

# 추가 확인
groups jack
# jack : jack sudo developers  ← 전체 그룹 목록

grep jack /etc/group
# sudo:x:27:jack
# developers:x:1001:jack
```

```txt
-a = Append (기존 그룹 유지하며 추가)
-G = 보조 그룹 지정

-a 없이 -G 만 쓰면:
  sudo usermod -G developers jack
  → jack 이 기존에 속한 sudo 등 전부 제거됨!
  → developers 만 남음 → 매우 위험!

반드시 -aG 세트로 사용
```

## sudo 그룹 추가 — 관리자 권한 부여 ⭐️

```bash
sudo usermod -aG sudo jack

# 확인
groups jack
# jack : jack sudo developers  ← sudo 포함

# 변경사항 적용 (재로그인 필요)
su - jack   # jack 으로 로그인 전환
sudo whoami # root
```

```txt
sudo 그룹에 추가하면:
  해당 사용자가 sudo 명령어 사용 가능
  root 로 직접 로그인 없이 관리 작업 수행
  작업 기록 남음 (감사 추적)

주의:
  sudo 권한은 신중하게 부여
  필요한 사용자에게만 추가
```

## UPG — 사용자 개인 그룹 ⭐️

```txt
adduser 로 사용자 생성 시 자동으로:
  사용자 이름과 같은 이름의 그룹 생성
  그 그룹을 기본 그룹(Primary Group)으로 설정

예시: adduser jack
  → jack 사용자 생성
  → jack 그룹 자동 생성
  → jack 의 기본 그룹 = jack

id jack
# uid=1001(jack) gid=1001(jack) groups=1001(jack)
#               ↑ uid           ↑ 기본 그룹 gid
```

## 멤버십 확인 — grep / groups

```bash
# 특정 그룹에 누가 있는지
grep research /etc/group
# research:x:5003:labex

# 특정 사용자가 어느 그룹에 있는지
groups labex
# labex : labex sudo ssl-cert public research

# /etc/group 전체에서 사용자 검색
grep labex /etc/group
# sudo:x:27:labex
# ssl-cert:x:121:labex
# labex:x:5000:
# public:x:5002:labex
# research:x:5003:labex
```

```txt
grep vs groups:
  grep 그룹명 /etc/group  → 그룹에 어떤 멤버가 있는지
  groups 사용자명         → 사용자가 어떤 그룹에 속하는지
```

## groupdel — 그룹 삭제

```bash
sudo groupdel research

# 삭제 확인 → 아무것도 안 나오면 성공
grep research /etc/group
# (아무 출력 없음)
```

```txt
주의:
  사용자의 기본 그룹(Primary Group)은 삭제 불가
  → 해당 사용자가 있으면 먼저 사용자 삭제 또는 기본 그룹 변경

  그룹 삭제 후 파일 소유권:
  → 해당 그룹이 소유한 파일의 GID 가 숫자로 남음
```

## groupmod — 그룹 수정

```bash
sudo groupmod -n newname oldname   # 그룹 이름 변경
sudo groupmod -g 3000 mygroup      # GID 변경
```

---

---

# ⑤ 사용자 정보 확인

```bash
whoami                  # 현재 사용자명
id                      # UID / GID / 소속 그룹 전체
id gong                 # 특정 사용자 정보
groups                  # 현재 사용자 소속 그룹 목록
groups gong             # 특정 사용자 그룹 목록

who                     # 현재 로그인된 사용자
w                       # 로그인 사용자 + 작업 내용
last                    # 최근 로그인 이력
```

```bash
id gong
# uid=1001(gong) gid=1001(gong) groups=1001(gong),999(docker),27(sudo)
#  ↑UID          ↑기본GID        ↑소속된 모든 그룹
```

---

---

# ⑥ UID / GID 완전 정리 ⭐️

## UID — 사용자 식별 번호

```txt
UID = User ID
리눅스가 사용자를 구분하는 숫자
이름이 아닌 UID 로 실제 권한 처리

범위:
  0          root (최고 권한 / 슈퍼유저)
  1 ~ 999    시스템 사용자 (데몬 / 서비스용 / 사람이 아님)
  1000 ~     일반 사용자 (사람이 쓰는 계정)
```

```bash
id gong
# uid=1001(gong)   ← UID 1001 = 일반 사용자
id root
# uid=0(root)      ← UID 0 = root

cat /etc/passwd | grep gong
# gong:x:1001:1001::/home/gong:/bin/bash
#           ↑UID  ↑GID
```

## GID — 그룹 식별 번호

```txt
GID = Group ID
그룹을 구분하는 숫자
사용자는 기본 그룹(Primary) 1개 + 보조 그룹(Secondary) 여러 개 가질 수 있음

범위:
  0          root 그룹
  1 ~ 999    시스템 그룹
  1000 ~     일반 그룹
```

```bash
cat /etc/group | grep docker
# docker:x:999:gong
#    ↑이름 ↑비번 ↑GID ↑멤버
```

## Primary Group vs Secondary Group

```txt
Primary Group (기본 그룹):
  사용자가 파일 생성 시 자동으로 부여되는 그룹
  /etc/passwd 의 GID 필드
  useradd 시 자동으로 사용자명과 같은 그룹 생성

Secondary Group (보조 그룹):
  추가로 속한 그룹 (권한 확장용)
  usermod -aG 로 추가
  docker / sudo / 팀 그룹 등
```

```bash
id gong
# uid=1001(gong) gid=1001(gong) groups=1001(gong),999(docker),27(sudo)
#                ↑Primary GID   ↑Secondary Groups (전부 포함)
```

## UID / GID 로 파일 소유권 확인

```bash
ls -l
# -rw-r--r-- 1 gong gong 1234 Apr 6 file.txt
#              ↑소유자  ↑소유그룹

# 실제로는 이름이 아닌 UID/GID 로 저장됨
# 사용자 삭제하면 이름 대신 숫자로 표시됨
ls -l
# -rw-r--r-- 1 1001 1001 1234 Apr 6 file.txt  ← 사용자 삭제 후
```

## UID / GID 직접 지정

```bash
# UID 지정하며 생성
sudo useradd -u 1500 myuser

# GID 지정하며 그룹 생성
sudo groupadd -g 2000 mygroup

# 특정 그룹으로 생성
sudo useradd -u 1500 -g 2000 myuser

# 현재 시스템에서 사용 중인 UID 확인
cat /etc/passwd | awk -F: '{print $3}' | sort -n
```

## /sbin/nologin 셸

```txt
시스템 사용자는 로그인 불가하도록 셸을 nologin 으로 설정
mysql / nginx / kafka 등 서비스 계정에 사용

로그인 시도하면:
  "This account is currently not available" 출력 후 차단
```

```bash
# 서비스 계정 생성 예시
sudo useradd -r -s /sbin/nologin kafka
# -r  시스템 계정 (UID 1000 미만 자동 할당)

cat /etc/passwd | grep kafka
# kafka:x:998:998::/home/kafka:/sbin/nologin
```

---

---

# ⑦ 실전 패턴

## Docker 그룹 추가 (sudo 없이 docker 쓰기)

```bash
sudo usermod -aG docker $USER
# 적용하려면 재로그인 또는
newgrp docker
```

## 서비스 전용 계정 생성

```bash
# 로그인 불가 / 홈 없는 서비스 계정
sudo useradd -r -s /sbin/nologin kafka
# -r  시스템 계정 (UID 1000 미만)
# -s /sbin/nologin  로그인 차단
```

## 사용자 생성 후 확인

```bash
sudo useradd -m -s /bin/bash gong
id gong
cat /etc/passwd | grep gong
ls /home/gong
```

---

---

# 한눈에 정리

|명령어|역할|
|---|---|
|`useradd -m -s /bin/bash 이름`|사용자 생성|
|`passwd 이름`|비밀번호 설정|
|`userdel -r 이름`|사용자 + 홈 삭제|
|`usermod -aG 그룹 이름`|그룹 추가|
|`usermod -L / -U 이름`|잠금 / 해제|
|`groupadd 그룹명`|그룹 생성|
|`groupdel 그룹명`|그룹 삭제|
|`id 이름`|사용자 UID/GID/그룹 확인|
|`groups 이름`|소속 그룹 확인|

---

---

# 자주 하는 실수

|실수|원인|해결|
|---|---|---|
|`usermod -G docker` 후 기존 그룹 사라짐|`-a` 없이 `-G` 사용|`usermod -aG docker 이름`|
|`useradd` 후 홈 디렉토리 없음|`-m` 옵션 빠짐|`useradd -m 이름`|
|그룹 추가 후 적용 안 됨|재로그인 필요|`newgrp 그룹명` 또는 재로그인|
|사용자 삭제 후 파일 남음|`-r` 없이 삭제|`userdel -r 이름`|
|docker 그룹인데 sudo 필요|그룹 적용 안 됨|재로그인 또는 `newgrp docker`|