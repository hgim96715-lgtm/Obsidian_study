---
aliases:
  - 커밋 컨벤션
  - 커밋 메시지
  - Conventional Commits
tags:
  - Git
related:
  - "[[00_Git_HomePage]]"
  - "[[Git_Commands]]"
  - "[[Git_Flow]]"
---

# Git_Commit_Convention — 커밋 메시지 컨벤션

## 한 줄 요약

```txt
좋은 커밋 메시지 = 왜 이 변경을 했는지 설명
타입(type): 제목 / 본문 / 꼬리말 구조
```

---

---

# ① 기본 구조

```txt
타입(스코프): 제목

본문 (선택)

꼬리말 (선택)
```

```bash
# 예시
feat(auth): JWT 로그인 기능 추가

기존 세션 방식을 JWT 토큰 방식으로 변경
- 액세스 토큰 만료 시간: 1시간
- 리프레시 토큰 만료 시간: 7일

Closes #123
```

---

---

# ② 타입 종류 ⭐️

|타입|설명|예시|
|---|---|---|
|`feat`|새 기능 추가|`feat: 로그인 기능 추가`|
|`fix`|버그 수정|`fix: 로그인 에러 수정`|
|`docs`|문서 변경|`docs: README 업데이트`|
|`style`|포맷팅 / 세미콜론 등 (로직 변경 없음)|`style: 코드 정렬`|
|`refactor`|코드 리팩토링 (기능 변경 없음)|`refactor: 서비스 레이어 분리`|
|`test`|테스트 추가/수정|`test: 로그인 단위 테스트 추가`|
|`chore`|빌드 / 설정 / 패키지 등|`chore: eslint 설정 추가`|
|`perf`|성능 개선|`perf: 쿼리 최적화`|
|`ci`|CI/CD 설정|`ci: GitHub Actions 추가`|
|`build`|빌드 시스템 변경|`build: webpack 설정 수정`|

---

---

# ③ 좋은 커밋 메시지 규칙

```txt
제목 규칙:
  50자 이내
  첫 글자 대문자 (영어) 또는 한글
  끝에 마침표 없음
  명령형으로 작성

  ✅ feat: 로그인 기능 추가
  ✅ fix: null 포인터 에러 수정
  ❌ feat: 로그인 기능을 추가했습니다.  (과거형 / 마침표)
  ❌ 기능추가  (타입 없음)

본문 규칙:
  제목과 빈 줄로 구분
  무엇을 왜 변경했는지 설명
  어떻게(how)보다 왜(why)에 집중
```

---

---

# ④ 스코프 (scope)

```bash
# 타입(스코프): 제목
# 스코프 = 어떤 모듈/영역에 변경이 있는지

feat(auth): JWT 로그인 추가
fix(movie): 목록 조회 null 에러 수정
docs(api): Swagger 문서 추가
chore(deps): axios 버전 업그레이드
```

---

---

# ⑤ 꼬리말 (footer)

```bash
# 이슈 연결
Closes #123       # 이 커밋으로 이슈 123 닫힘
Fixes #456        # 버그 픽스로 이슈 닫힘
Refs #789         # 참고 이슈 (닫지 않음)

# Breaking Change
BREAKING CHANGE: API 응답 형식 변경
  이전: { data: [...] }
  이후: { result: [...] }
```

---

---

# ⑥ 실전 예시

```bash
# 기능 추가
git commit -m "feat(movie): 영화 검색 기능 추가"

# 버그 수정
git commit -m "fix(auth): 토큰 만료 시 에러 처리 수정"

# 여러 줄 커밋 (본문 포함)
git commit -m "refactor(service): 영화 서비스 레이어 분리

MovieService 에서 비즈니스 로직 분리
- createMovie / updateMovie / deleteMovie 메서드 이동
- Repository 직접 접근 제거"
```

---

---

# ⑦ commitlint 자동 검사

```bash
# 설치 (팀 프로젝트에서 규칙 자동 강제)
npm install --save-dev @commitlint/cli @commitlint/config-conventional husky

# .commitlintrc.json
{
  "extends": ["@commitlint/config-conventional"]
}
```

```txt
commitlint:
  커밋 시 메시지 형식 자동 검사
  규칙 어기면 커밋 거부
  팀 전체 컨벤션 강제 적용
```