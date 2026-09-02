---
aliases:
  - Prisma
  - Patterns
  - 정규화 페어키 패턴
tags:
  - NestJS
related:
  - "[[00_NestJS_Ecosystem_HomePage]]"
  - "[[NestJS_Prisma]]"
  - "[[NestJS_Migration]]"
  - "[[NestJS_Transaction]]"
  - "[[NestJS_Pagination]]"
---
# NestJS_Prisma_Patterns — Prisma 실전 패턴

> [!info] 
> Prisma의 "어떻게 쓰는지" 패턴 모음. 
> API 레퍼런스 → [[NestJS_Prisma]], 설치·마이그레이션 → [[NestJS_Migration]]

---

# 관계 필터 — some / every / none ⭐️⭐️⭐️⭐️

```typescript
// "내가 멤버인 방" — members 중 userId가 일치하는 것이 하나라도 있으면
room.findMany({
  where: {
    members: { some: { userId } },
  },
});
// → SELECT * FROM room WHERE EXISTS (
//     SELECT 1 FROM room_member
//     WHERE room_id = room.id AND user_id = ?
//   )
```

|연산자|의미|예시 상황|
|---|---|---|
|`some`|관련 레코드 중 **하나라도** 조건에 맞으면|"내가 멤버인 방", "댓글이 하나라도 있는 게시글"|
|`every`|관련 레코드 **모두** 조건에 맞아야|"모든 멤버가 활성 상태인 방"|
|`none`|조건에 맞는 관련 레코드가 **하나도 없으면**|"신고가 없는 게시글", "내가 숨긴 메시지 제외"|

```typescript
// some — 하나라도
room.findMany({
  where: { members: { some: { userId } } },   // 내가 멤버인 방
});

// some — 여러 조건
post.findMany({
  where: {
    comments: { some: { authorId: userId, isDeleted: false } },
    // 내가 쓴 삭제 안 된 댓글이 하나라도 있는 게시글
  },
});

// every — 전부 다
room.findMany({
  where: { members: { every: { status: 'active' } } },
});

// none — 하나도 없음
post.findMany({
  where: { reports: { none: {} } },    // 신고가 하나도 없는 게시글
});

// none + 조건 — "내가 숨긴 것 제외"
message.findMany({
  where: {
    roomId,
    deletedAt: null,
    hides: { none: { userId } },
  },
});
```

```txt
hides: { none: { userId } } 읽는 법:

  모델 관계: Message → MessageHide (1:N)
  MessageHide.userId = 숨긴 유저 ID

  none: { userId } — "hides 중에서 userId = 나인 레코드가 하나도 없는 메시지"
                   = "나는 이 메시지를 숨기지 않았다"
                   = 내가 숨긴 메시지는 결과에서 제외

  SQL로 표현하면:
    WHERE NOT EXISTS (
      SELECT 1 FROM message_hide
      WHERE message_hide.message_id = message.id
        AND message_hide.user_id = ?
    )

some vs none 비교:
  hides: { some: { userId } }  → 내가 숨긴 메시지만
  hides: { none: { userId } }  → 내가 숨기지 않은 메시지만

패턴 응용:
  likes: { none: { userId } }   → 내가 좋아요 안 한 게시글
  reads: { none: { userId } }   → 내가 읽지 않은 메시지

조건 안에 빈 {}:
  { some: {} }  → 관련 레코드가 하나라도 "존재하면"
  { none: {} }  → 관련 레코드가 하나도 없으면

SQL은 JOIN 후 WHERE로 필터링하는 방식
Prisma는 관계 필터(some/every/none)로 선언적으로 표현
→ Prisma가 내부적으로 EXISTS 서브쿼리나 JOIN으로 변환
```

## is / isNot — 단수 관계 필터 ⭐️⭐️⭐️

```typescript
// some/every/none은 "hasMany" 관계(배열 필드)
// is/isNot은 "belongsTo" 관계(단수 필드)

comment.findMany({
  where: {
    post: { is: { isPublished: true } },
    // 댓글이 속한 게시글이 공개 상태인 것만
  },
});

comment.findMany({
  where: {
    post: { isNot: { isDeleted: true } },
    // 댓글이 속한 게시글이 삭제되지 않은 것만
  },
});
```

```txt
some/every/none   →  "hasMany" 관계 (배열 필드)  예: post.comments
is/isNot          →  "belongsTo" 관계 (단수 필드) 예: comment.post
```

---

# 원자적 업데이트 (Atomic Update) ⭐️⭐️⭐️⭐️

```typescript
// 숫자 필드를 읽지 않고 DB에서 직접 연산
await this.prisma.post.update({
  where: { id: postId },
  data:  { viewCount: { increment: 1 } },
});
// → SQL: UPDATE post SET view_count = view_count + 1 WHERE id = ?
```

