---
aliases:
  - Pagination
  - Offset
  - Cursor
  - take
  - query
tags:
  - NestJS
related:
  - "[[00_NestJS_Ecosystem_HomePage]]"
  - "[[NestJS_DTO]]"
  - "[[NestJS_Prisma]]"
  - "[[JS_URL_Encoding]]"
  - "[[NestJS_Prisma_Patterns]]"
---
# NestJS_Pagination — 페이지네이션

> [!info] 
> 페이지네이션 = 많은 데이터를 한 번에 다 보내지 않고 나눠서 보내는 것. 
> Offset(페이지 번호) 방식과 Cursor(마지막 항목 기준) 방식이 있고, NestJS + Prisma에서는 Cursor 방식이 더 안전하다. 
> `take + 1` 트릭으로 추가 COUNT 쿼리 없이 "다음 페이지 있음" 여부를 판단한다.

---

# 왜 필요한가 ⭐️⭐️⭐️⭐️

```txt
DB에 유저가 100,000명 있다면:
  findMany() → 100,000개를 한 번에 응답 → 수십 MB → 서버/클라이언트 둘 다 과부하

페이지네이션:
  "지금 당장 보여줄 20개만 가져와"
  → 사용자가 스크롤하면 "다음 20개 가져와"
  → 화면에 필요한 것만, 필요한 때에 가져옴
```

---

# Offset vs Cursor — 두 가지 방식 ⭐️⭐️⭐️⭐️

## Offset (페이지 번호 방식)

```typescript
// "3페이지 줘" = 앞 40개 건너뛰고 20개
prisma.user.findMany({ skip: 40, take: 20 });
//                     ↑ OFFSET    ↑ LIMIT
```

```txt
문제 — 데이터가 중간에 추가/삭제되면 틀어짐:

  1페이지 (skip 0): [A, B, C, D, E, F, G, H, I, J]
  사용자가 읽는 동안 맨 앞에 새 항목 X 추가됨
  2페이지 (skip 10): [J, K, L, M, N, O, P, Q, R, S]
                      ↑ J가 두 번 보임! (1페이지에도 있었음)

  반대로 항목이 삭제되면 빠지는 항목 발생

실시간으로 데이터가 바뀌는 채팅·피드에서 부적합
단순한 관리자 페이지처럼 정적인 데이터에는 괜찮음
```

## Cursor (마지막 항목 기준 방식)

```typescript
// "id='abc' 다음부터 20개 줘"
prisma.user.findMany({
  cursor: { id: 'abc' },
  skip: 1,    // cursor 항목 자체는 제외
  take: 20,
});
```

```txt
동작 원리:
  1페이지: 처음부터 20개 → 마지막 항목 id = 'abc'
  2페이지: 'abc' 이후부터 20개 → 마지막 항목 id = 'xyz'
  3페이지: 'xyz' 이후부터 20개

  중간에 새 항목이 생겨도:
  "abc 이후부터"라는 기준이 변하지 않음 → 중복/누락 없음

cursor = "내가 마지막으로 본 항목의 ID"
nextCursor = 응답에서 알려주는 다음 페이지의 시작점
```

## 비교

|구분|Offset|Cursor|
|---|---|---|
|구현 난이도|쉬움|보통|
|URL 파라미터|`?page=3`|`?cursor=abc`|
|중복/누락|실시간 변경 시 발생|없음|
|특정 페이지 이동|가능 (`?page=5`)|불가 (순서대로만)|
|대용량 성능|느림 (OFFSET이 클수록)|빠름|
|사용 사례|관리자 목록, 정적 데이터|피드, 채팅, 무한 스크롤|

---

# take + 1 트릭 — hasMore 판단 ⭐️⭐️⭐️⭐️

```txt
문제: 다음 페이지가 있는지 어떻게 알 수 있을까?

방법 1 — COUNT 쿼리:
  SELECT COUNT(*) FROM users WHERE ...  → 총 100,000개
  현재 skip=40, take=20 → 아직 남아있음
  → 쿼리 2번 (SELECT + COUNT) → 성능 손해

방법 2 — take + 1 트릭:
  take=20 이면 21개를 요청
  21개 돌아왔으면 → 더 있음 (hasMore = true)
  20개 이하 돌아왔으면 → 마지막 페이지 (hasMore = false)
  응답에는 21번째 항목을 잘라내고 20개만 보냄
  → 쿼리 1번으로 해결
```

