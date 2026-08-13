---
aliases:
  - 외부 API
  - 캐시 테이블(Pool)
  - ORDER BY RANDOM()
  - Cache-aside (Look-aside)
  - Hit 가드
tags:
  - NestJS
related:
  - "[[00_JS_Ecosystem_HomePage]]"
  - "[[NestJS_Scheduling]]"
  - "[[PG_DML]]"
---
# NestJS_CacheTable — 외부 API 캐시 테이블 패턴

>[!info]
>외부 API(TMDB·공공 API·Open API 등)를 매번 직접 호출하는 대신, 미리 데이터를 가져와 로컬 DB 테이블에 쌓아두는 패턴. 
>이 테이블을 **Pool(풀)** 또는 **캐시 테이블**이라 부른다. 
>앱은 외부 API 대신 DB만 조회하여 N+1·rate limit·랜덤 선택 문제가 사라진다. 크론 동기화 → [[NestJS_Scheduling]]

---

# 언제 이 패턴이 필요한가 ⭐️⭐️⭐️⭐️

```txt
외부 API를 매번 직접 호출할 때 생기는 문제:

  N+1:
    목록 20개 → 각 항목 상세를 외부 API로 20번 호출
    → 응답 느림, 비용 증가

  Rate Limit:
    외부 API는 초당·일일 호출 한도가 있음
    유저가 몰리면 429 Too Many Requests
    공공 API는 특히 제한이 빡빡함

  랜덤 선택 불가:
    외부 API 결과는 순서가 고정
    DB의 ORDER BY RANDOM()을 쓸 수 없음

  조건 필터링 어려움:
    외부 API에 없는 내부 조건(태그·머신·카테고리)을 적용하기 어려움

Pool(캐시 테이블)이 해결하는 것:
  크론이 미리 외부 API → DB에 동기화
  앱은 DB Pool만 조회
  → 빠름 · rate limit 무관 · RANDOM() 가능 · 필터 자유
```

---

# 테이블 설계 원칙 ⭐️⭐️⭐️⭐️

```prisma
model ResourcePool {
  id         String   @id @default(cuid())

  // 외부 API의 고유 식별자 — 중복 방지의 핵심
  sourceId   String   @unique

  // 자주 쓰는 메타데이터 (조회·필터에 쓸 것만)
  title      String
  imageUrl   String?
  metadata   Json?    // 구조가 불규칙하면 Json으로 유연하게

  // 내부 분류 태그 (외부 API에 없는 내부 기준)
  tags       String[]
  isActive   Boolean  @default(true)   // 숨기기·비활성화용

  // 동기화 추적
  syncedAt   DateTime @default(now())  // 마지막 동기화 시각
  createdAt  DateTime @default(now())

  @@map("resource_pool")
}
```

```txt
sourceId @unique:
  외부 API의 ID (tmdbId, publicApiId 등)
  upsert의 where 조건 — 중복 INSERT 방지

syncedAt:
  크론이 오래된 레코드를 재동기화할 때 기준
  WHERE syncedAt < NOW() - INTERVAL '7 days'

tags / isActive:
  외부 API에 없는 내부 분류 기준
  특정 풀(방·카테고리)과 연결하거나 숨기기 처리

metadata Json?:
  외부 API 응답 전체를 저장하거나
  구조가 바뀔 수 있는 부가 정보 보관용
```

---

# 동기화 전략 — upsert + 배치 ⭐️⭐️⭐️⭐️

```typescript
// pool.service.ts
@Injectable()
export class PoolService {
  constructor(private readonly prisma: PrismaService) {}

  async syncFromExternalApi(pages = 5): Promise<void> {
    for (let page = 1; page <= pages; page++) {
      const items = await this.fetchPage(page);

      // upsert — 있으면 업데이트, 없으면 삽입
      await this.prisma.$transaction(
        items.map((item) =>
          this.prisma.resourcePool.upsert({
            where:  { sourceId: String(item.id) },
            update: {
              title:    item.title,
              imageUrl: item.image,
              syncedAt: new Date(),
            },
            create: {
              sourceId: String(item.id),
              title:    item.title,
              imageUrl: item.image,
              metadata: item,           // 원본 전체 보관
            },
          }),
        ),
      );

      // rate limit 여유 — 페이지 사이 대기
      await new Promise(resolve => setTimeout(resolve, 300));
    }
  }

  private async fetchPage(page: number) {
    const res = await fetch(`${EXTERNAL_API_URL}?page=${page}`, {
      headers: { Authorization: `Bearer ${API_KEY}` },
    });
    const data = await res.json();
    return data.results ?? data.items ?? [];
  }
}
```

