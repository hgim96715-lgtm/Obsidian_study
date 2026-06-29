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
  - "[[SQL_Date_Functions]]"
  - "[[NestJS_StatsBucket]]"
  - "[[date-statistics-pattern]]"
---
# JS_Date — Date 객체

> [!info] 
> Date는 내부적으로 "UTC 타임스탬프" 하나만 저장한다. 문자열 파싱 방식(날짜만 / 시간 포함 / 오프셋 포함)에 따라 해석이 달라지고, "날짜 자체"와 "경과 시간(ms)"을 구분해서 다뤄야 상대 날짜 표시(오늘/어제/N일 전) 같은 곳에서 버그가 안 생긴다.

```
이 노트의 모든 "트릭"(+09:00 / T12:00:00 / en-CA / startOfDay)은
전부 "Date는 UTC 타임스탬프 하나, 시간대는 표시할 때만 적용" 이라는 규칙 하나에서 나온 변형임
→ 그 규칙부터 이해하면 나머지는 자연스럽게 이어짐
```

---

# Date 객체 생성

```javascript
// 현재 날짜 + 시간
const now = new Date();

// 문자열로 생성
const date1 = new Date('2026-06-08');
const date2 = new Date('2026-06-08T14:30:00');
const date3 = new Date('2026-06-08T14:30:00.000Z');  // UTC

// 숫자(밀리초)로 생성
const date4 = new Date(0);              // 1970-01-01 (Unix epoch)
const date5 = new Date(1000000000000);

// 연/월/일로 생성 (월은 0부터 시작)
const date6 = new Date(2026, 5, 8);    // 2026-06-08 (5 = 6월)
//                          ↑ 0 = 1월 / 5 = 6월 / 11 = 12월
```

```
⚠️ 월은 0부터 시작:
  new Date(2026, 0, 1)   → 2026년 1월 1일
  new Date(2026, 11, 31) → 2026년 12월 31일
  → 문자열 방식 권장: new Date('2026-06-08')
```

---

# 문자열 파싱 — 핵심 규칙 ⭐️⭐️

```
Date 객체는 내부적으로 항상 "UTC 기준 밀리초 타임스탬프" 하나로 저장됨
→ "시간대"는 저장되는 게 아니라, 화면에 표시할 때만 적용되는 개념

그런데 문자열로 Date 를 만들 때, 형태에 따라 해석 방식이 다름 (MDN 명시 사실):

  날짜만 있는 문자열      → UTC 로 해석
  new Date('2026-06-16')
  = 2026-06-16T00:00:00Z (UTC 자정)

  시간까지 있고 오프셋 없는 문자열 → 로컬 시간대로 해석
  new Date('2026-06-16T12:00:00')
  = 실행 환경의 로컬 시간 기준 정오

  오프셋(Z 또는 +09:00)이 명시된 문자열 → 그 오프셋으로 해석
  new Date('2026-06-16T12:00:00+09:00')
  = 한국 시간 기준 정오 (실행 환경 무관, 항상 동일)
```

```
⚠️ 이 비대칭은 JS 스펙의 역사적 오류(historical spec quirk)임
   "날짜만 있으면 왜 UTC?" 는 직관과 다름 — 그냥 외워야 하는 규칙
   → 이 규칙 하나가 이 노트의 모든 "트릭"의 원인
```

## 이 규칙 때문에 생기는 버그

```javascript
// 한국(KST, UTC+9)에서 실행
new Date('2026-06-16').getDate()
// → 16 (UTC 자정 + 9시간 = 같은 날 09시, 날짜는 안 밀림)

// 만약 실행 환경이 UTC-5(미국 동부)였다면
new Date('2026-06-16').getDate()
// → 15 (UTC 자정 - 5시간 = 전날 19시, 날짜가 하루 밀림 ⚠️)
```

```
서버(Railway/Vercel 등)는 보통 UTC 로 실행되고
배포 환경 / 팀원 PC / 사용자 브라우저마다 시간대가 다 다를 수 있음
→ "날짜만" 다루고 싶은데 위처럼 환경에 따라 결과가 달라지는 게 모든 문제의 시작점
```

---

# ISO 8601 — 날짜/시간 표준 문자열 ⭐️

```
ISO 8601 = 날짜·시간을 표현하는 국제 표준 형식
JS / DB / API 가 날짜를 주고받을 때 기본으로 쓰는 형식

형식:  YYYY-MM-DDTHH:mm:ss.sssZ
       └─날짜──┘ └────시간────┘└시간대┘

예시:  2026-06-16T09:30:00.000Z
        2026  06  16  T  09  30  00  .000  Z
        연      월  일     시   분  초  밀리초  UTC 표시
```