```typescript
const take = 20;
const rows = await prisma.user.findMany({ take: take + 1, ... });
//                                              ↑ 21개 요청

const hasMore = rows.length > take;             // 21개 왔으면 true
const items   = hasMore ? rows.slice(0, take) : rows;  // 21번째 제거
//                                 ↑ 20개만 반환

return {
  items,
  nextCursor: hasMore ? items[items.length - 1].id : null,
  //                    ↑ 마지막 항목(20번째)의 id = 다음 요청의 cursor
};
```

```txt
take + 1 흐름 시각화:

  요청한 것: [1, 2, 3, 4, 5, 6, 7, 8, 9, 10, 11, 12, 13, 14, 15, 16, 17, 18, 19, 20, 21]
                                                                                        ↑ 21번째 = 더 있음의 증거
  응답:      [1, 2, 3, 4, 5, 6, 7, 8, 9, 10, 11, 12, 13, 14, 15, 16, 17, 18, 19, 20]
                                                                                   ↑ nextCursor
```

---

# 구현 전체 ⭐️⭐️⭐️⭐️

## Query DTO

```typescript
// cursor-query.dto.ts
export class CursorQueryDto {
  @IsOptional()
  @IsString()
  cursor?: string;   // 이전 응답의 nextCursor 값
  //                    첫 요청은 없음, 이후 요청에 포함

  @IsOptional()
  @IsInt()
  @Min(1)
  @Max(100)
  @Type(() => Number)
  take?: number = 20;  // 기본값 20

  @IsOptional()
  @IsString()
  q?: string;   // 검색어 (선택)
}
```

```txt
cursor:
  첫 요청 → undefined (처음부터)
  이후 요청 → 이전 응답의 nextCursor 값

take:
  한 번에 받을 항목 수
  클라이언트가 조절 가능 (무한 스크롤: 20, 리스트: 50 등)

q (search query):
  검색어 — 있으면 LIKE 검색 포함, 없으면 전체
  이름이 q인 이유: 관례 (query의 약자, URL에서 ?q=홍길동)
```

## Prisma 쿼리

```typescript
async findUsers(query: CursorQueryDto) {
  const take = query.take ?? 20;

  // ① where 조건 — 검색어가 있으면 필터 추가
  const where: Prisma.UserWhereInput = query.q?.trim()
    ? {
        OR: [
          { nickname: { contains: query.q, mode: 'insensitive' } },
          { email:    { contains: query.q, mode: 'insensitive' } },
        ],
      }
    : {};

  // ② cursor 조건 — 있으면 그 항목 다음부터
  const cursorOption = query.cursor
    ? { cursor: { id: query.cursor }, skip: 1 }
    : {};
  //  cursor: { id: '...' }  →  이 id를 가진 항목을 기준점으로
  //  skip: 1                →  기준점 자체는 건너뜀 (이미 이전에 받음)

  // ③ take + 1 요청
  const rows = await this.prisma.user.findMany({
    where,
    take: take + 1,
    ...cursorOption,
    orderBy: [
      { nickname: 'asc' },   // 1차 정렬: 이름 오름차순
      { id: 'asc' },         // 2차 정렬: id (동률일 때 일관된 순서 보장)
    ],
    select: { id: true, nickname: true, image: true },
  });

  // ④ hasMore 판단 + nextCursor 계산
  const hasMore = rows.length > take;
  const items   = hasMore ? rows.slice(0, take) : rows;

  return {
    items,
    nextCursor: hasMore ? (items[items.length - 1].id ?? null) : null,
  };
}
```

## orderBy에 id를 추가하는 이유 ⭐️⭐️⭐️

```txt
nickname만으로 정렬하면:
  nickname = '홍길동'인 유저가 여러 명 있을 때
  DB가 임의의 순서로 반환 → 페이지가 바뀌면 순서가 달라질 수 있음

nickname + id로 정렬하면:
  nickname이 같더라도 id로 순서가 고정됨
  cursor 기반 페이지네이션이 정확하게 동작

규칙: orderBy에 항상 고유한 값(id)을 마지막에 추가
```

---

# 응답 타입 ⭐️⭐️⭐️

```typescript
// 페이지네이션 응답의 공통 구조
type CursorPage<T> = {
  items:      T[];           // 이번 페이지 항목들
  nextCursor: string | null; // null = 마지막 페이지
};
```

```typescript
// 사용 예
return {
  items,
  nextCursor: hasMore ? items[items.length - 1].id ?? null : null,
} satisfies CursorPage<typeof items[0]>;
```

---

# 클라이언트에서 사용 — 무한 스크롤 ⭐️⭐️⭐️

