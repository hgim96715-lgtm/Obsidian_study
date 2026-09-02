---
aliases:
  - Pagination
  - Offset
  - Cursor
  - take
  - skip
  - page
  - limit
  - hasMore
  - 페이지네이션
tags:
  - NestJS
relations:
  - "[[00_NestJS_Ecosystem_HomePage]]"
  - "[[NestJS_DTO]]"
  - "[[NestJS_Prisma]]"
  - "[[JS_URL_Encoding]]"
  - "[[NestJS_Prisma_Patterns]]"
---
# NestJS_Pagination — 페이지네이션

> [!info]
> 페이지네이션 = 많은 데이터를 나눠서 보내는 것.
> 핵심 용어(limit/take/skip/page)를 먼저 이해하고,
> Offset(페이지 번호) 방식 vs Cursor(마지막 항목 기준) 방식 중 선택.

---

# 페이지네이션이란 ⭐️⭐️⭐️⭐️

```txt
DB에 영화가 10,000개 있다면:
  findMany() → 10,000개 한 번에 전송 → 수십 MB → 서버/클라이언트 과부하

페이지네이션:
  "지금 화면에 보여줄 24개만 가져와"
  → 스크롤하면 "다음 24개 가져와"
  → 필요한 것만, 필요한 때에

HTTP 요청/응답 흐름:
  클라이언트  →  GET /movies?page=2&limit=24
  서버        →  { items: [...24개], total: 200, hasMore: true }
  클라이언트  →  GET /movies?page=3&limit=24
```

---

# 핵심 용어 — limit / page / take / skip / offset ⭐️⭐️⭐️⭐️⭐️

```txt
클라이언트가 보내는 것 (요청):
  limit   한 번에 받을 항목 수 (= 페이지 크기)  예: 24
  page    몇 번째 페이지인가                    예: 3

서버 내부에서 계산하는 것:
  take    DB에 실제로 요청할 개수 = limit 방어 보정 후 값
  skip    DB에서 건너뛸 개수     = (page - 1) * take
  offset  skip의 다른 이름 (SQL OFFSET)

서버가 보내는 것 (응답):
  total   전체 항목 수 (COUNT 쿼리)
  hasMore 다음 페이지가 있는가 (boolean)
```

```txt
limit vs take:
  limit = 클라이언트가 보낸 값  → 신뢰 불가 (0, -1, 99999 가능)
  take  = 서버가 보정한 값      → 항상 유효한 범위 (예: 1~48)

page vs skip:
  page = 1부터 시작하는 페이지 번호 (사람이 이해하기 쉬운 개념)
  skip = DB가 건너뛸 행 개수 (기계가 이해하는 오프셋)

  page=3, take=24 → skip = (3-1)*24 = 48
  → DB: "앞에 48개 건너뛰고, 그 다음 24개 줘"
```

```txt
SQL 대응:
  SELECT * FROM movies
  ORDER BY created_at DESC
  LIMIT 24     ← take
  OFFSET 48;   ← skip
```

| 용어 | 방향 | 예시 |
|---|---|---|
| `limit` | 클라이언트 → 서버 | `?limit=24` |
| `page` | 클라이언트 → 서버 | `?page=3` |
| `take` | 서버 내부 | `Math.min(Math.max(limit, 1), 48)` |
| `skip` | 서버 내부 → DB | `(page-1) * take` |
| `total` | 서버 → 클라이언트 | `200` |
| `hasMore` | 서버 → 클라이언트 | `true / false` |

---

# Offset vs Cursor — 두 가지 방식 ⭐️⭐️⭐️⭐️

```txt
Offset (페이지 번호):
  "3페이지 줘" = 앞 48개 건너뛰고 24개
  구현 쉬움, 특정 페이지 이동 가능
  실시간 데이터에서 중복/누락 발생

Cursor (마지막 항목 기준):
  "이 ID 다음 24개 줘"
  중복/누락 없음, 실시간 피드에 적합
  특정 페이지 이동 불가 (순서대로만)
```

| 구분 | Offset | Cursor |
|---|---|---|
| URL | `?page=3&limit=24` | `?cursor=abc&take=24` |
| 구현 난이도 | 쉬움 | 보통 |
| 중복/누락 | 실시간 변경 시 발생 | 없음 |
| 특정 페이지 이동 | 가능 | 불가 |
| 대용량 성능 | 느림 (OFFSET 클수록) | 빠름 |
| 사용 사례 | 관리자 목록, 정적 데이터 | 피드, 채팅, 무한 스크롤 |

---

# ① Offset 페이지네이션 ⭐️⭐️⭐️⭐️

## 파라미터 방어 보정

