---
aliases:
  - Git SSH
  - SSH 키
  - ssh-keygen
tags:
  - Git
related:
  - "[[00_Git_HomePage]]"
  - "[[AWS_Lightsail]]"
---

# Git_SSH — SSH 키 생성 & GitHub 연결

```txt
SSH = 비밀번호 없이 GitHub 에 안전하게 접속하는 방법
키 한 번 등록하면 git clone / push / pull 시 인증 불필요
```

---

---

# SSH vs HTTPS ⭐️

```txt
HTTPS 방식:
  git clone https://github.com/...
  → 매번 토큰(Personal Access Token) 입력 필요

SSH 방식:
  git clone git@github.com:...
  → 키 한 번 등록하면 이후 인증 없이 사용
  → 서버 자동 배포 환경에서 필수
```

---

---

# SSH 키 생성 ⭐️

## 기존 키 확인 먼저

```bash
ls ~/.ssh/
```

```txt
id_ed25519       → 개인키 (절대 공유 X)
id_ed25519.pub   → 공개키 (GitHub 에 등록)

파일 있으면 → 새로 생성 안 해도 됨
파일 없으면 → 아래에서 생성
```

## 키 생성

```bash
# ed25519 방식 (권장 / 최신 알고리즘)
ssh-keygen -t ed25519 -C "your@email.com"

# 또는 rsa 방식 (구형 환경 호환)
ssh-keygen -t rsa -b 4096 -C "your@email.com"
```

```txt
Enter file in which to save the key:
→ 엔터 (기본 경로 사용: ~/.ssh/id_ed25519)
   또는 커스텀 이름 입력: ~/.ssh/github_ed25519

Enter passphrase:
→ 엔터 (비밀번호 없이 사용)
   설정하면 git 사용할 때마다 비밀번호 입력 필요

⚠️ 파일 이름을 기본값(id_ed25519)이 아닌 커스텀으로 바꾸면
   SSH 가 키를 자동으로 찾지 못함 → config 파일 설정 필요 (아래 참고)
```

---

---

# SSH Agent 에 키 등록 ⭐️

```bash
# SSH Agent 실행
eval "$(ssh-agent -s)"

# 키 등록
ssh-add ~/.ssh/id_ed25519

# 커스텀 이름으로 만들었으면
ssh-add ~/.ssh/github_ed25519

# 등록된 키 확인
ssh-add -l
```

```txt
SSH Agent 란:
  비밀키를 메모리에 올려두고 필요할 때 자동으로 사용
  passphrase 설정 시 Agent 에 등록하면 한 번만 입력해도 됨
```

---

---

# 공개키 → GitHub 등록 ⭐️

```bash
# 공개키 내용 출력
cat ~/.ssh/id_ed25519.pub

# 커스텀 이름이면
cat ~/.ssh/github_ed25519.pub
```

```txt
ssh-ed25519 AAAAC3Nza... your@email.com
↑ 이 전체 내용 복사
```

```txt
GitHub → Settings → SSH and GPG keys
→ New SSH key
→ Title: 알아보기 쉬운 이름 (예: MacBook, aws_lightsail)
→ Key: 복사한 공개키 붙여넣기
→ Add SSH key
```

---

---

# 연결 확인

```bash
ssh -T git@github.com
```

```txt
성공:
  Hi {username}! You've successfully authenticated...

실패:
  git@github.com: Permission denied (publickey)
  → 아래 트러블슈팅 참고
```

---

---

# 커스텀 키 이름 사용 시 — config 파일 설정 ⭐️

```txt
키 파일명을 기본값(id_ed25519)이 아닌 다른 이름으로 만들었으면
SSH 가 어떤 키를 써야 하는지 모름 → Permission denied 발생

~/.ssh/config 파일을 만들어서 명시적으로 지정해줘야 함
```

```bash
nano ~/.ssh/config
```

```txt
Host github.com
    HostName github.com
    User git
    IdentityFile ~/.ssh/github_ed25519
    IdentitiesOnly yes
```

```bash
# config 파일 권한 설정 (필수)
chmod 600 ~/.ssh/config
```

```txt
IdentityFile:
  어떤 키 파일을 사용할지 명시
  기본값 이름(id_ed25519) 쓰면 이 설정 불필요

IdentitiesOnly yes:
  지정한 키만 사용 (다른 키 시도 안 함)
```

---

---

# ⚠️ 트러블슈팅 — Permission denied ⭐️

## 원인 1 — 파일 권한 문제 (가장 흔한 원인)

```bash
# 권한 확인
ls -la ~/.ssh/

# 권한 설정
chmod 700 ~/.ssh                        # .ssh 폴더
chmod 600 ~/.ssh/id_ed25519             # 개인키
chmod 644 ~/.ssh/id_ed25519.pub         # 공개키
chmod 600 ~/.ssh/config                 # config 파일
```

```txt
SSH 는 권한이 너무 열려있으면 보안 위협으로 보고 키 무시
→ 개인키는 반드시 600 (본인만 읽기)
```

## 원인 2 — GitHub 에 등록한 키와 로컬 키 불일치

```bash
# 로컬 공개키 확인
cat ~/.ssh/id_ed25519.pub

# GitHub 에 등록된 키와 비교
# GitHub → Settings → SSH keys → 등록된 키 확인
```

```txt
로컬에서 새로 키를 만들었는데 GitHub 에 이전 키가 등록돼 있으면
→ GitHub 에서 기존 키 삭제 후 새 공개키 다시 등록
```

## 원인 3 — 커스텀 키 이름인데 config 파일 없음

```bash
# config 파일 만들기
nano ~/.ssh/config

# 내용 입력
Host github.com
    HostName github.com
    User git
    IdentityFile ~/.ssh/github_ed25519
    IdentitiesOnly yes

# 권한 설정
chmod 600 ~/.ssh/config
```

## 원인 4 — Agent 에 키가 등록 안 됨

```bash
# 등록된 키 확인
ssh-add -l

# 아무것도 없으면 등록
ssh-add ~/.ssh/github_ed25519
```

## 디버그 모드로 상세 확인

```bash
ssh -vT git@github.com
```

```txt
-v = verbose (상세 출력)
어느 단계에서 실패하는지 확인 가능

출력에서 확인:
  Offering public key: ~/.ssh/... → 어떤 키를 시도하는지
  Authenticated to github.com    → 성공
  Permission denied              → 어느 단계에서 실패하는지
```

---

---

# SSH 로 git clone ⭐️

```bash
# HTTPS → SSH 방식으로 변경
# 기존:
git clone https://github.com/{username}/{repo}.git

# SSH 방식:
git clone git@github.com:{username}/{repo}.git
```

```bash
# 기존 로컬 레포의 remote 를 SSH 로 변경
git remote set-url origin git@github.com:{username}/{repo}.git

# 확인
git remote -v
```

---

---

# 핵심 흐름 정리

```txt
ssh-keygen -t ed25519 -C "email"
    ↓
eval "$(ssh-agent -s)"
ssh-add ~/.ssh/id_ed25519
    ↓
cat ~/.ssh/id_ed25519.pub  →  복사
    ↓
GitHub → Settings → SSH keys → 등록
    ↓
ssh -T git@github.com  →  성공 확인
    ↓
git clone git@github.com:...

──────────────────────────────────
커스텀 키 이름 사용 시 추가:
  nano ~/.ssh/config  →  IdentityFile 명시
  chmod 600 ~/.ssh/config
```