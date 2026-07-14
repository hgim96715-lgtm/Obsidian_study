---
aliases:
  - 국제화API
  - 날짜 포맷
  - 타임존 변환
tags:
  - JavaScript
related:
  - "[[00_JS_Ecosystem_HomePage]]"
  - "[[JS_Date]]"
  - "[[JS_BrowserAPI]]"
  - "[[JS_Primitive_Methods]]"
---
# JS_Intl — 국제화 API (Intl)

> [!info] 
> `Intl` = JavaScript 내장 국제화 네임스페이스
타임존 변환, 언어별 날짜·숫자 포맷, 상대 시간 표현을 import 없이 브라우저/Node 어디서든 사용 가능.

---

# Intl.DateTimeFormat — 날짜 포맷 & 타임존 변환 ⭐️⭐️⭐️⭐️

```typescript
// 기본 사용
new Intl.DateTimeFormat('ko-KR', {
  year:  'numeric',
  month: 'long',
  day:   'numeric',
}).format(new Date())
// → '2025년 1월 15일'

// 시간 포함
new Intl.DateTimeFormat('ko-KR', {
  hour:   'numeric',
  minute: 'numeric',
  hour12: true,
}).format(new Date())
// → '오후 3:45'
```

## 타임존 변환 — KST 시간 구하기 ⭐️⭐️⭐️⭐️

```typescript
const KST = 'Asia/Seoul';

function kstHour(date = new Date()): number {
  const hour = Number(
    new Intl.DateTimeFormat('en-US', {
      timeZone: KST,
      hour:     'numeric',
      hour12:   false,    // 24시간제
    }).format(date),
  );
  // ⚠️ 자정(00:00)에 일부 엔진에서 '24'를 반환하는 엔진 버그 방어
  return hour === 24 ? 0 : hour;
}
```

```txt
왜 이렇게 복잡하게 타임존을 구하는가:

  Date 객체 자체는 타임존이 없음 — 항상 UTC 기준의 타임스탬프
  date.getHours()는 "실행 환경의 로컬 시간" 기준
  → 서버(UTC) / 브라우저(사용자 로컬)마다 다른 값이 나옴

  Intl.DateTimeFormat에 timeZone을 지정하면
  그 타임존 기준으로 날짜/시간을 포맷해서 문자열로 반환
  → 문자열을 Number()로 변환해서 숫자로 사용

  "KST 기준 몇 시인가"를 정확히 구하는 유일한 표준 방법
  (UTC+9 하드코딩보다 일광절약시간 등 엣지케이스를 자동 처리)
```

```txt
hour === 24 방어 코드가 필요한 이유:
  hour12: false + 'numeric' 조합에서
  자정(00:00)을 일부 JS 엔진(V8 특정 버전)이 '0'이 아닌 '24'로 반환하는 버그 존재
  → Number('24') = 24 → 24 % 24 = 0 또는 명시적으로 === 24 ? 0 으로 방어
```

## 타임존 기반 greeting — 시간대별 메시지 ⭐️⭐️⭐️

```typescript
type WelcomeCopy = { greeting: string; wish: string };

export function getWelcomeCopy(date = new Date()): WelcomeCopy {
  const hour = kstHour(date);

  if (hour >= 5  && hour < 11) return { greeting: '좋은 아침이에요.', wish: '오늘 하루도 행복하시길.' };
  if (hour >= 11 && hour < 14) return { greeting: '점심 시간이에요.',  wish: '맛있는 식사 하세요.' };
  if (hour >= 14 && hour < 18) return { greeting: '좋은 오후예요.',   wish: '오늘도 화이팅!' };
  if (hour >= 18 && hour < 22) return { greeting: '좋은 저녁이에요.', wish: '오늘 하루 고생 많으셨어요.' };
  return { greeting: '늦은 시간이에요.', wish: '따뜻하게 쉬세요.' };
}
```

## 주요 옵션

|옵션|값|설명|
|---|---|---|
|`timeZone`|`'Asia/Seoul'`, `'UTC'`, `'America/New_York'`|IANA 타임존 이름|
|`year`|`'numeric'` / `'2-digit'`|연도 포맷|
|`month`|`'numeric'` / `'2-digit'` / `'long'` / `'short'` / `'narrow'`|월 포맷|
|`day`|`'numeric'` / `'2-digit'`|일 포맷|
|`hour`|`'numeric'` / `'2-digit'`|시간|
|`minute`|`'numeric'` / `'2-digit'`|분|
|`hour12`|`true` / `false`|12시간제(오전/오후) / 24시간제|
|`weekday`|`'long'` / `'short'` / `'narrow'`|요일|

