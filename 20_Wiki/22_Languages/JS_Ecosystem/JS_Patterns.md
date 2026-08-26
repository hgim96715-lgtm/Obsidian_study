---
aliases:
  - 내부 함수 추출
  - 조기반환
  - 함수 설계 패턴
  - 화살표 함수 암시적 반환
  - Early Return
  - fallback(폴백)
  - Fire-and-Forget
  - force=false
tags:
  - JavaScript
  - React
related:
  - "[[00_JS_Ecosystem_HomePage]]"
  - "[[JS_Operators]]"
  - "[[JS_Promise]]"
  - "[[NextJS_WebSocket]]"
  - "[[React_AsyncUI]]"
  - "[[JS_Algorithm_Concept]]"
  - "[[JS_AlgorithmPatterns]]"
---
# JS_Patterns — JS 코드 패턴

>[!info]
>JS 코드 패턴 모음 — 함수 설계·async·폴백·Map 캐시 등 실무에서 반복되는 패턴.
>자료구조·탐색 기초 → [[JS_Algorithm_Concept]]
>옵션 객체·조기 반환·async 래퍼·화살표 함수 암시적 반환(`=>{}` vs `=>`) 등.
>중괄호 없으면 자동 return, 중괄호 있으면 return 키워드 필수 — map 콜백에서 자주 실수.
>**폴백(fallback)** = 주요 방법이 실패하거나 없을 때 대신 시도하는 것 — `??` 기본값·캐시 미스·에러 시 백업 서비스 등.

---

# 옵션 객체 패턴 — `{ force = false }` ⭐️⭐️⭐️⭐️

```txt
함수 인자가 많아지거나 일부가 선택적일 때
boolean 인자를 여러 개 나열하는 대신 객체 하나로 묶는 패턴
```

```typescript
// ❌ 인자가 많아지면 순서 기억이 어려움
touchLastActiveAt(userId, role, true, false, 'admin');

// ✅ 옵션 객체 — 이름으로 의미가 명확
touchLastActiveAt(userId, role, { force: true });
```

## TypeScript 타입 정의

```typescript
interface TouchOptions {
  force?:  boolean;
  silent?: boolean;
}

async function touchLastActiveAt(
  userId:  number,
  role:    string,
  options: TouchOptions = {},  // 통째로 안 넘겨도 됨 (기본값 빈 객체)
): Promise<void> {
  const { force = false, silent = false } = options;

  if (!force && checkRecentlyUpdated()) return;  // force: true면 이 early return 건너뜀
  await update();
}
```

```txt
options = {} 기본값의 의미:
  호출하는 쪽이 세 번째 인자를 아예 안 넘겨도
  내부에서 {}(빈 객체)를 받은 것처럼 동작
  → { force = false, silent = false }가 모두 기본값으로 처리됨

  options?: TouchOptions로 선언하면:
  → options가 undefined일 수 있어서 구조분해 전에 ?? {} 처리 필요 → 번거로움
  → = {} 기본값이 더 깔끔
```

## 호출 방법


```typescript
await touchLastActiveAt(userId, role);                          // 기본값
await touchLastActiveAt(userId, role, { force: true });         // 일부만
await touchLastActiveAt(userId, role, { force: true, silent: true }); // 여러 개
```

## 파라미터에서 바로 구조분해 — 더 짧게

```typescript
async function touchLastActiveAt(
  userId: number,
  role:   string,
  { force = false }: TouchOptions = {},  // 파라미터에서 바로 구조분해
) {
  if (!force && checkRecentlyUpdated()) return;
  await update();
}
```

## force 플래그가 자주 쓰이는 패턴 ⭐️⭐️⭐️⭐️

```txt
force 플래그란:
  force = false (기본값) = "평소대로 해줘"
    → 캐시가 있으면 캐시 사용
    → 이미 처리된 것은 스킵
    → 조건이 맞으면 early return

  force = true = "강제로 새로 해줘, 평소 조건 무시해"
    → 캐시가 있어도 무시하고 새로 가져옴
    → 이미 처리된 것도 다시 실행
    → early return 건너뛰고 끝까지 실행

  이름이 force(강제)인 이유:
    "원래 건너뛸 상황인데 강제로 실행시킨다"는 의미
    기본값이 false인 이유: 보통은 강제할 필요 없음
    true는 예외 상황(수동 갱신, 버그 수정, 배치 재처리)에서만 사용
```

