---
aliases:
  - generics
  - T
  - "children: ReactNode"
  - ReactNode
  - satisfies
tags:
  - TypeScript
related:
  - "[[00_JS_Ecosystem_HomePage]]"
  - "[[JS_Promise]]"
  - "[[NextJS_API_Client]]"
  - "[[React_useRef]]"
  - "[[TS_TypeAssertion]]"
  - "[[React_Context]]"
  - "[[JS_Primitive_Methods]]"
  - "[[TS_ImportType]]"
---
# TS_Generics — `<T>`는 호출할 때 정해지는 타입 변수

> [!info] 
> `<T>`는 함수/타입을 "쓸 때" 채워지는 타입 자리표시자
> `any`처럼 타입 정보를 버리지 않으면서도, 함수 하나로 여러 타입에 똑같이 동작하게 해준다.

---

# 왜 필요한가 — any와 다른 점 ⭐️⭐️⭐️⭐️

```typescript
// any로 쓰면 — 타입 정보가 사라짐
async function fetchAny(path: string): Promise<any> {
  const res = await fetch(path);
  return res.json();
}
const data = await fetchAny('/posts'); // data의 타입: any (자동완성도 없음)

// 제네릭으로 쓰면 — 타입 정보가 유지됨
async function fetchAPI<T>(path: string): Promise<T> {
  const res = await fetch(path);
  return res.json() as Promise<T>;
}
const data = await fetchAPI<ApiPost[]>('/posts'); // data의 타입: ApiPost[] (자동완성 됨)
```

```txt
any:      "타입을 모른다" → 이후 그 값에 어떤 코드를 써도 TS가 검사를 안 함
제네릭<T>: "지금은 모르지만, 호출하는 쪽이 알려줄 것이다"
           → 호출 시점에 T가 정해지고 이후로는 정확히 그 타입으로 추적됨
```

---

# `<T>`가 정해지는 시점 — 호출할 때 ⭐️⭐️⭐️

```typescript
function identity<T>(value: T): T {
  return value;
}

identity<string>('hello');  // T = string으로 직접 지정
identity(42);               // 인자(42)를 보고 TS가 T = number로 자동 추론
```

```txt
<T>를 직접 적을 수도(identity<string>(...)) 있고, 생략해서 TS가 인자로부터 추론하게 둘 수도 있음
fetchAPI<ApiPost[]>('/posts')처럼 인자만 보고는 T를 추론할 거리가 없으면 직접 적어줘야 함
(path는 string이라 T와 무관 → 반환 타입에만 쓰인 T는 호출하는 쪽이 직접 알려줘야 함)
```

---

# 함수 타입 표기법 ⭐️⭐️⭐️⭐️

```txt
TypeScript에서 "함수 그 자체"를 타입으로 표현하는 방법
함수를 인자로 받거나(콜백), props로 내려줄 때 반드시 알아야 하는 문법
```

## 기본 형태

```typescript
// (매개변수: 타입) => 반환타입
() => void                        // 인자 없음, 반환값 없음
() => Promise<void>               // 인자 없음, async 함수
(id: number) => void              // 인자 하나, 반환값 없음
(id: number) => Promise<User>     // 인자 하나, Promise 반환
(a: string, b: number) => boolean // 인자 여러 개
```

```txt
void vs Promise<void>:
  void           동기 함수, 반환값 없음
  Promise<void>  async 함수 또는 Promise를 반환하는 함수, 반환값 없음
  → 이벤트 핸들러(onClick 등)는 void
  → API 호출을 포함하는 함수는 Promise<void>
```

## 실전 사용 — 함수를 인자로 받을 때 ⭐️⭐️⭐️⭐️

