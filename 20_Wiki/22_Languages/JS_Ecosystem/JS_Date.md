---
aliases:
  - Date 객체
  - format
  - ISO
  - JavaScript Date
  - JS 날짜
  - parseYmd
  - toLocaleDateString
tags:
  - JavaScript
related:
  - "[[00_JS_Ecosystem_HomePage]]"
  - "[[NestJS_DTO]]"
  - "[[NextJS_Concept]]"
  - "[[React_DatePicker]]"
  - "[[NestJS_StatsBucket]]"
  - "[[NestJS_Throttle]]"
  - "[[Snippet_date-statistics-pattern]]"
  - "[[JS_Intl]]"
---
# JS_Date — Date 객체

> [!info] 
> `Date` 객체는 특정 시점을 UTC 기준 밀리초(타임스탬프)로 저장한다. 
> 날짜를 "어떻게 표시할지"(포맷·타임존·언어)는 → [[JS_Intl]] 참고.

---

# Date 생성 ⭐️⭐️⭐️⭐️

```typescript
new Date()                          // 지금 이 순간
new Date(0)                         // 유닉스 에포크 (1970-01-01 UTC)
new Date(1_700_000_000_000)         // 타임스탬프(ms)로 생성

new Date('2024-01-15')              // ISO 날짜 문자열 (UTC 자정)
new Date('2024-01-15T09:30:00')     // ISO 날짜+시간 (로컬 타임존)
new Date('2024-01-15T09:30:00Z')    // ISO 날짜+시간 (명시적 UTC)

new Date(2024, 0, 15)               // year, month(0=1월), day — 로컬 타임존
new Date(2024, 0, 15, 9, 30, 0)    // year, month, day, hour, min, sec
```

```txt
⚠️ new Date(2024, 0, 15) 에서 month가 0부터 시작
  0 = 1월, 1 = 2월, ..., 11 = 12월

new Date('2024-01-15')
  날짜만 있으면 UTC 자정으로 해석 → 로컬이 UTC+9면 1월 15일 09:00로 표시
  시간까지 있으면 로컬 타임존으로 해석
  → 타임존을 명시하려면 'Z'(UTC) 또는 '+09:00'(KST) 붙이기
```

---

# 타임스탬프 — 날짜를 숫자로 ⭐️⭐️⭐️⭐️

```typescript
Date.now()           // 현재 시각 (ms) — 가장 빠름, new Date() 안 만들어도 됨
new Date().getTime() // 특정 Date 객체의 타임스탬프
+new Date()          // 단항 + 로 타임스탬프 변환 (getTime 축약)

// 날짜 비교 — 타임스탬프로
const a = new Date('2024-01-01');
const b = new Date('2024-06-01');
a < b   // true — Date는 비교 연산자 사용 가능 (타임스탬프로 비교됨)
a.getTime() < b.getTime()  // 명시적 비교 (의도가 더 명확)
```

---

# 날짜 읽기 ⭐️⭐️⭐️

```typescript
const d = new Date('2024-06-15T09:30:00');

d.getFullYear()   // 2024
d.getMonth()      // 5  ← 0부터 시작! 5 = 6월
d.getDate()       // 15 (일)
d.getDay()        // 6  ← 0=일요일, 1=월, ..., 6=토요일
d.getHours()      // 9  (로컬 타임존 기준)
d.getMinutes()    // 30
d.getSeconds()    // 0
d.getMilliseconds() // 0
d.getTime()       // 타임스탬프 (ms)

// UTC 기준 읽기
d.getUTCHours()   // KST 기준이면 9 - 9 = 0
```

```txt
⚠️ getMonth()는 0부터 — 흔한 실수
  d.getMonth() + 1  → 실제 월 (1~12)

⚠️ getHours()는 실행 환경 로컬 타임존 기준
  서버(UTC)에서 실행하면 UTC 시간
  KST 기준으로 구하려면 → [[JS_Intl]] "KST 시간 구하기" 참고
```

---

# 날짜 계산 ⭐️⭐️⭐️⭐️

```typescript
// N일 뒤
function addDays(date: Date, days: number): Date {
  const result = new Date(date);          // 원본 보존
  result.setDate(result.getDate() + days);
  return result;
}

// N시간 뒤 — 타임스탬프로 계산이 더 안전
function addHours(date: Date, hours: number): Date {
  return new Date(date.getTime() + hours * 60 * 60 * 1000);
}

// 두 날짜 사이 일수
function daysBetween(a: Date, b: Date): number {
  const ms = Math.abs(b.getTime() - a.getTime());
  return Math.floor(ms / (1000 * 60 * 60 * 24));
}
```

