
```txt
GUI 없이 명령어 하나로 시스템 전체를 제어하는 기술
```

---

---

## Level 0. 개념 잡기

```txt
Linux가 뭔지, 왜 데이터 엔지니어에게 필수인지
```

|노트|핵심 개념|
|---|---|
|[[Linux_Terminal_Basics]]|echo / date / cal / expr / clear / Tab / history / type / man / apropos|
|[[Linux_Concept_Overview]]|커널 / 쉘 / 배포판(Ubuntu) / CLI vs GUI / 왜 서버는 Linux인가|
|[[Linux_Directory_Structure]]|/ (루트) / home / etc / var / tmp / opt / 절대경로 vs 상대경로|
|[[Linux_Permission_Model]]|rwx / 소유자·그룹·기타 / chmod / chown / uid / gid / sudo|
|[[Linux_User_Group]]|useradd / userdel / usermod / passwd / groupadd / 계정 잠금 / /etc/skel|

---

---

## Level 1. 파일 시스템 조작

```txt
서버에서 파일과 폴더를 자유자재로 다루자
```

| 노트                             | 핵심 개념                                                          |
| ------------------------------ | -------------------------------------------------------------- |
| [[Linux_Directory_Commands]] ⭐ | mkdir / touch / {1..5} 확장 / cd / ls  / pwd / tree / 프로젝트 구조 설계 |
| [[Linux_File_Move_Copy]] ⭐     | mv (이동·이름변경) / cp / cp .bak 백업 패턴 / 롤백                         |
| [[Linux_File_Delete]]          | rm / rm -rf / rmdir / 와일드카드(*) 패턴 삭제 / 주의사항                    |
| [[Linux_Archive_Compress]] ⭐   | tar -czf / tar -tzf / tar -xzf / gzip / 로그 로테이션 패턴             |

---

---

## Level 2. 시스템 모니터링

```txt
서버가 지금 어떤 상태인지 읽는 법
```

| 노트                          | 핵심 개념                                                                                 |
| --------------------------- | ------------------------------------------------------------------------------------- |
| [[Linux_System_Info]] ⭐     | whoami / uname -a / uptime / id / hostname / 시스템 점검 루틴                                |
| [[Linux_Process_Monitor]] ⭐ | top / htop / ps aux / pgrep / pkill / kill / PID / Load Average / 좀비 프로세스             |
| [[Linux_Background_Jobs]]   | nohup / & / fg / bg / jobs / 2>&1 / 백그라운드 실행 / 로그아웃 후에도 유지                            |
| [[Linux_Disk_Memory]]       | df -h / du -sh / free -h / mount / umount / fdisk / 파티션·파일시스템·마운트 개념 / 디스크 100% 장애 대응 |
| [[Linux_Network_Check]]     | ip addr / ping -c / ss -tlnp / lsof -i :포트 ⭐️ / lsof -nP -iTCP -sTCP:LISTEN / curl    |
| [[Linux_Log]] ⭐             | dmesg / journalctl / syslog / logrotate / 에러 로그 추적                                    |

---

---

## Level 3. 텍스트 처리 & 검색

```txt
로그 파일에서 필요한 정보를 뽑아내자
```

|노트|핵심 개념|
|---|---|
|[[Linux_Text_Commands]]|cat / nl / less / more / head / tail -f / nano / 실시간 로그 모니터링|
|[[Linux_Search_Filter]] ⭐|grep / 와일드카드 / 파이프 / find / which / PATH / whereis|
|[[Linux_Text_Processing]]|cut / sort / uniq / wc / awk($0·$1·NR·END) / sed(s///g·-i) / tr / join(파일 간 SQL JOIN 개념) / paste / col|
|[[Linux_Xargs]] ⭐|find\|xargs 조합 / -I 자리표시자 / -0 공백파일명 안전 / -P 병렬실행 / -p 확인|
|[[Linux_Redirect]] ⭐|>> / 2> / 2>&1 / &> / /dev/null / 파일 비우기 / stdin·stdout·stderr|
|[[Linux_Diff]]|diff / diff -r / 스테이징 vs 프로덕션 비교 / 누락 파일 찾기|


---

---

## Level 4. 쉘 스크립트 & 자동화

```txt
반복 작업을 스크립트로 만들어서 자동화하자
```

| 노트                            | 핵심 개념                                               |
| ----------------------------- | --------------------------------------------------- |
| [[Linux_Shell_Script_Basics]] | #!/bin/bash / 변수 / if / for / while / 실행권한 chmod +x |
| [[Linux_Shell_Cron_Job]] ⭐    | crontab -e / 크론 표현식 / 로그 압축 자동화 / 백업 스케줄링           |


---

---

## Level 5. 실전 팁

| 노트                                | 핵심 개념                                                                     |
| --------------------------------- | ------------------------------------------------------------------------- |
| [[Linux_Firewall]]                | ufw / allow / deny / enable / SSH 먼저 허용 ⚠️ / Docker 주의사항                  |
| [[Linux_Environment_Variables]] ⭐ | 쉘변수 vs export vs 영구변수 / PATH 수정 / source vs bash / unset / .zshrc         |
| [[Linux_Package_APT]]             | apt update / apt install -y / apt show / apt remove / autoremove / 의존성 관리 |
| [[Linux_Curl_API]] ⭐                  | -sS 패턴 / .env export 방식 / 공공 API 테스트 / POST·헤더 / xmllint·jq 출력            |


---

---

## 프로젝트 적용

|노트|핵심 개념|
|---|---|
|[[Linux_FFmpeg]] ⭐|FFmpeg 개념 / 설치 / 포맷변환 / 압축 / 썸네일 추출 / ffprobe / 주요 옵션|