```javascript
// API 응답 / DB 에서 흔히 보는 형태
'2026-06-16T09:30:00.000Z'    // Z → UTC 기준 09:30 (한국 시간 18:30)
'2026-06-16T09:30:00+09:00'   // +09:00 → 이미 한국 시간 기준 09:30
'2026-06-16'                  // 날짜만 → 위 "핵심 규칙" 대로 UTC 로 해석됨

// 변수명 iso / isoString 은 보통 이 형식의 문자열을 의미
const formatDate = (iso: string) => {
  return new Date(iso).toLocaleDateString('ko-KR', { /* ... */ });
};
```

```bash
toISOString():
  Date 객체 → ISO 8601 문자열로 변환 (항상 UTC, Z로 끝남)
  new Date().toISOString()  // '2026-06-16T09:30:00.000Z'
  API 로 날짜를 보낼 때 표준 형식으로 변환하는 용도

new Date(iso) 가 항상 안전한 이유:
  ISO 8601 은 모든 JS 환경에서 동일하게 파싱되는 표준 형식
  반면 '20260616' (비표준) 은 환경마다 결과가 다를 수 있음
  → 비표준 형식 파싱은 [[#parseYmd — YYYYMMDD 형식 파싱 ⭐️]] 참고
```

---

# getTime() — 밀리초 변환 ⭐️

```javascript
const date = new Date('2026-06-08');
date.getTime()   // 1749340800000

// 날짜 비교
const start = new Date('2026-06-01');
const end   = new Date('2026-06-30');
start.getTime() < end.getTime()   // true
```

---

# 유효하지 않은 Date 검증 ⭐️

```javascript
const invalid = new Date('hello');
invalid instanceof Date    // true  ← Date 객체 맞음
invalid.getTime()          // NaN   ← 유효하지 않음

function isValidDate(date: Date): boolean {
  return !isNaN(date.getTime());
}

isNaN(new Date('2026-06-08').getTime())   // false → 유효
isNaN(new Date('hello').getTime())        // true  → 무효
```

```
잘못된 날짜도 Date 객체는 생성됨
→ instanceof Date 로는 유효성 판단 불가
→ getTime() → NaN 이면 유효하지 않은 날짜
```

---

# 날짜 조회 메서드

```javascript
const date = new Date('2026-06-08T14:30:00');

date.getFullYear()     // 2026  연도
date.getMonth()        // 5     월 (0부터 → 5 = 6월) ⚠️
date.getDate()         // 8     일
date.getDay()          // 1     요일 (0=일 / 1=월 / 6=토) ⚠️
date.getHours()        // 14    시
date.getMinutes()      // 30    분
date.getSeconds()      // 0     초
date.getTime()         // 밀리초 타임스탬프

date.getMonth() + 1    // 6 (실제 월 얻으려면 +1)
```

```
⚠️ get* 메서드들은 항상 "실행 환경의 로컬 시간대" 기준으로 동작함
   (위 "문자열 파싱 핵심 규칙" 에서 본 환경별 결과 차이가 여기서도 그대로 적용됨)
   시간대를 직접 지정하고 싶다면 get* 대신 toLocaleDateString(locale, { timeZone }) 사용
```

## 왜 getMonth() 는 0부터 시작하나 — 레거시 체인 ⭐️

```
"논리적인 설계"가 아니라 그대로 상속된 레거시임:

  1973 Unix(C, struct tm.tm_mon)   → 0~11 (0 = 1월)
  1995 Java(java.util.Date)        → C 관습 그대로 따라감
  1995 JavaScript(Date)            → Java Date 클래스를 그대로 본떠서 설계
                                      (Brendan Eich 본인이 "Java가 그렇게 했으니까"라고 직접 확인함)

→ Date 안에서도 기준이 통일되어 있지 않음:
  getMonth()  0부터 (0=1월 ~ 11=12월)
  getDate()   1부터 (1~31)
  getDay()    0부터 (0=일 ~ 6=토)

  → "왜 이렇게 설계했을까"를 논리적으로 추론할 수 있는 부분이 아니라, 그냥 외워야 하는 규칙
```

