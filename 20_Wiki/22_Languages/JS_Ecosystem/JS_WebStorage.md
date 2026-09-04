---
aliases:
  - localStorage
  - sessionStorage
  - IndexedDB
  - storage event
tags:
  - JavaScript
related:
  - "[[00_JS_Ecosystem_HomePage]]"
  - "[[JS_JSON]]"
  - "[[NextJS_TokenStorage]]"
---
# JS_WebStorage — 브라우저 저장소

> [!info]
> `localStorage`는 브라우저에 영구 저장, `sessionStorage`는 탭 단위 저장 — 둘 다 **string만** 저장되므로 객체는 `JSON.stringify/parse` 필수.

---

# 언제 이 파일이 필요한가

```txt
새로고침해도 유지해야 하는 상태가 생겼을 때:
  사용자 설정 (다크모드, 언어, 필터 선택)
  최근 검색어 / 최근 방문 항목
  폼 입력 임시 저장 (탭 닫히면 사라져도 되는 것)

에러가 발생했을 때:
  "ReferenceError: localStorage is not defined"
  → Next.js Server Component에서 직접 접근한 경우

  "QuotaExceededError"
  → 5MB 용량 초과 — try/catch 누락 또는 대용량 데이터 저장 시도

설계 결정이 필요할 때:
  "이 데이터를 어디에 저장하지?" — cookie / localStorage / sessionStorage / IndexedDB
  "탭을 닫아도 유지해야 하나?" — localStorage vs sessionStorage
  "다른 탭에서도 반영돼야 하나?" — storage 이벤트 패턴
```

---

# 브라우저 저장소란 — 왜 필요한가 ⭐️⭐️⭐️⭐️

```txt
JavaScript 변수(메모리):
  const user = { name: '홍길동' };
  → 새로고침하면 사라짐 — 페이지가 닫히면 메모리가 해제됨

브라우저 저장소:
  브라우저가 디스크에 관리하는 key-value 공간
  페이지를 닫아도 (localStorage) 또는 탭 단위로 (sessionStorage) 유지
  서버에 저장 X → 사용자 브라우저 로컬에 저장
  개발자 도구 → Application → Storage에서 확인 가능

same-origin 규칙:
  저장소는 origin(프로토콜 + 도메인 + 포트)이 같은 페이지끼리만 공유됨
  http://a.com  ≠  http://b.com     도메인 다름
  http://a.com  ≠  https://a.com    프로토콜 다름
  http://a.com  ≠  http://a.com:8080  포트 다름
  → 같은 사이트처럼 보여도 origin이 다르면 접근 불가
```

## 브라우저 저장소 종류 비교

|저장소|유지 기간|용량|서버 전송|JS 접근|용도|
|---|---|---|---|---|---|
|메모리(변수)|페이지 닫으면 소멸|제한 없음|X|O|렌더링 중 임시 상태|
|Cookie|직접 설정(만료일)|4KB|✅ 자동|O (HttpOnly 아니면)|인증 토큰 (HttpOnly 권장)|
|`sessionStorage`|탭 닫으면 소멸|~5MB|X|O|탭별 임시 상태, 폼 draft|
|`localStorage`|영구 (직접 삭제 전)|~5MB|X|O|사용자 설정, 최근 항목|
|IndexedDB|영구|수백MB~|X|O (비동기)|대용량 데이터, 구조적 쿼리|

```txt
Cookie vs localStorage:
  Cookie     HTTP 요청마다 서버로 자동 전송 → 인증에 씀
             HttpOnly 설정 시 JS 접근 불가 → XSS 공격에 안전
  localStorage  서버로 안 가고 브라우저만 앎 → JS로만 읽고 씀
             XSS 공격에 취약 (악성 스크립트도 읽을 수 있음)
  → 인증 토큰 저장 전략 → [[NextJS_TokenStorage]]

IndexedDB로 가야 하는 시점:
  저장 데이터가 5MB를 넘어갈 때
  배열을 인덱스로 쿼리해야 할 때
  비동기 I/O가 필요할 때 (localStorage는 동기 → 메인 스레드 블로킹)
```

