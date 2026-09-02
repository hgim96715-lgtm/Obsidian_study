---
aliases:
  - 날짜 선택
  - 달력
  - date-fns
  - react-day-picker
  - input type date
  - react-datepicker
tags:
  - React
  - NextJS
relations:
  - "[[JS_Date]]"
  - "[[JS_Intl]]"
  - "[[React_Input]]"
  - "[[00_JS_Ecosystem_HomePage]]"
---
# React_DatePicker — 날짜 선택 컴포넌트

> [!info]
> React에서 날짜를 입력받는 방법은 크게 두 가지다.
> **① 브라우저 내장** `input type="date"` — 설치 없이 바로, 커스텀 불가
> **② 라이브러리** `react-datepicker` — 커스텀 달력 UI, 범위·다중 선택 등 고급 기능
> 날짜 계산 → [[JS_Date]] | 날짜 포맷 표시 → [[JS_Intl]]

---

# 방법 선택 — 언제 뭘 쓰는가

```txt
input type="date" 로 충분한 경우:
  단순 날짜 입력 (생년월일, 마감일 등)
  브라우저 기본 UI여도 무방
  라이브러리 추가 없이 빠르게

라이브러리가 필요한 경우:
  달력 UI를 직접 디자인해야 함
  날짜 범위를 달력에서 드래그로 선택
  여러 날짜 multi-select
  특정 날짜 비활성화
  주/월 단위 선택
```

---

# ① 브라우저 내장 — `input type="date"` ⭐️⭐️⭐️⭐️

## 동작 원리

```txt
브라우저가 날짜 선택 UI(달력 팝업)를 직접 제공
→ 별도 설치 없음, 모바일에서는 네이티브 피커 연동

값 형식: 항상 "YYYY-MM-DD" (ISO 8601)
  선택됨  → "2024-01-15"
  미선택  → "" (빈 문자열, undefined·null 아님)
```


## 달력 팝업은 어떻게 나오는가 ⭐️⭐️⭐️⭐️

```txt
input type="date"만 쓰면 달력은 자동으로 나옴
→ 브라우저가 알아서 렌더링 (코드로 따로 만드는 게 아님)
→ 캘린더 아이콘 클릭 또는 input 클릭 → 브라우저 네이티브 달력 팝업

화면의 달력 UI는 OS + 브라우저가 결정:
  어두운 테마(다크모드) → OS 다크모드 설정이면 자동 적용
  한국어 날짜 표기 ("2026년 9월", "일 월 화 수 목 금 토") → 브라우저 언어 설정이 한국어
  모양(Chrome/Firefox/Safari 각자 다름) → 브라우저마다 다른 네이티브 UI
```

```txt
코드로 제어할 수 있는 것:
  max / min   → 선택 가능한 날짜 범위 제한
  value       → 선택된 날짜 (YYYY-MM-DD 문자열)
  disabled    → 입력 비활성화

코드로 제어할 수 없는 것:
  달력 팝업의 색상, 폰트, 레이아웃 → OS/브라우저가 그림
  CSS로 스타일링 불가 (::picker 등 일부 실험적 기능 제외)
  언어/로케일 → 브라우저 설정에 따름

→ 달력 UI를 직접 디자인해야 하면 라이브러리 필요 (아래 ② 참고)
```

```tsx
// max로 "오늘 이후 선택 불가" 구현
function getKstTodayDate(): string {
  // KST 기준 오늘 날짜를 "YYYY-MM-DD"로 반환
  const now = new Date();
  const kst = new Date(now.getTime() + 9 * 60 * 60 * 1000);
  return kst.toISOString().slice(0, 10);
}

<input
  type="date"
  value={watchedAt}
  max={getKstTodayDate()}   // 오늘까지만 선택 가능 → 미래 날짜는 비활성화
  onChange={(e) => setWatchedAt(e.target.value)}
/>
```

## 기본 사용

```tsx
function DateInput() {
  const [date, setDate] = useState('');  // "" = 미선택

  return (
    <input
      type="date"
      value={date}
      onChange={(e) => setDate(e.target.value)}
    />
  );
}
```


## label + htmlFor — 클릭 영역 확장 + 접근성 ⭐️⭐️⭐️⭐️

```tsx
<label htmlFor="watched-date-input">새로운 관람일</label>
<input
  id="watched-date-input"
  type="date"
  value={watchedAt}
  max={getKstTodayDate()}
  onChange={(e) => setWatchedAt(e.target.value)}
  disabled={isPending}
/>
```

