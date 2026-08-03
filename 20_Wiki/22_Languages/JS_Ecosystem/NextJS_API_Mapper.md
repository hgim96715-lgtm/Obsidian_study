---
aliases:
  - API 타입
  - apiTypes
  - mapper
  - Mapper 함수
tags:
  - NextJS
related:
  - "[[00_JS_Ecosystem_HomePage]]"
  - "[[NestJS_Swagger]]"
  - "[[NextJS_API_Client]]"
  - "[[NextJS_Types]]"
---
# NextJS_API_Mapper — 기능별 API 함수

> [!info] 
> 매퍼 함수 = apiFetch를 감싸서 특정 엔드포인트를 호출하고 ApiXxx → UiXxx 변환까지 하는 함수. 
> 컴포넌트는 apiFetch를 직접 모르고 fetchUser() 같은 의미 있는 함수만 호출한다. 
> apiFetch 래퍼 → [[NextJS_API_Client]], 타입 설계 → [[NextJS_Types]]

---

# 왜 매퍼 함수가 필요한가 ⭐️⭐️⭐️⭐️

```typescript
// ❌ 컴포넌트가 apiFetch를 직접 호출
function UserProfile({ userId }: { userId: string }) {
  useEffect(() => {
    apiGet<ApiUser>(`/users/${userId}`)
      .then(api => ({              // 변환 로직이 컴포넌트 안에
        ...api,
        image: api.image ?? '/default.png',
      }))
      .then(setUser);
  }, [userId]);
}
```

```typescript
// ✅ 매퍼 함수로 분리
// api/users.ts
export async function fetchUser(userId: string): Promise<UiUser> {
  const api = await apiGet<ApiUser>(`/users/${userId}`);
  return toUiUser(api);   // 변환 로직이 한 곳에
}

// 컴포넌트 — apiFetch, ApiUser, 변환 로직을 전혀 모름
function UserProfile({ userId }: { userId: string }) {
  useEffect(() => {
    fetchUser(userId).then(setUser);
  }, [userId]);
}
```

```txt
매퍼 함수가 해주는 것:
  ① 엔드포인트 URL 관리 (한 곳에서)
  ② ApiXxx → UiXxx 변환
  ③ 컴포넌트를 API 세부사항으로부터 분리

URL이 /users/${id} → /v2/users/${id} 로 바뀌면:
  매퍼 함수 하나만 수정
  컴포넌트는 수정 불필요
```

---

# 파일 구조 ⭐️⭐️⭐️

```txt
src/api/
├── users.api.ts      사용자 관련 함수
├── posts.api.ts      게시글 관련 함수
├── rooms.api.ts      채팅방 관련 함수
└── auth.api.ts       인증 관련 함수
```

```txt
파일당 하나의 리소스 → 기능 추가될수록 각 파일에 함수만 추가
컴포넌트에서 import: import { fetchUser } from '@/api/users.api'
```

---

# GET — 조회 ⭐️⭐️⭐️⭐️

```typescript
// api/users.api.ts

// 단건 조회
export async function fetchUser(userId: string): Promise<UiUser> {
  const api = await apiGet<ApiUser>(`/users/${userId}`);
  return toUiUser(api);
}

// 내 정보
export async function fetchMe(): Promise<UiUser> {
  const api = await apiGet<ApiUser>('/users/me');
  return toUiUser(api);
}

// 목록 + 페이지네이션
export async function fetchUsers(
  params: { cursor?: string; take?: number; q?: string } = {},
): Promise<CursorPage<UiUser>> {
  const sp = new URLSearchParams();
  if (params.cursor) sp.set('cursor', params.cursor);
  if (params.take)   sp.set('take',   String(params.take));
  if (params.q)      sp.set('q',      params.q);

  const qs  = sp.toString();
  const api = await apiGet<CursorPage<ApiUser>>(`/users${qs ? `?${qs}` : ''}`);
  return {
    items:      api.items.map(toUiUser),
    nextCursor: api.nextCursor,
  };
}
```

