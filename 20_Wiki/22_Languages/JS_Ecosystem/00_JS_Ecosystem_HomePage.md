---
aliases:
  - 00_JS_Ecosystem_HomePage — JS · TS · React · Next.js
tags:
  - HomePage
related:
  - "[[00_NestJS_Ecosystem_HomePage]]"
  - "[[00_DB_HomePage]]"
cssclasses:
  - max
  - table-max
  - table-wrap
---
# 00_JS_Ecosystem_HomePage — JS · TS · React · Next.js

>[!info]
>이 넷은 같은 런타임(JS)과 같은 타입 시스템(TS) 위에서 React가 컴포넌트 모델을, Next.js가 그 위의 프레임워크를 얹은 한 묶음이라 폴더를 합쳤다.

```mermaid-beautiful
flowchart LR
    JS["JavaScript"] --> TS["TypeScript"] --> REACT["React"] --> NEXT["Next.js"]
```

---

# 빠른 찾기

| 찾을 때 | 섹션 |
|---|---|
| 인증 / JWT / 토큰 | [[#🔐 인증 · 토큰]] |
| DOM / Canvas / 파일 | [[#🌐 브라우저 · DOM · Canvas]] |
| 스타일 / 폰트 | [[#🎨 스타일링]] |
| React 훅 | [[#⚛️ React 훅]] |
| 폼 / 입력 | [[#📝 폼 처리]] |
| API 통신 / 타입 | [[#📡 API 통신 · 타입 매핑]] |
| 라우팅 / 메타 | [[#🗺️ 라우팅 · 메타데이터]] |
| 날짜 / 문자열 / 유틸 | [[#📅 날짜 · 문자열 · 유틸]] |
| JS · TS 문법 기초 | [[#🔤 범용 문법 기초]] |
| 보안 | [[#🛡️ 보안]] |
| 독립 노트 | [[#📦 독립 노트]] |

---

## 🔐 인증 · 토큰

| | 노트 |
|---|---|
| **Next.js** | [[Auth_Concept]] · [[NextJS_TokenStorage]] · [[NextJS_AuthCache]] |

```txt
Auth_Concept        OAuth 흐름 · JWT vs 세션
NextJS_TokenStorage 토큰 저장 전략 (메모리 · 쿠키)
NextJS_AuthCache    캐시 + 인증 조합
Context로 로그인 상태 공유 → [[React_Context_Provider]] (⚛️ React 훅 섹션)
API 클라이언트 · 토큰 첨부 → [[NextJS_API_Client]] (📡 API 통신 섹션)
로그인 라우팅 → [[NextJS_Routing]] (🗺️ 라우팅 섹션)
```

---

## 🌐 브라우저 · DOM · Canvas

| | 노트 |
|---|---|
| **JS** | [[JS_BrowserAPI]] · [[JS_DOM]] · [[JS_Canvas]] · [[JS_FileAPI]] · [[JS_CustomEvent]] |
| **TS** | [[TS_DOM_Events]] |
| **Next.js** | [[NextJS_ServerClient]] |

```txt
JS_BrowserAPI  window · navigator · Clipboard · ResizeObserver · style 직접 조작
JS_DOM         querySelector · classList · Pointer Events · scrollIntoView · textarea 패턴
JS_Canvas      Canvas 2D · StrokeLayer 패턴 · 정규화 좌표(0~1)
JS_FileAPI     File 객체 · FileReader · dataURL · 파일 검증
DOM 요소 접근(useRef) → [[React_useRef]] (⚛️ React 훅 섹션)
```

---

## 🎨 스타일링

| | 노트 |
|---|---|
| **React** | [[React_CSSProperties]] · [[React_Styling]] · [[React_LucideIcons]] |
| **Next.js** | [[NextJS_Font]] |

```txt
NextJS_Font        next/font · 한글 폰트 · 커스텀 클래스 · CSS 변수
React_LucideIcons  아이콘 설치 · props · 동적 아이콘
classList 조작 → [[JS_DOM]] / style 직접 조작 → [[JS_BrowserAPI]] (🌐 섹션)
```

---

## ⚛️ React 훅

| 훅 / 역할            | 노트                             |
| ----------------- | ------------------------------ |
| **타입**            | [[React_Types]]                |
| 메모이제이션            | [[React_useMemo_useCallback]]  |
| 사이드 이펙트           | [[React_useEffect]]            |
| DOM 접근 · 값 보관     | [[React_useRef]]               |
| 전역 상태 공유          | [[React_Context_Provider]]     |
| 비동기 UI 패턴         | [[React_AsyncUI]]              |
| 고유 ID             | [[React_useId]]                |
| Portal · 모달 다이얼로그 | [[React_Portal_Dialog]]        |
| Suspense          | [[React_Suspense]]             |
| 외부 스토어            | [[React_useSyncExternalStore]] |

```txt
React_Types               children · ReactNode · 이벤트 타입 · Ref 타입
React_useMemo_useCallback  메모이제이션 · 언제 쓰는지 판단 · React.memo
React_useEffect           데이터 fetch · 이벤트 리스너 · cleanup · 의존성 · 무한루프
React_useRef              DOM 접근(focus · scroll) · 렌더링 무관 값 보관
React_AsyncUI             이벤트 핸들러 비동기 · fire-and-forget · applyLocal
```

---

## 📝 폼 처리

| | 노트 |
|---|---|
| **JS** | [[JS_FormData]] |
| **React** | [[React_useFormStatus]] · [[React_ControlledInput]] · [[React_DatePicker]] |
| **Next.js** | [[NextJS_Server_Actions]] |

---

## 📡 API 통신 · 타입 매핑

| | 노트 |
|---|---|
| **JS** | [[JS_Fetch_API]] |
| **Next.js** | [[NextJS_API_Client]] · [[NextJS_API_Mapper]] · [[NextJS_Types]] |

```txt
NextJS_API_Client  apiFetch 래퍼 · 토큰 · 에러 처리 · FormData
NextJS_API_Mapper  기능별 fetch 함수 · ApiXxx→UiXxx 변환 · 전체 흐름
NextJS_Types       API 타입(ApiXxx) · UI 타입(UiXxx) · 매퍼 패턴 (개념)
                   ← 백엔드 NestJS_DTO의 OpenAPI 타입 생성과 연결
```

---

## 🗺️ 라우팅 · 메타데이터

| | 노트 |
|---|---|
| **Next.js** | [[NextJS_Routing]] · [[NextJS_Metadata]] · [[NextJS_OGImage]] · [[NextJS_WebSocket]] |
| **패턴** | [[WebSocket_Patterns]] |

```txt
NextJS_OGImage    OG 이미지 · Apple 아이콘 · ImageResponse
NextJS_WebSocket  socket.io-client · 싱글턴 · emit 구조 · on/off · 재연결
WebSocket_Patterns 서버+클라이언트 패턴 — 연결·룸·브로드캐스트·개인 알림 양쪽 코드
```

---

## 📅 날짜 · 문자열 · 유틸

| 노트 | 내용 |
|---|---|
| [[JS_Date]] | Date 객체 내부(ms) · 계산 · 비교 · 타임존 주의 |
| [[JS_JSON]] | stringify · parse · unknown 패턴 |
| [[JS_WebStorage]] | localStorage · sessionStorage · Set 직렬화 |
| [[JS_Intl]] | DateTimeFormat · 상대시간("5분 전") · NumberFormat · 통화 |
| [[JS_Regex]] | test · match · 캡처그룹 · 시간 파싱 |
| [[JS_URL_Encoding]] | encodeURIComponent · new URL · URLSearchParams |

---

## 🔤 범용 문법 기초

### TypeScript

| 노트 | 내용 |
|---|---|
| [[TS_Type_Guards]] | typeof · instanceof · in · is · never · any vs unknown · as const |
| [[TS_Generics]] | `<T>` · keyof · Partial · readonly T[] · `&` 교차 타입 |
| [[TS_Utility_Types]] | Record · Partial · Omit · ReturnType · Awaited |
| [[TS_TypeAssertion]] | `as` · `!` non-null · `property!` 확정 할당 · `satisfies` |
| [[TS_ImportType]] | import type · type as alias · .d.ts · 경로 별칭 |
| [[TS_TsConfig]] | API vs Web 옵션 비교 |
| [[TS_Class_Patterns]] | implements · extends · readonly |
| [[TS_PartialUpdate]] | PATCH 객체 만들기 |

### JavaScript

| 노트 | 내용 |
|---|---|
| [[JS_Operators]] | 구조분해 · 스프레드 · ?. · ?? · ! · !! · Boolean() · Truthy/Falsy |
| [[JS_FunctionPatterns]] | 옵션 객체 패턴 · early return · async 래퍼 · 내부 함수 추출 |
| [[JS_Promise]] | async/await · Promise\<T\> 타입 · 래퍼 패턴 · Promise.all |
| [[JS_Array_Methods]] | some · filter · map · reduce · findLast · 불변성 · Set |
| [[JS_Object_Methods]] | Object.keys · entries · assign · fromEntries · Map · ID 인덱싱 |
| [[JS_Loops_Conditionals]] | if · switch · for · while |
| [[JS_Primitive_Methods]] | String · Number · Math · 문자열 조합 패턴 |

---

## 🛡️ 보안

| 노트 | 내용 |
|---|---|
| [[Web_XSS_CSRF]] | XSS · CSRF · SameSite |
| [[Web_Cookie]] | HttpOnly · 서드파티 · ITP · 프록시 |
| [[Web_Email]] | mailto · Resend · Formspree |

---

## 📦 독립 노트

| 트랙 | 노트 |
|---|---|
| **React** | [[React_Concept]] · [[React_Component]] · [[React_Vite]] |
| **Next.js** | [[NextJS_Concept]] · [[NextJS_Env_Config]] |
| **TS** | [[TS_YouTube]] |
| **도구** | [[Monorepo_PNPM]] · [[00_Deployment_HomePage]] |

---

```txt
폴더를 합친 이유:
  js / nextjs / react / typescript 네 폴더가 실제로 서로 계속 얽혀서 참조됨
  분류는 접두사(JS_ / TS_ / React_ / NextJS_)가 이미 하고 있음
  Python은 한 번도 얽힌 적 없어서 별도 유지
```