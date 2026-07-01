---
aliases:
  - throttling
  - fire-and-forget
  - rate limiting
tags:
  - NestJS
related:
  - "[[00_NestJS_Ecosystem_HomePage]]"
  - "[[JS_Map_Set]]"
  - "[[JS_Date]]"
  - "[[JS_Promise]]"
  - "[[NestJS_Prisma]]"
  - "[[React_useMemo_useCallback_useEffect]]"
---

# NestJS_Throttle — 스로틀링 패턴 & fire-and-forget

> [!info] 
> 스로틀링(throttling)은 "일정 시간 안에는 한 번만 실행"하는 패턴이다. API 요청이 올 때마다 DB를 업데이트하면 요청 수만큼 UPDATE가 발생하므로, 메모리의 Map으로 마지막 실행 시각을 기억해두고 일정 시간 이내이면 건너뛰는 방식으로 구현한다.

---

# 스로틀링이란 — 디바운싱과의 차이 ⭐️⭐️⭐️⭐️

```txt
스로틀링(throttling): "N ms 안에 한 번만 실행" — 처음 호출은 즉시 실행, 이후 N ms 동안 추가 호출은 무시
  12:00:00 요청 → 실행
  12:00:10 요청 → 무시 (60초 미만)
  12:00:30 요청 → 무시
  12:01:01 요청 → 실행 (60초 경과)

디바운싱(debouncing): "마지막 호출 이후 N ms 동안 추가 호출 없으면 실행" — 연속 호출 중에는 계속 미룸
  12:00:00 요청 → 60초 대기 시작
  12:00:10 요청 → 타이머 리셋, 다시 60초 대기
  12:01:10 요청 → 타이머 리셋, 다시 60초 대기
  12:02:10 이후 요청 없음 → 그제서야 실행

→ "최대 N ms마다 한 번씩 실행" = 스로틀링
  "마지막 호출 이후 잠잠해지면 실행" = 디바운싱

사용 예:
  스로틀링 → API 호출마다 lastActiveAt 갱신, scroll 이벤트 처리, 실시간 저장
  디바운싱 → 검색창 입력 완료 후 API 호출, 창 크기 조정 완료 후 재렌더
```

---

# Map + 타임스탬프로 스로틀링 구현 ⭐️⭐️⭐️⭐️

```typescript
const LAST_ACTIVE_THROTTLE_MS = 60_000; // 60초

@Injectable()
export class UsersService {
  // 사용자 ID → 마지막으로 DB 갱신을 시도한 시간(ms)을 메모리에 보관
  private readonly lastActiveTouchAt = new Map<string, number>();

  private async touchLastActiveAt(
    userId: string,
    role: UserRole,
    options?: { force?: boolean },
  ): Promise<void> {
    // 일반 사용자(UserRole.USER)만 추적 — 어드민 등은 갱신 안 함
    if (role !== UserRole.USER) return;

    const now = Date.now(); // 시간 간격 계산용 숫자 (ms)

    // force 옵션이 없으면 스로틀링 적용
    if (!options?.force) {
      const prev = this.lastActiveTouchAt.get(userId) ?? 0;
      if (now - prev < LAST_ACTIVE_THROTTLE_MS) return; // 아직 제한 시간 안 → 건너뜀
    }

    // DB 업데이트 전에 먼저 Map 갱신 (중복 실행 방지 우선)
    this.lastActiveTouchAt.set(userId, now);

    await this.prisma.user.update({
      where: { id: userId },
      data: { lastActiveAt: new Date() }, // DB에는 Date 객체로
    });
  }
}
```

```txt
Date.now()와 new Date()를 다른 목적으로 씀:
  Date.now()   → 숫자(ms) — "이전 갱신과 얼마나 차이나는지" 빠르게 계산하기 위해
  new Date()   → Date 객체 — Prisma의 DateTime 필드에 저장하기 위해
  (두 함수의 차이 자체는 [[JS_Date]] 참고)

Map의 역할([[JS_Map_Set]] 참고):
  { 'user-1' => 1719745000000, 'user-2' => 1719745060000 }
  키: 사용자 ID / 값: 마지막 갱신 시도 시각(ms)
  기록이 없으면 ?? 0 → 첫 요청은 반드시 통과
```

---

# Map 갱신을 DB 업데이트 앞에 두는 이유 ⭐️⭐️⭐️

