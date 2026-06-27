---
aliases:
  - React 개념
  - 리액트란
  - Virtual DOM
  - JSX
  - SPA
  - createRoot
tags:
  - React
related:
  - "[[00_JS_Ecosystem_HomePage]]"
  - "[[NextJS_Concept]]"
  - "[[React_Vite]]"
  - "[[React_Component]]"
---
# React_Concept — React 란

```txt
React = UI 를 만들기 위한 JavaScript 라이브러리
Facebook(Meta) 이 만들고 오픈소스로 공개
컴포넌트 단위로 UI 를 쪼개서 조립하는 방식
```

---

---

# React 가 나온 이유 ⭐️

```txt
기존 방식 (Vanilla JS / jQuery):
  DOM 을 직접 조작
  document.getElementById('title').innerText = '변경'
  상태가 많아질수록 코드가 복잡해지고 추적이 어려움

React 의 해결:
  상태(state) 가 바뀌면 UI 가 자동으로 업데이트
  개발자는 "어떻게 바꿀지" 가 아닌 "무엇을 보여줄지" 에 집중
  → 선언형(Declarative) 프로그래밍
```

---

---

# 설치하기 ⭐️

```bash
# Vite 로 시작 (현재 권장 방식) ⭐️
pnpm create vite my-app --template react-ts
cd my-app
pnpm install or  npm install
pnpm dev or npm run dev

# Next.js 로 시작 (SSR/라우팅까지 한 번에 필요할 때)
pnpm create next-app my-app

# ⚠️ create-react-app(CRA) 는 2025년 공식 deprecated
# → 신규 프로젝트에는 권장되지 않음
```

```txt
어떤 걸로 시작할지:
  순수 React(SPA) 만 필요          → Vite
  SSR / 폴더 기반 라우팅까지 필요   → Next.js
  → [[React_Vite]] / [[Next_Concept]] 비교 참고
```

---

---

# 프로젝트 구조 & 진입점 ⭐️

```txt
my-app/
├── index.html         ← 브라우저가 맨 처음 받는 HTML (진입점)
├── src/
│   ├── main.jsx        ← React 가 시작되는 자바스크립트 코드
│   ├── App.jsx          ← 최상위 컴포넌트
│   └── ...
└── package.json
```

```html
<!-- index.html -->
<body>
  <div id="root"></div>
  <!--      ↑ React 가 UI 를 그려넣을 빈 그릇 -->
  <script type="module" src="/src/main.jsx"></script>
</body>
```

```jsx
// src/main.jsx — React 앱이 실제로 시작되는 지점 ⭐️
import { StrictMode } from 'react';
import { createRoot } from 'react-dom/client';
import App from './App.jsx';

createRoot(document.getElementById('root')).render(
  <StrictMode>
    <App />
  </StrictMode>,
);
```

```txt
실행 흐름 (위에서 아래로):

  1. 브라우저가 index.html 을 불러옴
     → <div id="root"></div> 라는 빈 박스만 있는 상태

  2. <script src="/src/main.jsx"> 가 실행됨

  3. document.getElementById('root')
     → 그 빈 박스(div) 를 자바스크립트로 찾음

  4. createRoot(그 박스)
     → "여기에 React 로 UI 를 그릴 거야" 라고 React 에게 알려주는 자리 생성
     → React 18+ 의 새로운 렌더링 방식 (이전엔 ReactDOM.render 사용)

  5. .render(<App />)
     → 실제로 App 컴포넌트(전체 앱)를 그 박스 안에 그려넣음

  결과: 빈 div 였던 곳에 우리가 만든 화면 전체가 채워짐
```

## StrictMode 가 뭔지 ⭐️

```txt
<StrictMode> 로 감싸는 이유:
  개발 중에만 동작하는 "검사 모드" (빌드 결과물엔 영향 없음)
  컴포넌트를 의도적으로 두 번 렌더링해서
    → 부작용(side effect) 이 있는 잘못된 코드를 미리 발견하게 도와줌
  사용 안 해야 할 옛 API 사용 시 경고 표시

  실제 화면엔 영향 없음 — 순수히 "개발할 때 도움 주는 도구"
  뺄 수도 있지만 보통 그대로 둠 (버그 조기 발견에 유용)
```

```bash
App.jsx 는 또 무엇인가:
  실제 UI 를 구성하는 첫 번째(최상위) 컴포넌트
  여기서부터 다른 컴포넌트들을 import 해서 조립해나감
#  → [[React_Component]] 참고
```

---

---

# 컴포넌트 기반 ⭐️