```typescript
// opts?: { force?: boolean } 패턴 분해
async function getMovieCached(
  movieId: number,
  opts?:   { force?: boolean },
  //  ↑ opts 자체가 optional (없어도 됨)
  //              ↑ force도 optional (있어도 없어도 됨)
) {
  const cached = await db.movie.findUnique({ where: { id: movieId } });

  if (
    !opts?.force &&     // ① force가 아니면 캐시 사용 조건 체크
    cached &&           // ② 캐시가 있고
    cached.isComplete   // ③ 데이터가 완전하면
  ) {
    return cached;      // 캐시 반환 (빠름)
  }

  // force: true이거나, 캐시 없거나, 데이터 불완전 → 새로 가져옴
  const fresh = await fetchFromApi(movieId);
  await db.movie.upsert({ ... });  // 캐시 저장
  return fresh;
}
```

```txt
!opts?.force 읽는 법:

  opts?.force:
    opts가 없으면 undefined → false로 해석
    opts = {} 이면 force는 undefined → false로 해석
    opts = { force: true } 이면 true

  !opts?.force:
    "force가 true가 아닐 때" = "강제 새로고침이 아닐 때"
    → true(캐시 써도 됨)
    → false(강제로 새로 가져와야 함)

조건 전체 뜻:
  if (!opts?.force && cached && cached.isComplete)
     ↑ 강제 아님      ↑ 캐시 있음 ↑ 데이터 완전함
  → 세 조건 모두 만족 시 캐시 반환
  → 하나라도 false면 새로 가져옴

  → force: true 설정 시 캐시가 있어도 무조건 새로 가져옴
    = "캐시를 강제로 무효화"
```

```typescript
// 호출 방법
getMovieCached(123)                // 기본 — 캐시 있으면 사용
getMovieCached(123, { force: true }) // 강제 — 캐시 무시하고 새로 가져옴

// 실전 예: 관리자가 수동으로 데이터 갱신 버튼 클릭
await getMovieCached(movieId, { force: true });

// 실전 예: 배치에서 특정 조건의 데이터만 강제 갱신
if (needsRefresh(cached)) {
  await getMovieCached(movieId, { force: true });
}
```

## `Partial<T>` — 옵션 객체 타입 유틸리티 ⭐️⭐️

```typescript
interface UserUpdateData {
  name:  string;
  email: string;
  image: string;
}

// Partial<T> = 모든 필드를 optional로 만듦
async function updateUser(userId: number, data: Partial<UserUpdateData>) {
  // name만, email만, 셋 다 — 어떤 조합이든 타입 통과
}

updateUser(1, { name: '새이름' });  // ✅
```


```txt
옵션 객체는 보통 모든 필드가 선택적
→ interface에 ? 전부 붙이거나 Partial<T>를 쓰거나
→ Partial<T> 상세 → [[TS_Utility_Types]]
```

## 콜백을 opts 안에 넣기 — 진행률·훅 주입 ⭐️⭐️⭐️⭐️

```typescript
// 기본 파라미터는 앞에, 선택적 콜백은 opts 객체 안에
async function seedPool(
  filters: Record<string, string> = {},
  pages   = 5,
  opts?:  { onPageDone?: (page: number) => void },
  //  ↑ 전체 opts가 optional
  //                ↑ 콜백 자체도 optional
) {
  for (let page = 1; page <= pages; page++) {
    await fetchAndSave(filters, page);

    opts?.onPageDone?.(page);
    // opts가 없어도 → opts?.onPageDone? → undefined → 안전
    // opts는 있지만 콜백 없어도 → onPageDone? → undefined → 안전
    // 둘 다 있어야 호출됨
  }
}

// 콜백 없이 호출
await seedPool({ genre: 'action' }, 10);

// 콜백 포함
await seedPool({ genre: 'action' }, 10, {
  onPageDone: (page) => {
    console.log(`${page}페이지 완료`);
    setProgress(page);
  },
});
```

