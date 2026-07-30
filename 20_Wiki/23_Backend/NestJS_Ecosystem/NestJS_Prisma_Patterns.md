---
aliases:
  - Prisma
  - Patterns
tags:
  - NestJS
related:
  - "[[00_NestJS_Ecosystem_HomePage]]"
  - "[[NestJS_Prisma]]"
  - "[[NestJS_Migration]]"
  - "[[NestJS_Transaction]]"
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

---

# $transaction — 트랜잭션

```txt
배치 형태 / 인터랙티브 형태 / include 경고 / 선택 기준
→ [[NestJS_Transaction]]
```