---
aliases:
  - 날짜 선택
  - 달력
  - date-fns
  - react-day-picker
tags:
  - React
  - NextJS
related:
  - "[[00_JS_Ecosystem_HomePage]]"
  - "[[JS_Date]]"
---
# React_DatePicker — react-day-picker & date-fns

# 한 줄 요약

```txt
date-fns         날짜 계산 / 포맷 / 비교 유틸리티 함수 모음
react-day-picker 달력 UI 컴포넌트 (날짜 선택 / 범위 선택 / 커스텀 표시)

modifiers prop    날짜에 "표식"을 붙이는 기능 — 방문한 날 표시 등에 사용 ⭐️
```

---

---

# 설치 + 버전 확인 ⭐️

```bash
pnpm add react-day-picker date-fns
```

```txt
⚠️ react-day-picker 는 2026년에 v9 → v10 으로 메이저 업데이트됨
   v10 부터 패키지명이 @daypicker/react 로 변경됨 (react-day-picker 이름도 호환용으로 유지)
   classNames 의 키 이름이 다수 변경됨 (day_selected → selected 등, 아래에서 다룸)
   fromDate/toDate 같은 일부 prop 이 startMonth/endMonth 로 통합됨

   → package.json 에서 설치된 버전을 먼저 확인하고, 아래 예시는 v10 기준으로 작성함
```

---

---

# date-fns — 날짜 유틸리티 ⭐️

```txt
날짜 계산을 직접 하면 복잡: 월말 / 윤년 / 시간대 처리 어려움
date-fns 가 해결: 함수형 API / tree-shaking / 타입 지원
```

## 자주 쓰는 함수

```typescript
import {
  format, parseISO, isAfter, isBefore, isEqual,
  addDays, subDays, differenceInDays,
  startOfMonth, endOfMonth, isWithinInterval,
} from 'date-fns';
import { ko } from 'date-fns/locale';
```

| 함수                                       | 역할                       |
| ---------------------------------------- | ------------------------ |
| `format(date, fmt)`                      | Date → 지정한 형식의 문자열로 변환   |
| `parseISO(str)`                          | ISO 8601 문자열 → Date 로 변환 |
| `isAfter(a, b)`                          | a 가 b 보다 나중인지            |
| `isBefore(a, b)`                         | a 가 b 보다 이전인지            |
| `isEqual(a, b)`                          | 두 날짜가 정확히 같은지            |
| `addDays(date, n)`                       | n일 더한 날짜                 |
| `subDays(date, n)`                       | n일 뺀 날짜                  |
| `differenceInDays(a, b)`                 | a - b 의 날짜 차이 (일 단위)     |
| `startOfMonth(date)`                     | 해당 월의 1일 00:00           |
| `endOfMonth(date)`                       | 해당 월의 마지막 날 23:59:59.999 |
| `isWithinInterval(date, { start, end })` | date 가 start~end 사이인지    |

## format — 날짜 포맷 ⭐️

```typescript
const date = new Date('2026-06-16');

format(date, 'yyyy-MM-dd')                      // '2026-06-16'
format(date, 'yyyy년 MM월 dd일')                // '2026년 06월 16일'
format(date, 'M월 d일 (EEE)', { locale: ko })   // '6월 16일 (화)'
format(date, 'PPP', { locale: ko })             // '2026년 6월 16일'

const d = parseISO('2026-06-16T00:00:00.000Z');
format(d, 'yyyy-MM-dd')   // '2026-06-16'
```

```txt
포맷 토큰:
  yyyy 연도(4자리) / MM 월(2자리) / dd 일(2자리)
  M 월(1~2자리) / d 일(1~2자리)
  EEE 요일 축약 / EEEE 요일 전체
  HH 시간(24시) / mm 분 / ss 초
```

## 날짜 비교 / 계산

```typescript
const today  = new Date();
const target = new Date('2026-12-31');

isAfter(target, today)              // true (target 이 나중)
isBefore(target, today)             // false
isEqual(today, new Date())          // true

addDays(today, 7)                   // 7일 후
subDays(today, 7)                   // 7일 전
differenceInDays(target, today)     // 오늘부터 target 까지 남은 일수

isWithinInterval(today, {
  start: new Date('2026-06-01'),
  end:   new Date('2026-06-30'),
})  // 전시 기간 내인지 확인
```

