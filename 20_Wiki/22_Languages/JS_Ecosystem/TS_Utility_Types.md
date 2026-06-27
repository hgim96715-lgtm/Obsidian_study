---
aliases:
  - 유틸리티 타입
  - Partial
  - Omit
  - Pick
tags:
  - TypeScript
related:
  - "[[00_JS_Ecosystem_HomePage]]"
  - "[[NestJS_DTO]]"
  - "[[NextJS_API_Integration]]"
  - "[[TS_API_Types]]"
  - "[[JS_Fetch_API]]"
---
# TS_Utility_Types — 유틸리티 타입

# 한 줄 요약

```
기존 타입을 변환해서 새 타입을 만드는 내장 유틸리티
TS 가 기본 제공 / import 없이 바로 사용
```

---

---

# `Partial<T>` — 전부 선택적으로 ⭐️

```typescript
interface Movie {
  id: number;
  title: string;
  genre: string;
}

// Partial: 모든 속성을 ? (선택적) 으로 만듦
type PartialMovie = Partial<Movie>;
// {
//   id?: number;
//   title?: string;
//   genre?: string;
// }

// PATCH 요청 DTO 패턴
function updateMovie(id: number, updates: Partial<Movie>) {
  // updates 는 일부 필드만 있어도 됨
}
updateMovie(1, { title: '수정된 제목' });   // ✅ genre 없어도 OK
```

---

---

# `Required<T>` — 전부 필수로

```typescript
interface Config {
  host?: string;
  port?: number;
}

// Required: 모든 선택적 속성을 필수로
type RequiredConfig = Required<Config>;
// {
//   host: string;
//   port: number;
// }
```

---

---

# `Readonly<T>` — 읽기 전용

```typescript
interface Movie {
  id: number;
  title: string;
}

const movie: Readonly<Movie> = { id: 1, title: '마이클' };
movie.title = '수정';   // ❌ 에러 (읽기 전용)
```

---

---

# Pick<T, K> — 특정 키만 선택 ⭐️

```typescript
interface Movie {
  id: number;
  title: string;
  genre: string;
  description: string;
  createdAt: Date;
}

// 특정 키만 골라서 새 타입
type MovieSummary = Pick<Movie, 'id' | 'title'>;
// { id: number; title: string; }

// 응답에서 일부 필드만 반환할 때
function getMovieSummary(movie: Movie): Pick<Movie, 'id' | 'title'> {
  return { id: movie.id, title: movie.title };
}
```

---

---

# Omit<T, K> — 특정 키 제외 ⭐️

```typescript
interface User {
  id: number;
  name: string;
  email: string;
  password: string;
}

// 특정 키 제외
type SafeUser = Omit<User, 'password'>;
// { id: number; name: string; email: string; }

// DTO 패턴 — 생성 시 id 제외 (자동 생성이므로)
type CreateUserDto = Omit<User, 'id'>;
// { name: string; email: string; password: string; }
```

```
Pick vs Omit:
  Pick  포함할 키 나열 (적을 때 편함)
  Omit  제외할 키 나열 (제외가 적을 때 편함)
```

---

---

# Omit + 교차(&) — 특정 필드만 다른 타입으로 재정의 ⭐️⭐️⭐️

```
지금까지 본 Omit 은 "키를 그냥 빼버리는" 용도였음
근데 실무에서 더 자주 보는 패턴은 "빼고 → 그 자리에 다른 타입을 다시 채워넣는" 것
→ 라이브러리가 이미 정의해둔 타입(RequestInit 등)을 그대로 쓰되
  그 안의 "딱 한 필드만" 더 좁은(혹은 다른) 타입으로 바꾸고 싶을 때 씀
```

```typescript
type AdminFetchInit = Omit<RequestInit, "headers"> & {
  headers?: Record<string, string>;
};
```

```
한 줄씩:
  RequestInit                          fetch 의 두 번째 인자 타입 (method/body/headers 등 다 포함)
                                        원래 headers 의 타입은 HeadersInit
                                        (= string[][] | Record<string,string> | Headers, 셋 중 하나 허용)

  Omit<RequestInit, "headers">         RequestInit 에서 headers 필드만 빼버림
                                        (method/body 등 나머지는 전부 그대로 유지)

  & { headers?: Record<string,string> } 방금 뺀 자리에 "내가 원하는 더 좁은 타입" 의 headers 를 새로 추가
                                        → 결과: method/body 등은 RequestInit 그대로,
                                          headers 만 Record<string,string> 으로 좁혀진 새 타입
```

## 왜 그냥 RequestInit & {...} 으로는 안 되는가 ⭐️⭐️

```typescript
// ❌ Omit 없이 그냥 교차하면
type Bad = RequestInit & { headers?: Record<string, string> };
```

```
헷갈리기 쉬운 지점: "& 로 합치면 새로 적은 게 그냥 덮어쓰는 거 아닌가?"
→ 아님. & (교차 타입)은 "두 타입 모두를 동시에 만족" 해야 하는 타입을 만듦
  같은 이름의 속성(headers)이 양쪽에 다 있으면, "덮어쓰기" 가 아니라
  "그 속성의 타입을 둘 다 만족하도록 합치는" 쪽으로 동작함

  → RequestInit.headers 타입(HeadersInit)과 내가 새로 적은 타입(Record<string,string>)이
    뒤섞여서 의도와 다른, 다루기 애매한 타입이 되어버림
  → 그래서 먼저 Omit 으로 "원래 있던 headers 자체를 깨끗이 지운 다음"
    교차로 새 headers 를 추가해야 확실하게 "교체" 가 됨

핵심 원칙:
  교차 타입(&)으로 "기존 필드의 타입을 바꾸고" 싶다면
  → 그냥 새 필드를 더하지 말고, 항상 Omit 으로 그 필드를 먼저 지운 다음 더할 것
```

