---
aliases:
  - algorithm
  - 알고리즘
  - GCD
  - 서로소
  - 격자 산포 배치
  - 해시 함수
  - 유클리드 알고리즘
  - 황금비(0.618...)
tags:
  - JavaScript
related:
  - "[[00_JS_Ecosystem_HomePage]]"
  - "[[JS_Primitive_Methods]]"
  - "[[JS_Algorithm_Concept]]"
---

# JS_AlgorithmPatterns — 알고리즘 패턴

>[!info]
>기초 개념(자료구조·탐색·Big O) → [[JS_Algorithm_Concept]]
>실무 알고리즘 패턴 모음 — GCD·서로소 step·격자 산포 배치·Map 조회 캐시.
>해시 함수(FNV-1a) → [[JS_Primitive_Methods]] 해시 함수 섹션

---

# 시간 복잡도 (Big O) — 왜 알고리즘을 고민하나 ⭐️⭐️⭐️⭐️⭐️

```txt
Big O = 입력 크기(n)가 커질 때 연산 횟수가 얼마나 늘어나는가
  코드의 "최악의 경우 속도"를 한 글자로 표현하는 방법

n = 다루는 데이터 개수
  배열 길이 100 → n = 100
  배열 길이 10,000 → n = 10,000
```

## 자주 나오는 복잡도

| Big O | 이름 | n=100일 때 연산 수 | 예시 |
|---|---|---|---|
| O(1) | 상수 | 1번 | Map.get(), 배열 인덱스 접근 |
| O(log n) | 로그 | ~7번 | 이진 탐색 |
| O(n) | 선형 | 100번 | 배열 한 번 순회, .find() |
| O(n log n) | 선형로그 | ~700번 | 정렬 (Array.sort) |
| O(n²) | 이차 | 10,000번 | 이중 반복문, 반복 안에서 .find() |

```txt
실무에서 가장 자주 만나는 상황:
  O(n²) → O(n) 으로 줄이기
  → 이중 루프 또는 루프 안의 .find() 를 Map으로 대체
```

## O(n²) → O(n) — 배열 안에서 .find() 제거

```typescript
// ❌ O(n²) — 반복문 안에서 .find() 매번 실행
// reviews 100개 × movies 500개 = 최대 50,000번 비교
reviews.map((review) => {
  const movie = movies.find((m) => m.tmdbId === review.tmdbId); // O(n) 탐색
  return { ...review, title: movie?.title };
});

// ✅ O(n) — Map으로 전처리 후 O(1) 조회
// 전처리 500번 + 조회 100번 = 600번
const titleMap = new Map(movies.map((m) => [m.tmdbId, m.title])); // O(n) 1회
reviews.map((review) => ({
  ...review,
  title: titleMap.get(review.tmdbId), // O(1)
}));
```

```txt
new Map(array.map(item => [key, value])) 구조:
  array.map(item => [key, value])
  → [[1234, 'Inception'], [5678, 'Dune'], ...]  엔트리 배열 생성  O(n)
  → new Map(엔트리 배열)
  → Map { 1234 → 'Inception', 5678 → 'Dune', ... }  해시 테이블 구성

Map.get(key) = O(1) 인 이유:
  내부적으로 해시 테이블(Hash Table) 사용
  key를 해시 함수로 변환 → 버킷(bucket) 위치 직접 계산
  몇 개가 들어있든 항상 1번만 접근
  배열 .find()는 앞에서부터 하나씩 비교 → n번
```

## O(n) 중복 비동기 호출 → Promise 캐싱

```typescript
// ❌ 같은 tmdbId를 여러 번 API 호출
reviews.map(async (review) => {
  const movie = await tmdbService.getMovie(review.tmdbId); // 중복 호출 발생
  return { ...review, title: movie.title };
});

// ✅ Map<key, Promise> — 호출 1번, 결과 여러 번 재사용
const movieCache = new Map<number, ReturnType<TmdbService['getMovieCached']>>();

function getMovieCached(tmdbId: number) {
  if (!movieCache.has(tmdbId)) {
    movieCache.set(tmdbId, tmdbService.getMovieCached(tmdbId)); // Promise 저장
  }
  return movieCache.get(tmdbId)!;
}

await Promise.all(
  reviews.map(async (review) => {
    const movie = await getMovieCached(review.tmdbId); // 같은 id → 동일 Promise
    return { ...review, title: movie.title };
  }),
);
```

```txt
ReturnType<TmdbService['getMovieCached']> 타입 해석:
  TmdbService['getMovieCached'] → 메서드 타입 추출
  ReturnType<...>               → 그 메서드의 반환 타입 추출 (= Promise<Movie>)
  → Promise<Movie> 를 하드코딩하지 않고 메서드 시그니처에서 가져옴
  → 메서드 반환 타입이 바뀌면 자동으로 따라감

Promise 자체를 캐싱하는 이유:
  movieCache.set(tmdbId, Promise)  ← resolve된 값이 아니라 Promise 저장
  두 번째 요청 → 동일 Promise 반환 → API 실제 호출 없음
  await는 이미 resolve된 Promise도 즉시 값 반환
```