## 전시 상태 계산 패턴

```typescript
import { isAfter, isBefore, parseISO } from 'date-fns';

type ExhibitionStatus = 'ongoing' | 'upcoming' | 'ended';

function getExhibitionStatus(startDate: string, endDate: string): ExhibitionStatus {
  const today = new Date();
  const start = parseISO(startDate);
  const end   = parseISO(endDate);

  if (isBefore(today, start)) return 'upcoming';
  if (isAfter(today, end))    return 'ended';
  return 'ongoing';
}
```

---

---

# react-day-picker — 달력 UI ⭐️

```txt
달력 UI 컴포넌트
  날짜 단일 선택 / 범위 선택 / 다중 선택 지원
  modifiers 로 "선택"과 별개인 임의의 표식도 가능 (방문 표시 등)
  Tailwind 로 스타일 커스터마이징
  date-fns 와 함께 사용
```

## DayPicker 주요 Props ⭐️

```tsx
<DayPicker
  mode="single"          // 단일 선택
  mode="range"           // 범위 선택
  mode="multiple"        // 여러 날짜 선택

  selected={date}        // 선택된 날짜 — 넘기면 onSelect 도 함께 필요
  onSelect={setDate}     // 선택 변경 핸들러

  defaultMonth={today}   // 처음 보여줄 월 (기본: 현재 월)
  numberOfMonths={2}     // 달력 몇 개 나란히 표시

  locale={ko}            // 한국어 (date-fns/locale)
  weekStartsOn={0}       // 0=일요일 / 1=월요일 시작

  showOutsideDays        // 이전·다음 달 날짜 표시 여부
  fixedWeeks             // 항상 6주 표시 (달마다 높이 고정)
  showWeekNumber         // 주 번호 표시

  disabled={[{ before: new Date() }]}  // 오늘 이전 비활성화
  startMonth={new Date()}              // 선택 가능한 시작 월 (v10, 구 fromDate/fromMonth)
  endMonth={new Date('2027-12-31')}    // 선택 가능한 종료 월 (v10, 구 toDate/toMonth)

  modifiers={{ visited: visitedDates }}        // 커스텀 표식 ⭐️ (아래 섹션에서 다룸)
  modifiersClassNames={{ visited: 'has-dot' }} // 표식별 className

  classNames={{ ... }}      // 내부 요소별 className (v10 키 이름, 아래에서 다룸)
/>
```

```txt
⚠️ disabled 는 v9/v10 에서도 그대로 유지되지만,
   fromDate/toDate/fromMonth/toMonth 는 v9 에서 deprecated → v10 에서 완전히 제거됨
   startMonth / endMonth 로 교체해야 함
```

## weekStartsOn ⭐️

```tsx
weekStartsOn={0}   // 일 월 화 수 목 금 토 (한국 달력 기본)
weekStartsOn={1}   // 월 화 수 목 금 토 일 (ISO 8601 표준 / 유럽)
```

```txt
locale={ko} 만 설정하면 월요일 시작이 기본
→ 한국 스타일 일요일 시작 원하면 weekStartsOn={0} 명시
```

## showOutsideDays ⭐️

```tsx
<DayPicker showOutsideDays />   // 6월 달력에 5월말/7월초 날짜도 흐리게 표시
<DayPicker />                   // 표시 안 함 (기본값) — 빈 칸으로 남음
```

## captionLayout — 월/연도 드롭다운 네비게이션 ⭐️

```tsx
<DayPicker
  captionLayout="dropdown"          // 월 + 연도 둘 다 드롭다운
  // captionLayout="dropdown-months"  // 월만 드롭다운
  // captionLayout="dropdown-years"   // 연도만 드롭다운
  // captionLayout="label"            // 기본값 — "2026년 6월" 텍스트만, 클릭 이동 불가
  startMonth={new Date(2020, 0)}
  endMonth={new Date(2027, 11)}
/>
```

