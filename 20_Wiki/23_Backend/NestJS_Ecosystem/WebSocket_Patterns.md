---
aliases:
  - 실전 패턴
  - Socket.IO
tags:
  - NestJS
  - NextJS
related:
  - "[[00_JS_Ecosystem_HomePage]]"
  - "[[00_NestJS_Ecosystem_HomePage]]"
  - "[[NestJS_WebSocket]]"
  - "[[NextJS_WebSocket]]"
---
# WebSocket_Patterns — Socket.IO 실전 패턴

> [!info] 서버(NestJS)와 클라이언트(Next.js) 양쪽 코드를 패턴별로 나란히 정리. 개념과 상세 설명 → [[NestJS_WebSocket]] · [[NextJS_WebSocket]]

---

# 패턴 1 — 연결 + 인증 ⭐️⭐️⭐️⭐️

```txt
흐름:
  클라이언트 io(url, { auth: { token } })
  → 서버 handleConnection(client) 자동 실행
  → JWT 검증 → 성공: client.data.userId 저장
  → 실패: client.disconnect()
```

**서버**

```typescript
async handleConnection(client: AuthedSocket) {
  try {
    const token =
      (client.handshake.auth?.token as string | undefined) ??
      (client.handshake.query?.token as string | undefined);

    if (!token) { client.disconnect(); return; }

    const payload = await this.jwtService.verifyAsync<JwtPayload>(token, {
      secret: this.configService.getOrThrow('JWT_SECRET'),
    });

    client.data.userId = payload.sub;
    await client.join(`user:${payload.sub}`);  // 개인 룸 자동 입장
  } catch {
    client.disconnect();
  }
}
```

**클라이언트**

```typescript
let socket: Socket | null = null;

export function getSocket(): Socket {
  if (socket?.connected) return socket;

  const token = getAccessToken();
  if (!token) throw new Error('로그인이 필요합니다.');

  if (!socket) {
    socket = io(`${getApiBaseUrl()}/chat`, {
      autoConnect:     false,
      auth:            { token },
      withCredentials: true,
    });
  } else {
    socket.auth = { token };
  }

  if (!socket.connected) socket.connect();
  return socket;
}

export function disconnectSocket() {
  socket?.disconnect();
  socket = null;
}
```

---

# 패턴 2 — 룸 입장/퇴장 ⭐️⭐️⭐️⭐️

```txt
흐름:
  클라이언트 emit('featureA:join', { resourceId })
  → 서버 @SubscribeMessage('featureA:join')
  → client.join(`room:${resourceId}`)
  → return { ok: true }  (acknowledgement)
  → 클라이언트 콜백으로 { ok: true } 수신
```

**서버**

```typescript
@SubscribeMessage('featureA:join')
async onJoin(
  @ConnectedSocket() client: AuthedSocket,
  @MessageBody() body: { resourceId: string },
): Promise<{ ok: boolean }> {
  const userId = client.data.userId;
  if (!userId || !body?.resourceId) return { ok: false };

  await this.featureService.assertAccess(body.resourceId, userId);
  await client.join(`room:${body.resourceId}`);
  return { ok: true };
}

@SubscribeMessage('featureA:leave')
async onLeave(
  @ConnectedSocket() client: AuthedSocket,
  @MessageBody() body: { resourceId: string },
): Promise<{ ok: boolean }> {
  if (!body?.resourceId) return { ok: false };
  await client.leave(`room:${body.resourceId}`);
  return { ok: true };
}
```

**클라이언트**

```typescript
export function socketJoin(resourceId: string): Promise<{ ok: boolean }> {
  const s = getSocket();
  return new Promise((resolve) => {
    const doEmit = () => {
      s.emit(
        'featureA:join',          // ① 이벤트 이름 (서버와 동일해야 함)
        { resourceId },           // ② payload (@MessageBody로 받는 값)
        (res: { ok: boolean }) => resolve(res ?? { ok: false }),  // ③ acknowledgement
      );
    };
    if (s.connected) doEmit();
    else s.once('connect', doEmit);
  });
}

export function socketLeave(resourceId: string): Promise<{ ok: boolean }> {
  const s = getSocket();
  return new Promise((resolve) => {
    s.emit('featureA:leave', { resourceId }, (res: { ok: boolean }) => resolve(res));
  });
}
```

---

# 패턴 3 — REST 저장 후 WS 브로드캐스트 ⭐️⭐️⭐️⭐️

```txt
흐름:
  클라이언트 POST /resources/:id/items (HTTP)
  → Controller → Service (DB 저장)
  → Gateway.emitToRoom(resourceId, item)
  → server.to(`room:${resourceId}`).emit('item:created', item)
  → 룸 안의 모든 클라이언트 수신
```

**서버 — Gateway에 emit 메서드 추가**

```typescript
@Injectable()
export class SharedGateway {
  @WebSocketServer() server: Server;

  emitToRoom(resourceId: string, event: string, payload: unknown) {
    this.server.to(`room:${resourceId}`).emit(event, payload);
  }

  emitToUser(userId: string, event: string, payload: unknown) {
    this.server.to(`user:${userId}`).emit(event, payload);
  }
}
```

**서버 — Controller에서 호출**

```typescript
@Post(':id/items')
async createItem(
  @UserId() userId: string,
  @Param('id') resourceId: string,
  @Body() dto: CreateItemDto,
) {
  const item = await this.service.createItem(resourceId, userId, dto);  // DB 저장
  this.gateway.emitToRoom(resourceId, 'item:created', item);            // WS 브로드캐스트
  return item;                                                           // HTTP 응답
}
```

**클라이언트 — 이벤트 구독**