```typescript
// fn: () => Promise<unknown>
//   unknown = "fn이 어떤 타입을 resolve하든 상관없음, 어차피 await만 함"
//   Promise<void>로 두면 반환값 있는 함수(Promise<ApiFriendship> 등)를 넣을 때 TS 에러
const runAction = async (fn: () => Promise<unknown>) => {
  setActing(true);
  setError('');
  try {
    await fn();          // 반환값은 사용 안 함 — unknown이라 안전
    await loadRelation();
  } catch (err) {
    setError(err instanceof Error ? err.message : '요청에 실패했어요.');
  } finally {
    setActing(false);
  }
};

// void runAction(...): onClick에서 runAction의 반환 Promise를 버리는 연산자
<button onClick={() => void runAction(() => likePost(id))}>좋아요</button>
<button onClick={() => void runAction(() => createFriendRequest(id))}>친구 요청</button>
```

```txt
⚠️ void가 두 곳에 등장하는데 완전히 다른 얘기:

  onClick에서 void runAction(...)
    → runAction이 반환하는 Promise를 "의도적으로 버리는" JS 연산자
    → floating promise 경고 방지 — 타입이 아니라 연산자

  fn: () => Promise<unknown>
    → unknown = "fn이 어떤 타입을 resolve하든 상관없음"
    → runAction은 fn()의 결과를 쓰지 않으니 가장 넓은 unknown이 맞음
    → Promise<void>로 두면 Promise<ApiFriendship>처럼 반환값 있는 함수를 넣을 때 TS 에러

비교:
  Promise<void>     반환값이 없는 Promise
  Promise<unknown>  반환값이 뭐든 상관없는 Promise — 가장 넓은 타입
  void runAction()  Promise를 버리는 JS 연산자 — 타입이 아님

() => likePost(id)를 넘기는 이유:
  likePost(id)라고 쓰면 그 자리에서 즉시 실행됨
  () => likePost(id)라고 쓰면 "나중에 호출될 함수"가 됨
  → runAction 안의 await fn()이 실행될 때 비로소 likePost(id)가 호출됨

패턴 자체 설명 → [[JS_Promise]] "async 래퍼 패턴" 참고
```

## 제네릭과 함수 타입 조합 ⭐️⭐️⭐️

```typescript
// 반환값이 있는 버전 — <T>로 반환 타입을 유연하게
const runActionWithResult = async <T>(fn: () => Promise<T>): Promise<T | null> => {
  try {
    return await fn();
  } catch {
    return null;
  }
};

// 사용
const post = await runActionWithResult<Post>(() => fetchPost(id));  // T = Post
const list = await runActionWithResult<Post[]>(() => fetchPosts()); // T = Post[]
```

## 함수 타입 별칭 — 반복될 때 ⭐️

```typescript
// 매번 (fn: () => Promise<void>) 타이핑이 번거로울 때
type AsyncAction = () => Promise<void>;
type AsyncCallback<T> = () => Promise<T>;

const runAction = async (fn: AsyncAction) => { ... };
const runActionWithResult = async <T>(fn: AsyncCallback<T>): Promise<T | null> => { ... };
```

---

# 실전 예시 — `fetchAPI<T>` ⭐️⭐️⭐️

```typescript
async function fetchAPI<T>(path: string, init?: RequestInit): Promise<T> {
  const res = await fetch(`${getApiBaseUrl()}${path}`, init);
  await throwIfNotOk(res, path);
  return res.json() as Promise<T>;
}

const posts = await fetchAPI<ApiPost[]>('/posts');     // T = ApiPost[]
const user  = await fetchAPI<ApiAuthResponse>('/me');  // T = ApiAuthResponse
```

```txt
같은 함수 하나(fetchAPI)가 어떤 엔드포인트를 호출하든 재사용됨
"로직은 같은데 다루는 타입만 매번 다른" 함수를 중복 없이 작성 → 이게 제네릭이 풀어주는 문제
```

---

# 실전 예시 — `useRef<T>` ⭐️⭐️⭐️

```typescript
const rootRef  = useRef<HTMLDivElement>(null); // T = HTMLDivElement
const countRef = useRef<number>(0);            // T = number
```

```txt
useRef도 내부 동작(값을 박스에 담아 리렌더 없이 유지)은 항상 같고,
"그 박스 안에 뭐가 들어가는지"만 호출할 때 T로 결정됨 — fetchAPI<T>와 같은 발상
→ [[React_useRef]] 참고
```

