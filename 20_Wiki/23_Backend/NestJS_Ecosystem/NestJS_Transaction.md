---
aliases:
  - Transaction
  - 트랜잭션
tags:
  - NestJS
related:
  - "[[00_NestJS_Ecosystem_HomePage]]"
  - "[[NestJS_Prisma_Patterns]]"
  - "[[PG_Transaction]]"
---
# NestJS_Transaction — Prisma 트랜잭션

> [!info] 
> 트랜잭션 = 여러 DB 쿼리를 하나의 작업 단위로 묶는 것. 하나라도 실패하면 전부 롤백된다. 
> Prisma에서 쓰는 방법은 배치(`$transaction([...])`)와 인터랙티브(`$transaction(async tx => {})`) 두 가지. 
> SQL 레벨 트랜잭션(BEGIN · SAVEPOINT · Isolation Level · DEADLOCK) → [[PG_Transaction]]

---

# 트랜잭션이 필요한 이유 ⭐️⭐️⭐️

```typescript
// ❌ 트랜잭션 없음 — 두 번째 쿼리에서 에러나면 데이터 불일치
await prisma.order.create({ data: { userId, productId } });
await prisma.product.update({
  where: { id: productId },
  data:  { stock: { decrement: 1 } },
});
// 주문은 생성됐는데 재고는 안 줄어든 상태가 될 수 있음

// ✅ 트랜잭션 — 둘 다 성공하거나 둘 다 롤백
await prisma.$transaction(async (tx) => {
  await tx.order.create({ data: { userId, productId } });
  await tx.product.update({
    where: { id: productId },
    data:  { stock: { decrement: 1 } },
  });
});
```

```txt
트랜잭션의 핵심 성질 — All or Nothing:
  여러 쿼리가 하나의 논리적 단위로 묶임
  중간에 하나라도 실패(throw)하면 이미 실행된 쿼리들까지 전부 되돌아감
  성공하면 COMMIT, 실패하면 ROLLBACK

ACID 개념(원자성 · 일관성 · 격리성 · 지속성) → [[DB_Transaction]]
```

---

# Connection과 트랜잭션의 관계 — tx를 반드시 써야 하는 이유 ⭐️⭐️⭐️⭐️

```txt
DB 트랜잭션은 하나의 Connection 위에서 동작한다

일반 prisma 쿼리의 흐름:
  Connection Pool에서 쿼리마다 Connection을 빌림
  → 쿼리 실행
  → Connection 반납
  쿼리가 각자 다른 Connection을 쓸 수 있음

트랜잭션의 흐름:
  Connection Pool에서 Connection 하나를 "점유"
  → 이 Connection 위에서 BEGIN 실행
  → 모든 쿼리를 이 Connection 위에서 실행
  → COMMIT 또는 ROLLBACK
  → Connection 반납

핵심: 트랜잭션 안의 모든 쿼리가 같은 Connection을 공유해야
      DB가 "이것들이 한 묶음"이라고 인식하고 함께 롤백할 수 있음
```

```typescript
await prisma.$transaction(async (tx) => {
  // tx = 트랜잭션 전용 Connection을 래핑한 Prisma 클라이언트
  await tx.order.create(...);    // ✅ 같은 Connection
  await tx.product.update(...);  // ✅ 같은 Connection

  await prisma.log.create(...);  // ❌ prisma = 다른 Connection에서 실행
                                 //    트랜잭션 밖 → 롤백되지 않음
});
```

```txt
prisma vs tx:
  prisma  — Connection Pool에서 임의의 Connection을 빌려 실행
  tx      — 트랜잭션을 위해 점유한 특정 Connection 위에서 실행

  콜백 안에서 prisma를 쓰면 다른 Connection에서 실행되어
  트랜잭션이 롤백될 때 함께 되돌아가지 않음
```

---

# 방식 1 — 배치 `$transaction([...])` ⭐️⭐️⭐️

```typescript
const [order, updatedProduct] = await prisma.$transaction([
  prisma.order.create({ data: { userId, productId } }),
  prisma.product.update({
    where: { id: productId },
    data:  { stock: { decrement: 1 } },
  }),
]);
// 반환값은 배열 — 넘긴 순서대로 구조분해로 받음
```

