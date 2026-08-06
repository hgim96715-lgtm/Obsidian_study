---
aliases:
  - Lazy
  - 지연 로딩
  - dynamic
tags:
  - React
related:
  - "[[00_JS_Ecosystem_HomePage]]"
  - "[[React_Suspense]]"
---

# React_Lazy — 코드 스플리팅 · React.lazy

> [!info] 
> React.lazy() = 컴포넌트를 처음 필요할 때만 다운로드하는 지연 로딩.
>  초기 번들 크기를 줄여서 첫 화면 로딩 속도를 개선한다.
>   반드시 `<Suspense>`와 함께 사용해야 한다.
>    Next.js에서는 `dynamic()`이 같은 역할을 한다. 로딩 경계 → [[React_Suspense]]

---

# 번들이란 — 왜 크기가 문제인가 ⭐️⭐️⭐️⭐️

```txt
번들(bundle):
  React 앱을 배포하면 모든 JavaScript 코드가 하나(또는 몇 개)의 파일로 합쳐짐
  이 파일을 번들이라고 함

문제:
  앱이 커질수록 번들도 커짐
  사용자가 첫 페이지를 열면 → 번들 전체를 다운로드
  → 첫 화면이 뜨는 데 오래 걸림

  예시:
  홈페이지만 보러 온 사용자도
  관리자 대시보드, 차트 라이브러리, 에디터 코드를 전부 다운로드

코드 스플리팅(Code Splitting):
  번들을 여러 조각으로 나눔
  처음에 필요한 것만 먼저 로드
  나머지는 필요할 때 다운로드
  → 초기 로딩 빠름
```

---

# import() — 동적 import ⭐️⭐️⭐️⭐️

```typescript
// 정적 import — 파일 시작 시 항상 로드 (번들에 포함)
import { HeavyChart } from './HeavyChart';

// 동적 import — 필요할 때 로드 (별도 청크로 분리)
const { HeavyChart } = await import('./HeavyChart');
//                            ↑ Promise를 반환 — 파일이 다운로드될 때 resolve
```

```txt
import()의 특징:
  Promise를 반환 — 비동기로 파일을 가져옴
  브라우저가 해당 파일을 별도로 요청 (HTTP 요청)
  파일을 처음 요청할 때만 다운로드, 이후에는 캐시 사용

  번들러(Vite, Webpack)가 import()를 감지하면:
  해당 파일을 별도의 chunk 파일로 분리
  → main.js와 HeavyChart.chunk.js 로 나뉨
  → main.js가 먼저 로드되고, HeavyChart는 나중에 필요할 때 로드
```

---

# React.lazy() — 컴포넌트 지연 로딩 ⭐️⭐️⭐️⭐️

```typescript
import { lazy, Suspense } from 'react';

// ❌ 정적 import — 항상 번들에 포함
import HeavyEditor from './HeavyEditor';

// ✅ lazy — 처음 렌더링될 때 다운로드
const HeavyEditor = lazy(() => import('./HeavyEditor'));
//                   ↑ 함수를 전달 (즉시 실행 아님)
//                       ↑ default export 컴포넌트를 반환하는 Promise
```

```txt
lazy()가 하는 것:
  () => import('./HeavyEditor') 함수를 저장만 해둠 (실행 안 함)
  HeavyEditor가 처음 렌더링되려는 순간 → 그때 import() 실행
  파일 다운로드 중 → Suspense에 "준비 안 됐어" 신호 (Promise throw)
  다운로드 완료 → 렌더링

주의: default export만 지원
  lazy(() => import('./module'))
  → './module'이 default export 컴포넌트여야 함
  named export라면 re-export 필요:
  lazy(() => import('./module').then(m => ({ default: m.MyComponent })))
```

## Suspense와 반드시 함께

```tsx
const HeavyEditor = lazy(() => import('./HeavyEditor'));

function App() {
  return (
    <Suspense fallback={<div>에디터 로딩 중...</div>}>
      <HeavyEditor />
    </Suspense>
  );
}
```

```txt
Suspense 없이 lazy() 쓰면:
  다운로드 중 → React가 어디에 fallback을 보여줄지 모름 → 에러

Suspense가 하는 역할:
  lazy() 컴포넌트가 다운로드 중일 때 → fallback 표시
  다운로드 완료 → fallback 제거, 컴포넌트 표시
```

---

# 언제 lazy()를 쓰는가 ⭐️⭐️⭐️⭐️

