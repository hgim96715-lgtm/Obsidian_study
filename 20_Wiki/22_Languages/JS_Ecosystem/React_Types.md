---
aliases:
  - Types
  - children
  - ReactNode
  - 이벤트타입
  - Fragment
  - Ref
tags:
  - React
related:
  - "[[00_JS_Ecosystem_HomePage]]"
  - "[[TS_Generics]]"
  - "[[TS_Type_Guards]]"
---
# React_Types — React TypeScript 타입

>[!info]
>React 컴포넌트에서 자주 쓰는 TypeScript 타입 모음. 
>`children?: ReactNode`, 이벤트 타입(`ChangeEvent`, `MouseEvent`), Ref 타입, 컴포넌트 타입. 
>`<Fragment key={id}>`는 map() 렌더링에서 DOM 노드 없이 key를 붙일 때 사용. 
>순수 TypeScript 타입 → [[TS_Generics]] · [[TS_Type_Guards]]

---

# children — 컴포넌트 안에 들어가는 내용 ⭐️⭐️⭐️⭐️

```tsx
// 여는 태그와 닫는 태그 사이의 내용 = children
<Button>클릭하세요</Button>      // children = "클릭하세요" (문자열)

<Layout>
  <Header />                    // children = JSX 엘리먼트들
  <Main />
</Layout>
```

```txt
children = 컴포넌트가 "감싸는" 내용
  HTML의 <div> 안에 뭐든 넣을 수 있는 것처럼
  React 컴포넌트도 children을 선언하면 안에 뭐든 받을 수 있음

children을 받지 않는 컴포넌트:
  <Input value={...} onChange={...} />  ← 스스로 닫는 태그
  props에 children을 선언 안 하면 됨
```

---

# React.ReactNode — 렌더링 가능한 모든 것 ⭐️⭐️⭐️⭐️

```typescript
// ReactNode = 렌더링 가능한 모든 타입의 union
//   ReactElement → <div />, <MyComponent /> 같은 JSX
//   string       → 텍스트 노드
//   number       → 숫자 (자동으로 문자열로 렌더링)
//   boolean      → 렌더링 안 됨 (조건부 렌더링의 false)
//   null         → 아무것도 안 보임
//   undefined    → 아무것도 안 보임
//   ReactNode[]  → 위의 것들의 배열 (재귀)

// 전부 유효한 children
<Box>클릭하세요</Box>                            // string ✅
<Box>{42}</Box>                               // number ✅
<Box><Icon /></Box>                           // ReactElement ✅
<Box>{null}</Box>                             // null ✅ (아무것도 안 보임)
<Box>{isLoaded && <Icon />}</Box>             // false | ReactElement ✅
<Box>{items.map(i => <Item key={i.id} />)}</Box>  // 배열 ✅
```

## children?: React.ReactNode 선언 ⭐️⭐️⭐️⭐️

```typescript
import { type ReactNode } from 'react';

type CardProps = {
  title:     string;
  children?: ReactNode;   // ? = 선택적 (없어도 됨)
};

function Card({ title, children }: CardProps) {
  return (
    <div>
      <h2>{title}</h2>
      <div>{children}</div>   {/* children 없으면 빈 div */}
    </div>
  );
}

// 사용
<Card title="제목" />                     // children 없음 ✅
<Card title="제목"><p>내용</p></Card>     // children 있음 ✅
```

```txt
children?: ReactNode (? 있음) — 선택적, 없어도 됨
children: ReactNode  (? 없음) — 필수, 반드시 안에 뭔가 넣어야 함
```

## PropsWithChildren ⭐️⭐️⭐️

```typescript
// children?: ReactNode를 자동으로 추가해주는 유틸 타입
type CardProps = React.PropsWithChildren<{
  title: string;
}>;
// = { title: string; children?: ReactNode }
```

## ReactNode vs JSX.Element ⭐️⭐️⭐️