```txt
왜 { increment: 1 }을 쓰는가 — 읽고-쓰기(read-modify-write) 경쟁 조건 방지:

  ❌ 일반 방식 (경쟁 조건 위험):
    const post = await prisma.post.findUnique({ where: { id } });
    await prisma.post.update({ data: { viewCount: post.viewCount + 1 } });
    // 두 요청이 동시에 읽으면 둘 다 같은 값을 +1 → 한 번만 증가

  ✅ 원자적 업데이트:
    data: { viewCount: { increment: 1 } }
    // DB가 단일 SQL로 처리 → 경쟁 조건 없음
```

## 연산자

|연산자|의미|예시|
|---|---|---|
|`{ increment: n }`|`+ n`|`{ viewCount: { increment: 1 } }`|
|`{ decrement: n }`|`- n`|`{ memberCount: { decrement: 1 } }`|
|`{ multiply: n }`|`* n`|`{ price: { multiply: 1.1 } }`|
|`{ divide: n }`|`/ n`|`{ ratio: { divide: 2 } }`|
|`{ set: n }`|`= n` (명시적 덮어쓰기)|`{ retryCount: { set: 0 } }`|

```typescript
// 여러 필드 동시에
await this.prisma.stats.update({
  where: { id },
  data: {
    winCount:   { increment: 1 },
    totalGames: { increment: 1 },
    winRate:    { set: newRate },   // 계산된 값을 set으로
  },
});
```

## $transaction과 조합 — 두 테이블 동시 원자 업데이트 ⭐️⭐️⭐️

```typescript
// 멤버 추가 + 카운터 증가를 하나의 트랜잭션으로
const [member] = await this.prisma.$transaction([
  this.prisma.roomMember.create({
    data: { roomId, userId, role: 'member' },
  }),
  this.prisma.room.update({
    where: { id: roomId },
    data:  { memberCount: { increment: 1 } },
  }),
]);
```

```txt
$transaction 배치 방식 + 원자적 업데이트 조합:
  두 작업이 전부 성공하거나 전부 실패 (트랜잭션 보장)
  memberCount는 DB 레벨에서 atomic하게 증가
  const [member] = 구조분해 → 배치 트랜잭션 결과는 배열 순서대로

배치 트랜잭션 상세 → [[NestJS_Transaction]]
```

---

# 토글 패턴 — 있으면 삭제, 없으면 생성 ⭐️⭐️⭐️⭐️

```typescript
// 이모지 반응 토글 — 같은 이모지면 취소, 다르면 교체
async toggleReaction(resourceId: string, userId: string, emoji: string) {
  const existing = await this.prisma.reaction.findUnique({
    where: { resourceId_userId: { resourceId, userId } },
  });

  // 같은 이모지 → 삭제 (토글 off)
  if (existing?.emoji === emoji) {
    await this.prisma.reaction.delete({
      where: { resourceId_userId: { resourceId, userId } },
    });
    return { resourceId, userId, emoji, removed: true as const };
  }

  // 없거나 다른 이모지 → upsert (생성 or 교체)
  const reaction = await this.prisma.reaction.upsert({
    where:  { resourceId_userId: { resourceId, userId } },
    create: { resourceId, userId, emoji },
    update: { emoji },
  });
  return { ...reaction, removed: false as const };
}
```

```txt
existing?.emoji === emoji 분기:
  null/undefined (없음)  → 새로 생성
  같은 이모지            → 삭제 (토글 off)
  다른 이모지            → 교체 (upsert)

removed 필드의 역할 — 작업 구분자:
  서비스가 삭제를 했는지, 생성을 했는지 호출한 쪽이 알아야 함
  removed: true  → 삭제됐음 → 클라이언트에서 이모지 제거
  removed: false → 추가됐음 → 클라이언트에서 이모지 추가

  // 컨트롤러/소켓에서 분기
  const result = await reactionService.toggle(resourceId, userId, emoji);
  if (result.removed) {
    this.gateway.emitReactionRemoved(result);
  } else {
    this.gateway.emitReactionAdded(result);
  }

removed: true as const / false as const:
  boolean이 아닌 리터럴 타입 true / false 로 반환
  호출하는 쪽에서 if (result.removed) 분기 시 TypeScript가 정확히 좁힐 수 있음
  → [[TS_Type_Guards]] as const 참고

upsert (없으면 create, 있으면 update):
  where  → 고유 조건 (이미 있는지 확인)
  create → 없을 때 만들 데이터
  update → 있을 때 바꿀 데이터
```

----
# 정규화 페어키 패턴 — 두 사용자 간 유일한 DM ⭐️⭐️⭐️⭐️

