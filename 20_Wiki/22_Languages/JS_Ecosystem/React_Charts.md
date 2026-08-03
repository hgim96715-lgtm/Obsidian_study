---
aliases:
  - Nivo
  - chart library
  - PartialTheme
tags:
  - React
related:
  - "[[00_JS_Ecosystem_HomePage]]"
  - "[[NestJS_Prisma]]"
  - "[[NextJS_ServerClient]]"
  - "[[NestJS_StatsBucket]]"
---
# React_Charts — 차트 라이브러리 공통 패턴

> [!info] 
> 어떤 차트 라이브러리(Nivo, Recharts 등)를 쓰든 반복되는 패턴이 있다 — 버전이 바뀌면 타입 export 위치가 바뀔 수 있고, 모노레포에서는 transitive 의존성을 직접 import하려면 명시적으로 추가해야 하며, SVG 기반 차트는 보통 SSR이 안전하지 않다. 라이브러리 자체보다 이 패턴들이 더 오래 쓰인다.

---

# 라이브러리를 고를 때 보는 기준 ⭐️⭐️

```txt
어떤 차트 라이브러리를 비교하든 보통 이 기준들로 따짐:
  기본 미감 — 손 안 대도 봐줄 만한가, 커스텀이 많이 필요한가
  지원하는 차트 종류 — pie/bar/line 정도만인지, 캘린더 히트맵·방사형 등 다양한지
  React 지원 — 공식 패키지가 있는지
  테마/브랜드 통일 — 색상·폰트를 한곳(theme 객체)에서 관리할 수 있는지

이 판단 기준 자체는 어떤 라이브러리를 비교하든 동일하게 적용됨 — 결과(어느 라이브러리)만 프로젝트마다 다름
```

---

# 버전 간 타입 export 위치 변경 — 흔한 함정 ⭐️⭐️⭐️⭐️

```txt
예: Nivo 0.99에서는 Theme 타입이 @nivo/core가 아니라 @nivo/theming으로 옮겨감
  구버전/예전 튜토리얼: import type { Theme } from '@nivo/core'
  최신 버전:           import type { PartialTheme } from '@nivo/theming'

→ 라이브러리가 메이저/마이너 버전을 올리면서 타입 export 위치 자체가 바뀌는 일은 흔함
  (Prisma 6→7의 datasource url 위치 변경, React 19.2.10의 FormEvent→SubmitEvent 변경과 같은 종류)
  버전이 안 맞는 듀토리얼을 그대로 따라하면 "예제 그대로인데 안 됨" 상황의 흔한 원인이 됨
  → 에러가 나면 가장 먼저 설치된 버전과 보고 있는 문서/예제의 버전이 같은지부터 확인
```

```txt
PartialTheme처럼 "Partial"이 붙은 타입이 흔한 이유:
  차트의 theme prop은 보통 라이브러리의 거대한 기본 테마 객체를 "전체"가 아니라
  "일부 속성만" 덮어쓰는 용도로 씀 — 그래서 모든 필드가 옵셔널인 Partial 버전 타입을
  따로 제공하는 라이브러리가 많음 (TS의 Partial<T> 자체는 [[TS_Utility_Types]] 참고)
```

---

# pnpm 모노레포 — transitive 의존성은 직접 import 안 될 수 있음 ⭐️⭐️⭐️⭐️

```txt
상황: 차트 라이브러리의 메인 패키지(예: @nivo/pie)가 내부적으로 테마 패키지(예: @nivo/theming)를
의존하고 있어서, node_modules 안에는 이미 깔려있음

근데 내 코드에서 그 테마 패키지를 "직접" import하려고 하면(타입을 쓰려고 등) 에러가 날 수 있음
→ pnpm은 "내가 package.json에 직접 적은 의존성만 import 가능"하게 엄격히 격리함
  (npm/yarn은 이런 phantom dependency를 어느 정도 허용해서 같은 코드가 거기선 동작하기도 함)

해결: 그 패키지를 내 앱의 package.json에 "직접" 추가
  pnpm add @nivo/theming --filter web

→ "이미 설치돼 있는 것 같은데 import가 안 된다"는 모노레포 특유의 흔한 증상 —
  transitive로만 들어온 패키지를 직접 쓰려면 항상 명시적으로 추가해야 함
```

---

# CSS 변수를 차트 theme에 못 쓰는 이유 ⭐️⭐️⭐️

```txt
대부분의 SVG 기반 차트 라이브러리는 theme/colors prop에 var(--color-brand-primary) 같은
CSS 변수를 그대로 넣기 어려움 — 라이브러리가 그 값을 JS 객체로 다루기 때문에,
브라우저가 실제로 CSS 변수를 해석해주는 시점과 안 맞는 경우가 많음

해결: 브랜드 색상을 hex 값으로 한 번 더 정의해서 차트 전용 theme 파일에 둠
  ⚠️ 이렇게 되면 "진짜 정본"(CSS 변수 쪽)과 "차트용 복사본"(hex) 두 곳이 생기는 셈이라,
     브랜드 색이 바뀌면 두 곳을 같이 수정해야 함 — 어느 파일이 출처(정본)인지 주석으로 남겨두면 좋음
```

---

# SSR과 차트 — 클라이언트에서만 로드하기 ⭐️⭐️⭐️⭐️

```tsx
import dynamic from 'next/dynamic';

const PieChart = dynamic(() => import('@/components/charts/PieChart'), {
  ssr: false,
  loading: () => <p className="text-sm text-neutral-500">차트 불러오는 중…</p>,
});
```

