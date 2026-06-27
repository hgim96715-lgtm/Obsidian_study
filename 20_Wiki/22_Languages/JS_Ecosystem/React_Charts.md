---
aliases:
  - recharts
  - PieChart
  - ResponsiveContainer
  - Chart.js
  - 차트
  - 그래프
tags:
  - React
  - NextJS
related:
  - "[[00_JS_Ecosystem_HomePage]]"
  - "[[React_Component]]"
  - "[[NextJS_Data_Fetching]]"
---

# React_Charts — React 에서 차트 그리기 (recharts)

# 한 줄 요약

```
recharts = React 컴포넌트로 차트를 그리는 라이브러리 (D3 기반, JSX 스타일 API)
<PieChart><Pie data={...} /></PieChart> 처럼 차트도 그냥 컴포넌트 조립하듯 작성
```

---

---

# recharts 란 — 왜 쓰는가 ⭐️

```
차트를 직접 그리려면(D3 등) 보통 "이 좌표에 이 모양을 그려라" 같은 명령형 코드가 필요함
recharts 는 그 복잡한 부분을 감춰두고
"이런 데이터를, 이런 모양(Pie/Bar/Line)으로 보여달라" 고 React 컴포넌트로 선언만 하면 끝나게 해줌

→ React 의 다른 UI 컴포넌트들(button, div 등)을 조립하는 것과 똑같은 방식으로 차트도 조립
```

```bash
pnpm add recharts --filter web
```

---

---

# 다른 차트 라이브러리들과 비교 ⭐️

|라이브러리|방식|장점|단점|
|---|---|---|---|
|recharts ⭐️|React 컴포넌트 (D3 내부 사용)|JSX 로 바로 작성, React 와 가장 자연스러움|아주 복잡한 커스텀 시각화엔 한계|
|Chart.js (+ react-chartjs-2)|Canvas 기반, 설정 객체 방식|가볍고 빠름, 오래되고 자료 많음|React 식이 아니라 옵션 객체를 직접 만져야 함|
|visx (Airbnb)|D3 의 각 기능을 React 컴포넌트로 쪼갠 것|아주 유연함, 뭐든 만들 수 있음|코드가 길어짐, 학습 곡선 있음|
|ECharts (+ echarts-for-react)|기능이 매우 많은 종합 차트 라이브러리|기능 풍부, 인터랙션 다양|상대적으로 무거움|
|D3.js (직접)|가장 low-level, 명령형|뭐든 가능, 완전한 자유도|React 의 가상 DOM 과 직접 안 맞아 통합이 까다로움|

```
선택 기준:
  대시보드/통계 화면처럼 "흔한 차트 모양" 이면      → recharts (가장 쉽고 React 스러움) ⭐️
  아주 독특하고 복잡한 시각화가 필요하면            → visx 또는 D3 직접
  이미 Chart.js 에 익숙하거나 가벼움이 중요하면     → Chart.js
```

---

---

# 왜 "use client" 가 필요한가 ⭐️⭐️

```typescript
'use client';
```

```
차트(특히 ResponsiveContainer)는 "지금 이 컴포넌트가 화면에서 실제로 몇 픽셀 차지하는지" 를
브라우저에게 직접 물어봐서(ResizeObserver 등) 차트 크기를 맞춤
→ 이건 브라우저에만 있는 기능이라 서버에서는 실행할 수 없음
→ 그래서 차트를 그리는 컴포넌트는 항상 Client Component('use client') 여야 함

Server Component 와 같이 쓰는 법 (이 코드가 보여주는 패턴):
  데이터를 가져오는 부분(Prisma/API 호출)은 Server Component 에 그대로 두고
  "차트를 그리는 부분만" 별도 파일로 쪼개서 그 파일에만 'use client' 를 붙임
  → 부모(Server Component)가 이미 구한 숫자(visible, hidden)를 props 로 넘겨주기만 하면 됨

  AdminStatsChart({ visible, hidden }: {...})
  → 차트 컴포넌트는 "원본 데이터" 가 아니라 "이미 계산된 숫자" 만 받음
    (자세한 Server/Client 분리 기준은 [[Next_Data_Fetching]] 참고)
```

---

---

# 실전 — Pie 차트 한 줄씩 분해 ⭐️⭐️⭐️

