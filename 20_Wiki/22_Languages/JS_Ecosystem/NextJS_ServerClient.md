---
aliases:
  - Client Component
  - Server Component
  - use client
  - use server
tags:
  - NextJS
  - React
related:
  - "[[00_JS_Ecosystem_HomePage]]"
  - "[[React_useId]]"
---
# NextJS_ServerClient — Server Component vs Client Component

> [!info] 
> Next.js App Router에서 모든 컴포넌트는 기본적으로 Server Component다. 
> `'use client'`를 붙인 파일만 Client Component가 된다.
>  `'use server'`는 그 반대(Server Component 표시)가 아니라 완전히 다른 것 
>  — 클라이언트에서 호출 가능한 서버 함수(Server Action)를 표시하는 directive다.

---

# 기본값은 Server Component — 아무것도 안 붙여도 ⭐️⭐️⭐️

```txt
Next.js App Router에서는 디렉티브를 아무것도 안 붙이면 그게 곧 Server Component임
"Server Component 표시용 directive"는 따로 없음 — 기본값 자체가 Server Component이기 때문
```

|할 수 있는 것 (Server)|할 수 없는 것 (Server)|
|---|---|
|컴포넌트 안에서 직접 `await fetch(...)`|`useState`, `useEffect` 같은 훅|
|DB 클라이언트, 시크릿 환경변수 직접 접근|`onClick`, `onChange` 같은 이벤트 핸들러|
|이 컴포넌트의 JS 코드는 브라우저로 전송 안 됨(번들 크기 0)|`useContext` 등 Context 구독|
||브라우저 전용 API(`window`, `localStorage` 등)|

```txt
오른쪽 칸(할 수 없는 것)에 해당하는 게 하나라도 필요해지는 순간, 그 컴포넌트는 'use client'가 필요함
```

---

# 'use client' — 언제 붙이나 ⭐️⭐️⭐️

```tsx
'use client'; // 파일 맨 위, 다른 import보다도 앞에

import { useState } from 'react';

export function LikeButton() {
  const [liked, setLiked] = useState(false);
  return <button onClick={() => setLiked(!liked)}>{liked ? '❤️' : '🤍'}</button>;
}
```

|필요한 게 하나라도 있으면 'use client' 필요|
|---|
|`useState`, `useReducer`, `useEffect` 등 React 훅|
|`onClick`, `onChange`, `onSubmit` 등 이벤트 핸들러|
|`window`, `localStorage`, `document` 등 브라우저 API|
|`useContext` (Context를 만드는 Provider도 보통 client)|
|위 항목을 내부에서 쓰는 커스텀 훅(`useRouter`, `usePathname` 등도 포함)|

```txt
'use client'를 붙이면 정확히 무슨 일이 일어나나:
  그 파일이 "서버 ↔ 클라이언트 경계"가 됨
  그 파일 안에서 정의한 컴포넌트뿐 아니라, 그 파일이 직접 import해서 JSX로 그리는 모든 것도
  같이 클라이언트 번들에 포함됨 — 자세한 건 아래 "경계" 섹션
```

---

# ⚠️ 'use server'는 'use client'의 반대가 아님 ⭐️⭐️⭐️

```txt
가장 흔한 오해: "'use client'가 클라이언트 컴포넌트 표시니까, 'use server'는 서버 컴포넌트 표시겠지"
→ 틀림. Server Component는 표시할 directive가 필요 없음(이미 기본값이므로)

'use server'는 완전히 다른 용도: Server Action(클라이언트에서 호출할 수 있는, 서버에서 실행되는 함수)을
표시하는 directive — 컴포넌트가 아니라 함수에 붙는 것
```

```tsx
// actions.ts — 파일 전체를 Server Action 모음으로 표시
'use server';

export async function createPost(formData: FormData) {
  // 서버에서만 실행됨 — DB 접근 등 자유롭게
}
```

```tsx
// 'use client' 컴포넌트에서 그 함수를 가져와 form에 연결
'use client';
import { createPost } from './actions';

export function NewPostForm() {
  return <form action={createPost}>{/* ... */}</form>;
}
```

|구분|Server Component|Server Action|
|---|---|---|
|표시 방법|없음(기본값)|`'use server'`|
|역할|화면을 서버에서 렌더링|클라이언트가 호출하면 서버에서 실행되는 함수(주로 데이터 변경)|
|단위|컴포넌트|함수|
|호출/사용|그냥 JSX에 렌더링|`<form action={fn}>` 또는 직접 호출|