```
실용적으로 남은 이유 하나 — 배열 인덱싱:
  0부터 시작하면 월 이름 배열에 인덱스로 바로 꽂을 수 있음
  const monthNames = ['1월','2월','3월', /* ... */ '12월'];
  monthNames[new Date().getMonth()]   // 인덱스랑 값이 그대로 맞음

  반대로 "사람이 읽는 월(1~12)"이 필요한 모든 곳(화면 표시, DB 저장, API 전송)에선
  반드시 +1 을 해줘야 함 — 이게 이 노트 곳곳에 getMonth() + 1 이 반복되는 이유
```

> [!info] Java는 이후 java.time(LocalDate 등)에서 month를 1부터 시작하도록 고쳤지만, JS의 Date는 "웹을 깨지 않는다"는 원칙 때문에 0-indexed를 그대로 유지하고 있음 — 대신 2026년 3월 TC39에서 Stage 4를 통과해 ES2026 표준에 포함된 `Temporal` API는 month를 1부터 시작하도록 새로 설계됨(Node.js 26부터 네이티브 지원). 자세한 내용은 맨 아래 "예약" 섹션 참고.

---

# 날짜 비교 / 연산

```javascript
const start = new Date('2026-06-01');
const end   = new Date('2026-12-31');

start < end                       // true (직접 비교 가능)
start.getTime() < end.getTime()   // 명확한 숫자 비교

// 날짜 차이 (일) — "경과 시간" 기준
const diff = end.getTime() - start.getTime();
const days = Math.floor(diff / (1000 * 60 * 60 * 24));

// N일 후
new Date(date.getTime() + 7 * 24 * 60 * 60 * 1000)

// 다음 달
const nextMonth = new Date(date);
nextMonth.setMonth(date.getMonth() + 1);
```

```
⚠️ 위 days 계산(getTime() 차이 ÷ 86400000)은 "경과 시간(ms)" 기준임
   두 시각의 시:분:초가 다르면, 캘린더 날짜로는 같은 차이라도 결과가 다르게 나올 수 있음
   예) "어제 23:50" → "오늘 00:10" 은 20분 차이라서 days = 0 (그런데 캘린더상으론 "어제"임)
   → "오늘/어제/3일 전" 같은 캘린더 날짜 차이가 필요하면 ↓ differenceInCalendarDays 섹션 참고
```

---

# 캘린더 날짜 차이 — startOfDay / differenceInCalendarDays ⭐️⭐️

```
상황: "어제 작성한 글", "3일 전 알림" 처럼 UI에 보여줄 상대 날짜를 계산할 때
문제: 위 "경과 시간(ms)" 방식은 시:분:초까지 포함해서 빼기 때문에
      두 시각이 같은 날인지/다른 날인지가 아니라 "정확히 24시간이 지났는지"를 따짐
      → 캘린더 날짜 차이가 필요하면 시:분:초를 먼저 지워야 함
```

```typescript
// 1. 시:분:초를 0으로 만들어 "그 날짜의 자정"으로 정규화
function startOfDay(date: Date): Date {
  const d = new Date(date);
  d.setHours(0, 0, 0, 0);   // 로컬 시간대 기준 자정
  return d;
}

// 2. 자정끼리 빼면 "캘린더 날짜" 차이만 남음
function differenceInCalendarDays(later: Date, earlier: Date): number {
  const ms = startOfDay(later).getTime() - startOfDay(earlier).getTime();
  return Math.round(ms / 86_400_000);   // floor 대신 round ↓ 이유는 아래
}
```

```
⚠️ Math.floor 대신 Math.round 를 쓰는 이유 — DST(일광절약시간) 보정:
  setHours(0,0,0,0) 은 "로컬 시간대" 기준 자정으로 맞춤
  서머타임을 쓰는 지역(미국/유럽 등)은 1년에 두 번,
  하루가 23시간 또는 25시간이 되는 날이 있음
  → 자정끼리의 ms 차이가 86,400,000의 정확한 배수가 아닐 수 있음 (±3,600,000 오차)
  → Math.floor 면 하루가 통째로 누락될 수 있고, Math.round 면 가장 가까운 정수로 보정됨
  한국은 DST 가 없어서 체감 빈도는 낮지만, 범용 유틸이라면 round 가 더 안전함
```

## 더 큰 단위로 — startOfMonth / startOfYear ⭐️⭐️⭐️⭐️

```typescript
// startOfDay를 그대로 재사용해서 한 칸 더 0으로
function startOfMonth(date: Date): Date {
  const start = startOfDay(date);
  start.setDate(1);
  return start;
}

function startOfYear(date: Date): Date {
  const start = startOfMonth(date);
  start.setMonth(0);
  return start;
}
```

