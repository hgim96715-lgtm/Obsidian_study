

## 한 줄 요약

`settings.json`은 VS Code의 동작 방식, 포맷팅, 저장 시 자동 정리, 언어별 설정을 직접 관리하는 설정 파일이다.

---

# ⭐️ settings.json 여는 방법

```txt
Command Palette 열기
Mac: Cmd + Shift + P

검색:
Preferences: Open User Settings (JSON)
```

또는:

```txt
VS Code Settings UI
→ 오른쪽 위 문서 아이콘 클릭
→ settings.json 열기
```

---

#  자주 쓰는 기본 설정

```json
{
    "window.autoDetectColorScheme": true,
    "git.autofetch": true,
    "workbench.iconTheme": "material-icon-theme",

    "editor.detectIndentation": false,
    "editor.insertSpaces": true,
    "editor.tabSize": 4,
    "editor.indentSize": 4,
    "editor.formatOnSave": true
}
```

## 의미

```txt
window.autoDetectColorScheme
→ Mac 시스템 다크/라이트 모드에 맞춰 VS Code 테마 자동 변경

git.autofetch
→ Git 변경사항 자동 감지

workbench.iconTheme
→ 파일 아이콘 테마 설정

editor.detectIndentation
→ 파일마다 들여쓰기 자동 추측하지 않음

editor.insertSpaces
→ Tab 대신 공백 사용

editor.tabSize
→ Tab 한 번 = 공백 4칸

editor.indentSize
→ 들여쓰기 크기 4칸

editor.formatOnSave
→ 저장할 때 자동 포맷팅
```

---

#  JavaScript / TypeScript 설정

```json
{
    "[javascript]": {
        "editor.defaultFormatter": "esbenp.prettier-vscode",
        "editor.insertSpaces": true,
        "editor.tabSize": 4,
        "editor.indentSize": 4
    },
    "[typescript]": {
        "editor.defaultFormatter": "esbenp.prettier-vscode",
        "editor.insertSpaces": true,
        "editor.tabSize": 4,
        "editor.indentSize": 4
    }
}
```

## 의미

```txt
JavaScript, TypeScript 파일은 Prettier로 포맷팅한다.
저장할 때 들여쓰기 기준은 공백 4칸으로 맞춘다.
```

---

#  ⭐️ JSON / Markdown 설정

```json
{
    "[json]": {
        "editor.defaultFormatter": "esbenp.prettier-vscode",
        "editor.insertSpaces": true,
        "editor.tabSize": 4,
        "editor.indentSize": 4
    },
    "[markdown]": {
        "diffEditor.ignoreTrimWhitespace": true
    }
}
```

## 의미

```txt
JSON
→ Prettier로 정리

Markdown
→ diff 볼 때 공백 차이를 너무 민감하게 보지 않음
```

---

#  ⭐️ 현재 사용 중인 전체 예시

```json
{
    "window.autoDetectColorScheme": true,
    "git.autofetch": true,
    "workbench.iconTheme": "material-icon-theme",

    "[markdown]": {
        "diffEditor.ignoreTrimWhitespace": true
    },

    "editor.detectIndentation": false,
    "editor.insertSpaces": true,
    "editor.tabSize": 4,
    "editor.indentSize": 4,
    "editor.formatOnSave": true,

    "[javascript]": {
        "editor.defaultFormatter": "esbenp.prettier-vscode",
        "editor.insertSpaces": true,
        "editor.tabSize": 4,
        "editor.indentSize": 4
    },

    "[typescript]": {
        "editor.defaultFormatter": "esbenp.prettier-vscode",
        "editor.insertSpaces": true,
        "editor.tabSize": 4,
        "editor.indentSize": 4
    },

    "[json]": {
        "editor.defaultFormatter": "esbenp.prettier-vscode",
        "editor.insertSpaces": true,
        "editor.tabSize": 4,
        "editor.indentSize": 4
    }
}
```

---

# ⭐️ 주의할 점

```txt
1. settings.json은 JSON 파일이라 주석을 못 쓴다.
2. 마지막 속성 뒤에는 쉼표를 붙이지 않는다.
3. formatter 설정을 했는데 동작하지 않으면 해당 확장 프로그램이 설치되어 있는지 확인한다.
4. 프로젝트에 .prettierrc, eslint 설정이 있으면 프로젝트 설정이 우선될 수 있다.
```

---

# 최종 한 줄 요약

`settings.json`은 VS Code의 작업 방식을 고정하는 설정 파일이고, 포맷팅·들여쓰기·언어별 formatter를 정리해두면 프로젝트마다 코딩 스타일이 흔들리지 않는다.