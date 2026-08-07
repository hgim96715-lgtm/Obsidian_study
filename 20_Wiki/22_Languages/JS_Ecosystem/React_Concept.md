---
aliases:
  - JSX
  - Virtual DOM
  - Concept
  - CSR
  - SSR
  - SPA
tags:
  - React
related:
  - "[[00_JS_Ecosystem_HomePage]]"
  - "[[NextJS_Concept]]"
  - "[[React_Component]]"
  - "[[NestJS_Concept]]"
  - "[[NextJS_Routing]]"
---
# React_Concept — React 핵심 개념

>[!info]
>React = UI를 컴포넌트 단위로 만드는 JavaScript 라이브러리. 
>SPA 방식으로 페이지 전환 시 새로고침 없이 화면만 교체한다. 
>상태(state)가 바뀌면 React가 변경된 부분만 다시 그린다.
> Next.js는 React 위에 라우팅·SSR을 얹은 프레임워크 → [[NextJS_Concept]]

---

# SPA란 — Single Page Application ⭐️⭐️⭐️⭐️

```txt
전통적인 방식 (MPA — Multi Page Application):
  링크 클릭 → 서버에 새 페이지 요청 → 서버가 HTML 전체를 응답
  → 브라우저가 페이지 전체를 다시 그림 (깜빡임, 느림)
  → 매번 서버에 요청

SPA — 단일 페이지 앱:
  처음에 HTML·JS를 한 번만 받음
  이후 링크 클릭 → 서버에 요청 안 함
  → JavaScript가 화면의 일부만 교체 (새로고침 없음)
  → 빠른 화면 전환, 앱처럼 느껴짐

  데이터가 필요하면? → API(JSON)만 서버에 요청
  → HTML 전체가 아닌 데이터만 받아서 JS가 화면에 반영
```

```txt
SPA의 장점:
  페이지 전환이 빠름 (새로고침 없음)
  앱처럼 부드러운 UX
  서버 부하 감소 (HTML이 아닌 JSON만 응답)

SPA의 단점:
  첫 로딩이 느림 (JS 번들을 전부 받아야 함)
  SEO 불리 (처음에 빈 HTML)
  → Next.js의 SSR이 이 단점을 보완 → [[NextJS_Concept]]
```

---

# React가 없던 시절 ⭐️⭐️⭐️⭐️

```txt
jQuery 시대:
  "id가 'count'인 요소의 텍스트를 숫자+1로 바꿔라"
  → DOM을 직접 조작

  document.getElementById('count').textContent = count + 1;
  document.getElementById('btn').addEventListener('click', handler);
  ...

문제:
  앱이 커질수록 "어떤 상태일 때 어떤 DOM을 어떻게 바꿔야 하는가"를
  개발자가 전부 직접 추적해야 함
  → 코드가 복잡해지고 버그가 늘어남
  → 상태와 UI가 불일치하는 문제 발생

React의 해결:
  "상태(state)를 바꾸면 UI는 React가 알아서 업데이트"
  개발자는 "이 상태일 때 UI가 이렇게 보여야 한다"만 선언
  DOM 직접 조작 없음
```

---

# React의 핵심 아이디어 ⭐️⭐️⭐️⭐️

```typescript
// React의 핵심:
// UI = f(state)
// "상태(state)를 함수에 넣으면 UI가 나온다"

function Counter() {
  const [count, setCount] = useState(0);
  //     ↑ 상태           ↑ 상태 변경 함수

  return (
    <div>
      <p>{count}</p>
      <button onClick={() => setCount(count + 1)}>+</button>
    </div>
  );
  // count가 바뀌면 → React가 이 컴포넌트를 다시 실행
  // → 새 JSX를 반환 → React가 바뀐 부분만 DOM에 반영
}
```

```txt
개발자가 하는 것:
  "count가 3일 때 UI가 이렇게 생겼어" → JSX로 선언

React가 하는 것:
  count가 바뀌면 컴포넌트를 다시 실행
  이전 UI와 새 UI를 비교 (Virtual DOM)
  바뀐 부분만 실제 DOM에 반영

개발자는 DOM을 직접 건드리지 않음
```

---

# 컴포넌트 ⭐️⭐️⭐️⭐️

```txt
컴포넌트 = UI를 만드는 함수
  입력: props (외부에서 전달받는 데이터)
  출력: JSX (화면에 그릴 내용)

  작은 컴포넌트들을 조합해서 큰 UI를 만듦
  레고 블록처럼
```

```typescript
// 컴포넌트 = JSX를 반환하는 함수
function Button({ label, onClick }: { label: string; onClick: () => void }) {
  return <button onClick={onClick}>{label}</button>;
}

// 사용
<Button label="저장" onClick={handleSave} />
<Button label="취소" onClick={handleCancel} />
// 같은 컴포넌트를 다른 props로 재사용
```

```txt
컴포넌트 이름은 반드시 대문자로 시작:
  <Button>  → React 컴포넌트 (대문자)
  <button>  → HTML 태그 (소문자)
  → React가 이걸로 구분

컴포넌트는 순수 함수처럼 동작해야 함:
  같은 props → 항상 같은 JSX
  렌더링 중에 외부를 변경하면 안 됨 (사이드이펙트 금지)
  사이드이펙트는 useEffect로 처리 → [[React_useEffect]]
```

