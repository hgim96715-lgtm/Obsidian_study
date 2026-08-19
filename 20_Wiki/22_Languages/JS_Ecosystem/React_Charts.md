---
aliases: [chart library, Nivo, PartialTheme]
tags: [React]
related:
  - "[[00_JS_Ecosystem_HomePage]]"
  - "[[NestJS_Prisma]]"
  - "[[NestJS_StatsBucket]]"
  - "[[NextJS_ServerClient]]"
---
# React_Charts — 차트 라이브러리

>[!info]
>React 차트 라이브러리 비교 + 실전 패턴.
> Nivo = D3 기반, 패키지별 분리, `Responsive*` 컴포넌트로 부모 크기에 자동 대응.
>  `data` prop 형식이 차트 타입마다 다름.

---

# 라이브러리 비교 ⭐️⭐️⭐️⭐️

```txt
D3(Data-Driven Documents)란:
  SVG를 직접 조작해서 데이터를 시각화하는 저수준 라이브러리
  "데이터가 바뀌면 DOM을 이렇게 바꿔라"를 직접 코드로 작성
  자유도가 높지만 코드가 복잡하고 배우기 어려움

  D3로 꺾은선 하나 그리려면:
    SVG 요소 만들기 → 스케일 함수 정의 → 축 생성 → path 계산 → append
    → 수십 줄의 코드

Nivo, Recharts, Chart.js:
  D3 위에 만들어진 React 래퍼
  내부적으로 D3가 계산을 하지만
  개발자는 props만 넘기면 됨
  → data={...} 한 줄로 꺾은선이 그려짐
```

|라이브러리|기반|특징|적합한 경우|
|---|---|---|---|
|**Nivo**|D3 + SVG|패키지 분리·Responsive·애니메이션·테마|커스터마이즈가 많은 어드민·대시보드|
|**Recharts**|D3 + SVG|API가 단순·컴포넌트 조합|빠르게 붙이는 간단한 차트|
|**Chart.js** (react-chartjs-2)|Canvas|Canvas 렌더링·대용량 데이터 성능|데이터가 많거나 실시간 업데이트|
|**Tremor**|Tailwind + Recharts|Tailwind 통합·admin UI 컴포넌트|Tailwind 프로젝트 빠른 대시보드|

```txt
Recharts:
  npm 다운로드 1위 — 레퍼런스·예제가 많음
  <LineChart><Line /><XAxis /></LineChart> 식 조합
  Nivo보다 props 구조가 단순

Chart.js:
  Canvas 기반 → 데이터 수천 개도 빠름
  react-chartjs-2로 React 래핑
  SVG 아님 → CSS 커스터마이즈 제한

Tremor:
  Tailwind 기반 어드민 컴포넌트 라이브러리
  차트가 포함된 완성형 UI 블록
  커스터마이즈 자유도 낮음
```

---

# Nivo ⭐️⭐️⭐️⭐
️
> https://nivo.rocks
## 설치

```bash
pnpm --filter web add @nivo/core @nivo/line @nivo/bar @nivo/pie @nivo/heatmap

# 추가로 필요한 경우
pnpm --filter web add @nivo/calendar @nivo/scatterplot @nivo/stream
```

## 핵심 개념

```tsx
// Responsive* — 부모 크기에 자동 맞춤 (권장)
<div style={{ height: 300 }}>   {/* 부모에 height 반드시 */}
  <ResponsiveLine data={data} />
</div>

// 고정 크기
<Line width={600} height={300} data={data} />
```

```typescript
// data 형식 — 차트 타입마다 다름 ⭐️

// Line — [{ id, data: [{x, y}] }]
const lineData = [
  { id: '방문자', data: [{ x: '08-12', y: 120 }, { x: '08-13', y: 85 }] },
  { id: '가입자', data: [{ x: '08-12', y: 30 },  { x: '08-13', y: 45 }] },
];

// Bar — 평탄한 객체 배열
const barData = [
  { date: '08-12', 방문자: 120, 가입자: 30 },
  { date: '08-13', 방문자: 85,  가입자: 45 },
];

// Pie — [{ id, value }]
const pieData = [
  { id: '모바일', value: 60 },
  { id: '데스크탑', value: 35 },
];
```

## Line 차트

```tsx
'use client';
import { ResponsiveLine } from '@nivo/line';

<div style={{ height: 300 }}>
  <ResponsiveLine
    data={data}
    margin={{ top: 20, right: 20, bottom: 50, left: 60 }}
    xScale={{ type: 'point' }}        // point: 이산값 / linear: 연속 숫자
    yScale={{ type: 'linear', min: 0 }}
    axisBottom={{ legend: '날짜', legendOffset: 36 }}
    axisLeft={{   legend: '방문자', legendOffset: -50 }}
    enablePoints={true}               // 꺾이는 점 표시
    useMesh={true}                    // hover tooltip 감지
    curve="monotoneX"                 // 선 형태: linear | monotoneX | step
    enableArea={false}                // true → 면 채우기
    legends={[{ anchor: 'top-right', direction: 'column', itemWidth: 80, itemHeight: 20 }]}
  />
</div>
```

## Bar 차트

```tsx
'use client';
import { ResponsiveBar } from '@nivo/bar';

<div style={{ height: 300 }}>
  <ResponsiveBar
    data={data}
    keys={['방문자', '가입자']}   // 막대로 그릴 필드
    indexBy="date"              // x축 기준 필드
    margin={{ top: 20, right: 80, bottom: 50, left: 60 }}
    padding={0.3}
    groupMode="grouped"         // grouped: 나란히 | stacked: 쌓기
    layout="vertical"           // vertical | horizontal
    axisBottom={{ legend: '날짜', legendOffset: 36 }}
    axisLeft={{ legend: '수', legendOffset: -50 }}
    legends={[{ dataFrom: 'keys', anchor: 'bottom-right', direction: 'column', itemWidth: 100, itemHeight: 20 }]}
  />
</div>
```