```tsx
"use client";

import { Legend, Pie, PieChart, ResponsiveContainer, Tooltip } from "recharts";

const COLORS = ["#666666", "#D97706"];

export default function AdminStatsChart({
  visible,
  hidden,
}: {
  visible: number;
  hidden: number;
}) {
  const data = [
    { name: "공개", value: visible, fill: COLORS[0] },
    { name: "숨김", value: hidden, fill: COLORS[1] },
  ];

  const total = visible + hidden;

  if (total === 0) {
    return (
      <div className="flex h-48 items-center justify-center rounded-xl border border-neutral-200 bg-white text-sm text-neutral-500">
        표시할 데이터가 없습니다.
      </div>
    );
  }

  return (
    <div className="rounded-xl border border-neutral-200 bg-white p-4">
      <p className="text-sm font-medium text-neutral-900">공개 / 숨김 비율</p>
      <div className="mt-3 h-48 w-full">
        <ResponsiveContainer width="100%" height="100%" minWidth={0}>
          <PieChart>
            <Pie
              data={data}
              dataKey="value"
              nameKey="name"
              cx="50%"
              cy="50%"
              innerRadius={50}
              outerRadius={70}
            />
            <Tooltip />
            <Legend />
          </PieChart>
        </ResponsiveContainer>
      </div>
    </div>
  );
}
```

## data 배열 — 차트가 읽는 "재료"

```typescript
const data = [
  { name: "공개", value: visible, fill: COLORS[0] },
  { name: "숨김", value: hidden, fill: COLORS[1] },
];
```

```
recharts 는 "이런 모양의 객체 배열을 줘" 라고 기대하고, 그 배열을 그대로 그림으로 바꿔줌
  name   각 조각의 이름 (Tooltip/Legend 에 표시됨)
  value  각 조각의 크기 (이 값들의 비율대로 파이가 나뉨)
  fill   각 조각의 색 (직접 지정 — 아래 COLORS 설명 참고)

→ props 로는 원본 데이터(visible/hidden 숫자)만 받고
  recharts 가 원하는 모양(data 배열)으로 "변환" 하는 건 이 컴포넌트 안에서 직접 함
  (이게 바로 [[Next_API_Integration]] 의 "매퍼 함수" 와 똑같은 발상 —
   외부에서 받은 형태를 그대로 안 쓰고, 이 컴포넌트가 필요한 형태로 한 번 가공하는 것)
```

## COLORS + fill — 색을 어떻게 지정하는가

```typescript
const COLORS = ["#666666", "#D97706"];
// ...
{ name: "공개", value: visible, fill: COLORS[0] },   // 첫 조각: 회색
{ name: "숨김", value: hidden,  fill: COLORS[1] },   // 둘째 조각: 주황색
```

```
recharts 는 색을 자동으로 정해주는 기본 동작도 있지만
각 데이터 항목에 fill 속성을 직접 넣어주면 "이 조각은 이 색" 이라고 확실하게 고정할 수 있음
→ "공개=회색, 숨김=주황" 처럼 의미가 있는 색 배정을 하고 싶을 때는 이렇게 직접 지정하는 게 안전함
  (자동 배정에 맡기면 데이터 순서가 바뀔 때 색도 같이 바뀌어버릴 수 있음)
```

## Pie 컴포넌트 props

|prop|의미|
|---|---|
|`data`|그릴 데이터 배열|
|`dataKey`|data 의 각 항목에서 "크기" 로 쓸 필드 이름 (`"value"`)|
|`nameKey`|data 의 각 항목에서 "이름" 으로 쓸 필드 이름 (`"name"`)|
|`cx` / `cy`|파이의 중심 위치 (퍼센트 또는 px, `"50%"` = 정중앙)|
|`innerRadius`|안쪽 반지름 — 0 이면 꽉 찬 파이, 0 보다 크면 도넛 모양|
|`outerRadius`|바깥쪽 반지름 — 파이 전체 크기|

```
innerRadius={50} + outerRadius={70} 의 의미:
  반지름 50~70 사이만 색칠됨 → 가운데가 뚫린 "도넛" 형태
  innerRadius 를 0 으로 하면 도넛이 아니라 꽉 찬 파이가 됨
```

## ResponsiveContainer / Tooltip / Legend — 거의 안 건드리고 쓰는 것들