```typescript
export function onItemCreated(
  handler: (item: ApiItem) => void,
): () => void {
  const s = getSocket();
  s.on('item:created', handler);
  return () => s.off('item:created', handler);  // cleanup 함수 반환
}

// React 컴포넌트에서 사용
useEffect(() => {
  const off = onItemCreated((item) => {
    setItems((prev) => {
      if (prev.some((i) => i.id === item.id)) return prev;  // 중복 방지
      return [...prev, item];
    });
  });
  return off;  // 언마운트 시 자동 해제
}, []);
```

---

# 패턴 4 — 특정 유저에게만 알림 ⭐️⭐️⭐️⭐️

```txt
흐름:
  서버가 특정 userId만 타겟으로 emit
  → handleConnection에서 user:${userId} 룸에 자동 입장했으므로
  → server.to(`user:${userId}`).emit(...)
  → 그 유저의 모든 탭/기기가 수신
```

**서버**

```typescript
// handleConnection에서 개인 룸 자동 입장 (패턴 1에 포함)
await client.join(`user:${payload.sub}`);

// 특정 유저에게만 emit
emitToUser(userId: string, event: string, payload: unknown) {
  this.server.to(`user:${userId}`).emit(event, payload);
}

// 사용 예 — 강퇴
this.gateway.emitToUser(targetUserId, 'resource:kicked', { resourceId });
```

**클라이언트**

```typescript
export function onKicked(
  handler: (payload: { resourceId: string }) => void,
): () => void {
  const s = getSocket();
  s.on('resource:kicked', handler);
  return () => s.off('resource:kicked', handler);
}

useEffect(() => {
  const off = onKicked(({ resourceId }) => {
    router.replace('/');  // 강퇴 시 메인으로 이동
  });
  return off;
}, [router]);
```

---

# 패턴 5 — 재연결 시 자동 재입장 ⭐️⭐️⭐️

```txt
재연결 시 서버는 소켓이 어느 룸에 있었는지 모름
→ 클라이언트가 'connect' 이벤트에서 다시 join을 보내야 함
```

**클라이언트**

```typescript
export function onSocketConnect(handler: () => void): () => void {
  const s = getSocket();
  s.on('connect', handler);
  return () => s.off('connect', handler);
}

// 컴포넌트에서
useEffect(() => {
  let joined = false;

  // 최초 입장
  void socketJoin(resourceId).then((res) => {
    if (res.ok) joined = true;
  });

  // 재연결 시 자동 재입장
  const offConnect = onSocketConnect(() => {
    void socketJoin(resourceId);
  });

  return () => {
    offConnect();
    if (joined) void socketLeave(resourceId);
  };
}, [resourceId]);
```

---

# 패턴 6 — Gateway 책임 분리 ⭐️⭐️⭐️⭐️

```txt
ModuleA와 ModuleB가 서로 emit이 필요할 때 순환 참조 발생
→ SharedGateway(연결·emit)를 SharedModule로 분리

SharedModule  ← SharedGateway (연결 + emit만)
     ▲                ▲
 ModuleA          ModuleB
(이벤트 A)        (이벤트 B)
```

**서버**

```typescript
// shared/shared.module.ts
@Module({ providers: [SharedGateway], exports: [SharedGateway] })
export class SharedModule {}

// feature-a/feature-a-events.gateway.ts
@WebSocketGateway({ namespace: '/chat' })  // 같은 namespace 공유 가능
export class FeatureAGateway {
  constructor(private readonly sharedGateway: SharedGateway) {}

  @SubscribeMessage('featureA:join')
  async onJoin(@ConnectedSocket() client: AuthedSocket, @MessageBody() body: { resourceId: string }) {
    await client.join(`room:${body.resourceId}`);
    return { ok: true };
  }
}

// feature-a/feature-a.module.ts
@Module({
  imports:     [SharedModule],                           // 순환 없음
  providers:   [FeatureAService, FeatureAGateway],
  controllers: [FeatureAController],
})
export class FeatureAModule {}
```

---

# 룸 네이밍 규칙

|용도|형식|예시|
|---|---|---|
|기능별 그룹|`feature:${resourceId}`|`room:abc123` · `dm:xyz456`|
|개인 알림|`user:${userId}`|`user:user-uuid`|
|관리자|`admin:channel`|`admin:dashboard`|

```txt
prefix를 붙이는 이유:
  resourceId가 같아도 다른 기능이면 다른 룸이어야 함
  `123`(룸) vs `123`(DM) → room:123 · dm:123 으로 구분
  로그에서 `room:abc`만 봐도 어떤 룸인지 바로 파악
```

---

# 이벤트 이름 규칙

```txt
단순 동사          connect · disconnect
리소스:동작        room:updated · item:created · user:kicked

서버 이벤트명 = 클라이언트 이벤트명 (정확히 같아야 함)
  서버: server.to(room).emit('item:created', data)
  클라이언트: socket.on('item:created', handler)
  이름이 다르면 에러 없이 조용히 무시됨
```

---

# 자주 만나는 문제

| 증상                | 원인                     | 해결                                                |
| ----------------- | ---------------------- | ------------------------------------------------- |
| emit이 서버에 안 도달    | 연결 완료 전에 emit          | `s.connected` 확인 + `s.once('connect', doEmit)` 패턴 |
| 재연결 후 이벤트 수신 안 됨  | 재입장 안 함                | `onSocketConnect`에서 re-join                       |
| 같은 메시지 두 번 수신     | 소켓 `on` 중복 등록          | `useEffect` cleanup에서 `off()` 필수                  |
| 강퇴됐는데 클라이언트 반응 없음 | `user:${userId}` 룸 미입장 | `handleConnection`에서 개인 룸 join 확인                 |
| 순환 참조 에러          | 모듈이 서로 import          | SharedModule로 공통 기능 분리                            |