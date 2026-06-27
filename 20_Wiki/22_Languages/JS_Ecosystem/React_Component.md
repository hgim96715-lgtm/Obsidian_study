---
aliases:
  - 함수형 컴포넌트
  - JSX
  - props
  - children
  - ComponentProps
tags:
  - React
related:
  - "[[00_JS_Ecosystem_HomePage]]"
  - "[[NextJS_UI_Types]]"
  - "[[NextJS_Data_Fetching]]"
  - "[[NextJS_API_Client]]"
  - "[[React_State]]"
---
# React_Component — 함수형 컴포넌트 & JSX

# 한 줄 요약

```
함수형 컴포넌트  = 함수 하나 = UI 조각 하나 (return 으로 JSX 반환)
JSX             = JS 안에서 HTML 비슷하게 쓰는 문법 (실제로는 함수 호출로 변환됨)
props           = 부모 → 자식으로 전달하는 데이터 (단방향)
children        = 태그 "사이"에 넣은 내용이 자동으로 들어오는 특별한 prop
```

---

---

# 함수형 컴포넌트란 ⭐️

```javascript
function App() {
  return <h1>Hello</h1>;
}

export default App;
```

```
규칙:
  함수 이름은 반드시 대문자로 시작 (PascalCase) ⭐️
    function app() { ... }   ❌ React 가 HTML 태그 <app> 으로 착각함
    function App() { ... }   ✅ 컴포넌트로 인식

  반드시 JSX(또는 null)를 return 해야 함
  export default 로 다른 파일에서 import 해서 씀
```

---

---

# JSX 기초 ⭐️⭐️

```js
import './App.css';

function App() {
  const title = "React JSX basics";
  const imageUrl = "https://images.unsplash.com/photo-...";

  return (
    <>
      <h1 className='title-style'>{title}</h1>
      <img src={imageUrl} alt="image" width={200} />
    </>
  );
}
export default App;
```

```
이 코드를 한 줄씩:

  import './App.css'
    JS 파일 안에서 CSS 파일을 직접 import — 번들러(Vite/webpack)가 처리해줌

  const title = "..."
    일반 JS 변수 — JSX 안에서 그대로 끼워 쓸 수 있음

  return ( <> ... </> )
    컴포넌트는 결국 "하나의" JSX 트리를 반환해야 함 → 그래서 전체를 괄호로 감쌈
    (자세한 이유는 아래 "최상위 요소는 하나여야 함" 참고)

  <h1 className='title-style'>{title}</h1>
    className   → class 가 아님 (아래에서 이유 설명)
    {title}     → JS 표현식을 JSX 안에 끼워넣는 문법

  <img src={imageUrl} alt="image" width={200} />
    img 처럼 닫는 태그가 없는 요소는 끝에 / 를 꼭 붙여야 함 (자기닫힘)
    width={200} → 숫자를 props 로 넘길 때도 {} 사용 (문자열 "200" 아님)
```

## {} — JS 표현식 끼워넣기 ⭐️

```javascriptreact
const name = '공이';
const age = 20;

<p>{name}</p>                    // 변수
<p>{age + 1}</p>                 // 연산
<p>{age >= 18 ? '성인' : '미성년'}</p>   // 삼항 연산자
<p>{items.length}개</p>          // 메서드 호출
<p>{items.map(i => <span key={i.id}>{i.name}</span>)}</p>   // 배열 → JSX 배열

// ❌ {} 안에 못 쓰는 것: if문, for문 같은 "문장(statement)"
// {if (age >= 18) { ... }}  ← 에러, JSX 의 {}는 "표현식(expression)"만 허용
```

```
{} 안에는 "값으로 평가되는 표현식"만 들어갈 수 있음
  if / for / let / const 같은 선언문·제어문은 안 됨
  → if 분기가 필요하면 삼항 연산자나 변수에 미리 계산해서 넣음
  (자세한 if/삼항/반복 비교는 [[JS_Control_Flow]] 참고)
```

