---
aliases:
  - Regex
  - JavaScript
  - Pattern
tags:
  - JavaScript
related:
  - "[[00_JS_Ecosystem_HomePage]]"
  - "[[JS_Primitive_Methods]]"
  - "[[JS_Array_Methods]]"
  - "[[TS_Type_Guards]]"
---
# JS_Regex — 정규식

>[!info]
>정규식(Regex) = 문자열에서 특정 패턴을 찾거나 교체하는 표현식.
> `/패턴/플래그` 형태로 작성.
>  `str.replace(/\/$/, '')` = 문자열 끝의 `/`를 제거. 
>  `.test()`로 패턴 존재 여부 확인, `.replace()`로 교체, `.match()`로 추출.

---

# 정규식이란 ⭐️⭐️⭐️⭐️

```txt
정규식 = 문자열 패턴을 표현하는 언어

"이 문자열에 이메일 형식이 있나?"
"URL 끝의 슬래시를 제거해라"
"숫자만 추출해라"

→ 이런 것을 문자열 검색·교체·추출에 사용
```

```typescript
// 정규식 리터럴 (가장 많이 씀)
const regex = /패턴/플래그;

// 동적으로 만들 때 (변수를 패턴에 포함할 때)
const regex = new RegExp('패턴', '플래그');
const regex = new RegExp(variable);  // 변수를 패턴으로
```

```txt
/ / 사이에 패턴을 씀:
  /hello/     → "hello" 라는 문자열
  /\d+/       → 하나 이상의 숫자
  /^\//       → 슬래시로 시작하는 문자열
  /\/$/       → 슬래시로 끝나는 문자열 
```

---

# 특수 문자 — 하나씩 이해하기 ⭐️⭐️⭐️⭐️

## 위치 지정

|패턴|의미|예시|
|---|---|---|
|`^`|문자열 시작|`/^http/` → "http"로 시작|
|`$`|문자열 끝|`/\.ts$/` → ".ts"로 끝|
|`^...$`|전체 문자열 일치|`/^\d+$/` → 전부 숫자|

## 문자 종류

|패턴|의미|
|---|---|
|`\d`|숫자 (0-9)|
|`\D`|숫자가 아닌 것|
|`\w`|영문자·숫자·언더스코어 (a-z, A-Z, 0-9, _)|
|`\W`|그 외|
|`\s`|공백 (스페이스, 탭, 줄바꿈)|
|`\S`|공백이 아닌 것|
|`.`|아무 문자 하나 (줄바꿈 제외)|

## 수량

|패턴|의미|예시|
|---|---|---|
|`*`|0개 이상|`/\d*/` → 숫자 없어도 됨|
|`+`|1개 이상|`/\d+/` → 숫자 하나 이상 필수|
|`?`|0개 또는 1개|`/\d?/` → 숫자 있어도 없어도|
|`{n}`|정확히 n개|`/\d{4}/` → 숫자 4자리|
|`{n,m}`|n~m개|`/\d{2,4}/` → 숫자 2~4자리|

## 이스케이프 — 특수 문자를 문자 그대로 쓸 때

```txt
정규식에서 . * + ? $ ^ { } [ ] ( ) | \ 는 특수 의미가 있음
이 문자들을 "문자 그대로" 쓰려면 앞에 \ 를 붙임

\/   → 슬래시 /  (정규식 시작·끝 기호라서 이스케이프)
\.   → 점 .      (점은 "아무 문자"라서 이스케이프)
\$   → 달러 $
\+   → 플러스 +
\(   → 괄호 (
```

---

#  코드 분석 ⭐️⭐️⭐️⭐️

```typescript
url.replace(/\/$/, "");
```

```txt
/\/$/ 를 분해하면:

  /        정규식 시작
  \/       슬래시 / (이스케이프)
  $        문자열 끝
  /        정규식 끝

  → "문자열 끝에 있는 슬래시"를 찾는 패턴

예시:
  'https://example.com/'  → 'https://example.com'  (끝 / 제거)
  'https://example.com'   → 'https://example.com'  (/ 없으면 그대로)
  '/api/posts/'           → '/api/posts'            (끝 / 제거)

왜 쓰는가:
  URL 끝에 슬래시가 있어도 없어도 동일하게 처리
  API_URL 환경변수가 어떻게 설정돼 있든 일관된 URL 사용
```

---

# 플래그 ⭐️⭐️⭐️

```typescript
/패턴/g   // g = global — 전체에서 모두 찾기 (없으면 첫 번째만)
/패턴/i   // i = ignore case — 대소문자 무시
/패턴/m   // m = multiline — 줄바꿈마다 ^ $ 적용
/패턴/gi  // 여러 플래그 조합 가능
```

