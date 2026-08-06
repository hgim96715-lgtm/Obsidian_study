---
aliases:
  - "@Cron"
  - 스케줄링
  - Cron
  - Task Scheduling
  - CronExpression
tags:
  - NestJS
related:
  - "[[00_NestJS_Ecosystem_HomePage]]"
  - "[[JS_Date]]"
  - "[[NestJS_Logger]]"
  - "[[NestJS_Email]]"
---
# NestJS_Scheduling — 스케줄링 · Cron

>[!info]
>스케줄링 = 특정 시간 또는 주기에 자동으로 코드를 실행하는 것.
> `@Cron`으로 Cron 표현식 기반 스케줄을, `@Interval`로 주기 실행을, `@Timeout`으로 지연 실행을 등록한다.
>  `timeZone: 'Asia/Seoul'`로 한국 시간 기준 설정.

---

# 스케줄링이란 — 왜 필요한가 ⭐️⭐️⭐️⭐️

```txt
스케줄링이 필요한 경우:
  매일 새벽 2시에 통계 집계
  매시간 캐시 갱신
  30분마다 만료된 세션 삭제
  월요일 오전 9시에 주간 리포트 이메일 발송

이런 작업들을 클라이언트 요청 없이 서버가 자동으로 실행
→ NestJS @nestjs/schedule 모듈로 처리
```

---

# 설치 및 설정 ⭐️⭐️⭐️

```bash
pnpm add @nestjs/schedule
```

```typescript
// app.module.ts
import { ScheduleModule } from '@nestjs/schedule';

@Module({
  imports: [
    ScheduleModule.forRoot(),  // 전역 스케줄러 등록
  ],
})
export class AppModule {}
```

```typescript
// 스케줄 메서드를 담는 Service
import { Injectable } from '@nestjs/common';
import { Cron, CronExpression, Interval, Timeout } from '@nestjs/schedule';

@Injectable()
export class TasksService {
  @Cron(CronExpression.EVERY_DAY_AT_2AM, { timeZone: 'Asia/Seoul' })
  handleDailyStats() {
    // 매일 새벽 2시 (한국 시간)에 자동 실행
  }
}
```

```txt
ScheduleModule.forRoot()를 AppModule에 등록해야
@Cron, @Interval, @Timeout 데코레이터가 작동함
등록 안 하면 데코레이터를 붙여도 아무것도 실행 안 됨
```

---

# Cron이란 — 표현식 읽는 법 ⭐️⭐️⭐️⭐️

```txt
Cron = Unix/Linux에서 유래한 시간 기반 작업 스케줄러
"언제 실행할 것인가"를 5~6개 필드로 표현

NestJS는 6개 필드 (초 포함):
  ┌─────────── 초 (0-59)
  │ ┌───────── 분 (0-59)
  │ │ ┌─────── 시 (0-23)
  │ │ │ ┌───── 일 (1-31)
  │ │ │ │ ┌─── 월 (1-12)
  │ │ │ │ │ ┌─ 요일 (0-7, 0·7=일요일)
  * * * * * *
```

## 특수 문자 의미

```txt
*   모든 값 (매초, 매분, 매시간 등)
,   여러 값  (1,3,5 = 1일, 3일, 5일)
-   범위     (1-5 = 1~5)
/   간격     (*/10 = 10마다)
```

## 예시로 읽는 법

```txt
0 0 2 * * *    → 매일 새벽 2시 정각
│ │ │
│ │ └── 시: 2시
│ └──── 분: 0분
└────── 초: 0초

0 */30 * * * *  → 매 30분마다
│  │
│  └── 분: 0, 30, 60(=0), ...
└───── 초: 0초

0 0 9 * * 1    → 매주 월요일 오전 9시
│ │ │     │
│ │ │     └── 요일: 1 (월요일)
│ │ └──────── 시: 9
│ └────────── 분: 0
└──────────── 초: 0

0 0 0 1 * *    → 매월 1일 자정
│ │ │ │
│ │ │ └── 일: 1일
│ │ └──── 시: 0시
│ └────── 분: 0분
└──────── 초: 0초
```

---

# @Cron — Cron 표현식으로 스케줄 ⭐️⭐️⭐️⭐️