```txt
문제: A → B 대화와 B → A 대화가 같은 방이어야 함
      단순히 (senderId, receiverId) 복합키로 @@unique를 걸면
      A→B와 B→A가 다른 조합으로 인식되어 대화방이 두 개 생김

해결: 두 UUID를 항상 같은 순서로 정렬해서 하나의 키를 만들면
      어느 방향으로 만들어도 항상 같은 키 → @@unique로 중복 방지
```

## 유틸 함수

```typescript
// utils/dm.ts
export function dmPairKey(a: string, b: string): string {
  return a < b ? `${a}:${b}` : `${b}:${a}`;
}
// a < b → "a:b"
// b < a → 그래도 "a:b"  ← 항상 같은 결과
```


```txt
문자열 비교(a < b)로 UUID를 정렬:
  UUID는 하이픈 포함 36자 문자열 → < 비교로 사전순 정렬 가능
  어느 쪽에서 호출해도 (A, B)든 (B, A)든 항상 작은 UUID가 앞에 옴
  → 같은 두 사람 사이의 키는 항상 동일하게 고정됨
```

## Schema

```prisma
model DirectMessageRoom {
  id      String @id @default(uuid()) @db.Uuid
  pairKey String @unique              // "uuid1:uuid2" 형태, 작은 값이 항상 앞
  //
  messages DirectMessage[]
  createdAt DateTime @default(now()) @db.Timestamptz(3)
}
```

```txt
@unique를 pairKey 하나에 걸면 충분 — 복합 @@unique([a, b]) 쓰면 순서 문제가 그대로 남음
pairKey는 단순 String이지만 DB가 중복 생성을 원천 차단
```

## 서비스 — findOrCreate 패턴

```typescript
async findOrCreateDmRoom(userAId: string, userBId: string) {
  const pairKey = dmPairKey(userAId, userBId);

  return this.prisma.directMessageRoom.upsert({
    where:  { pairKey },
    create: { pairKey },
    update: {},          // 이미 있으면 그대로 — 덮어쓸 것 없음
  });
}
```


```txt
upsert를 쓰는 이유:
  findUnique → null이면 create 패턴은 두 요청이 동시에 null을 받으면
  둘 다 create를 시도 → P2002(unique 위반) 발생 가능

  upsert는 DB 레벨에서 "없으면 create, 있으면 update"를 원자적으로 처리
  → 동시 요청에서도 안전

  update: {} 는 "있으면 아무것도 바꾸지 않음"
  → pairKey가 이미 있으면 그 행을 그대로 반환
```

## 조회 — 기존 DM 방 찾기

```typescript
async getDmRoom(userAId: string, userBId: string) {
  const pairKey = dmPairKey(userAId, userBId);

  return this.prisma.directMessageRoom.findUnique({
    where: { pairKey },
  });
}
```


```txt
pairKey가 @unique이므로 findUnique 사용 가능
어느 방향(A→B, B→A)으로 호출해도 같은 방을 찾음
```

---
# 조건부 where 조립 패턴 ⭐️⭐️⭐️⭐️

```txt
여러 조건이 선택적으로 붙는 쿼리에서
각 조건을 별도 변수에 담고 마지막에 spread로 합치는 패턴
```

```typescript
// 각 조건을 담을 변수 — 처음엔 비어있음
let feedWhere:   object = {};
let cursorWhere: object = {};

// 조건에 따라 채움
if (feed === 'friends') {
  feedWhere = { authorId: { in: friendIds } };
}
if (query.cursor) {
  cursorWhere = { OR: [{ createdAt: { lt: cursorDate } }, ...] };
}

// 마지막에 한 번에 합침
const rows = await this.prisma.post.findMany({
  where: {
    hidden: false,
    ...feedWhere,    // 비어있으면 spread해도 아무 영향 없음
    ...cursorWhere,  // 있으면 해당 조건이 추가됨
  },
});
```

```txt
핵심: 빈 객체를 spread하면 아무것도 추가되지 않음

  { hidden: false, ...{} }
  = { hidden: false }  ← 변화 없음

  { hidden: false, ...{ authorId: { in: [...] } } }
  = { hidden: false, authorId: { in: [...] } }  ← 조건 추가됨

왜 이렇게 하는가:
  조건마다 if문 안에서 findMany를 따로 호출하면 코드가 중복됨
  조건들을 각자 처리하고 마지막 쿼리 하나에서 합치면 깔끔

let x: object = {} 타입:
  object는 "어떤 객체든 가능"한 넓은 타입
  나중에 Prisma.PostWhereInput 처럼 구체적인 타입으로 교체하면 더 안전
  단계적으로 조건을 조립할 때 object가 편의상 많이 쓰임
```


```typescript
// 더 타입 안전한 방법 — Prisma 타입 사용
let feedWhere: Prisma.PostWhereInput = {};

if (feed === 'friends') {
  feedWhere = { authorId: { in: friendIds } };
}
```
---

# 날짜 범위 쿼리 — 캘린더 패턴 ⭐️⭐️⭐️