```
ResponsiveContainer   부모 div 의 실제 크기(width/height)를 측정해서 차트를 그 크기에 맞춤
                      width="100%" height="100%" → 부모 div(h-48 w-full)에 꽉 차게
Tooltip               마우스를 조각에 올리면 자동으로 값을 보여주는 말풍선 (커스터마이즈도 가능)
Legend                "■ 공개 ■ 숨김" 같은 범례를 차트 아래/옆에 자동 생성

세 개 다 그냥 컴포넌트로 넣어두기만 하면 알아서 동작함 (children 으로 PieChart 안에 배치)
```

## total === 0 — 빈 데이터 가드 ⭐️

```typescript
if (total === 0) {
  return ( /* "표시할 데이터가 없습니다" UI */ );
}
```

```
왜 필요한가:
  visible/hidden 이 둘 다 0(데이터 자체가 없음)이면
  파이 차트는 "그릴 게 없는" 상태가 되어 빈 원이나 이상한 모양으로 보일 수 있음
  → 차트를 그리기 전에 먼저 "그릴 데이터가 있는지" 확인하고
    없으면 차트 대신 안내 문구를 보여주는 게 사용자 경험상 더 나음
```

---

---

# 부모(Server Component)에서 사용하기 ⭐️

```tsx
// app/admin/page.tsx (Server Component)
import AdminStatsChart from './AdminStatsChart';

export default async function AdminPage() {
  const stats = await prisma.post.groupBy({
    by: ['hidden'],
    _count: { _all: true },
  });

  const visible = stats.find((s) => !s.hidden)?._count._all ?? 0;
  const hidden  = stats.find((s) => s.hidden)?._count._all ?? 0;

  return (
    <div>
      <h1>관리자 대시보드</h1>
      <AdminStatsChart visible={visible} hidden={hidden} />
    </div>
  );
}
```

```
역할 분리:
  AdminPage(Server)        DB 조회 + 숫자 계산까지 끝낸 "결과" 만 만듦
  AdminStatsChart(Client)  그 결과를 받아서 "그림으로 그리는 것" 만 책임짐
  → 데이터를 가져오는 책임과 그리는 책임이 깔끔하게 나뉨
```

---

---

# 다른 차트 타입 맛보기 — 기본 패턴은 동일

```tsx
// Bar 차트 — Pie 와 구조가 거의 같음
import { Bar, BarChart, XAxis, YAxis, CartesianGrid, Tooltip, ResponsiveContainer } from "recharts";

<ResponsiveContainer width="100%" height={300}>
  <BarChart data={data}>
    <CartesianGrid strokeDasharray="3 3" />
    <XAxis dataKey="name" />
    <YAxis />
    <Tooltip />
    <Bar dataKey="value" fill="#666666" />
  </BarChart>
</ResponsiveContainer>
```

```
공통 패턴:
  항상 ResponsiveContainer 로 감싸고
  안에 OOChart(Pie/Bar/Line) 를 두고
  그 안에 실제 도형(Pie/Bar/Line) + 보조 요소(Tooltip/Legend/XAxis/YAxis)를 나열

→ 차트 종류가 달라져도 "구성하는 방식" 자체는 거의 똑같아서
  Pie 차트를 한 번 이해하면 Bar/Line 도 비슷하게 바로 적용 가능
```

---

---

# 한눈에

```
recharts 핵심:
  <ResponsiveContainer> 로 감싸고 그 안에 <PieChart>/<BarChart> 등을 둠
  data 배열({name, value, ...})을 만들어서 넘기는 게 핵심 — recharts 가 나머지를 그림

"use client" 가 필요한 이유: ResponsiveContainer 가 브라우저에서 실제 크기를 측정하기 때문

컴포넌트 분리 패턴:
  Server Component (데이터 조회/계산) → props 로 숫자만 전달 → Client 차트 컴포넌트 (그리기)

차트 그리기 전 항상 "데이터가 비었을 때" 가드 추가 (total === 0 체크 등)
```

|상황|선택|
|---|---|
|흔한 대시보드 차트, React 와 자연스럽게|recharts ⭐️|
|아주 독특한 시각화|visx / D3 직접|
|가벼움 중요, Chart.js 익숙함|Chart.js + react-chartjs-2|