## console.log 를 {} 안에 쓰고 싶을 때 ⭐️

```javascript
// 사실 이것도 "동작은 함" — console.log() 도 표현식이라서 {} 안에 들어갈 수 있음
return (
  <div>
    {console.log(title)}   {/* 콘솔에 찍히긴 함 */}
    <h1>{title}</h1>
  </div>
);
```

```
화면엔 안 보이는데 콘솔엔 찍히는 이유:
  console.log() 의 반환값은 undefined
  React 는 {} 안의 값이 null / undefined / boolean 이면 "아무것도 렌더링 안 함" 으로 처리
  → 화면엔 빈 칸이지만, 함수 호출 자체는 일어났으니 콘솔에는 정상적으로 찍힘
```

```
그래도 권장하지 않는 이유:
  1. 렌더링 함수는 "순수 함수" 여야 한다는 원칙이 있음
     ([[React_useEffect]] 의 "사이드 이펙트란" 과 같은 개념 — console.log 도 엄밀히는 사이드 이펙트)
     렌더링 결과(JSX)와 무관한 일을 렌더링 도중에 하는 것이기 때문
  2. 개발 모드의 StrictMode 는 컴포넌트를 의도적으로 두 번 렌더링함
     → {console.log(...)} 도 두 번 찍혀서 "왜 두 번 찍히지?" 하고 헷갈리게 됨
  3. 리렌더링될 때마다 매번 실행됨 → 자주 리렌더링되는 컴포넌트에 넣으면 콘솔이 도배됨
```

```javascript
// ✅ 더 나은 방법 — {} 안에 욱여넣지 않고, return 위에서 그냥 평범한 코드로
function App() {
  const title = "Hello";
  console.log(title);   // 일반 statement — {} 필요 없음

  return (
    <div>
      <h1>{title}</h1>
    </div>
  );
}
```

```
{} 는 "JSX 화면에 값을 보여주기 위한" 자리이고
console.log 는 "값을 화면에 보여주는" 게 목적이 아니라 "기록만 하는" 게 목적
→ 굳이 {} 안에 넣을 이유가 없음, return 전에 그냥 한 줄 쓰면 똑같이 작동하고 더 명확함
```

## className — class 대신 쓰는 이유 ⭐️

```javascriptreact
<div className="box">...</div>   // ✅
<div class="box">...</div>       // ❌ (경고는 뜨지만 동작은 함 — 권장 안 됨)
```

```
class 는 JavaScript 의 예약어(class 선언문)임
JSX 는 결국 JS 코드로 변환되기 때문에 속성 이름도 JS 식별자 규칙을 따라야 함
→ class 라는 이름을 속성으로 그대로 쓸 수 없어서 className 으로 대체

같은 이유로 HTML 의 for(label) → htmlFor 로 바뀜
나머지 속성들도 camelCase 로 통일: onclick → onClick, tabindex → tabIndex
```

## 자기닫힘 태그 — 왜 / 가 꼭 필요한가

```javascriptreact
<img src={imageUrl} />     // ✅ 닫는 태그 없는 요소는 끝에 / 필수
<img src={imageUrl}>       // ❌ JSX 에서는 에러

<br />                     // ✅
<input type="text" />      // ✅
```

```
HTML 에서는 <img> 처럼 닫는 슬래시를 생략해도 브라우저가 알아서 처리해주지만
JSX 는 XML 과 비슷한 엄격한 문법을 따름 → 닫는 태그가 없는 요소는 항상 self-closing(/) 필요
```

## 최상위 요소는 하나여야 함 ⭐️

```javascriptreact
// ❌ 최상위에 요소 2개 — 에러
function App() {
  return (
    <h1>제목</h1>
    <p>내용</p>
  );
}

// ✅ 하나의 부모로 감싸기
function App() {
  return (
    <div>
      <h1>제목</h1>
      <p>내용</p>
    </div>
  );
}
```