---

# 제네릭에 제약 걸기 — extends ⭐️

```typescript
function getId<T extends { id: string }>(item: T): string {
  return item.id;
}
```

```txt
T extends { id: string } — "T가 뭐든 상관없지만, 최소한 id: string 필드는 있어야 한다"는 제약
이 제약이 없으면 item.id를 쓸 수 없음 (T가 아무 타입이나 될 수 있어서)
```

---
# 스프레드 → 타입 느슨해짐 → 콜백 추론 실패 ⭐️⭐️⭐️⭐️

```typescript
// ❌ 스프레드가 섞이면 타입이 넓어짐
const playerOptions = {
  ...defaultOptions,          // 스프레드
  playerVars: { autoplay: 1 },
  events: {
    onError: (e) => {         // e의 타입 추론 실패 → 'any'
      console.log(e.data);
    },
  },
};

// ✅ 콜백 파라미터를 직접 명시
const playerOptions = {
  ...defaultOptions,
  playerVars: { autoplay: 1 },
  events: {
    onError: (e: { data: number }) => {  // 직접 타입 지정
      console.log(e.data);
    },
  },
};
```

```txt
왜 스프레드가 타입을 느슨하게 만드는가:

  TS가 객체 리터럴을 볼 때:
    { autoplay: 1, rel: 0 }         → 키가 딱 보임 → 정확한 타입으로 연결
    { ...뭔가, autoplay: 1, rel: 0 } → "이 객체에 뭐가 더 붙을지 모름"
                                       → 넓은 타입(index signature)으로 잡힘

  스프레드가 섞이면:
    new YT.Player(..., { playerVars, events }) 쪽에서
    옵션이 정확한 PlayerOptions로 좁혀지지 않음
    → events.onError의 (e) 타입이 비어서 → strict 모드에서:
      "Parameter 'e' implicitly has an 'any' type" 에러

  즉 스프레드 자체가 에러가 아니라,
  스프레드 때문에 콜백 인자 타입이 비어서 에러

해결:
  1. 콜백 파라미터에 직접 타입 명시 → (e: { data: number }) => ...
  2. 옵션 객체 전체를 satisfies로 강제 → satisfies PlayerOptions
  3. 스프레드 없이 직접 객체 리터럴로 작성
```

## satisfies — 타입 강제 + 추론 유지 ⭐️⭐️⭐️

```typescript
// satisfies: 이 객체가 PlayerOptions를 만족하는지 체크 + 타입은 리터럴로 유지
const playerOptions = {
  ...defaultOptions,
  playerVars: { autoplay: 1 },
  events: {
    onError: (e) => { ... },  // PlayerOptions를 satisfies로 연결하면 e 타입 추론됨
  },
} satisfies YT.PlayerOptions;
```

```txt
satisfies vs as (타입 단언):
  as PlayerOptions  → TS에 "이거 PlayerOptions야"라고 강요 (검증 없음)
  satisfies         → "이거 PlayerOptions를 만족하는지 검증해" (안 맞으면 에러)
                      + 추론은 리터럴 타입 그대로 유지

스프레드로 타입이 느슨해질 때 satisfies로 연결하면
콜백 파라미터 타입이 다시 연결됨
```
---
# 함수 타입 — (param: T) => void ⭐️⭐️⭐️⭐️

```typescript
// 함수를 타입으로 표현하는 문법
type Handler  = () => void;                              // 파라미터 없음
type OnChange = (value: string) => void;                 // 파라미터 있음
type Fetcher  = (id: string) => Promise<User>;           // 비동기 반환
type Transform<T, U> = (input: T) => U;                  // 제네릭
```

```txt
type Fetcher = (id: string) => Promise<User> 읽는 법:
  "string 타입의 id를 받아서,
   User를 돌려주는 Promise를 반환하는 함수"

  → 즉, await했을 때 User가 나오는 async 함수

  실제 함수로 쓰면:
    const fetchUser: Fetcher = async (id) => {
      const res = await fetch(`/users/${id}`);
      return res.json();   // User 타입
    };

  왜 Promise<User>인가:
    async 함수는 항상 Promise를 반환 (await 가능한 것)
    (id: string) => User     → 동기 함수 (즉시 값 반환)
    (id: string) => Promise<User> → 비동기 함수 (나중에 값이 옴)
```

