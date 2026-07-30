---
aliases:
  - localStorage
  - sessionStorage
tags:
  - JavaScript
related:
  - "[[00_JS_Ecosystem_HomePage]]"
  - "[[JS_BrowserAPI]]"
  - "[[JS_JSON]]"
  - "[[TS_Type_Guards]]"
  - "[[JS_Array_Methods]]"
  - "[[React_useSyncExternalStore]]"
  - "[[NextJS_WebSocket]]"
---
# JS_WebStorage — localStorage & sessionStorage

> [!info] 
> localStorage = 브라우저에 **영구** 저장 (탭/창 닫아도 유지).
>  sessionStorage = 탭이 살아있는 동안만 유지. 
>  둘 다 **문자열만** 저장 가능 — 객체/배열은 JSON.stringify 필요.

---

# 기본 API ⭐️⭐️⭐️

```typescript
// 저장
localStorage.setItem('key', 'value');

// 읽기 — 없으면 null
const value = localStorage.getItem('key');

// 삭제
localStorage.removeItem('key');

// 전체 삭제
localStorage.clear();

// 키 개수
localStorage.length;
```

---

# SSR 가드 — typeof window ⭐️⭐️⭐️⭐️

```typescript
// Next.js 등 SSR 환경에서 localStorage는 서버에 없음 → 에러 방지
if (typeof window === 'undefined') return null;

// 또는 함수 초반에
function getItem(key: string): string | null {
  if (typeof window === 'undefined') return null;
  return localStorage.getItem(key);
}
```

```txt
왜 필요한가:
  Next.js는 서버(Node.js)에서도 컴포넌트를 실행
  서버에는 window, localStorage가 없음 → ReferenceError
  → typeof window === 'undefined' 체크로 서버 실행 시 안전하게 건너뜀
```

---

# 키 설계 — prefix 패턴 ⭐️⭐️⭐️⭐️

```typescript
// 1. 단순 키
const KEY = 'theme';

// 2. prefix로 네임스페이스 구분
const PREFIX = 'app:';
const THEME_KEY = `${PREFIX}theme`;          // 'app:theme'
const LANG_KEY  = `${PREFIX}lang`;           // 'app:lang'

// 3. 유저별 키 — 같은 브라우저에서 계정 전환 시 분리
function userKey(prefix: string, userId: string) {
  return `${prefix}${userId}`;
}
userKey('chat-font:', 'user123')             // 'chat-font:user123'
userKey('support-notice-seen:', 'user123')   // 'support-notice-seen:user123'

// 4. 유저 + 리소스 조합
function resourceKey(prefix: string, userId: string, resourceId: string) {
  return `${prefix}${userId}:${resourceId}`;
}
resourceKey('room-notice-seen:', 'user1', 'room99')
// 'room-notice-seen:user1:room99'
```

```txt
prefix를 쓰는 이유:
  여러 기능의 키가 localStorage에 섞임 → prefix로 용도 구분
  cleanup 시 특정 prefix만 골라서 삭제 가능

유저별 키가 필요한 경우:
  같은 브라우저에서 계정 A → 계정 B로 전환
  userId 없이 저장하면 계정 B가 계정 A의 설정/읽음 상태를 공유
  → key에 userId 포함으로 계정별 독립 유지
```

---

# 문자열만 저장 가능 — JSON 직렬화 ⭐️⭐️⭐️⭐️

```typescript
// ❌ 객체를 그대로 저장하면 '[object Object]' 문자열이 저장됨
localStorage.setItem('user', { id: 1, name: 'Tom' });   // '[object Object]'

// ✅ JSON.stringify로 직렬화
localStorage.setItem('user', JSON.stringify({ id: 1, name: 'Tom' }));

// 읽을 때 JSON.parse로 복원
const raw  = localStorage.getItem('user');
const user = raw ? JSON.parse(raw) : null;
```

## 안전하게 읽기 — unknown + 타입 검증 ⭐️⭐️⭐️⭐️

```typescript
export type FontPrefs = { fontId: string; scale: string };

function getFontPrefs(userId: string): FontPrefs | null {
  if (typeof window === 'undefined') return null;
  const raw = localStorage.getItem(`chat-font:${userId}`);
  if (!raw) return null;

  try {
    const parsed = JSON.parse(raw) as Partial<FontPrefs>;
    // 필드별 검증 후 반환
    if (typeof parsed.fontId !== 'string') return null;
    if (typeof parsed.scale  !== 'string') return null;
    return { fontId: parsed.fontId, scale: parsed.scale };
  } catch {
    return null;   // JSON 파싱 실패 → null로 안전하게
  }
}
```

```txt
Partial<T>로 캐스팅하는 이유:
  JSON.parse 결과는 any → Partial<T>로 캐스팅
  Partial = 모든 필드가 있을 수도 없을 수도 있음을 명시
  → 필드별로 검증 후 사용

try-catch가 필요한 이유:
  저장값이 깨진 JSON이면 JSON.parse가 SyntaxError를 던짐
  → catch에서 null 반환으로 안전하게 처리
```

---

# 구조 진화 — 하위 호환 읽기 ⭐️⭐️⭐️⭐️

```typescript
// 예전: 단순 문자열 'lp-bar' 저장
// 지금: JSON 객체 { presetId, backgroundUrl } 저장

function readPrefs(roomId: string): RoomThemePrefs | null {
  const raw = localStorage.getItem(`room-theme:${roomId}`);
  if (!raw) return null;

  // ① 예전 포맷 — 단순 문자열
  if (isPresetId(raw)) {
    return { presetId: raw, backgroundUrl: null };
  }

  // ② 새 포맷 — JSON
  try {
    const parsed = JSON.parse(raw) as Partial<RoomThemePrefs>;
    if (!isPresetId(parsed.presetId ?? '')) return null;
    return {
      presetId:      parsed.presetId!,
      backgroundUrl: parsed.backgroundUrl ?? null,
    };
  } catch {
    return null;
  }
}
```

