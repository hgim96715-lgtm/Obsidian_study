---
aliases:
  - mv
  - cp
  - 파일이동
  - 파일복사
  - 백업패턴
tags:
  - Linux
related:
  - "[[00_Linux_HomePage]]"
  - "[[Linux_Directory_Commands]]"
  - "[[Linux_File_Delete]]"
  - "[[Linux_Archive_Compress]]"
---

# Linux_File_Move_Copy — 파일 이동 & 복사

## 한 줄 요약

```txt
mv  → 파일 이동 / 이름 변경 (원본 사라짐)
cp  → 파일 복사 (원본 유지)
.bak 패턴 → 수정 전 원본 보존 필수 습관
```

---

---

# ① mv — 파일 이동 / 이름 변경 ⭐️

```txt
기본: mv [원본] [대상]

패턴 1 — 이동:        mv 파일명   디렉토리/
패턴 2 — 이름 변경:   mv 기존이름 새이름
패턴 3 — 이동+이름:   mv 파일명   디렉토리/새이름
```

```bash
# 패턴 1 — 이동
mv main_app.py src/         # src 폴더로 이동
mv config.json config/      # config 폴더로 이동
mv README.md docs/          # docs 폴더로 이동
mv styles.css css/          # css 폴더로 이동

# 패턴 2 — 이름 변경 (같은 디렉토리 안에서)
mv old_name.py new_name.py
mv app.py main.py           # 이름만 바꿀 때

# 패턴 3 — 이동 + 이름 변경 동시에
mv old.py src/new.py        # src 로 이동하면서 이름도 변경

# 여러 파일을 한 폴더로 이동
mv file1.py file2.py file3.py src/

# 와일드카드로 여러 파일 한번에 이동 ⭐️
mv *.js scripts/            # .js 파일 전부 scripts 폴더로
mv *.css css/               # .css 파일 전부 css 폴더로
mv *.log /var/log/archive/  # 로그 파일 전부 이동

# 디렉토리 통째로 이동
mv ~/project/shared_docs ~/project/phoenix_project/docs/
```

## 대상 경로 끝 / 의 의미 ️

```bash
mv styles.css css/    # css/ → 슬래시 있음 = 디렉토리 안으로 이동
mv styles.css css     # css  → 슬래시 없음 = css 라는 이름으로 변경인지 이동인지 모호

# 슬래시를 붙이는 것은 선택 사항이지만
# 파일을 디렉토리 안으로 옮긴다는 의미를 명확하게 해줌
# → 대상이 디렉토리면 / 붙이는 습관 권장
```

```txt
mv 특징:
  이동 후 원본 파일/폴더 사라짐
  디렉토리도 내부 파일 전부 함께 이동
  cp 와 달리 -r 옵션 불필요 (디렉토리 자동 처리)
  와일드카드 사용 가능 (*.js / *.css 등)
```

## mv 주의사항

```bash
# 대상 파일이 이미 있으면 덮어씀 (경고 없음!)
mv a.txt b.txt   # b.txt 가 있으면 b.txt 덮어씌워짐

# 덮어쓰기 전 확인
mv -i a.txt b.txt   # -i = interactive (덮어쓸지 물어봄)

# 덮어쓰기 금지
mv -n a.txt b.txt   # -n = no-clobber (대상 있으면 무시)

# 이동 과정 출력 (-v)
mv -v *.js scripts/
# 'app.js' -> 'scripts/app.js'
# 'util.js' -> 'scripts/util.js'
```

---

---

# ② cp — 파일 복사 ⭐️

```bash
# 기본: cp [원본] [복사본]
cp config.json config.json.bak   # 같은 위치에 복사
cp file.txt /tmp/file.txt        # 다른 경로에 복사

# 디렉토리로 복사 — 끝에 / 붙이면 명확 ⭐️
cp favorite_song.mp3 ~/project/music/
#                                   ↑ / 붙이면 music 이 디렉토리임을 명시

# 여러 파일을 디렉토리로 한번에 복사
cp report1.txt report2.txt notes.txt ~/project/documents/
#  ↑ 공백으로 구분, 마지막이 대상 디렉토리

# 와일드카드로 특정 확장자만 복사 ⭐️
cp ~/project/*.pdf ~/project/pdf_files/   # pdf 파일 전체
cp ~/project/*.txt ~/project/docs/        # txt 파일 전체
cp ~/project/*.log /backup/logs/          # log 파일 전체

# 디렉토리 복사 (-r 필수)
cp -r src/ src_backup/           # src 폴더 전체 복사
cp -r config/ ~/backup/          # 백업 위치로 복사

# 속성 유지하며 복사 (권한 / 타임스탬프)
cp -p file.txt backup.txt
cp -rp src/ src_backup/
```

