---
aliases:
  - localStorage
  - sessionStorage
tags:
  - JavaScript
related:
  - "[[00_JS_Ecosystem_HomePage]]"
  - "[[JS_JSON]]"
  - "[[NextJS_TokenStorage]]"
---
# JS_WebStorage — localStorage · sessionStorage

>[!info]
>`localStorage` = 브라우저에 영구 저장 (탭·브라우저 닫아도 유지).
> `sessionStorage` = 탭 단위 저장 (탭 닫으면 소멸). 
> 둘 다 **string만 저장** → 객체는 `JSON.stringify/parse` 필요. 
> 인증 토큰 저장 전략(보안·XSS) → [[NextJS_TokenStorage]]

---

# 왜 브라우저 저장소가 필요한가 ⭐️⭐️⭐️⭐️

```txt
JavaScript 변수(메모리):
  const user = { name: '홍길동' };
  → 새로고침하면 사라짐

localStorage / sessionStorage:
  브라우저가 관리하는 저장소
  페이지를 닫아도 (localStorage) 또는 탭 단위로 (sessionStorage) 유지

어디에 저장되는가:
  서버에 저장 X → 사용자 브라우저에 저장
  같은 도메인(origin)에서만 접근 가능
  개발자 도구 → Application → Storage에서 확인 가능
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

```txt
localStorage를 쓰는 경우:
  사용자 설정 (다크모드, 언어)
  마지막으로 본 항목
  로그인 상태 유지 (보안 고려 필요)

sessionStorage를 쓰는 경우:
  폼 입력 임시 저장 (탭 닫으면 삭제돼야 함)
  탭별로 독립된 상태 (여러 탭에서 다른 작업)
  일회성 세션 데이터
```

---

# localStorage API ⭐️⭐️⭐️⭐️

```typescript
// 저장
localStorage.setItem('theme', 'dark');
localStorage.setItem('lang', 'ko');

// 읽기
localStorage.getItem('theme');    // 'dark'
localStorage.getItem('없는키');  // null (없으면 null 반환)

// 삭제
localStorage.removeItem('theme');

// 전체 삭제
localStorage.clear();

// 키 개수
localStorage.length;  // 2

// 인덱스로 키 이름 읽기
localStorage.key(0);  // 'lang'
```

---

# 객체 저장 — JSON 필수 ⭐️⭐️⭐️⭐️

```typescript
// ❌ 그냥 저장하면
localStorage.setItem('user', { name: '홍길동', id: 1 });
localStorage.getItem('user');  // '[object Object]' (쓸모없음)

// ✅ JSON으로 직렬화
localStorage.setItem('user', JSON.stringify({ name: '홍길동', id: 1 }));
localStorage.getItem('user');  // '{"name":"홍길동","id":1}' (문자열)

// 읽을 때 파싱
const raw  = localStorage.getItem('user');
const user = raw ? JSON.parse(raw) : null;
```

## 안전한 get/set 유틸

```typescript
// 파싱 실패 대비 try/catch
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
    // 용량 초과 등
  }
}

// 사용
storageSet('settings', { theme: 'dark', lang: 'ko' });
const settings = storageGet('settings', { theme: 'light', lang: 'en' });
```

---

# Next.js에서 사용 시 주의 ⭐️⭐️⭐️⭐️

```typescript
// ❌ Server Component에서 직접 사용하면 에러
localStorage.getItem('theme');  // ReferenceError: localStorage is not defined

// ✅ typeof window 체크
const theme = typeof window !== 'undefined'
  ? localStorage.getItem('theme')
  : null;

// ✅ 또는 'use client' + useEffect 안에서만
'use client';
import { useEffect, useState } from 'react';

function ThemeProvider() {
  const [theme, setTheme] = useState('light');  // 초기값 먼저

  useEffect(() => {
    // 클라이언트(브라우저)에서만 실행됨
    const saved = localStorage.getItem('theme') ?? 'light';
    setTheme(saved);
  }, []);

  return <div data-theme={theme}>...</div>;
}
```

```txt
typeof window === 'undefined' 체크가 필요한 이유:
  Next.js는 Server Component를 서버에서 실행
  서버에는 window, localStorage가 없음
  → 체크 없이 쓰면 ReferenceError
  → null 반환하거나 useEffect 안에서만 사용
```

---

# sessionStorage API

```typescript
// localStorage와 완전히 동일한 API
sessionStorage.setItem('temp', '임시 데이터');
sessionStorage.getItem('temp');
sessionStorage.removeItem('temp');
sessionStorage.clear();
```

```typescript
// 폼 임시 저장 예시 — 탭 닫으면 자동 삭제
function saveDraft(content: string) {
  sessionStorage.setItem('draft', content);
}

function loadDraft(): string {
  return sessionStorage.getItem('draft') ?? '';
}

function clearDraft() {
  sessionStorage.removeItem('draft');
}
```

---

# 한계와 주의사항 ⭐️⭐️⭐️

```txt
string만 저장:
  숫자, 객체, 배열 → JSON.stringify/parse 필요
  Date → 문자열로 저장됨, 읽을 때 new Date() 변환 필요

용량 제한:
  약 5MB (브라우저마다 다름)
  → 큰 데이터는 IndexedDB 사용

동기 API:
  읽기·쓰기가 메인 스레드를 블로킹
  → 대용량 데이터에 부적합

XSS 취약:
  JS로 직접 접근 가능 → 악성 스크립트도 읽을 수 있음
  → 민감한 정보(인증 토큰 등) 저장 시 보안 고려 필요
  → 토큰 저장 전략 → [[NextJS_TokenStorage]]

same-origin 규칙:
  http://a.com의 localStorage는 http://b.com에서 접근 불가
  http://a.com과 https://a.com도 다른 origin → 공유 안 됨
```

---

# 자주 쓰는 패턴

```typescript
// 다크모드 저장
const THEME_KEY = 'app_theme';

export function getTheme(): 'light' | 'dark' {
  if (typeof window === 'undefined') return 'light';
  return (localStorage.getItem(THEME_KEY) as 'light' | 'dark') ?? 'light';
}

export function setTheme(theme: 'light' | 'dark') {
  localStorage.setItem(THEME_KEY, theme);
}

// 마지막 방문 경로
sessionStorage.setItem('lastPath', window.location.pathname);

// Set 직렬화 (Set은 JSON.stringify가 안 됨)
const mySet = new Set(['a', 'b', 'c']);
localStorage.setItem('mySet', JSON.stringify([...mySet]));
const restored = new Set(JSON.parse(localStorage.getItem('mySet') ?? '[]'));
```