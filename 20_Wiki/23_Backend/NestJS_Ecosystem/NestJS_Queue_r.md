---
aliases:
  - Bull Queue
  - BullMQ
  - NestJS Queue
  - Redis
tags:
  - NestJS
related:
  - "[[00_NestJS_Ecosystem_HomePage]]"
  - "[[NestJS_FFmpeg_r]]"
  - "[[NestJS_Scheduling]]"
---

# NestJS_Queue — 큐

```txt
Queue = 작업을 순서대로 쌓아두고 Worker 가 하나씩 처리하는 구조
오래 걸리는 작업을 메인 서버와 분리해서 처리
```

---

---

# Stack vs Queue ⭐️

```txt
Stack (스택)                Queue (큐)
  LIFO — Last In First Out    FIFO — First In First Out
  나중에 들어온 것 먼저 처리    먼저 들어온 것 먼저 처리

  예시: 뒤로 가기 기능          예시: 이메일 발송 / 영상 인코딩
  ↑ 마지막 페이지부터 되돌아감   ↑ 요청 순서대로 처리
```

---

---

# Queue 가 필요한 이유 ⭐️

```txt
Request → Response 흐름:
  클라이언트 요청 → 서버 처리 → 응답

오래 걸리는 작업 (이메일 발송 / 영상 인코딩 / 보고서 생성):
  처리 완료까지 클라이언트가 기다려야 함
  → 응답 지연 / 서버 부하 / 타임아웃 발생

Queue 사용 시:
  클라이언트 요청 → 작업을 Queue 에 넣기 → 즉시 응답
  Worker 가 Queue 에서 꺼내 백그라운드에서 처리
  → 응답 속도 빠름 / 서버 부하 분산
```

---

---

# Queue 의 장점

```txt
서버 성능 최적화
  오래 걸리는 작업을 Worker Node 에 전가
  메인 서버는 Request ↔ Response 에 집중

스케일링 유연성
  Worker Node 만 따로 스케일링 가능
  여러 Worker Node 생성 → 동시에 작업 처리

작업 안정성
  실패한 작업 자동 재시도
  실패한 작업 메타데이터 유지 → 원인 분석 가능

우선순위 지정
  중요한 작업을 먼저 처리하도록 설정 가능

레이턴시 감소
  Request ↔ Response 라이프사이클 딜레이 최소화
```

---

---

# NestJS Queue — BullMQ ⭐️

```txt
NestJS 에서 공식 지원하는 Queue 라이브러리
내부적으로 Redis 를 저장소로 사용
작업(Job) 을 Redis 에 저장 → Worker 가 꺼내서 처리
```

```bash
pnpm install @nestjs/bullmq bullmq
```

```bash
Redis 가 필요:
  BullMQ 는 Redis 에 Job 을 저장
  Docker 로 Redis 실행 후 연결
# →  Redis 설정 참고
```

---
---
# BullMQ 기본 구조

```txt
Producer   Job 을 Queue 에 추가하는 쪽 (Controller / Service)
Queue      Job 을 저장하는 공간 (Redis 기반)
Consumer   Queue 에서 Job 을 꺼내 처리하는 Worker
```
---
---
# BullModule 설정 ⭐️

## forRoot — 간단 (하드코딩)

```typescript
BullModule.forRoot({
  connection: {
    host: 'localhost',
    port: 6379,
  },
})
```

## forRootAsync — ConfigService 사용 (권장) ⭐️

```typescript
BullModule.forRootAsync({
  imports: [ConfigModule],
  inject: [ConfigService],
  useFactory: (config: ConfigService) => ({
    connection: {
      host: config.getOrThrow('REDIS_HOST'),
      port: config.getOrThrow('REDIS_PORT'),
    },
  }),
})
```

```txt
forRootAsync 쓰는 이유:
  Docker 로 Redis 실행 시 host / port 를 .env 로 관리
  ConfigService 로 값을 안전하게 주입
  → TypeORM forRootAsync 와 동일한 패턴

.env:
  REDIS_HOST=localhost
  REDIS_PORT=6379
```

## BullModule.registerQueue ⭐️

```typescript
BullModule.registerQueue({
  name: 'thumbnail',
})
```

