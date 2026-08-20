---
aliases:
  - 국제화API
  - 날짜 포맷
  - 타임존 변환
  - ISO 3166-1
  - Intl.DateTimeFormatPartTypes
tags:
  - JavaScript
related:
  - "[[00_JS_Ecosystem_HomePage]]"
  - "[[JS_Intl]]"
  - "[[React_DatePicker]]"
  - "[[Snippet_date-statistics-pattern]]"
---
# JS_Intl — 국제화 포맷 API

> [!info] 
> Intl = 브라우저 내장 국제화(Internationalization) API. 
> 날짜·시간·숫자·통화·상대시간을 로케일(언어/지역)에 맞게 포맷한다. 
> `"2024-01-15"` → `"2024년 1월 15일"` 변환, `"5분 전"` 같은 상대 시간 표현. 
> Date 객체 생성·계산 → [[JS_Date]]

---

# Intl이란 ⭐️⭐️⭐️⭐️

```txt
브라우저에 내장된 국제화 API — 설치 없이 사용 가능

"국제화"가 필요한 이유:
  날짜 표기:  미국 1/15/2024  vs  한국 2024년 1월 15일  vs  독일 15.01.2024
  숫자 구분:  미국 1,000.50   vs  독일 1.000,50
  통화:       $100            vs  ₩100,000

Intl.DateTimeFormat   날짜·시간 포맷
Intl.RelativeTimeFormat  상대 시간 ("5분 전", "3일 후")
Intl.NumberFormat     숫자·통화 포맷
```

## 로케일(locale)이란

```txt
로케일 = "어느 나라, 어느 언어"를 나타내는 문자열
  'ko-KR'  한국어 / 한국
  'en-US'  영어 / 미국
  'en-GB'  영어 / 영국  ← hourCycle: 'h23' 조합에서 순수 숫자만 반환
  'en-CA'  영어 / 캐나다 ← 날짜가 YYYY-MM-DD 형태 → ISO 8601 파싱에 활용
  'ja-JP'  일본어 / 일본
  'de-DE'  독일어 / 독일
  'sv-SE'  스웨덴어 / 스웨덴 ← 날짜가 YYYY-MM-DD HH:mm:ss 형태 → 로그·타임스탬프에 활용

Intl의 모든 클래스는 첫 번째 인자로 로케일을 받음
로케일을 생략하면 브라우저/시스템의 기본 설정을 따름
```

---

# Intl.DateTimeFormat — 날짜 포맷 ⭐️⭐️⭐️⭐️

```typescript
const date = new Date('2024-01-15T09:30:00');

// 한국 형식
new Intl.DateTimeFormat('ko-KR').format(date)
// "2024. 1. 15."

// 옵션으로 상세 조정
new Intl.DateTimeFormat('ko-KR', {
  year:  'numeric',  // 2024
  month: 'long',     // 1월
  day:   'numeric',  // 15일
}).format(date)
// "2024년 1월 15일"

// 시간 포함
new Intl.DateTimeFormat('ko-KR', {
  month:  'long',
  day:    'numeric',
  hour:   '2-digit',
  minute: '2-digit',
}).format(date)
// "1월 15일 오전 09:30"
```

## 주요 옵션

| 옵션          | 값                                                | 출력 예                     |
| ----------- | ------------------------------------------------ | ------------------------ |
| `year`      | `'numeric'` / `'2-digit'`                        | `2024` / `24`            |
| `month`     | `'long'` / `'short'` / `'numeric'` / `'2-digit'` | `1월` / `1월` / `1` / `01` |
| `day`       | `'numeric'` / `'2-digit'`                        | `15` / `15`              |
| `hour`      | `'numeric'` / `'2-digit'`                        | `9` / `09`               |
| `minute`    | `'numeric'` / `'2-digit'`                        | `30` / `30`              |
| `weekday`   | `'long'` / `'short'`                             | `월요일` / `월`              |
| `timeZone`  | `'Asia/Seoul'` 등                                 | 서울 기준으로 변환               |
| `hour12`    | `true` / `false`                                 | 오전/오후 여부                 |
| `hourCycle` | `'h11'`/`'h12'`/`'h23'`/`'h24'`                  | 시간 범위 명시적 지정             |


## hourCycle — 시간 범위를 정확히 ⭐️⭐️⭐️