```txt
대부분의 차트 라이브러리(특히 "반응형" 컨테이너를 쓰는 것들)는 실제 DOM 요소의 크기를
측정해서 그 크기에 맞춰 SVG를 그림 — 근데 서버에는 DOM 자체가 없음(SSR 환경)
→ 서버에서 렌더링하려고 하면 에러가 나거나, 크기를 못 구해서 이상하게 그려짐

next/dynamic(..., { ssr: false })로 클라이언트에서만 로드되게 만드는 게 일반적인 해법 —
"이 컴포넌트는 브라우저에만 존재하는 정보(DOM 크기 등)가 꼭 필요하다"는 신호가 보이면
이 패턴을 먼저 떠올릴 것 (서버에 브라우저 전역 객체가 없다는 일반론은 [[JS_BrowserAPI]] 참고)
```

---

# 반응형 차트 컨테이너 — 부모가 안 줄어드는 문제 ⭐️⭐️⭐️⭐️

```txt
차트를 flex/grid 레이아웃 안에 넣었는데 부모가 의도보다 넓어지거나 안 줄어드는 증상의 흔한 원인:

flex/grid 아이템의 기본 min-width는 "내용물 크기"임 — 안에 있는 SVG/차트가 어떤 크기 이하로는
안 줄어들게 설계돼 있다면, 그 차트를 담은 부모도 그 크기 밑으로는 줄어들 수 없게 됨

해결: 부모 쪽에 min-width: 0(Tailwind면 min-w-0)을 명시적으로 줘서
"이 아이템은 내용물 크기보다 더 작아질 수 있다"고 브라우저에 알려줘야 함

차트 래퍼 자체에는 보통 최소 높이(예: min-h-[12rem])를 줌 —
데이터가 로딩 중이거나 비어있을 때도 레이아웃이 안 흔들리게 하기 위해서
```

---

# 차트의 공통 UX 규칙 ⭐️⭐️⭐️

|상황|규칙|
|---|---|
|데이터가 비어있음|차트를 그냥 안 그리고, 안내 문구(텍스트)로 대체|
|로딩 중|차트 컴포넌트 안에서 직접 fetch하지 않고, 페이지/섹션 단위 스켈레톤으로 감싸기|
|접근성|`aria-label`, 범례 텍스트를 같이 제공 — 색상만으로 정보를 전달하지 않기|

```txt
"차트 안에서 fetch X"가 원칙인 이유: 차트 컴포넌트는 "데이터를 받아서 그리기"에만 집중하고,
데이터를 가져오는 책임(로딩/에러 상태 포함)은 그 차트를 쓰는 페이지/섹션 쪽에 두는 게 역할이 분명함
→ 같은 차트 컴포넌트를 다른 데이터 소스에도 재사용하기 쉬워짐
```

---

# 데이터 shape 정규화 — API 응답을 그대로 넘기지 않기 ⭐️⭐️⭐️

```txt
대부분의 차트 라이브러리는 자기만의 데이터 형식을 요구함
(예: { id, value, label? }[] 같은 형태) — API가 주는 응답 모양과 다른 경우가 흔함

→ 차트 컴포넌트에 API 응답을 그대로 넘기지 않고, mapper 함수에서 차트가 원하는
  shape으로 한 번 변환해서 넘기는 게 일반적 — API 응답 구조가 바뀌어도 차트 컴포넌트
  자체는 안 바뀌고 mapper만 고치면 되는 효과도 있음
  (API 응답 ↔ UI 타입을 변환하는 mapper 패턴 자체는 [[NextJS_API_Mapper]] 참고)

  날짜별/시간대별 통계처럼 "빈 구간도 0으로" 채워서 내려주는 백엔드 쪽 패턴은 [[NestJS_StatsBucket]] 참고 —
  그 노트에서 만든 배열이 정확히 여기서 말하는 "차트가 원하는 shape"으로 이미 정규화된 데이터임
```

---

# 한눈에

| 주제                    | 핵심                                                      |
| --------------------- | ------------------------------------------------------- |
| 라이브러리 선택 기준           | 미감 / 지원 차트 종류 / React 지원 / 테마 통일 방법                     |
| 버전별 타입 위치 변경          | 메이저 버전 업그레이드 시 타입 export 위치가 바뀔 수 있음 — 버전 확인 먼저         |
| `PartialTheme`류 타입    | 테마 prop은 보통 "일부만 덮어쓰기" 용도라 Partial 버전 타입이 따로 있는 경우가 많음  |
| pnpm + transitive 의존성 | 직접 import하려면 그 패키지를 내 앱에 직접 추가해야 함                      |
| CSS 변수 미지원            | theme에 못 넣으면 hex로 별도 정의 — 정본과 동기화 필요                    |
| SSR 위험                | DOM 크기 측정이 필요한 차트는 `next/dynamic(..., { ssr: false })`로 |
| 반응형 안 줄어듦             | 부모에 `min-width: 0` 필요 — flex/grid 아이템 기본값 때문            |
| 공통 UX 규칙              | 빈 데이터는 문구로, 로딩은 상위에서, 색만으로 정보 전달 금지                     |
| 데이터 shape             | API 응답을 그대로 넘기지 말고 mapper로 차트 형식에 맞게 변환                 |


```cardlink
url: https://awesome.cube.dev/tools/nivo
title: "nivo — a charting library"
description: "nivo examples, tutorials, compatibility, and popularity"
host: awesome.cube.dev
favicon: https://awesome.cube.dev/favicon-32x32.png
image: https://cubedev-blog-images.s3.us-east-2.amazonaws.com/482a86d6-d049-4d92-a0bf-0fcc5830476f.jpeg
```