---

# localStorage vs sessionStorage ⭐️⭐️⭐️⭐️

|구분|localStorage|sessionStorage|
|---|---|---|
|유지 기간|영구 (직접 지우기 전까지)|탭/창 닫으면 소멸|
|범위|같은 origin의 모든 탭|현재 탭만|
|탭 간 공유|✅ 같은 origin 탭 전부 공유|❌ 이 탭에서만|
|새로고침|✅ 유지|✅ 유지|
|브라우저 닫기|✅ 유지|❌ 소멸|
|용량|약 5MB|약 5MB|
|API|동일|동일|

```txt
localStorage 쓰는 경우:
  사용자 설정 (다크모드, 언어, 필터)
  최근 검색어 / 최근 방문 항목
  로그인 유지 기억 (보안 고려 필요)

sessionStorage 쓰는 경우:
  폼 입력 임시 저장 (탭 닫으면 삭제돼야 하는 draft)
  탭별로 독립된 상태 (같은 사이트를 여러 탭에서 다르게 쓸 때)
  step-by-step 마법사 폼의 중간 상태

판단 기준:
  "브라우저 닫고 나서도 기억해야 하는가?" → localStorage
  "이 탭에서만, 탭 닫으면 버려도 되는가?" → sessionStorage
```

---

# API — 읽기 · 쓰기 · 삭제 ⭐️⭐️⭐️⭐️

```typescript
// localStorage / sessionStorage 완전히 동일한 API
const storage = localStorage; // sessionStorage도 동일

// 저장
storage.setItem('theme', 'dark');

// 읽기 — 없는 키는 null 반환
storage.getItem('theme');   // 'dark'
storage.getItem('없는키');  // null

// 삭제
storage.removeItem('theme');

// 전체 삭제 (이 origin의 저장소 전체)
storage.clear();

// 키 개수
storage.length;  // 1

// 인덱스로 키 이름 읽기 (순서는 보장 안 됨)
storage.key(0);  // 'theme'
```

```txt
getItem 반환 타입: string | null
  없는 키 → null 반환 (에러 아님)
  → 반드시 null 체크 후 사용:
    const raw = localStorage.getItem('key');
    const value = raw ?? '기본값';

⚠️ 사생활 보호 모드(Private Browsing):
  일부 브라우저에서 setItem 호출 시 SecurityError 발생
  → 반드시 try/catch로 감쌀 것
```

---

# string만 저장 — JSON 직렬화 필수 ⭐️⭐️⭐️⭐️

```typescript
// ❌ 객체를 그냥 저장하면
localStorage.setItem('user', { name: '홍길동', id: 1 });
localStorage.getItem('user');  // '[object Object]' (쓸모없음)

// ✅ JSON.stringify로 직렬화
localStorage.setItem('user', JSON.stringify({ name: '홍길동', id: 1 }));
localStorage.getItem('user');  // '{"name":"홍길동","id":1}'

// 읽을 때 JSON.parse
const raw  = localStorage.getItem('user');
const user = raw ? JSON.parse(raw) : null;
```

## 안전한 get/set 유틸

```typescript
// JSON.parse 실패 / 용량 초과 / 사생활 보호 모드 대비 try/catch
function storageGet<T>(key: string, fallback: T): T {
  try {
    const raw = localStorage.getItem(key);
    return raw !== null ? (JSON.parse(raw) as T) : fallback;
  } catch {
    return fallback;
  }
}

function storageSet(key: string, value: unknown): void {
  try {
    localStorage.setItem(key, JSON.stringify(value));
  } catch {
    // QuotaExceededError, SecurityError 등
  }
}

storageSet('settings', { theme: 'dark', lang: 'ko' });
const settings = storageGet('settings', { theme: 'light', lang: 'en' });
```

## 특수 타입 직렬화 — Date · Set · Map ⭐️⭐️⭐️