```typescript
// 현재 KST 시간을 0~23 정수로 얻기
function kstHour(now = new Date()): number {
  return Number(
    new Intl.DateTimeFormat('en-GB', {
      timeZone:  'Asia/Seoul',
      hour:      'numeric',
      hourCycle: 'h23',   // 자정 = 0, 오후 11시 = 23
    }).format(now),
  );
}
// kstHour() → 0~23 정수
```

```txt
hourCycle 4가지:
  h11  → 0~11  오전/오후 구분, 자정 = 0
  h12  → 1~12  오전/오후 구분, 자정 = 12 (일반적인 12시간제)
  h23  → 0~23  24시간제, 자정 = 0        ← 통계·집계에 가장 유용
  h24  → 1~24  24시간제, 자정 = 24

h23을 쓰는 이유:
  0~23 = 배열 인덱스와 1:1 대응 → hourly[kstHour()] = 바로 버킷에 접근
  h24면 자정이 24 → 배열 범위 초과

hour12: false 대신 hourCycle: 'h23'을 쓰는 이유:
  hour12: false → 로케일마다 자정을 "0" 또는 "24"로 다르게 출력할 수 있음
  hourCycle: 'h23' → 항상 0~23 범위 고정, 로케일 무관

en-GB (영국 영어):
  hour: 'numeric' + hourCycle: 'h23' 조합에서
  "15" 처럼 단순 숫자만 반환 — Number() 변환이 깔끔
  en-US는 로케일에 따라 "3 PM" 같은 형태가 나올 수 있음
  en-GB는 이 조합에서 숫자만 출력하는 것이 안정적
```

## 타임존 지정

```typescript
new Intl.DateTimeFormat('ko-KR', {
  timeZone: 'Asia/Seoul',
  year:  'numeric',
  month: 'long',
  day:   'numeric',
  hour:  '2-digit',
  minute: '2-digit',
}).format(new Date())
// 서버나 다른 환경에서도 항상 한국 시간으로 표시
```

```txt
timeZone을 명시하는 이유:
  지정 안 하면 코드 실행 환경의 로컬 시간 기준
  서버(UTC)에서 실행되면 브라우저와 다른 결과 → 버그

  서울 시간을 항상 보여주려면 timeZone: 'Asia/Seoul' 명시
```

## toLocaleString 간편 버전

```typescript
// new Intl.DateTimeFormat().format()의 줄임 표현
date.toLocaleString('ko-KR', { month: 'long', day: 'numeric' })
// 동일한 결과, 짧은 표현

date.toLocaleDateString('ko-KR')  // 날짜만
date.toLocaleTimeString('ko-KR')  // 시간만
```

## formatToParts — 부분별로 꺼내기 ⭐️⭐️⭐️⭐️


```typescript
// format()  → "2024년 1월 15일 월요일 오후 3시 30분" (완성된 문자열)
// formatToParts() → [{ type: 'year', value: '2024' }, { type: 'month', ... }]
//                    각 부분을 따로 꺼낼 수 있는 배열

const parts = new Intl.DateTimeFormat('ko-KR', {
  timeZone: 'Asia/Seoul',
  year:    'numeric',
  month:   'long',
  day:     'numeric',
  weekday: 'short',
  hour:    '2-digit',
  minute:  '2-digit',
  hour12:  false,
}).formatToParts(new Date());
// parts = [
//   { type: 'year',    value: '2024' },
//   { type: 'literal', value: '년 ' },
//   { type: 'month',   value: '1' },
//   { type: 'literal', value: '월 ' },
//   { type: 'day',     value: '15' },
//   { type: 'weekday', value: '월' },
//   { type: 'hour',    value: '15' },
//   { type: 'minute',  value: '30' },
//   ...
// ]
```

```typescript
// 타입별로 꺼내는 헬퍼 함수
const get = (type: Intl.DateTimeFormatPartTypes) => {
  return parts.find((part) => part.type === type)?.value ?? '';
  //            ↑ type이 일치하는 part를 찾아서 value만 꺼냄
  //                                              ↑ 없으면 빈 문자열
};

// 원하는 형식으로 조합
const label = `${get('year')}년 ${get('month')}월 ${get('day')}일 · ${get('weekday')} · ${get('hour')}:${get('minute')}`;
// "2024년 1월 15일 · 월 · 15:30"
```