```txt
값 4가지:
  label            기본값 — 상단에 "2026년 6월" 텍스트만 표시 (이전/다음 버튼으로만 이동)
  dropdown         월 / 연도 둘 다 <select> 드롭다운으로 바로 점프 가능
  dropdown-months  월만 드롭다운, 연도는 텍스트
  dropdown-years   연도만 드롭다운, 월은 텍스트

⚠️ startMonth / endMonth 를 명시 안 하면:
  드롭다운 범위가 기본으로 "최근 100년 전 ~ 올해 연말" 로 자동 설정됨
  → 생일 선택처럼 과거로 멀리 가야 하면 그대로 두면 되지만
    전시 예약 캘린더처럼 가까운 미래만 필요하면 startMonth/endMonth 로 범위를 좁혀야
    드롭다운에 의미 없는 연도가 99개씩 뜨는 걸 막을 수 있음

reverseYears={true}   연도 드롭다운 순서를 뒤집음 (최신 연도가 위로 오게 하고 싶을 때 등)
```

```txt
언제 쓰나:
  생일 선택 / 회원가입처럼 "연도를 멀리, 빠르게" 옮겨야 하는 입력  → dropdown 또는 dropdown-years
  이번 달 근처만 보면 되는 일반 일정/예약 캘린더               → 기본값(label) 그대로 두는 게 깔끔
```

## components prop — 완전 커스텀 캡션 만들기 ⭐️

```txt
captionLayout="dropdown" 는 DayPicker 가 이미 만들어둔 드롭다운 UI 를 그대로 씀
→ 디자인을 내 마음대로 바꾸기는 어려움 (라이브러리가 정해준 모양)

직접 디자인한 드롭다운(바깥 클릭하면 닫히게, 커스텀 스타일, 추가 검증 로직 등)이
필요하면 → components prop 으로 DayPicker 내부의 특정 영역을 내가 만든 컴포넌트로
통째로 갈아끼움
```

```tsx
<DayPicker
  mode="single"
  required={false}
  hideNavigation   // 기본 이전/다음(<, >) 버튼 숨김 — 내가 만든 캡션이 그 역할을 대신함
  components={{
    MonthCaption: (props) => (
      <VisitMonthCaption {...props} bounds={bounds} onMonthChange={handleMonthChange} />
    ),
  }}
/>
```

```txt
components 가 하는 일:
  DayPicker 내부의 각 부분(월 grid, 이전/다음 버튼, 캡션 등)이 "슬롯" 이름으로 정해져 있고
  components={{ 슬롯이름: 내컴포넌트 }} 형태로 그 부분만 교체할 수 있음
  → MonthCaption 슬롯 = 매달 위에 보이는 "2026년 6월" 영역

hideNavigation 을 같이 쓰는 이유:
  기본 이전(<)/다음(>) 버튼은 DayPicker 가 자체적으로 그려줌
  내가 만든 캡션 안에서 연/월을 직접 눌러 옮기게 할 거라면 기본 버튼은 필요 없으니 숨김
  ⚠️ disableNavigation 과는 다름 — hideNavigation 은 "안 보이게만", disableNavigation 은 "기능 자체를 끔"
```

```typescript
// DayPicker 가 MonthCaption 컴포넌트에 기본으로 넘겨주는 props
type MonthCaptionProps = {
  calendarMonth: { date: Date /* ... */ };
  // 그 외 displayIndex 등 정보도 함께 넘어옴
};

// 직접 만든 캡션이 추가로 필요한 정보(bounds, onMonthChange)는 타입을 합쳐서 받음
type VisitMonthCaptionProps = MonthCaptionProps & {
  bounds: CalendarBounds;                // 직접 정의한 타입 — 선택 가능한 연/월 범위
  onMonthChange: (month: Date) => void;  // 연/월 선택 시 실행할 콜백
};
```

```bash
⚠️ bounds, onMonthChange 는 DayPicker 가 모르는 값이라 자동으로 안 넘어옴
   → 위 예시처럼 화살표 함수로 한 번 감싸서 직접 주입해줘야 함
   (DayPicker 가 주는 건 calendarMonth 뿐 — 나머지는 전부 내가 끼워넣는 것)

캡션 내부 로직은 DayPicker 고유 API 가 아니라 일반 React 로직임:
  바깥 클릭하면 드롭다운 닫기 → useEffect + document.addEventListener('mousedown', ...)
                                 (# [[React_useEffect]] 의 이벤트 리스너 클린업 패턴과 동일)
  연도 바꿨는데 현재 월이 범위 밖 → 그 연도에서 선택 가능한 첫 달을 찾아 대신 선택
```

