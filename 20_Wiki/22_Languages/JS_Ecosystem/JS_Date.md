---
aliases: [날짜 범위 순회, Date 객체, format, ISO, toISOString()]
tags: [JavaScript]
related:
  - "[[00_JS_Ecosystem_HomePage]]"
  - "[[JS_Intl]]"
  - "[[NestJS_Prisma_Patterns]]"
  - "[[PG_Types]]"
  - "[[React_DatePicker]]"
  - "[[Snippet_date-statistics-pattern]]"
---
# JS_Date — Date 객체

> [!info]
>  Date 객체의 내부는 "1970년 1월 1일 0시(UTC)부터 지금까지의 밀리초" 숫자 하나다. 
>  덕분에 날짜 계산이 숫자 계산으로 단순해진다. 
>  서버 전송엔 `toISOString()`, input 연동엔 `.slice(0,10)`. 
>  화면 포맷 → [[JS_Intl]], DatePicker → [[React_DatePicker]]

---

# Date 객체란 — 내부는 숫자 하나 ⭐️⭐️⭐️⭐️

```txt
Date 객체가 저장하는 것:
  1970년 1월 1일 00:00:00 UTC 부터 지금까지의 밀리초(ms)
  = Unix timestamp (ms 단위)

  2024-01-15 → 내부적으로는 1705276800000 같은 숫자 하나

이 숫자 하나로 모든 것이 가능:
  두 날짜의 차이 = 두 숫자의 빼기
  n일 후 = 숫자 + (n × 24 × 60 × 60 × 1000)
  크기 비교 = 숫자 비교
```

```typescript
const now = new Date();
now.getTime()  // 밀리초 숫자 (예: 1705276800000)
Date.now()     // 현재 밀리초 — new Date().getTime() 와 동일, 더 짧음
```

---

# Date 만들기 ⭐️⭐️⭐️

```typescript
// 현재 시각
new Date()
Date.now()              // 밀리초 숫자만 필요할 때 (더 가벼움)

// 특정 날짜 — ISO 8601 형식 (권장)
new Date('2024-01-15')                    // 날짜만 (자정 UTC 기준)
new Date('2024-01-15T09:00:00')           // 날짜 + 시간 (로컬 시간 기준)
new Date('2024-01-15T00:00:00.000Z')      // UTC 명시

// API 응답 string → Date
new Date(createdAt)   // "2024-01-15T09:00:00.000Z" 같은 ISO 문자열
new Date(timestamp)   // 밀리초 숫자
```

```txt
new Date('2024-01-15') vs new Date('2024-01-15T00:00:00'):
  날짜만 있으면 UTC 자정으로 해석 → 한국(UTC+9)에서는 오전 9시로 표시됨
  시간까지 있으면 로컬 시간으로 해석

  API에서 오는 날짜 문자열은 보통 ISO 8601 형식 ("2024-01-15T09:00:00.000Z")
  → new Date(isoString)으로 안전하게 파싱
```

---

# 값 읽기 ⭐️⭐️⭐️

```typescript
const d = new Date('2024-01-15T09:30:00');

d.getFullYear()     // 2024
d.getMonth()        // 0  ← ⚠️ 0부터 시작 (0=1월, 11=12월)
d.getDate()         // 15  (날짜)
d.getDay()          // 1   (요일: 0=일요일, 1=월요일 ... 6=토요일)
d.getHours()        // 9
d.getMinutes()      // 30
d.getSeconds()      // 0
d.getTime()         // 밀리초 숫자
```

```txt
⚠️ getMonth()가 0부터 시작하는 것은 유명한 함정
  1월 = 0, 2월 = 1 ... 12월 = 11
  화면에 표시할 때: d.getMonth() + 1
```

---

# 날짜 계산 ⭐️⭐️⭐️⭐️

## 두 날짜의 차이

```typescript
const start = new Date('2024-01-01');
const end   = new Date('2024-01-15');

const diffMs   = end.getTime() - start.getTime();  // 밀리초 차이
const diffDays = diffMs / (1000 * 60 * 60 * 24);  // 일 차이 = 14

// 소수점이 생기는 경우 (시간이 포함된 날짜 비교)
Math.floor(diffMs / (1000 * 60 * 60 * 24))   // 내림 (완전한 날수)
Math.round(diffMs / (1000 * 60 * 60 * 24))   // 반올림
```

```txt
날짜 계산 = 숫자 빼기:
  getTime()으로 내부 숫자를 꺼내서 빼면 됨
  1000 * 60 * 60 * 24 = 하루를 밀리초로 표현
  (1000ms = 1초, 60초 = 1분, 60분 = 1시간, 24시간 = 1일)
```

