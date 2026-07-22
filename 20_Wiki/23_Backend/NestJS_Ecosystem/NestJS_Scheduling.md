---
aliases:
  - "@Cron"
  - 스케줄링
  - Cron
  - Task Scheduling
tags:
  - NestJS
related:
  - "[[00_NestJS_Ecosystem_HomePage]]"
  - "[[NestJS_Prisma]]"
  - "[[NestJS_Service_Provider]]"
  - "[[NestJS_Env_Config]]"
---
# NestJS_Scheduling — 스케줄링 & 반복 작업

> [!info] 
> `@nestjs/schedule`은 `node-cron`을 래핑한 NestJS 공식 스케줄링 모듈. 
> `@Cron`(특정 시각 반복) · `@Interval`(N밀리초마다) · `@Timeout`(N밀리초 후 1회) 세 가지 데코레이터를 제공한다.

---

# 설치 & 등록 ⭐️⭐️⭐️

```bash
pnpm add @nestjs/schedule
```

```typescript
// app.module.ts
import { ScheduleModule } from '@nestjs/schedule';

@Module({
  imports: [
    ScheduleModule.forRoot(),  // 앱 전체에서 스케줄링 활성화
  ],
})
export class AppModule {}
```

```typescript
// 스케줄 작업을 담는 서비스
import { Injectable } from '@nestjs/common';
import { Cron, CronExpression, Interval, Timeout } from '@nestjs/schedule';

@Injectable()
export class TasksService {
  @Cron(CronExpression.EVERY_MINUTE)
  handleEveryMinute() {
    console.log('매 분 실행');
  }
}
```

```typescript
// tasks.module.ts
@Module({
  providers: [TasksService],
})
export class TasksModule {}
```

---

# Cron 표현식 구조 ⭐️⭐️⭐️⭐️

```txt
  *    *    *    *    *    *
  ┬    ┬    ┬    ┬    ┬    ┬
  │    │    │    │    │    └── 요일 (0-7, 0과 7 모두 일요일)
  │    │    │    │    └─────── 월 (1-12)
  │    │    │    └──────────── 일 (1-31)
  │    │    └───────────────── 시 (0-23)
  │    └────────────────────── 분 (0-59)
  └─────────────────────────── 초 (0-59)  ← node-cron은 6자리 (초 포함)
```

```txt
특수 문자:
  *   모든 값
  ,   여러 값 목록  (1,3,5 → 1, 3, 5에 실행)
  -   범위          (1-5 → 1,2,3,4,5에 실행)
  /   간격          (*/5 → 5마다, 10/5 → 10부터 5마다)

예시:
  0 0 9 * * *      매일 오전 9:00:00
  0 30 8 * * 1-5   평일(월-금) 8:30:00
  0 0 */2 * * *    2시간마다 정각
  0 0 0 1 * *      매월 1일 자정
  0 0 0 * * 0      매주 일요일 자정
  0 */30 9-18 * * * 오전 9시~오후 6시 사이 30분마다
```

---

# CronExpression — 자주 쓰는 값 ⭐️⭐️⭐️⭐️

```typescript
import { CronExpression } from '@nestjs/schedule';

// 직접 cron 문자열 대신 enum으로 가독성 향상
@Cron(CronExpression.EVERY_MINUTE)
@Cron(CronExpression.EVERY_HOUR)
@Cron(CronExpression.EVERY_DAY_AT_MIDNIGHT)
```

