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
# JS_Regex — 정규식 (Regular Expression)

> [!info] 
> 정규식 = 문자열 패턴을 표현하는 언어. "이 문자열이 이 형식에 맞는가?" 검증,
>  "이 패턴에 맞는 부분을 꺼내라" 추출에 사용한다.

---

# 기본 형태 ⭐️⭐️⭐️

```typescript
/패턴/플래그

/^\d+$/        // 숫자로만 이루어진 문자열
/^(\d+):(\d+)$/ // mm:ss 형식
/[a-z]+/gi    // 영문자 (대소문자 무시, 전체)
```

## 주요 패턴 기호

|기호|의미|예시|
|---|---|---|
|`^`|문자열 시작|`/^hello/` → 'hello'로 시작|
|`$`|문자열 끝|`/world$/` → 'world'로 끝|
|`\d`|숫자 [0-9]|`/\d+/` → 숫자 1개 이상|
|`\w`|단어 문자 [a-zA-Z0-9_]|`/\w+/`|
|`\s`|공백 (스페이스/탭/줄바꿈)|`/\s+/`|
|`.`|아무 문자 1개 (줄바꿈 제외)|`/a.c/` → 'abc', 'axc'|
|`+`|1개 이상|`/\d+/`|
|`*`|0개 이상|`/\d*/`|
|`?`|0개 또는 1개 (선택적)|`/\d?/`, `/[0-5]?\d/`|
|`{n}`|정확히 n개|`/\d{4}/` → 숫자 4개|
|`{n,m}`|n~m개|`/\d{2,4}/`|
|`[abc]`|a, b, c 중 하나|`/[aeiou]/`|
|`[^abc]`|a, b, c 를 제외한 것|`/[^0-9]/`|
|`(...)`|그룹 (캡처)|`/(\d+):(\d+)/`|
|`(?:...)`|그룹 (캡처 안 함)|`/(?:\d+):(\d+)/`|
|`|`|또는|

## 플래그

|플래그|의미|
|---|---|
|`g`|전체 매칭 (첫 번째만 아니라 전부)|
|`i`|대소문자 무시|
|`m`|멀티라인 (^/$가 각 줄에 적용)|

---

# test() — 패턴 일치 여부 (boolean) ⭐️⭐️⭐️⭐️

```typescript
/^\d+$/.test('123')      // true  — 숫자로만 이루어짐
/^\d+$/.test('12a3')     // false — 숫자가 아닌 문자 포함
/^\d+$/.test('')         // false — 빈 문자열

// 이메일 형식 간단 체크
/^[^\s@]+@[^\s@]+\.[^\s@]+$/.test('user@example.com')  // true
```

```txt
regex.test(str):
  str이 regex 패턴에 맞으면 true, 아니면 false
  "이 문자열이 형식에 맞는가?" 검증에 사용

^ 와 $ 를 같이 쓰는 이유:
  ^  없으면: 'abc123'도 \d+ 에 매칭됨 (123 부분이 있으니)
  $  없으면: '123abc'도 ^\d+ 에 매칭됨 (앞이 숫자니)
  ^$ 모두 있어야 "전체가 이 패턴" 을 보장
```

---

# match() — 캡처 그룹으로 추출 ⭐️⭐️⭐️⭐️

```typescript
const result = '02:30'.match(/^(\d+):([0-5]?\d)$/);
// result: ['02:30', '02', '30', index: 0, ...]
//          전체매칭  m[1]  m[2]

if (!result) {
  // 패턴에 안 맞음 → null 반환
}

result[0]  // '02:30'  — 전체 매칭
result[1]  // '02'     — 첫 번째 그룹 ()
result[2]  // '30'     — 두 번째 그룹 ()
```

```typescript
// 명명 그룹으로 더 읽기 쉽게 (ES2018+)
const m = '02:30'.match(/^(?<mm>\d+):(?<ss>[0-5]?\d)$/);
m?.groups?.mm  // '02'
m?.groups?.ss  // '30'
```

```txt
match() 반환값:
  패턴 매칭 성공 → [전체매칭, 그룹1, 그룹2, ...] (배열)
  패턴 매칭 실패 → null

  → 항상 null 체크 후 사용
  if (!m) return NaN;  // 매칭 실패 처리
  Number(m[1]) * 60 + Number(m[2])  // 그룹값 사용
```

---

# 실전 — 시간 입력 파싱 (초 또는 mm:ss) ⭐️⭐️⭐️⭐️

