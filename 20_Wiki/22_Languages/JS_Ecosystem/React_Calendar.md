---
aliases: [달력 그리드, 캘린더 UI, firstDay, getCalendarDays, lastDate]
tags: [React, JavaScript]
relations:
  - "[[00_JS_Ecosystem_HomePage]]"
  - "[[JS_Array_Methods]]"
  - "[[JS_Date]]"
  - "[[NestJS_Prisma_Patterns]]"
  - "[[React_useMemo_useCallback]]"
---
# React_Calendar — 달력 그리드 빌드 패턴

> [!info]
> `new Date(year, month - 1, 1).getDay()` 로 1일의 요일을 구하고,
> `new Date(year, month, 0).getDate()` 로 마지막 날짜를 구해 달력 배열을 만드는 패턴.
> 날짜 범위 API 쿼리 → [[JS_Date]] | Prisma 날짜 범위 쿼리 → [[NestJS_Prisma_Patterns]]

---

# 달력 그리드 만드는 원리 ⭐️⭐️⭐️⭐️

## 목표

```txt
7 × N 달력 그리드:
  일  월  화  수  목  금  토
  [null][null][1][2][3][4][5]   ← 2025년 1월: 1일이 수요일(3)이므로 앞에 null 3개
  [6][7][8]...
```

## new Date(year, month - 1, 1).getDay() — 1일의 요일

```typescript
const firstDay = new Date(year, month - 1, 1).getDay();
// getDay() → 0=일, 1=월, 2=화, 3=수, 4=목, 5=금, 6=토
```

```txt
month - 1 이유:
  JavaScript Date의 month는 0-indexed
  → 1월 = 0, 12월 = 11
  → 사용자에게 "month=1(1월)"로 받으면 Date에는 month-1 넘겨야 함

예시:
  new Date(2025, 0, 1).getDay()  → 3 (수요일)
  = 2025년 1월 1일은 수요일
  → 앞에 null 3개 (일·월·화 자리)를 채워야 달력이 맞음
```

## new Date(year, month, 0).getDate() — 마지막 날짜

```typescript
const lastDate = new Date(year, month, 0).getDate();
// "month월의 0번째 날" = 전달의 마지막 날 = 이번 달의 마지막 날
```

```txt
트릭: "0번째 날"을 이용한 월 말일 계산

  new Date(2025, 1, 0)
  = 2025년 2월의 0번째 날
  = 2025년 1월의 마지막 날
  = 2025-01-31 → .getDate() = 31

  new Date(2025, 2, 0)
  = 2025년 3월의 0번째 날
  = 2025년 2월의 마지막 날
  = 2025-02-28 → .getDate() = 28 (윤년이면 29)

  month (0-indexed 아닌 사용자 값 그대로) 넘기면 다음 달의 0번째 날 = 이번 달 마지막
  → month - 1을 하지 않는 것이 포인트
```

## 완성 — getCalendarDays

```typescript
const WEEKDAYS = ['일', '월', '화', '수', '목', '금', '토'];

function getCalendarDays(year: number, month: number): (number | null)[] {
  // 1일이 무슨 요일인가? (0=일 ~ 6=토)
  const firstDay = new Date(year, month - 1, 1).getDay();

  // 이번 달 마지막 날짜는 몇 일인가?
  const lastDate = new Date(year, month, 0).getDate();

  return [
    // 앞 빈 칸: 1일 전까지 null로 채움
    ...Array.from({ length: firstDay }, () => null),
    // 실제 날짜: 1 ~ lastDate
    ...Array.from({ length: lastDate }, (_, i) => i + 1),
  ];
}

// 결과 예시 (2025년 1월, 1일=수요일, 31일까지)
// [null, null, null, 1, 2, 3, 4, 5, 6, 7, ..., 31]
//   일     월     화  수  목  금  토  일  월  화
```

---

# Array.from 패턴 ⭐️⭐️⭐️

```typescript
// null N개 배열
Array.from({ length: N }, () => null)
// [null, null, null, ...]

// 1 ~ N 숫자 배열
Array.from({ length: N }, (_, i) => i + 1)
// [1, 2, 3, ..., N]
```

```txt
Array.from({ length: N }, mapFn):
  { length: N } — "길이가 N인 배열처럼 생긴 객체"
  mapFn(value, index) — 각 자리를 채울 값
  → value는 undefined (빈 슬롯), index로 숫자 생성

  (_, i) => i + 1:
    _  → value 무시 (사용 안 하면 관례상 _ 로 표기)
    i  → 0-based index → +1로 1-based 날짜

Array(N).fill(null) 대신 Array.from 쓰는 이유:
  fill은 같은 참조를 채움 (원시값은 상관없음)
  mapFn으로 각 요소를 다르게 채울 수 있어 더 유연
```

---

# 달력 그리드 렌더링 ⭐️⭐️⭐️

## 날짜 key 생성

```typescript
function getDateKey(year: number, month: number, day: number): string {
  return `${year}-${String(month).padStart(2, '0')}-${String(day).padStart(2, '0')}`;
  // "2025-01-05"
}
```

## 컴포넌트 렌더링

