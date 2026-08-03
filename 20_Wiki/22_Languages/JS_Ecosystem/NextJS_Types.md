---
aliases:
  - ApiTypes
  - uiTypes
tags:
  - NextJS
related:
  - "[[00_JS_Ecosystem_HomePage]]"
  - "[[NestJS_DTO]]"
  - "[[NextJS_API_Client]]"
  - "[[NextJS_API_Mapper]]"
---
# NextJS_Types — API 타입 · UI 타입 · 매퍼

> [!info]
>  API 타입(ApiXxx) = 서버가 보내주는 것의 형태. 
>  UI 타입(UiXxx) = 화면이 필요로 하는 형태. 둘이 같으면 그냥 쓰고, 다르면 매퍼로 변환한다. 
>  NestJS DTO → OpenAPI → ApiXxx → (매퍼) → UiXxx → 컴포넌트.

---

# 전체 흐름 — 타입이 어디서 와서 어디로 가는가 ⭐️⭐️⭐️⭐️

```txt
NestJS 백엔드                       Next.js 프론트엔드
┌──────────────┐                   ┌─────────────────────────────┐
│ DTO          │                   │                             │
│ @ApiProperty │ → Swagger JSON →  │ ApiXxx 타입 (자동 생성)       │
│              │                   │    ↓ (매퍼 — 필요할 때만)     │
└──────────────┘                   │ UiXxx 타입                   │
                                   │    ↓                        │
                                   │ React 컴포넌트               │
                                   └─────────────────────────────┘
```

```txt
핵심 질문 하나:
  "API 타입을 그냥 써도 되는가?"
  → 예  : ApiXxx를 컴포넌트에 직접 넘김 (매퍼 불필요)
  → 아니오: UiXxx 타입을 별도로 만들고 매퍼로 변환

"아니오"가 되는 경우:
  날짜가 string인데 Date 객체로 쓰고 싶을 때
  API에 없는 UI 전용 필드가 필요할 때 (isSelected, isExpanded 등)
  여러 API 응답을 합쳐서 하나의 타입으로 만들 때
  null/undefined가 너무 많아서 컴포넌트에서 쓰기 불편할 때
```

---

# API 타입 (ApiXxx) — 서버가 보내주는 것의 형태 ⭐️⭐️⭐️⭐️

```txt
API 타입 = NestJS DTO가 응답으로 내보내는 것을 TypeScript로 표현
  보통 prefix를 Api로 붙여서 "서버에서 온 것"임을 명시
  ApiUser, ApiPost, ApiRoom 등
```

## 방법 1 — openapi-typescript로 자동 생성 (권장)

```bash
pnpm add -D openapi-typescript
npx openapi-typescript http://localhost:3000/api-json -o src/types/api.ts
```

```typescript
// 자동 생성된 api.ts에서 꺼내 쓰기
import type { components } from './types/api';

type ApiUser = components['schemas']['UserDto'];
type ApiPost = components['schemas']['PostDto'];
```

```txt
NestJS에서 Swagger 스펙을 켜두면 (http://localhost:3000/api-json)
openapi-typescript가 그 스펙을 읽어서 TypeScript 타입을 자동 생성

장점:
  NestJS DTO가 바뀌면 명령어 한 번으로 타입 동기화
  수동으로 타입 관리하면 DTO 바뀌었는데 프론트 타입은 그대로인 불일치 발생
```

## 방법 2 — 수동 정의

```typescript
// 자동 생성이 안 되는 환경이거나 스펙이 불완전할 때
type ApiUser = {
  id:        string;
  nickname:  string;
  email:     string;
  image:     string | null;
  role:      'user' | 'admin';
  createdAt: string;    // API는 string으로 줌
};
```

---

# UI 타입 (UiXxx) — 화면이 필요로 하는 형태 ⭐️⭐️⭐️⭐️

```txt
UI 타입은 컴포넌트가 실제로 다루는 데이터의 형태
API 타입과 같을 수도, 다를 수도 있음
```

## API 타입을 그냥 쓰는 경우 — 매퍼 불필요

```typescript
// API 응답을 그대로 표시만 할 때
function UserCard({ user }: { user: ApiUser }) {
  return <div>{user.nickname}</div>;
}

// 타입 alias만 만들어두면 충분
type UiUser = ApiUser;  // 변환 없이 동일
```

## UI 타입이 별도로 필요한 경우

```typescript
// ① UI 전용 필드 추가
type UiUser = ApiUser & {
  isSelected: boolean;   // API에 없는 선택 상태
  isOnline:   boolean;   // API에 없는 실시간 상태
};

// ② API 타입에서 필요한 것만 뽑기
type UiUserSummary = Pick<ApiUser, 'id' | 'nickname' | 'image'>;

// ③ null을 처리해서 더 쓰기 편하게
type UiUser = Omit<ApiUser, 'image'> & {
  image: string;  // null이 아닌 기본 이미지 URL로 처리됨
};

// ④ 여러 API 응답을 합칠 때
type UiRoomDetail = ApiRoom & {
  members:      ApiMember[];  // /rooms/:id/members 응답
  lastMessage:  ApiMessage | null;  // /rooms/:id/messages?take=1 응답
};
```