```txt
언제 captionLayout 대신 components 를 쓰나:
  captionLayout="dropdown"   라이브러리 기본 드롭다운으로 충분할 때 (코드 적고 빠름)
  components.MonthCaption    디자인/동작을 완전히 직접 통제해야 할 때
```

## 읽기 전용 달력 패턴 ⭐️

```tsx
const today = useMemo(() => new Date(), []);

<DayPicker
  mode="single"
  selected={today}
  onSelect={() => {}}    // 빈 함수 → 선택 이벤트 무시
  defaultMonth={today}
  locale={ko}
  weekStartsOn={0}
  showOutsideDays
  className="home-calendar"
  disabled               // 모든 날짜 클릭 비활성화
/>
```

```bash
onSelect={() => {}} 쓰는 이유:
  mode="single" + selected 를 쓰려면 onSelect 도 필요 (v10 부터 더 엄격하게 요구됨)
  실제로 날짜 선택은 안 하게 하려면 빈 함수 전달 → 오늘 날짜 하이라이트만 보여주는 표시용 달력

useMemo(() => new Date(), []):
  렌더링마다 new Date() 가 새로 생성되면 selected prop 의 참조가 매번 바뀜
#  → 자세한 이유는 [[React_useMemo_useCallback]] 의 "원시값 vs 참조값" 참고
```

---

---

# modifiers — 날짜 커스텀 표시 ⭐️⭐️

```txt
selected / disabled 는 DayPicker 가 이미 알고 있는 "내장 표식" 이고,
modifiers 는 내가 직접 "이런 의미의 날짜다" 라는 표식을 새로 만드는 prop 임

예: "이 날짜들은 사용자가 전시를 관람한 날이다" → visited 라는 이름의 표식을 직접 만듦
```

## 동작 흐름 — 데이터 → 클래스 → 디자인 ⭐️

```txt
modifiers 와 modifiersClassNames 는 한 세트로 움직이는 두 개의 스위치임:

  modifiers            "어떤 날이 관람일인지" 데이터를 달력에 알려줌
  modifiersClassNames  그 날의 HTML 요소에 CSS 클래스 이름을 붙여줌
  CSS                  그 클래스를 보고 점 / 색 같은 실제 스타일을 입힘

  데이터 → 클래스 → 디자인  순서로 책임이 나뉘어 있음
```

```tsx
modifiers={{ visited: visitedDates }}
modifiersClassNames={{ visited: 'visit-calendar-has-visit' }}
```

```css
/* 위 className 이 붙은 날짜 칸에 실제 스타일을 입히는 쪽 — React 코드가 아니라 CSS 의 책임 */
.visit-calendar-has-visit::after {
  content: '';
  display: block;
  width: 6px; height: 6px;
  border-radius: 50%;
  background: #3b82f6;
  margin: 2px auto 0;
}
```

```txt
3단계로 나누는 이유:
  modifiers 는 "어떤 날짜인가"(데이터)만 책임지고
  점을 찍을지 / 배경색을 바꿀지 / 밑줄을 그을지(디자인)는 CSS 가 책임짐
  → 같은 visited 데이터를 그대로 두고 className(=디자인)만 바꿔서 끼워넣을 수 있음
  → 클래스 단계를 생략하고 바로 인라인 스타일로 가고 싶으면 modifiersStyles 사용
    (아래 "modifiersStyles" 섹션 참고)
```

## modifiers 가 받는 값 — 왜 Date[] 가 필요한가 ⭐️

```tsx
<DayPicker
  modifiers={{ visited: visitedDates }}
  //           ↑ 표식 이름   ↑ Matcher: Date[] / Date / DateRange / (date) => boolean 등
/>
```

```txt
modifiers 의 값(Matcher)으로 허용되는 타입은 정해져 있음:
  Date            특정 하루
  Date[]          여러 날짜 (배열 안의 각 항목이 "이 날짜와 같은 날"인지 비교됨)
  { before, after } 같은 범위 객체
  (date: Date) => boolean   직접 판단하는 함수

→ 문자열 배열('2026-06-18' 같은)은 Matcher 타입에 없음
  DayPicker 내부적으로 날짜를 비교할 때 Date 의 연/월/일 값을 직접 비교하기 때문에
  반드시 Date 인스턴스로 변환해서 넘겨야 함
  → 그래서 ISO 문자열(visitedAt)을 그대로 못 쓰고 Date 로 변환하는 과정이 필요한 것
```