---

# Intl.DateTimeFormat vs toLocaleString ⭐️⭐️⭐️

```typescript
// 둘은 결과가 같음
new Intl.DateTimeFormat('ko-KR', options).format(date)
date.toLocaleString('ko-KR', options)

// 차이: DateTimeFormat은 인스턴스를 재사용할 수 있어서 여러 날짜를 포맷할 때 더 빠름
const formatter = new Intl.DateTimeFormat('ko-KR', { month: 'long', day: 'numeric' });
dates.map(d => formatter.format(d));  // 인스턴스 재사용 → 빠름
```

---

# Intl.NumberFormat — 숫자·통화 포맷 ⭐️⭐️⭐️

```typescript
// 통화
new Intl.NumberFormat('ko-KR', {
  style:    'currency',
  currency: 'KRW',
}).format(50000)
// → '₩50,000'

// 달러
new Intl.NumberFormat('en-US', {
  style:    'currency',
  currency: 'USD',
}).format(1234.56)
// → '$1,234.56'

// 천 단위 구분자
new Intl.NumberFormat('ko-KR').format(1234567)
// → '1,234,567'

// 소수점 자리수 제한
new Intl.NumberFormat('ko-KR', {
  minimumFractionDigits: 0,
  maximumFractionDigits: 2,
}).format(3.14159)
// → '3.14'

// 퍼센트
new Intl.NumberFormat('ko-KR', { style: 'percent' }).format(0.75)
// → '75%'
```

---

# Intl.RelativeTimeFormat — 상대 시간 ⭐️⭐️⭐️

```typescript
const rtf = new Intl.RelativeTimeFormat('ko', { numeric: 'auto' });

rtf.format(-1, 'day')    // → '어제'
rtf.format(-3, 'day')    // → '3일 전'
rtf.format(1, 'day')     // → '내일'
rtf.format(-2, 'hour')   // → '2시간 전'
rtf.format(-30, 'minute') // → '30분 전'
rtf.format(1, 'week')    // → '다음 주'
```

## 실전 — "N분 전" 유틸 함수

```typescript
function timeAgo(date: Date): string {
  const rtf  = new Intl.RelativeTimeFormat('ko', { numeric: 'auto' });
  const diff  = date.getTime() - Date.now();  // ms (음수 = 과거)
  const abs   = Math.abs(diff);

  if (abs < 60_000)        return rtf.format(Math.round(diff / 1_000),    'second');
  if (abs < 3_600_000)     return rtf.format(Math.round(diff / 60_000),   'minute');
  if (abs < 86_400_000)    return rtf.format(Math.round(diff / 3_600_000),'hour');
  if (abs < 2_592_000_000) return rtf.format(Math.round(diff / 86_400_000),'day');
  return rtf.format(Math.round(diff / 2_592_000_000), 'month');
}

timeAgo(new Date(Date.now() - 5 * 60 * 1000));  // '5분 전'
timeAgo(new Date(Date.now() - 2 * 3600 * 1000)); // '2시간 전'
```

---

# Intl.Collator — 언어별 문자열 정렬 ⭐️⭐️

```typescript
// ❌ localeCompare 없이 — ASCII 순서로 정렬 (한글 제대로 안 됨)
['다', '가', '나'].sort()

// ✅ 한국어 기준 정렬
['다', '가', '나'].sort(
  new Intl.Collator('ko').compare
)
// → ['가', '나', '다']

// 대소문자 무시 정렬
new Intl.Collator('en', { sensitivity: 'base' }).compare('A', 'a')  // 0 (같음)
```

---

# 한눈에

```txt
Intl.DateTimeFormat:
  new Intl.DateTimeFormat(locale, options).format(date)
  timeZone 옵션으로 특정 타임존 기준 포맷 가능
  KST 시간 구하기: timeZone: 'Asia/Seoul', hour12: false → Number(결과)
  ⚠️ 자정에 '24' 반환 엔진 버그 → === 24 ? 0 방어

Intl.NumberFormat:
  통화: style: 'currency', currency: 'KRW'
  천 단위 구분: 기본 동작
  소수점: minimumFractionDigits / maximumFractionDigits

Intl.RelativeTimeFormat:
  rtf.format(-3, 'day')  → '3일 전'
  numeric: 'auto'로 '어제', '내일' 같은 자연어 표현 가능

Intl.Collator:
  배열.sort(new Intl.Collator('ko').compare)  → 한국어 정렬

공통:
  import 없이 전역으로 사용 가능 (브라우저 + Node.js 10+)
  인스턴스를 만들어서 재사용하면 여러 값 포맷 시 성능 이점
```