```typescript
// 여러 콜백을 opts에 담는 경우
type SeedOpts = {
  onPageDone?:  (page: number) => void;
  onError?:     (err: Error, page: number) => void;
  onComplete?:  (total: number) => void;
};

async function seedPool(
  filters: Record<string, string> = {},
  pages   = 5,
  opts?:  SeedOpts,
) {
  let total = 0;
  for (let page = 1; page <= pages; page++) {
    try {
      const count = await fetchAndSave(filters, page);
      total += count;
      opts?.onPageDone?.(page);
    } catch (err) {
      opts?.onError?.(err as Error, page);
    }
  }
  opts?.onComplete?.(total);
}
```

```txt
왜 직접 파라미터 대신 opts 객체에 넣는가:

  직접 넣으면:
    function seedPool(filters, pages, onPageDone?, onError?, onComplete?)
    → 파라미터가 많아질수록 순서를 기억해야 함
    → onError만 넣고 싶을 때 onPageDone 자리에 undefined 필요

  opts 객체에 넣으면:
    seedPool(filters, pages, { onError: handler })
    → 필요한 것만 이름으로 전달
    → 순서 무관, 나중에 콜백 추가해도 기존 호출 코드 안 바뀜

opts?.callback?.() — 옵셔널 체이닝 두 번:
  opts?       → opts가 undefined이면 여기서 멈춤
  .onPageDone?→ 콜백이 undefined이면 여기서 멈춤
  ()          → 둘 다 있을 때만 호출

콜백 타입 읽는 법:
  (page: number) => void
  ↑ 인자 이름: 타입   ↑ 반환 없음 (결과 무시)
  → [[React_Types]] React.Dispatch와 비교
```
---

# 조기 반환 (Early Return) ⭐️⭐️⭐️

```typescript
// ❌ 중첩이 깊어짐
function processUser(user: User | null) {
  if (user) {
    if (user.isActive) {
      if (user.role === 'admin') {
        // 실제 로직
      }
    }
  }
}

// ✅ early return으로 평탄하게
function processUser(user: User | null) {
  if (!user) return;
  if (!user.isActive) return;
  if (user.role !== 'admin') return;

  // 실제 로직 — 조건이 모두 통과된 상태에서 작성
}
```

```txt
early return의 장점:
  실패 케이스를 함수 위에서 먼저 처리 → 아래로 내려올수록 조건이 보장됨
  중첩 if를 없애서 인지 부하 감소
  → useEffect 안의 조건 가드 패턴도 같은 원리
    if (!user || !roomId) return;
```

---

# async 래퍼 패턴 ⭐️⭐️⭐️

```typescript
// 여러 이벤트 핸들러에서 같은 try/catch/finally 구조가 반복될 때
const runAction = async (fn: () => Promise<unknown>) => {
  setActing(true);
  setError('');
  try {
    await fn();
    await reload();
  } catch (err) {
    setError(err instanceof Error ? err.message : '요청에 실패했어요.');
  } finally {
    setActing(false);
  }
};

// 사용 — 함수를 인자로 넘김
void runAction(() => deleteComment(id));
void runAction(() => likeComment(id));
```

```txt
fn: () => Promise<unknown>:
  반환 타입을 unknown으로 선언 — 어떤 API 함수든 래핑 가능
  내부에서 결과를 쓰지 않을 때는 unknown이 any보다 안전

runAction이 적합한 경우:
  공통 후처리(reload)가 모든 액션에 동일
  에러 메시지 포맷이 전부 같을 때

개별 핸들러가 더 적합한 경우:
  액션마다 다른 후처리 (삭제 성공 후 편집 모드 닫기 등)
  대상별 pending id를 각자 관리해야 할 때
  → [[React_AsyncUI]] 이벤트 핸들러 섹션 참고
```

---
# Fire-and-Forget — 쏘고 잊기 ⭐️⭐️⭐️⭐️