```txt
Intl.DateTimeFormatPartTypes:
  TypeScript 타입 — formatToParts()가 반환하는 part.type의 가능한 값들
  'year' | 'month' | 'day' | 'weekday' | 'hour' | 'minute' | 'second'
  | 'dayPeriod' | 'era' | 'timeZoneName' | 'literal'

  literal:
    숫자 사이의 구분자 ("년 ", "월 ", " · " 등)
    로케일에 따라 자동으로 들어오는 텍스트

왜 formatToParts()를 쓰는가:
  format()은 로케일 형식 그대로만 나옴 → 커스텀 불가
  formatToParts()는 각 부분을 따로 꺼내서 원하는 형식으로 조합 가능

  예: "2024년 1월 15일 · 월 · 15:30"
  → 이 형식은 어떤 locale도 기본으로 지원 안 함
  → formatToParts()로 year·month·day·weekday·hour·minute 각각 꺼내서 조합

get() 헬퍼 패턴:
  parts.find((p) => p.type === 'year')?.value
  → type으로 찾고 value만 꺼내는 패턴을 함수로 재사용
  Intl.DateTimeFormatPartTypes로 type에 자동완성 + 오타 방지
```

---

# 상대 시간 — "방금 전", "3일 후" ⭐️⭐️⭐️⭐️

```typescript
const rtf = new Intl.RelativeTimeFormat('ko-KR', { numeric: 'auto' });

rtf.format(-5, 'minute')   // "5분 전"
rtf.format(-1, 'day')      // "어제"  ← numeric: 'auto' 덕분에
rtf.format(-3, 'day')      // "3일 전"
rtf.format(-1, 'month')    // "지난달"
rtf.format(1,  'day')      // "내일"
rtf.format(3,  'week')     // "3주 후"
```

```txt
numeric: 'auto' vs 'always':
  'auto':   -1 day → "어제", -1 month → "지난달" (자연스러운 표현)
  'always': -1 day → "1일 전", -1 month → "1달 전" (항상 숫자 형식)
```

## 자동 단위 선택 패턴 ⭐️⭐️⭐️⭐️

```typescript
// API 응답의 createdAt 문자열을 "5분 전" 형태로 표시
function formatRelativeTime(dateString: string): string {
  const date = new Date(dateString);
  const now  = new Date();
  const diffMs = now.getTime() - date.getTime();

  const seconds = Math.floor(diffMs / 1000);
  const minutes = Math.floor(seconds / 60);
  const hours   = Math.floor(minutes / 60);
  const days    = Math.floor(hours   / 24);
  const months  = Math.floor(days    / 30);
  const years   = Math.floor(days    / 365);

  const rtf = new Intl.RelativeTimeFormat('ko-KR', { numeric: 'auto' });

  if (seconds < 60)  return rtf.format(-seconds, 'second');  // "5초 전"
  if (minutes < 60)  return rtf.format(-minutes, 'minute');  // "5분 전"
  if (hours   < 24)  return rtf.format(-hours,   'hour');    // "3시간 전"
  if (days    < 30)  return rtf.format(-days,    'day');     // "어제", "3일 전"
  if (months  < 12)  return rtf.format(-months,  'month');   // "지난달", "3달 전"
  return rtf.format(-years, 'year');                          // "작년", "3년 전"
}

// 사용
formatRelativeTime('2024-01-15T09:30:00.000Z')  // "3일 전"
```

```txt
음수(-) 를 넣는 이유:
  과거 = 음수, 미래 = 양수
  diff는 양수(now - past > 0)이므로 format에 넣을 때 음수로 변환
  rtf.format(-5, 'minute') → "5분 전"
  rtf.format(5, 'minute')  → "5분 후"
```

---

# Intl.NumberFormat — 숫자·통화 포맷 ⭐️⭐️⭐️

```typescript
// 기본 숫자 (천 단위 구분)
new Intl.NumberFormat('ko-KR').format(1234567)
// "1,234,567"

// 통화
new Intl.NumberFormat('ko-KR', {
  style:    'currency',
  currency: 'KRW',
}).format(50000)
// "₩50,000"

new Intl.NumberFormat('en-US', {
  style:    'currency',
  currency: 'USD',
}).format(99.99)
// "$99.99"

// 소수점 제한
new Intl.NumberFormat('ko-KR', {
  minimumFractionDigits: 0,
  maximumFractionDigits: 2,
}).format(3.14159)
// "3.14"

// 퍼센트
new Intl.NumberFormat('ko-KR', { style: 'percent' }).format(0.857)
// "86%"
```