```typescript
/**
 * 시간 문자열 → 초(number)로 변환
 * 입력: '90' → 90초 / '1:30' → 90초
 * 반환: number(성공) | undefined(빈 입력) | NaN(잘못된 형식)
 */
function parseTimeInput(raw: string): number | undefined {
  const t = raw.trim();
  if (!t) return undefined;              // 빈 입력 → undefined (에러 아님)
  if (/^\d+$/.test(t)) return Number(t); // 순수 숫자 → 그대로 초로
  const m = t.match(/^(\d+):([0-5]?\d)$/);
  if (!m) return NaN;                    // 형식 안 맞음 → NaN (에러)
  return Number(m[1]) * 60 + Number(m[2]);
}

// 사용 후 검증
const start = parseTimeInput(startInput);
const end   = parseTimeInput(endInput);

if (Number.isNaN(start) || Number.isNaN(end)) {
  setError('구간은 초 또는 mm:ss 로 적어 주세요.');
  return;
}
```

```txt
반환값 설계: number | undefined | NaN
  undefined  빈 문자열 → "입력 안 함" (오류 메시지 불필요)
  NaN        형식 오류 → "잘못된 입력" (오류 메시지 필요)
  number     정상 → 이후 로직 진행

/^(\d+):([0-5]?\d)$/ 패턴 분해:
  ^          시작
  (\d+)      분(mm) — 숫자 1개 이상, 그룹1로 캡처
  :          콜론 구분자
  ([0-5]?\d) 초(ss) — 0-5 선택적 + 숫자 한 개 (00~59), 그룹2로 캡처
  $          끝

  → '1:30' ✅  '90:00' ✅  '1:60' ❌ (60초 이상)  '1:3x' ❌
```

## 범용성 ⭐️⭐️⭐️

```typescript
// 같은 패턴으로 활용 가능한 곳:
// 비디오/음악 플레이어 구간 설정
// 유튜브 타임스탬프 입력
// 스케줄 시간 입력 (h:mm)
// 운동 기록 (mm:ss)

// h:mm:ss로 확장
function parseHhMmSs(raw: string): number | undefined {
  const t = raw.trim();
  if (!t) return undefined;
  if (/^\d+$/.test(t)) return Number(t);
  const m2 = t.match(/^(\d+):([0-5]?\d)$/);
  if (m2) return Number(m2[1]) * 60 + Number(m2[2]);
  const m3 = t.match(/^(\d+):([0-5]?\d):([0-5]?\d)$/);
  if (!m3) return NaN;
  return Number(m3[1]) * 3600 + Number(m3[2]) * 60 + Number(m3[3]);
}
```

---

# 자주 쓰는 패턴 모음 ⭐️⭐️⭐️

```typescript
// 숫자만
/^\d+$/.test(str)

// 소수점 숫자
/^\d+(\.\d+)?$/.test(str)

// mm:ss 시간
/^(\d+):([0-5]?\d)$/.test(str)

// YYYY-MM-DD 날짜
/^\d{4}-\d{2}-\d{2}$/.test(str)

// 이메일 (간단)
/^[^\s@]+@[^\s@]+\.[^\s@]+$/.test(str)

// URL (간단)
/^https?:\/\/.+/.test(str)

// 한글 포함 여부
/[\uAC00-\uD7A3]/.test(str)

// 앞뒤 공백 제거 후 비어있는지
!/\S/.test(str)  // 공백만 있으면 true (비어있음)

// # 접두사 제거
str.replace(/^#/, '')  // '#react' → 'react'
str.replace(/^#+/, '') // '##제목' → '제목'
```

---

# replace — 패턴 치환 ⭐️⭐️⭐️

```typescript
// 단순 치환
'hello world'.replace('world', 'JS')   // 'hello JS' (첫 번째만)
'aaa'.replace(/a/g, 'b')               // 'bbb' (전체, g 플래그 필요)

// # 접두사 제거
'#react'.replace(/^#/, '')             // 'react'

// 공백 제거
'  hello  '.replace(/\s+/g, '')        // 'hello'
'  hello  '.trim()                     // 'hello' (앞뒤만)

// 캡처 그룹 참조 ($1, $2)
'2026-06-18'.replace(/^(\d{4})-(\d{2})-(\d{2})$/, '$3/$2/$1')
// → '18/06/2026'
```

---

# 한눈에

```txt
리터럴:  /패턴/플래그
플래그:  g(전체) i(대소문자무시) m(멀티라인)

검증:
  regex.test(str) → true/false
  ^$ 를 같이 써야 "전체가 이 패턴"

추출:
  str.match(regex) → [전체, 그룹1, 그룹2, ...] 또는 null
  null 체크 필수

치환:
  str.replace(regex, 대체문자) — g 플래그 없으면 첫 번째만

시간 파싱 패턴 (초 or mm:ss):
  /^\d+$/.test(t)        → 순수 숫자
  t.match(/^(\d+):([0-5]?\d)$/) → mm:ss → [1]*60 + [2]
  반환: number | undefined | NaN (undefined=빈값, NaN=잘못된형식)

주요 패턴:
  \d  숫자  \w  단어문자  \s  공백
  +   1+    *   0+    ?   0~1   {n}  n개
  ^   시작  $   끝    ()  캡처그룹
  [a-z]  범위  [^a-z]  제외
```