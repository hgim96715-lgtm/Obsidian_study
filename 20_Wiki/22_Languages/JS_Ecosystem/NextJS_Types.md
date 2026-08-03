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
  - "[[JS_Fetch_API]]"
  - "[[NestJS_Swagger]]"
---
# NextJS_Types — API 타입 · UI 타입 · 매퍼

> [!info] 
> API 타입(ApiXxx) = 서버가 JSON으로 보내주는 것의 형태. 
> `Date`는 JSON에 없어서 항상 `string`으로 옴
> UI 타입(UiXxx) = 화면이 필요로 하는 형태. 
> 둘이 같으면 그냥 쓰고, 다르면 매퍼로 변환. 
> NestJS DTO → OpenAPI → ApiXxx → (매퍼) → UiXxx → 컴포넌트.

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

## 방법 2 — 수동 정의 ⭐️⭐️⭐️⭐️

```txt
자동 생성이 안 되는 환경이거나 스펙이 불완전할 때 NestJS DTO를 보고 직접 작성
핵심: "서버가 실제로 JSON으로 보내주는 것"을 그대로 TypeScript 타입으로 옮김
```

### 요청 DTO ≠ 응답 타입 — 다른 층 ⭐️⭐️⭐️⭐️

```txt
처음에 헷갈리는 부분:
  ListAdminRoomsQueryDto  →  q · status · cursor · limit 4개 뿐
  ApiAdminRoom            →  id · name · description · ... 필드가 많음

  왜 이렇게 다른가?
  완전히 다른 개념이기 때문

요청 DTO (ListAdminRoomsQueryDto):
  클라이언트가 서버에 보내는 것 (쿼리 파라미터, body)
  "어떤 조건으로 가져올지"를 담음
  → 필터·정렬·페이지네이션 파라미터만 → 당연히 작음

응답 타입 (ApiAdminRoom):
  서버가 DB에서 꺼내서 클라이언트에게 주는 것
  "방 하나의 전체 정보"를 담음
  → DB 컬럼 거의 전부 + 관계 포함 → 당연히 큼

  요청 필드 4개, 응답 필드 15개 — 완전히 정상
```

### 응답에 전용 DTO 클래스가 없는 경우 — Prisma select를 보면 됨 ⭐️⭐️⭐️⭐️

```txt
NestJS에서 응답 DTO를 항상 명시적으로 만드는 건 아님

방법 1 — 전용 응답 DTO 클래스 있음:
  export class AdminRoomDto { ... }
  → 이 클래스를 보고 API 타입 작성

방법 2 — 응답 DTO 없이 Prisma select 결과를 그대로 반환:
  return this.prisma.room.findMany({ select: { ... } });
  → Prisma select 안의 필드들이 곧 응답 형태
  → select를 보고 API 타입 작성
```

```typescript
// 서비스에서 이렇게 반환하면
const rooms = await this.prisma.room.findMany({
  where,
  select: {
    id:          true,
    name:        true,
    description: true,
    visibility:  true,
    status:      true,
    memberCount: true,
    passwordHint: true,
    createdAt:   true,
    updatedAt:   true,
    ownerId:     true,
    owner: {
      select: { id: true, nickname: true, email: true },
    },
  },
});

// 프론트 API 타입은 select를 그대로 옮김
export type ApiAdminRoom = {
  id:          string;
  name:        string;
  description: string | null;   // nullable 컬럼
  visibility:  'public' | 'private' | 'invite';
  status:      'active' | 'closed' | 'archived';
  memberCount: number;
  passwordHint: string | null;
  createdAt:   string;          // Date → string
  updatedAt:   string;
  ownerId:     string;
  owner: { id: string; nickname: true; email: string };
};
```

```txt
passwordHash가 select에 없는 이유:
  select에 명시한 필드만 응답에 포함됨
  민감한 필드(passwordHash, refreshToken 등)는 select에서 제외
  → 응답 타입에도 포함 안 됨

select가 응답 타입을 결정:
  select에 있다 → 응답에 있다 → API 타입에 포함
  select에 없다 → 응답에 없다 → API 타입에 포함 안 함
  select 없이 findMany()하면 → scalar 전부 → API 타입도 전부

어떻게 응답 형태를 파악하는가:
  ① @ApiProperty가 붙은 응답 DTO 클래스가 있으면 → 그걸 보고 작성
  ② 없으면 서비스 코드의 Prisma select를 보고 작성
  ③ 자동 생성(openapi-typescript)이 가능하면 → 그게 가장 정확
```

```typescript
// NestJS DTO (백엔드)
export class RoomDto {
  @ApiProperty()
  id: string;

  @ApiProperty()
  name: string;

  @ApiProperty({ nullable: true })
  description: string | null;    // nullable: true → string | null

  @ApiProperty({ enum: ['public', 'private', 'invite'] })
  visibility: 'public' | 'private' | 'invite';

  @ApiProperty()
  memberCount: number;

  @ApiProperty()
  createdAt: Date;   // ← DTO는 Date 타입

  @ApiProperty({ type: () => UserSummaryDto })
  owner: UserSummaryDto;         // 관계 → 중첩 객체
}
```

```typescript
// NextJS API 타입 (프론트엔드) — DTO를 보고 만든 것
export type ApiRoom = {
  id:          string;
  name:        string;
  description: string | null;   // nullable: true → string | null
  visibility:  'public' | 'private' | 'invite';
  memberCount: number;
  createdAt:   string;          // ← Date가 아닌 string! (JSON 전송 특성)
  owner: {
    id:       string;
    nickname: string;
    email:    string;
  };
};
```