```txt
✅ lazy()가 효과적인 경우:
  처음 화면에 안 보이는 컴포넌트
    → 모달, 사이드 패널, 드롭다운 내용
  특정 조건에서만 보이는 컴포넌트
    → 로그인 사용자만 보는 관리자 UI
    → 탭을 클릭해야 보이는 패널
  크고 무거운 라이브러리를 포함한 컴포넌트
    → 차트 (recharts, chart.js)
    → 코드 에디터 (monaco-editor)
    → 지도 (leaflet, google maps)

❌ lazy()가 불필요한 경우:
  첫 화면에서 바로 보이는 컴포넌트
    → Header, Footer, 메인 콘텐츠
    → 지연 로딩하면 첫 렌더에 깜빡임 발생
  작은 컴포넌트 (Button, Icon 등)
    → 파일 크기가 작아서 분리 이득이 없음
  이미 다른 chunk에 있는 컴포넌트
    → 중복 분리는 오히려 요청 수만 늘어남
```

---

# 실전 패턴

## 조건부 표시 컴포넌트

```tsx
const AdminPanel = lazy(() => import('./AdminPanel'));
const ReportModal = lazy(() => import('./ReportModal'));

function App() {
  const { user } = useAuth();
  const [showReport, setShowReport] = useState(false);

  return (
    <>
      <MainContent />

      {/* 어드민만 보는 패널 — 일반 사용자는 다운로드 안 함 */}
      {user?.role === 'admin' && (
        <Suspense fallback={null}>
          <AdminPanel />
        </Suspense>
      )}

      {/* 버튼 클릭 시에만 로드 */}
      {showReport && (
        <Suspense fallback={<ModalSkeleton />}>
          <ReportModal onClose={() => setShowReport(false)} />
        </Suspense>
      )}
    </>
  );
}
```

## 탭 패널

```tsx
const Tab1 = lazy(() => import('./tabs/Tab1'));
const Tab2 = lazy(() => import('./tabs/Tab2'));
const Tab3 = lazy(() => import('./tabs/Tab3'));

function TabbedPanel() {
  const [activeTab, setActiveTab] = useState(0);

  const tabs = [Tab1, Tab2, Tab3];
  const ActiveComponent = tabs[activeTab];

  return (
    <>
      <TabButtons activeTab={activeTab} onChange={setActiveTab} />
      <Suspense fallback={<TabSkeleton />}>
        <ActiveComponent />   {/* 탭 클릭할 때 해당 파일 다운로드 */}
      </Suspense>
    </>
  );
}
```

---

# Next.js — dynamic() ⭐️⭐️⭐️⭐️

```typescript
// Next.js에서는 React.lazy() 대신 dynamic() 사용
import dynamic from 'next/dynamic';

// React.lazy()와 동일한 효과
const HeavyChart = dynamic(() => import('./HeavyChart'));

// loading 옵션 — Suspense fallback
const HeavyChart = dynamic(() => import('./HeavyChart'), {
  loading: () => <ChartSkeleton />,
});

// ssr: false — 서버에서 렌더링하지 않음
// window, document 등 브라우저 API를 쓰는 컴포넌트
const BrowserOnlyMap = dynamic(() => import('./Map'), {
  ssr: false,   // 서버에서는 렌더링 안 함 → 클라이언트에서만
  loading: () => <MapSkeleton />,
});
```

```txt
dynamic() vs React.lazy():
  React.lazy()  → React 기본 기능, Suspense 필요
  dynamic()     → Next.js 전용, loading 옵션으로 Suspense 대체 가능

  Next.js에서는 dynamic()이 더 편리:
    ① loading 옵션으로 Suspense를 별도로 안 써도 됨
    ② ssr: false로 SSR 비활성화 가능 (브라우저 전용 라이브러리)
    ③ Next.js 최적화와 통합

  ssr: false를 쓰는 경우:
    leaflet(지도), canvas 기반 라이브러리 등
    서버에서 실행하면 window/document 없어서 에러나는 컴포넌트
    → [[NextJS_Concept]] 서버 환경 주의사항 참고
```

---

# 번들 분석 — 효과 확인

```bash
# Next.js — 번들 분석
pnpm add -D @next/bundle-analyzer

# 빌드 후 분석 리포트
ANALYZE=true pnpm build
```

```txt
번들 분석기로 확인할 것:
  어떤 파일이 얼마나 큰지
  lazy()로 분리됐는지 확인
  불필요하게 큰 라이브러리가 번들에 포함됐는지

lazy() 효과 확인:
  분리 전: main.js 500KB
  분리 후: main.js 350KB + HeavyChart.chunk.js 150KB
  → 첫 로딩: 350KB만 다운로드 (나머지는 차트 볼 때만)
```