```
이유: 함수의 return 은 값을 "하나만" 반환할 수 있음
      JSX 도 결국 하나의 트리(객체) 로 변환되기 때문에 최상위 요소가 하나여야 함

div 로 감싸면 되긴 하지만, 의미 없는 div 가 실제 DOM 에 그대로 남는 단점이 있음
→ 그래서 등장한 게 바로 Fragment
```

---

---

# Fragment — <>...</> ⭐️

```javascript
function App() {
  return (
    <>
      <h1 className='title-style'>{title}</h1>
      <img src={imageUrl} alt="image" width={200} />
    </>
  );
}
```

```
Fragment 가 하는 일:
  여러 요소를 "묶기는" 하지만 실제 DOM 에는 아무 태그도 남기지 않음
  <div> 로 감쌌을 때 생기는 불필요한 래퍼(wrapper) 가 없어짐

<>...</>  는 <React.Fragment>...</React.Fragment> 의 축약 문법
```

```javascriptreact
// 배열을 렌더링할 때 key 가 필요하면 축약형(<>) 대신 풀네임을 써야 함
items.map(item => (
  <Fragment key={item.id}>
    <dt>{item.term}</dt>
    <dd>{item.desc}</dd>
  </Fragment>
))
// <>...</> 축약형에는 key 같은 props 를 줄 수 없음 ⚠️
```

---

---

# props — 컴포넌트에 데이터 전달 ⭐️

```javascriptreact
// 부모 → 자식으로 데이터 전달 (단방향)
function App() {
  return <Greeting name="공이" age={20} />;
}

function Greeting({ name, age }) {
  //              ↑ 구조분해로 바로 꺼내 씀 (props.name, props.age 와 동일)
  return <p>{name}님은 {age}살입니다</p>;
}
```

```
props 는 객체 하나로 전달됨:
  <Greeting name="공이" age={20} />
  → Greeting({ name: '공이', age: 20 }) 로 호출되는 것과 같음

읽기 전용:
  props 는 자식 컴포넌트에서 직접 수정하면 안 됨 (React 의 단방향 데이터 흐름 원칙)
  값을 바꿔야 하면 부모에서 state 로 관리하고, 변경 함수를 props 로 내려줌 ([[React_State]] 참고)
```

## 기본값 설정 (JS)

```javascriptreact
function Greeting({ name, age = 0 }) {
  //                       ↑ age 안 넘기면 0
  return <p>{name}님은 {age}살입니다</p>;
}

<Greeting name="공이" />   // age 는 0
```

---

---

# TypeScript 로 props 타입 지정하기 — `{Component}Props` 패턴 ⭐️⭐️⭐️

```
실무에서는 props 를 그냥 두지 않고 거의 항상 타입을 지정함 — 컴포넌트 이름 + Props 로 이름 짓는 게 표준 관례
```

```typescript
type GreetingProps = {
  name: string;
  age?: number;   // ? = 없어도 됨(optional)
};

function Greeting({ name, age = 0 }: GreetingProps) {
  //                                 ↑ 타입은 GreetingProps, 기본값은 구조분해에서(JS 와 동일)
  return <p>{name}님은 {age}살입니다</p>;
}
```

```
age?: number(타입에서 선택)와 age = 0(구조분해에서 기본값)은 같이 동작함:
  ? 가 없으면 호출하는 쪽에서 age 를 안 넘기면 TS 에러
  ? 가 있으면 안 넘겨도 되고, 그때 실제 값은 = 0 기본값이 채움
  → 둘 중 하나만 있어도 되지만, "선택값 + 기본값" 조합은 거의 항상 같이 씀
```

## 이름 붙은 타입 vs 인라인 타입 — 언제 뭘 쓰나

|방식|예시|언제|
|---|---|---|
|인라인|`function Greeting({ name }: { name: string })`|props 1~2개, 아주 간단할 때|
|`{Component}Props` (실무 표준) ⭐️|`type GreetingProps = {...}` 따로 선언|props 가 늘어날 때, 다른 곳(테스트 등)에서 타입을 재사용하고 싶을 때|