```
패턴: 더 큰 단위 함수가 그보다 작은 단위 함수를 그대로 불러서 "한 칸만 더" 0으로 만듦
  startOfYear → startOfMonth → startOfDay 순으로 점점 더 작은 단위에 의존함
→ setHours+setDate+setMonth를 매번 따로 다 적지 않고, 작은 함수를 쌓아서 큰 함수를 만드는 방식

이 함수들도 startOfDay와 마찬가지로 new Date(date)로 복사부터 시작해야 함
(Date가 mutable이라 원본을 직접 건드리면 안 되는 이유는 위 "이 규칙 때문에 생기는 버그" 참고)
```

## 기간 이동 — addDays / addMonths ⭐️⭐️⭐️⭐️

```typescript
function addDays(date: Date, days: number): Date {
  const result = new Date(date);
  result.setDate(result.getDate() + days);
  return result;
}

// 음수를 넘기면 "이전"이 됨 — 오늘 포함 최근 7일의 시작점
const sevenDaysAgo = addDays(startOfDay(new Date()), -6);
```

```
setDate에 음수나 그 달의 일수를 넘는 값을 넘기면, JS가 자동으로 "이전 달/다음 달"로 굴려서 처리해줌
  예: 6월 1일에서 setDate(1 - 3) → 자동으로 5월 날짜로 계산됨
→ 월말/월초 경계를 직접 계산할 필요 없이, 그냥 일(day) 단위로 더하고 빼면 Date가 알아서 처리함
```

```typescript
function addMonths(date: Date, months: number): Date {
  const result = new Date(date);
  result.setMonth(result.getMonth() + months);
  return result;
}
```

```
⚠️ 말일 근처에서 함정이 있음 — 1월 31일에 1개월을 더하면?
  setMonth가 "2월 31일"을 만들려고 시도하는데 2월은 28일(또는 29일)까지만 있어서
  자동으로 넘쳐서 3월 2일/3일처럼 의도와 다른 날짜가 되어버릴 수 있음
→ "그 달의 마지막 날"을 다루는 계산이 섞여있다면, addMonths 결과를 한 번 더 검증할 것
```

## 경과 시간(ms) 기준 vs 캘린더 날짜 기준

|기준|코드|"어제 23:50 작성 → 오늘 00:10 조회"|
|---|---|---|
|경과 시간(ms) 기준|`Math.floor((b.getTime()-a.getTime())/86400000)`|0일 (20분 차이라서)|
|캘린더 날짜 기준|`differenceInCalendarDays(b, a)`|1일 ("어제"로 표시되어야 자연스러움)|

```
어느 쪽이 "맞다"가 아니라 용도가 다름:
  마감일까지 정확히 몇 시간 남았는지 → 경과 시간(ms) 기준
  "오늘/어제/N일 전" 같은 캘린더 표시 → 캘린더 날짜 기준 (startOfDay 사용)
```

---

# 날짜만 다루고 싶을 때 — 정오(T12:00:00) 고정 트릭 ⭐️

```
상황: 캘린더에서 "방문 날짜"를 고른다 — 시간은 의미 없고 "날짜" 자체만 중요함
문제: 이 날짜를 그냥 new Date(iso) 로 다루면
      앞서 본 "핵심 규칙" 때문에 실행 환경(서버 UTC / 팀원 PC / 사용자 브라우저)에 따라
      날짜가 하루 밀릴 수 있음
```

```typescript
const toVisitDate = (iso: string) => new Date(`${iso.slice(0, 10)}T12:00:00`);

toVisitDate('2026-06-16')                  // '2026-06-16' → 'T12:00:00' 붙임
toVisitDate('2026-06-16T09:30:00.000Z')    // 시간 부분 버리고 날짜만 사용
```

```
이 함수가 하는 일, 한 줄씩:

  1. iso.slice(0, 10)
     → 시간 정보가 섞여 들어와도 앞 10글자(YYYY-MM-DD)만 취함

  2. `${...}T12:00:00`  (Z 도 오프셋도 없음)
     → "핵심 규칙" 에 의해 이제 "날짜-시간 형식" 이 되어 UTC 가 아니라
       "로컬 시간대" 로 해석됨 (날짜만 있을 때의 UTC 해석을 피하는 것)

  3. 시간을 굳이 12:00(정오)으로 고정하는 이유
     → 전 세계 시간대 오프셋은 -12 ~ +14 사이
     → 자정(00:00)을 기준으로 하면 오프셋이 음수 방향이면 바로 전날로 넘어갈 위험이 큼
     → 정오를 기준으로 하면 어느 쪽으로 변환되어도(±12시간 안팎)
       대부분의 실제 시간대에서 날짜가 절대 앞/뒤로 안 넘어감
       → "날짜"만 안전하게 보존하기 위한 안전지대로 정오를 고른 것
```