```txt
단순 문자열 → JSON 객체로 구조가 바뀔 때:
  기존 사용자 데이터를 잃지 않으려면 구 포맷도 처리해야 함
  isPresetId(raw)로 구 포맷 여부 먼저 확인
  → 새 저장은 항상 JSON으로 → 다음 방문에 구 포맷이 자동으로 덮어써짐
```

---

# 읽음 처리 — 내용 기준 ⭐️⭐️⭐️⭐️

```typescript
// "마지막으로 본 내용"과 현재 내용을 비교
const SEEN_KEY = 'notice-seen:';

export function getSeenContent(userId: string, resourceId: string): string | null {
  if (typeof window === 'undefined') return null;
  return localStorage.getItem(`${SEEN_KEY}${userId}:${resourceId}`);
}

export function markSeen(userId: string, resourceId: string, content: string | null) {
  if (typeof window === 'undefined') return;
  localStorage.setItem(
    `${SEEN_KEY}${userId}:${resourceId}`,
    content?.trim() ?? '',
  );
}

export function hasUnread(
  userId: string,
  resourceId: string,
  currentContent: string | null,
): boolean {
  const body = currentContent?.trim() ?? '';
  if (!body) return false;
  return getSeenContent(userId, resourceId) !== body;
}
```

```txt
동작 원리:
  처음 방문  → getSeenContent = null → null !== '공지내용' → true (unread)
  읽음 처리  → markSeen() → localStorage에 현재 내용 저장
  다음 방문  → '공지내용' === '공지내용' → false (read)
  내용 변경  → '공지내용' !== '새공지내용' → true (unread)

언제 쓰는가:
  내용이 편집될 수 있는 단일 공지 (방 공지, 서비스 공지)
  "이 내용을 봤는가"를 내용 자체로 판단
```

---

# 읽음 처리 — publishedAt 타임스탬프 기준 ⭐️⭐️⭐️⭐️

```typescript
// "마지막으로 본 항목의 날짜"와 목록 항목들을 비교
const SEEN_AT_KEY = 'support-notice-seen:';

export function getSeenAt(userId: string): string | null {
  if (typeof window === 'undefined') return null;
  return localStorage.getItem(`${SEEN_AT_KEY}${userId}`);
}

export function markSeenAt(userId: string, publishedAt: string) {
  if (typeof window === 'undefined') return;
  const at = publishedAt.trim();
  if (!at) return;

  const prev   = getSeenAt(userId);
  const prevMs = prev ? new Date(prev).getTime() : 0;
  const nextMs = new Date(at).getTime();

  if (prevMs >= nextMs) return;   // 더 오래된 날짜로 퇴행 방지
  localStorage.setItem(`${SEEN_AT_KEY}${userId}`, at);
}

export function hasUnseenByDate(
  userId: string,
  notices: { publishedAt: string | null }[],
): boolean {
  if (typeof window === 'undefined') return false;
  const seen   = getSeenAt(userId);
  const seenMs = seen ? new Date(seen).getTime() : 0;
  // seen이 없으면 seenMs = 0 → 모든 항목이 unseen

  return notices.some((n) => {
    const at = n.publishedAt?.trim();
    if (!at) return false;
    return new Date(at).getTime() > seenMs;
  });
}
```

```txt
동작 원리:
  처음 방문  → seenMs = 0 → 모든 항목 > 0 → true (전부 unseen)
  읽음 처리  → markSeenAt(최신 publishedAt) 저장
  다음 방문  → 새 항목 없으면 false / 더 최신 항목 있으면 true

prevMs >= nextMs 이면 갱신 안 하는 이유:
  최신 목록(2025-06) → markSeenAt 후 오래된 상세(2025-01) 진입
  → 오래된 날짜로 퇴행하면 이미 본 NEW가 다시 표시됨

내용 기준 vs 타임스탬프 기준:
  내용 기준     단일 공지, 내용이 편집될 때 감지
  날짜 기준     목록 중 새 항목 추가 감지, 공지 개수와 무관
```

---

# Set 직렬화 ⭐️⭐️

```typescript
// Set은 JSON 직렬화 안 됨 → 배열로 변환
const ids = new Set(['a', 'b', 'c']);
localStorage.setItem('ids', JSON.stringify([...ids]));

// 복원
const raw = localStorage.getItem('ids');
const set = new Set<string>(raw ? JSON.parse(raw) : []);
```

---

# 한눈에

```txt
기본:
  setItem(key, value)   문자열 저장
  getItem(key)          읽기 (없으면 null)
  removeItem(key)       삭제

키 설계:
  'prefix:userId'       유저별 분리
  'prefix:userId:id'    유저 + 리소스 분리

문자열만 저장:
  객체/배열 → JSON.stringify / JSON.parse
  try-catch 필수 (파싱 실패 대비)
  Partial<T>로 캐스팅 후 필드별 검증

SSR 가드:
  typeof window === 'undefined' → return null

읽음 처리 (내용 기준):
  markSeen(userId, id, content) → 현재 내용 저장
  hasUnread() → stored !== current

읽음 처리 (날짜 기준):
  markSeenAt(userId, publishedAt) → 최신 날짜만 갱신 (퇴행 방지)
  hasUnseenByDate() → notices.some(n => n.publishedAt > seenAt)
```