## KST 기준 월 범위 조회

```typescript
async getCalendar(userId: string, year: number, month: number) {
  // 1. 월 범위 계산 (KST 기준)
  const monthKey = `${year}-${String(month).padStart(2, '0')}-01`;
  const nextMonth = new Date(Date.UTC(year, month, 1)); // month는 0-based 아님 주의
  const nextMonthKey = `${nextMonth.getUTCFullYear()}-${String(
    nextMonth.getUTCMonth() + 1,
  ).padStart(2, '0')}-01`;

  const start = kstDayRange(monthKey).start;  // KST 00:00 → UTC instant
  const end   = kstDayRange(nextMonthKey).start;

  // 2. Prisma 날짜 범위 필터 — gte/lt 패턴
  const rows = await this.prisma.userMovie.findMany({
    where: {
      userId,
      kind: 'watched',
      watchedAt: { gte: start, lt: end },   // start 포함, end 미포함
    },
    orderBy: { watchedAt: 'asc' },
    select: { tmdbId: true, watchedAt: true },  // 필요한 컬럼만 선택
  });

  // 3. Promise.all — 각 행에 대해 병렬 async 처리
  const items = await Promise.all(
    rows.map(async (row) => ({
      tmdbId:    row.tmdbId,
      date:      kstDateKey(row.watchedAt!),   // null assertion — select로 보장됨
      watchedAt: row.watchedAt!.toISOString(),
      movie:     await this.tmdbService.getMovieCached(row.tmdbId),
    })),
  );

  return { year, month, items };
}
```

```txt
날짜 범위 필터 — gte / lt:
  gte: start  → start 이상 (start 포함)
  lt:  end    → end 미만  (end 미포함)
  → "7월 1일 KST 00:00 ~ 8월 1일 KST 00:00 미만" = 7월 전체

  lte (end 포함)가 아닌 lt (end 미포함):
    다음 달 1일 00:00 KST = 전날 마지막 순간 바로 다음
    → lt로 경계를 딱 잘라냄 (lte 쓰면 8월 1일 00:00 데이터도 포함돼버림)

select — 필요한 컬럼만:
  select: { tmdbId: true, watchedAt: true }
  → id, userId, kind 등 불필요한 컬럼을 DB에서 전송하지 않음
  → 쿼리 결과 크기 ↓, 직렬화 비용 ↓
  → 단, select 사용 시 include와 함께 쓸 수 없음 (Prisma 제약)

row.watchedAt!  — null assertion 이유:
  Prisma 스키마에서 watchedAt이 DateTime? (optional)로 정의된 경우
  where 절에 watchedAt: { gte: start } 조건이 있어도
  Prisma의 타입 추론은 여전히 Date | null로 반환됨
  → 실제로 null이 올 수 없음을 개발자가 알고 있으므로 ! 로 단언
```

## Promise.all vs for...of await

```txt
rows.map(async (row) => ...) + Promise.all:
  rows 전체를 동시에 실행 → 병렬 처리
  외부 API(TMDB) 호출이 있는 경우 전체 시간 = 가장 느린 한 건의 시간
  → 10건이라면 10개 API 요청이 동시에 나감

for...of + await:
  한 건씩 순서대로 실행 → 직렬 처리
  전체 시간 = 모든 건의 합산
  → 10건이라면 10개 API 요청이 순서대로 나감

선택 기준:
  외부 I/O(API, DB)가 독립적이면 → Promise.all (빠름)
  앞 결과가 다음 입력에 필요하면 → for...of + await (순서 보장)
  건수가 많고 API rate limit 있으면 → p-limit으로 동시 실행 수 제한
```

```typescript
// p-limit — 동시 실행 수 제한 예시
import pLimit from 'p-limit';
const limit = pLimit(5); // 최대 5개 동시

const items = await Promise.all(
  rows.map((row) =>
    limit(async () => ({
      tmdbId: row.tmdbId,
      movie:  await this.tmdbService.getMovieCached(row.tmdbId),
    }))
  )
);
```

```txt
kstDayRange(key):
  'YYYY-MM-DD' 형태의 KST 날짜 문자열을 받아
  { start: Date, end: Date } UTC 기준 범위로 변환하는 유틸
  → KST 기준 하루의 시작/끝을 UTC로 정확하게 변환
  ([[JS_Date]] 월 범위 계산 섹션 참고)

kstDateKey(date: Date):
  UTC Date 객체를 받아 KST 기준 'YYYY-MM-DD' 문자열 반환
  → DB에 UTC로 저장된 값을 클라이언트에 KST 날짜로 표시할 때 사용
```

---

# $transaction — 트랜잭션

```txt
배치 형태 / 인터랙티브 형태 / include 경고 / 선택 기준
→ [[NestJS_Transaction]]
```