```typescript
async list(page = 1, limit = 24) {
  // 클라이언트 값을 그대로 쓰면 위험 → 보정 필수
  const take     = Math.min(Math.max(limit, 1), 48);
  const safePage = Math.max(page, 1);
  const skip     = (safePage - 1) * take;
}
```

```txt
take = Math.min(Math.max(limit, 1), 48):
  Math.max(limit, 1) → 음수/0 방지 (하한 1 보장)
  Math.min(..., 48)  → 99999 같은 과도한 값 방지 (상한 48 보장)
  limit이 뭐가 오든 → take는 항상 1~48

  이 패턴을 클램핑(clamping)이라 함:
    clamp(value, min, max) = Math.min(Math.max(value, min), max)
    별점: clamp(rating, 0, 5)   → 0~5 보장
    볼륨: clamp(volume, 0, 100) → 0~100 보장

safePage = Math.max(page, 1):
  page=0  → skip = (0-1)*24 = -24 → DB 오류
  page=-1 → skip = (-1-1)*24 = -48 → DB 오류
  → 반드시 1 이상으로 보정

skip = (safePage - 1) * take:
  page 1 → skip = 0  (처음부터)
  page 2 → skip = 24 (24개 건너뜀)
  page 3 → skip = 48 (48개 건너뜀)
```

## hasMore 계산

```typescript
hasMore: skip + pagedItems.length < total
```

```txt
take=24, total=100, page=2 (skip=24):
  24 + 24 = 48 < 100 → true  (아직 52개 남음)

마지막 페이지 page=5 (skip=96):
  pagedItems.length = 4 (남은 게 4개뿐)
  96 + 4 = 100 < 100 → false  (더 없음)

읽는 법:
  skip + 이번에 받은 수 = 지금까지 총 본 수
  지금까지 본 수 < 전체 → 아직 남아있음
```

## 전체 구현

```typescript
async list(page = 1, limit = 24) {
  // ① 파라미터 보정
  const take     = Math.min(Math.max(limit, 1), 48);
  const safePage = Math.max(page, 1);
  const skip     = (safePage - 1) * take;

  // ② DB 쿼리 (데이터 + 전체 개수 동시)
  const [rows, total] = await Promise.all([
    this.prisma.movie.findMany({
      skip,
      take,
      orderBy: { updatedAt: 'desc' },
      select:  { id: true, title: true, updatedAt: true },
    }),
    this.prisma.movie.count(),
  ]);

  // ③ 응답
  return {
    items:   rows,
    page:    safePage,
    total,
    hasMore: skip + rows.length < total,
  };
}
```

```txt
Promise.all로 두 쿼리 동시 실행:
  findMany (데이터) + count (전체 수) → 순차 실행하면 2배 느림
  Promise.all → 병렬 실행 → 더 빠름

COUNT 없이 hasMore만 판단하려면 take+1 트릭:
  const rows = findMany({ skip, take: take + 1 });
  const hasMore = rows.length > take;
  const items = hasMore ? rows.slice(0, take) : rows;
  → count 쿼리 없이 "더 있음" 여부만 판단 (total은 알 수 없음)
```

---

# ② Cursor 페이지네이션 ⭐️⭐️⭐️⭐️

## 왜 Cursor인가

```txt
Offset의 문제:
  1페이지 [A, B, C, ... J] 읽는 동안 맨 앞에 X 추가됨
  2페이지 skip=10 → J가 다시 나옴 (중복)
  삭제되면 항목이 빠짐 (누락)

Cursor:
  "A~J를 봤고 마지막이 J야, J 다음부터 줘"
  → 중간에 뭐가 추가/삭제돼도 "J 다음"은 변하지 않음
```

## take + 1 트릭 — COUNT 없이 hasMore 판단

```txt
문제: 다음 페이지가 있는지 알려면 COUNT가 필요 → 쿼리 2번

해결: 요청할 때 take+1 개를 가져옴
  take=24이면 25개 요청
  25개 왔으면 → 더 있음 (hasMore=true), 25번째는 잘라냄
  24개 이하 왔으면 → 마지막 페이지 (hasMore=false)
  → 쿼리 1번으로 해결
```

```typescript
const take = 24;
const rows = await prisma.movie.findMany({ take: take + 1, ... });

const hasMore = rows.length > take;       // 25개 왔으면 true
const items   = hasMore ? rows.slice(0, take) : rows;  // 25번째 제거

return {
  items,
  nextCursor: hasMore ? items[items.length - 1].id : null,
  //                    ↑ 마지막(24번째).id = 다음 요청의 cursor
};
```

## 전체 구현