```txt
cp vs mv:
  cp  원본 유지 / 복사본 생성
  mv  원본 이동 / 원본 사라짐

디렉토리 복사:
  cp 폴더/  대상/    → ❌ 에러 (디렉토리에는 -r 필수)
  cp -r 폴더/ 대상/  → ✅

대상 경로 끝 / 의 의미:
  ~/project/music/   → music 이 디렉토리 (명확)
  ~/project/music    → music 이 파일인지 디렉토리인지 모호
  → 대상이 디렉토리면 / 붙이는 습관 권장
```

## cp 주요 옵션 ⭐️

```bash
cp -i file.txt dest/   # interactive — 덮어쓰기 전 확인
cp -n file.txt dest/   # no-clobber — 기존 파일 덮어쓰기 안 함
cp -u file.txt dest/   # update — 원본이 더 새것일 때만 복사
cp -v file.txt dest/   # verbose — 복사 과정 상세 출력
cp -r src/ dest/       # recursive — 디렉토리 전체 복사
cp -p file.txt dest/   # preserve — 권한/타임스탬프 유지
cp -l file.txt dest/   # link — 파일 대신 하드링크 생성
cp -s file.txt dest/   # symbolic — 파일 대신 심볼릭링크 생성
```

```txt
실무에서 자주 쓰는 조합:
  cp -i   실수로 덮어쓰기 방지 (대화형)
  cp -n   자동화 스크립트에서 안전하게
  cp -u   동기화 (변경된 것만 복사)
  cp -rv  디렉토리 복사 + 과정 확인
  cp -rp  디렉토리 복사 + 권한/날짜 보존

-i vs -n:
  -i  덮어쓸지 물어봄 (사람이 결정)
  -n  아예 덮어쓰지 않음 (자동 건너뜀)
  → 스크립트 자동화: -n
  → 수동 작업 안전하게: -i
```

---

---

# ③ .bak 백업 패턴 ⭐️

```txt
수정 전 원본 보존 → 롤백 가능
인프라 엔지니어의 0순위 철칙
```

```bash
# 설정 파일 수정 전 백업
cp config.json config.json.bak     # 백업 생성
nano config.json                    # 수정

# 잘못 수정했으면 롤백
cp config.json.bak config.json     # 원본으로 복구
```

## 날짜 포함 백업 패턴

```bash
# 날짜 포함 백업 (여러 버전 관리)
cp config.json config.json.$(date +%Y%m%d)
# → config.json.20260420

cp nginx.conf nginx.conf.bak
cp fstab fstab.orig               # .orig 도 자주 씀
cp script.sh script.sh.20260420
```

```txt
확장자 관례:
  .bak   = 일반 백업
  .orig  = 원본 (original)
  .old   = 이전 버전
  .날짜  = 날짜별 버전 관리
```

---

---

# ④ 실전 패턴

## 프로젝트 파일 정리

```bash
cd ~/project/phoenix_project

# 파일들을 용도별 폴더로 정리
mv main_app.py src/
mv config.json config/
mv README.md docs/

# 정리 결과 확인
ls -F
ls -F src/
ls -F config/
```

## 배포 전 설정 백업

```bash
# 중요 설정 파일 수정 전 백업
cp /etc/nginx/nginx.conf /etc/nginx/nginx.conf.bak
cp /etc/ssh/sshd_config /etc/ssh/sshd_config.bak

# 수정 후 문제 생기면 복구
sudo cp /etc/nginx/nginx.conf.bak /etc/nginx/nginx.conf
sudo systemctl restart nginx
```

## 빌드 결과물 배포 이동

```bash
# 빌드된 파일을 배포 경로로 이동
mv dist/ /var/www/html/
mv build/app.py /opt/myapp/

# 스크립트에서 자동화
BUILD_DIR="./dist"
DEPLOY_DIR="/var/www/html"
mv $BUILD_DIR $DEPLOY_DIR
```

---

---

# 자주 하는 실수

|실수|원인|해결|
|---|---|---|
|mv 로 파일 덮어씀|경고 없이 덮어씀|`mv -i` 로 확인|
|cp 로 폴더 복사 에러|`-r` 옵션 없음|`cp -r 폴더/ 대상/`|
|백업 없이 설정 수정|습관 미형성|수정 전 `.bak` 복사 필수|
|mv 후 원본 찾음|mv 가 이동인 줄 몰랐음|복사는 `cp` / 이동은 `mv`|