```typescript
// JSX.Element — JSX 엘리먼트만
const a: JSX.Element = <div />;    // ✅
const b: JSX.Element = 'hello';    // ❌ 문자열 안 됨
const c: JSX.Element = null;       // ❌ null 안 됨

// ReactNode — 렌더링 가능한 모든 것
const d: ReactNode = <div />;      // ✅
const e: ReactNode = 'hello';      // ✅
const f: ReactNode = null;         // ✅
const g: ReactNode = 42;           // ✅
```

```txt
언제 뭘 쓰는가:
  children?: ReactNode       감싸는 컴포넌트 → 거의 항상 이것
  icon?: ReactElement        아이콘처럼 JSX만 받을 때
  label?: string             텍스트만 받을 때
  render?: () => ReactNode   렌더 함수 패턴
```

---

# 이벤트 타입 ⭐️⭐️⭐️⭐️

```typescript
// input onChange
onChange: (e: React.ChangeEvent<HTMLInputElement>) => void
// e.target.value → 입력값

// select onChange
onChange: (e: React.ChangeEvent<HTMLSelectElement>) => void

// button onClick
onClick: (e: React.MouseEvent<HTMLButtonElement>) => void

// form onSubmit
onSubmit: (e: React.FormEvent<HTMLFormElement>) => void
// → e.preventDefault() 필수

// 키보드
onKeyDown: (e: React.KeyboardEvent<HTMLInputElement>) => void
// e.key → 'Enter', 'Escape' 등
```

```typescript
// 실전 패턴
function SearchInput() {
  const [value, setValue] = useState('');

  const handleChange = (e: React.ChangeEvent<HTMLInputElement>) => {
    setValue(e.target.value);
  };

  const handleKeyDown = (e: React.KeyboardEvent<HTMLInputElement>) => {
    if (e.key === 'Enter') doSearch(value);
    if (e.key === 'Escape') setValue('');
  };

  return <input onChange={handleChange} onKeyDown={handleKeyDown} />;
}
```

```txt
React.ChangeEvent<HTMLInputElement>:
  React.ChangeEvent = React의 이벤트 래퍼 타입
  <HTMLInputElement> = 어떤 HTML 요소에서 발생했는지

  e.target → 이벤트가 발생한 요소 (HTMLInputElement)
  e.target.value → input의 현재 값
  e.target.checked → checkbox의 체크 여부

이벤트 타입을 모를 때:
  핸들러에서 e에 마우스를 올려보면 TypeScript가 타입 추론해줌
  onChangeHandler의 인자 타입을 확인
```

---

# Ref 타입 ⭐️⭐️⭐️

```typescript
// DOM 요소를 직접 참조할 때
const inputRef = useRef<HTMLInputElement>(null);
const divRef   = useRef<HTMLDivElement>(null);
const btnRef   = useRef<HTMLButtonElement>(null);

// 사용
<input ref={inputRef} />
inputRef.current?.focus()     // null 체크 후 사용
inputRef.current?.value       // input 현재 값
```

```txt
useRef<HTMLInputElement>(null):
  초기값 null — 컴포넌트 마운트 전에는 DOM이 없으니까
  ref.current가 null일 수 있어서 ?.로 접근

자주 쓰는 HTML 요소 타입:
  HTMLInputElement     input
  HTMLTextAreaElement  textarea
  HTMLDivElement       div
  HTMLButtonElement    button
  HTMLFormElement      form
  HTMLSelectElement    select
  HTMLImageElement     img
  HTMLCanvasElement    canvas

→ [[React_useRef]] 상세
```

---

# 컴포넌트 타입 ⭐️⭐️⭐️