```typescript
async list(cursor?: string, take = 24) {
  // cursor가 있으면 그 다음부터, 없으면 처음부터
  const cursorOption = cursor
    ? { cursor: { id: cursor }, skip: 1 }
    : {};
  //  cursor: { id } → 이 항목을 기준점으로
  //  skip: 1       → 기준점 자체는 건너뜀 (이미 받은 것)

  const rows = await this.prisma.movie.findMany({
    ...cursorOption,
    take: take + 1,
    orderBy: [
      { createdAt: 'desc' },
      { id: 'asc' },          // 동률일 때 일관된 순서 보장 (필수!)
    ],
    select: { id: true, title: true, createdAt: true },
  });

  const hasMore = rows.length > take;
  const items   = hasMore ? rows.slice(0, take) : rows;

  return {
    items,
    nextCursor: hasMore ? (items.at(-1)?.id ?? null) : null,
  };
}
```

```txt
orderBy에 id를 추가하는 이유:
  createdAt이 같은 항목이 여러 개면 순서가 매번 달라질 수 있음
  id(고유값)를 추가하면 동률에서도 순서 고정
  → cursor 기반 페이지네이션이 정확하게 동작

  규칙: orderBy 마지막에 항상 고유 필드(id) 추가
```

## 응답 타입

```typescript
type CursorPage<T> = {
  items:      T[];
  nextCursor: string | null;  // null = 마지막 페이지
};
```

---

# 복합 커서 — createdAt + id 조합 ⭐️⭐️⭐️

```txt
Prisma cursor: { id } 의 한계:
  Prisma 기본 cursor는 내부적으로 OFFSET처럼 동작
  orderBy가 createdAt인데 cursor가 id면 순서가 안 맞을 수 있음

더 정확한 방법:
  마지막 항목의 createdAt과 id를 WHERE로 직접 범위 지정
```

```typescript
let cursorWhere = {};

if (cursor) {
  const cursorRow = await this.prisma.post.findFirst({
    where:  { id: cursor },
    select: { id: true, createdAt: true },
  });

  if (cursorRow) {
    cursorWhere = {
      OR: [
        { createdAt: { lt: cursorRow.createdAt } },                     // 더 오래된 것
        { createdAt: cursorRow.createdAt, id: { lt: cursorRow.id } },   // 동시각이면 id로
      ],
    };
  }
}

const rows = await this.prisma.post.findMany({
  where:   { ...cursorWhere, ...otherWhere },
  orderBy: [{ createdAt: 'desc' }, { id: 'desc' }],
  take:    take + 1,
});
```

```txt
OR 두 조건:
  ① createdAt < T           → 확실히 이전 (시간이 더 오래됨)
  ② createdAt = T, id < xyz → 같은 시각이면 id 기준으로

  이 두 조건의 합집합 = cursor 이후 행 전체
  → Prisma cursor 방식보다 정확한 범위 지정
```

---

# 클라이언트 — 무한 스크롤 ⭐️⭐️⭐️

```typescript
const [items,   setItems]   = useState<Movie[]>([]);
const [cursor,  setCursor]  = useState<string | null>(null);
const [hasMore, setHasMore] = useState(true);

const loadMore = async () => {
  const params = new URLSearchParams({ take: '24' });
  if (cursor) params.set('cursor', cursor);

  const res = await apiFetch<CursorPage<Movie>>(`/movies?${params}`);

  setItems(prev => cursor ? [...prev, ...res.items] : res.items);
  setCursor(res.nextCursor);
  setHasMore(res.nextCursor !== null);
};

// 스크롤 끝에 도달 시
const onScrollEnd = () => {
  if (hasMore) void loadMore();
};
```

```txt
cursor가 null이면 처음부터 (초기 로드 or 검색어 변경 시 리셋)
cursor가 있으면 그 다음부터 (스크롤로 추가 로드)
nextCursor가 null이면 마지막 → "더 보기" 버튼/트리거 숨김
```

---

# 자주 만나는 문제

| 증상 | 원인 | 해결 |
|---|---|---|
| 항목 중복 노출 | orderBy에 id 누락 | orderBy 마지막에 id 추가 |
| cursor 항목이 결과에 포함 | `skip: 1` 누락 | cursor 있을 때 skip: 1 추가 |
| 마지막 페이지인데 hasMore=true | take+1 로직 오류 | `rows.length > take` 조건 확인 |
| 검색 후 이전 결과 섞임 | q 변경 시 items 초기화 안 함 | q 변경 시 items/cursor 리셋 |
| skip이 음수 | page=0/-1 입력 | `Math.max(page, 1)` |
| limit=99999로 DB 과부하 | 상한 없음 | `Math.min(limit, 48)` |