```txt
fire-and-forget = 비동기 작업을 시작하고 결과를 기다리지 않는 패턴

  일반 await:
    결과를 받아야 다음으로 진행
    작업이 끝날 때까지 응답이 지연됨

  fire-and-forget:
    작업을 시작만 하고 즉시 다음으로 진행
    작업은 백그라운드에서 계속 실행됨
    응답 속도가 빨라짐

  이름 유래:
    총을 쏘고(fire) 탄이 어디 떨어지는지 신경 쓰지 않는(forget) 것
    "일단 실행시켜놓고 결과는 나중에 (또는 신경 안 씀)"
```

## 기본 구조

```typescript
// await 있음 — 결과를 기다림
const enriched = await this.enrichIfNeeded(id, title, overview);
return { ...movie, ...enriched };  // enriched가 완료된 후 반환

// fire-and-forget — 결과를 기다리지 않음
void this.enrichIfNeeded(id, title, overview)
  .catch(err => this.logger.warn(`enrich 실패: ${err.message}`));
return movie;  // enrichIfNeeded가 끝나기 전에 즉시 반환
```


```txt
void 키워드:
  "이 Promise의 결과를 의도적으로 무시한다"고 명시
  void 없으면 TypeScript/ESLint가 "await 안 했음" 경고
  → void = "알고 있어, 일부러 기다리지 않는 거야"

.catch() 가 반드시 필요한 이유:
  await 없이 Promise를 실행하면 에러가 어디에도 잡히지 않음
  → UnhandledPromiseRejection → 앱 크래시 가능
  .catch()로 에러를 잡아서 로그에 기록
  → 에러가 나도 응답에 영향 없음
```

## 실전 — fromPool 캐시 hit 후 백그라운드 보강

```typescript
private async fromPool(row: MoviePool): Promise<GachaMovie> {
  // 캐시에서 즉시 응답할 데이터 조립
  const movie: GachaMovie = {
    id:           row.tmdbId,
    title:        row.title,
    overview:     row.overview,
    poster_path:  row.posterPath,
    release_date: row.releaseDate,
    director:     row.director,
  };

  // 번역·보강은 백그라운드에서 — 응답은 지금 바로
  void this.enrichIfNeeded(
    row.tmdbId,
    movie.title,
    movie.overview ?? '',
    movie.release_date ?? '',
  ).catch(err =>
    this.logger.warn(`background enrich 실패: ${(err as Error).message}`),
  );

  return movie;  // enrichIfNeeded 완료 전에 즉시 반환
}
```


```txt
흐름:
  캐시 hit → movie 조립 (빠름)
  enrichIfNeeded 시작 (백그라운드)  ← 기다리지 않음
  즉시 movie 반환 → 클라이언트가 빠르게 받음
  (시간이 지나면 enrichIfNeeded 완료 → DB 업데이트)

왜 이렇게 하는가:
  enrichIfNeeded = 번역·AI 호출 등 느린 작업
  이걸 기다리면 응답이 느려짐
  캐시에 이미 기본 데이터는 있으니 먼저 반환
  보강 데이터는 다음 캐시 hit 때 사용됨
```

## 언제 fire-and-forget 쓰는가


```typescript
// ✅ 적합한 경우

// ① 결과가 현재 응답에 필요 없을 때
void this.notificationService.send(userId, '가입 환영합니다');
return { success: true };

// ② 중요하지 않은 사이드 이펙트 (통계, 로그)
void this.statsService.increment('pageView');
return pageData;

// ③ 캐시를 백그라운드에서 갱신
void this.cacheService.refresh(key)
  .catch(err => this.logger.warn('캐시 갱신 실패'));
return cachedData;

// ❌ 적합하지 않은 경우

// 결과가 응답에 필요할 때
void this.getUserProfile(id);  // ← 이 결과가 없으면 응답 불완전
return { user: ??? };

// 실패 시 반드시 알아야 할 때 (결제, 주문)
void this.paymentService.charge(amount);  // ← 결제 실패를 모름
return { orderId };  // ← 사실 결제 안 됐을 수 있음
```


```txt
fire-and-forget 쓰면 안 되는 경우:
  결과 데이터가 현재 응답에 필요한 경우
  실패 시 사용자에게 알려야 하는 중요한 작업
  다음 작업이 이 작업의 완료에 의존하는 경우

  → 중요한 작업은 반드시 await
  → 실패해도 괜찮은 부가 작업에만 fire-and-forget
```