## 비교 — 언제 어떤 고정 방법을 쓰나

|목적|방법|예시|
|---|---|---|
|날짜 자체만 의미 있음 (시간 무관, 캘린더 선택값 등)|로컬 정오 고정|`${iso.slice(0,10)}T12:00:00`|
|특정 시간대 기준 "하루의 시작~끝" 범위 필요 (DB 쿼리)|명시적 오프셋 고정|`${date}T00:00:00+09:00`|
|서버 간 주고받는 절대 시각 (API 응답 등)|UTC 고정|`date.toISOString()` → `...Z`|

```
toVisitDate (정오 고정)        "이 날짜" 자체를 안전하게 보존하고 싶을 때
getTodayRangeKst (+09:00 고정)  "한국 기준 오늘 하루 전체" 범위가 필요할 때
→ 목적이 다름 — 아래 "시간대 지정" 섹션에서 +09:00 방식을 자세히 다룸
```

---

# toLocaleDateString — 지역화 포맷 ⭐️

```javascript
new Date().toLocaleDateString(locale, options)
//                             ↑ 언어  ↑ 표시 옵션
```

## options 값 종류 ⭐️

```
year:
  'numeric'   2026
  '2-digit'   26

month:
  'numeric'   6
  '2-digit'   06
  'long'      6월 / June
  'short'     6월 / Jun
  'narrow'    6 / J

day:
  'numeric'   16
  '2-digit'   16

weekday:
  'long'      화요일 / Tuesday
  'short'     화 / Tue
  'narrow'    화 / T
```

## 실전 패턴 ⭐️

```javascript
const date = new Date('2026-06-16T09:00:00');

// 전체 날짜 + 요일
date.toLocaleDateString('ko-KR', {
  year: 'numeric', month: 'long', day: 'numeric', weekday: 'long',
})
// → '2026년 6월 16일 화요일'

// 짧은 날짜
date.toLocaleDateString('ko-KR', {
  year: 'numeric', month: '2-digit', day: '2-digit',
})
// → '2026. 06. 16.'

// 월 + 일만
date.toLocaleDateString('ko-KR', { month: 'long', day: 'numeric' })
// → '6월 16일'
```

## locale: 'en-CA' — YYYY-MM-DD 포맷 트릭 ⭐️

```javascript
// 'en-CA' (캐나다 영어) 는 날짜 형식이 YYYY-MM-DD
new Date().toLocaleDateString('en-CA', { timeZone: 'Asia/Seoul' })
// → '2026-06-16'

// 비교: 다른 로케일은 형식이 다름
new Date().toLocaleDateString('ko-KR', { timeZone: 'Asia/Seoul' })  // '2026. 6. 16.'
new Date().toLocaleDateString('en-US', { timeZone: 'Asia/Seoul' })  // '6/16/2026'
```

```
왜 'en-CA' 를 쓰나:
  ISO 8601(YYYY-MM-DD) 형식을 별도 가공(padStart 등) 없이 바로 얻는 방법
  toISOString() 은 항상 UTC 기준이라 시간대 보정이 따로 필요함
  toLocaleDateString('en-CA', { timeZone }) 은
    지정한 시간대 기준으로 바로 YYYY-MM-DD 를 줌
  → 라이브러리(date-fns 의 format(date, 'yyyy-MM-dd') 등) 없이 같은 결과를 얻는 방법
```

---

# 시간대(timeZone) 명시 — 서버 환경에서 "오늘" 구하기 ⭐️

```
timeZone 옵션이 필요한 경우:
  Next.js 서버 컴포넌트 / Node.js 환경 → 서버는 보통 UTC 기준으로 실행
  → 'Asia/Seoul' 명시 안 하면 의도와 다른(UTC) 날짜/시간이 나올 수 있음

브라우저에서는: OS 시간대가 자동 적용 → timeZone 생략 가능
```

```javascript
// 서버에서 한국 시간 표시
export const formatToday = () =>
  new Date().toLocaleDateString('ko-KR', {
    timeZone: 'Asia/Seoul',   // ← 명시 필수
    year: 'numeric', month: 'long', weekday: 'long', day: 'numeric',
  });
// → '2026년 6월 16일 화요일'
```