```txt
upsert가 필요한 이유:
  크론이 반복 실행되면 같은 sourceId가 다시 들어옴
  INSERT만 하면 중복 에러
  upsert = "이미 있으면 update, 없으면 create"

배치 크기 조절:
  items가 많으면 $transaction 한 번에 너무 많은 쿼리
  → 100개씩 chunk해서 처리하는 것도 방법

대기 시간 (300ms):
  외부 API rate limit 소모를 최소화
  크론은 새벽에 돌기 때문에 여유 있게 설정 가능
```

---

# 크론으로 자동 동기화 ⭐️⭐️⭐️

```typescript
// pool.scheduler.ts
@Injectable()
export class PoolScheduler {
  constructor(private readonly poolService: PoolService) {}

  // 매일 새벽 3시 동기화
  @Cron('0 3 * * *', { timeZone: 'Asia/Seoul' })
  async scheduledSync() {
    await this.poolService.syncFromExternalApi(10);
  }
}
```

```txt
크론 주기 결정 기준:
  데이터 변경이 잦은가? → 주기 짧게 (매시간)
  데이터가 거의 안 바뀌는가? → 주기 길게 (매주)
  트래픽 없는 시간대(새벽)에 실행 → 서비스 영향 없음

수동 트리거도 만들어두면 편리:
  @Post('admin/sync') 엔드포인트
  → 급하게 동기화가 필요할 때 관리자가 직접 실행
```

---

# Pool에서 조회 ⭐️⭐️⭐️⭐️

## 랜덤 선택

```typescript
// DB에서 랜덤으로 N개 선택
async drawRandom(options: {
  count:      number;
  tags?:      string[];    // 태그 필터
  excludeIds?: string[];   // 이미 선택된 것 제외
}): Promise<ResourcePool[]> {
  return this.prisma.$queryRaw`
    SELECT *
    FROM resource_pool
    WHERE is_active = true
      ${options.tags?.length
        ? Prisma.sql`AND tags && ${options.tags}::text[]`
        : Prisma.empty}
      ${options.excludeIds?.length
        ? Prisma.sql`AND source_id != ALL(${options.excludeIds}::text[])`
        : Prisma.empty}
    ORDER BY RANDOM()
    LIMIT ${options.count}
  `;
}
```

## N+1 해소

```typescript
// 목록 조회 시 외부 API 호출 없이 Pool JOIN
const posts = await this.prisma.post.findMany({
  include: {
    resourcePool: true,   // Pool 테이블에서 JOIN — API 호출 0번
  },
});
```

```txt
ORDER BY RANDOM():
  PostgreSQL 내장 — 매번 다른 순서 보장
  뽑기·추천 기능에서 공정한 랜덤 선택 [[PG_DML]]

tags && array (&&):
  PostgreSQL 배열 겹침 연산자
  pool.tags = ['horror', 'thriller'] && ['horror'] → 매칭

N+1 해소:
  기존: 목록 20개 → 외부 API 20번 → 느림
  Pool: JOIN 한 번 → 즉시 반환
```

---

# 패턴이 적합하지 않은 경우

```txt
실시간 데이터가 필요할 때:
  주가, 환율, 재고 — 캐시가 바로 구식(stale)이 됨
  → 직접 호출이 맞음

데이터 변경이 너무 잦을 때:
  하루에 수백 번 바뀌는 데이터
  → 크론 주기를 따라갈 수 없음

트래픽이 거의 없을 때:
  rate limit 걱정이 없으면 굳이 복잡도를 높일 필요 없음

외부 API 응답이 앱마다 달라야 할 때:
  유저별 personalization이 필요한 경우
  → 풀에서 선택 후 앱 레이어에서 재정렬
```

---
# 세부 구현 패턴 ⭐️⭐️⭐️⭐️

## Mapper — DB row → 앱 형태 변환

