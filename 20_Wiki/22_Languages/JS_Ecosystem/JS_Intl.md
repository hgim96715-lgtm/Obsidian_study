---
aliases:
  - 국제화API
  - 날짜 포맷
  - 타임존 변환
tags:
  - JavaScript
related:
  - "[[00_JS_Ecosystem_HomePage]]"
  - "[[JS_Intl]]"
  - "[[React_DatePicker]]"
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
  'ja-JP'  일본어 / 일본
  'de-DE'  독일어 / 독일
  'sv-SE'  스웨덴어 / 스웨덴  ← 날짜가 YYYY-MM-DD HH:mm:ss 형태라 로그·타임스탬프에 활용

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

|옵션|값|출력 예|
|---|---|---|
|`year`|`'numeric'` / `'2-digit'`|`2024` / `24`|
|`month`|`'long'` / `'short'` / `'numeric'` / `'2-digit'`|`1월` / `1월` / `1` / `01`|
|`day`|`'numeric'` / `'2-digit'`|`15` / `15`|
|`hour`|`'numeric'` / `'2-digit'`|`9` / `09`|
|`minute`|`'numeric'` / `'2-digit'`|`30` / `30`|
|`weekday`|`'long'` / `'short'`|`월요일` / `월`|
|`timeZone`|`'Asia/Seoul'` 등|서울 기준으로 변환|

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
