---
aliases:
  - Task Scheduling
  - 스케줄링
  - Cron
  - "@Cron"
tags:
  - NestJS
related:
  - "[[00_NestJS_Ecosystem_HomePage]]"
  - "[[NestJS_Caching]]"
  - "[[Linux_Shell_Cron_Job]]"
  - "[[NestJS_Service_Provider]]"
  - "[[NestJS_Prisma]]"
---
# NestJS_Scheduling — 스케줄링

# 한 줄 요약

```txt
@nestjs/schedule = 정해진 시간에 자동으로 코드 실행
@Cron      특정 시각/주기 반복 (매일 새벽 3시 등)
@Interval  일정 간격 반복 (5초마다 등)
@Timeout   앱 시작 후 한 번만 지연 실행
```

---

---

# 설치

```bash
pnpm add @nestjs/schedule
```

```typescript
// app.module.ts
import { ScheduleModule } from '@nestjs/schedule';

@Module({
  imports: [
    ScheduleModule.forRoot(),   // 전역 등록 필수
  ],
})
export class AppModule {}
```

```txt
ScheduleModule.forRoot():
  앱 전체에서 @Cron / @Interval / @Timeout 데코레이터 활성화
  등록 안 하면 데코레이터가 동작 안 함
  AppModule 한 곳에만 등록하면 됨
```

---

---

# @Cron — 특정 시각/주기 반복 ⭐️

```typescript
import { Cron, CronExpression } from '@nestjs/schedule';

@Injectable()
export class ExhibitionSyncService {
  // 매일 새벽 3시에 실행
  @Cron('0 0 3 * * *')
  async syncExhibitions() {
    console.log('전시 동기화 시작');
  }

  // CronExpression enum 사용 (가독성 좋음) ️
  @Cron(CronExpression.EVERY_DAY_AT_3AM)
  async syncDaily() {
    // 매일 새벽 3시
  }
}
```

## Cron 표현식 구조 ⭐️

```txt
   ┌──────────── 초 (0-59, 생략 가능)
   │ ┌────────── 분 (0-59)
   │ │ ┌──────── 시 (0-23)
   │ │ │ ┌────── 일 (1-31)
   │ │ │ │ ┌──── 월 (1-12)
   │ │ │ │ │ ┌── 요일 (0-7, 0과 7은 일요일)
   │ │ │ │ │ │
   *  *  *  *  *  *

'0 0 3 * * *'   매일 새벽 3시 0분 0초
'*/5 * * * * *' 5초마다
'0 */10 * * * *' 10분마다
'0 0 0 1 * *'   매월 1일 0시
'0 0 9 * * 1'   매주 월요일 9시
```

## CronExpression — 자주 쓰는 값 ⭐️

```typescript
CronExpression.EVERY_SECOND
CronExpression.EVERY_5_SECONDS
CronExpression.EVERY_10_SECONDS
CronExpression.EVERY_MINUTE
CronExpression.EVERY_5_MINUTES
CronExpression.EVERY_HOUR
CronExpression.EVERY_DAY_AT_MIDNIGHT
CronExpression.EVERY_DAY_AT_3AM
CronExpression.EVERY_WEEK
CronExpression.EVERY_1ST_DAY_OF_MONTH_AT_NOON
CronExpression.EVERY_WEEKDAY      // 평일만
CronExpression.EVERY_WEEKEND      // 주말만
```

```txt
직접 문자열 vs CronExpression enum:
  '0 0 3 * * *'                      → 의미 파악에 시간 걸림
  CronExpression.EVERY_DAY_AT_3AM    → 이름만 봐도 명확

  표준 시간이면 enum 사용 권장
  특수한 시간(매월 15일 등)은 직접 문자열 작성
```

## 타임존 지정 ⭐️

```typescript
@Cron(CronExpression.EVERY_DAY_AT_3AM, {
  name:     'syncExhibitions',
  timeZone: 'Asia/Seoul',
})
async syncDaily() {
  // 한국 시간 기준 새벽 3시에 실행
}
```

```txt
timeZone 안 주면:
  서버가 실행되는 환경의 시간대 기준 (보통 UTC)
  Railway / Vercel 등 배포 환경은 UTC 가 기본
  → 한국 시간 기준으로 돌리려면 timeZone: 'Asia/Seoul' 명시 필수

name 옵션:
  여러 Cron 작업을 구분하는 이름
  SchedulerRegistry 로 동적 제어할 때 이 이름으로 찾음
```

---

---

# @Interval — 일정 간격 반복 ⭐️

```typescript
import { Interval } from '@nestjs/schedule';

@Injectable()
export class HealthCheckService {
  @Interval(5000)   // 5000ms = 5초마다
  checkHealth() {
    console.log('서버 상태 체크');
  }

  @Interval('cleanup', 60000)   // 이름 + 60초마다
  cleanup() {
    console.log('1분마다 정리 작업');
  }
}
```