```txt
UI 를 독립적인 조각(컴포넌트) 으로 쪼개서 조립
각 컴포넌트는 자신의 상태와 UI 를 관리

장점:
  재사용   Button 컴포넌트 하나로 여러 곳에 사용
  분리     컴포넌트별로 독립적으로 개발 / 테스트
  유지보수  특정 컴포넌트만 수정하면 됨
```

```jsx
// 컴포넌트 = 함수 (JSX 반환)
function Button({ label, onClick }) {
  return <button onClick={onClick}>{label}</button>;
}

// 조립
function App() {
  return (
    <div>
      <Button label="저장" onClick={handleSave} />
      <Button label="취소" onClick={handleCancel} />
    </div>
  );
}
```

---

---

# Virtual DOM ⭐️

```txt
DOM(Document Object Model):
  브라우저가 HTML 을 파싱해서 만드는 트리 구조
  직접 조작하면 느림 (렌더링 비용 높음)

Virtual DOM:
  React 가 메모리 안에 유지하는 가상의 DOM 복사본
  상태 변경 → 새 Virtual DOM 생성
  이전 Virtual DOM 과 비교 (diffing)
  실제로 바뀐 부분만 실제 DOM 에 반영 (reconciliation)
  → 불필요한 DOM 조작 최소화 → 성능 향상
```

```txt
흐름:
  state 변경
    ↓
  새 Virtual DOM 생성
    ↓
  이전 Virtual DOM 과 비교 (diff)
    ↓
  변경된 부분만 실제 DOM 업데이트
```

---

---

# JSX ⭐️

```txt
JSX = JavaScript + XML
JavaScript 안에서 HTML 처럼 UI 를 작성하는 문법
브라우저가 직접 이해 못 함 → Babel/Vite 가 JavaScript 로 변환
```

```jsx
// JSX
const element = <h1 className="title">Hello</h1>;

// 변환 후 (브라우저가 실제로 이해하는 형태)
const element = React.createElement('h1', { className: 'title' }, 'Hello');
```

```txt
JSX 규칙:
  class → className   (JS 예약어 충돌)
  for   → htmlFor
  반드시 하나의 루트 요소로 감싸기
    → <div> 또는 <> (Fragment) 사용
  표현식은 {} 안에
    → {name} / {1 + 1} / {isLoggedIn ? '로그아웃' : '로그인'}
```

---

---

# CSR vs SSR ⭐️

```txt
CSR (Client Side Rendering):
  브라우저가 빈 HTML 을 받고
  JavaScript 를 다운로드·실행해서 화면을 그림
  → 첫 로딩이 느림 / SEO 불리
  → React 기본 방식 (Vite 등)

SSR (Server Side Rendering):
  서버가 이미 완성된 HTML 을 만들어서 브라우저에 전송
  → 첫 로딩이 빠름 / SEO 유리
  → Next.js 가 지원

SSG (Static Site Generation):
  빌드 시점에 HTML 을 미리 생성
  → 가장 빠름 / 정적 콘텐츠에 적합
  → Next.js 가 지원
```

| |CSR|SSR|SSG|
|---|---|---|---|
|렌더링 시점|브라우저|요청마다 서버|빌드 시|
|첫 로딩|느림|빠름|가장 빠름|
|SEO|불리|유리|유리|
|사용|React SPA (Vite)|Next.js|Next.js|

---

---

# SPA vs MPA

```txt
SPA (Single Page Application):
  HTML 파일 1개 / JavaScript 로 화면 전환
  페이지 이동 시 새로고침 없음
  React / Vue / Angular

MPA (Multi Page Application):
  페이지마다 별도 HTML
  페이지 이동 시 새로고침
  전통적인 웹사이트
```

---

---

# React vs NestJS ⭐️

```txt
React:
  브라우저에서 실행 (프론트엔드)
  UI 렌더링 담당
  사용자가 보는 화면

NestJS:
  서버에서 실행 (백엔드)
  API / DB / 비즈니스 로직 담당
  데이터 처리

둘의 관계:
  React → NestJS API 호출 → 데이터 받아서 화면 표시
  React 는 클라이언트 / NestJS 는 서버
```

---

---

# 한눈에

```txt
설치:    pnpm create vite my-app --template react-ts (또는 Next.js)
진입점:   index.html → main.jsx → App.jsx
         <div id="root"> 에 createRoot().render(<App />) 로 채워넣음

StrictMode:  개발 중 버그 조기 발견용 (운영 빌드엔 영향 없음)

Virtual DOM:  실제 DOM 직접 조작 대신 메모리에서 비교 후 변경분만 반영
JSX:          className(class 아님) / htmlFor(for 아님) / 루트 하나로 감싸기
CSR/SSR/SSG:  React 기본=CSR / Next.js=SSR·SSG 지원
```