```txt
왜 원본 Date에 setDate를 직접 안 하는가:
  Date 객체는 mutable — setDate()가 원본을 직접 바꿈
  new Date(date)로 복사본을 만들어서 수정해야 원본이 안 바뀜

타임스탬프 계산이 더 안전한 경우:
  월말 경계 (1월 31일 + 1달), 일광절약시간 같은 엣지케이스를
  setDate로 처리하면 예상 밖 결과가 나올 수 있음
  → 시간·분·초 단위 계산은 타임스탬프(ms) 산술이 명확함
```

## 단위별 ms 상수

```typescript
const MS = {
  SECOND: 1_000,
  MINUTE: 60_000,
  HOUR:   3_600_000,
  DAY:    86_400_000,
  WEEK:   604_800_000,
};

// 7일 뒤
new Date(Date.now() + 7 * MS.DAY);

// 30분 후 만료
const expiresAt = new Date(Date.now() + 30 * MS.MINUTE);
```

---

# 날짜 쓰기 — set* 메서드 ⭐️⭐️

```typescript
const d = new Date();
d.setFullYear(2025);
d.setMonth(11);        // 12월 (0 기반)
d.setDate(31);
d.setHours(0, 0, 0, 0); // 자정으로 (hour, min, sec, ms)
```

```txt
자주 쓰는 패턴:
  setHours(0, 0, 0, 0)  → 그 날의 자정(00:00:00.000)
  setHours(23, 59, 59, 999) → 그 날의 끝 (23:59:59.999)
```

---

# ISO 문자열 변환 ⭐️⭐️⭐️

```typescript
const d = new Date();

d.toISOString()         // '2024-06-15T09:30:00.000Z'  (항상 UTC, Z 포함)
d.toJSON()              // toISOString과 동일
d.toString()            // 로컬 타임존 기준 읽기 어려운 문자열 (디버깅용)

// 날짜만 (YYYY-MM-DD)
d.toISOString().slice(0, 10)  // '2024-06-15'

// DB 저장 / API 전송에는 toISOString() 또는 getTime() 사용
```

---

# 날짜 유효성 확인 ⭐️⭐️

```typescript
// new Date()가 유효한 날짜를 만들었는지 확인
function isValidDate(d: unknown): d is Date {
  return d instanceof Date && !isNaN(d.getTime());
}

isValidDate(new Date('2024-01-15'))     // true
isValidDate(new Date('invalid'))        // false — NaN
isValidDate(new Date(undefined as any)) // false — Invalid Date
```

---

# 자주 쓰는 패턴 모음 ⭐️⭐️⭐️

```typescript
// 오늘 자정 (로컬 기준)
const startOfToday = new Date();
startOfToday.setHours(0, 0, 0, 0);

// 오늘 끝 (로컬 기준)
const endOfToday = new Date();
endOfToday.setHours(23, 59, 59, 999);

// 이번 주 월요일
const monday = new Date();
monday.setDate(monday.getDate() - ((monday.getDay() + 6) % 7));
monday.setHours(0, 0, 0, 0);

// N분 전인지 확인
function isWithinMinutes(date: Date, minutes: number): boolean {
  return Date.now() - date.getTime() < minutes * MS.MINUTE;
}

// 오래된 순으로 정렬
dates.sort((a, b) => a.getTime() - b.getTime());

// 최신순으로 정렬
dates.sort((a, b) => b.getTime() - a.getTime());
```

---

# 한눈에

```txt
생성:
  new Date()                현재
  new Date(timestamp)       타임스탬프(ms)
  new Date('YYYY-MM-DD')    ISO 문자열 (UTC 자정)
  new Date(y, m, d)         연,월(0=1월),일 (로컬 타임존)

읽기:
  getFullYear() / getMonth()+1 / getDate() / getDay()
  getHours() / getMinutes() / getSeconds()
  ⚠️ getMonth() = 0부터 / getHours() = 로컬 타임존 기준

타임스탬프:
  Date.now()         현재 (ms)
  d.getTime()        특정 Date (ms)
  a < b              비교 가능 (타임스탬프로 변환됨)

계산:
  new Date(d.getTime() + N * MS.DAY)   날짜 연산은 타임스탬프가 안전
  new Date(date)로 복사 후 set* 사용   원본 보존

출력:
  d.toISOString()    'YYYY-MM-DDTHH:mm:ss.sssZ' (UTC, DB/API 저장에)
  .slice(0, 10)      'YYYY-MM-DD'

사람이 읽을 수 있는 포맷 (타임존 변환 포함) → [[JS_Intl]]
```