## React Props에서 콜백 패턴 ⭐️⭐️⭐️⭐️

```typescript
type Props = {
  // 필수 — 없으면 컴파일 에러
  lyricsText: string;
  value:      ApiLyricCardCustomization;
  onChange:   (value: ApiLyricCardCustomization) => void;

  // 선택 — 없어도 됨 (?:)
  trackTitle?:  string;
  trackArtist?: string;
  note?:        string | null;   // null 허용 (없음 vs 명시적 비움 구분)
  onSave?:      () => void;
  onClear?:     () => void;
  saving?:      boolean;
};
```


```txt
(value: ApiLyricCardCustomization) => void:
  "ApiLyricCardCustomization 타입의 값을 받아서 아무것도 반환하지 않는 함수"
  void = 반환값을 쓰지 않겠다는 의미 (undefined와 비슷하지만 더 넓음)
  → return 자체는 해도 되지만, 호출한 쪽에서 반환값을 사용하지 않음

?  (선택 props):
  trackTitle?: string  → 없으면 undefined
  전달 안 하면 에러 없음

string | null:
  string | undefined  → 그 prop 자체를 아예 안 넘긴 것
  string | null       → 명시적으로 "없음"을 전달하는 것
  실전: note?:string | null = "노트가 있을 수도, 명시적으로 null(삭제됨)일 수도, 아예 안 올 수도"

onXxx 명명 관행:
  onSave / onClear / onChange / onClose / onSubmit
  "어떤 이벤트가 발생했을 때 호출할 함수"라는 의미
  부모가 "뭘 할지" 결정하고, 자식은 그냥 "이 일이 일어났다"고 알림
```

## void vs undefined vs never

```typescript
type A = () => void;       // 반환은 하지만 호출한 쪽이 사용 안 함
type B = () => undefined;  // 반드시 undefined를 return해야 함
type C = () => never;      // 절대 정상 반환 안 됨 (throw or infinite loop)

// void는 실제로 값을 return해도 됨 — 타입 검사에서만 무시
const fn: () => void = () => 42;  // 에러 없음 — void는 "무시"
```

## 콜백 Props 사용 패턴

```typescript
// 자식 컴포넌트
function LyricCardEditor({ onChange, onSave, saving }: Props) {
  const handleChange = (newValue: ApiLyricCardCustomization) => {
    onChange(newValue);   // 부모에게 변경 알림
  };

  return (
    <div>
      {/* ... */}
      {onSave && (   // onSave가 있을 때만 버튼 표시
        <button onClick={onSave} disabled={saving}>
          {saving ? '저장 중…' : '저장'}
        </button>
      )}
    </div>
  );
}

// 부모 컴포넌트
<LyricCardEditor
  value={customization}
  lyricsText={lyrics}
  onChange={(v) => setCustomization(v)}
  onSave={handleSave}           // 선택 — 안 넘겨도 됨
  // onClear 생략 가능
/>
```

```txt
onSave && 패턴:
  onSave?:() => void 이므로 undefined일 수 있음
  onSave && <button onClick={onSave}>  → undefined면 아무것도 렌더링 안 함
  이렇게 선택 prop으로 UI의 일부를 show/hide 제어하는 패턴이 흔함
```
---
## size variant — 크기 변형 prop ⭐️⭐️⭐️

```typescript
// size를 리터럴 유니온으로 제한 + 기본값 설정
function ColorSwatches({
  size = 'md',   // ← 기본값
}: {
  size?: 'sm' | 'md';  // 허용 값을 타입으로 제한
}) {
  const swatch = size === 'sm' ? 'size-7' : 'size-9';  // 크기별 클래스
  // ...
}
```