```txt
htmlFor + id 연결 구조:
  <label htmlFor="watched-date-input"> ← "이 id를 가진 input과 연결됨"
  <input id="watched-date-input" ...>  ← "watched-date-input"이라는 id를 가짐

  → label 텍스트 클릭 = input 클릭과 동일
  → type="date"에서: 라벨 클릭 시 달력 팝업이 열림

왜 htmlFor인가 — HTML for vs React htmlFor:
  HTML  : <label for="id">
  React : <label htmlFor="id">

  for는 JavaScript 예약어 (for 루프에 쓰는 그 for)
  → JSX에서 그대로 쓰면 문법 충돌
  → React가 htmlFor로 이름을 바꿈 (HTML className과 같은 이유)

  className ← HTML class (JS 예약어 class와 충돌)
  htmlFor   ← HTML for  (JS 예약어 for와 충돌)
```

```txt
label + htmlFor 없으면:
  input 박스 자체만 클릭 가능 → 클릭 영역이 작음
  스크린리더가 이 input이 무엇을 입력하는 필드인지 모름

label + htmlFor 있으면:
  라벨 텍스트("새로운 관람일")를 클릭해도 input 활성화
  → 모바일에서 터치 타깃 확장
  스크린리더: "새로운 관람일, 날짜 입력" 으로 안내
```

| HTML 속성 | React prop | 이유 |
|---|---|---|
| `class` | `className` | `class`는 JS 예약어 |
| `for` | `htmlFor` | `for`는 JS 예약어 |
| `tabindex` | `tabIndex` | camelCase 통일 |
| `readonly` | `readOnly` | camelCase 통일 |

## string vs Date — 상태를 뭘로 관리하는가 ⭐️⭐️⭐️⭐️

```txt
input type="date"는 string("YYYY-MM-DD")으로 주고받음

string 상태 유지 (권장 — input과 바로 연결):
  계산이 필요할 때만 new Date(dateStr)로 변환

Date 상태 유지 (계산이 많을 때):
  input에 넣을 때 다시 "YYYY-MM-DD"로 변환 필요
```

```typescript
// ✅ string 상태 (권장)
const [dateStr, setDateStr] = useState<string>('');

// 계산이 필요할 때만 변환
const dateObj = dateStr ? new Date(dateStr) : null;
```

```typescript
// Date 상태 (계산 많을 때)
const [date, setDate] = useState<Date | null>(null);

function toInputValue(d: Date | null): string {
  if (!d) return '';
  return d.toISOString().slice(0, 10);  // "2024-01-15T..." → "2024-01-15"
}

<input
  type="date"
  value={toInputValue(date)}
  onChange={(e) => setDate(e.target.value ? new Date(e.target.value) : null)}
/>
```

## 컴포넌트로 분리

```tsx
type DatePickerProps = {
  value:     string;
  onChange:  (value: string) => void;
  label?:    string;
  min?:      string;   // "YYYY-MM-DD"
  max?:      string;
  required?: boolean;
};

function DatePicker({ value, onChange, label, min, max, required }: DatePickerProps) {
  return (
    <div className="flex flex-col gap-1">
      {label && <label className="text-sm font-medium">{label}</label>}
      <input
        type="date"
        value={value}
        onChange={(e) => onChange(e.target.value)}
        min={min}
        max={max}
        required={required}
        className="rounded border px-3 py-2"
      />
    </div>
  );
}

// 사용
<DatePicker
  label="생년월일"
  value={birthDate}
  onChange={setBirthDate}
  max={new Date().toISOString().slice(0, 10)}  // 오늘까지만
/>
```

## min/max — 선택 가능 날짜 범위 제한

```typescript
function todayString(): string {
  return new Date().toISOString().slice(0, 10);
}

function daysFromNow(n: number): string {
  const d = new Date();
  d.setDate(d.getDate() + n);
  return d.toISOString().slice(0, 10);
}
```

```tsx
{/* 오늘 이후만 선택 (예약·마감일) */}
<input type="date" min={todayString()} />

{/* 오늘 이전만 선택 (생년월일) */}
<input type="date" max={todayString()} />

{/* 오늘부터 30일 이내 */}
<input type="date" min={todayString()} max={daysFromNow(30)} />
```

```txt
⚠️ min/max는 브라우저 UI만 막음
   직접 타이핑 입력은 제한 못함 → 서버에서도 유효성 검사 필수
```

## 날짜 범위 선택 (Range Picker)