## .catch(() => undefined) — 에러를 조용히 삼키기 ⭐️⭐️⭐️⭐️

```typescript
// 방문 기록 — 실패해도 사용자 경험에 영향 없음
if (accessToken) {
  void recordLobbyVisitRequest(accessToken).catch(() => undefined);
}
```

```txt
.catch(() => undefined) 분해:
  .catch(fn)       → Promise가 reject되면 fn 실행
  () => undefined  → 에러를 받아서 undefined를 반환 (아무것도 안 함)
  결과: reject → undefined (정상 resolve처럼 처리됨)
  에러 메시지도 없고, 로그도 없고, 아무 일도 없었던 것처럼

void + .catch(() => undefined) 조합:
  void   → "이 Promise 결과 안 기다림" (ESLint no-floating-promises 만족)
  .catch → "에러도 무시함"
  → 완전히 선택적인 사이드 이펙트 처리
```

```typescript
// .catch() 변형 3가지 비교

// ① 에러 로깅 — 실패는 알고 싶지만 응답엔 영향 없을 때
void someTask().catch(err => this.logger.warn(`실패: ${err.message}`));

// ② 조용히 무시 — 실패해도 완전히 상관없을 때
void someTask().catch(() => undefined);

// ③ 기본값으로 대체 — reject 시 fallback 값이 필요할 때
const result = await someTask().catch(() => null);
if (!result) { /* fallback 처리 */ }
```

```txt
언제 .catch(() => undefined) 쓰나:
  실패해도 UX에 아무 영향 없는 순수 사이드 이펙트
  → 방문 기록, 조회수 카운팅, 추천 알고리즘 피드백
  → 실패 로그조차 노이즈가 될 때

쓰면 안 되는 경우:
  실패 원인을 추적해야 할 때 → ① 로깅 버전 사용
  실패 시 fallback 처리가 필요할 때 → ③ await + catch 사용
  중요한 작업 (결제, 인증, 데이터 저장) → 반드시 await + 에러 핸들링
```

---

# 내부 함수 추출 — 즉시 실행 vs 참조로 전달 ⭐️⭐️⭐️⭐️

```txt
같은 코드를 두 가지 시점에 실행해야 할 때
내부 함수로 추출하면 중복 없이 두 경우 모두 처리 가능
```

```typescript
// ❌ 내부 함수 없이 — 같은 코드가 두 번 반복
if (isReady) {
  doWork(data);                   // 즉시 실행
} else {
  waitForReady.once('ready', () => {
    doWork(data);                 // 나중에 실행 — 완전히 같은 코드 반복
  });
}

// ✅ 내부 함수로 추출 — 한 곳에서 정의, 두 곳에서 사용
const execute = () => {
  doWork(data);
};

if (isReady) execute();           // 즉시 실행
else waitForReady.once('ready', execute);  // 나중에 참조로 전달
```

```txt
execute()   → 소괄호 있음 → "지금 즉시 실행"
execute     → 소괄호 없음 → "나중에 실행할 함수를 참조로 넘김"

  once('ready', execute())  — ❌ once 호출 시점에 즉시 실행
  once('ready', execute)    — ✅ ready 이벤트가 오면 그때 execute 실행

이 구분을 모르면:
  once(event, fn()) — fn이 즉시 실행되고 그 반환값(보통 undefined)이 콜백으로 등록
  → 이벤트가 와도 아무 일도 안 일어남
```

## 실전 예 — WebSocket 연결 대기

```typescript
// 연결이 됐으면 지금 emit, 아직이면 연결 완료 후 emit
const doEmit = () => {
  socket.emit('featureA:join', { resourceId }, callback);
};

if (socket.connected) doEmit();            // 즉시 실행
else socket.once('connect', doEmit);       // 참조로 전달 → 나중에 실행
```

```txt
자세한 설명 → [[NextJS_WebSocket]] acknowledgement 섹션
```

## 실전 예 — 조건에 따라 다른 시점에 같은 작업