```
인라인이 props 2~3개를 넘어가면 함수 시그니처 한 줄이 너무 길어져서 가독성이 떨어짐
→ 거의 모든 실무 코드는 처음부터 {Component}Props 로 이름 붙여서 분리해두는 쪽을 택함
```

## 함수형(콜백) props 타이핑

```typescript
type ButtonProps = {
  label: string;
  onClick: () => void;              // 인자/반환값 없는 콜백
  onChange?: (value: string) => void;  // 인자 있는 콜백 + 선택값
};

function Button({ label, onClick }: ButtonProps) {
  return <button onClick={onClick}>{label}</button>;
}
```

```
콜백 타입은 "함수가 받는 인자 → 반환 타입" 화살표로 그대로 적음 — 이벤트 핸들러도 같은 원리 ([[React_Event]] 참고)
```

## children 도 Props 타입에 같이 포함

```typescript
type CardProps = {
  title: string;
  children: React.ReactNode;   // 태그 사이의 내용 (아래 "children" 섹션 참고)
};

function Card({ title, children }: CardProps) {
  return (
    <div className="card">
      <h2>{title}</h2>
      {children}
    </div>
  );
}
```

## 이미 만들어둔 도메인 타입을 그대로 props 타입으로 재사용 ⭐️⭐️⭐️

```
직접 { id, title, ... } 를 일일이 다시 적지 않고, 이미 정의해둔 타입을 그대로 가져다 씀
```

```typescript
import type { Recommendation } from '@/lib/types';

type FeedCardProps = {
  recommendation: Recommendation;
};

export function FeedCard({ recommendation }: FeedCardProps) {
  return <h3>{recommendation.title}</h3>;
}
```

```
이 Recommendation 타입이 어디서 왔는지, 어떻게 설계하는지는 [[NextJS_UI_Types]] 참고
api.ts → apiTypes → mapper → 이 타입으로 변환되는 전체 흐름은 [[NextJS_API_Client]] 참고
→ 그 흐름의 "마지막 도착점" 이 정확히 여기, 컴포넌트의 props 타입임
```

---

---

# children — 태그 "사이"의 내용 ⭐️

```javascript
function Card({ children }) {
  return <div className="card">{children}</div>;
}

// 사용하는 쪽
function App() {
  return (
    <Card>
      <h2>제목</h2>
      <p>카드 안 내용</p>
    </Card>
  );
}
```

```
children 은 일반 props 와 동일하지만 전달 방식이 다름:
  <Card name="x" />              → props.name 으로 전달
  <Card>여기 내용</Card>          → props.children 으로 전달 (태그 사이에 쓴 것)

children 이 유용한 경우:
  레이아웃/래퍼 컴포넌트 (Card, Modal, Layout 등)
  "안에 뭐가 들어갈지는 모르지만 감싸는 틀은 똑같다" 는 상황
```

## children 의 타입 — React.ReactNode ⭐️⭐️

```typescript
function Card({ children }: { children: React.ReactNode }) {
  return <div className="card">{children}</div>;
}
```

```
React.ReactNode 가 의미하는 것:
  "React 가 화면에 그릴 수 있는 모든 것" 을 통째로 가리키는 타입
  → JSX 엘리먼트(<div>...</div>) / 문자열 / 숫자 / 배열 / null / undefined / Fragment(<>...</>) 등
    이 전부 React.ReactNode 안에 포함됨

왜 children 에는 항상 이 타입을 쓰는가:
  children 으로 들어올 수 있는 건 위 경우 전부 다임
    <Card>제목</Card>              ← 문자열
    <Card>{count}</Card>           ← 숫자
    <Card><h2>제목</h2><p>내용</p></Card>  ← 여러 JSX 엘리먼트
    <Card>{isVisible && <p>...</p>}</Card> ← null 일 수도 있음 (조건부 렌더링)
  → 이 모든 경우를 다 받아낼 수 있는 가장 넓은 타입이 React.ReactNode

JSX.Element 와 헷갈리지 않기:
  JSX.Element  <div>...</div> 처럼 "JSX 문법으로 직접 만든 결과물 하나" 만 가리키는 좁은 타입
  ReactNode    그것보다 훨씬 넓음 — 문자열/숫자/null/배열까지 다 포함
  → children 처럼 "뭐가 들어올지 모르는" props 는 거의 항상 ReactNode 를 씀
    (JSX.Element 로 좁게 쓰면 문자열만 넘겨도 타입 에러가 남)
```