```typescript
// Date — JSON.stringify 시 ISO 문자열로 변환, 읽을 때 복원 필요
const date = new Date();
localStorage.setItem('date', JSON.stringify(date));  // '"2026-09-04T00:00:00.000Z"'

const raw = localStorage.getItem('date');
const restored = raw ? new Date(JSON.parse(raw)) : null;
// JSON.parse → ISO 문자열 → new Date() → Date 객체 복원

// Set — JSON.stringify가 {} (빈 객체)로 변환 → 배열 변환 후 저장
const mySet = new Set(['a', 'b', 'c']);
localStorage.setItem('mySet', JSON.stringify([...mySet]));  // '["a","b","c"]'

const rawSet = localStorage.getItem('mySet');
const restoredSet = new Set<string>(rawSet ? JSON.parse(rawSet) : []);

// Map — entries()로 배열 변환 후 저장
const myMap = new Map([['key1', 'val1'], ['key2', 'val2']]);
localStorage.setItem('myMap', JSON.stringify([...myMap.entries()]));

const rawMap = localStorage.getItem('myMap');
const restoredMap = new Map<string, string>(rawMap ? JSON.parse(rawMap) : []);
```

```txt
JSON.stringify가 의도대로 동작하지 않는 타입:
  Date      ISO 문자열로 자동 변환됨 — 읽을 때 new Date() 감싸야 복원
  Set       {} (빈 객체) — [...set]으로 배열 변환 후 저장
  Map       {} (빈 객체) — [...map.entries()]로 배열 변환 후 저장
  undefined 키 자체가 사라짐 (null로 변환)
  함수, Symbol  무시됨 (직렬화 불가)
```

---

# 실전 패턴 ⭐️⭐️⭐️⭐️

## 사용자 설정 저장 — 다크모드

```typescript
const THEME_KEY = 'app_theme';

export function getTheme(): 'light' | 'dark' {
  if (typeof window === 'undefined') return 'light';  // SSR 안전 체크
  return (localStorage.getItem(THEME_KEY) as 'light' | 'dark') ?? 'light';
}

export function setTheme(theme: 'light' | 'dark'): void {
  localStorage.setItem(THEME_KEY, theme);
}
```

## 배열 저장 + upsert + 최대 개수 관리

최근 검색어, 최근 방문 장소, 최근 열람 항목 등 목록 관리 패턴.

```typescript
const STORAGE_KEY = 'app.recent-items';
const MAX_ITEMS   = 8;

function getRecentItems<T>(): T[] {
  try {
    const raw = localStorage.getItem(STORAGE_KEY);
    return raw ? (JSON.parse(raw) as T[]) : [];
  } catch {
    return [];
  }
}

// upsert: 동일 항목 제거 → 맨 앞에 추가 → 최대 개수 제한
function addRecentItem<T>(item: T, isSame: (a: T, b: T) => boolean): void {
  const items = getRecentItems<T>();
  const updated = [item, ...items.filter((i) => !isSame(i, item))].slice(0, MAX_ITEMS);
  try {
    localStorage.setItem(STORAGE_KEY, JSON.stringify(updated));
  } catch {}
}
```

```typescript
// 사용 예 — 장소 이름 normalize 기준 중복 제거
type RecentPlace = { name: string; address: string };
const normalize = (v: string) => v.replace(/\s+/g, '').toLowerCase();

addRecentItem<RecentPlace>(
  { name: '스타벅스 강남역점', address: '서울 강남구' },
  (a, b) => normalize(a.name) === normalize(b.name),
);
```

```txt
upsert 동작 순서:
  1. filter로 동일 항목(isSame 기준) 기존 데이터 제거
  2. 새 항목을 맨 앞에 추가 (spread)
  3. slice(0, MAX_ITEMS)로 최대 개수 제한
  → 동일 항목 재선택 시 가장 최근 선택이 맨 위로 올라옴

normalize 기준 비교를 쓰는 이유:
  id가 달라도 "스타벅스강남역점" / "스타벅스 강남역점"이 같은 장소여야 함
  → id 기준 중복 제거만으로는 공백·대소문자 차이를 잡지 못함
```

## storage 이벤트 — 탭 간 동기화 ⭐️⭐️⭐️