```txt
size?: 'sm' | 'md' 패턴:
  string 으로 하면 'xxl', 'huge' 같은 임의 값도 다 통과됨
  리터럴 유니온으로 제한하면 허용된 값만 받고, 자동완성도 됨

  variant prop 에서도 같은 패턴:
    variant?: 'primary' | 'secondary' | 'ghost'
    intent?:  'default' | 'danger' | 'success'

기본값 = 'md':
  함수 파라미터 구조분해에서 직접 지정
  { size = 'md' } = 프롭을 안 넘기면 'md'
  타입은 ?:로 optional이어도 내부에서는 항상 string으로 처리됨

크기 → 클래스 매핑:
  const swatch = size === 'sm' ? 'size-7' : 'size-9'

  또는 Record로 정리:
  const SIZE_CLASS: Record<'sm' | 'md', string> = {
    sm: 'size-7',
    md: 'size-9',
  };
  const swatch = SIZE_CLASS[size];
```


---
# ReactNode — 렌더링 가능한 모든 것 ⭐️⭐️⭐️⭐️

```typescript
import { type ReactNode } from 'react';

// children prop에 가장 자주 쓰임
type Props = {
  children: ReactNode;
  label?:   ReactNode;  // 텍스트뿐 아니라 JSX도 받을 수 있게
};
```

```txt
ReactNode = React가 렌더링할 수 있는 모든 것의 유니온
  JSX.Element  — React 엘리먼트 (<div />, <MyComponent /> 등)
  string
  number
  boolean
  null
  undefined
  ReactNode[]  — 위의 배열

→ "어떤 값이 와도 렌더링할 수 있다"는 가장 넓은 타입
```

## ReactNode vs JSX.Element vs ReactElement ⭐️⭐️⭐️

```typescript
// JSX.Element — JSX 표현식 하나 (가장 좁음)
const a: JSX.Element = <div />;      // ✅
const b: JSX.Element = 'hello';      // ❌ 문자열 안 됨
const c: JSX.Element = null;         // ❌ null 안 됨

// ReactNode — JSX + string, number, null, undefined 전부 포함 (가장 넓음)
const d: ReactNode = <div />;        // ✅
const e: ReactNode = 'hello';        // ✅
const f: ReactNode = null;           // ✅
const g: ReactNode = 42;             // ✅
```

```txt
언제 뭘 쓰는가:

  children?: ReactNode
    가장 흔한 사용 — 문자열·숫자·null·JSX 뭐든 받을 수 있게

  renderHeader?: () => JSX.Element
    "반드시 하나의 JSX 엘리먼트를 반환하는 함수"로 제한할 때

  icon?: React.ReactElement
    JSX 엘리먼트만 받되, 문자열·null은 안 받을 때
    (예: 아이콘 prop — 문자열이 오면 의도와 다름)

children?: ReactNode가 children?: JSX.Element보다 유연한 이유:
  <MyComp>텍스트</MyComp>  → children = string → ReactNode ✅, JSX.Element ❌
  <MyComp>{cond && <Icon />}</MyComp>
                           → cond = false면 children = false
                           → ReactNode ✅ (false는 렌더링 안 됨), JSX.Element ❌
```

## 실전 패턴

```typescript
// Provider — 항상 children: ReactNode
export function ThemeProvider({ children }: { children: ReactNode }) {
  return <ThemeContext.Provider value={theme}>{children}</ThemeContext.Provider>;
}

// slot prop — 없어도 되고 뭐든 올 수 있음
type CardProps = {
  title:    string;
  children: ReactNode;
  footer?:  ReactNode;
};

// label이 텍스트일 수도 아이콘+텍스트일 수도 있을 때
type ButtonProps = {
  label: ReactNode;  // "저장" 또는 <><SaveIcon /> 저장</>
};
```
---
# keyof — 타입의 키를 유니온으로 추출 ⭐️⭐️⭐️⭐️

```typescript
type LyricDecorDisplay = {
  lyrics: boolean;
  track:  boolean;
  note:   boolean;
  mood:   boolean;
  date:   boolean;
  cover:  boolean;
};

type DisplayKey = keyof LyricDecorDisplay;
// → 'lyrics' | 'track' | 'note' | 'mood' | 'date' | 'cover'
```

