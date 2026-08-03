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