## 밀리초 상수 — 자주 쓰는 값 ⭐️⭐️⭐️

```typescript
// 매번 계산하면 헷갈리므로 상수로 정의
const MS = {
  SECOND: 1_000,               // 1,000
  MINUTE: 60 * 1_000,          // 60,000
  HOUR:   60 * 60 * 1_000,     // 3,600,000
  DAY:    24 * 60 * 60 * 1_000, // 86,400,000  ← 86400000
  WEEK:    7 * 24 * 60 * 60 * 1_000, // 604,800,000
} as const;

// 사용
const in7days  = new Date(Date.now() + 7 * MS.DAY);
const in1hour  = new Date(Date.now() + MS.HOUR);
const weekAgo  = new Date(Date.now() - MS.WEEK);
```

```txt
86400000 = 24 * 60 * 60 * 1000 = MS.DAY
  24시간 × 60분 × 60초 × 1000밀리초 = 하루

7 * 86400000 = MS.WEEK
  하루 × 7 = 1주

왜 숫자를 직접 쓰지 않는가:
  86400000 → 한 눈에 "하루"라고 인식 어려움
  7 * MS.DAY → "7일" 의도가 명확
  코드 리뷰, 디버깅에서 혼동 방지
```
## n일 후/전 계산

```typescript
const now = new Date();

// n일 후
const after7days = new Date(now.getTime() + 7 * 24 * 60 * 60 * 1000);

// n일 전
const before30days = new Date(now.getTime() - 30 * 24 * 60 * 60 * 1000);

// setDate를 쓰는 방법 (원본 변경 주의)
const d = new Date();
d.setDate(d.getDate() + 7);   // 원본이 바뀜
```

```txt
setDate(d.getDate() + 7) 의 장점:
  월 경계를 알아서 처리 (1월 28일 + 7 = 2월 4일 자동)
  밀리초 계산은 DST(서머타임) 같은 예외에서 1일이 23시간 또는 25시간이 될 수 있음

안전한 방법:
  날짜 계산은 date-fns 라이브러리를 쓰면 예외 케이스를 다 처리해줌
  pnpm add date-fns
  [[React_DatePicker]] 참조
```

## 월 시작/끝

```typescript
const now = new Date();

// 이번 달 1일
const startOfMonth = new Date(now.getFullYear(), now.getMonth(), 1);

// 이번 달 마지막 날 — 다음달 0일 = 이번달 마지막
const endOfMonth = new Date(now.getFullYear(), now.getMonth() + 1, 0);
endOfMonth.getDate()  // 28, 29, 30, 31 중 하나
```

---

# 비교 ⭐️⭐️⭐️

```typescript
const a = new Date('2024-01-01');
const b = new Date('2024-06-15');

// 대소 비교 — 숫자처럼 비교 가능
a < b   // true  (a가 더 과거)
a > b   // false
a <= b  // true

// 같은지 확인 — == 안 됨, getTime() 비교
a == b           // false (객체 참조 비교)
a.getTime() === b.getTime()  // true/false (값 비교) ✅

// 오늘 이후인지
new Date() < deadline  // true = 아직 기간 남음

// 정렬
dates.sort((a, b) => a.getTime() - b.getTime());  // 오름차순 (과거 → 최신)
dates.sort((a, b) => b.getTime() - a.getTime());  // 내림차순 (최신 → 과거)
```

---

# 날짜 → 문자열 변환 ⭐️⭐️⭐️⭐️

```typescript
const d = new Date('2024-01-15T09:30:00.000Z');

d.toISOString()         // "2024-01-15T09:30:00.000Z"  ← UTC 기준 ISO 8601
d.toJSON()              // "2024-01-15T09:30:00.000Z"  ← toISOString()과 동일

d.toISOString().slice(0, 10)   // "2024-01-15"  ← 날짜만 (서버 전송·input type="date"에 사용)
d.toISOString().slice(11, 16)  // "09:30"       ← 시간만

d.toLocaleDateString('ko-KR')  // "2024. 1. 15."  ← 로케일 기반 날짜
d.toLocaleTimeString('ko-KR')  // "오전 6:30:00"  ← 로케일 기반 시간 (로컬 시간 기준)
d.toLocaleString('ko-KR')      // "2024. 1. 15. 오전 6:30:00"  ← 둘 다

d.toDateString()        // "Mon Jan 15 2024"  ← 영문 고정 (로케일 무시)
d.toString()            // "Mon Jan 15 2024 18:30:00 GMT+0900 (KST)"
d.toUTCString()         // "Mon, 15 Jan 2024 09:30:00 GMT"
```

