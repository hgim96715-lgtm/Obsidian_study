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
  - "[[NestJS_Scheduling]]"
---
# NestJS_Logger — 로거

>[!info]
>Logger = NestJS 내장 로깅 클래스. 
>`console.log` 대신 쓴다. 레벨(log·warn·error·debug), 타임스탬프, 컨텍스트(어느 클래스에서 찍었는지)를 자동으로 포함한다. 
>프로덕션에서 레벨별로 출력 여부를 제어할 수 있다.

---

# console.log 대신 Logger를 쓰는 이유 ⭐️⭐️⭐️⭐️

```typescript
// ❌ console.log
console.log('통계 집계 시작');
console.log('에러 발생', error);

// ✅ Logger
this.logger.log('통계 집계 시작');
this.logger.error('통계 집계 실패', error.stack);
```

```txt
출력 비교:
  console.log:  통계 집계 시작
  Logger:       [Nest] 1234 - 2024/01/15, 02:00:00  LOG   [AdminStatsCron] 통계 집계 시작

Logger가 자동으로 추가하는 것:
  ① 타임스탬프   — 언제 발생했는지
  ② 레벨         — LOG, WARN, ERROR, DEBUG, VERBOSE
  ③ 컨텍스트     — [AdminStatsCron] 어느 클래스에서 찍었는지

이것들이 있어야:
  에러가 발생했을 때 언제, 어디서 발생했는지 바로 파악
  프로덕션에서 DEBUG 레벨만 끄는 등 레벨별 제어 가능
```

---

# 기본 사용법 ⭐️⭐️⭐️⭐️

```typescript
import { Injectable, Logger } from '@nestjs/common';

@Injectable()
export class AdminStatsCron {
  // 클래스명을 컨텍스트로 전달 — 어느 클래스에서 찍었는지 자동 표시
  private readonly logger = new Logger(AdminStatsCron.name);
  //                                    ↑
  //                            'AdminStatsCron' 문자열 = 클래스 이름
  //                            .name = TypeScript 클래스의 이름 속성

  async run() {
    this.logger.log('통계 집계 시작');

    try {
      const result = await this.statsService.aggregate();
      this.logger.log(
        `stats snapshot date=${result.date} recommendations=${result.recommendations} signups=${result.signups}`,
      );
    } catch (error) {
      this.logger.error('통계 집계 실패', error instanceof Error ? error.stack : error);
    }
  }
}
```

## ClassName.name 이 무엇인가

```typescript
class AdminStatsCron {}

AdminStatsCron.name  // 'AdminStatsCron' — 클래스 이름을 문자열로 반환

// 직접 문자열을 넣어도 되지만
new Logger('AdminStatsCron')   // ← 클래스 이름을 변경하면 수동으로 수정 필요

// .name을 쓰면 클래스 이름과 항상 일치
new Logger(AdminStatsCron.name)  // ✅ 클래스 이름 바뀌면 자동으로 따라감
```

---

# 로그 레벨 5가지 ⭐️⭐️⭐️⭐️

```typescript
this.logger.log('일반적인 정보 — 정상 동작 흐름');       // LOG
this.logger.warn('주의가 필요한 상황 — 에러는 아님');    // WARN
this.logger.error('에러 발생', error.stack);             // ERROR
this.logger.debug('개발 중 디버깅 정보');                 // DEBUG
this.logger.verbose('상세한 실행 흐름 — 매우 자세한 정보'); // VERBOSE
```

```txt
각 레벨은 언제 쓰는가:

  log (INFO):
    서비스 시작/완료, 중요 비즈니스 이벤트
    "통계 집계 완료", "이메일 발송 성공"
    → 운영 중 항상 남겨야 하는 기록

  warn (WARNING):
    에러는 아니지만 주의가 필요한 상황
    "재시도 중 (2/3)", "캐시 미스로 DB 조회"
    → 나중에 점검이 필요할 수 있는 상황

  error (ERROR):
    예외 발생, 처리 실패
    에러 메시지 + stack trace 같이 남김
    → 즉각 확인이 필요한 문제

  debug (DEBUG):
    개발 중에만 필요한 상세 정보
    "쿼리 파라미터: { id: 123 }"
    → 프로덕션에서는 끄는 것이 일반적

  verbose (VERBOSE):
    매우 상세한 실행 흐름
    → 거의 안 씀, debug보다 더 자세한 경우
```

---

# error 레벨 — stack trace ⭐️⭐️⭐️⭐️