```tsx
const days = useMemo(
  () => getCalendarDays(year, month),
  [year, month],   // year·month가 바뀔 때만 재계산
);

return (
  <div className="grid grid-cols-7">
    {/* 요일 헤더 */}
    {WEEKDAYS.map((d) => (
      <div key={d} className="text-center font-bold">{d}</div>
    ))}

    {/* 날짜 칸 */}
    {days.map((day, idx) => {
      if (day === null) {
        return <div key={`empty-${idx}`} />;   // 빈 칸
      }

      const dateKey = getDateKey(year, month, day);
      const movies  = moviesByDate.get(dateKey) ?? [];

      return (
        <div
          key={dateKey}
          onClick={() => setSelectedDate(dateKey)}
          className="p-1 cursor-pointer hover:bg-gray-100"
        >
          <span>{day}</span>
          {movies.map((m) => (
            <div key={m.tmdbId} className="text-xs truncate">{m.movie.title}</div>
          ))}
        </div>
      );
    })}
  </div>
);
```

## moviesByDate — Map으로 O(1) 날짜 조회

```typescript
const moviesByDate = useMemo(() => {
  const grouped = new Map<string, UserMovieCalendarItem[]>();

  for (const item of calendar?.items ?? []) {
    const movies = grouped.get(item.date) ?? [];  // 없으면 빈 배열
    movies.push(item);
    grouped.set(item.date, movies);
  }

  return grouped;
}, [calendar]);
```

```txt
Map을 쓰는 이유:
  달력 그리드 렌더링 시 날짜마다 해당 항목을 조회해야 함

  배열로 하면:
    const movies = items.filter(i => i.date === dateKey)
    → 날짜 칸마다 items 전체를 순회 → O(n × 날짜 수)

  Map으로 하면:
    const movies = moviesByDate.get(dateKey) ?? []
    → O(1) 조회 — 달력 렌더 성능 ↑

grouped.get(item.date) ?? []:
  Map에 해당 키가 없으면 undefined → ?? [] 로 빈 배열 기본값
  → push 후 set으로 다시 저장

useMemo deps = [calendar]:
  calendar 데이터가 바뀔 때만 Map 재생성
  → year/month 변경으로 calendar가 바뀌면 자동 갱신됨
```

---

# 월 이동 패턴 ⭐️⭐️⭐️

```typescript
const now = new Date();
const [year, setYear]   = useState(now.getFullYear());
const [month, setMonth] = useState(now.getMonth() + 1);  // 0-indexed → +1

function prevMonth() {
  if (month === 1) {
    setYear((y) => y - 1);
    setMonth(12);
  } else {
    setMonth((m) => m - 1);
  }
}

function nextMonth() {
  if (month === 12) {
    setYear((y) => y + 1);
    setMonth(1);
  } else {
    setMonth((m) => m + 1);
  }
}

// ✅ KST-safe 버전 — TZ 무관하게 항상 KST 기준 year/month 사용
function moveKstMonth(year: number, month: number, amount: number) {
  // Date.UTC(year, month - 1 + amount, 1, 12) → 정오(UTC 12:00) 기준 생성
  // UTC 12:00 = KST 21:00 → 어느 환경에서도 KST 날짜가 뒤집히지 않음
  const movedDate = new Date(Date.UTC(year, month - 1 + amount, 1, 12));
  return getKstYearMonth(movedDate);  // → [[JS_Intl]] getKstYearMonth 참고
}

function moveMonth(amount: number) {
  const next = moveKstMonth(year, month, amount);
  setYear(next.year);
  setMonth(next.month);
}
```

```txt
now.getMonth() + 1:
  JS Date.getMonth() = 0-indexed (1월=0, 12월=11)
  → +1 해서 사람이 쓰는 1~12로 저장

KST-safe moveKstMonth:
  단순 if/else로 1월↔12월 경계 처리도 가능하지만
  Date.UTC(year, month - 1 + amount, 1, 12)를 쓰면:
    ① 경계(1월/12월) 처리를 JS Date가 자동으로 해줌
       → month - 1 + (-1) = -1 → Date가 자동으로 전년 12월로 변환
    ② 시각을 UTC 12:00(정오)로 지정 → KST 21:00 → 날짜 뒤집힘 없음
       UTC 00:00을 KST로 읽으면 +9h = 09:00 → 같은 날이지만
       UTC에서 날짜 경계 근처(자정)에서 생성하면 KST 날짜가 달라질 수 있음
    ③ getKstYearMonth()로 KST 기준 year·month 추출
       → 서버(UTC TZ)에서 실행해도 항상 KST 기준 ([[JS_Intl]] 참고)
```

---

# 한눈에

```txt
getCalendarDays(year, month):
  new Date(year, month-1, 1).getDay()   → 1일의 요일 (0=일 ~ 6=토)
  new Date(year, month, 0).getDate()    → 마지막 날짜 (0번째 날 트릭)
  Array.from({length:firstDay}, () => null)     → 앞 빈 칸
  Array.from({length:lastDate}, (_, i) => i+1)  → 1~lastDate

Map 그루핑 + useMemo:
  items 배열 → Map<dateKey, items[]>로 변환 → O(1) 날짜 조회
  useMemo로 calendar 데이터 바뀔 때만 재계산

Array.from 패턴:
  { length: N }       → 길이 N인 유사 배열
  (_, i) => i + 1     → 1-based 숫자 생성
```