|CronExpression|Cron 문자열|설명|
|---|---|---|
|`EVERY_SECOND`|`* * * * * *`|매 초|
|`EVERY_5_SECONDS`|`*/5 * * * * *`|5초마다|
|`EVERY_10_SECONDS`|`*/10 * * * * *`|10초마다|
|`EVERY_30_SECONDS`|`*/30 * * * * *`|30초마다|
|`EVERY_MINUTE`|`0 * * * * *`|매 분 정각|
|`EVERY_5_MINUTES`|`0 */5 * * * *`|5분마다|
|`EVERY_10_MINUTES`|`0 */10 * * * *`|10분마다|
|`EVERY_30_MINUTES`|`0 */30 * * * *`|30분마다|
|`EVERY_HOUR`|`0 0 * * * *`|매 시 정각|
|`EVERY_2_HOURS`|`0 0 */2 * * *`|2시간마다|
|`EVERY_DAY_AT_MIDNIGHT`|`0 0 0 * * *`|매일 자정|
|`EVERY_DAY_AT_NOON`|`0 0 12 * * *`|매일 정오|
|`EVERY_DAY_AT_1AM`|`0 0 1 * * *`|매일 오전 1시|
|`EVERY_WEEK`|`0 0 0 * * 0`|매주 일요일 자정|
|`EVERY_MONTH`|`0 0 0 1 * *`|매월 1일 자정|
|`EVERY_QUARTER`|`0 0 0 1 */3 *`|매 분기 1일 자정|
|`EVERY_YEAR`|`0 0 0 1 1 *`|매년 1월 1일 자정|
|`EVERY_WEEKDAY`|`0 0 0 * * 1-5`|평일(월-금) 자정|

---

# @Cron ⭐️⭐️⭐️⭐️

```typescript
import { Cron, CronExpression } from '@nestjs/schedule';

@Injectable()
export class TasksService {

  // CronExpression enum 사용
  @Cron(CronExpression.EVERY_DAY_AT_MIDNIGHT)
  async cleanupExpiredTokens() {
    await this.authService.deleteExpiredTokens();
  }

  // 직접 cron 문자열 사용
  @Cron('0 30 8 * * 1-5')   // 평일 오전 8:30
  async sendMorningDigest() {
    await this.notificationService.sendDigest();
  }

  // 옵션 추가
  @Cron('0 0 9 * * *', {
    name:     'dailyReport',  // 이름 (SchedulerRegistry로 제어 시 필요)
    timeZone: 'Asia/Seoul',   // 타임존
    disabled: process.env.NODE_ENV === 'test',  // 테스트 환경에서 비활성화
  })
  async generateDailyReport() {
    await this.reportService.generate();
  }
}
```

## @Cron 옵션

|옵션|타입|설명|
|---|---|---|
|`name`|`string`|작업 이름 — SchedulerRegistry로 제어 시 사용|
|`timeZone`|`string`|IANA 타임존 (`'Asia/Seoul'`, `'UTC'` 등)|
|`utcOffset`|`number`|UTC 오프셋 (분 단위, `timeZone`과 둘 중 하나만)|
|`disabled`|`boolean`|`true`면 등록은 되지만 실행 안 됨|
|`unrefTimeout`|`boolean`|이 작업만 남아있을 때 프로세스 종료 허용|

---

# 타임존 지정 ⭐️⭐️⭐️⭐️

```typescript
// cron 문자열은 UTC 기준 — 한국 시간으로 동작하려면 timeZone 필수
@Cron('0 0 9 * * *', { timeZone: 'Asia/Seoul' })
async morningTaskKST() {
  // KST 오전 9시 실행 (UTC 기준으로는 00:00)
}

// UTC 오프셋으로 지정하는 방법 (분 단위)
@Cron('0 0 9 * * *', { utcOffset: 9 * 60 })  // KST = UTC+9
async morningTaskWithOffset() {}
```

```txt
타임존을 지정하지 않으면:
  서버 로컬 타임존 또는 UTC 기준으로 실행됨
  Railway, AWS 같은 클라우드는 보통 UTC 기준
  → KST 9시에 실행하고 싶은데 UTC 9시에 실행되는 문제 발생

  timeZone: 'Asia/Seoul' 로 명시하면
  node-cron이 cron 표현식을 KST 기준으로 해석
  → KST 오전 9:00:00에 실행됨을 보장

자주 쓰는 타임존:
  'Asia/Seoul'      KST (UTC+9)
  'Asia/Tokyo'      JST (UTC+9)
  'America/New_York' EST/EDT
  'UTC'             협정 세계시
  'Europe/London'   GMT/BST
```