```typescript
// 초기화가 됐으면 바로, 아직이면 초기화 완료 후 실행
const initialize = () => {
  setupPlayer(videoId);
};

if (isApiReady) {
  initialize();                            // 즉시
} else {
  window.onApiReady = initialize;          // API 준비되면 실행 (참조)
}
```

```txt
함수가 "값"처럼 쓰이는 것:
  JavaScript에서 함수는 일급 객체 — 변수에 담거나, 인자로 넘기거나, 반환할 수 있음
  execute 처럼 이름을 붙여두면 여러 곳에서 참조로 전달 가능
  → [[JS_Promise]] 비동기 흐름과 같이 자주 나오는 패턴
```

---

# 화살표 함수 암시적 반환 — 중괄호 유무 ⭐️⭐️⭐️⭐️

```typescript
// 중괄호 없음 — 암시적 반환 (expression이 자동으로 return됨)
const double = (x: number) => x * 2;
//                            ↑ 자동으로 return

// 중괄호 있음 — 명시적 반환 (return 키워드 필수)
const double = (x: number) => {
  return x * 2;  // return 없으면 undefined 반환
};

// 중괄호 있는데 return 빠진 경우 ❌
const double = (x: number) => {
  x * 2;  // return 없음 → undefined 반환 (계산 결과가 버려짐)
};
```

```txt
중괄호의 의미:
  {} 없음 → "=> 뒤의 식(expression) 하나를 바로 반환"
  {} 있음 → "함수 본문 블록 시작, return을 직접 써야 값 반환"

  이 차이가 map, filter 등 콜백에서 가장 자주 문제가 됨
```

## map에서의 함정 ⭐️⭐️⭐️⭐️

```typescript
// ✅ 중괄호 없음 — upsert Promise가 자동 반환됨
await this.prisma.$transaction(
  rows.map(({ metric, count }) =>
    this.prisma.snapshot.upsert({ ... })  // 자동으로 return
  )
);
// $transaction에 Promise[]가 전달됨

// ❌ 중괄호 있는데 return 없음
await this.prisma.$transaction(
  rows.map(({ metric, count }) => {
    this.prisma.snapshot.upsert({ ... })  // return 없음 → undefined
  })
);
// $transaction에 undefined[]가 전달됨 → 오류!
```

```txt
이 실수가 발생하는 이유:
  "중괄호 = 함수 본문"이라는 건 알지만
  "중괄호 없음 = 자동 return"이라는 것을 잊어버림

  중괄호를 쓰고 싶다면 반드시 return 추가:
  rows.map(({ metric, count }) => {
    return this.prisma.snapshot.upsert({ ... });
  })
```

## 객체를 반환할 때 추가 주의 ⭐️⭐️⭐️

```typescript
// ❌ 객체 리터럴의 {}가 함수 블록으로 해석됨
const toObj = (x: number) => { value: x };
//                             ↑ 함수 블록으로 해석 → undefined 반환

// ✅ 객체를 암시적 반환하려면 소괄호로 감싸기
const toObj = (x: number) => ({ value: x });
//                             ↑ 객체 리터럴임을 명시

// 실전
items.map(item => ({
  id:    item.id,
  label: item.name,
}));
```

```txt
({ ... }) 패턴:
  소괄호로 감싸면 "이건 블록이 아니라 객체야"라고 JS에게 알림
  map, reduce에서 객체를 반환할 때 필수
```

## 한눈에 비교

```typescript
// ① 암시적 반환 — 단일 식
x => x * 2
x => fetch(url)                   // Promise 반환
x => ({ id: x.id, name: x.name }) // 객체 반환 (소괄호 필요)

// ② 명시적 반환 — 블록 본문
x => {
  const result = x * 2;
  return result;     // ← 필수
}

// ③ 흔한 실수 — return 없는 블록
x => {
  x * 2;             // ← 계산은 하지만 반환 안 함 → undefined
}
```

---
# 폴백(Fallback) 패턴 ⭐️⭐️⭐️⭐️