```typescript
// 짧은 날짜 포맷 함수 — ISO 문자열을 받아서 'yyyy. MM. dd.' 형식으로
const formatDate = (iso: string) => {
  return new Date(iso).toLocaleDateString('ko-KR', {
    timeZone: 'Asia/Seoul',
    year: 'numeric', month: '2-digit', day: '2-digit',
  });
};

formatDate('2026-06-16T00:00:00.000Z')   // '2026. 06. 16.'
```

## 오늘 날짜 범위 구하기 — KST 기준 ⭐️

```typescript
const getTodayRangeKst = () => {
  // 1. 한국 시간 기준 오늘 날짜를 YYYY-MM-DD 로 구함 (위의 en-CA 트릭)
  const iso = new Date().toLocaleDateString('en-CA', { timeZone: 'Asia/Seoul' });

  return {
    date:  iso,                                    // '2026-06-16'
    start: new Date(`${iso}T00:00:00+09:00`),       // 오늘 00:00 KST
    end:   new Date(`${iso}T23:59:59.999+09:00`),   // 오늘 23:59:59.999 KST
  };
};

// 사용 — 오늘 등록된 데이터만 조회
const { start, end } = getTodayRangeKst();
await prisma.exhibition.findMany({ where: { createdAt: { gte: start, lte: end } } });
```

```
이 함수가 하는 일:

  1. en-CA 트릭으로 "한국 기준 오늘"을 YYYY-MM-DD 문자열로 정확히 구함
     → 서버가 어떤 시간대로 돌고 있어도 영향받지 않음

  2. `${iso}T00:00:00+09:00`
     → "핵심 규칙" 대로면 오프셋 없는 T00:00:00 은 로컬 시간대로 해석되어
       서버 환경에 따라 결과가 달라질 위험이 있음
     → +09:00 을 직접 명시해서 "항상 한국 시간 기준 00:00" 으로 고정

  3. `${iso}T23:59:59.999+09:00`
     → 오늘의 마지막 순간(밀리초까지) — 23:59:59 까지만 쓰면
       그 다음 밀리초(.000~.999) 구간이 범위에서 누락될 수 있음

언제 쓰나:
  "오늘 등록된 데이터" 같은 날짜 범위 필터
  서버가 UTC 로 도는 환경(Railway, Vercel 등)에서
  한국 기준 하루의 시작/끝을 정확히 구해야 할 때
```

---

# 상대 날짜 표시 패턴 — 오늘/어제/N일 전 ⭐️⭐️

```
어디서 쓰나: 피드/댓글/알림/Admin 리스트 등 "타임스탬프를 사람이 읽기 쉽게" 보여주는 모든 곳
공통 구조: 절대 날짜 포맷터 + 캘린더 날짜 차이(differenceInCalendarDays)를 합쳐서
          "절대값" 또는 "상대값 · 절대값"을 보여주는 패턴이 많이 쓰임
```

```typescript
// 절대 날짜 (예: 2026.06.25)
function formatDisplayDate(iso: string): string {
  const d = new Date(iso);
  const y = d.getFullYear();
  const m = String(d.getMonth() + 1).padStart(2, '0');
  const day = String(d.getDate()).padStart(2, '0');
  return `${y}.${m}.${day}`;
}

// 상대 날짜 — differenceInCalendarDays 위에 얹는 표시 로직
function formatRelativeDate(iso: string): string {
  const absolute = formatDisplayDate(iso);
  const days = differenceInCalendarDays(new Date(), new Date(iso));

  if (days >= 7) return absolute;             // 일주일 넘으면 절대값만
  if (days <= 0) return `오늘 · ${absolute}`;   // 음수 방지(미래 타임스탬프 포함)
  if (days === 1) return `어제 · ${absolute}`;
  return `${days}일 전 · ${absolute}`;
}
```

## 임계값 표 — days 값에 따른 출력

|`days` 값|출력 예시|
|---|---|
|`<= 0`|`오늘 · 2026.06.25`|
|`1`|`어제 · 2026.06.24`|
|`2 ~ 6`|`3일 전 · 2026.06.22`|
|`>= 7`|`2026.06.01` (절대값만)|

```
days <= 0 도 처리하는 이유:
  서버/클라이언트 시계 오차, 또는 작성 시각이 "지금"보다 살짝 미래인 경우를 대비
  음수가 나와도 "오늘"로 처리해서 "-1일 전" 같은 비정상 문구가 노출되지 않게 막음

임계값(7일)은 정해진 표준이 아니라 프로젝트마다 다르게 정함
→ 여기서는 "관용적으로 많이 쓰는 기준"만 표로 남김, 실제 값은 프로젝트별로 [[Project_Notes]] 참고
```