## Pie 차트

```tsx
'use client';
import { ResponsivePie } from '@nivo/pie';

<div style={{ height: 300 }}>
  <ResponsivePie
    data={data}
    margin={{ top: 20, right: 80, bottom: 80, left: 80 }}
    innerRadius={0.5}     // 0 = 파이, 0.5~0.7 = 도넛
    padAngle={2}
    cornerRadius={3}
    arcLabelsTextColor="white"
  />
</div>
```

## Heatmap

```tsx
'use client';
import { ResponsiveHeatMap } from '@nivo/heatmap';

// 행: 시간대(0~23), 열: 날짜
const data = [
  { id: '00시', data: [{ x: '08-12', y: 5 }, { x: '08-13', y: 12 }] },
  { id: '01시', data: [{ x: '08-12', y: 2 }, { x: '08-13', y: 8 }] },
];

<div style={{ height: 400 }}>
  <ResponsiveHeatMap
    data={data}
    margin={{ top: 20, right: 20, bottom: 60, left: 60 }}
    valueFormat=">-.0f"
    colors={{ type: 'sequential', scheme: 'blues' }}
  />
</div>
```

## 통계 데이터 → Nivo 데이터 변환

```typescript
type DailyStat = { date: string; visits: number; signups: number };

// API 응답 → Line 데이터
function toLineData(stats: DailyStat[]) {
  return [
    { id: '방문자', data: stats.map(s => ({ x: s.date, y: s.visits })) },
    { id: '가입자', data: stats.map(s => ({ x: s.date, y: s.signups })) },
  ];
}

// 빈 날짜 0으로 채우기 — 차트 x축이 비어있으면 채워야 이어짐
function fillEmptyDays(stats: DailyStat[], keys: string[]): DailyStat[] {
  const map = new Map(stats.map(s => [s.date, s]));
  return keys.map(date => map.get(date) ?? { date, visits: 0, signups: 0 });
}
```

## Next.js 주의사항

```tsx
// 'use client' 필수 — SVG는 브라우저에서만
'use client';
export function LineChart({ data }: { data: LineData[] }) {
  return (
    <div style={{ height: 300 }}>  {/* height 없으면 차트 안 보임 */}
      <ResponsiveLine data={data} />
    </div>
  );
}

// SSR 비활성화 방법
import dynamic from 'next/dynamic';
const LineChart = dynamic(() => import('./LineChart'), { ssr: false });
```

---

# Recharts — 빠른 구현 ⭐️⭐️⭐️

```bash
pnpm --filter web add recharts
```

```tsx
// 컴포넌트 조합 방식 — Nivo보다 단순
import { LineChart, Line, XAxis, YAxis, CartesianGrid, Tooltip, ResponsiveContainer } from 'recharts';

const data = [
  { date: '08-12', visits: 120, signups: 30 },
  { date: '08-13', visits: 85,  signups: 45 },
];

<ResponsiveContainer width="100%" height={300}>
  <LineChart data={data}>
    <CartesianGrid strokeDasharray="3 3" />
    <XAxis dataKey="date" />
    <YAxis />
    <Tooltip />
    <Line type="monotone" dataKey="visits"  stroke="#8884d8" />
    <Line type="monotone" dataKey="signups" stroke="#82ca9d" />
  </LineChart>
</ResponsiveContainer>
```

```txt
Nivo vs Recharts:
  Nivo   → props 한 컴포넌트에 집중, 상세 커스터마이즈
  Recharts → JSX 조합, 직관적, API 레퍼런스 많음
  데이터 형식: Recharts는 { date, visits } 평탄한 배열 (Bar와 같음)
```

---

# Chart.js (react-chartjs-2) — 대용량 데이터 ⭐️⭐️⭐️

```bash
pnpm --filter web add chart.js react-chartjs-2
```

```tsx
import { Line } from 'react-chartjs-2';
import { Chart as ChartJS, CategoryScale, LinearScale, PointElement, LineElement, Tooltip, Legend } from 'chart.js';

// 필요한 컴포넌트 등록 (Tree-shaking)
ChartJS.register(CategoryScale, LinearScale, PointElement, LineElement, Tooltip, Legend);

const data = {
  labels: ['08-12', '08-13', '08-14'],
  datasets: [
    {
      label: '방문자',
      data: [120, 85, 200],
      borderColor: '#8884d8',
    },
  ],
};

<div style={{ height: 300 }}>
  <Line data={data} options={{ responsive: true, maintainAspectRatio: false }} />
</div>
```

```txt
Nivo·Recharts(SVG) vs Chart.js(Canvas):
  SVG    → DOM 요소 → CSS 접근 가능, 적은 데이터에 적합
  Canvas → 픽셀 직접 그림 → 수천 개 데이터도 빠름
  실시간 업데이트가 많으면 Chart.js 권장
```

---

# Tremor — Tailwind 통합 대시보드 ⭐️⭐️

```bash
pnpm --filter web add @tremor/react
```

```tsx
import { LineChart } from '@tremor/react';

const data = [
  { date: '08-12', 방문자: 120 },
  { date: '08-13', 방문자: 85 },
];

<LineChart
  data={data}
  index="date"
  categories={['방문자']}
  colors={['blue']}
  className="h-72"   // Tailwind height
/>
```

```txt
가장 단순 — 설정이 거의 없음
Tailwind 기반이라 className으로 스타일
커스터마이즈가 제한적 → 빠른 프로토타입에 적합
```