```txt
동작 방식:
  Prisma 작업들을 배열로 미리 정의해서 넘김
  내부적으로 하나의 Connection에서 BEGIN → 순서대로 실행 → COMMIT
  하나라도 실패하면 ROLLBACK

특징:
  작업들이 실행 전에 전부 정의되어 있어야 함 (런타임 분기 불가)
  앞 작업의 결과를 다음 작업에 활용하는 것도 불가
  불필요한 결과는 쉼표로 건너뜀 → [[JS_Operators]] 배열 구조분해 skip 참고
```

```typescript
// 필요한 결과만 구조분해
const [, , updatedRoom] = await this.prisma.$transaction([
  this.prisma.roomMember.delete({ ... }),  // [0] 필요 없음
  this.prisma.log.create({ ... }),         // [1] 필요 없음
  this.prisma.room.update({ ... }),        // [2] 이 결과만 필요
]);
```

## ⚠️ 배치 + include → pg client 경고

```typescript
// ❌ 배치 형태 + include — PostgreSQL 어댑터에서 경고 발생 가능
const [, , , message] = await this.prisma.$transaction([
  this.prisma.roomBan.upsert({ ... }),
  this.prisma.roomMember.delete({ ... }),
  this.prisma.room.update({ ... }),
  this.prisma.roomMessage.create({
    data:    { ... },
    include: messageInclude,  // ← include 있으면 Prisma가 추가 쿼리를 내부적으로 실행
  }),
]);
```

```txt
왜 경고가 나는가:
  배치 $transaction은 각 쿼리를 독립적으로 보내면서 트랜잭션으로 묶는 방식
  include가 붙으면 Prisma가 내부적으로 추가 SELECT를 실행 (JOIN 대신 별도 쿼리)
  → pg 어댑터에서 "중첩 client.query" 경고 발생

  기능은 작동하지만 콘솔에 경고 로그가 쌓임
  → include가 하나라도 있으면 인터랙티브 형태 사용 권장
```

---

# 방식 2 — 인터랙티브 `$transaction(async tx => {})` ⭐️⭐️⭐️⭐️

```typescript
await prisma.$transaction(async (tx) => {
  // 읽고
  const product = await tx.product.findUnique({ where: { id: productId } });

  // 검증하고 (실패 시 throw → 자동 롤백)
  if (!product || product.stock <= 0) {
    throw new BadRequestException('재고 없음');
  }

  // 쓰기
  await tx.order.create({ data: { userId, productId } });
  await tx.product.update({
    where: { id: productId },
    data:  { stock: { decrement: 1 } },
  });
});
```

```txt
throw → 자동 롤백의 원리:
  Prisma가 콜백을 try/catch로 감싸고 있음
  throw가 발생하면 Prisma가 ROLLBACK을 실행한 뒤 예외를 다시 던짐
  → 명시적으로 ROLLBACK을 작성하지 않아도 됨
  → NestJS HTTP 예외(NotFoundException 등)를 그대로 throw해도 롤백됨

특징:
  읽기 → 조건 분기 → 쓰기가 모두 같은 Connection 안에서 이루어짐
  include 포함 모든 쿼리 안전하게 사용 가능
  마지막 return 값이 $transaction 전체의 반환값
```

## NestJS Service에서 사용 패턴

```typescript
@Injectable()
export class OrderService {
  constructor(private readonly prisma: PrismaService) {}

  async createOrder(userId: number, productId: number) {
    return this.prisma.$transaction(async (tx) => {
      const product = await tx.product.findUnique({
        where:  { id: productId },
        select: { id: true, stock: true, price: true },
      });

      if (!product)          throw new NotFoundException('상품을 찾을 수 없습니다.');
      if (product.stock <= 0) throw new BadRequestException('재고가 없습니다.');

      const order = await tx.order.create({
        data: { userId, productId, amount: product.price },
      });

      await tx.product.update({
        where: { id: productId },
        data:  { stock: { decrement: 1 } },
      });

      return order;
    });
  }
}
```