## 왜 굳이 headers 를 Record<string,string> 으로 좁히는가 ⭐️⭐️

```typescript
async function AdminFetch(path: string, init: AdminFetchInit = {}) {
  // ...
  const res = await fetch(`${API_URL}${path}`, {
    ...init,
    headers: {
      Authorization: `Bearer ${token}`,
      ...init.headers,   // ← 여기서 스프레드(...)로 합침
    },
  });
}
```

```
원래 HeadersInit 타입은 3가지 형태를 다 허용함:
  Record<string,string>   { 'Content-Type': 'application/json' } 같은 평범한 객체
  string[][]               [['Content-Type','application/json']] 같은 배열
  Headers                  new Headers() 로 만든 클래스 인스턴스

근데 코드에서는 ...init.headers 로 "객체 스프레드" 를 해서 합치고 있음
→ 객체 스프레드는 "평범한 key-value 객체" 일 때만 의도대로 동작함
  string[][] 나 Headers 인스턴스를 스프레드하면 키-값이 제대로 안 펼쳐져서
  Authorization 헤더와 제대로 합쳐지지 않는 등 예상과 다르게 동작할 수 있음

→ 그래서 타입 자체를 Record<string,string> 으로 좁혀서
  "이 함수에 headers 를 넘길 땐 평범한 객체만 가능하다" 고 컴파일 시점에 강제함
  → 호출하는 쪽이 실수로 Headers 인스턴스나 배열을 넘기는 걸 TS 가 미리 막아줌
```

```
일반화한 패턴 — 다른 곳에도 그대로 적용 가능:
  서드파티 타입(RequestInit, HTMLAttributes 등)을 가져다 쓰는데
  그 중 한 필드만 내 코드에 맞게 더 좁히거나 다른 타입으로 바꾸고 싶을 때
  → Omit<원래타입, '그필드'> & { 그필드: 새타입 }
```

---

---

# Record<K, V> — 키-값 맵 타입

```typescript
// Record<키 타입, 값 타입>
type MovieMap = Record<number, string>;
// { [key: number]: string }

const movies: MovieMap = {
  1: '마이클',
  2: '악마는 프라다를 입는다',
};

// 고정 키 목록
type Status = 'pending' | 'approved' | 'rejected';
type StatusLabel = Record<Status, string>;

const labels: StatusLabel = {
  pending:  '대기중',
  approved: '승인됨',
  rejected: '거부됨',
};
```

---

---

# `ReturnType<T>` — 함수 반환 타입 추출

```typescript
function getMovie() {
  return { id: 1, title: '마이클', genre: '드라마' };
}

// 함수의 반환 타입 자동 추출
type MovieType = ReturnType<typeof getMovie>;
// { id: number; title: string; genre: string; }

// 서비스 반환 타입 재사용
type ServiceResult = ReturnType<typeof movieService.findAll>;
```

---

---

# 조합 패턴

```typescript
interface Movie {
  id: number;
  title: string;
  genre: string;
  password: string;
  createdAt: Date;
}

// 생성 DTO: id / createdAt 제외 + 전부 선택적
type CreateMovieDto = Partial<Omit<Movie, 'id' | 'createdAt'>>;

// 수정 DTO: id / password 제외 + 전부 선택적
type UpdateMovieDto = Partial<Omit<Movie, 'id' | 'password'>>;

// 응답 DTO: password 제외
type MovieResponse = Omit<Movie, 'password'>;
```

```
이 조합들과 위 "Omit + &" 의 차이:
  Partial<Omit<T, K>>   "키를 빼고, 남은 키들을 선택적으로" — 변형(transform)을 두 번 적용
  Omit<T, K> & {...}    "키를 빼고, 그 자리에 다른 타입을 다시 넣음" — 한 필드를 교체(override)

  목적이 다름: 앞은 "필드 구성 자체를 줄이거나 느슨하게" 하는 것
             뒤는 "필드 구성은 거의 그대로 두되, 특정 필드의 타입만 바꾸는" 것
```

---

---

# 한눈에

|유틸리티|변환|주 용도|
|---|---|---|
|`Partial<T>`|모든 속성 → 선택적|PATCH DTO|
|`Required<T>`|모든 속성 → 필수|완전한 객체 보장|
|`Readonly<T>`|모든 속성 → 읽기 전용|불변 객체|
|`Pick<T, K>`|일부 키만 선택|특정 필드만 노출|
|`Omit<T, K>`|일부 키 제외|민감 정보 제외 ⭐️|
|`Omit<T, K> & {...}`|특정 필드만 다른 타입으로 교체|서드파티 타입 확장 시 필드 좁히기 ⭐️⭐️|
|`Record<K, V>`|키-값 맵|상태 맵 / 딕셔너리|
|`ReturnType<T>`|함수 반환 타입|타입 재사용|

```
필드 타입을 바꾸고 싶을 때 기억할 한 줄:
  그냥 & 로 더하면 타입이 뒤섞임 → 항상 Omit 으로 먼저 지우고 & 로 새로 채울 것
```