```txt
현재 순서: Map 먼저 갱신 → DB 업데이트

이 순서의 의미:
  DB 업데이트가 실패해도 Map에는 현재 시간이 이미 기록됨
  → 제한 시간 동안 재시도하지 않음 → 중복 실행 방지가 우선

반대로 할 경우 (DB 먼저 → Map 갱신):
  DB 업데이트 성공 후에만 Map에 기록 → 실패 시 다음 요청에서 재시도 가능
  하지만 동시에 여러 요청이 들어올 때 중복 UPDATE 가능성이 커짐

어느 쪽이 맞는가: 요구사항에 따라 다름
  "DB 실패 시 재시도 허용" → DB 먼저
  "중복 실행 방지 우선" → Map 먼저 (현재 코드 방식)
```

---

# fire-and-forget — 응답을 막지 않고 실행 ⭐️⭐️⭐️⭐️

```typescript
import { Injectable, Logger } from '@nestjs/common';
//                  ↑ @nestjs/common 내장 — 별도 설치 불필요

@Injectable()
export class UsersService {
  constructor(){} 
  private readonly logger = new Logger(UsersService.name);

  /** JwtAuthGuard — 응답 막지 않음 */
  touchLastActiveAtFromGuard(userId: string, role: UserRole): void {
    void this.touchLastActiveAt(userId, role).catch((error) => {
      this.logger.warn(
        `업데이트 실패: user=${userId}`,
        error instanceof Error ? error.stack : String(error),
      );
    });
  }
}
```

```txt
await 방식 (응답이 기다림):
  JWT 인증 → lastActiveAt DB 업데이트 → 업데이트 완료 → 컨트롤러 실행 → 응답
  DB가 느리면 사용자도 그만큼 기다려야 함

void 방식 (fire-and-forget, 응답 안 기다림):
  JWT 인증 ┬─ lastActiveAt 업데이트 시작 (백그라운드)
            └─ 컨트롤러 즉시 실행 → 응답 반환

lastActiveAt은 부가 정보라서 이것 때문에 핵심 API 응답이 늦어지게 할 필요가 없음
→ fire-and-forget으로 사용자 응답 속도에 영향을 주지 않고 실행
```

```txt
void가 필요한 이유:
  void 없이 그냥 this.touchLastActiveAt(...)만 호출하면
  ESLint의 no-floating-promises 규칙이 경고를 냄
  → void를 붙여서 "이 Promise의 결과를 의도적으로 기다리지 않는다"를 명시
  (void 연산자 자체의 동작은 [[React_useMemo_useCallback_useEffect]]의 "void 패턴" 참고)

.catch(() => undefined) 대신 logger.warn을 쓰는 이유:
  undefined로 삼키면 → DB가 계속 실패해도 로그에 아무것도 안 남아 운영 중 파악 불가
  warn으로 남기면 → 오류는 조용히 처리하되 흔적이 남아 나중에 확인 가능

error.stack vs String(error):
  error는 catch에서 unknown 타입 — Error 인스턴스가 아닌 값이 throw될 수도 있음
  instanceof Error로 확인해서 Error면 .stack(호출 경로), 아니면 String()으로 안전하게 변환
  (Logger 자체의 설치/사용법은 [[NestJS_Logger]] 참고)
```


## Guard에서 호출하는 패턴

```typescript
// JwtAuthGuard 안에서 — 실제 흐름
async canActivate(context: ExecutionContext): Promise<boolean> {
  // ... @Public() 데코레이터 확인 ...

  const request = context.switchToHttp().getRequest<Request>();
  const token = this.extractBearerToken(request);
  if (!token) throw new UnauthorizedException();

  try {
    const payload = await this.jwtService.verifyAsync<JwtPayload>(token, {
      secret: this.configService.getOrThrow('JWT_SECRET'),
    });

    request.user = payload;                // ① request에 페이로드 세팅
    this.someService.touchLastActiveAtFromGuard(payload.sub, payload.role);        // ② fire-and-forget
    return true;    // DB 업데이트 완료를 기다리지 않고 바로 true 반환
  } catch {
    throw new UnauthorizedException();
  }
}
```