---

# POST — 생성 ⭐️⭐️⭐️⭐️

```typescript
// api/posts.api.ts

type CreatePostInput = {
  title:   string;
  content: string;
};

// 방법 1 — apiPost 헬퍼 사용 (내부에서 method·headers·body 처리)
export async function createPost(input: CreatePostInput): Promise<UiPost> {
  const api = await apiPost<ApiPost>('/posts', input);
  return toUiPost(api);
}

// 방법 2 — apiFetch에 직접 옵션 넣기 (헬퍼 없을 때, 더 명시적으로 쓸 때)
export function createFriendRequest(userId: string): Promise<ApiFriendship> {
  return apiFetch<ApiFriendship>('/friends/requests', {
    method:  'POST',
    headers: { 'Content-Type': 'application/json' },
    body:    JSON.stringify({ userId }),
  });
}
```

```txt
method · headers · body 세트:
  HTTP에서 "데이터를 보내는" 요청(POST · PATCH · PUT)은 이 세 가지가 한 묶음

  method: 'POST'
    서버에 "새로 만들어줘"라는 의도 전달

  headers: { 'Content-Type': 'application/json' }
    "body가 JSON 형식이야"라고 서버에 알림
    이 헤더 없으면 서버가 body를 어떤 형식으로 파싱할지 모름

  body: JSON.stringify({ userId })
    실제로 보낼 데이터 — 객체를 JSON 문자열로 변환
    fetch의 body는 문자열이나 FormData여야 함 → JSON.stringify 필수

apiPost 헬퍼가 하는 것:
  apiFetch(path, { method: 'POST', body: JSON.stringify(body) })
  + Content-Type: application/json 헤더 자동 추가
  → 직접 쓰는 것과 동일, 반복을 줄여주는 것

둘 중 선택 기준:
  헬퍼(apiPost)   → 대부분의 경우, 간결함
  직접 options   → 헬퍼 없는 환경, 헤더를 커스텀해야 할 때
```

# PATCH — 수정 ⭐️⭐️⭐️⭐️

```typescript
// api/users.api.ts

type UpdateMeInput = Partial<{
  nickname: string;
  image:    string;
}>;

// 방법 1 — apiPatch 헬퍼
export async function updateMe(input: UpdateMeInput): Promise<UiUser> {
  const api = await apiPatch<ApiUser>('/users/me', input);
  return toUiUser(api);
}

// 방법 2 — 직접 옵션
export function respondFriendRequest(
  friendshipId: string,
  action:       'accept' | 'decline',
): Promise<ApiFriendship> {
  return apiFetch<ApiFriendship>(`/friends/requests/${friendshipId}`, {
    method:  'PATCH',
    headers: { 'Content-Type': 'application/json' },
    body:    JSON.stringify({ action }),
  });
}
```

```txt
POST vs PATCH:
  POST  → 새 리소스 생성   /friends/requests
  PATCH → 기존 리소스 수정  /friends/requests/:id

body 구조:
  POST  → 생성에 필요한 전체 데이터  { userId }
  PATCH → 바꿀 필드만              { action } 또는 { nickname } 등
  NestJS에서 PATCH DTO는 보통 Partial<CreateDto>
```

# DELETE ⭐️⭐️⭐️

```typescript
// 응답이 없는 경우 (204 No Content)
export async function deletePost(postId: string): Promise<void> {
  await apiDelete(`/posts/${postId}`);
}

// 직접 옵션
export function cancelFriendRequest(friendshipId: string): Promise<void> {
  return apiFetch<void>(`/friends/requests/${friendshipId}`, {
    method: 'DELETE',
    // DELETE는 body 없음 → headers도 불필요
  });
}
```

---

# 변환 함수 (toUiXxx) ⭐️⭐️⭐️⭐️