```typescript
// localStorage 변경 시 같은 origin의 다른 탭에 storage 이벤트 발생
// (같은 탭에서 setItem해도 그 탭에는 이벤트 발생 안 함)
window.addEventListener('storage', (e: StorageEvent) => {
  if (e.key === 'theme') {
    applyTheme(e.newValue as 'light' | 'dark');
  }
});

// StorageEvent 주요 속성
// e.key         변경된 키 (clear() 호출 시 null)
// e.oldValue    변경 전 값 (string | null)
// e.newValue    변경 후 값 (삭제면 null)
// e.url         변경이 일어난 페이지 URL
// e.storageArea localStorage 또는 sessionStorage 객체
```

```txt
storage 이벤트가 유용한 경우:
  탭 A에서 로그아웃 → 탭 B에서도 자동 로그아웃 처리
  탭 A에서 테마 변경 → 탭 B에서도 즉시 반영
  탭 A에서 장바구니 추가 → 탭 B 배지 카운트 업데이트

⚠️ sessionStorage는 탭 간 공유 안 됨 → storage 이벤트도 안 발생
⚠️ 같은 탭에서 setItem한 것은 이벤트 안 발생 — 다른 탭에서만 수신
```

---

# Next.js에서 사용 시 주의 ⭐️⭐️⭐️⭐️

```typescript
// ❌ Server Component에서 직접 사용하면 에러
localStorage.getItem('theme');  // ReferenceError: localStorage is not defined

// ✅ 유틸 함수에서 — typeof window 체크
export function getTheme(): 'light' | 'dark' {
  if (typeof window === 'undefined') return 'light';
  return (localStorage.getItem('theme') as 'light' | 'dark') ?? 'light';
}

// ✅ 컴포넌트에서 — 'use client' + useEffect 안에서만
'use client';
import { useEffect, useState } from 'react';

function ThemeProvider() {
  const [theme, setTheme] = useState<'light' | 'dark'>('light');

  useEffect(() => {
    // 클라이언트(브라우저)에서만 실행됨
    const saved = (localStorage.getItem('theme') as 'light' | 'dark') ?? 'light';
    setTheme(saved);
  }, []);

  return <div data-theme={theme}>...</div>;
}
```

```txt
typeof window === 'undefined' 체크가 필요한 이유:
  Next.js는 Server Component를 서버(Node.js)에서 실행
  Node.js에는 window, document, localStorage가 없음
  → 체크 없이 쓰면 ReferenceError

패턴 선택:
  유틸 함수에서 → typeof window 체크로 null/기본값 반환
  컴포넌트에서  → 'use client' + useEffect 안에서만 접근
```

---

# 한계와 주의사항 ⭐️⭐️⭐️

|한계|내용|대안|
|---|---|---|
|string만 저장|객체·배열은 JSON 필수, Date·Set·Map은 수동 변환 필요|—|
|용량 ~5MB|초과 시 QuotaExceededError 발생|IndexedDB|
|동기 API|읽기·쓰기가 메인 스레드를 블로킹|IndexedDB (비동기)|
|XSS 취약|JS로 직접 접근 가능 → 악성 스크립트도 읽음|HttpOnly Cookie|
|same-origin 격리|origin 다르면 접근 불가, 공유 안 됨|—|
|사생활 보호 모드|브라우저에 따라 setItem이 SecurityError 발생|try/catch 필수|

```txt
try/catch를 반드시 써야 하는 이유:
  ① QuotaExceededError — 5MB 초과 시 setItem 실패
  ② SecurityError     — 사생활 보호 모드에서 발생하는 브라우저 있음
  ③ JSON.parse 실패   — 손상된 데이터를 읽을 때
  → localStorage를 쓰는 모든 읽기/쓰기는 try/catch로 감쌀 것

XSS 취약성:
  localStorage에 저장된 값은 JS로 직접 읽을 수 있음
  XSS 공격 성공 시 악성 스크립트가 값을 탈취 가능
  → 인증 토큰 같은 민감한 정보는 HttpOnly Cookie 권장
  → 상세 → [[NextJS_TokenStorage]]
```
