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

# 타임존 기준 날짜 처리 — KST 예시 ⭐️⭐️⭐️⭐️

```txt
문제:
  Date 객체는 UTC 기준 — getHours()는 서버/로컬 시스템 시각
  배포 서버(UTC)에서 getHours() → 한국 시각과 9시간 차이

해결:
  Intl.DateTimeFormat에 timeZone 지정 → 해당 타임존의 시각을 문자열로 추출
  → 타임존에 관계없이 어떤 환경에서도 KST 기준 날짜 계산 가능
```

## KST 유틸 함수

```typescript
const KST = 'Asia/Seoul';

/** KST 기준 날짜 키 YYYY-MM-DD */
export function toKstDateKey(date: Date): string {
  return new Intl.DateTimeFormat('en-CA', {  // en-CA → YYYY-MM-DD 포맷
    timeZone: KST,
    year:     'numeric',
    month:    '2-digit',
    day:      '2-digit',
  }).format(date);
  // 예: new Date('2024-06-15T03:00:00Z') → '2024-06-15' (KST 12:00)
}

/** KST 그날 00:00:00.000 — DB timestamptz 비교용 */
export function startOfKstDay(reference = new Date()): Date {
  const key = toKstDateKey(reference);        // 'YYYY-MM-DD'
  return new Date(`${key}T00:00:00+09:00`);  // KST 자정
}

/** KST 시각 (0-23) */
export function getKstHour(date: Date): number {
  return Number(
    new Intl.DateTimeFormat('en-US', {
      timeZone: KST,
      hour:     'numeric',
      hour12:   false,
    }).format(date),
  );
}

/** KST 기준 월 키 YYYY-MM */
export function getKstMonthKey(date: Date): string {
  return toKstDateKey(date).slice(0, 7);
}
```

```txt
en-CA 로케일을 쓰는 이유:
  Intl.DateTimeFormat 출력 포맷은 로케일마다 다름
  ko-KR → '2024. 6. 15.' (한국 표기)
  en-US → '6/15/2024'
  en-CA → '2024-06-15' ← ISO 8601 순서 (YYYY-MM-DD), 파싱하기 쉬움
  → 날짜 문자열을 DB 키나 비교에 쓸 때 en-CA가 편함

`${key}T00:00:00+09:00`:
  key = 'YYYY-MM-DD'
  T00:00:00   → 자정 (시:분:초)
  +09:00      → KST 오프셋 명시
  → new Date()가 정확히 KST 그날 자정을 UTC로 변환해서 생성

왜 시스템 시각을 안 쓰는가:
  date.setHours(0, 0, 0, 0)        → 서버 시스템 시각 기준 자정
  startOfKstDay()                   → 항상 KST 기준 자정
  UTC 서버에서 실행해도 한국 날짜 경계가 정확함

Intl.DateTimeFormat 상세 → [[JS_Intl]]
```

## 실전 사용

```typescript
const now = new Date();

// DB 쿼리: 오늘(KST) 이후 데이터
const startOfToday = startOfKstDay(now);
where: { createdAt: { gte: startOfToday } }

// 이번 주(오늘 포함 7일) 시작
const startOfWeek = new Date(startOfToday.getTime() - 6 * 86_400_000);
where: { createdAt: { gte: startOfWeek } }

// 시간대별 통계 키
const hour = getKstHour(new Date());  // 현재 KST 시각 (0-23)
const dateKey = toKstDateKey(new Date());  // '2024-06-15'
const monthKey = getKstMonthKey(new Date()); // '2024-06'
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

## 단위별 ms 상수 ⭐️⭐️⭐️⭐️

```typescript
// 어떻게 계산하는가
1_000           // 1초    = 1,000ms
60_000          // 1분    = 60 * 1,000
3_600_000       // 1시간  = 60 * 60 * 1,000
86_400_000      // 1일    = 24 * 60 * 60 * 1,000
604_800_000     // 1주    = 7 * 24 * 60 * 60 * 1,000

// 상수로 정의해서 사용
const MS_PER_SECOND = 1_000;
const MS_PER_MINUTE = 60 * 1_000;
const MS_PER_HOUR   = 60 * 60 * 1_000;
const MS_PER_DAY    = 24 * 60 * 60 * 1_000;   // = 86_400_000
const MS_PER_WEEK   = 7 * MS_PER_DAY;          // = 604_800_000

// 또는 객체로 묶기
const MS = {
  SECOND: 1_000,
  MINUTE: 60_000,
  HOUR:   3_600_000,
  DAY:    86_400_000,
  WEEK:   604_800_000,
};
```

```txt
_ (숫자 구분자):
  86400000 보다 86_400_000 이 읽기 쉬움
  JavaScript/TypeScript 문법 — 실행에는 영향 없음
  천 단위 또는 의미 있는 단위로 끊어서 표기

86_400_000 검산:
  하루 = 24시간 × 60분 × 60초 × 1,000ms
  24 × 60 = 1,440분
  1,440 × 60 = 86,400초
  86,400 × 1,000 = 86,400,000ms = 86_400_000
```

## 실전 계산 패턴 ⭐️⭐️⭐️⭐️

```typescript
const now = Date.now();  // 현재 타임스탬프 (ms)

// N일 전
const startOfWeek = new Date(startOfToday.getTime() - 6 * MS_PER_DAY);
// 오늘 포함 7일 = 6일 전 자정부터

// N일 뒤 (만료 시각)
const expiresAt = new Date(now + 7 * MS_PER_DAY);    // 7일 후
const tokenExp  = new Date(now + 15 * MS_PER_MINUTE); // 15분 후 토큰 만료

// 두 날짜 사이 일수
const diffMs   = Math.abs(b.getTime() - a.getTime());
const diffDays = Math.floor(diffMs / MS_PER_DAY);

// 1시간 이내인지 확인
const isRecent = (Date.now() - createdAt.getTime()) < MS_PER_HOUR;

// 30분 내 중복 요청 방지 (스로틀링)
const lastSentAt = new Date(stored);
if (Date.now() - lastSentAt.getTime() < 30 * MS_PER_MINUTE) {
  throw new Error('잠시 후 다시 시도해주세요.');
}
```

```txt
타임스탬프(ms) 산술이 안전한 이유:
  addDays처럼 setDate() 쓰면 월말, 일광절약시간(DST) 경계에서 엣지케이스 발생
  ms 단위 덧셈은 항상 정확한 시간 차이를 보장

  6 * 86_400_000:
  6일을 ms로 표현 = 6 * 하루ms
  → 타임스탬프에서 빼면 "6일 전 그 시각"의 타임스탬프

  startOfToday.getTime() - 6 * 86_400_000:
  오늘 자정 타임스탬프 - 6일ms = 6일 전 자정 타임스탬프
  → "오늘 포함 7일치 데이터 조회" 시작점
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