---

# @Interval — N밀리초마다 반복 ⭐️⭐️⭐️

```typescript
import { Interval } from '@nestjs/schedule';

@Injectable()
export class TasksService {

  // 5초마다 실행 (5000ms)
  @Interval(5000)
  async checkHealthStatus() {
    const status = await this.healthService.check();
    if (!status.ok) this.logger.warn('서비스 헬스 체크 실패');
  }

  // 이름 지정 (SchedulerRegistry로 제어 가능)
  @Interval('heartbeat', 10_000)
  async heartbeat() {
    this.logger.log('heartbeat');
  }
}
```

```txt
@Interval vs @Cron:
  @Interval(5000)  → 앱 시작 후 5초마다 실행 (정확한 시각 상관없음)
  @Cron(...)       → 특정 시각/패턴에 실행 (예: 매일 오전 9시)

  @Interval은 처음 실행도 5초 후부터 시작 (앱 시작 즉시 실행이 아님)
  즉시 실행 + 반복이 필요하면 @Interval + 서비스 onModuleInit()를 조합

ms 상수 활용:
  @Interval(60 * 1000)        // 1분
  @Interval(60 * 60 * 1000)   // 1시간
```

---

# @Timeout — N밀리초 후 1회 실행 ⭐️⭐️

```typescript
import { Timeout } from '@nestjs/schedule';

@Injectable()
export class TasksService {

  // 앱 시작 후 5초 뒤에 단 한 번 실행
  @Timeout(5000)
  async initialSetup() {
    await this.dataService.seedInitialData();
    this.logger.log('초기 데이터 세팅 완료');
  }

  // 이름 지정
  @Timeout('warmup', 3000)
  async warmUpCache() {
    await this.cacheService.warmUp();
  }
}
```

```txt
@Timeout 사용 사례:
  앱 시작 직후 바로 실행하면 DB 연결이 안 됐을 수 있는 초기화 작업
  앱 시작 N초 후에만 한 번 필요한 워밍업 작업
  테스트용 딜레이 작업

onModuleInit() 과의 차이:
  onModuleInit()    모듈 초기화 시 즉시 동기적으로 실행
  @Timeout(0)       이벤트 루프 한 사이클 뒤 (실질적으로 비동기적 즉시)
  @Timeout(5000)    5초 후 비동기 실행
```

---

# SchedulerRegistry — 동적 제어 ⭐️⭐️⭐️

```typescript
import { SchedulerRegistry } from '@nestjs/schedule';
import { CronJob } from 'cron';

@Injectable()
export class TasksService {
  constructor(private readonly schedulerRegistry: SchedulerRegistry) {}

  // 이름으로 cron 작업 중지/재시작
  stopJob(name: string) {
    const job = this.schedulerRegistry.getCronJob(name);
    job.stop();
    this.logger.log(`${name} 중지됨`);
  }

  startJob(name: string) {
    const job = this.schedulerRegistry.getCronJob(name);
    job.start();
  }

  // 런타임에 동적으로 cron 작업 추가
  addCronJob(name: string, cronTime: string) {
    const job = new CronJob(cronTime, () => {
      this.logger.log(`동적 작업 실행: ${name}`);
    });

    this.schedulerRegistry.addCronJob(name, job);
    job.start();
  }

  // 작업 삭제
  deleteCronJob(name: string) {
    this.schedulerRegistry.deleteCronJob(name);
  }

  // 등록된 모든 cron 작업 목록
  listJobs() {
    const jobs = this.schedulerRegistry.getCronJobs();
    jobs.forEach((value, key) => {
      this.logger.log(`작업: ${key} | 다음 실행: ${value.nextDate()}`);
    });
  }
}
```