---

# 매퍼 — API → UI 변환 함수 ⭐️⭐️⭐️⭐️

```txt
매퍼(mapper) = API 타입을 UI 타입으로 변환하는 함수
변환이 필요할 때만 만들고, 필요 없으면 매퍼 자체가 없어도 됨
```

## 기본 패턴

```typescript
// api-types/user.mapper.ts
function toUiUser(api: ApiUser): UiUser {
  return {
    ...api,
    image:    api.image ?? '/default-avatar.png',  // null → 기본값
    nickname: api.nickname.trim(),                 // 정리
  };
}

// 배열에 적용
const uiUsers = apiUsers.map(toUiUser);
```

## 날짜 변환 패턴

```typescript
type ApiPost = { id: string; createdAt: string };   // API는 string
type UiPost  = { id: string; createdAt: Date };      // UI는 Date 객체

function toUiPost(api: ApiPost): UiPost {
  return {
    ...api,
    createdAt: new Date(api.createdAt),
  };
}
```

```txt
날짜를 Date 객체로 변환하는 이유:
  날짜 계산 (며칠 지났는지, 정렬 등)
  date-fns, dayjs 같은 라이브러리에 바로 넘길 수 있음

반대로 string으로 두는 게 나은 경우:
  표시만 할 때 (new Date(str).toLocaleDateString())
  Intl.DateTimeFormat 바로 사용 → [[JS_Intl]]
```

## 여러 응답을 합치는 매퍼

```typescript
// 두 API 응답을 하나의 UI 타입으로
type UiRoomDetail = {
  id:          string;
  name:        string;
  members:     UiMember[];
  lastMessage: UiMessage | null;
};

async function fetchRoomDetail(roomId: string): Promise<UiRoomDetail> {
  const [room, members, messages] = await Promise.all([
    apiFetch<ApiRoom>(`/rooms/${roomId}`),
    apiFetch<ApiMember[]>(`/rooms/${roomId}/members`),
    apiFetch<ApiMessage[]>(`/rooms/${roomId}/messages?take=1`),
  ]);

  return {
    id:          room.id,
    name:        room.name,
    members:     members.map(toUiMember),
    lastMessage: messages[0] ? toUiMessage(messages[0]) : null,
  };
}
```

---

# 어디서 변환하는가 — 매퍼 위치 ⭐️⭐️⭐️

```typescript
// ① API 클라이언트 레이어에서 변환 (권장)
// api/users.ts
export async function fetchUser(id: string): Promise<UiUser> {
  const api = await apiFetch<ApiUser>(`/users/${id}`);
  return toUiUser(api);  // 여기서 변환
}

// 컴포넌트에서는 UiUser만 다룸
const user = await fetchUser(id);  // 이미 UiUser
```

```typescript
// ② 컴포넌트에서 변환 (API 응답 직접 노출)
const apiUser = await apiFetch<ApiUser>(`/users/${id}`);
const uiUser  = toUiUser(apiUser);  // 컴포넌트 레벨에서 변환
```

```txt
API 클라이언트 레이어에서 변환을 권장하는 이유:
  컴포넌트가 API 타입을 직접 알 필요 없음
  변환 로직이 한 곳에 모임 → API 응답 형태가 바뀌면 매퍼만 수정
  컴포넌트는 항상 UiXxx만 다루어서 일관성 유지
```

---

# 판단 흐름 — 실전에서 어떻게 결정하는가 ⭐️⭐️⭐️⭐️

```txt
새 API를 연결할 때 스스로 묻는 질문:

1. API 응답을 그대로 컴포넌트에 넘겨도 되는가?
   → 예: type UiXxx = ApiXxx (또는 그냥 ApiXxx 바로 사용)
   → 아니오: 계속

2. 무엇이 문제인가?
   → null이 불편함     : null 처리 매퍼
   → 날짜가 string    : Date 변환 매퍼
   → UI 전용 필드 필요 : ApiXxx & { 추가 필드 }
   → 여러 API 합치기  : 합치는 매퍼 함수

3. 변환이 복잡한가?
   → 예: toUiXxx() 함수로 분리
   → 아니오: 인라인으로 { ...api, image: api.image ?? '/default.png' }
```

---

# 파일/폴더 구조

```txt
src/
├── types/
│   ├── api.ts          자동 생성 API 타입 (openapi-typescript 결과)
│   └── ui/
│       ├── user.ts     UiUser, UiUserSummary 등
│       ├── post.ts     UiPost, UiPostDetail 등
│       └── room.ts     UiRoom, UiRoomDetail 등
├── api/
│   ├── users.ts        fetchUser() — ApiUser → UiUser 변환 포함
│   ├── posts.ts        fetchPost() — 변환 포함
│   └── rooms.ts
└── components/
    └── UserCard.tsx    UiUser만 알고 있음, API 타입 모름
```

```txt
컴포넌트가 Api 타입을 직접 import하지 않는 게 이상적
  컴포넌트: UiUser 사용
  api/users.ts: ApiUser → UiUser 변환 담당
  API 응답 형태가 바뀌어도 컴포넌트는 수정 불필요
```