```txt
어떤 메서드를 언제 쓰는가:

  toISOString()
    서버에 날짜 데이터를 전송할 때 (항상 UTC, 일관된 형식)
    DB에 저장할 날짜 문자열 만들 때
    → "2024-01-15T09:30:00.000Z"

  toISOString().slice(0, 10)
    input type="date"의 value에 넣을 때 (YYYY-MM-DD 형식 필요)
    날짜만 필요한 API 파라미터 만들 때
    → "2024-01-15"

  toLocaleString() / toLocaleDateString()
    화면에 표시할 때 — 로케일에 맞는 형식
    옵션 지정이 필요하면 Intl.DateTimeFormat이 더 정밀 → [[JS_Intl]]
    → "2024. 1. 15."

  toDateString() / toString()
    디버깅용 — 로케일 무시하고 영문 고정
    실제 UI에 쓰면 안 됨
```

```txt
⚠️ toISOString()은 항상 UTC 기준:
  서울(UTC+9) 오전 6시 30분인 날짜 d
  d.toISOString() → "2024-01-15T09:30:00.000Z"  (UTC 기준으로 변환됨)

  한국 시간 그대로 보내고 싶으면:
  → Intl.DateTimeFormat으로 포맷하거나
  → toISOString() 대신 수동으로 로컬 시간 조합
```

---

# 타임존 주의사항 ⭐️⭐️⭐️

```txt
Date 객체 내부는 항상 UTC
getHours()는 로컬 시간 기준으로 반환

한국(UTC+9)에서:
  new Date('2024-01-15T00:00:00Z').getHours()  // 9 (UTC 자정 = 한국 오전 9시)

서버(UTC)에서 같은 코드:
  new Date('2024-01-15T00:00:00Z').getHours()  // 0

→ 서버와 클라이언트가 다른 결과 → 버그 원인

안전한 방법:
  날짜 표시는 getHours() 같은 로컬 메서드 대신 Intl로
  계산은 getTime()(UTC 기반 ms)으로
  타임존 포함 저장/전송은 ISO 8601 형식 ("2024-01-15T00:00:00.000Z")
```

---

# 자주 쓰는 유틸 함수

```typescript
// 오늘 자정 (00:00:00)
function startOfToday(): Date {
  const d = new Date();
  d.setHours(0, 0, 0, 0);
  return d;
}

// 두 날짜가 같은 날인지 (시간 무시)
function isSameDay(a: Date, b: Date): boolean {
  return (
    a.getFullYear() === b.getFullYear() &&
    a.getMonth()    === b.getMonth() &&
    a.getDate()     === b.getDate()
  );
}

// N일 전 날짜
function daysAgo(n: number): Date {
  return new Date(Date.now() - n * 24 * 60 * 60 * 1000);
}
```

---
# 날짜 범위 순회 — while + getTime() ⭐️⭐️⭐️

```typescript
// from 날짜부터 to 날짜까지 하루씩 순회
function eachDay(from: Date, to: Date, callback: (day: Date) => void) {
  let t = from.getTime();         // 시작 날짜 → 밀리초
  const end = to.getTime();       // 끝 날짜 → 밀리초

  while (t <= end) {
    callback(new Date(t));        // ms → Date 객체로 변환해서 처리
    t += 86400000;                // 하루(DAY_MS) 씩 전진
  }
}

// 날짜 범위를 키 배열로 생성 — 차트 x축, 버킷 초기화 등
function dayKeyRange(from: string, to: string): string[] {
  const keys: string[] = [];
  let t = new Date(`${from}T00:00:00+09:00`).getTime(); // KST 자정
  const end = new Date(`${to}T00:00:00+09:00`).getTime();

  while (t <= end) {
    const d = new Date(t);
    const key = `${d.getUTCFullYear()}-${String(d.getUTCMonth() + 1).padStart(2, '0')}-${String(d.getUTCDate()).padStart(2, '0')}`;
    keys.push(key);
    t += 86400000;
  }
  return keys;
}
// dayKeyRange('2026-08-12', '2026-08-18')
// → ['2026-08-12', '2026-08-13', ..., '2026-08-18']
```