```txt
SchedulerRegistry 사용 전제:
  @Cron(), @Interval(), @Timeout()에 name 옵션이 있어야 조회 가능
  name 없으면 자동 생성된 이름으로 접근해야 함 (불편)

메서드 종류:
  getCronJob(name)        이름으로 CronJob 인스턴스 가져오기
  addCronJob(name, job)   동적으로 cron 작업 추가
  deleteCronJob(name)     cron 작업 삭제
  getCronJobs()           전체 cron 작업 Map 반환
  getInterval(name)       interval 작업 가져오기
  deleteInterval(name)    interval 작업 삭제
  getTimeout(name)        timeout 작업 가져오기
  deleteTimeout(name)     timeout 작업 삭제
```

---

# 실전 패턴 ⭐️⭐️⭐️

## 만료된 데이터 정리

```typescript
@Cron(CronExpression.EVERY_DAY_AT_1AM, { timeZone: 'Asia/Seoul' })
async cleanupExpiredData() {
  const deleted = await this.prisma.refreshToken.deleteMany({
    where: { expiresAt: { lt: new Date() } },
  });
  this.logger.log(`만료 토큰 ${deleted.count}개 삭제`);
}
```

## 에러 처리 — 작업이 실패해도 다음 실행에 영향 없음

```typescript
@Cron(CronExpression.EVERY_HOUR)
async hourlySyncJob() {
  try {
    await this.syncService.syncData();
  } catch (error) {
    // 에러를 잡지 않으면 unhandled exception → 앱이 죽을 수 있음
    this.logger.error('hourly sync 실패', error);
    // 필요하면 Slack/이메일 알림 전송
  }
}
```

## 중복 실행 방지 (단일 인스턴스)

```typescript
private isRunning = false;

@Cron(CronExpression.EVERY_MINUTE)
async heavyTask() {
  if (this.isRunning) {
    this.logger.warn('이전 작업이 아직 실행 중 — 건너뜀');
    return;
  }
  this.isRunning = true;
  try {
    await this.doHeavyWork();
  } finally {
    this.isRunning = false;
  }
}
```

```txt
중복 실행이 문제가 되는 경우:
  작업 하나가 1분 이상 걸리는데 @Cron이 매 분 실행될 때
  이전 작업이 끝나기 전에 다음 실행이 시작되면 데이터 충돌 가능

다중 인스턴스 환경 (여러 서버):
  단순 boolean 플래그로는 서버 간 중복 방지 불가
  → Redis Lock (Redlock) 또는 DB 레벨 락 필요
```

---

# 한눈에

```txt
설치: pnpm add @nestjs/schedule
등록: ScheduleModule.forRoot() → AppModule imports

데코레이터:
  @Cron('0 0 9 * * *')                   특정 시각/패턴에 반복
  @Cron(CronExpression.EVERY_HOUR)        enum 사용 (가독성)
  @Cron('...', { timeZone: 'Asia/Seoul' }) KST 기준 실행
  @Interval(5000)                         5초마다 반복
  @Timeout(3000)                          3초 후 1회 실행

Cron 표현식 (6자리, 초 포함):
  초 분 시 일 월 요일
  0 0 9 * * *  → 매일 9:00:00

타임존:
  클라우드 서버는 대부분 UTC → timeZone: 'Asia/Seoul' 명시 필요
  지정 안 하면 서버 로컬 또는 UTC 기준

CronExpression:
  EVERY_MINUTE / EVERY_HOUR / EVERY_DAY_AT_MIDNIGHT 등 enum 제공

SchedulerRegistry:
  name을 지정한 작업을 getCronJob(name)으로 가져와 stop()/start()
  동적으로 cron 추가: new CronJob() + addCronJob()

실전:
  try-catch 필수 — 에러가 unhandled면 앱이 죽을 수 있음
  isRunning 플래그로 중복 실행 방지 (단일 인스턴스)
  다중 인스턴스는 Redis Lock 필요
```