```typescript
'aAbBc'.replace(/a/g, 'X')   // 'XAbBc'  → 첫 a만
'aAbBc'.replace(/a/gi, 'X')  // 'XXbBc'  → 모든 a, A

'hello world'.replace(/o/, '_')   // 'hell_ world'   → 첫 o만
'hello world'.replace(/o/g, '_')  // 'hell_ w_rld'   → 전부
```

---

# String 메서드와 함께 ⭐️⭐️⭐️⭐️

## .replace() — 교체

```typescript
// 패턴을 찾아서 교체
'Hello World'.replace(/World/, 'Everyone')
// → 'Hello Everyone'

// 끝의 슬래시 제거
url.replace(/\/$/, '')

// 앞의 슬래시 제거
path.replace(/^\//, '')

// 앞뒤 슬래시 모두 제거
path.replace(/^\/|\/$/g, '')
//                 ↑ | = 또는 (OR)

// 공백 전부 제거
str.replace(/\s/g, '')

// 숫자만 남기기 (숫자 아닌 것 전부 삭제)
'phone: 010-1234-5678'.replace(/\D/g, '')
// → '01012345678'
```

## .test() — 패턴 존재 여부 확인

```typescript
// true / false 반환
/\d+/.test('abc123')   // true  (숫자 있음)
/\d+/.test('abcdef')   // false (숫자 없음)

/^https/.test('https://example.com')   // true
/^https/.test('http://example.com')    // false

// 이메일 형식 확인
/^[^\s@]+@[^\s@]+\.[^\s@]+$/.test('user@example.com')  // true
```

## .match() — 찾아서 추출

```typescript
// 첫 번째 매칭 (g 플래그 없음)
'phone: 010-1234-5678'.match(/\d+/)
// → ['010', index: 7, ...]  배열로 반환

// 전체 매칭 (g 플래그)
'010-1234-5678'.match(/\d+/g)
// → ['010', '1234', '5678']

// 없으면 null
'hello'.match(/\d+/)   // null
```

## .split() — 패턴으로 나누기

```typescript
'a,b,,c'.split(/,+/)   // ['a', 'b', 'c']  → 연속 쉼표도 하나로
'a1b2c3'.split(/\d/)   // ['a', 'b', 'c', '']
```

---

# 자주 쓰는 패턴 ⭐️⭐️⭐️⭐️

```typescript
/\/$/        // 끝의 슬래시
/^\//        // 시작의 슬래시
/^\/|\/$/g   // 앞뒤 슬래시 모두

/\s/g        // 공백 (replace로 제거)
/\s+/g       // 연속 공백을 하나로

/\d+/        // 숫자 하나 이상
/^\d+$/      // 전부 숫자인지

/\.ts$/      // .ts로 끝나는지
/\.(jpg|png|gif)$/i  // 이미지 확장자

// 이메일 (간단 버전)
/^[^\s@]+@[^\s@]+\.[^\s@]+$/

// URL에서 도메인 추출
'https://example.com/path'.match(/^https?:\/\/[^/]+/)
// → ['https://example.com']
```

## 이메일 정규식 분해 ⭐️⭐️⭐️⭐️

```typescript
const EMAIL_RE = /^[^\s@]+@[^\s@]+\.[^\s@]+$/;
```

```txt
분해:
  ^           문자열 시작
  [^\s@]+     공백(\s)도 @도 아닌 문자가 1개 이상  ← 로컬 파트 (hong.gil)
              [^...] = "이것들을 제외한 모든 문자"
  @           @ 문자 그대로
  [^\s@]+     공백도 @도 아닌 문자가 1개 이상       ← 도메인 (example)
  \.          점 . (이스케이프 — 그냥 .은 "아무 문자"라서)
  [^\s@]+     공백도 @도 아닌 문자가 1개 이상       ← TLD (com, kr)
  $           문자열 끝

매칭 예시:
  hong.gil@example.com   ✅  로컬@도메인.TLD
  test@a.b               ✅
  @example.com           ❌  로컬 파트 없음
  hong@                  ❌  도메인 없음
  hong example@test.com  ❌  공백 포함

[^\s@] — 이게 핵심:
  \s = 공백 문자 (스페이스·탭)
  @  = 골뱅이
  ^  = 대괄호 안에서는 "제외"를 의미 (문자열 시작의 ^ 와 다름)
  → 공백이나 @가 아닌 모든 문자 허용
  → 이메일에 공백이나 @ 여러 개가 들어오는 걸 막음
```

