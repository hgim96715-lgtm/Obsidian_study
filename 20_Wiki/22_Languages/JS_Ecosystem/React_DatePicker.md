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
  - "[[React_DatePicker]]"
---
# React_DatePicker — 날짜 선택

> [!info] 
> `input type="date"` = 브라우저 내장 날짜 선택 UI. 
> 값은 `"2024-01-15"` 형식(YYYY-MM-DD) 문자열로 주고받는다. 
> 커스텀 스타일이 필요하거나 범위·시간·달력 UI가 필요하면 react-datepicker 등 라이브러리를 쓴다. 
> 날짜 계산 → [[JS_Date]], 날짜 포맷 표시 → [[JS_Intl]]

---

# input type="date" — 가장 간단한 방법 ⭐️⭐️⭐️⭐️

```tsx
function DateInput() {
  const [date, setDate] = useState('');  // 빈 문자열 = 미선택

  return (
    <input
      type="date"
      value={date}
      onChange={(e) => setDate(e.target.value)}
    />
  );
}
```

```txt
input type="date" 가 해주는 것:
  브라우저가 날짜 선택 UI를 직접 제공 (달력 팝업)
  키보드·마우스로 날짜 입력 가능
  모바일에서는 네이티브 날짜 피커

값 형식:
  항상 "YYYY-MM-DD" (ISO 8601 날짜 형식)
  예: "2024-01-15"

  선택 안 된 상태: "" (빈 문자열)
  → undefined나 null이 아닌 ""임에 주의
```

---

# string vs Date — 상태를 뭘로 관리하는가 ⭐️⭐️⭐️⭐️

```txt
input type="date"는 string으로 주고받음 (YYYY-MM-DD)
그대로 string으로 저장하면 input에 다시 value로 넣기 쉬움

Date 객체로 저장하면:
  계산·비교는 쉬움
  input에 넣을 때 다시 "YYYY-MM-DD" 형식으로 변환해야 함
```

```typescript
// string 상태 유지 (권장 — input과 바로 연결)
const [dateStr, setDateStr] = useState<string>('');
// dateStr = "2024-01-15" 또는 ""

// 계산이 필요할 때만 Date로 변환
const dateObj = dateStr ? new Date(dateStr) : null;
```