```txt
왜 Date 객체 대신 getTime() 밀리초로 반복하는가:

  Date 객체를 직접 조작:
    d.setDate(d.getDate() + 1) → 서버 TZ에 따라 결과 다름
    UTC 서버에서 "8월 18일 + 1일" 계산이 TZ 의존적

  getTime() 밀리초 산술:
    숫자 + 숫자 → TZ 완전 무관
    t += 86400000 → 항상 정확히 24시간 전진
    KST처럼 DST 없는 TZ에서 안전

왜 for 루프 대신 while인가:
  for (let i = 0; i < dayCount; i++) → 일수를 미리 계산해야 함
  while (t <= end)                   → 끝 조건이 직관적
  날짜 범위 순회는 while이 의도를 더 잘 표현

실전 용도:
  chart 라이브러리 x축 라벨 생성 (날짜 문자열 배열)
  통계 버킷 초기화 — 빈 날짜도 0으로 채우기
  → Snippet_date-statistics-pattern 참고
```
---

# 월 범위 계산 — 캘린더 쿼리 패턴 ⭐️⭐️⭐️⭐️

```txt
캘린더 API에서 "해당 월 전체" 조회가 필요할 때
  → 월의 첫 날 ~ 다음 달 첫 날 범위로 gte/lt 쿼리

핵심:
  padStart(2, '0')  — 월/일을 항상 2자리로 포맷 ('1' → '01')
  Date.UTC(y, m, 1) — m이 12일 때 자동으로 다음 해 1월로 넘어감
                       TZ 무관하게 UTC 기준으로 안전하게 계산
```

## padStart — 월/일 2자리 포맷

```typescript
String(month).padStart(2, '0')   // 1 → '01', 12 → '12'
String(day).padStart(2, '0')     // 3 → '03', 15 → '15'

// 날짜 키 만들기
const monthKey = `${year}-${String(month).padStart(2, '0')}-01`;
// year=2024, month=1  →  '2024-01-01'
// year=2024, month=12 →  '2024-12-01'
```

## Date.UTC — 다음 달 첫 날 계산

```typescript
// month가 1~12 기준이라면 month 자체가 "다음 달의 인덱스(0-based)"가 됨
const nextMonth = new Date(Date.UTC(year, month, 1));
// year=2024, month=12 → Date.UTC(2024, 12, 1) → 2025-01-01T00:00:00.000Z
//                        month 12 → JS Date의 0-based 인덱스에서 자동으로 연도 올림

const nextMonthKey = `${nextMonth.getUTCFullYear()}-${String(
  nextMonth.getUTCMonth() + 1,  // getUTCMonth()는 0-based → +1 필요
).padStart(2, '0')}-01`;
```

```txt
Date.UTC(year, month, 1) 에서 month를 1~12로 받아서 그대로 넣으면:
  month=1  (1월) → Date.UTC(year, 1, 1)  → 2월 1일  (0-based 인덱스 1 = 2월)
  → 이 코드는 외부에서 month를 1~12로 받되 Date.UTC에 그대로 넘겨
    "month번째 달의 다음 달" 을 계산하는 트릭

  year=2024, month=12 → Date.UTC(2024, 12, 1)
    → JS가 자동으로 2025년 1월로 처리 (연도 자동 올림)
    → 수동으로 if (month === 12) year++ 불필요

getUTCFullYear() / getUTCMonth() 사용 이유:
  서버 TZ에 따라 getFullYear()/getMonth() 결과가 달라질 수 있음
  UTC 계열 메서드로 TZ 무관하게 안전하게 추출
```

## 전체 패턴 — 월 범위 start/end 추출

```typescript
function getMonthRange(year: number, month: number) {
  // month: 1~12
  const monthKey = `${year}-${String(month).padStart(2, '0')}-01`;

  const nextMonthDate = new Date(Date.UTC(year, month, 1)); // month 그대로 → 다음달
  const nextMonthKey = `${nextMonthDate.getUTCFullYear()}-${String(
    nextMonthDate.getUTCMonth() + 1,
  ).padStart(2, '0')}-01`;

  const start = kstDayRange(monthKey).start;    // 해당 월 1일 KST 00:00 → UTC
  const end   = kstDayRange(nextMonthKey).start; // 다음 월 1일 KST 00:00 → UTC

  return { start, end };
  // Prisma: watchedAt: { gte: start, lt: end }
  // → 해당 월 전체 KST 날짜 커버
}
```

| 함수 | 역할 |
|---|---|
| `padStart(2, '0')` | 1자리 월/일을 2자리로 포맷 |
| `Date.UTC(y, m, 1)` | TZ 무관 UTC 날짜 생성, 월 자동 올림 처리 |
| `getUTCFullYear()` / `getUTCMonth()` | TZ 무관 연/월 추출 |
| `kstDayRange(key).start` | 날짜 키 → KST 기준 UTC 범위 변환 |