```typescript
import { Cron, CronExpression } from '@nestjs/schedule';

@Injectable()
export class TasksService {

  // 방법 1 — CronExpression enum (권장)
  @Cron(CronExpression.EVERY_DAY_AT_2AM, { timeZone: 'Asia/Seoul' })
  async aggregateDailyStats() {
    await this.statsService.aggregate();
  }

  // 방법 2 — 직접 문자열
  @Cron('0 0 2 * * *', { timeZone: 'Asia/Seoul' })
  async aggregateDailyStats2() { ... }

  // 이름 지정 (동적 제어에 사용)
  @Cron('0 0 * * * *', { name: 'hourly-cleanup', timeZone: 'Asia/Seoul' })
  async hourlyCleanup() { ... }
}
```

## timeZone 옵션 ⭐️⭐️⭐️⭐️

```typescript
@Cron(CronExpression.EVERY_DAY_AT_2AM, {
  timeZone: 'Asia/Seoul',  // 한국 시간 기준
})
```

```txt
timeZone을 안 쓰면:
  서버의 시스템 시간 기준 (보통 UTC)
  배포 서버가 UTC이면 새벽 2시 = 한국 시간 오전 11시

timeZone: 'Asia/Seoul':
  한국 표준시(KST = UTC+9) 기준으로 Cron 실행
  새벽 2시 = 실제로 한국 새벽 2시에 실행

→ 서버가 UTC인 환경에서 한국 시간으로 스케줄하려면 반드시 설정
```

## 자주 쓰는 CronExpression ⭐️⭐️⭐️⭐️

```typescript
CronExpression.EVERY_SECOND           // 매초
CronExpression.EVERY_10_SECONDS       // 10초마다
CronExpression.EVERY_30_SECONDS       // 30초마다
CronExpression.EVERY_MINUTE           // 매분
CronExpression.EVERY_10_MINUTES       // 10분마다
CronExpression.EVERY_30_MINUTES       // 30분마다
CronExpression.EVERY_HOUR             // 매시간
CronExpression.EVERY_DAY_AT_MIDNIGHT  // 매일 자정
CronExpression.EVERY_DAY_AT_1AM       // 매일 새벽 1시
CronExpression.EVERY_DAY_AT_2AM       // 매일 새벽 2시
CronExpression.EVERY_WEEK             // 매주 일요일 자정
CronExpression.EVERY_MONTH            // 매월 1일 자정
```

```txt
CronExpression이 없는 경우 — 직접 문자열로:
  매주 월요일 오전 9시  → '0 0 9 * * 1'
  평일 오전 8시 30분   → '0 30 8 * * 1-5'
  매월 마지막 날        → 직접 계산 로직 필요 (Cron 표현식으로 불가)
```

---

# @Interval — 주기적 실행 ⭐️⭐️⭐️

```typescript
import { Interval } from '@nestjs/schedule';

@Injectable()
export class TasksService {

  @Interval(30000)  // 30,000ms = 30초마다
  async refreshCache() {
    await this.cacheService.refresh();
  }

  @Interval('health-check', 10000)  // 이름 지정 + 10초마다
  checkHealth() {
    console.log('서버 상태 확인');
  }
}
```

```txt
@Cron vs @Interval:
  @Cron    → "몇 시에" 실행 (특정 시각 기준)
  @Interval → "얼마마다" 실행 (앱 시작 후부터 주기적으로)

  @Interval은 앱이 시작된 시점부터 카운트 시작
  새벽 2시 정각에 실행이 필요하면 @Cron
  앱 시작 후 매 N초마다 실행이면 @Interval
```

---

# @Timeout — 지연 후 1회 실행 ⭐️⭐️

```typescript
import { Timeout } from '@nestjs/schedule';

@Injectable()
export class TasksService {

  @Timeout(5000)  // 앱 시작 5초 후 1회 실행
  async initializeSeed() {
    await this.seedService.run();
  }
}
```

---

# 동적 제어 — 스케줄 수동 조작 ⭐️⭐️⭐️

```typescript
import { SchedulerRegistry } from '@nestjs/schedule';
import { CronJob } from 'cron';

@Injectable()
export class TasksService {
  constructor(private schedulerRegistry: SchedulerRegistry) {}

  // 스케줄 중지
  stopHourlyCleanup() {
    const job = this.schedulerRegistry.getCronJob('hourly-cleanup');
    job.stop();
    console.log('스케줄 중지됨');
  }

  // 스케줄 재시작
  startHourlyCleanup() {
    const job = this.schedulerRegistry.getCronJob('hourly-cleanup');
    job.start();
  }

  // 동적으로 Cron 등록
  addCronJob(name: string, cronTime: string) {
    const job = new CronJob(cronTime, () => {
      console.log(`${name} 실행`);
    });
    this.schedulerRegistry.addCronJob(name, job);
    job.start();
  }
}
```