## 패턴 선택 기준

```txt
반복 안에 .find() 가 있다
  → O(n²) → Map으로 전처리 후 O(1) 조회로 전환

같은 key로 비동기 API를 여러 번 호출한다
  → Map<key, Promise>로 Promise 캐싱

둘 다 해당한다
  → titleMap (동기 값) + movieCache (비동기) 함께 사용
```

---

# GCD — 최대공약수 ⭐️⭐️⭐️⭐️

```txt
GCD(Greatest Common Divisor) = 두 수의 최대공약수
  두 수가 공통으로 나누어지는 수 중 가장 큰 것

  GCD(12, 8) = 4   (12와 8 모두 4로 나눠짐)
  GCD(7,  5) = 1   (공약수가 1뿐 → 서로소)
  GCD(6,  4) = 2
```

## 유클리드 알고리즘 ⭐️⭐️⭐️

```typescript
function gcd(a: number, b: number): number {
  let x = a;
  let y = b;
  while (y) {         // y가 0이 될 때까지
    const t = y;
    y = x % y;       // x를 y로 나눈 나머지
    x = t;           // x ← y, y ← 나머지
  }
  return x;           // 마지막 y가 0이 되기 전 x = GCD
}

gcd(12, 8)   // 4
gcd(7,  5)   // 1  ← 서로소
gcd(100, 75) // 25
```

```txt
유클리드 알고리즘 원리:
  GCD(a, b) = GCD(b, a % b)
  a를 b로 나눈 나머지가 0이면 b가 GCD

  예: GCD(12, 8)
  x=12, y=8 → y=12%8=4, x=8
  x=8,  y=4 → y=8%4=0,  x=4
  y=0  → GCD = 4  ✅

재귀 버전 (더 짧음):
```

```typescript
const gcd = (a: number, b: number): number => b === 0 ? a : gcd(b, a % b);
```

---

# 서로소 step — 모든 슬롯 방문 보장 ⭐️⭐️⭐️⭐️

```txt
서로소(coprime) = 두 수의 GCD가 1인 것
  GCD(7, 5) = 1  → 7과 5는 서로소

서로소 step이 필요한 이유:
  N개 슬롯을 step씩 건너뛰며 순회할 때
  step이 N과 서로소이면 → 모든 슬롯을 정확히 한 번씩 방문
  step이 N과 공약수 있으면 → 일부 슬롯을 반복 방문, 일부는 건너뜀

  N=6, step=3 → 0, 3, 0, 3, ... (2개 슬롯만 반복 — 나쁨)
  N=6, step=5 → 0, 5, 4, 3, 2, 1 (6개 모두 방문 — GCD(6,5)=1)
```

```typescript
function scatterStep(total: number): number {
  // 황금비(0.6180...)를 곱한 값을 step 초기값으로
  let step = Math.max(1, Math.floor(total * 0.6180339887));
  //                                  ↑ 황금비 ≈ 0.618
  //                                    황금비와 곱하면 분산이 고름

  // total과 서로소가 될 때까지 step을 줄여가며 찾기
  while (step > 1 && gcd(step, total) !== 1) step -= 1;
  return step || 1;
}

scatterStep(6)  // 5 (GCD(6,5)=1)
scatterStep(8)  // 5 (GCD(8,5)=1)
scatterStep(10) // 7 (GCD(10,7)=1)
```

```typescript
// 활용: index를 step씩 건너뛰며 슬롯 배정
const n    = 10;
const step = scatterStep(n);  // 7

for (let i = 0; i < n; i++) {
  const slot = (i * step) % n;
  // i=0 → 0, i=1 → 7, i=2 → 4, i=3 → 1 ...
  // 모든 0~9 슬롯이 겹치지 않고 등장
}
```

```txt
황금비(0.618...)를 쓰는 이유:
  황금비는 어떤 정수로도 잘 나눠지지 않음
  → 가장 "고르게 분산"되는 성질
  황금비 * N ≈ 피보나치 수열 → 자연스러운 배치

피보나치·황금비 분산은 해바라기 씨 배열·나뭇잎 각도에서도 나타남
```

---

# 격자 산포 배치 — ID 해시 기반 안정 레이아웃 ⭐️⭐️⭐️⭐️

```txt
목표:
  N개 아이템을 컨테이너 안에 흩뿌려서 배치
  같은 id → 항상 같은 위치 (새로고침해도 안 움직임)
  id마다 약간씩 다른 색상·각도·딜레이 (생동감)
  겹치지 않게 균등 분산 (서로소 step)
```