---
# sv-SE 트릭 — 타임존 적용된 읽기 좋은 날짜 ⭐️⭐️⭐️⭐️

```typescript
// 처음 보면 의아한 코드 — 왜 스웨덴어 로케일?
new Intl.DateTimeFormat('sv-SE', {
  timeZone: 'Asia/Seoul',
  year:   'numeric',
  month:  '2-digit',
  day:    '2-digit',
  hour:   '2-digit',
  minute: '2-digit',
  second: '2-digit',
  hour12: false,
}).format(new Date())
// "2024-01-15 02:30:00"
```

```txt
sv-SE (스웨덴 로케일)를 쓰는 이유:
  sv = 스웨덴어(Swedish), SE = 스웨덴(Sweden) → 로케일 코드 형식: 언어-국가
  로케일 코드 개념 → 위 "로케일(locale)이란" 섹션 참고

  스웨덴 날짜 형식이 "YYYY-MM-DD HH:mm:ss" 형태
  = 사람이 읽기 좋은 ISO 8601 형식과 동일

  toISOString()과의 차이:
    toISOString()       → "2024-01-15T02:30:00.000Z"  (항상 UTC, T와 Z 포함)
    sv-SE + timeZone    → "2024-01-15 02:30:00"         (지정 타임존 기준, 공백 구분)

  왜 유용한가:
    로그에 남길 때, ops 메일에 넣을 때, DB에 사람이 읽는 형태로 저장할 때
    한국 시간 기준으로 "2024-01-15 02:30:00" 형태를 원하면 sv-SE + KST가 가장 간단
```

```typescript
// 유틸 함수로 만들어두면 편함
const KST = 'Asia/Seoul';

export function formatKstDateTime(date = new Date()): string {
  return new Intl.DateTimeFormat('sv-SE', {
    timeZone: KST,
    year:   'numeric',
    month:  '2-digit',
    day:    '2-digit',
    hour:   '2-digit',
    minute: '2-digit',
    second: '2-digit',
    hour12: false,
  }).format(date);
}
// formatKstDateTime() → "2024-01-15 02:30:00"

// 날짜만 필요하면 slice
formatKstDateTime().slice(0, 10)  // "2024-01-15"
```

## en-CA — @db.Date용 KST 달력 날짜 ⭐️⭐️⭐️⭐️

```typescript
/** 임의 시각 → KST 달력 날짜 Date (@db.Date 컬럼에 저장할 때) */
export function toKstDate(instant: Date = new Date()): Date {
  const parts = new Intl.DateTimeFormat('en-CA', {
    timeZone: 'Asia/Seoul',
    year:  'numeric',
    month: '2-digit',
    day:   '2-digit',
  }).format(instant);
  // parts = "2024-01-16"  (en-CA = YYYY-MM-DD 형식)
  return new Date(`${parts}T00:00:00.000Z`);
  // "2024-01-16T00:00:00.000Z" — Prisma @db.Date가 날짜만 저장
}
```

```txt
왜 en-CA인가:
  en-CA (캐나다 영어)의 날짜 형식 = "YYYY-MM-DD"
  → new Date(parts) 에 바로 넣을 수 있는 ISO 8601 날짜 문자열
  sv-SE와 달리 시각 없이 날짜만 가져올 때 편함

왜 T00:00:00.000Z를 붙이는가:
  Prisma @db.Date 컬럼은 Date 객체를 받음
  날짜 문자열 "2024-01-16"만 넘기면 JS가 로컬 TZ로 해석 → 하루 밀릴 수 있음
  "2024-01-16T00:00:00.000Z"처럼 UTC 자정으로 명시하면 안전

언제 쓰는가:
  @db.Date 컬럼에 "오늘 KST 날짜"를 저장할 때
  UTC 기준으로 오전 9시 이전에는 KST 날짜가 하루 앞서므로 직접 변환 필요

예시:
  new Date('2024-01-15T17:30:00.000Z')  // UTC 1월 15일 오후 5시 30분
  toKstDate(...)                         // KST 기준 1월 16일 → Date("2024-01-16T00:00:00.000Z")
  // → DB에 2024-01-16 저장

sv-SE vs en-CA:
  sv-SE → 날짜 + 시각이 필요할 때  ("2024-01-16 02:30:00")
  en-CA → 날짜만 필요할 때         ("2024-01-16")
```