```txt
동적 제어가 필요한 경우:
  특정 조건에서 스케줄을 일시 중지·재시작
  사용자가 설정한 시간에 스케줄 등록
  관리자 UI에서 스케줄 on/off

이름(@Cron의 name 옵션)으로 특정 스케줄을 찾아서 제어
```

---

# 실전 패턴 ⭐️⭐️⭐️⭐️

```typescript
@Injectable()
export class StatsScheduler {
  private readonly logger = new Logger(StatsScheduler.name);

  constructor(private readonly statsService: StatsService) {}

  // 매일 새벽 2시 (한국 시간) — 전날 통계 집계
  @Cron(CronExpression.EVERY_DAY_AT_2AM, { timeZone: 'Asia/Seoul' })
  async aggregateDailyStats() {
    this.logger.log('일별 통계 집계 시작');
    try {
      await this.statsService.aggregateYesterday();
      this.logger.log('일별 통계 집계 완료');
    } catch (error) {
      this.logger.error('일별 통계 집계 실패', error);
      // 실패해도 앱이 죽으면 안 됨 → try/catch 필수
    }
  }

  // 매시간 만료된 세션 삭제
  @Cron(CronExpression.EVERY_HOUR, { timeZone: 'Asia/Seoul' })
  async cleanExpiredSessions() {
    const deleted = await this.sessionService.deleteExpired();
    this.logger.log(`만료 세션 ${deleted}개 삭제`);
  }
}
```

## 실패 시 ops 메일 발송 ⭐️⭐️⭐️⭐️

```typescript
/** 매일 02:00 KST — 어제 일 합계. 실패 시 즉시 ops 메일 */
@Cron(CronExpression.EVERY_DAY_AT_2AM, { timeZone: 'Asia/Seoul' })
async snapshotYesterday() {
  try {
    const result = await this.adminStatsService.snapshotKstDay(yesterday);
    this.logger.log(
      `stats snapshot date=${result.date} recommendations=${result.recommendations} signups=${result.signups} active=${result.active}`,
    );
  } catch (error) {
    const message = error instanceof Error ? error.message : String(error);

    // ① 에러 로그
    this.logger.error(`stats snapshot failed · error=${message}`);

    // ② 운영팀에 즉시 알림 이메일 발송
    await this.mailService.sendOpsAlert(
      'stats snapshot 실패',
      [
        'AdminStatsCron.snapshotYesterday 실패',
        `time: ${new Date().toISOString()}`,
        `error: ${message}`,
        '',
        '대시보드「어제 스냅샷 저장」또는 POST /admin/stats/snapshot 으로 재실행',
      ].join('\n'),
    );
  }
}
```

```txt
스케줄 실패 시 ops 메일을 보내는 이유:
  스케줄은 자동으로 실행 → 실패해도 아무도 모를 수 있음
  Logger 에러 로그만으로는 운영팀이 실시간으로 파악하기 어려움
  → 실패 시 즉시 이메일로 알림

  메일 본문에 포함할 것:
    어느 작업이 실패했는지 (함수명)
    언제 실패했는지 (ISO 타임스탬프)
    왜 실패했는지 (error message)
    어떻게 수동으로 재실행하는지 (API 경로)

이메일 발송 자체가 실패할 수 있음:
  sendOpsAlert() 내부에서 에러를 잡아서 처리 → [[NestJS_Email]]
  이메일 실패가 앱을 죽이면 안 됨
```

```txt
스케줄 메서드 주의사항:
  try/catch 필수 — 예외가 발생해도 앱이 죽으면 안 됨
  Logger 사용 — 실행 여부와 에러를 로그로 남겨야 추적 가능
  실행 시간 주의 — 오래 걸리는 작업은 다음 스케줄 실행 전에 끝나야 함
  중복 실행 주의 — 스케일 아웃 환경에서 여러 인스턴스가 동시 실행 가능
                  → Redis 기반 분산 락 또는 단일 인스턴스에서만 실행 설정 필요
```