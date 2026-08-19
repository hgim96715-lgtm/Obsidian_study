---
aliases:
  - stats bucket
  - time bucket
  - aggregation
  - buildDailyBuckets
tags:
  - NestJS
related:
  - "[[00_NestJS_Ecosystem_HomePage]]"
  - "[[Snippet_date-statistics-pattern]]"
  - "[[React_Charts]]"
  - "[[JS_Array_Methods]]"
---
# NestJS_StatsBucket — 통계 집계 패턴

> [!info] 
> 통계 버킷 = 시간 단위(일·시간)로 카운트를 쌓는 DB 테이블 + 집계 로직. 
> 이벤트가 발생할 때마다 `upsert`로 누적하거나, 쿼리 시 `GROUP BY`로 실시간 집계하는 두 가지 방법이 있다. 
> KST 기준 날짜 처리 → [[Snippet_date-statistics-pattern]], 차트 연동 → [[React_Charts]]

---

# 통계 버킷이란 ⭐️⭐️⭐️⭐️

```txt
버킷(bucket) = "시간 단위 칸"

  일별 버킷: 2026-08-12: 120명, 2026-08-13: 85명 ...
  시간별 버킷: 0시: 5명, 1시: 2명, ..., 23시: 18명

왜 버킷인가:
  차트의 x축 = 시간 단위
  "날짜별 방문자 수"를 차트로 보려면
  각 날짜에 해당하는 카운트가 필요

두 가지 집계 방법:
  ① 실시간 집계 (Real-time aggregation)
     이벤트 원본 테이블 → 쿼리 시 GROUP BY로 집계
     장점: DB 단순, 구현 빠름
     단점: 데이터가 많아지면 쿼리 느림

  ② 사전 집계 (Pre-aggregation)
     이벤트 발생 즉시 stat 테이블에 upsert로 누적
     별도 dailyStat, hourlyStat 테이블 관리
     장점: 조회 빠름, 차트용 데이터 즉시 반환
     단점: 테이블 추가, upsert 로직 필요
```

---

# 방법 1 — 실시간 집계 (GROUP BY) ⭐️⭐️⭐️

```typescript
// 원본 테이블에서 직접 집계
async getDailyStats(from: string, to: string) {
  const { start, end } = kstDayRange(from, to);

  const rows = await this.prisma.$queryRaw<{ date: string; count: number }[]>`
    SELECT
      to_char(created_at AT TIME ZONE 'Asia/Seoul', 'YYYY-MM-DD') AS date,
      COUNT(*)::int AS count
    FROM "user_events"
    WHERE created_at >= ${start} AND created_at < ${end}
    GROUP BY date
    ORDER BY date
  `;

  // 빈 날짜 0으로 채우기
  const map = new Map(rows.map(r => [r.date, r.count]));
  const keys = kstDayKeys(from, to);  // ['2026-08-12', ..., '2026-08-18']
  return keys.map(date => ({ date, count: map.get(date) ?? 0 }));
}
```

```txt
언제 실시간 집계를 쓰는가:
  데이터가 적을 때 (수만 건 이하)
  빠른 구현이 필요할 때
  stat 테이블 별도 관리가 부담스러울 때

GROUP BY + COUNT:
  원본 테이블을 날짜별로 묶어서 카운트
  PostgreSQL: to_char(timestamptz AT TIME ZONE 'Asia/Seoul', 'YYYY-MM-DD')
              UTC로 저장된 값을 KST로 변환 후 날짜 추출
```

---

# 방법 2 — 사전 집계 (Pre-aggregation) ⭐️⭐️⭐️⭐️

## 테이블 설계

```prisma
model AdminDailyStat {
  date    String @id           // "2026-08-12" KST 날짜 키
  visits      Int @default(0)
  signups     Int @default(0)
  logins      Int @default(0)

  @@map("admin_daily_stats")
}

model AdminHourlyStat {
  date    String               // "2026-08-12"
  hour    Int                  // 0~23 (KST 기준)
  visits  Int @default(0)
  signups Int @default(0)

  @@id([date, hour])           // 복합 PK
  @@map("admin_hourly_stats")
}
```

## 이벤트 발생 시 누적

```typescript
type CountField = keyof Pick<AdminDailyStat, 'visits' | 'signups' | 'logins'>;

async countIncrement(field: CountField, now = new Date()): Promise<void> {
  const date = kstDateKey(now);   // "2026-08-12"
  const hour = kstHour(now);      // 0~23

  // 어떤 필드를 증가할지 동적으로 결정 — [field] 계산된 속성명
  const daily = this.prisma.adminDailyStat.upsert({
    where:  { date },
    create: { date, [field]: 1 },           // 없으면 생성
    update: { [field]: { increment: 1 } },  // 있으면 +1
  });

  const hourly =
    field === 'logins'
      ? null                                // logins는 hourly 불필요
      : this.prisma.adminHourlyStat.upsert({
          where:  { date_hour: { date, hour } },
          create: { date, hour, [field]: 1 },
          update: { [field]: { increment: 1 } },
        });

  // null 제거 후 트랜잭션으로 한 번에
  await this.prisma.$transaction(
    [daily, hourly].filter((q) => q !== null),
  );
}
```

