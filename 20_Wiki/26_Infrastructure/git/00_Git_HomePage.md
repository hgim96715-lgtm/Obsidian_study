

```
버전 관리 + 협업 도구
코드의 변경 이력을 추적하고 팀과 함께 작업하는 기술
```

---

---

## Level 0. 개념 잡기

|노트|핵심 개념|
|---|---|
|[[Git_Concept]]|버전 관리란 / 로컬 vs 원격 / Staging Area / HEAD|
|[[Git_Install_Setup]]|git init / git config / --global vs 로컬 / user.name / alias / autocrlf / color.ui|

---

---

## Level 1. 기본 명령어

| 노트               | 핵심 개념                                                                              |
| ---------------- | ---------------------------------------------------------------------------------- |
| [[Git_Commands]] | add / commit / git diff / git diff --staged / push / reset --soft / commit --amend |
| [[Git_Log]] ⭐    | git log / --oneline / --stat / -p / --grep / -S pickaxe / --since / git shortlog   |
| [[Git_Branch]]   | branch / switch -c / merge / cherry-pick / rebase -i / branch -d                   |
| [[Git_Tags]]     | 경량 태그 / 주석 태그 -a -m / tag 목록 / detached HEAD / push --tags                         |
| [[Git_Remote]]   | remote / fetch / pull / clone/submodule                                            |


---

---

## Level 2. 실전 패턴

| 노트                | 핵심 개념                                                                  |
| ----------------- | ---------------------------------------------------------------------- |
| [[Git_Undo]]      | restore --staged / restore / reset --soft·hard / revert / stash        |
| [[Git_Stash]] ⭐   | stash / stash -u / push -m / apply / pop / drop / clear / stash branch |
| [[Git_Conflict]]  | 충돌 해결 / merge vs rebase / 전략                                           |
| [[Git_Gitignore]] | .gitignore 작성법 / 이미 올라간 파일 제거                                          |

---

---

## Level 3. 협업 & 워크플로우

| 노트                        | 핵심 개념                                     |
| ------------------------- | ----------------------------------------- |
| [[Git_Flow]]              | main / develop / feature 브랜치 전략           |
| [[Git_PR_Review]]         | Pull Request / Code Review / squash merge |
| [[Git_Commit_Convention]] | 커밋 메시지 컨벤션 / feat / fix / chore / docs    |

---

---
## Level 4. CI/CD 자동화

```
코드 변경 → 자동 빌드 / 테스트 / 배포
```

| 노트                      | 핵심 개념                                                                                                       |
| ----------------------- | ----------------------------------------------------------------------------------------------------------- |
| [[Git_GitHubActions]] ⭐ | CI/CD / env / secrets / setup-node / 빌드·아티팩트 개념 ⭐️ / upload-artifact️/Matrix Build/Job Dependencies — needs |

----
---
## SSH 설정

|노트|핵심 개념|
|---|---|
|[[Git_SSH]] ⭐|ssh-keygen / SSH Agent / GitHub 등록 / config 파일 / 권한 설정 / Permission denied 트러블슈팅|