```txt
폴백(fallback) = 주요 방법이 실패하거나 없을 때 대신 시도하는 것

  1차 시도 → 실패 or 없음 → 폴백(대안) → 결과

코딩에서 가장 자주 나오는 패턴 중 하나
```

## 종류별 폴백 패턴

```typescript
// ① 기본값 폴백 — 값이 없으면 대체값
const name = user?.name ?? '익명';
//                         ↑ null/undefined이면 '익명'

const port = process.env.PORT ?? '3030';

// ② 캐시 폴백 — DB에 없으면 외부 API
const cached = await db.findUnique({ where: { id } });
if (cached) return cached;   // 캐시 히트
const fresh = await externalApi.fetch(id);  // 캐시 미스 → 폴백
await db.create({ data: fresh });
return fresh;

// ③ 배열 폴백 — 빈 배열이면 다른 방법
const results = await search(query);
if (results.length > 0) return results;
// 결과 없음 → 폴백: 유사어로 재검색
const fallbacks = generateFallbacks(query);
for (const q of fallbacks) {
  const r = await search(q);
  if (r.length > 0) return r;
}
return [];

// ④ 에러 폴백 — 실패하면 다른 방법
try {
  return await primaryService.fetch(id);
} catch {
  return await fallbackService.fetch(id);  // 1차 실패 → 폴백 서비스
}

// ⑤ 설정값 폴백 — 단계적으로 확인
function getConfig(key: string): string {
  return (
    process.env[key]           ??  // 환경변수 → 없으면
    configFile[key]            ??  // 설정 파일 → 없으면
    defaultConfig[key]         ??  // 기본값 → 없으면
    ''                             // 빈 문자열
  );
}
```

```txt
폴백이 나오는 곳:
  API 검색  → 결과 없음 → 다른 쿼리로 재검색
  캐시      → 미스 → 원본 API 호출
  서비스    → 실패 → 백업 서비스
  인증      → 토큰 만료 → refresh → 재로그인
  폰트      → 커스텀 폰트 없음 → 시스템 폰트 (CSS fallback font)
  UI 이미지 → 로드 실패 → 기본 이미지 (onError)

  ?? (nullish coalescing):
  JavaScript에서 가장 단순한 폴백 표현
  left ?? right = left가 null/undefined이면 right

  || (OR) vs ??:
  || = 0, '', false 도 폴백으로 넘어감 (falsy 전체)
  ?? = null/undefined 만 폴백으로 넘어감
  → 0이나 빈 문자열이 유효한 값이면 ?? 사용
```

## 실전 예 — 이미지 로드 폴백

```typescript
// React — 이미지 로드 실패 시 기본 이미지로
<img
  src={user.avatarUrl}
  onError={(e) => {
    e.currentTarget.src = '/default-avatar.png';  // 폴백 이미지
  }}
  alt={user.name}
/>
```

## 실전 예 — 인증 폴백

```typescript
// 토큰 만료 → refresh → 재시도 → 실패 시 로그아웃
async function fetchWithAuth(url: string) {
  let res = await fetch(url, { headers: bearerHeader(accessToken) });

  if (res.status === 401) {
    // 1차 폴백: 토큰 갱신 시도
    const newToken = await refreshToken();
    if (newToken) {
      res = await fetch(url, { headers: bearerHeader(newToken) });
    }
  }

  if (res.status === 401) {
    // 2차 폴백: 갱신도 실패 → 로그아웃
    clearSession();
    router.push('/login');
    throw new Error('인증 만료');
  }

  return res.json();
}
```
---

# Map 조회 캐시 패턴 — 중복 조회 제거 ⭐️⭐️⭐️⭐️

## 배경 — 왜 Map인가

```txt
배열에서 특정 항목을 반복 조회할 때:

  ❌ 안티 패턴 — 매번 O(n) 선형 탐색
  reviews.map(r => {
    const movie = cachedMovies.find(m => m.tmdbId === r.tmdbId); // 매번 전체 탐색
    return { ...r, title: movie?.title };
  });
  → reviews 100개 × cachedMovies 500개 = 최대 50,000번 비교

  ✅ 패턴 — O(n) 전처리 → O(1) 조회
  const titleMap = new Map(cachedMovies.map(m => [m.tmdbId, m.title]));
  reviews.map(r => ({ ...r, title: titleMap.get(r.tmdbId) }));
  → 전처리 500번 + 조회 100번 = 600번
```