```typescript
// api/users.api.ts — 매퍼 함수와 같은 파일에

function toUiUser(api: ApiUser): UiUser {
  return {
    ...api,
    image: api.image ?? '/default-avatar.png',  // null → 기본값
  };
}

// 내보낼 필요 없으면 private 함수로 — 파일 내에서만 사용
// 여러 파일에서 쓴다면 types/mappers.ts 로 분리
```

```txt
toUiXxx를 같은 파일에 두는 이유:
  fetchUser()와 toUiUser()는 항상 같이 바뀜
  ApiUser가 바뀌면 fetchUser와 toUiUser 둘 다 수정
  같은 파일에 있으면 수정 위치를 찾기 쉬움

변환이 복잡하거나 여러 파일에서 공유한다면:
  types/ui-mappers.ts 로 분리해서 export
```

---

# 여러 API 응답을 합치는 경우 ⭐️⭐️⭐️

```typescript
// api/rooms.api.ts

export async function fetchRoomDetail(roomId: string): Promise<UiRoomDetail> {
  // 병렬 요청 — 서로 의존하지 않으면 동시에 보냄
  const [room, members, messages] = await Promise.all([
    apiGet<ApiRoom>(`/rooms/${roomId}`),
    apiGet<ApiMember[]>(`/rooms/${roomId}/members`),
    apiGet<ApiMessage[]>(`/rooms/${roomId}/messages?take=1`),
  ]);

  return {
    ...room,
    members:     members.map(toUiMember),
    lastMessage: messages[0] ? toUiMessage(messages[0]) : null,
  };
}
```

```txt
Promise.all:
  3개의 요청을 동시에 보냄
  가장 느린 요청이 완료될 때까지 기다림 (직렬보다 빠름)

언제 직렬로 써야 하는가:
  B 요청이 A 응답의 데이터를 필요로 할 때
  const room = await fetchRoom(roomId);
  const owner = await fetchUser(room.ownerId);  // room 먼저 필요
```

---

# 토글 (upsert) 패턴 ⭐️⭐️⭐️

```typescript
// api/reactions.api.ts

type ToggleResult = {
  messageId: string;
  emoji:     string;
  userId:    string;
  removed:   boolean;
};

export async function toggleReaction(
  roomId:    string,
  messageId: string,
  emoji:     string,
): Promise<ToggleResult> {
  return apiPost<ToggleResult>(`/rooms/${roomId}/messages/${messageId}/reactions`, {
    emoji,
  });
}
```

---

# 컴포넌트에서 사용 패턴 ⭐️⭐️⭐️

```typescript
// 1. useEffect에서 로드 (자동 실행)
useEffect(() => {
  let cancelled = false;
  fetchUser(userId).then(user => {
    if (!cancelled) setUser(user);
  });
  return () => { cancelled = true; };
}, [userId]);

// 2. 이벤트 핸들러에서 (버튼 클릭)
const handleSave = async () => {
  setSubmitting(true);
  try {
    const updated = await updateMe({ nickname });
    setUser(updated);
  } catch (err) {
    setError(err instanceof ApiRequestError ? err.message : '저장에 실패했습니다.');
  } finally {
    setSubmitting(false);
  }
};

// 3. fire-and-forget (결과 불필요)
void markAsRead(roomId);  // 읽음 처리 — 실패해도 무관
```

---

# 전체 흐름 요약

```txt
컴포넌트
  fetchUser(userId)             ← 의미 있는 이름의 매퍼 함수
      ↓
  apiGet<ApiUser>('/users/...')  ← apiFetch 래퍼
      ↓
  fetch(baseUrl + path, {        ← 실제 HTTP 요청
    headers: { Authorization }
  })
      ↓
  ApiUser (서버 응답)
      ↓
  toUiUser(api)                  ← 타입 변환
      ↓
  UiUser (컴포넌트 상태에 저장)
```