```typescript
// 컴포넌트를 props로 받을 때
type Props = {
  icon:   React.ComponentType;              // props 없는 컴포넌트
  Icon:   React.ComponentType<{ size: number }>; // size prop 있는 컴포넌트
  render: () => ReactNode;                  // 렌더 함수 패턴
};

// 사용
<MyComponent icon={HomeIcon} />
<MyComponent Icon={HomeIcon} />   // 대문자 → JSX로 직접 렌더링 가능
// <Icon size={24} />
```

```typescript
// React.FC — 함수 컴포넌트 타입 (잘 안 씀)
const MyComponent: React.FC<Props> = ({ title }) => { ... };

// 요즘은 그냥 함수 선언 + return 타입 생략이 더 일반적
function MyComponent({ title }: Props) { ... }
```

```txt
React.FC를 잘 안 쓰는 이유:
  children을 자동으로 포함했지만 React 18부터 제거됨
  제네릭 컴포넌트 작성이 불편함
  → 그냥 함수 선언 방식이 더 유연
```

---

# Fragment — 불필요한 DOM 없이 여러 요소 반환 ⭐️⭐️⭐️⭐️

```tsx
// ❌ React 컴포넌트는 하나의 루트 요소를 반환해야 함
function Component() {
  return (
    <h1>제목</h1>
    <p>내용</p>   // ❌ 에러
  );
}

// ❌ div로 감싸면 불필요한 DOM 노드 생성
function Component() {
  return (
    <div>   // ← 레이아웃에 영향을 줄 수 있는 불필요한 div
      <h1>제목</h1>
      <p>내용</p>
    </div>
  );
}

// ✅ Fragment — DOM 노드 없이 여러 요소를 하나로 묶음
function Component() {
  return (
    <>
      <h1>제목</h1>
      <p>내용</p>
    </>
  );
}
```

```txt
Fragment란:
  "여러 JSX 요소를 묶지만 실제 DOM에는 아무것도 추가하지 않는 것"
  React는 하나의 루트 요소 반환 규칙이 있음
  div로 감싸면 HTML 구조가 지저분해지거나 flex/grid가 깨질 수 있음
  → Fragment로 감싸면 DOM에 흔적 없이 규칙 충족
```

## `<>` 단축형 vs `<Fragment>` ⭐️⭐️⭐️⭐️

```tsx
import { Fragment } from 'react';

// <> 단축형 — 대부분의 경우
function Simple() {
  return (
    <>
      <dt>이름</dt>
      <dd>홍길동</dd>
    </>
  );
}

// <Fragment key={...}> — key가 필요한 경우에만
function List({ comments }: { comments: Comment[] }) {
  return (
    <>
      {comments.map(comment => (
        <Fragment key={comment.id}>   {/* key를 붙여야 해서 Fragment 사용 */}
          <CommentItem comment={comment} />
          <CommentDivider />
        </Fragment>
      ))}
    </>
  );
}
```

```txt
<> 와 <Fragment>의 차이:

  <>...</>       → props 없음 (key 포함) — 대부분 이걸 씀
  <Fragment>     → key prop 붙일 수 있음

key가 필요한 이유:
  map()으로 목록을 렌더링할 때 React가 각 항목을 추적하기 위해 필요
  <div key={id}> 처럼 감싸면 불필요한 div가 DOM에 생김
  → <Fragment key={id}>로 DOM 없이 key만 붙임
```

## 실전 — dl/dt/dd 패턴

```tsx
// 정의 목록(dl) — dt와 dd가 쌍으로 나와야 함
// <div>로 감싸면 dl > div > dt, dd 구조 → HTML 규칙 위반

function DefinitionList({ items }: { items: { term: string; desc: string }[] }) {
  return (
    <dl>
      {items.map(item => (
        <Fragment key={item.term}>
          <dt>{item.term}</dt>
          <dd>{item.desc}</dd>
        </Fragment>
      ))}
    </dl>
  );
}
// DOM: <dl><dt>...</dt><dd>...</dd><dt>...</dt><dd>...</dd></dl>
// div 없이 dl의 직접 자식으로 dt, dd 위치
```