```typescript
// 실전 — 타입 안전한 상수 배열
const DISPLAY_CHIPS: { key: keyof LyricDecorDisplay; label: string }[] = [
  { key: 'lyrics', label: '가사' },
  { key: 'track',  label: '곡' },
  { key: 'note',   label: '메모' },
  { key: 'mood',   label: '감정' },
  { key: 'date',   label: '날짜' },
  { key: 'cover',  label: '커버' },
];
```

```txt
{ key: keyof LyricDecorDisplay; label: string }[]:
  key 필드가 LyricDecorDisplay의 키 중 하나임을 보장
  'lyric' 같은 오타를 넣으면 컴파일 에러 → 런타임 버그 방지

활용:
  DISPLAY_CHIPS.map(({ key, label }) => (
    <Toggle key={key} label={label} on={display[key]} />
  ))
  → display[key] 접근도 타입 안전 (key가 keyof 이므로)
```

---
# `Partial<T>` + 스프레드 패치 패턴 ⭐️⭐️⭐️⭐️


```typescript
// 전체 객체 중 일부 필드만 업데이트하는 패치 함수
const patch = (partial: Partial<ApiLyricCardCustomization>) => {
  onChange({ ...normalizeDecor(value), ...partial });
  //         ↑ 현재 값 전체                ↑ 바꿀 필드만 덮어씀
};

// 사용
patch({ display: { ...d, [key]: on } });
//     ↑ display 필드만 바꾸고 나머지는 유지
```

```txt
Partial<T>:
  T의 모든 필드를 optional로 만듦
  { a: string; b: number } → { a?: string; b?: number }
  "일부 필드만 보내면 되는" 패치 함수의 파라미터로 자주 씀

스프레드 패치 패턴:
  { ...기존값, ...partial }
  partial의 필드가 기존값의 필드를 덮어씀
  나머지 필드는 그대로 유지

  주의: 중첩 객체는 스프레드로 합쳐지지 않음 (얕은 병합)
  → display 같은 중첩 객체도 바꾸려면 명시적으로 펼쳐야 함
  patch({ display: { ...d, [key]: on } })  ← display를 따로 펼침
```

---

# `readonly T[]` — 읽기 전용 배열 파라미터 ⭐️⭐️⭐️⭐️

```typescript
// readonly T[] — "이 함수는 배열을 읽기만 하고, 절대 바꾸지 않는다"는 선언
function pickOne<T>(items: readonly T[]): T {
  return items[Math.floor(Math.random() * items.length)];
}
```

```typescript
// 호출 — const 배열도, mutable 배열도 모두 넣을 수 있음
const WISHES = ['행복하세요.', '좋은 하루 되세요.'] as const;
pickOne(WISHES);          // ✅ readonly (as const) 배열

const arr = ['a', 'b'];
pickOne(arr);             // ✅ 일반 배열도 가능 (readonly로 넣으면 더 넓게 받음)
```

```txt
readonly T[] vs T[]:

  파라미터 타입이 T[] 이면:
    일반 배열만 받을 수 있음
    readonly 배열('as const'로 만든 것)을 넣으면 TS 에러

  파라미터 타입이 readonly T[] 이면:
    일반 배열 + readonly 배열 모두 받을 수 있음 (더 넓게 받음)
    함수 안에서 .push() .pop() .splice() 같은 변경 메서드 호출 불가
    → "이 함수는 배열을 바꾸지 않는다"는 약속을 타입으로 강제

  → 배열을 읽기만 하는 함수라면 readonly T[]를 쓰는 것이 더 정확하고 유연함
```

## as const 배열과 조합 ⭐️⭐️⭐️