## 실전 — 방문한 날짜 표시 (visitedDates 패턴) ⭐️

```typescript
const visitedDates = useMemo(() => {
  const seen = new Set<string>();
  const dates: Date[] = [];

  for (const visit of visits) {
    const key = visit.visitedAt.slice(0, 10);
    if (seen.has(key)) continue;
    seen.add(key);
    dates.push(toVisitDate(visit.visitedAt));
  }
  return dates;
}, [visits]);
```

```tsx
<DayPicker
  mode="single"
  modifiers={{ visited: visitedDates }}
  modifiersClassNames={{ visited: 'rdp-visited' }}
  locale={ko}
/>
```

```bash
이 코드를 이해하려면 3가지 배경 지식이 필요함:

  1. "왜 Date[] 가 필요한가" → 위 섹션 (modifiers 의 Matcher 타입)

  2. "왜 toVisitDate 로 변환하나" # → [[JS_Date]] 의 "정오(T12:00:00) 고정 트릭"
     visitedAt('2026-06-18T03:00:00.000Z')을 그대로 new Date() 하면
     실행 환경에 따라 날짜가 하루 밀릴 수 있어서, 날짜 부분만 떼어 안전하게 고정함

  3. "왜 Set<string> 으로 먼저 중복 제거하나" # → [[JS_WeakMap_Set]] 의 "Set 은 참조로 비교"
     같은 날 전시를 2건 봤으면 visits 에 같은 날짜가 2번 들어있음
     → Date 객체끼리는 내용이 같아도 참조가 달라 Set 으로 직접 거를 수 없음
     → 그래서 비교 가능한 "문자열(YYYY-MM-DD)" 을 먼저 만들어 그걸로 중복을 거르고
       실제로 화면에 쓸 Date 는 그 뒤에 따로 만들어서 넣음

  결과: dates 에는 "전시를 본 날짜"가 하루당 정확히 하나씩만 들어있는 Date[] 가 됨
  → 이 배열을 그대로 modifiers.visited 에 넘기면 DayPicker 가 해당 날짜 칸에
    'rdp-visited' className 을 자동으로 붙여줌 (CSS 로 점 표시 등 스타일링)
```

## modifiersStyles — className 없이 인라인 스타일로

```tsx
<DayPicker
  modifiers={{ visited: visitedDates }}
  modifiersStyles={{
    visited: { fontWeight: 'bold', textDecoration: 'underline' },
  }}
/>
```

## (참고) DayPicker 자체의 시간대 보정 기능 — timeZone / noonSafe ⭐️

```txt
react-day-picker 는 실험적(experimental)으로 timeZone, noonSafe prop 을 제공함

<DayPicker timeZone="Asia/Seoul" noonSafe />

timeZone   지정한 시간대 기준으로 달력 계산
noonSafe   초 단위 옛 시간대 오프셋 때문에 날짜가 자정 경계에서 밀리는 걸 방지
           (toVisitDate 의 "정오 고정" 과 같은 목적을 라이브러리 차원에서 제공)

아직 실험적 기능이라 프로젝트에 바로 적용하기보다는,
"라이브러리도 같은 문제를 정오 고정으로 해결한다" 는 것만 참고로 알아두면 됨
```

---

---

# 날짜 범위 선택 ⭐️

```tsx
import { DateRange, DayPicker } from 'react-day-picker';

export default function RangeDatePicker() {
  const [range, setRange] = useState<DateRange | undefined>();

  return (
    <DayPicker
      mode="range"
      selected={range}
      onSelect={setRange}
      locale={ko}
      numberOfMonths={2}
    />
  );
}
```

```typescript
type DateRange = {
  from: Date | undefined;
  to?:  Date | undefined;
};

if (range?.from && range?.to) {
  const days = differenceInDays(range.to, range.from);
  console.log(`${days}일 선택됨`);
}
```

---

---

# 날짜 제한 — 비활성화 ⭐️

```tsx
<DayPicker
  mode="single"
  selected={selected}
  onSelect={setSelected}
  disabled={[
    new Date('2026-12-25'),
    { before: new Date() },
    { after: new Date('2026-12-31') },
  ]}
  startMonth={new Date('2026-01-01')}   // v10: 구 fromMonth
  endMonth={new Date('2026-12-31')}     // v10: 구 toMonth
  defaultMonth={new Date('2026-06-01')}
/>
```