```txt
@Cron vs @Interval:
  @Cron      특정 시각 기준 (매일 3시, 매주 월요일 등)
  @Interval  앱 시작 시점부터 일정 간격 (5초마다, 1분마다)

  배치 작업 (전시 동기화) → @Cron
  헬스체크 / 캐시 정리   → @Interval
```

---

---

# @Timeout — 한 번만 지연 실행

```typescript
import { Timeout } from '@nestjs/schedule';

@Injectable()
export class InitService {
  @Timeout(5000)   // 앱 시작 5초 후 딱 한 번 실행
  afterStart() {
    console.log('초기화 작업 실행');
  }
}
```

```txt
@Timeout 용도:
  앱 시작 직후 바로 하면 안 되는 초기화 작업
  DB 연결 완료를 약간 기다린 후 실행하고 싶을 때

  반복 안 됨 — 딱 한 번만 실행
  OnApplicationBootstrap 과 비슷하지만 지연 시간 직접 지정 가능
```

---

---

# SchedulerRegistry — 동적 제어 ⭐️

```typescript
import { SchedulerRegistry } from '@nestjs/schedule';

@Injectable()
export class CronControlService {
  constructor(private schedulerRegistry: SchedulerRegistry) {}

  // 특정 Cron 작업 일시 정지
  stopJob(name: string) {
    const job = this.schedulerRegistry.getCronJob(name);
    job.stop();
  }

  // 다시 시작
  startJob(name: string) {
    const job = this.schedulerRegistry.getCronJob(name);
    job.start();
  }

  // 등록된 모든 Cron 작업 이름 조회
  listJobs() {
    return [...this.schedulerRegistry.getCronJobs().keys()];
  }
}
```

```txt
SchedulerRegistry 가 필요한 경우:
  관리자 페이지에서 배치 작업 켜고 끄기
  특정 조건에서만 스케줄 작업 실행하도록 제어
  실행 중인 작업 목록 확인

getCronJob(name):
  @Cron(..., { name: 'syncExhibitions' }) 에서 지정한 name 으로 찾음
  → name 옵션을 줘야 SchedulerRegistry 로 제어 가능
```

---

---

# 실전 패턴 — 전시 데이터 동기화 ⭐️

```typescript
@Injectable()
export class ExhibitionSyncService {
  constructor(
    private readonly prisma:        PrismaService,
    private readonly externalApi:   ExternalApiService,
    private readonly logger:        LoggerService,
    // ⚠️ NestJS 기본 LoggerService 는 컨텍스트(클래스명) 가 안 붙음
    // setContext 로 직접 지정해야 로그에 클래스명이 표시됨
  ) {
    this.logger.setContext?.(ExhibitionSyncService.name);
  }

  @Cron(CronExpression.EVERY_DAY_AT_3AM, {
    name:     'syncExhibitions',
    timeZone: 'Asia/Seoul',
  })
  async syncExhibitions() {
    this.logger.log('전시 동기화 시작', ExhibitionSyncService.name);

    try {
      const exhibitions = await this.externalApi.fetchExhibitions();

      for (const item of exhibitions) {
        await this.prisma.exhibition.upsert({
          where:  { seq: item.seq },
          create: item,
          update: item,
        });
      }

      this.logger.log(`동기화 완료: ${exhibitions.length}건`, ExhibitionSyncService.name);
    } catch (e) {
      this.logger.error('동기화 실패', e, ExhibitionSyncService.name);
      // ⚠️ Cron 작업 안에서 에러를 던지면 로그만 남고 다음 스케줄에 영향 없음
      // try/catch 로 감싸서 명확하게 처리하는 것이 안전
    }
  }
}
```

```bash
Cron 작업에서 에러 처리 중요한 이유:
  @Cron 메서드 안에서 처리 안 된 에러가 발생해도
  앱 전체가 죽지 않고 다음 스케줄에 다시 실행됨
  → 그래도 try/catch + 로깅은 필수 (원인 파악 위해)
 # → [[NestJS_Logging]] 참고
```

---

---

# 한눈에

```txt
설치:
  pnpm add @nestjs/schedule
  ScheduleModule.forRoot()  ← app.module.ts 에 등록 필수

데코레이터:
  @Cron(표현식)       특정 시각/주기 (배치 작업)
  @Interval(ms)       일정 간격 (헬스체크 등)
  @Timeout(ms)        앱 시작 후 한 번만

Cron 표현식:
  '초 분 시 일 월 요일'
  CronExpression.EVERY_DAY_AT_3AM  ← enum 사용 권장

타임존:
  { timeZone: 'Asia/Seoul' }  ← 배포 환경(UTC) 에서 필수

동적 제어:
  SchedulerRegistry.getCronJob(name).stop() / .start()
  name 옵션 줘야 제어 가능

에러 처리:
  Cron 메서드 안에서 try/catch 필수
  → 처리 안 해도 앱은 안 죽지만 원인 파악 어려움
```