```typescript
// DB 스키마와 앱 DTO를 분리 — 둘 중 하나가 바뀌어도 mapper만 수정
private fromPool(row: ResourcePool): AppResource {
  return {
    id:          row.sourceId,
    title:       row.title,
    imageUrl:    row.imageUrl,
    description: row.description,
    // DB 컬럼명 → 앱 DTO 필드명으로 변환
  };
}
```

```txt
fromPool 패턴을 쓰는 이유:
  DB 컬럼명과 앱 DTO 필드명이 다를 때
  외부 API 응답 형태와 DB 저장 형태가 다를 때
  → 변환 로직을 한 곳에 모아서 유지보수 쉬움
```


## Cache-aside — 캐시 우선, 없으면 외부 API

```typescript
async getCached(externalId: string): Promise<AppResource> {
  // 1. 캐시(Pool) 먼저 확인
  const cached = await this.prisma.resourcePool.findUnique({
    where: { sourceId: externalId },
  });
  if (cached) return this.fromPool(cached); // 캐시 히트 → 즉시 반환

  // 2. 캐시 미스 → 외부 API 호출
  const data = await this.externalApi.fetch(externalId);

  // 3. 결과를 Pool에 저장 (다음 요청엔 캐시 히트)
  await this.prisma.resourcePool.upsert({
    where:  { sourceId: externalId },
    create: { sourceId: externalId, title: data.title },
    update: { title: data.title, syncedAt: new Date() },
  });

  return data;
}
```

```txt
Cache-aside (Look-aside) 패턴:
  캐시 히트 → DB 반환 (빠름)
  캐시 미스 → 외부 API → DB 저장 → 반환 (느리지만 1회)
  두 번째 요청부터는 항상 히트

upsert를 쓰는 이유:
  동시 요청이 같은 캐시 미스 → 두 번 create → 충돌
  upsert = "있으면 update, 없으면 create" → 안전
```

## Hit 가드 — 불완전한 캐시 걸러내기 ⭐️⭐️⭐️⭐

```txt
캐싱 용어 먼저:
  Cache Hit  = 캐시에서 데이터를 찾음 → DB 반환 (빠름)
  Cache Miss = 캐시에 없음 → 외부 API 호출 (느림)

  일반적으로 Hit = "데이터가 있으니 쓸 수 있다"고 가정
  → 하지만 항상 그렇지 않음

문제 — "있지만 불완전한" 캐시:
  seedPool()로 Discover 목록을 먼저 저장 (title, poster만)
  상세 필드(genreIds, originCountries)는 아직 비어 있음
  이후 getCached() 호출 → Cache Hit
  → 하지만 genreIds = [] → 장르 필터 기능 못 씀

Hit 가드:
  "Hit가 됐어도, 내가 필요한 데이터가 들어있는지 확인"
  = 캐시 존재 여부(있음/없음)가 아닌 캐시 유효성(완전/불완전) 확인
  불완전하면 → Miss처럼 취급 → 외부 API 재호출 → 완전한 데이터로 덮어씀
```


```txt
문제:
  seedPool()이나 배치 동기화로 row가 먼저 생성됨
  이 시점에 genreIds · originCountries 같은 상세 필드가 비어 있음
  이후 getCached() 호출 시 캐시 "히트"가 되지만
  데이터가 불완전 → 화면에 장르·국가가 안 뜸

해결 — hit 가드:
  캐시가 있어도 "필수 필드가 있는지" 확인
  없으면 히트로 취급하지 않고 외부 API를 다시 호출
  → 이번에 가져온 데이터로 row를 update(upsert)
```

```typescript
async getCached(externalId: string): Promise<AppResource> {
  const cached = await this.prisma.resourcePool.findUnique({
    where: { sourceId: externalId },
  });

  // Hit 가드 — 있어도 필수 필드가 비어 있으면 히트 아님
  if (
    cached &&
    (cached.genreIds.length > 0 || cached.originCountries.length > 0)
  ) {
    return this.fromPool(cached);  // 완전한 데이터 → 반환
  }
  // cached가 없거나 데이터 불완전 → 외부 API 호출

  const data = await this.externalApi.fetch(externalId);

  await this.prisma.resourcePool.upsert({
    where:  { sourceId: externalId },
    create: { sourceId: externalId, title: data.title, genreIds: data.genre_ids },
    update: { title: data.title, genreIds: data.genre_ids, syncedAt: new Date() },
    //       ↑ 이번에 가져온 완전한 데이터로 덮어씀
  });

  return data;
}
```