---

# JSX ⭐️⭐️⭐️⭐️

```tsx
// JSX = JavaScript 안에서 HTML처럼 쓰는 문법
function Welcome({ name }: { name: string }) {
  return (
    <div className="wrapper">   {/* class 대신 className */}
      <h1>안녕하세요, {name}!</h1>   {/* {} 안에 JS 표현식 */}
      <p>{1 + 2}</p>             {/* 3 */}
    </div>
  );
}
```

```txt
JSX는 HTML이 아님:
  React.createElement() 호출로 변환됨
  <h1>제목</h1> → React.createElement('h1', null, '제목')

JSX 규칙:
  하나의 루트 요소만 반환 → 여러 개면 <> </> (Fragment)로 감싸기
  모든 태그는 닫아야 함 → <br /> (self-closing)
  class → className, for → htmlFor (JS 예약어 충돌 방지)
  {} 안에 JS 표현식 사용 가능 (문은 불가 — if 문 안 됨, 삼항 연산자 사용)
```

---

# State — 상태 ⭐️⭐️⭐️⭐️

```typescript
const [value, setValue] = useState(초기값);
//     ↑ 현재 값  ↑ 변경 함수
```

```txt
State:
  컴포넌트가 기억하는 값
  바뀌면 → React가 컴포넌트를 다시 렌더링

일반 변수와 다른 점:
  let count = 0;
  count = count + 1;   // 화면 안 바뀜 — React가 모름

  const [count, setCount] = useState(0);
  setCount(count + 1);  // 화면 바뀜 — React가 알고 다시 렌더링

State를 직접 수정하면 안 됨:
  state.name = '홍길동'     // ❌ React가 변경 감지 못함
  setState({ ...state, name: '홍길동' })  // ✅ 새 객체로 교체
```

---

# Props — 외부에서 전달받는 데이터 ⭐️⭐️⭐️⭐️

```tsx
// Props = 부모 → 자식으로 데이터 전달
function PostCard({ title, author, onClick }: {
  title:   string;
  author:  string;
  onClick: () => void;
}) {
  return (
    <div onClick={onClick}>
      <h2>{title}</h2>
      <p>{author}</p>
    </div>
  );
}

// 사용
<PostCard title="게시글 제목" author="홍길동" onClick={() => ...} />
```

```txt
Props vs State:
  Props → 부모가 내려주는 것 (읽기 전용, 바꾸면 안 됨)
  State → 컴포넌트 자신이 관리하는 것 (변경 가능)

  Props는 위에서 아래로만 흐름 (단방향 데이터 흐름)
  자식이 부모 데이터를 바꾸려면 → 부모가 콜백 함수를 props로 내려줌
```

---

# 렌더링 ⭐️⭐️⭐️⭐️

```txt
렌더링 = React가 컴포넌트 함수를 실행해서 JSX를 만드는 것

언제 렌더링이 발생하는가:
  ① 처음 화면에 나타날 때 (마운트)
  ② state가 바뀔 때 (setState 호출)
  ③ 부모가 다시 렌더링될 때 (props가 바뀌지 않아도)

  → React는 바뀐 부분만 실제 DOM에 반영 (전체 다시 그리지 않음)

마운트 / 언마운트:
  마운트 = 컴포넌트가 화면에 처음 나타남
  언마운트 = 컴포넌트가 화면에서 사라짐
  useEffect cleanup이 언마운트 시 실행됨 → [[React_useEffect]]
```

## Virtual DOM ⭐️⭐️⭐️

```txt
Virtual DOM = 실제 DOM의 가벼운 사본 (메모리에 있는 JS 객체)

렌더링 과정:
  ① 상태 변경 → 컴포넌트 함수 다시 실행 → 새 Virtual DOM 생성
  ② 이전 Virtual DOM과 새 Virtual DOM을 비교 (Diffing)
  ③ 실제로 바뀐 부분만 실제 DOM에 반영 (Reconciliation)

왜 Virtual DOM을 쓰는가:
  실제 DOM 조작은 비쌈 (레이아웃 계산, 화면 다시 그리기)
  Virtual DOM은 메모리에서 비교 → 최소한의 DOM 조작만
```

---

# React와 Next.js의 관계 ⭐️⭐️⭐️⭐️

```txt
React:
  UI 라이브러리
  컴포넌트, 상태, 렌더링을 담당
  라우팅, 서버 렌더링 기능 없음

Next.js:
  React 위에 만든 프레임워크
  React의 컴포넌트·상태를 그대로 사용
  + 파일 기반 라우팅 → [[NextJS_Routing]]
  + SSR/SSG → [[NextJS_Concept]]
  + 이미지 최적화, API Routes 등

  React만으로 충분한 경우:
    서버 렌더링 불필요한 앱 (관리자 도구, 대시보드)
    SEO가 중요하지 않은 경우

  Next.js를 쓰는 경우:
    SEO가 필요한 공개 페이지
    초기 로딩 속도가 중요한 경우
    풀스택 (API Routes 활용)
```