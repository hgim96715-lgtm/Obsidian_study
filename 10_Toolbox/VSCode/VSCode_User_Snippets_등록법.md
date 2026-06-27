

## 한 줄 요약

`User Snippets`는 자주 쓰는 코드 구조를 짧은 키워드로 자동 입력하는 VS Code 기능이다.

---

# 1. ⭐️ 언제 쓰는가?

반복해서 자주 쓰는 코드가 있을 때 사용한다.

```txt
예시:
describe / it 테스트 구조
NestJS Controller 기본 구조
NestJS Service 기본 구조
Jest mock 기본 구조
console.log 디버깅 구조
```

단, 아직 코드 흐름을 배우는 단계라면 무조건 스니펫부터 만들기보다 직접 작성하면서 익히는 것도 좋다.

---

# 2. ⭐️ 등록 위치

```txt
Cmd + Shift + P
→ Configure User Snippets
→ 언어 선택
```

예시:

```txt
typescript.json
javascript.json
markdown.json
```

또는 전역으로 쓰고 싶으면:

```txt
New Global Snippets file
```

---

# 3. ⭐️ 기본 구조

```json
{
    "스니펫 이름": {
        "prefix": "입력할 단축어",
        "body": [
            "실제로 입력될 코드 1줄",
            "실제로 입력될 코드 2줄"
        ],
        "description": "스니펫 설명"
    }
}
```

---

# 4. ⭐️ 간단한 예시

```json
{
    "Console Log": {
        "prefix": "clg",
        "body": ["console.log($1);"],
        "description": "console.log shortcut"
    }
}
```

## 사용 흐름

```txt
clg 입력
→ Tab 또는 Enter
→ console.log();
```

---

# 5. ⭐️ 자리 표시자

```txt
$1
→ 첫 번째 커서 위치

$2
→ 두 번째 커서 위치

$0
→ 마지막 커서 위치
```

예시:

```json
{
    "Describe It": {
        "prefix": "dit",
        "body": [
            "describe('$1', () => {",
            "    it('$2', async () => {",
            "        $0",
            "    });",
            "});"
        ],
        "description": "Jest describe it block"
    }
}
```

---

# 6. ⭐️ 지금은 보류해도 되는 이유

```txt
스니펫은 편하지만,
처음부터 너무 많이 쓰면 코드 흐름을 외우기 어려울 수 있다.

먼저 직접 작성하면서 구조를 익히고,
정말 반복되는 패턴만 나중에 스니펫으로 만드는 것이 좋다.
```

---

# 7. ⭐️ 스니펫으로 만들기 좋은 기준

```txt
3번 이상 반복해서 작성했다
매번 구조가 거의 같다
외워야 하는 코드가 아니라 단순 반복 코드다
복사 붙여넣기하다가 실수할 가능성이 높다
```

---

# 8. ⭐️ 스니펫으로 만들기 애매한 기준

```txt
아직 원리를 이해 중인 코드
테스트마다 구조가 많이 달라지는 코드
실수하면 테스트 의미가 바뀌는 코드
팀 컨벤션이 정해지지 않은 코드
```

---

# 최종 한 줄 요약

VS Code User Snippets는 반복 코드를 빠르게 입력하는 기능이지만, 처음 배우는 단계에서는 직접 작성하며 흐름을 익힌 뒤 정말 자주 반복되는 코드만 등록하는 것이 좋다.