```tsx
function DateRangePicker() {
  const [startDate, setStartDate] = useState('');
  const [endDate,   setEndDate]   = useState('');

  const handleStartChange = (value: string) => {
    setStartDate(value);
    if (endDate && value > endDate) setEndDate('');  // 시작 > 종료면 종료 초기화
  };

  return (
    <div className="flex items-center gap-2">
      <input
        type="date"
        value={startDate}
        onChange={(e) => handleStartChange(e.target.value)}
        max={endDate || undefined}
      />
      <span>~</span>
      <input
        type="date"
        value={endDate}
        onChange={(e) => setEndDate(e.target.value)}
        min={startDate || undefined}
      />
    </div>
  );
}
```

```txt
"YYYY-MM-DD" 형식은 사전순 정렬 = 날짜 순서
  "2024-01-15" > "2024-01-01"  → true
  → new Date() 없이 문자열 비교로 날짜 대소 비교 가능
```

## datetime-local — 날짜 + 시간

```tsx
const [datetime, setDatetime] = useState('');
// 형식: "2024-01-15T09:30"

<input
  type="datetime-local"
  value={datetime}
  onChange={(e) => setDatetime(e.target.value)}
/>
```

```txt
datetime-local 값 형식: "YYYY-MM-DDTHH:mm"

서버 전송 시 ISO 변환:
  new Date(datetime).toISOString()  → "2024-01-15T00:30:00.000Z" (UTC)

⚠️ new Date("2024-01-15") → UTC 자정으로 파싱
   날짜가 하루 전으로 나오는 원인
   → new Date("2024-01-15T00:00:00") 로컬 시간 명시하면 해결
```

## 선택한 날짜 한국어 표시

```typescript
// "2024-01-15" → "2024년 1월 15일"
function formatDate(dateStr: string): string {
  if (!dateStr) return '';
  return new Intl.DateTimeFormat('ko-KR', {
    year: 'numeric', month: 'long', day: 'numeric',
  }).format(new Date(dateStr));
}
```

---

# ② 라이브러리 — react-datepicker ⭐️⭐️⭐️

## 설치

```bash
pnpm add react-datepicker
pnpm add -D @types/react-datepicker
```

## 기본 사용

```tsx
import DatePicker from 'react-datepicker';
import 'react-datepicker/dist/react-datepicker.css';
import { ko } from 'date-fns/locale';

function MyDatePicker() {
  const [date, setDate] = useState<Date | null>(null);

  return (
    <DatePicker
      selected={date}
      onChange={(d) => setDate(d)}
      locale={ko}
      dateFormat="yyyy년 MM월 dd일"
      placeholderText="날짜 선택"
    />
  );
}
```

```txt
react-datepicker는 Date 객체로 상태 관리
  selected: Date | null
  onChange: (date: Date | null) => void

서버로 보낼 때:
  date?.toISOString()  → UTC ISO 문자열
  date?.toLocaleDateString('ko-KR')  → "2024. 1. 15."
```

## 날짜 범위 선택

```tsx
const [startDate, setStartDate] = useState<Date | null>(null);
const [endDate,   setEndDate]   = useState<Date | null>(null);

<DatePicker
  selectsRange
  startDate={startDate}
  endDate={endDate}
  onChange={([start, end]) => {
    setStartDate(start);
    setEndDate(end);
  }}
  locale={ko}
  dateFormat="yyyy.MM.dd"
  placeholderText="기간 선택"
/>
```

## 주요 옵션

|prop|타입|설명|
|---|---|---|
|`selected`|`Date \| null`|선택된 날짜|
|`onChange`|`(date) => void`|날짜 변경 콜백|
|`minDate`|`Date`|선택 가능 최소 날짜|
|`maxDate`|`Date`|선택 가능 최대 날짜|
|`excludeDates`|`Date[]`|선택 불가 날짜 목록|
|`showTimeSelect`|`boolean`|시간 선택 활성화|
|`dateFormat`|`string`|표시 형식 (date-fns 패턴)|
|`locale`|`Locale`|언어 설정 (`ko`)|
|`inline`|`boolean`|팝업 없이 항상 표시|

---

# 자주 만나는 문제

|증상|원인|해결|
|---|---|---|
|input이 비어있음|`value`가 `undefined`|`''`(빈 문자열)로 초기화|
|날짜가 하루 전으로 나옴|`new Date("2024-01-15")`가 UTC 자정 파싱|`new Date("2024-01-15T00:00:00")` 로컬 명시|
|max 넘는 날짜가 입력됨|브라우저 UI만 막음|서버 유효성 검사 추가|
|모바일 UI 이상함|브라우저마다 다른 피커 UI|라이브러리 사용 검토|
|스타일이 안 맞음|브라우저 기본 스타일 덮어쓰기 어려움|`react-datepicker` 커스텀 CSS|