```
실전 — Next.js layout.tsx 가 항상 이 타입을 쓰는 이유:
  export default function RootLayout({ children }: { children: React.ReactNode }) {
    return <html><body>{children}</body></html>;
  }
  → children 자리에 그 레이아웃 밑의 모든 page.tsx 결과물이 통째로 들어옴
  → 그게 단일 페이지든, 여러 컴포넌트로 구성된 복잡한 트리든 다 받아내야 해서
    역시 가장 넓은 타입인 React.ReactNode 를 씀
```

---

---

# 컴포넌트 분리 ⭐️

```
하나의 컴포넌트가 너무 커지거나, 같은 UI 가 반복되면 분리함
```

```javascript
// 분리 전 — App 하나에 다 들어있음
function App() {
  const title = "React JSX basics";
  const imageUrl = "https://...";

  return (
    <>
      <h1 className='title-style'>{title}</h1>
      <img src={imageUrl} alt="image" width={200} />
    </>
  );
}

// 분리 후 — 역할별로 나눔
function Title({ text }) {
  return <h1 className='title-style'>{text}</h1>;
}

function FeaturedImage({ src }) {
  return <img src={src} alt="image" width={200} />;
}

function App() {
  const title = "React JSX basics";
  const imageUrl = "https://...";

  return (
    <>
      <Title text={title} />
      <FeaturedImage src={imageUrl} />
    </>
  );
}
```

```
언제 분리하나:
  같은 마크업이 여러 곳에서 반복될 때 (재사용)
  한 컴포넌트가 너무 길어져서 한눈에 안 들어올 때 (가독성)
  데이터 페칭과 인터랙션처럼 책임이 다른 부분이 섞여 있을 때
    (Server/Client Component 분리도 같은 원리 → [[NextJS_Data_Fetching]] 의 "컴포넌트 분리 패턴" 참고)

너무 잘게 쪼개는 것도 주의:
  컴포넌트 하나당 역할이 너무 작으면 파일 왔다갔다하며 읽기 더 힘들어짐
  "재사용되는가" 또는 "독립적인 의미 단위인가" 를 기준으로 판단
```

---

---

# 한눈에

```
함수형 컴포넌트:
  함수 이름 PascalCase 필수
  JSX(또는 null) 를 return

JSX 핵심 규칙:
  {} 안에는 표현식만 (if/for 같은 문장 X)
  class → className / for → htmlFor / 속성은 camelCase
  닫는 태그 없는 요소는 / 로 자기닫힘 (<img />)
  최상위 요소는 반드시 하나 → 여러 개면 <>...</> 로 감싸기

props / children:
  props      부모 → 자식 데이터 전달 (읽기 전용, 객체로 전달됨)
  children   태그 "사이"에 쓴 내용 → 레이아웃/래퍼 컴포넌트에 유용, 타입은 React.ReactNode

TypeScript props 타이핑:
  {Component}Props 로 이름 붙여서 따로 선언하는 게 실무 표준 (인라인은 아주 간단할 때만)
  optional(?) + 구조분해 기본값(=) 을 같이 써서 선택값 처리
  이미 설계해둔 도메인 타입(Post, Recommendation 등)을 그대로 가져다 props 타입으로 재사용 가능
    → 그 타입의 설계 자체는 [[NextJS_UI_Types]], API 에서 거기까지 오는 흐름은 [[NextJS_API_Client]]

컴포넌트 분리 기준: 재사용되는가 / 너무 길어졌는가 / 책임이 섞여 있는가
```