```txt
[field]: 1 — 동적 키:
  field = 'visits' → { visits: 1 }
  어떤 컬럼을 올릴지 런타임에 결정

upsert:
  없으면 create, 있으면 update → 중복 없이 누적
  P2002(중복 키) 없음 — upsert가 알아서 처리

$transaction([...].filter(q => q !== null)):
  null 제거 후 남은 쿼리만 묶음
  logins는 hourly null → daily 하나만 실행
  → [[NestJS_Prisma]] $transaction 섹션

언제 서비스 레이어에서 호출하는가:
  사용자가 페이지를 방문할 때 → visits++
  회원가입 완료 시 → signups++
  로그인 성공 시 → logins++
```

---

# 통계 조회 — 차트 데이터 반환 ⭐️⭐️⭐️⭐️

## 일별 통계 (N일 범위)

```typescript
async getDailyStats(from: string, to: string) {
  const rows = await this.prisma.adminDailyStat.findMany({
    where: { date: { gte: from, lte: to } },
    orderBy: { date: 'asc' },
  });

  // 빈 날짜 0으로 채우기 — 차트 x축이 비어있으면 선이 끊김
  const map = new Map(rows.map(r => [r.date, r]));
  const keys = kstDayKeys(from, to);  // ['2026-08-12', ..., '2026-08-18']

  return keys.map(date => ({
    date,
    visits:  map.get(date)?.visits  ?? 0,
    signups: map.get(date)?.signups ?? 0,
    logins:  map.get(date)?.logins  ?? 0,
  }));
}
```

## 시간별 통계 (24시간 버킷)

```typescript
async getHourlyStats(date: string) {
  const rows = await this.prisma.adminHourlyStat.findMany({
    where: { date },
    orderBy: { hour: 'asc' },
  });

  // 0~23 전체 버킷 초기화 후 채우기
  const buckets = Array.from({ length: 24 }, (_, hour) => ({
    hour,
    visits:  0,
    signups: 0,
  }));

  rows.forEach(r => {
    buckets[r.hour].visits  = r.visits;
    buckets[r.hour].signups = r.signups;
  });

  return buckets;
  // → [{ hour: 0, visits: 5, signups: 1 }, ..., { hour: 23, visits: 18, signups: 3 }]
}
```

```txt
빈 날짜·시간을 0으로 채우는 이유:
  Nivo Line 차트는 x축 값이 없으면 그 구간 선이 끊김
  통계가 없는 날 = 0 (방문자가 없는 것) → 0으로 표시해야 자연스러움

Array.from({ length: 24 }, (_, hour) => ({ hour, ... })):
  0~23 버킷을 먼저 만들고
  DB 결과로 채워넣는 패턴
  → [[JS_Array_Methods]] 초기값으로 채우기 섹션
```

---

# 프론트엔드 차트 연결 ⭐️⭐️⭐️

## API 응답 → Nivo 데이터 변환

```typescript
// API 응답
type DailyStat = { date: string; visits: number; signups: number };
type HourlyStat = { hour: number; visits: number; signups: number };

// Nivo Line 데이터로 변환
function toDailyLineData(stats: DailyStat[]) {
  return [
    { id: '방문자', data: stats.map(s => ({ x: s.date, y: s.visits })) },
    { id: '가입자', data: stats.map(s => ({ x: s.date, y: s.signups })) },
  ];
}

// Nivo Bar 데이터 — 변환 없이 바로 사용 가능
// data={stats} keys={['visits', 'signups']} indexBy="date"

// 시간별 → 24시간 Line
function toHourlyLineData(stats: HourlyStat[]) {
  return [
    { id: '방문자', data: stats.map(s => ({ x: `${s.hour}시`, y: s.visits })) },
  ];
}
```

```txt
일별: date "2026-08-12" → x축 문자열 그대로 사용
시간별: hour 0~23 → "0시" ~ "23시" 레이블로 변환

Nivo에서 xScale: { type: 'point' }:
  x값이 이산(discrete) 문자열일 때
  날짜 문자열, 시간 레이블 모두 point 방식

→ [[React_Charts]] Nivo Line 섹션
```

---

# 두 방법 비교 ⭐️⭐️⭐️

```txt
실시간 집계 (GROUP BY):
  ✅ DB 테이블 단순 (stat 테이블 없음)
  ✅ 구현 빠름, 유지보수 쉬움
  ❌ 데이터 많아지면 쿼리 느림
  ❌ 복잡한 집계(여러 테이블 조인)는 쿼리 복잡

사전 집계 (Pre-aggregation):
  ✅ 조회 빠름 — stat 테이블에서 바로 반환
  ✅ 차트 데이터 포맷이 이미 집계돼 있음
  ❌ 이벤트마다 upsert 추가 쿼리
  ❌ stat 테이블 설계·관리 필요

선택 기준:
  트래픽이 적고 빠른 구현 필요 → 실시간 집계
  어드민 대시보드·높은 조회 빈도 → 사전 집계
  보통 사전 집계가 어드민 통계에 더 적합
```