```txt
hit 가드 조건 설계:
  "이 필드가 없으면 불완전하다"는 기준을 코드로 표현
  length > 0  → 배열이 채워져 있는지
  !== null    → 단일 값이 있는지
  !== ''      → 문자열이 비어있지 않은지

  너무 엄격하면: 항상 외부 API 호출 → 캐시 의미 없음
  너무 느슨하면: 불완전한 데이터 반환

  기준 예시:
    최소 데이터(title만 있으면 됨) → hit 가드 불필요
    장르·국가 필터 기능이 있음     → genreIds.length > 0 필요
    상세 페이지가 있음              → overview, posterPath 필요

언제 불완전한 캐시가 생기는가:
  seedPool()이 Discover 목록만 가져오고 상세는 생략했을 때
  배치가 실패해서 일부 필드만 저장됐을 때
  스키마에 새 컬럼을 추가했지만 기존 row는 비어 있을 때
  → hit 가드로 "필요한 필드가 있는 row만" 캐시로 사용
```

---
## Prisma 랜덤 선택 — skip + count

```typescript
// raw SQL 없이 Prisma만으로 랜덤 선택
async pickRandom(where: object = {}): Promise<ResourcePool | null> {
  const count = await this.prisma.resourcePool.count({ where });
  if (count === 0) return null;

  return this.prisma.resourcePool.findFirst({
    where,
    skip: Math.floor(Math.random() * count),
    // 전체 count 중 무작위 위치로 건너뜀 → 랜덤 효과
  });
}
```

```txt
skip + count 랜덤 vs ORDER BY RANDOM():
  skip + count: Prisma만으로 가능 · raw SQL 불필요
  ORDER BY RANDOM(): $queryRaw · 행마다 난수 붙여 정렬

  둘 다 소량(수백~수만)이면 MVP에 충분
  둘 다 대용량에선 약함
    - skip/OFFSET 커지면 느림
    - ORDER BY RANDOM() = 전체 정렬이라 더 부담

  수십만 건 이상 → TABLESAMPLE 또는 랜덤 ID 방식
  → [[PG_DML]] ORDER BY RANDOM() 섹션
```

---
## Fallback 전략 — Pool → 외부 API

```typescript
async pickWithFallback(
  filters: Record<string, string> = {},
  excludeIds: string[] = [],
): Promise<AppResource> {

  // 조건 없는 요청 → Pool 우선
  if (Object.keys(filters).length === 0) {
    const where = excludeIds.length
      ? { sourceId: { notIn: excludeIds } }
      : {};

    const count = await this.prisma.resourcePool.count({ where });
    if (count > 0) {
      const row = await this.prisma.resourcePool.findFirst({
        where,
        skip: Math.floor(Math.random() * count),
      });
      if (row) return this.fromPool(row);
    }
  }

  // 특수 조건 또는 Pool 비어있음 → 외부 API 폴백
  const result = await this.externalApi.search(filters);
  if (!result.items.length) {
    throw new ServiceUnavailableException('결과를 찾지 못했습니다.');
  }

  // 재시도로 excludeIds 처리
  for (let attempt = 0; attempt < 8; attempt++) {
    const item = result.items[Math.floor(Math.random() * result.items.length)];
    if (!excludeIds.includes(item.id)) {
      return this.getCached(item.id); // 캐시에 저장하면서 반환
    }
  }
  throw new ServiceUnavailableException('선택 가능한 항목이 없습니다.');
}
```

```txt
Fallback 설계 기준:
  필터 없음 + Pool에 데이터 있음 → Pool (빠름, rate limit 무관)
  특수 필터 또는 Pool 비어있음   → 외부 API

  getCached()와 연결:
  외부 API 결과도 getCached()로 가져오면
  → 자동으로 Pool에 쌓임 → 다음엔 Pool에서 히트

재시도 (attempt 루프):
  excludeIds에 걸리면 다시 랜덤 선택
  최대 N번 시도 후 없으면 예외
```