```typescript
// Date 상태 유지 (계산이 많을 때)
const [date, setDate] = useState<Date | null>(null);

// input value: Date → "YYYY-MM-DD" 변환 필요
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

---

# 기본 DatePicker 컴포넌트 ⭐️⭐️⭐️

```tsx
type DatePickerProps = {
  value:    string;          // "YYYY-MM-DD" 또는 ""
  onChange: (value: string) => void;
  label?:   string;
  min?:     string;          // "YYYY-MM-DD"
  max?:     string;
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
const [birthDate, setBirthDate] = useState('');

<DatePicker
  label="생년월일"
  value={birthDate}
  onChange={setBirthDate}
  max={new Date().toISOString().slice(0, 10)}  // 오늘까지만 선택 가능
/>
```

---

# min/max — 선택 가능 날짜 범위 제한 ⭐️⭐️⭐️

```typescript
// 오늘 날짜를 "YYYY-MM-DD" 형식으로
function todayString(): string {
  return new Date().toISOString().slice(0, 10);
}

// n일 후를 "YYYY-MM-DD" 형식으로
function daysFromNow(n: number): string {
  const d = new Date();
  d.setDate(d.getDate() + n);
  return d.toISOString().slice(0, 10);
}
```

```tsx
{/* 오늘 이후만 선택 가능 (예약, 마감일) */}
<input type="date" min={todayString()} />

{/* 오늘 이전만 선택 가능 (생년월일) */}
<input type="date" max={todayString()} />

{/* 오늘부터 30일 이내만 선택 */}
<input type="date" min={todayString()} max={daysFromNow(30)} />
```

```txt
min/max를 벗어난 날짜:
  브라우저 UI에서 선택 불가 (회색 처리)
  직접 타이핑으로 입력하면 값이 들어갈 수 있음
  → 폼 제출 시 서버에서도 유효성 검사 필요
```

---

# 날짜 범위 선택 (Range Picker) ⭐️⭐️⭐️⭐️

```tsx
function DateRangePicker() {
  const [startDate, setStartDate] = useState('');
  const [endDate,   setEndDate]   = useState('');

  const handleStartChange = (value: string) => {
    setStartDate(value);
    // 시작일이 종료일보다 늦으면 종료일 초기화
    if (endDate && value > endDate) setEndDate('');
  };

  const handleEndChange = (value: string) => {
    setEndDate(value);
  };

  return (
    <div className="flex items-center gap-2">
      <input
        type="date"
        value={startDate}
        onChange={(e) => handleStartChange(e.target.value)}
        max={endDate || undefined}  // 종료일 이전만 선택 가능
      />
      <span>~</span>
      <input
        type="date"
        value={endDate}
        onChange={(e) => handleEndChange(e.target.value)}
        min={startDate || undefined}  // 시작일 이후만 선택 가능
      />
    </div>
  );
}
```

```txt
string 비교로 날짜 대소 비교 가능:
  "YYYY-MM-DD" 형식은 사전순 정렬 = 날짜 순서
  "2024-01-15" > "2024-01-01"  → true (더 최근)
  new Date() 없이도 간단히 비교 가능

  if (value > endDate)  →  시작일이 종료일보다 늦으면
  max={endDate}          →  endDate 이전 날짜만 선택 가능
```

---

# input type="datetime-local" — 날짜 + 시간 ⭐️⭐️⭐️

```tsx
const [datetime, setDatetime] = useState('');
// 형식: "2024-01-15T09:30"

<input
  type="datetime-local"
  value={datetime}
  onChange={(e) => setDatetime(e.target.value)}
  min="2024-01-01T00:00"
  max="2024-12-31T23:59"
/>
```

```txt
datetime-local 값 형식:
  "YYYY-MM-DDTHH:mm"  (예: "2024-01-15T09:30")
  초는 포함 안 됨 (브라우저마다 다를 수 있음)

Date 객체로 변환:
  new Date("2024-01-15T09:30")   → 로컬 시간 기준으로 파싱
  new Date("2024-01-15T09:30Z")  → UTC 기준 (Z 붙이면)

서버 전송 시:
  new Date(datetime).toISOString()  → "2024-01-15T00:30:00.000Z" (UTC)
```

---

# 선택한 날짜를 화면에 표시하는 방법 ⭐️⭐️⭐️

```tsx
// "2024-01-15" → "2024년 1월 15일"
function formatDate(dateStr: string): string {
  if (!dateStr) return '';
  return new Intl.DateTimeFormat('ko-KR', {
    year:  'numeric',
    month: 'long',
    day:   'numeric',
  }).format(new Date(dateStr));
}

function DateDisplay({ dateStr }: { dateStr: string }) {
  return (
    <div>
      <input type="date" value={dateStr} ... />
      {dateStr && (
        <p className="text-sm text-gray-600">
          선택한 날짜: {formatDate(dateStr)}
          {/* "선택한 날짜: 2024년 1월 15일" */}
        </p>
      )}
    </div>
  );
}
```

---

# 유효성 검사 ⭐️⭐️⭐️

```typescript
type DateValidation = {
  required?: boolean;
  min?:      string;  // "YYYY-MM-DD"
  max?:      string;
};

function validateDate(value: string, rules: DateValidation): string | null {
  if (rules.required && !value) return '날짜를 선택해주세요.';
  if (!value) return null;  // 선택 안 했고 필수도 아님

  if (rules.min && value < rules.min)
    return `${formatDate(rules.min)} 이후 날짜를 선택해주세요.`;
  if (rules.max && value > rules.max)
    return `${formatDate(rules.max)} 이전 날짜를 선택해주세요.`;

  return null;  // 유효
}

// 사용
const error = validateDate(birthDate, { required: true, max: todayString() });
```

---

# 라이브러리가 필요한 경우

```txt
input type="date"로 부족할 때:
  ① 커스텀 달력 UI (브라우저 기본 UI 대신)
  ② 날짜 범위를 달력에서 드래그로 선택
  ③ 여러 날짜 선택 (multi-select)
  ④ 커스텀 비활성화 날짜 (특정 날짜 선택 불가)
  ⑤ 주/월 단위 선택

추천 라이브러리:
  react-datepicker  — 가장 많이 쓰임, 한국어 지원
  react-day-picker  — 가볍고 접근성 좋음
  @nextui-org/date-picker  — Tailwind CSS 친화적

설치 예시
  pnpm add react-datepicker
  pnpm add -D @types/react-datepicker

  import DatePicker from 'react-datepicker';
  import 'react-datepicker/dist/react-datepicker.css';
  import { ko } from 'date-fns/locale';

  <DatePicker
    selected={date}
    onChange={(d) => setDate(d)}
    locale={ko}
    dateFormat="yyyy년 MM월 dd일"
    placeholderText="날짜 선택"
  />
```

---

# 자주 만나는 문제

|증상|원인|해결|
|---|---|---|
|선택했는데 input이 비어있음|`value`가 `undefined`|빈 문자열 `''`로 초기화|
|날짜가 하루 전으로 나옴|`new Date("2024-01-15")`가 UTC 자정으로 해석|`new Date("2024-01-15T00:00:00")` 로컬 시간 명시|
|`max`보다 늦은 날짜가 입력됨|브라우저 UI는 막지만 직접 입력은 막지 못함|서버 유효성 검사 추가|
|모바일에서 UI가 이상함|브라우저마다 다른 날짜 피커 UI|라이브러리 사용 검토|