```txt
DTO → API 타입 변환 규칙:

  DTO 타입          →  API 타입
  ─────────────────────────────────────────────
  string            →  string
  number            →  number
  boolean           →  boolean
  Date              →  string  ← 가장 중요!
  string | null     →  string | null
  string[]          →  string[]
  enum SomeEnum     →  'value1' | 'value2' | ...  (리터럴 유니온)
  OtherDto          →  { id: string; ... }  (중첩 객체로 펼침)
  OtherDto[]        →  { id: string; ... }[]
  OtherDto | null   →  { ... } | null
```

### Date가 string인 이유 ⭐️⭐️⭐️⭐️

```txt
NestJS DTO에서 createdAt: Date 라고 해도
JSON으로 직렬화(JSON.stringify)되면 string이 됨

  new Date('2024-01-15') → JSON.stringify → "2024-01-15T00:00:00.000Z"

JSON 형식에는 Date 타입이 없음 — string, number, boolean, object, array, null 만 있음
→ Date는 항상 ISO 8601 형식 문자열로 전송됨

따라서 API 타입은 항상 string으로 선언:
  createdAt: string   ← "2024-01-15T00:00:00.000Z" 형태의 문자열
  updatedAt: string

UI 타입에서 Date 객체가 필요하면 매퍼에서 변환:
  createdAt: new Date(api.createdAt)  → [[NextJS_Types]] 매퍼 섹션
```

### nullable vs optional ⭐️⭐️⭐️

```typescript
// nullable: 값이 있지만 null일 수 있음
description: string | null;   // 항상 JSON에 키가 있음, 값이 null일 수 있음

// optional: 키 자체가 없을 수 있음 (드묾)
nickname?: string;             // JSON에 'nickname' 키 자체가 없을 수 있음
```

```txt
API 응답에서는 대부분 nullable(string | null)을 씀
  키는 항상 있고 값이 null인 형태
  → DTO의 @ApiProperty({ nullable: true }) 를 보면 string | null

optional(?)은 다른 의미:
  응답 객체에 그 키가 아예 없을 수 있음
  → DTO의 @IsOptional() 이 있는 필드
```

### 관계(nested object) 타입 ⭐️⭐️⭐️

```typescript
// DTO에서 다른 DTO를 참조하는 경우
export class RoomDto {
  @ApiProperty({ type: () => UserSummaryDto })
  owner: UserSummaryDto;
}

export class UserSummaryDto {
  id:       string;
  nickname: string;
  email:    string;
}
```

```typescript
// 방법 1 — 인라인으로 펼침 (간단한 중첩)
export type ApiRoom = {
  owner: { id: string; nickname: string; email: string };
};

// 방법 2 — 별도 타입으로 분리 (여러 곳에서 재사용)
export type ApiUserSummary = {
  id:       string;
  nickname: string;
  email:    string;
};

export type ApiRoom = {
  owner: ApiUserSummary;  // 재사용
};
```

```txt
언제 인라인, 언제 별도 타입:
  한 곳에서만 씀 → 인라인이 더 간결
  여러 타입에서 같은 구조를 씀 → 별도 타입으로 분리
```

### 페이지네이션 응답 타입 ⭐️⭐️⭐️

```typescript
// 서버가 페이지네이션 응답을 이 형태로 줄 때
export type ApiRoomsPage = {
  items:      ApiRoom[];
  nextCursor: string | null;  // 다음 페이지 없으면 null
};

// 제네릭으로 만들면 재사용 가능
export type ApiPage<T> = {
  items:      T[];
  nextCursor: string | null;
};

export type ApiRoomsPage = ApiPage<ApiRoom>;
export type ApiUsersPage = ApiPage<ApiUser>;
```

### 실전 예시 — ApiAdminRoom 전체 과정

```typescript
// ① NestJS AdminRoomDto를 보고 각 필드의 JSON 타입을 결정
// ② Date → string, enum → 리터럴 유니온, nullable → string | null
// ③ 관계 필드는 중첩 객체로 표현

export type ApiAdminRoom = {
  id:           string;
  name:         string;
  description:  string | null;                           // nullable
  topicTags:    string[];                                // string 배열
  visibility:   'public' | 'private' | 'invite';        // enum → 리터럴 유니온
  status:       'active' | 'closed' | 'archived';
  memberCount:  number;
  passwordHint: string | null;
  createdAt:    string;    // Date → string (JSON 직렬화)
  updatedAt:    string;
  ownerId:      string;    // FK 컬럼 (ID만)
  owner: {                 // 관계 → 중첩 객체
    id:       string;
    nickname: string;
    email:    string;
  };
};

export type ApiAdminRoomsPage = {
  items:      ApiAdminRoom[];
  nextCursor: string | null;
};
```

```txt
ownerId와 owner 둘 다 있는 이유:
  ownerId  → FK 컬럼 값 (단순 string ID)
  owner    → JOIN/include로 가져온 User 객체
  DTO에서 둘 다 @ApiProperty로 노출하면 둘 다 타입에 포함
  UI에서 owner.nickname을 표시하려면 owner 객체가 필요
  owner 없이 ownerId만 있으면 별도 API 호출 필요
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