## en-CA + +09:00 — KST 기간 범위 ⭐️⭐️⭐️⭐️

```typescript
/** KST 오늘 00:00 ~ 익일 00:00 — DB 쿼리 기간 범위 */
function kstTodayRange(now = new Date()): { start: Date; end: Date } {
  const day = new Intl.DateTimeFormat('en-CA', {
    timeZone: 'Asia/Seoul',
    year: 'numeric', month: '2-digit', day: '2-digit',
  }).format(now);                            // "2026-07-15"

  const start = new Date(`${day}T00:00:00+09:00`);
  //                                       ↑ +09:00 = KST 오프셋 명시
  //                                         KST 자정 → UTC instant로 정확 변환
  const end   = new Date(start.getTime() + 24 * 60 * 60 * 1000);
  return { start, end };
}

/** KST 이번 주 월요일 00:00 ~ 다음 월요일 00:00 */
function kstWeekRange(now = new Date()): { start: Date; end: Date } {
  const { start: todayStart } = kstTodayRange(now);

  const weekday = new Intl.DateTimeFormat('en-US', {
    timeZone: 'Asia/Seoul',
    weekday: 'short',    // 'Mon' | 'Tue' | 'Wed' | 'Thu' | 'Fri' | 'Sat' | 'Sun'
  }).format(now);
  // ↑ 왜 en-US인가:
  //   'ko-KR' → '월' '화' '수' ... (한글 — 코드 매핑에 불편)
  //   'en-US' → 'Mon' 'Tue' 'Wed' ... (영문 — { Mon:0, Tue:1 } 매핑이 단순)
  //   en-CA도 같은 결과 — 어떤 영문 로케일이든 weekday: 'short'는 동일

  const monOffset =
    { Mon: 0, Tue: 1, Wed: 2, Thu: 3, Fri: 4, Sat: 5, Sun: 6 }[weekday] ?? 0;
  //  ↑ 오늘이 월요일로부터 며칠째인가

 // 밀리초 상수 [[JS_Date#밀리초 상수 — 자주 쓰는 값 ⭐️⭐️⭐️]] 여기에서 86400000 참조 
 // 86400000 = 24 * 60 * 60 * 1000
  const start = new Date(todayStart.getTime() - monOffset * 86400000);
  const end   = new Date(start.getTime() + 7 * 86400000);
  return { start, end };
}
```

```txt
T00:00:00+09:00 패턴:
  en-CA로 "2026-07-15" 날짜 문자열을 얻고
  +09:00을 붙여 KST 자정을 UTC instant로 변환
  → Railway·Vercel 서버 TZ가 UTC여도 항상 KST 자정

  setHours(0,0,0,0)을 쓰면 안 되는 이유:
  서버 TZ에 따라 결과가 달라짐
  UTC 서버에서 setHours(0)은 UTC 자정 → KST 자정이 아님

kstTodayRange 언제 쓰는가:
  WHERE created_at >= start AND created_at < end
  오늘 DAU · 오늘 뽑기 횟수 · 오늘 게시글 수

kstWeekRange 언제 쓰는가:
  이번 주 기준 집계 쿼리
  주간 순위 · 주간 DAU · 이번 주 게시글 수

  monOffset 계산 예:
  오늘 수요일(Wed: 2) → 오늘 00:00 - 2일 = 월요일 00:00
  월요일 00:00 + 7일 = 다음 주 월요일 00:00 (범위 끝)
```

## weekday 값으로 요일 조건 판단 ⭐️⭐️⭐️

```typescript
// KST 기준 오늘이 월요일인지 확인
function isKstMonday(now = new Date()): boolean {
  return (
    new Intl.DateTimeFormat('en-US', {
      timeZone: 'Asia/Seoul',
      weekday:  'short',
    }).format(now) === 'Mon'
    // weekday: 'short' + en-US → 'Mon' | 'Tue' | 'Wed' | 'Thu' | 'Fri' | 'Sat' | 'Sun'
    // 'Mon' 과 비교 → true/false
  );
}
```

```typescript
// 범용 패턴 — 어떤 요일이든
function isKstWeekday(
  day: 'Mon' | 'Tue' | 'Wed' | 'Thu' | 'Fri' | 'Sat' | 'Sun',
  now = new Date(),
): boolean {
  return (
    new Intl.DateTimeFormat('en-US', {
      timeZone: 'Asia/Seoul',
      weekday:  'short',
    }).format(now) === day
  );
}

isKstWeekday('Mon')  // 오늘 KST 기준 월요일인지
isKstWeekday('Sun')  // 일요일인지
```