## 패턴 1 — Array → Map 변환 (동기 값 조회)

```typescript
// titleMap: 로비 통계용
// moviePool에서 영화 제목을 한 번에 가져온 뒤 tmdbId → title 로 빠르게 찾음
// TMDB 재조회·provider 조회를 줄이는 역할

const titleMap = new Map(
  cachedMovies.map((movie) => [movie.tmdbId, movie.title]),
);

// 사용
const title = titleMap.get(tmdbId);  // O(1)
```

```txt
new Map([ [key, value], [key, value], ... ]) 구조
  cachedMovies.map(movie => [movie.tmdbId, movie.title])
  → [ [1234, 'Inception'], [5678, 'Dune'], ... ]  (엔트리 배열)
  → Map { 1234 → 'Inception', 5678 → 'Dune', ... }

언제 쓰나:
  이미 메모리에 있는 배열을 여러 번 key로 조회할 때
  DB/API 재호출 없이 인메모리 룩업 테이블 구성
  반복문 안에서 .find() 대신
```

## 패턴 2 — Map<key, Promise> (비동기 중복 호출 방지)

```typescript
// movieCache: 후기 목록용
// 여러 후기가 같은 영화를 가리킬 때 getMovieCached() 호출을 재사용
// 같은 요청 안에서 중복 조회를 막는 역할

const movieCache = new Map<
  number,
  ReturnType<TmdbService['getMovieCached']>  // = Promise<Movie>
>();

function getMovieCached(tmdbId: number) {
  if (!movieCache.has(tmdbId)) {
    // 처음 요청 → Promise를 Map에 저장
    movieCache.set(tmdbId, tmdbService.getMovieCached(tmdbId));
  }
  return movieCache.get(tmdbId)!;  // 이미 있으면 같은 Promise 반환
}

// 병렬로 여러 후기 처리
const results = await Promise.all(
  reviews.map(async (review) => {
    const movie = await getMovieCached(review.tmdbId);  // 같은 tmdbId면 재사용
    return { ...review, title: movie.title };
  }),
);
```

```txt
핵심: Promise 자체를 캐싱
  movieCache.set(tmdbId, Promise)   ← API 호출 결과가 아니라 Promise를 저장
  같은 tmdbId 두 번째 요청 → Map.has() true → 동일 Promise 반환
  → API는 한 번만 실제로 호출됨 (Promise.all이 resolve를 기다림)

Map<key, ReturnType<Service['method']>> 타입 해석:
  ReturnType<TmdbService['getMovieCached']>
  = TmdbService의 getMovieCached 메서드의 반환 타입
  = Promise<Movie> (제네릭 없이 메서드 시그니처에서 추출)
  → 하드코딩 없이 타입 안전하게 캐시 Map 선언

스코프:
  요청 단위 인메모리 캐시 (함수/요청 종료 시 GC)
  Redis 같은 외부 캐시와 달리 영속성 없음
  같은 API 요청 처리 안에서만 유효
```

## 두 패턴 비교

| | titleMap (동기) | movieCache (비동기) |
|---|---|---|
| 값 타입 | `Map<number, string>` | `Map<number, Promise<Movie>>` |
| 전처리 | `new Map(array.map(...))` | 첫 접근 시 lazy 등록 |
| 재사용 | 인메모리 룩업 | Promise 재사용 |
| 목적 | DB/API 조회 없이 O(1) 찾기 | 동일 키 중복 호출 방지 |
| 스코프 | 함수 내 | 함수/요청 내 |

```txt
함께 쓰는 경우:
  titleMap → DB에서 이미 가져온 데이터를 빠르게 찾을 때
  movieCache → 필요할 때마다 외부 API를 호출하되 중복은 막을 때

  로딩이 느릴 때 체크포인트:
  1. 배열 반복 안에서 .find() 쓰고 있나? → titleMap으로 전환
  2. 같은 tmdbId를 여러 번 API 호출하나? → movieCache로 전환
```