```typescript
// 상태 관리
const [items, setItems] = useState<User[]>([]);
const [cursor, setCursor] = useState<string | null>(null);
const [hasMore, setHasMore] = useState(true);

// 첫 로드 또는 검색어 변경
useEffect(() => {
  setItems([]);
  setCursor(null);
  setHasMore(true);
  void loadMore(null);
}, [q]);

// 다음 페이지 로드
const loadMore = async (currentCursor: string | null) => {
  const params = new URLSearchParams({ take: '20' });
  if (currentCursor) params.set('cursor', currentCursor);
  if (q)            params.set('q', q);

  const res = await apiFetch<CursorPage<User>>(`/users?${params}`);

  setItems(prev => currentCursor ? [...prev, ...res.items] : res.items);
  setCursor(res.nextCursor);
  setHasMore(res.nextCursor !== null);
};

// 스크롤 끝에 도달했을 때
const onScrollEnd = () => {
  if (hasMore && cursor) void loadMore(cursor);
};
```

```txt
nextCursor가 null이면 → 마지막 페이지 → "더 보기" 버튼 숨김
nextCursor가 있으면  → 다음 요청에 cursor 파라미터로 전달
```

---

# 자주 만나는 문제

| 증상                       | 원인                    | 해결                          |
| ------------------------ | --------------------- | --------------------------- |
| 항목이 중복 노출                | orderBy에 고유 필드(id) 누락 | orderBy 마지막에 id 추가          |
| 마지막 페이지인데 hasMore = true | take + 1 로직 오류        | `rows.length > take` 조건 확인  |
| cursor 항목이 결과에 포함됨       | skip: 1 누락            | cursor 있을 때 skip: 1 추가      |
| 검색 후 이전 결과가 섞임           | q 변경 시 items 초기화 안 함  | q 변경 시 items/cursor 리셋      |
| 첫 요청에 cursor 포함          | 초기화 안 됨               | 첫 요청은 cursor 없이 (undefined) |

---
# 복합 커서 — createdAt + id 조합 ⭐️⭐️⭐️⭐️

```txt
기본 Prisma cursor({ id }) 방식의 한계:
  cursor: { id } 는 Prisma가 내부적으로 id 기준으로 찾아서 SKIP하는 방식
  orderBy가 createdAt 기준인데 cursor가 id면 순서가 맞지 않을 수 있음

createdAt 기준 정렬 + id 커서를 안전하게 조합하는 방법:
  마지막 항목의 createdAt과 id를 기억
  "이 시각보다 이전 OR 같은 시각이면서 id가 더 작은 것"으로 필터
```


```typescript
// 서비스
let cursorWhere: object = {};

if (query.cursor) {
  // ① cursor ID로 해당 행의 실제 createdAt 조회
  const cursorRow = await this.prisma.post.findFirst({
    where:  { id: query.cursor },
    select: { id: true, createdAt: true },
  });

  if (cursorRow) {
    // ② createdAt + id 조합으로 "다음" 범위 설정
    cursorWhere = {
      OR: [
        { createdAt: { lt: cursorRow.createdAt } },                          // 더 오래된 것
        { createdAt: cursorRow.createdAt, id: { lt: cursorRow.id } },         // 같은 시각이면 id로
      ],
    };
  }
}

const rows = await this.prisma.post.findMany({
  where: { ...cursorWhere, ...otherWhere },
  orderBy: [{ createdAt: 'desc' }, { id: 'desc' }],  // createdAt 먼저, 동률이면 id
  take: take + 1,
});
```


```txt
OR 조건이 두 가지인 이유:

  정렬: [createdAt desc, id desc]
  cursor: createdAt = T, id = 'xyz'

  "cursor 다음"이란:
  ① createdAt < T   → 확실히 이전 (시간이 더 오래됨)
  ② createdAt = T, id < 'xyz'  → 같은 시각인데 id가 더 작음 (id desc 정렬 기준)

  이 두 조건의 합집합 = cursor 이후에 오는 모든 행

Prisma cursor: { id } 와의 차이:
  Prisma 기본 cursor는 내부적으로 OFFSET 비슷하게 동작
  → orderBy createdAt인데 cursor id면 매칭이 부정확할 수 있음
  복합 커서는 WHERE 조건으로 직접 범위를 지정 → 정확함

nextCursor로 뭘 넘기는가:
  items[items.length - 1].id  ← 마지막 항목의 id만 넘김
  다음 요청에서 이 id로 createdAt을 다시 조회해서 복합 커서 생성
  id 하나로 커서를 표현하되, 서버에서 createdAt도 함께 조회
```