```txt
언제 쓰는가:
  주간 seed, 주간 초기화 크론 → 월요일에만 실행
  주말 알림 → Sat·Sun 체크
  특정 요일 한정 기능

왜 getDay() 안 쓰는가:
  new Date().getDay() → 서버 TZ 기준 (UTC 서버면 UTC 요일)
  Intl.DateTimeFormat + Asia/Seoul → KST 기준 요일 보장
  UTC 서버에서 월요일 새벽 9시 이전에는 getDay()가 일요일을 반환

en-US + weekday: 'short' → 'Mon'~'Sun' 영문 고정
ko-KR + weekday: 'short' → '월'~'일' 한글 (문자열 비교가 덜 직관적)
```

## ko-KR month + day — "8월 11일" 형식  & 지난주 KST 월~일 라벨⭐️⭐️⭐️

```typescript
// 날짜를 "8월 11일" 형식으로
const fmt = (d: Date) =>
  new Intl.DateTimeFormat('ko-KR', {
    timeZone: 'Asia/Seoul',
    month: 'long',    // '8월'
    day:   'numeric', // '11일'
  }).format(d);

fmt(new Date('2026-08-11T00:00:00+09:00'))  // "8월 11일"
fmt(new Date('2026-08-17T00:00:00+09:00'))  // "8월 17일"
```

```typescript
/** 지난주 KST 월~일 라벨 — "8월 11일 ~ 8월 17일" */
function kstPreviousWeekRangeLabel(now = new Date()): string {
  const DAY_MS    = 86400000;
  const todayKey  = kstDateKey(now);
  const todayStart = new Date(`${todayKey}T00:00:00+09:00`);

  // 이번 주 월요일 구하기 (kstWeekRange와 동일 로직)
  const weekday = new Intl.DateTimeFormat('en-US', {
    timeZone: 'Asia/Seoul', weekday: 'short',
  }).format(now);
  const monOffset = { Mon: 0, Tue: 1, Wed: 2, Thu: 3, Fri: 4, Sat: 5, Sun: 6 }[weekday] ?? 0;
  const thisMon = new Date(todayStart.getTime() - monOffset * DAY_MS);

  // 지난주 범위: 이번 주 월요일 기준으로 7일 전
  const prevMon = new Date(thisMon.getTime() - 7 * DAY_MS);  // 지난주 월요일
  const prevSun = new Date(thisMon.getTime() - DAY_MS);      // 지난주 일요일 (이번 주 월요일 - 1일)

  // ko-KR로 "8월 11일" 형식 포맷
  const fmt = (d: Date) =>
    new Intl.DateTimeFormat('ko-KR', {
      timeZone: 'Asia/Seoul',
      month: 'long',
      day:   'numeric',
    }).format(d);

  return `${fmt(prevMon)} ~ ${fmt(prevSun)}`;
  // "8월 11일 ~ 8월 17일"
}
```

```txt
계산 흐름:
  오늘 todayStart (KST 자정)
  - monOffset * DAY_MS = thisMon (이번 주 월요일 00:00 KST)
  - 7 * DAY_MS         = prevMon (지난주 월요일)
  thisMon - 1일         = prevSun (지난주 일요일)

  왜 prevSun = thisMon - 1일인가:
  이번 주 월요일 00:00 - 하루 = 지난주 일요일 00:00
  일요일 자정을 가리키지만 날짜만 쓰므로 "지난주 일요일"

locale 조합:
  en-US + weekday: 'short' → 요일 계산용 'Mon'~'Sun' 영문
  ko-KR + month/day         → 화면 표시용 "8월 11일" 한글
  한 함수 안에서 두 locale을 목적에 맞게 혼용
```

---
# 자주 쓰는 유틸 함수 정리