```txt
registerQueue 란:
  이름을 가진 Queue 를 하나 생성해서 등록
  이 Queue 에 Job 을 넣거나(Producer)
  이 Queue 의 Job 을 처리할(Consumer) 때
  name 으로 어떤 Queue 인지 식별

여러 Queue 등록:
  BullModule.registerQueue({ name: 'thumbnail' })
  BullModule.registerQueue({ name: 'email' })
  BullModule.registerQueue({ name: 'report' })
  → 용도별로 Queue 분리 가능

Producer 에서:
  @InjectQueue('thumbnail') private queue: Queue
  → 'thumbnail' Queue 에 Job 추가

Consumer 에서:
  @Processor('thumbnail')
  → 'thumbnail' Queue 의 Job 처리
```

## app.module.ts 전체

```typescript
@Module({
  imports: [
    ConfigModule.forRoot({ isGlobal: true }),

    BullModule.forRootAsync({
      imports: [ConfigModule],
      inject: [ConfigService],
      useFactory: (config: ConfigService) => ({
        connection: {
          host: config.getOrThrow('REDIS_HOST'),
          port: config.getOrThrow('REDIS_PORT'),
        },
      }),
    }),

    BullModule.registerQueue({
      name: 'thumbnail',   // 썸네일 생성 Queue
    }),
  ],
})
export class AppModule {}
```

---
---
# Producer — Job 추가

```typescript
@Injectable()
export class ThumbnailService {
  constructor(
    @InjectQueue('thumbnail') private thumbnailQueue: Queue
    // ↑ 'thumbnail' Queue 인스턴스를 주입받음
    // BullModule.registerQueue({ name: 'thumbnail' }) 으로 등록한 것
  ) {}

  async addThumbnailJob(movies: Movie[]) {
    await this.thumbnailQueue.add(
      'thumbnail',                             // Job 이름
      {                                        // Job 데이터
        videoId:   movies.map((m) => m.filename),
        videoPath: movies.map((m) => m.path),
      },
      {                                        // Job 옵션
        priority:         1,
        delay:            100,
        attempts:         3,
        lifo:             true,
        removeOnComplete: true,
        removeOnFail:     true,
      },
    );
  }
}
```

```txt
add(이름, 데이터, 옵션) 세 번째 인자 — Job 옵션:

  priority: 1
    숫자가 낮을수록 우선순위 높음
    여러 Job 이 쌓여있을 때 먼저 처리됨

  delay: 100
    Job 이 Queue 에 들어간 후 100ms 뒤에 실행
    즉시 처리 막고 약간 딜레이 주고 싶을 때

  attempts: 3
    실패 시 최대 3번까지 재시도
    모두 실패하면 failed 상태로 이동

  lifo: true
    Last In First Out — 나중에 추가된 Job 먼저 처리
    기본값(false) 은 FIFO (먼저 들어온 것 먼저)

  removeOnComplete: true
    Job 성공 시 Redis 에서 자동 삭제
    → 메모리 낭비 방지

  removeOnFail: true
    Job 실패 시 Redis 에서 자동 삭제
    false 이면 실패 Job 목록에 남아서 나중에 확인 가능
```


```txt
@InjectQueue('이름'):
  registerQueue 에서 등록한 Queue 를 이름으로 주입
  name 이 일치해야 같은 Queue 를 바라봄

  registerQueue({ name: 'thumbnail' })  ← 등록
  @InjectQueue('thumbnail')             ← 주입 (이름 일치 필수)
```

----
---
# Consumer — Job 처리

```typescript
import { Processor, WorkerHost } from '@nestjs/bullmq';
import { Job } from 'bullmq';

@Processor('thumbnail')   // Queue 이름과 일치
export class ThumbnailProcessor extends WorkerHost {

  async process(job: Job) {
    switch (job.name) {
      case 'thumbnail':
        const { videoId, videoPath } = job.data;
        // 썸네일 생성 로직
        break;
    }
  }
}
```