## include와 함께 — 인터랙티브가 필요한 이유

```typescript
// ✅ include + 복잡한 쿼리는 인터랙티브로
return this.prisma.$transaction(async (tx) => {
  await tx.roomBan.upsert({ ... });
  await tx.roomMember.delete({ ... });
  await tx.room.update({
    where: { id: roomId },
    data:  { memberCount: { decrement: 1 } },
  });
  return tx.roomMessage.create({
    data:    { ... },
    include: messageInclude,  // ✅ tx 안에서 include 사용해도 경고 없음
  });
});
```

---

# 배치 vs 인터랙티브 — 선택 기준 ⭐️⭐️⭐️

| |배치 `$transaction([...])`|인터랙티브 `$transaction(async tx => {})`|
|---|---|---|
|작업 간 의존성|없음|있음 (앞 결과로 뒤를 결정)|
|조건 분기|불가|가능|
|include 사용|경고 발생 가능|안전|
|코드 복잡도|낮음|높음|
|결과 받는 방법|배열 구조분해|마지막 return 값|

```txt
include가 하나라도 있으면 → 인터랙티브
중간에 조건 분기가 필요하면 → 인터랙티브
단순 INSERT/UPDATE/DELETE 묶음 + include 없으면 → 배치도 가능
```

---

# 주의사항 — Connection 점유 ⭐️⭐️

```txt
인터랙티브 트랜잭션은 실행 중에 Connection을 점유한다
콜백이 끝날 때까지 해당 Connection은 다른 쿼리에 쓰일 수 없음

Connection Pool 고갈 위험:
  트랜잭션 안에서 시간이 오래 걸리는 작업을 하면
  → Connection을 그 시간만큼 점유
  → 동시 요청이 많을 때 Pool의 모든 Connection이 트랜잭션에 묶일 수 있음

⚠️ 트랜잭션 안에서 하면 안 되는 것:
  - 외부 API 호출 (HTTP 요청, 이메일 전송 등)
  - 무거운 계산 작업
  - 파일 I/O

올바른 패턴:
  필요한 DB 작업만 트랜잭션 안에 넣고
  외부 연동(이메일, 이벤트 발행 등)은 트랜잭션 성공 후 바깥에서 실행
```

```typescript
// ✅ 외부 연동은 트랜잭션 바깥에서
async createOrder(userId: number, productId: number) {
  const order = await this.prisma.$transaction(async (tx) => {
    // DB 작업만
    const result = await tx.order.create({ ... });
    await tx.product.update({ ... });
    return result;
  });

  // 트랜잭션 완료 후 이메일 발송
  await this.emailService.sendOrderConfirmation(order);

  return order;
}
```

---

# 격리 수준 / 타임아웃 옵션 ⭐️⭐️

```typescript
await prisma.$transaction(async (tx) => {
  // ...
}, {
  isolationLevel: Prisma.TransactionIsolationLevel.Serializable,
  timeout:        5000,   // ms — 트랜잭션 최대 실행 시간 (초과 시 롤백)
  maxWait:        2000,   // ms — Connection Pool에서 대기하는 최대 시간
});
```

```txt
Prisma.TransactionIsolationLevel:
  ReadUncommitted  /  ReadCommitted (기본)  /  RepeatableRead  /  Serializable

언제 격리 수준을 올리는가:
  RepeatableRead   트랜잭션 안에서 같은 행을 두 번 읽는데 일관성이 필요할 때
  Serializable     완벽한 직렬화가 필요한 금융 계산 등 (성능 트레이드오프 있음)

격리 수준 이상 현상(Dirty Read · Non-Repeatable Read · Phantom Read) → [[PG_Transaction]]
```

---

# 비관적 잠금 — FOR UPDATE ⭐️⭐️⭐️