```typescript
type ScatterLayout = {
  left:   string;  // "12.5%"
  top:    string;  // "34.2%"
  hue:    string;  // "180"  (CSS hue-rotate 등에 사용)
  delay:  string;  // "0.8s" (애니메이션 딜레이)
  rotate: string;  // "2.3deg"
  scale:  string;  // "1.03"
  z:      number;  // 1~5
};

function capsuleLayout(
  id:     string,
  index:  number,
  total:  number,
  cols:   number,  // 열 수 (크기에 따라 조정)
): ScatterLayout {
  const h = hashId(id);  // id → u32 (항상 같은 값)

  // id 기반 시각 속성 — 같은 id면 항상 같은 값
  const hue    = String(12 + (h % 300));               // 12~311 (300가지 색조)
  const delay  = `${(h % 2600) / 1000}s`;              // 0~2.6s
  const rotate = `${((h % 11) - 5) * 0.55}deg`;       // -2.75~2.75deg
  const scale  = `${0.94 + ((h >>> 8) % 12) / 100}`;  // 0.94~1.05
  const z      = 1 + (h % 5);                          // 1~5

  // 격자 슬롯 계산
  const n    = Math.max(total, 1);
  const rows = Math.max(2, Math.ceil(n / cols));
  const step = scatterStep(n);        // 서로소 step → 고른 분산
  const slot = (index * step) % n;   // 실제 격자 슬롯 번호
  const col  = slot % cols;
  const row  = Math.floor(slot / cols);

  // % 기반 위치 계산 (패딩·볼 크기 고려)
  const ballPct = 15;  // 볼 지름 약 15%
  const padX = 3, padY = 4;
  const usableW = 100 - padX * 2 - ballPct;
  const usableH = 100 - padY * 2 - ballPct;
  const cellW   = usableW / cols;
  const cellH   = usableH / rows;

  // 홀수 행 stagger + id 기반 jitter (작은 무작위 오프셋)
  const stagger = row % 2 === 1 ? cellW * 0.32 : 0;
  const jitterX = ((h % 1000) / 1000 - 0.5) * cellW * 0.75;
  const jitterY = (((h >>> 10) % 1000) / 1000 - 0.5) * cellH * 0.75;

  const rawLeft = padX + col * cellW + cellW * 0.12 + stagger + jitterX;
  const rawTop  = padY + row * cellH + cellH * 0.12 + jitterY;

  return {
    left:   `${Math.min(Math.max(rawLeft, padX), 100 - ballPct - padX)}%`,
    top:    `${Math.min(Math.max(rawTop, padY),  100 - ballPct - padY)}%`,
    hue, delay, rotate, scale, z,
  };
}
```

## 각 계산의 역할

```txt
hashId(id):
  id → 항상 같은 u32 숫자
  "같은 id면 항상 같은 위치"를 보장하는 핵심
  → [[JS_Primitive_Methods]] FNV-1a 해시 섹션

scatterStep(n):
  서로소 step → index 순서대로 슬롯을 고르게 분산
  item 0은 앞쪽, item 1은 뒤쪽 ... 이 아니라
  item 0, 7, 4, 1, 8, 5, 2, ... 식으로 섞임

jitter (지터):
  격자 칸 안에서 id 기반 소량 오프셋
  완벽한 격자 느낌을 없애고 자연스럽게 흩뿌린 느낌
  h % 1000 → 0~999를 /1000 → 0~0.999 → -0.5~0.499 오프셋

stagger (엇갈림):
  홀수 행을 반 칸씩 오른쪽으로 밀기
  벌집 구조처럼 빈 공간을 줄이는 효과

Math.min(Math.max(v, min), max):
  "v가 min~max 범위를 벗어나지 않도록 클램핑"
  clamp(v, min, max) 패턴
```

## 클램프 패턴

```typescript
// clamp — 범위 제한
const clamp = (v: number, min: number, max: number) =>
  Math.min(Math.max(v, min), max);

clamp(150, 0, 100)  // 100 (위 초과)
clamp(-10, 0, 100)  // 0   (아래 초과)
clamp(50,  0, 100)  // 50  (범위 안)

// 위치 계산 후 컨테이너를 벗어나지 않도록
const left = clamp(rawLeft, padX, 100 - ballPct - padX);
```

---

# 정리 — 패턴 한눈에

```txt
격자 산포 배치 흐름:

  1. hashId(id)  → 아이템마다 고유한 "시드" 숫자
  2. scatterStep → total과 서로소인 step
  3. slot = (index * step) % total → 고른 격자 슬롯
  4. col = slot % cols, row = slot / cols → 격자 좌표
  5. cellW/cellH 계산 → 패딩·볼 크기 고려
  6. stagger + jitter → 생동감 있는 오프셋
  7. clamp → 컨테이너 벗어남 방지

언제 이 패턴이 필요한가:
  N개 아이템을 컨테이너 안에 "자연스럽게" 배치
  새로고침/업데이트해도 자리가 바뀌면 안 됨 (id 기반 고정)
  완벽한 격자가 아닌 약간 흩뿌린 느낌
  ex) 리뷰 볼, 스티커, 태그 클라우드
```