```txt
"use client의 반대 짝인 use server는 없다 — use server는 지금은 Server Action을 표시하는 데만 쓰인다"
(Next.js 팀이 GitHub 디스커션에서 직접 확인한 내용) — 이 한 문장이 가장 핵심
```

---

# 'use client' 경계 — 그 파일이 import하는 것도 같이 클라이언트가 됨 ⭐️⭐️⭐️

```tsx
// FeedClientWrapper.tsx
'use client';
import { ScrollObserver } from './ScrollObserver'; // 'use client' 없는 평범한 컴포넌트라도

export function FeedClientWrapper() {
  return <ScrollObserver />; // 여기서 직접 import해서 그리면 → 클라이언트 번들에 포함됨
}
```

```txt
ScrollObserver 자신에게는 'use client'가 없어도, FeedClientWrapper(이미 client 경계)가
이 컴포넌트를 직접 import해서 JSX로 그리는 순간 같이 클라이언트로 묶여 들어감
→ "직접 import + 그 파일 안에서 렌더링" 하면, 그 하위 트리는 전부 클라이언트 취급
```

---

# Server Component를 Client Component 안에 "살려서" 넣는 법 — children/props로 전달 ⭐️⭐️⭐️

```txt
방금 본 규칙의 유일한 예외: Client Component가 직접 import해서 그리는 게 아니라,
바깥(Server Component)에서 이미 만들어진 결과를 children/props로 "전달"받으면
그 부분은 여전히 서버에서 렌더링됨 — 이게 "client 자식만 client"라는 말의 정확한 의미
```

```tsx
// app/feed/page.tsx — Server Component (디렉티브 없음, 기본값)
import { FeedClientWrapper } from './FeedClientWrapper';
import { FeedHeader } from './FeedHeader'; // 이 컴포넌트도 'use client' 없음

export default function FeedPage() {
  return (
    <FeedClientWrapper>
      <FeedHeader /> {/* children으로 전달 — 서버에서 렌더링된 "결과물"만 넘어감 */}
    </FeedClientWrapper>
  );
}
```

```tsx
// FeedClientWrapper.tsx
'use client';
import { useEffect } from 'react';

export function FeedClientWrapper({ children }: { children: React.ReactNode }) {
  useEffect(() => {
    // 무한 스크롤, 인터섹션 옵저버 등 클라이언트 전용 로직
  }, []);
  return <div>{children}</div>;
}
```

```txt
왜 FeedHeader는 'use client' 없이 둬도 되는가:
  FeedClientWrapper.tsx 파일 안에서는 FeedHeader를 import조차 안 함 — 그냥 children이라는
  "이미 렌더링된 React 노드"를 받아서 그 자리에 끼워 넣을 뿐임
  → FeedHeader는 FeedPage(Server Component) 쪽에서 이미 서버에서 렌더링이 끝난 뒤,
    그 결과만 FeedClientWrapper에게 전달되는 것 — FeedClientWrapper는 그 내용을 몰라도 됨

  반대로 FeedClientWrapper.tsx 안에서 import { FeedHeader } 하고 그 안에서 직접
  <FeedHeader />로 그렸다면, 위 "경계" 섹션 규칙대로 클라이언트로 묶여 들어갔을 것
```

|패턴|FeedHeader가 서버에 남아있나|
|---|---|
|`FeedClientWrapper` 내부에서 직접 import + 렌더링|❌ 클라이언트로 편입됨|
|`FeedClientWrapper` 바깥(Server Component)에서 children/props로 전달|✅ 서버에 그대로 남음|

---

# 한눈에

| 키워드                    | 한 줄 정리                                                               |
| ---------------------- | -------------------------------------------------------------------- |
| 기본값                    | 디렉티브 없음 = Server Component (표시할 게 따로 없음)                             |
| `'use client'`         | useState/이벤트핸들러/브라우저API/Context 중 하나라도 필요하면 그 파일 맨 위에                |
| `'use server'`         | `'use client'`의 반대가 아님 — Server Action(함수)을 표시하는 별개의 directive       |
| 클라이언트 경계               | 'use client' 파일이 직접 import해서 그리는 모든 것도 같이 클라이언트로 묶임                  |
| 예외 — children/props 전달 | Server Component에서 만들어서 전달한 children은 Client Component 안에 있어도 서버에 남음 |
| "client 자식만 client"    | 직접 import해서 그 파일 안에서 그린 것만 client — 밖에서 전달받은 children은 원래 종류 유지      |