```typescript
// 날짜 포맷 (화면 표시용)
function formatDate(date: Date | string): string {
  return new Intl.DateTimeFormat('ko-KR', {
    year:  'numeric',
    month: 'long',
    day:   'numeric',
  }).format(new Date(date));
  // "2024년 1월 15일"
}

// 날짜 + 시간
function formatDateTime(date: Date | string): string {
  return new Intl.DateTimeFormat('ko-KR', {
    month:  'long',
    day:    'numeric',
    hour:   '2-digit',
    minute: '2-digit',
  }).format(new Date(date));
  // "1월 15일 오전 09:30"
}

// 상대 시간 ("5분 전")
function formatRelativeTime(date: Date | string): string {
  const ms      = Date.now() - new Date(date).getTime();
  const seconds = Math.floor(ms / 1000);
  const minutes = Math.floor(seconds / 60);
  const hours   = Math.floor(minutes / 60);
  const days    = Math.floor(hours   / 24);
  const rtf     = new Intl.RelativeTimeFormat('ko-KR', { numeric: 'auto' });

  if (seconds < 60) return rtf.format(-seconds, 'second');
  if (minutes < 60) return rtf.format(-minutes, 'minute');
  if (hours   < 24) return rtf.format(-hours,   'hour');
  return rtf.format(-days, 'day');
}

// 통화
function formatCurrency(amount: number, currency = 'KRW'): string {
  return new Intl.NumberFormat('ko-KR', {
    style: 'currency',
    currency,
  }).format(amount);
  // "₩50,000"
}
```

---
# ISO 3166-1 — 국가 코드 표준 ⭐️⭐️⭐️⭐️

```txt
ISO 3166-1 = 국가를 2글자 알파벳으로 표현하는 국제 표준
  KR = 대한민국
  JP = 일본
  US = 미국
  GB = 영국 (Great Britain)
  FR = 프랑스
  DE = 독일
  CN = 중국
  TW = 대만
  HK = 홍콩

외부 API에서 자주 보이는 필드명:
  iso_3166_1       → 표준 이름 그대로 (TMDB, 공공 API 등)
  countryCode      → 앱마다 다른 이름으로 래핑
  country          → 2글자 코드를 그냥 country로 쓰기도 함
```

```typescript
// TMDB API 응답 예시
type ProductionCountry = {
  iso_3166_1: string;  // "KR", "US", "JP"
  name:       string;  // "South Korea", "United States of America"
};

// 실전 — 국가 코드 추출 패턴
const originCountries =
  detail.origin_country ??                                    // 배열이면 바로 사용
  detail.production_countries?.map((c) => c.iso_3166_1) ??  // 없으면 국가명에서 추출
  [];
```

```txt
detail.origin_country ?? ... ?? []:
  origin_country가 있으면 그대로 사용 (string[])
  없으면 production_countries에서 iso_3166_1만 뽑음
  둘 다 없으면 빈 배열

?? (nullish coalescing):
  null 또는 undefined일 때만 다음으로 넘어감
  빈 배열 []은 falsy지만 ?? 에서는 그대로 사용됨
  → || 와 다름 (|| 는 빈 배열도 falsy로 취급)
```

## Intl.DisplayNames — 코드 → 국가 이름 표시

```typescript
// 국가 코드를 사람이 읽을 수 있는 이름으로 변환
const countryNames = new Intl.DisplayNames(['ko'], { type: 'region' });

countryNames.of('KR')  // "대한민국"
countryNames.of('US')  // "미국"
countryNames.of('JP')  // "일본"
countryNames.of('GB')  // "영국"

// 영문으로
const enNames = new Intl.DisplayNames(['en'], { type: 'region' });
enNames.of('KR')  // "South Korea"

// 유틸 함수
function toCountryName(code: string, locale = 'ko'): string {
  try {
    return new Intl.DisplayNames([locale], { type: 'region' }).of(code) ?? code;
  } catch {
    return code;  // 알 수 없는 코드면 코드 그대로
  }
}

toCountryName('KR')      // "대한민국"
toCountryName('UNKNOWN') // "UNKNOWN" (fallback)

// API 응답에서 국가 이름 표시
const countries = detail.production_countries
  ?.map((c) => toCountryName(c.iso_3166_1))
  .join(', ');
// "미국, 영국"
```

## 관련 표준

```txt
ISO 3166-1 alpha-2: 2글자 (KR, US, JP) ← 가장 흔히 보이는 것
ISO 3166-1 alpha-3: 3글자 (KOR, USA, JPN)
ISO 3166-1 numeric: 숫자 (410, 840, 392)

ISO 639-1: 언어 코드 (ko, en, ja)
  → locale 문자열: "ko-KR", "en-US", "ja-JP"
  → Intl API의 locale 파라미터에서 사용
```