```typescript
try {
  await this.statsService.aggregate();
} catch (error) {
  // ❌ 메시지만 남기면 어디서 발생했는지 모름
  this.logger.error('실패');

  // ✅ stack trace 포함 — 어느 파일 몇 번째 줄인지 확인 가능
  this.logger.error(
    '통계 집계 실패',
    error instanceof Error ? error.stack : String(error),
  );
}
```

```txt
error.stack이란:
  에러가 발생한 위치와 호출 경로를 문자열로 담은 것
  "Error: ... at AdminStatsCron.run (admin-stats.cron.ts:25:13)"
  → 파일명과 줄 번호로 정확한 위치 파악 가능

error instanceof Error 체크:
  Promise rejection이나 서드파티 에러는 Error 객체가 아닐 수 있음
  → instanceof 체크 후 .stack 접근
```

---

# 로그 레벨 설정 — 프로덕션 제어 ⭐️⭐️⭐️

```typescript
// main.ts
async function bootstrap() {
  const app = await NestFactory.create(AppModule, {
    logger: ['log', 'warn', 'error'],  // debug·verbose 제외
    // logger: false,                  // 전체 끄기
    // logger: ['error'],              // 에러만
  });
}
```

```typescript
// 환경별 로그 레벨
const logLevels = process.env.NODE_ENV === 'production'
  ? ['log', 'warn', 'error'] as const
  : ['log', 'warn', 'error', 'debug', 'verbose'] as const;

const app = await NestFactory.create(AppModule, { logger: logLevels });
```

```txt
왜 프로덕션에서 debug를 끄는가:
  debug 로그가 너무 많으면 중요한 log/error가 묻힘
  로그 저장 비용 증가
  민감한 정보가 debug 로그에 포함될 수 있음

  개발: log + warn + error + debug
  프로덕션: log + warn + error
```

---

# 언제 로그를 남기는가 ⭐️⭐️⭐️⭐️

```typescript
@Injectable()
export class AdminStatsCron {
  private readonly logger = new Logger(AdminStatsCron.name);

  @Cron(CronExpression.EVERY_DAY_AT_2AM, { timeZone: 'Asia/Seoul' })
  async run() {
    // ① 시작 로그 — 스케줄이 실제로 실행됐는지 확인
    this.logger.log('일별 통계 집계 시작');

    try {
      const result = await this.statsService.aggregate();

      // ② 완료 로그 — 결과를 key=value 형태로 남김
      this.logger.log(
        `stats snapshot date=${result.date} recommendations=${result.recommendations} signups=${result.signups} active=${result.active}`,
      );

    } catch (error) {
      // ③ 에러 로그 — stack trace 포함
      this.logger.error('일별 통계 집계 실패', error instanceof Error ? error.stack : error);
      // try/catch로 감싸야 에러가 나도 앱이 죽지 않음
    }
  }
}
```

```txt
로그를 남겨야 하는 시점:
  ① 중요한 작업 시작 (스케줄, 배치, 외부 API 호출)
  ② 작업 완료 + 결과 요약
  ③ 예외 발생 (항상 error 레벨로)
  ④ 예상과 다른 상황 (warn)

로그를 남기지 않아도 되는 시점:
  모든 HTTP 요청 (NestJS가 기본으로 남김)
  단순 CRUD의 모든 단계
  → 너무 많은 로그는 중요한 것을 찾기 어렵게 만듦

key=value 형태로 남기는 이유:
  date=2024-01-15 recommendations=42 signups=5
  → 나중에 로그 검색 시 특정 값으로 필터 가능
  → "recommendations=0 인 날 찾기" 등 분석에 용이
```

---

# 자주 만나는 패턴

```typescript
// 서비스에서 Logger 선언
@Injectable()
export class PostsService {
  private readonly logger = new Logger(PostsService.name);

  async create(userId: string, dto: CreatePostDto) {
    const post = await this.prisma.post.create({ data: { ...dto, authorId: userId } });
    this.logger.log(`게시글 생성 userId=${userId} postId=${post.id}`);
    return post;
  }
}
```

```typescript
// 외부 API 호출 실패 warn
async fetchExternalData(id: string) {
  try {
    return await this.httpService.get(`/api/${id}`);
  } catch (error) {
    this.logger.warn(`외부 API 조회 실패 id=${id}, 캐시 데이터 사용`);
    return this.cache.get(id);  // fallback
  }
}
```