```txt
포인트 1 — canActivate는 async / Promise<boolean>
  JWT 검증(jwtService.verifyAsync)이 비동기이므로 Guard 자체도 async가 되어야 함
  단순 예시에서 boolean으로 표기하는 경우가 있지만, 실제 JWT Guard는 항상 async

포인트 2 — 데이터 출처는 payload, request.user가 아님
  ① request.user = payload    → 이후 컨트롤러/서비스에서 @CurrentUser() 등으로 꺼낼 수 있도록 세팅
  ② touchLastActiveAtFromGuard(payload.sub, payload.role)
     → request.user를 통하지 않고 payload에서 직접 꺼내 호출
  ① ②는 순서 바꿔도 동작은 같지만, request.user를 먼저 세팅하는 게 의도를 더 명확히 드러냄

포인트 3 — payload.sub vs user.id
  sub(subject)는 JWT 표준 클레임 — 토큰이 "누구에 관한 것"인지를 나타냄
  관례적으로 userId를 sub에 담음
  → JwtPayload 타입 안에서 sub: string 으로 정의해두면 payload.sub = userId

포인트 4 — someService 위치
  touchLastActiveAt 로직을 AuthService에 둘 수도 있고, UsersService에 둘 수도 있음
  Guard가 어떤 Service를 주입받는지에 따라 달라지며, 범용 규칙은 없음
  → 중요한 것은 "서비스명"이 아니라 fire-and-forget 호출 패턴 자체
```

---

# force 옵션 — 스로틀링 무시하고 강제 갱신 ⭐️⭐️

```typescript
// private이라 같은 클래스 내부에서만 직접 호출 가능
await this.touchLastActiveAt(userId, role, { force: true });
```

```txt
force: true를 넣으면 Map 확인을 건너뛰고 바로 DB 업데이트
사용하는 경우: 로그아웃 직전, 특정 중요 활동을 정확히 기록해야 할 때

현재 touchLastActiveAt이 private이라서 같은 Service 클래스 내부에서만 force 옵션 사용 가능
외부(Guard 등)에서는 touchLastActiveAtFromGuard()만 호출 — 이쪽은 force 없이 항상 스로틀링 적용
```

---

# 스케일아웃 한계 — Map은 프로세스 메모리 ⭐️⭐️⭐️⭐️

```txt
이 Map은 NestJS 서버 프로세스 메모리에만 존재함:
  서버 재시작 → Map 초기화 → 다음 요청에서 다시 DB 업데이트 (큰 문제 아님)

  서버가 여러 대면:
    서버 A의 Map / 서버 B의 Map → 서로 독립
    요청 1 → 서버 A → DB 업데이트
    요청 2 → 서버 B → 서버 B Map에 기록 없으므로 DB 업데이트 (중복)

  → 여러 인스턴스에서 스로틀링을 정확히 적용하려면 Redis TTL 같은 공유 저장소가 필요
```

```txt
Redis TTL 방식 (스케일아웃 대응):
  SET last-active:user-123 1 EX 60   // 60초 TTL로 저장
  GET last-active:user-123            // 키가 있으면 → 아직 제한 시간 안
  키 없음                              // → 제한 시간 지남, DB 업데이트

  모든 서버 인스턴스가 같은 Redis를 바라보므로 중복 UPDATE를 정확히 막을 수 있음
  MVP/소규모에서는 Map 방식으로 시작하고, 인스턴스가 늘어나는 시점에 Redis로 전환
```

---

# 한눈에

```txt
스로틀링: "N ms 안에 한 번만 실행" — Map<userId, 마지막갱신ms>으로 구현
  now - prev < THROTTLE_MS → 건너뜀
  처음이거나 제한 시간 지남 → Map 갱신 후 DB 업데이트

Map 갱신 순서:
  DB 앞: 중복 실행 방지 우선 (실패 시 제한 시간 동안 재시도 안 함)
  DB 뒤: 성공 시에만 기록 (실패 시 재시도 가능, 동시 요청 중복 위험)

fire-and-forget:
  void asyncFn().catch(err => logger.warn(...))
  응답 속도에 영향 없이 실행 — 부가 정보 갱신에 적합
  .catch는 최소한 로그라도 남기는 게 운영에 유리

Date.now()  숫자(ms) — 간격 계산용
new Date()  Date 객체 — Prisma DateTime 저장용

한계:
  Map은 프로세스 메모리 → 서버 재시작 시 초기화, 다중 인스턴스에서 공유 안 됨
  → 다중 서버라면 Redis TTL로 대체
```