---

---

# 폼과 함께 사용 — 날짜 입력 + 달력 팝업

```tsx
'use client';
import { useState } from 'react';
import { DayPicker } from 'react-day-picker';
import { format } from 'date-fns';
import { ko } from 'date-fns/locale';

export default function DateInputWithPicker() {
  const [selected, setSelected]     = useState<Date | undefined>();
  const [showPicker, setShowPicker] = useState(false);

  return (
    <div className="relative">
      <input
        readOnly
        value={selected ? format(selected, 'yyyy-MM-dd') : ''}
        placeholder="날짜 선택"
        onClick={() => setShowPicker(true)}
        className="border rounded px-3 py-2 cursor-pointer"
      />

      {showPicker && (
        <div className="absolute z-10 bg-white shadow-lg rounded-lg p-2">
          <DayPicker
            mode="single"
            selected={selected}
            onSelect={(date) => {
              setSelected(date);
              setShowPicker(false);
            }}
            locale={ko}
          />
        </div>
      )}
    </div>
  );
}
```

---

---

# Tailwind 스타일 커스터마이징 — v10 classNames ⭐️

```tsx
<DayPicker
  mode="single"
  selected={selected}
  onSelect={setSelected}
  classNames={{
    months:        'flex gap-4',
    month:         'space-y-4',
    month_caption: 'flex justify-center relative items-center',
    caption_label: 'text-sm font-medium',
    nav:           'flex items-center gap-1',
    button_previous: 'p-1 rounded hover:bg-gray-100',
    button_next:     'p-1 rounded hover:bg-gray-100',
    month_grid:    'w-full border-collapse',
    weekday:       'text-gray-500 text-xs font-normal',
    day:           'h-9 w-9 rounded hover:bg-gray-100 cursor-pointer',
    selected:      'bg-blue-500 text-white hover:bg-blue-600',
    today:         'font-bold text-blue-500',
    disabled:      'text-gray-300 cursor-not-allowed',
    range_middle:  'bg-blue-100 rounded-none',
  }}
/>
```

```txt
⚠️ 구버전(v8/v9) 키 → v10 키 매핑 (헷갈리기 쉬운 것들):
  day_selected     → selected
  day_today        → today
  day_disabled     → disabled
  day_range_start  → range_start
  day_range_end    → range_end
  day_range_middle → range_middle
  table            → month_grid
  nav_button       → button_previous / button_next (이전/다음 분리됨)
  caption          → month_caption
```

---

---

# 전시 검색 필터 패턴 ⭐️

```typescript
import { isWithinInterval, parseISO } from 'date-fns';
import { DateRange } from 'react-day-picker';

function filterByDateRange(exhibitions: Exhibition[], range: DateRange | undefined) {
  if (!range?.from) return exhibitions;

  return exhibitions.filter(e => {
    const start = parseISO(e.startDate);
    const end   = parseISO(e.endDate);

    return isWithinInterval(range.from!, { start, end })
      || (range.to && isWithinInterval(range.to, { start, end }));
  });
}
```

---

---

# 한눈에

| |date-fns|react-day-picker|
|---|---|---|
|역할|날짜 계산 / 포맷|달력 UI|
|주요 함수|`format` / `parseISO` / `isAfter` / `differenceInDays`|`DayPicker`|
|모드|-|`single` / `range` / `multiple`|
|커스텀 표시|-|`modifiers` + `modifiersClassNames` ⭐️|
|한국어|`{ locale: ko }`|`locale={ko}`|

```bash
설치: pnpm add react-day-picker date-fns

자주 쓰는 조합:
  format(date, 'yyyy-MM-dd')              날짜 → 문자열
  parseISO('2026-06-16')                  문자열 → Date
  differenceInDays(end, start)            날짜 차이
  isWithinInterval(date, { start, end })  기간 내 여부

방문/표시 날짜 배열 만들 때 ⭐️:
  1. 날짜 문자열(YYYY-MM-DD)로 Set 중복 제거  # → [[JS_WeakMap_Set]]
  2. toVisitDate 로 안전하게 Date 변환        # → [[JS_Date]]
  3. modifiers={{ name: dates }} 로 DayPicker 에 전달
```