---

# 날짜 → 문자열 포맷

```javascript
const date = new Date('2026-06-08T14:30:00');

date.toISOString()         // "2026-06-08T14:30:00.000Z"
date.toLocaleDateString()  // "2026. 6. 8." (로케일 따라 다름)
date.toLocaleString()      // "2026. 6. 8. 오후 11:30:00"

// 직접 포맷 (padStart 로 자릿수 맞추기)
// [[JS_Primitive_Methods]] 참고
const year  = date.getFullYear();
const month = String(date.getMonth() + 1).padStart(2, '0');
const day   = String(date.getDate()).padStart(2, '0');
`${year}-${month}-${day}`   // "2026-06-08"
```

## toLocalDateKey / toLocalMonthKey — 키로 쓸 고정 형식 함수로 ⭐️⭐️⭐️⭐️

```typescript
function toLocalDateKey(date: Date): string {
  const y = date.getFullYear();
  const m = String(date.getMonth() + 1).padStart(2, '0');
  const d = String(date.getDate()).padStart(2, '0');
  return `${y}-${m}-${d}`;
}

function toLocalMonthKey(date: Date): string {
  const y = date.getFullYear();
  const m = String(date.getMonth() + 1).padStart(2, '0');
  return `${y}-${m}`;
}
```

```
바로 위 "직접 포맷" 코드와 똑같은 계산을, 매번 다시 풀어쓰지 않고 이름 붙여 재사용하기 위한 함수
→ Map의 키로 쓰거나(날짜별 통계 집계 등), 같은 날인지 비교하는 기준 문자열로 자주 씀
  ([[JS_Map_Set]]의 "같은 날짜 중복 제거" 패턴, [[NestJS_StatsBucket]]의 버킷 키가 정확히 이 함수들)

"로컬(Local)"이라는 이름이 붙은 이유는 toISOString()이 UTC 기준이라 시간대에 따라
날짜가 밀릴 수 있기 때문 — 자세한 건 위 "문자열 파싱 — 핵심 규칙" 참고
```

---

# Date.now() — 현재 밀리초

```javascript
const timestamp = Date.now();   // Date 객체 없이 현재 타임스탬프

// 실행 시간 측정
const start = Date.now();
// ... 작업 ...
console.log(`${Date.now() - start}ms`);
```

---

# parseYmd — YYYYMMDD 형식 파싱 ⭐️

```
공공 API 등에서 날짜를 "20260601" 처럼 하이픈 없이 보내는 경우
new Date("20260601") 은 ISO 8601 표준 형식이 아니라서 환경마다 결과가 다를 수 있음
→ 직접 파싱 함수 사용
```

```typescript
export function parseYmd(ymd: string | null | undefined): Date | null {
  if (!ymd || ymd.length !== 8) return null;

  const year  = parseInt(ymd.slice(0, 4), 10);
  const month = parseInt(ymd.slice(4, 6), 10) - 1;  // 월은 0부터
  const day   = parseInt(ymd.slice(6, 8), 10);

  const date = new Date(year, month, day);
  return isNaN(date.getTime()) ? null : date;
}

parseYmd('20260601')   // Date(2026-06-01)
parseYmd('99999999')   // null (유효하지 않은 날짜)
parseYmd(null)         // null
```

```
slice 로 쪼개는 이유:
  ymd.slice(0, 4) → "2026"  연도
  ymd.slice(4, 6) → "06"    월 (- 1 해서 0부터 시작)
  ymd.slice(6, 8) → "01"    일
```

---

# NestJS 에서 Date 사용

```typescript
// [[NestJS_DTO]] 참고
export class CreateExhibitionDto {
  @Type(() => Date)   // 문자열 → Date 객체 변환
  @IsDate()           // Date + getTime() 으로 유효성 검증
  startDate: Date;

  @Type(() => Date)
  @IsDate()
  endDate: Date;
}

// 서비스에서 날짜 비교
if (dto.startDate.getTime() > dto.endDate.getTime()) {
  throw new BadRequestException('시작일이 종료일보다 늦을 수 없습니다.');
}
```

---

# Next.js 환경에서 상대 날짜 표시 시 주의점 — Hydration Mismatch

