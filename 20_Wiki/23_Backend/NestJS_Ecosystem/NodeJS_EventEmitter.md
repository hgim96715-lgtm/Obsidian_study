---
aliases: [EventEmitter, on/emit, pub/sub Node]
tags:
  - NodeJS
related:
  - "[[00_NestJS_Ecosystem_HomePage]]"
  - "[[JS_CustomEvent]]"
---
# NodeJS_EventEmitter — 서버 쪽 발행/구독 패턴

> [!info] 
> `EventEmitter`는 Node.js 서버 환경에서의 발행/구독 패턴이다. 
> 브라우저에서 `dispatchEvent`/`addEventListener`로 했던 것과 정확히 같은 발상을, 서버 프로세스 안에서 `emit`/`on`으로 한다.

---

# on / emit — 기초 ⭐️⭐️⭐️⭐️

```typescript
import { EventEmitter } from 'events';

const emitter = new EventEmitter();

emitter.on('user-created', (user) => {
  console.log('새 유저:', user.email);
});

emitter.emit('user-created', { email: 'a@example.com' });
// → 위 on으로 등록한 콜백이 즉시 실행됨
```

|메서드|역할|JS_CustomEvent의 대응|
|---|---|---|
|`emitter.on(name, fn)`|구독 — name 이벤트가 발생하면 fn 실행|`window.addEventListener`|
|`emitter.emit(name, data)`|발행 — name 이벤트를 지금 발생시킴, data를 콜백 인자로 전달|`window.dispatchEvent(new CustomEvent(name, { detail: data }))`|
|`emitter.off(name, fn)`|구독 해제|`window.removeEventListener`|
|`emitter.once(name, fn)`|딱 한 번만 실행되고 자동으로 구독 해제|(브라우저엔 직접 대응 없음, `{ once: true }` 옵션과 유사)|

```txt
브라우저의 CustomEvent와 다른 점 하나: emit의 두 번째 인자(data)는 detail로 한 번 더 감싸지 않고
그냥 콜백의 인자로 바로 전달됨 — 이벤트 객체를 거치지 않고 데이터를 직접 주고받는 더 단순한 구조
```

---

# 왜 필요한가 — 같은 프로세스 안에서 느슨하게 연결하기 ⭐️⭐️⭐️

```txt
서버 코드에서도 [[JS_CustomEvent]]와 똑같은 문제가 생김:
  서로 직접 호출하는 관계가 아닌 두 모듈/서비스가, 한쪽에서 일어난 일을
  다른 쪽에 알려야 하는 상황(주문 생성 → 알림 발송, 결제 완료 → 로그 기록 등)

해결 방향도 동일함:
  값이 바뀐 곳 → emit으로 "이런 일이 있었다" 발행
  관심 있는 모듈들 → on으로 구독, 자기 역할만 수행
  → 두 모듈이 서로의 존재를 몰라도 됨 (느슨한 연결)
```

---

# NestJS에서는 — @nestjs/event-emitter ⭐️

```txt
NestJS는 이 패턴을 데코레이터 기반으로 감싼 별도 패키지(@nestjs/event-emitter, 내부적으로
EventEmitter2 기반)를 제공함 — 직접 new EventEmitter()를 안 만들고 모듈로 등록해서 씀
```

```typescript
// 발행 — EventEmitter2를 주입받아 사용
constructor(private eventEmitter: EventEmitter2) {}

this.eventEmitter.emit('user.created', { userId });
```

```typescript
// 구독 — 데코레이터로 표시
@OnEvent('user.created')
handleUserCreated(payload: { userId: string }) {
  // ...
}
```

```txt
기본 발상(emit으로 발행, 구독자가 처리)은 위 vanilla EventEmitter와 동일함 —
NestJS는 그 등록/구독을 DI 컨테이너와 데코레이터로 더 편하게 감싸놓은 것일 뿐
```

---

# JS_CustomEvent와 비교 — 같은 패턴, 다른 환경 ⭐️⭐️

| |[[JS_CustomEvent]] (브라우저)|EventEmitter (Node 서버)|
|---|---|---|
|발행|`window.dispatchEvent(new CustomEvent(...))`|`emitter.emit(name, data)`|
|구독|`window.addEventListener(name, fn)`|`emitter.on(name, fn)`|
|쓰는 곳|클라이언트 컴포넌트 간 동기화|서버 프로세스 내 모듈/서비스 간 동기화|
|범위|그 탭(브라우저)의 window 전역|그 Node 프로세스 내부에서만 (다른 서버 인스턴스와는 공유 안 됨)|

```txt
"다른 서버 인스턴스와는 공유 안 됨"이 중요한 한계:
  서버를 여러 대로 확장(scale-out)하면, 한 인스턴스에서 emit한 이벤트는
  그 인스턴스 안의 구독자만 받음 — 인스턴스 간 이벤트 공유가 필요하다면
  EventEmitter가 아니라 Redis Pub/Sub, 메시지 큐(Kafka 등) 같은 별도 인프라가 필요함
```

---

# 한눈에

| 개념                         | 핵심                                            |
| -------------------------- | --------------------------------------------- |
| `emitter.on(name, fn)`     | 구독 — 브라우저의 addEventListener와 같은 발상            |
| `emitter.emit(name, data)` | 발행 — 브라우저의 dispatchEvent와 같은 발상               |
| `emitter.once(name, fn)`   | 한 번만 실행되고 자동 해제                               |
| `@nestjs/event-emitter`    | NestJS가 데코레이터(@OnEvent)로 감싼 버전, 내부는 같은 패턴     |
| 한계                         | 같은 프로세스 안에서만 동작 — 서버를 여러 대로 늘리면 별도 메시지 브로커 필요 |