```txt
WorkerHost ⭐️:
  BullMQ 에서 제공하는 추상 클래스
  extends WorkerHost 하면 process() 메서드 구현 강제

  역할:
    Queue 에서 Job 을 꺼내는 Worker 를 자동으로 생성·관리
    Job 이 들어오면 process() 를 자동 호출
    성공 시 Job 을 완료 처리 / 실패 시 재시도 처리

  extends 안 하면:
    process() 를 만들어도 BullMQ 가 자동 호출 안 함
    → 직접 Worker 인스턴스 관리해야 함 (복잡)

  WorkerHost 가 관리하는 것:
    process() 자동 호출
    Job 완료 / 실패 / 재시도 상태 관리
    attempts 초과 시 failed 처리

@Processor('큐이름'):
  해당 Queue 의 Job 이 들어오면 이 클래스의 process() 호출
  Queue 이름과 반드시 일치해야 함

process(job):
  job.name  → 어떤 Job 인지 구분 (switch 로 분기)
  job.data  → Job 에 담긴 데이터
```

---
---
# Module 등록

```typescript
@Module({
  imports: [
    BullModule.registerQueue({ name: 'email' }),
  ],
  providers: [EmailService, EmailProcessor],
})
export class EmailModule {}
```
---
---

# Worker 서버 분리 — ConditionalModule ⭐️

```txt
Consumer(Processor) 는 메인 API 서버와 같이 실행할 필요 없음
오래 걸리는 작업 처리 전용 Worker 서버를 별도로 실행

문제:
  Consumer 가 API 서버에 함께 있으면
  영상 변환 같은 무거운 작업이 API 응답에 영향을 줌

해결:
  Worker 모듈을 분리 → 환경변수로 조건부 등록
  TYPE=worker 일 때만 WorkerModule 실행
  → API 서버 / Worker 서버 역할 분리
```

## 폴더 구조

```txt
src/
├── worker/
│   ├── worker.module.ts
│   └── thumbnail-generation.worker.ts
└── app.module.ts
```

## Processor 작성


```typescript
// worker/thumbnail-generation.worker.ts
import { Processor, WorkerHost } from '@nestjs/bullmq';
import { Job } from 'bullmq';

@Processor('thumbnail-generation')   // Queue 이름과 일치
export class ThumbnailGenerationProcessor extends WorkerHost {
  async process(job: Job, token?: string): Promise<any> {
    const { videoId, videoPath } = job.data;

    console.log(videoId, videoPath);
    // 실제 처리 로직 (FFmpeg 썸네일 생성 등)
    return 0;
  }
}
```

## WorkerModule


```typescript
// worker/worker.module.ts
@Module({
  imports: [
    BullModule.registerQueue({ name: 'thumbnail-generation' }),
  ],
  providers: [ThumbnailGenerationProcessor],
})
export class WorkerModule {}
```

## app.module.ts — ConditionalModule 로 조건부 등록

```typescript
import { ConditionalModule } from '@nestjs/config';

@Module({
  imports: [
    // TYPE=worker 일 때만 WorkerModule 등록
    ConditionalModule.registerWhen(
      WorkerModule,
      (env: NodeJS.ProcessEnv) => env['TYPE'] === 'worker',
    ),
  ],
})
export class AppModule {}
```

## package.json 스크립트

```json
{
  "scripts": {
    "start:dev":        "nest start --watch",
    "start:dev:worker": "export TYPE=worker && export PORT=3002 && nest start --watch"
  }
}
```

```txt
실행:
  pnpm start:dev         → API 서버 (포트 3001) — WorkerModule 없음
  pnpm start:dev:worker  → Worker 서버 (포트 3002) — WorkerModule 실행

TYPE=worker:
  ConditionalModule 조건 함수에 env['TYPE'] 전달
  'worker' 이면 true → WorkerModule 등록
  없으면 false → WorkerModule 스킵

PORT=3002:
  Worker 서버는 HTTP 요청 처리 불필요하지만
  NestJS 앱이라 포트 필요 → 메인 서버(3001)와 충돌 방지

두 서버 동시 실행:
  터미널 2개 열어서 각각 실행
  API 서버가 Queue 에 Job 추가 → Worker 서버가 처리
```

---

---

# 핵심 흐름 정리

```txt
API 서버 (TYPE 없음)          Worker 서버 (TYPE=worker)
  ↓                              ↓
클라이언트 요청              WorkerModule 등록
  ↓                              ↓
Producer → Queue 에 Job 추가  Consumer → Queue 에서 Job 꺼냄
  ↓                              ↓
즉시 응답 (201 OK)            처리 (썸네일 생성 / 이메일 발송 등)
```