```typescript
// as const — 배열을 리터럴 타입의 readonly 튜플로 만듦
const MORNING_WISHES = [
  '오늘도 활기차게 시작해요.',
  '좋은 하루 되세요.',
  '오늘 하루도 잘 부탁드려요.',
] as const;

// MORNING_WISHES 타입:
// readonly ['오늘도 활기차게 시작해요.', '좋은 하루 되세요.', '오늘 하루도 잘 부탁드려요.']

// pickOne 반환 타입: 세 문자열 리터럴 유니온
// → '오늘도 활기차게 시작해요.' | '좋은 하루 되세요.' | '오늘 하루도 잘 부탁드려요.'
const wish = pickOne(MORNING_WISHES);
```

```txt
as const의 효과:
  배열이 readonly 튜플로 변환됨 → 요소 수정, 추가, 삭제 불가
  요소 타입이 string에서 각 문자열 리터럴 타입으로 좁혀짐

pickOne<T>에서 T 추론:
  items: readonly T[]에 MORNING_WISHES를 넣으면
  T = '오늘도...' | '좋은 하루...' | '오늘 하루...' 로 추론됨
  반환 타입도 자동으로 그 유니온 타입

실전에서 자주 쓰는 패턴:
  const ITEMS = ['a', 'b', 'c'] as const;  // 상수 배열 정의
  type Item = typeof ITEMS[number];          // 'a' | 'b' | 'c' 타입 추출
  function pickOne<T>(items: readonly T[]): T { ... }  // 꺼내기
```

---

# ReadonlySet / ReadonlyMap / ReadonlyArray ⭐️⭐️⭐️

```typescript
// ReadonlySet — 읽기 전용 Set
const ids: ReadonlySet<string> = new Set(['a', 'b']);
ids.has('a');    // ✅ 읽기는 가능
ids.add('c');    // ❌ TS 에러 — 수정 메서드 없음
ids.delete('a'); // ❌ TS 에러

// ReadonlyMap
const map: ReadonlyMap<string, number> = new Map();
map.get('key');  // ✅
map.set('k', 1); // ❌

// readonly 배열
const arr: ReadonlyArray<string> = ['a', 'b'];
// 또는 축약형
const arr: readonly string[] = ['a', 'b'];
arr[0];          // ✅
arr.push('c');   // ❌
```

```txt
언제 쓰는가:
  Context value에서 "이 Set은 Provider 안에서만 바꾸고, 읽는 쪽은 읽기만 해라"
  → type FriendIdsContextValue = { ids: ReadonlySet<string>; reload: () => void }

  상태를 외부에 노출할 때 수정 권한을 제한
  "이 값을 받은 쪽은 건드리지 마라"는 의도를 타입으로 표현

  ⚠️ 타입 수준의 제약 — 런타임에 실제로 막히는 건 아님
     setIds(원래Set)처럼 Provider 안에서는 여전히 수정 가능
```

---

# 한눈에

| 개념                                          | 핵심                                                        |
| ------------------------------------------- | --------------------------------------------------------- |
| `<T>`                                       | 함수/타입을 호출·사용하는 시점에 정해지는 타입 변수                             |
| `any`와의 차이                                  | any는 타입 정보를 버림, 제네릭은 타입 정보를 끝까지 추적함                       |
| 추론                                          | 인자로부터 T를 알 수 있으면 생략 가능, 알 수 없으면 직접 명시                     |
| 쓰는 이유                                       | "로직은 같은데 다루는 타입만 다른" 함수를 중복 없이 작성                         |
| `T extends 조건`                              | T가 최소한 갖춰야 하는 모양을 제약                                      |
| `() => void`                                | 동기 함수 타입, 반환값 없음                                          |
| `() => Promise<void>`                       | async 함수 타입, 반환값 없음                                       |
| `() => Promise<unknown>`                    | 반환값이 뭐든 상관없는 async 함수 — 결과를 쓰지 않는 콜백에 사용                  |
| `(fn: () => Promise<unknown>) => {}`        | 함수를 인자로 받는 고차 함수 — async 래퍼 패턴                            |
| `void runAction()`                          | Promise를 버리는 JS 연산자 — 타입이 아님, onClick floating promise 방지 |
| `type AsyncAction = () => Promise<unknown>` | 반복되는 함수 타입을 별칭으로 추출                                       |
