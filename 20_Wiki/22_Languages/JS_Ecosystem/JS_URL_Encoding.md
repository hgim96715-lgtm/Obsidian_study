---
aliases:
  - URLSearchParams
  - URL
  - 인코딩
  - 디코딩
  - window.location.search
tags:
  - JavaScript
  - NextJS
related:
  - "[[00_JS_Ecosystem_HomePage]]"
  - "[[NextJS_Routing]]"
  - "[[NestJS_Pagination]]"
  - "[[JS_BrowserAPI]]"
---
# JS_URL_Encoding — URL 인코딩 · URLSearchParams

>[!info]
>URL에는 특수문자(&, =, 공백, # 등)를 그대로 쓸 수 없다 — URL 구조를 깨뜨리기 때문. 
>`URLSearchParams` = 쿼리스트링을 안전하게 조합하거나 파싱하는 도구.
> `window.location.search`로 현재 URL의 쿼리스트링을 읽고 URLSearchParams로 파싱한다.

---

# URL 구조 ⭐️⭐️⭐️⭐️

```txt
https://api.example.com/users/search?q=홍길동&limit=20&cursor=abc#top
───┬───  ───────┬──────  ──────┬───── ─────────────┬──────────────  ─┬─
   │             │              │                   │                  │
   프로토콜      호스트         경로(path)           쿼리스트링        fragment
```

```txt
쿼리스트링:
  ? 로 시작
  key=value 형태
  여러 개는 & 로 연결
  → ?q=홍길동&limit=20&cursor=abc

여기서 & = # 공백은 URL 구조를 구분하는 특수문자
→ 이 문자들이 "값" 안에 있으면 URL이 깨짐
```

---

# 왜 인코딩이 필요한가 ⭐️⭐️⭐️⭐️

```typescript
// 검색어가 "홍길동 & 김철수"라면 어떻게 될까?

// ❌ 직접 문자열 조합
const url = `/users/search?q=${opts.q}&limit=20`;
// → /users/search?q=홍길동 & 김철수&limit=20
//                        ↑ 공백  ↑ & 가 URL에 그대로 들어감

// 서버가 이 URL을 파싱하면:
//   q     = "홍길동 "   (& 앞까지만)
//   " 김철수" = ""       (키로 인식)
//   limit  = "20"
// → "홍길동 & 김철수"를 검색하는 게 아니라 깨진 요청이 됨
```

```typescript
// ✅ URLSearchParams — 특수문자 자동 인코딩
const sp = new URLSearchParams();
sp.set('q', '홍길동 & 김철수');
sp.set('limit', '20');
sp.toString()
// → "q=%ED%99%8D%EA%B8%B8%EB%8F%99+%26+%EA%B9%80%EC%B2%A0%EC%88%98&limit=20"
//     ↑ 한글은 %xx 형태로, 공백은 +, &는 %26으로 변환됨

// 서버가 파싱하면:
//   q     = "홍길동 & 김철수"   ← 정확히 원하는 값
//   limit = "20"
```

```txt
인코딩 = 특수문자를 % + 16진수 코드로 변환
  공백 → + 또는 %20
  &    → %26
  =    → %3D
  ?    → %3F
  #    → %23
  한글 → %ED%99%8D ... (UTF-8 바이트를 % + hex로)

URLSearchParams가 이 변환을 자동으로 해줌
직접 문자열 조합하면 매번 인코딩을 직접 해야 함 → 실수 발생 쉬움
```

---

# URLSearchParams — 쿼리스트링 안전하게 만들기 ⭐️⭐️⭐️⭐️

```typescript
const sp = new URLSearchParams();

// 값 추가
sp.set('q', '검색어');          // 없으면 추가, 있으면 덮어씀
sp.append('tag', '개발');       // 기존 값 유지하고 추가 (같은 키 여러 개)
sp.delete('q');                 // 삭제
sp.has('q')                     // 있는지 확인 → true/false
sp.get('q')                     // 값 읽기 → string | null
sp.toString()                   // "q=%EA%B2%80%EC%83%89%EC%96%B4" 형태로 출력

// 초기값으로 생성
const sp2 = new URLSearchParams({ q: '검색어', limit: '20' });
const sp3 = new URLSearchParams('q=%EA%B2%80%EC%83%89%EC%96%B4&limit=20');
```

## 조건부 파라미터 — 있을 때만 추가 ⭐️⭐️⭐️⭐️

```typescript
// 실전 예시 — 사용자 검색 API
export function searchUsers(opts: {
  q:       string;
  cursor?: string;
  limit?:  number;
}): Promise<ApiUserSearchPage> {
  const sp = new URLSearchParams();

  sp.set('q', opts.q.trim());                                // 필수 — 항상 추가

  if (opts.cursor)       sp.set('cursor', opts.cursor);      // 있을 때만
  if (opts.limit != null) sp.set('limit', opts.limit.toString()); // 0도 포함

  return apiFetch<ApiUserSearchPage>(`/users/search?${sp.toString()}`);
}
```

```txt
if (opts.cursor) sp.set(...):
  cursor가 undefined이거나 빈 문자열이면 추가 안 함
  첫 페이지 요청 = cursor 없음 → ?q=홍길동 만
  다음 페이지 요청 = cursor 있음 → ?q=홍길동&cursor=abc

if (opts.limit != null) sp.set(...):
  != null 은 null과 undefined 둘 다 걸러냄
  !== undefined 만 쓰면 null이 통과됨
  0도 유효한 limit이므로 if (opts.limit)을 쓰면 안 됨
    → if (opts.limit) 은 0을 falsy로 처리 → limit=0 이 무시됨
    → if (opts.limit != null) 은 0도 통과 ✅

opts.limit.toString():
  URLSearchParams의 값은 항상 문자열
  number를 그냥 넣으면 TypeScript 에러 → toString()으로 변환
```

## ?${sp.toString()} 조합 ⭐️⭐️⭐️

```typescript
const sp = new URLSearchParams();
sp.set('q', '홍길동');
sp.set('limit', '20');

// sp.toString() → "q=%ED%99%8D%EA%B8%B8%EB%8F%99&limit=20"
const url = `/users/search?${sp.toString()}`;
// → "/users/search?q=%ED%99%8D%EA%B8%B8%EB%8F%99&limit=20"

// 파라미터가 없을 수도 있으면 — ? 자체를 붙이지 않기
const qs = sp.toString();
const url2 = `/users/search${qs ? `?${qs}` : ''}`;
```

---

# encodeURIComponent — 단일 값 수동 인코딩 ⭐️⭐️⭐️

```typescript
// URL의 일부(path 세그먼트)에 특수문자가 들어갈 때
const username = '홍길동/admin';  // / 가 path 구분자로 잘못 해석됨
const url = `/users/${encodeURIComponent(username)}`;
// → /users/%ED%99%8D%EA%B8%B8%EB%8F%99%2Fadmin  ← / 가 %2F로 인코딩

// encodeURI vs encodeURIComponent
encodeURI('https://example.com/path?q=홍길동 & 김철수')
// → "https://example.com/path?q=%ED%99%8D..." (URL 구조 유지, 공백·한글만 인코딩)

encodeURIComponent('홍길동 & 김철수')
// → "%ED%99%8D%EA%B8%B8%EB%8F%99%20%26%20%EA%B9%80%EC%B2%A0%EC%88%98"
// (& 포함 모든 특수문자 인코딩 — 값 하나에 사용)
```

```txt
언제 뭘 쓰는가:
  쿼리스트링 여러 파라미터   → URLSearchParams (자동)
  URL path의 값 하나        → encodeURIComponent (수동)
  URL 전체를 인코딩         → encodeURI (거의 안 씀)
```

---

# new URL() — URL 파싱 · 조작 ⭐️⭐️⭐️

```typescript
// URL 파싱
const url = new URL('https://example.com/users?q=홍길동&limit=20');

url.protocol  // 'https:'
url.hostname  // 'example.com'
url.pathname  // '/users'
url.search    // '?q=%ED%99%8D...&limit=20'
url.searchParams.get('q')     // '홍길동'  (자동 디코딩)
url.searchParams.get('limit') // '20'

// URL 조작
const url2 = new URL('https://example.com/users');
url2.searchParams.set('q', '홍길동');
url2.searchParams.set('limit', '20');
url2.toString()
// → 'https://example.com/users?q=%ED%99%8D...&limit=20'
```

```txt
new URL()의 장점:
  protocol, hostname, pathname, search를 각각 조작 가능
  searchParams가 URLSearchParams 인스턴스 → set/get/delete 사용 가능
  브라우저 환경에서 현재 URL 파싱할 때 유용

  window.location.href 대신:
  const url = new URL(window.location.href);
  url.searchParams.get('cursor');  // 현재 URL의 cursor 파라미터
```

---

# 실전 패턴

## 페이지네이션 + 검색 조합 ⭐️⭐️⭐️⭐️

```typescript
// [[NestJS_Pagination]] 참고
export function fetchItems(params: {
  q?:      string;
  cursor?: string;
  take?:   number;
} = {}): Promise<CursorPage<UiItem>> {
  const sp = new URLSearchParams();

  if (params.q?.trim())    sp.set('q',      params.q.trim());
  if (params.cursor)       sp.set('cursor', params.cursor);
  if (params.take != null) sp.set('take',   params.take.toString());

  const qs = sp.toString();
  return apiFetch<CursorPage<ApiItem>>(`/items${qs ? `?${qs}` : ''}`);
}
```

```txt
params.q?.trim():
  q가 undefined면 ?. 로 조용히 undefined 반환 → if 조건 false → 추가 안 함
  q가 '   '처럼 공백만 있으면 trim() 후 빈 문자열 → falsy → 추가 안 함
  q가 '홍길동'이면 trim() 후 '홍길동' → truthy → sp.set()
```

## 현재 URL에서 파라미터 읽기 (Next.js)

```typescript
// app/search/page.tsx — Server Component
export default function SearchPage({
  searchParams,
}: {
  searchParams: { q?: string; cursor?: string };
}) {
  const q      = searchParams.q      ?? '';
  const cursor = searchParams.cursor ?? undefined;
  // 이미 디코딩된 값으로 들어옴
}

// Client Component — useSearchParams
import { useSearchParams } from 'next/navigation';

function SearchComponent() {
  const searchParams = useSearchParams();
  const q      = searchParams.get('q')      ?? '';
  const cursor = searchParams.get('cursor') ?? undefined;
}
```

## window.location.search — 현재 URL 쿼리스트링 ⭐️⭐️⭐️⭐️

```typescript
// 현재 URL이 /search?q=홍길동&cursor=abc&limit=20 라면

window.location.search   // "?q=%ED%99%8D%EA%B8%B8%EB%8F%99&cursor=abc&limit=20"
//                           ↑ ? 포함, 인코딩된 상태
```

```txt
window.location.search란:
  현재 브라우저 URL에서 ? 이후 쿼리스트링 부분을 문자열로 반환
  인코딩된 상태 그대로 옴 → 직접 쓰면 깨진 문자열
  → URLSearchParams에 넘겨서 파싱해야 함
```

```typescript
// URLSearchParams로 파싱 — 자동 디코딩
const params = new URLSearchParams(window.location.search);

params.get('q')       // '홍길동'  (디코딩됨)
params.get('cursor')  // 'abc'
params.get('limit')   // '20'     (항상 string)
params.get('없는키')   // null

// 한 줄로 특정 파라미터만
const id = new URLSearchParams(window.location.search).get('id')?.trim();
//                                                              ↑ null이면 undefined
```

```typescript
// window.location 전체 구조
window.location.href      // "https://example.com/search?q=홍길동#top"
window.location.origin    // "https://example.com"
window.location.pathname  // "/search"
window.location.search    // "?q=홍길동&cursor=abc"   ← 쿼리스트링
window.location.hash      // "#top"                   ← 해시
```

```txt
⚠️ SSR(Next.js) 주의:
  window.location은 브라우저에만 있음
  서버(SSR)에서는 ReferenceError 발생
  → useEffect 안에서만 사용, 또는 Next.js의 useSearchParams() 훅 사용

Next.js에서 쿼리 파라미터를 읽는 방법 비교:
  Server Component  → page.tsx의 searchParams prop
  Client Component  → useSearchParams() 훅 (Next.js)
  순수 JS           → new URLSearchParams(window.location.search)
                       (useEffect 안에서만, SSR 없는 환경)
```

## URL 직접 조합이 괜찮은 경우

```typescript
// 값이 단순 숫자나 고정 문자열 — 특수문자 없음
const url = `/users/${userId}`;          // UUID는 안전 (하이픈·알파벳·숫자만)
const url = `/rooms/${roomId}/messages`; // 동일

// 특수문자가 들어갈 가능성이 없는 경우만
// 검색어·사용자 입력·한글이 포함될 수 있으면 반드시 URLSearchParams
```