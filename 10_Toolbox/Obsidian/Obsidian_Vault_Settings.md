---
aliases:
  - Obsidian vault 설정
tags:
  - Obsidian
related:
  - "[[00_Obsidian_HomePage]]"
  - "[[Obsidian_Plugins]]"
---
# Obsidian_Vault_Settings — vault · 테마 · 코어 설정

> [!note]
> `.obsidian/` 폴더는 gitignore 대상이라 **설정이 바뀌면 이 노트도 같이 고칠 것**.

---

# 테마 · 외관

| 항목 | 값 |
|---|---|
| CSS 테마 | **Minimal** |
| 기본 글자 크기 | 16 |
| 반투명(translucency) | ON |
| CSS 스니펫 | `global` 활성 (`snippets/global.css`) |

`global.css` — Callout(info/note/tip 등) 보라 톤 통일, 패딩·테두리 조정.

---

# Vault 동작 (`app.json`)

| 설정 | 값 | 의미 |
|---|---|---|
| `attachmentFolderPath` | `99_Assets(이미지&첨부파일저장소)` | 이미지·첨부 저장 위치 |
| `alwaysUpdateLinks` | true | 파일 이동 시 링크 자동 갱신 |
| `readableLineLength` | false | 줄 길이 제한 없음 |
| `newFileLocation` | current | 새 노트 = 현재 폴더 |
| `foldHeading` / `foldIndent` | true | 제목·들여쓰기 접기 |
| `showInlineTitle` | true | 편집 화면에 제목 인라인 표시 |

---

# 템플릿

| 항목 | 값 |
|---|---|
| 템플릿 폴더 | `30_Resources/Templates` |
| 플러그인 | Templater (`templater-obsidian`) — [[Obsidian_Plugins]] |

---

# 코어 플러그인 — 켜짐 / 꺼짐

## ON (자주 쓰는 것)

| 플러그인 | 용도 |
|---|---|
| File explorer | 파일 탐색 |
| Global search | 기본 검색 (Omnisearch와 병행) |
| Quick switcher | `Cmd+O` 파일 전환 |
| Graph view | 위키링크 그래프 |
| Canvas | 캔버스 보드 |
| Bookmarks | 북마크 |
| Outline | 목차 패널 |
| Templates | 기본 템플릿 (Templater와 별도) |
| Footnotes | 각주 |
| Properties | frontmatter 속성 |
| Page preview | 링크 호버 미리보기 |
| Slash commands | `/` 명령 |
| Editor status | 하단 상태줄 |
| File recovery | 자동 복구 |
| Sync | Obsidian Sync |
| Web viewer | 웹 뷰어 |

## OFF (의도적으로 끔)

| 플러그인 | 끈 이유 (추정) |
|---|---|
| Backlinks | 패널 안 씀 — 필요 시 켜기 |
| Outgoing links | 동일 |
| Tag pane | 태그 패널 대신 검색·Dataview |
| Daily notes | Calendar 플러그인 사용 |
| Note composer | 안 씀 |
| Command palette | Slash command 사용 |
| Word count | |
| Workspaces | |

---

# 설정 파일 위치 참고

```
.obsidian/
├── app.json              # vault 동작
├── appearance.json       # 테마·스니펫
├── community-plugins.json # 활성 커뮤니티 플러그인 ID 목록
├── core-plugins.json     # 코어 ON/OFF
├── snippets/global.css   # Callout 스타일
└── templates.json        # 템플릿 폴더 경로
```
