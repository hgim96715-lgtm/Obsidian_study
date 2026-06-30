---
aliases:
  - Logger
  - logger.log
  - logger.warn
  - logger.error
tags:
  - NestJS
related:
  - "[[00_NestJS_Ecosystem_HomePage]]"
  - "[[NestJS_Throttle]]"
  - "[[NestJS_Controller]]"
  - "[[TS_Type_Guards]]"
---
# NestJS_Logger — 내장 Logger

> [!info] 
> NestJS의 Logger는 `@nestjs/common`에서 바로 import하는 내장 유틸이다. 
> 별도 패키지 설치가 필요 없고, 클래스마다 컨텍스트 이름을 붙여서 "어느 클래스에서 찍힌 로그인지" 자동으로 구분해준다.

---

# 설치 없이 바로 사용 ⭐️⭐️⭐️⭐️

```typescript
import { Injectable, Logger } from '@nestjs/common';
//                  ↑ @nestjs/common에 포함 — 별도 설치 불필요

@Injectable()
export class AuthService {
  private readonly logger = new Logger(AuthService.name);
  //                                   ↑ 클래스 이름을 컨텍스트로 등록
}
```

```txt
new Logger(AuthService.name)에서 AuthService.name이 하는 것:
  JS 클래스의 .name 속성 — 클래스 이름 문자열을 반환함 ('AuthService')
  직접 문자열로 써도 됨: new Logger('AuthService')
  .name을 쓰면 클래스 이름을 나중에 바꿔도 로그 컨텍스트가 자동으로 따라감

출력 예시:
  [Nest] LOG    [AuthService] 로그인 성공: user=abc
  [Nest] WARN   [AuthService] 업데이트 실패: user=abc
  [Nest] ERROR  [AuthService] 예기치 않은 오류
                ↑ 컨텍스트 이름   ↑ 메시지
```

---

# 로그 레벨 ⭐️⭐️⭐️⭐️

```typescript
this.logger.log('일반 정보 — 정상 흐름 확인용');
this.logger.warn('경고 — 오류는 아니지만 주의가 필요한 상황');
this.logger.error('에러 — 예외 처리, 요청 실패 등');
this.logger.debug('개발 중 상세 정보 — 프로덕션에서는 보통 꺼둠');
this.logger.verbose('아주 상세한 정보 — 거의 안 씀');
```

|메서드|언제|
|---|---|
|`log()`|정상 흐름 확인 (가입 성공, 로그인 성공 등)|
|`warn()`|오류는 아니지만 주의 필요 — fire-and-forget 실패, 재시도 상황 등|
|`error()`|실제 에러 — 예외 catch, 요청 처리 실패|
|`debug()`|개발 중 상세 정보 — 배포 환경에서는 보통 꺼둠|

---

# error 두 번째 인자 — 스택 트레이스 ⭐️⭐️⭐️

```typescript
try {
  await this.someService.call();
} catch (error) {
  this.logger.error(
    '요청 처리 실패',
    error instanceof Error ? error.stack : String(error),
    //                       ↑ Error 객체면 스택 트레이스 / 아니면 문자열로 변환
  );
}
```

```txt
두 번째 인자로 스택 트레이스를 넘기면 로그에 호출 경로가 같이 찍힘 — 어디서 터진 건지 바로 파악 가능
instanceof Error 체크가 필요한 이유:
  try/catch에서 error는 unknown 타입 — 문자열이나 객체가 throw될 수도 있음
  Error 인스턴스가 아니면 .stack이 없어서 String()으로 안전하게 변환
  (unknown 타입을 다루는 패턴은 [[TS_Type_Guards]] 참고)
```

---

# fire-and-forget에서 warn 쓰기 ⭐️⭐️⭐️⭐️

```typescript
@Injectable()
export class UsersService {
  private readonly logger = new Logger(UsersService.name);

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
.catch(() => undefined) 대신 .catch(error => logger.warn(...))을 쓰는 이유:
  undefined로 삼키면 → 오류가 발생해도 아무 흔적이 남지 않아 운영 중 파악 불가
  warn으로 남기면 → 오류는 조용히 처리하되 로그에는 기록 → 나중에 디버깅 가능

error는 warn으로 (error 레벨 아닌 이유):
  lastActiveAt은 부가 정보 — 실패해도 사용자 요청은 정상 처리됨
  핵심 기능 실패가 아니라 "알아는 두되 대응 필수는 아님" 수준 → warn이 더 적절
```

---

# 한눈에

```txt
import { Logger } from '@nestjs/common'  — 별도 설치 불필요, NestJS 내장
new Logger(ClassName.name)              — 클래스 이름이 로그 컨텍스트로 자동 표시

레벨: log(정상) / warn(주의) / error(에러) / debug(개발용)

error 두 번째 인자: error instanceof Error ? error.stack : String(error)
  → Error 인스턴스면 스택 트레이스, 아니면 문자열 변환

fire-and-forget에서는 .catch(() => undefined) 대신 .catch(err => logger.warn(...))
  → 오류를 삼키되 흔적은 남겨서 운영 중 파악 가능하게
```