```
문제 상황: formatRelativeDate 안의 new Date() (현재 시각)는
서버 렌더링 시점과 브라우저가 화면을 그리는 시점이 다름
→ 서버에서는 "어제"였는데 클라이언트에서 "오늘"로 바뀌는 등 텍스트가 일치하지 않아
React가 hydration mismatch 경고를 띄울 수 있음

지금은 "발생 가능한 이슈"로만 기록
실제 해결 패턴(useEffect 재계산 / 클라이언트 전용 렌더 등)은
Next.js 렌더링 모델(서버 vs 클라이언트 컴포넌트)을 더 깊게 다룰 때 [[NextJS_Concept]] 로 분리해서 정리
```

---

# 예약 — 추후 추가 (아직 안 씀) ⭐️

```
아래는 아직 정식으로 학습/정리하지 않은 항목 — 이름만 예약해두고, 배운 뒤 이 노트에 추가할 것
틀린 내용을 미리 적어두지 않기 위해 의도적으로 비워둠
```

|예약 항목|메모|
|---|---|
|`formatLongDate`|"2026년 6월 25일 화요일" 같은 풀 포맷 — `toLocaleDateString` 옵션 조합으로 구현 예정|
|`formatTodayLine`|"오늘" 한 줄 요약 표시용 — `formatRelativeDate`와의 관계 정리 필요|
|date-fns|`differenceInCalendarDays`, `formatDistanceToNow` 등 위에서 직접 구현한 함수들과 1:1 대응되는 라이브러리 — 학습 후 "직접 구현 vs date-fns" 비교 섹션 추가 예정|
|`Temporal`|JS Date를 대체하는 신규 네이티브 API. 2026년 3월 TC39 Stage 4 통과 → ES2026 표준 포함. Node.js 26(2026-05), Chrome 144, Firefox 139부터 네이티브 지원. month 1-indexed / 불변 객체 / 시간대 내장 — 이 노트의 트릭 상당수를 대체할 수 있어 date-fns보다 먼저 봐도 좋음|

---

# 한눈에

|메서드 / 패턴|역할|
|---|---|
|`new Date()`|현재 날짜|
|`new Date('2026-06-08')`|문자열로 생성 (날짜만 → UTC 해석 ⚠️)|
|`date.getTime()`|밀리초 타임스탬프|
|`isNaN(date.getTime())`|유효하지 않은 날짜 검증 ⭐️|
|`date.getFullYear()` / `getMonth()+1` / `getDate()`|연 / 월 / 일|
|`date.getDay()`|요일 (0=일 ~ 6=토)|
|`Date.now()`|현재 밀리초|
|`date.toISOString()`|ISO 8601 문자열 (UTC, `Z`)|
|`toLocaleDateString('ko-KR', options)`|한국어 포맷 ⭐️|
|`toLocaleDateString('en-CA', { timeZone })`|YYYY-MM-DD 문자열 트릭|
|`` `${date}T12:00:00` ``|날짜만 보존 — 로컬 정오 고정 ⭐️|
|`` `${date}T00:00:00+09:00` ``|특정 시간대 기준 시각 고정|
|`startOfDay(date)`|시:분:초를 0으로 — 날짜만 비교하기 위한 정규화 ⭐️|
|`startOfMonth(date)` / `startOfYear(date)`|startOfDay를 재사용해 더 큰 단위의 시작점 구하기|
|`addDays(date, n)` / `addMonths(date, n)`|n일/n개월 전후 — 음수면 "전", 월 경계는 자동으로 굴러감|
|`toLocalDateKey(date)` / `toLocalMonthKey(date)`|`YYYY-MM-DD`/`YYYY-MM` 키 문자열 — Map 키·중복 체크 기준으로 자주 씀|
|`differenceInCalendarDays(b, a)`|캘린더 날짜 차이 (ms 경과 시간 기준 아님) ⭐️|
|`formatRelativeDate(iso)`|오늘/어제/N일 전 표시 — 캘린더 날짜 차이 기반|

```
핵심 규칙 한 번 더:
  날짜만 있는 문자열   → UTC 로 해석
  시간 있고 오프셋 없음 → 로컬 시간대로 해석
  오프셋 명시          → 그 오프셋으로 해석 (환경 무관, 항상 동일)

  캘린더 날짜 차이 ≠ ms 경과 시간
  "오늘/어제/N일 전" 표시는 항상 startOfDay 로 시:분:초를 지운 뒤 뺄 것

options 핵심:
  'numeric' → 숫자 그대로 / '2-digit' → 두 자리 / 'long' → 전체 이름 / 'short' → 축약

timeZone: 'Asia/Seoul' → 서버 환경에서 한국 시간 다룰 때 필수
```