```typescript
// 사용
EMAIL_RE.test('user@example.com')   // true
EMAIL_RE.test('invalid email')      // false (공백)
EMAIL_RE.test('user@@example.com')  // false (@ 두 번)
```

---

# 캡처 그룹 — 부분 추출 ⭐️⭐️⭐️

```typescript
// ( ) = 캡처 그룹 — 괄호 안을 따로 추출
const match = 'John 30'.match(/(\w+)\s(\d+)/);
// match[0] = 'John 30'  (전체 매칭)
// match[1] = 'John'     (첫 번째 그룹)
// match[2] = '30'       (두 번째 그룹)

// replace에서 그룹 참조 ($1, $2)
'2024-01-15'.replace(/(\d{4})-(\d{2})-(\d{2})/, '$2/$3/$1')
// → '01/15/2024'  (MM/DD/YYYY 형식으로 변환)
```

---

# 정규식 테스트 방법

```txt
브라우저 개발자 도구 Console에서:
  /\/$/.test('https://example.com/')   → true
  'hello world'.match(/\w+/g)          → ['hello', 'world']

온라인 도구:
  https://regex101.com
  → 패턴 입력하면 실시간으로 어디가 매칭되는지 시각화
  → 각 특수문자의 의미도 설명해줌 (정규식 공부에 매우 유용)
```

---

# 유니코드 범위 — \u 패턴 ⭐️⭐️⭐️⭐️

```txt
\uXXXX = 유니코드 코드 포인트를 16진수로 지정
  \u0041 = 'A' (U+0041)
  \uAC00 = '가' (U+AC00)
  \uD7AF = '힣' (U+D7AF)

[범위] = 그 범위 안의 모든 문자
  [\uAC00-\uD7AF] = 한글 가~힣 (11,172자 한글 음절 전체)
  [\u0020-\u007E] = 기본 ASCII 출력 가능 문자 (스페이스~물결표)
```

## 자주 쓰는 유니코드 범위

```typescript
/[\uAC00-\uD7AF]/   // 한글 음절 (가~힣)
/[\u0020-\u007E]/   // ASCII 출력 가능 (스페이스, 영문, 숫자, 기호)
/[^\u0020-\u007E]/  // ASCII가 아닌 문자 (한글, 일본어, 중국어, 특수문자 등)
/[\u3040-\u309F]/   // 일본어 히라가나
/[\u4E00-\u9FFF]/   // CJK 통합 한자
```

## 번역 필요 여부 판단 — 실전 패턴 ⭐️⭐️⭐️⭐️

```typescript
// "한글이 없고 비ASCII 문자가 있으면 번역 필요"
const needsTitle =
  !/[\uAC00-\uD7AF]/.test(title) &&  // 한글이 없고
  /[^\u0020-\u007E]/.test(title);    // ASCII도 아닌 문자가 있음

const needsDirector =
  !!director &&                            // director가 있고
  !/[\uAC00-\uD7AF]/.test(director) &&    // 한글이 없고
  /[^\u0020-\u007E]/.test(director);      // 비ASCII 문자가 있음
```

```txt
조건 분해:

  !/[\uAC00-\uD7AF]/.test(title):
    한글이 없다 = 한국어 제목이 아님
    /[\uAC00-\uD7AF]/ 에 매칭 안 되면 한글 없음

  /[^\u0020-\u007E]/.test(title):
    ASCII가 아닌 문자가 있다 = 영어도 아님
    일본어, 중국어, 아랍어, 특수 유니코드 등

  둘 다 true:
    한글도 없고 영어도 아닌 문자가 있음
    = 일본어나 중국어 원제를 한국어로 번역해야 함

  예시:
    title = "Sing Street"   → 한글 없음 ✓, ASCII만 있음 → false (번역 불필요)
    title = "シング・ストリート" → 한글 없음 ✓, 비ASCII 있음 ✓ → true (번역 필요)
    title = "싱 스트리트"   → 한글 있음 → false (이미 한국어)

  !!director:
    director가 null/undefined/'' 이면 false → 전체 false
    있을 때만 번역 여부 체크
```

```typescript
// 범용 유틸
function needsKoreanTranslation(text: string): boolean {
  const hasKorean = /[\uAC00-\uD7AF]/.test(text);
  const hasNonAscii = /[^\u0020-\u007E]/.test(text);
  return !hasKorean && hasNonAscii;
}

needsKoreanTranslation('Sing Street')        // false — 영문
needsKoreanTranslation('싱 스트리트')         // false — 이미 한국어
needsKoreanTranslation('シング・ストリート')   // true  — 일본어 → 번역 필요
needsKoreanTranslation('少年의 時間')         // true  — 한자 → 번역 필요
```