```txt
인터랙티브 트랜잭션만으로는 동시 요청 방어에 한계가 있다:
  요청 A, 요청 B가 거의 동시에 도착
  A가 product 읽음 → stock: 1 → 통과
  B가 product 읽음 → stock: 1 → 통과  (A가 아직 update 안 했을 때)
  A, B 모두 update → stock이 -1이 됨

FOR UPDATE를 추가하면:
  A가 SELECT ... FOR UPDATE → 해당 행을 잠금
  B가 같은 행에 SELECT ... FOR UPDATE 시도 → A가 끝날 때까지 대기
  A가 update + COMMIT → B가 잠금 획득 → stock 다시 확인 → 0이면 에러
```

```typescript
// Prisma ORM은 FOR UPDATE를 직접 지원하지 않음 → $queryRaw 사용
await prisma.$transaction(async (tx) => {
  const [product] = await tx.$queryRaw<{ id: number; stock: number }[]>`
    SELECT id, stock FROM "Product"
    WHERE id = ${productId}
    FOR UPDATE
  `;

  if (product.stock <= 0) throw new BadRequestException('재고 없음');

  await tx.product.update({
    where: { id: productId },
    data:  { stock: { decrement: 1 } },
  });
});
```

```txt
비관적 잠금 vs 낙관적 잠금 선택 기준:
  충돌이 드물다  → 낙관적 잠금 (version 필드, 충돌 시 재시도)
  충돌이 빈번하다 → 비관적 잠금 (FOR UPDATE, 순서 보장)
  → [[NestJS_Idempotency]] 참고

FOR UPDATE의 DEADLOCK 위험 → [[PG_Transaction]] DEADLOCK 섹션
```

---

# 에러 처리 ⭐️⭐️⭐️

```typescript
async createOrder(userId: number, productId: number) {
  try {
    return await this.prisma.$transaction(async (tx) => {
      const product = await tx.product.findUnique({ where: { id: productId } });
      if (!product) throw new NotFoundException('상품 없음');
      // ...
    });
  } catch (e) {
    if (e instanceof Prisma.PrismaClientKnownRequestError) {
      if (e.code === 'P2002') throw new ConflictException('중복 요청입니다.');
      if (e.code === 'P2025') throw new NotFoundException('데이터를 찾을 수 없습니다.');
      if (e.code === 'P2034') {
        // Serializable 충돌 → 재시도 필요
        throw new ConflictException('동시 요청 충돌. 다시 시도해 주세요.');
      }
    }
    if (e instanceof HttpException) throw e;  // 직접 throw한 NestJS 예외는 그대로
    throw new InternalServerErrorException('처리 중 오류가 발생했습니다.');
  }
}
```

|코드|의미|보통 던지는 예외|
|---|---|---|
|`P2002`|Unique constraint 위반|`ConflictException`|
|`P2025`|대상 없음 (update/delete할 행이 없음)|`NotFoundException`|
|`P2034`|Serializable 격리 수준에서 트랜잭션 충돌|`ConflictException` + 재시도 안내|

```txt
P2034가 발생하는 이유:
  Serializable 격리 수준은 직렬화 가능성을 보장하기 위해
  "이 트랜잭션을 직렬로 실행했을 때와 결과가 다를 것 같다"고 판단하면 에러를 던짐
  → catch에서 P2034를 잡아 재시도 로직 구현 또는 클라이언트에 재시도 안내

throw → 자동 롤백 흐름:
  콜백 안에서 throw
  → Prisma가 ROLLBACK 실행
  → 예외가 $transaction 밖으로 전파
  → 바깥 try/catch에서 잡힘
```

---

# 자주 만나는 에러

| 증상              | 원인                         | 해결                      |
| --------------- | -------------------------- | ----------------------- |
| 롤백이 안 됨         | 콜백 안에서 `prisma` 사용 (tx 아님) | `prisma` → `tx` 로 교체    |
| include 시 콘솔 경고 | 배치 형태에서 include 사용         | 인터랙티브 형태로 전환            |
| P2034 에러        | Serializable 격리 수준 충돌      | catch에서 P2034 잡아 재시도 처리 |
| Connection 고갈   | 트랜잭션 안에서 외부 API 호출         | 외부 연동은 트랜잭션 바깥으로 이동     |
| 트랜잭션 타임아웃       | 콜백 안에 무거운 작업               | timeout 옵션 조정 또는 작업 분리  |