---
aliases:
  - Gateway
  - SocketIO
  - WebSocket
  - emit
  - leave
  - disconnect
  - token
tags:
  - NestJS
related:
  - "[[00_NestJS_Ecosystem_HomePage]]"
  - "[[NextJS_WebSocket]]"
  - "[[NestJS_CORS]]"
  - "[[TS_Generics]]"
  - "[[NestJS_Module]]"
---
# NestJS_WebSocket — Socket.IO Gateway

>[!info]
>Gateway = WebSocket 이벤트를 처리하는 Controller.
> `@SubscribeMessage`가 `@Get/@Post` 역할, `@ConnectedSocket`이 `@Req` 역할을 한다. 
> 연결한 클라이언트 하나 = Socket 객체.
>  클라이언트(Next.js) 구현 → [[NextJS_WebSocket]]

---

# Gateway란 — WebSocket의 Controller ⭐️⭐️⭐️⭐️

```txt
HTTP Controller와 Gateway를 나란히 보면:

  HTTP Controller:
    @Controller('rooms')         → 어떤 URL 그룹인가
    @Get(':id')                  → 어떤 HTTP 메서드 + 경로인가
    @Param('id') roomId          → URL 파라미터 꺼내기
    @Body() dto                  → 요청 body 꺼내기
    @Req() req                   → 요청 객체 전체

  WebSocket Gateway:
    @WebSocketGateway({ namespace: '/chat' })  → 어떤 네임스페이스인가
    @SubscribeMessage('join')                  → 어떤 이벤트인가
    @MessageBody() body                        → 이벤트 payload 꺼내기
    @ConnectedSocket() client                  → 이 연결의 소켓 객체 전체

  Controller에서 @Req()로 요청 정보를 꺼내듯
  Gateway에서 @ConnectedSocket()으로 클라이언트 소켓을 꺼냄
```

```typescript
@WebSocketGateway({ namespace: '/chat', cors: { origin: true, credentials: true } })
export class RoomsGateway implements OnGatewayConnection {

  @WebSocketServer()             // 서버 인스턴스 — 모든 클라이언트에게 emit 가능
  server: Server;

  // 클라이언트가 연결될 때 자동 실행 (Controller에는 없는 WebSocket 전용 생명주기)
  async handleConnection(client: AuthedSocket) { ... }

  // HTTP의 @Get()처럼 특정 이벤트를 처리
  @SubscribeMessage('join')
  async onJoin(
    @ConnectedSocket() client: AuthedSocket,  // @Req() 역할
    @MessageBody() body: { roomId: string },  // @Body() 역할
  ) {
    ...
    return { ok: true };  // acknowledgement — HTTP 응답처럼 클라이언트에 전달
  }
}
```

---

# 설치

```bash
pnpm add @nestjs/websockets @nestjs/platform-socket.io socket.io
```

---

# CORS 설정 ⭐️⭐️⭐️

```typescript
// 개발 환경 — origin: true (모든 출처 허용, 간단)
@WebSocketGateway({
  namespace: '/chat',
  cors: { origin: true, credentials: true },
})

// 운영 환경 권장 — 허용 출처 명시
@WebSocketGateway({
  namespace: '/chat',
  cors: {
    origin: [
      'http://localhost:3031',    // 로컬 개발 — localhost
      'http://127.0.0.1:3031',   // 로컬 개발 — 127.0.0.1 (브라우저마다 다르게 처리)
      process.env.FRONTEND_URL
        ? new URL(process.env.FRONTEND_URL).origin
        : undefined,
    ].filter(Boolean),  // undefined 제거
    credentials: true,
  },
})
```

```txt
localhost vs 127.0.0.1 둘 다 넣는 이유:
  localhost와 127.0.0.1은 같은 IP를 가리키지만 브라우저는 다른 origin으로 취급
  http://localhost:3031  ≠  http://127.0.0.1:3031  (다른 출처)
  어떤 브라우저/환경은 localhost를, 어떤 것은 127.0.0.1을 사용하는 경우가 있음
  → 둘 다 허용해야 로컬 개발 시 CORS 에러 없음

origin: true  vs  origin: [배열]:
  true  → 어떤 출처에서든 허용 (개발 시 편하지만 운영에선 위험)
  [배열] → 허용 출처 명시 (운영 권장)

new URL(process.env.FRONTEND_URL).origin:
  'https://my-app.vercel.app/some/path' → 'https://my-app.vercel.app'
  경로가 포함된 URL에서 origin(프로토콜+도메인+포트)만 추출
  환경변수에 경로가 포함돼 있어도 정확한 origin을 얻을 수 있음

.filter(Boolean):
  FRONTEND_URL 없으면 ternary가 undefined 반환
  배열에 undefined 있으면 에러 → filter(Boolean)으로 제거

credentials: true:
  쿠키 포함 요청 허용 — 클라이언트 socket.io의 withCredentials: true 와 세트
  이 설정 없이 withCredentials: true 를 클라이언트에서 쓰면 CORS 에러
```

---

# 연결 흐름 — 전체 생명주기 ⭐️⭐️⭐️⭐️

```txt
① 클라이언트가 io('http://localhost:3000/chat', { auth: { token } }) 호출
   → TCP 연결 시작

② handleConnection(client) 자동 실행
   → 토큰 검증
   → 성공: client.data.userId = payload.sub
   → 실패: client.disconnect() (연결 강제 종료)

③ 클라이언트가 socket.emit('join', { roomId }) 호출
   → @SubscribeMessage('join') 핸들러 실행
   → client.join('room:xxx') (이 소켓이 그 룸에 입장)
   → return { ok: true } (acknowledgement 응답)

④ 누군가 REST로 메시지 전송
   → Controller → Service(DB 저장) → Gateway.emitMessage()
   → server.to('room:xxx').emit('message', data)
   → 룸 안의 모든 소켓이 수신

⑤ 클라이언트가 socket.emit('leave', { roomId }) 또는 연결 종료
   → client.leave('room:xxx')
   → handleDisconnect (선택적)
```

---

# Socket 객체 — 클라이언트 연결 하나 ⭐️⭐️⭐️⭐️

```txt
Socket = 서버에 연결된 클라이언트 하나를 나타내는 객체
  HTTP의 Request 객체와 비슷한 역할
  이 연결의 정보에 접근하고, 이 연결로 메시지를 보냄
```

```typescript
client.id          // 서버가 부여한 고유 ID (재연결 시 바뀜)
client.handshake   // 연결 시 클라이언트가 보낸 정보 (토큰, 헤더 등)
client.data        // 이 소켓에 자유롭게 저장하는 공간 (userId 등)
client.rooms       // 현재 입장한 룸 Set
client.connected   // 연결 상태 boolean
```

## handshake — 연결 시 받는 정보

```typescript
client.handshake.auth     // io(url, { auth: { token } }) 로 보낸 것
client.handshake.query    // io(url + '?token=...')로 보낸 것
client.handshake.headers  // HTTP 헤더
client.handshake.address  // 클라이언트 IP

// 토큰 추출 — auth 우선, 없으면 query에서
const token =
  (client.handshake.auth?.token as string | undefined) ??
  (client.handshake.query?.token as string | undefined);
```

```txt
왜 auth와 query 둘 다 체크하는가:
  브라우저 WebSocket은 HTTP Authorization 헤더를 못 보냄
  io(url, { auth: { token } }) → handshake.auth.token
  일부 환경(WebView 등) → 쿼리스트링으로 보낼 수 있음
  → 두 곳 모두 확인해서 어느 방식이든 지원
```

## AuthedSocket — 타입 확장 ⭐️⭐️⭐️

```typescript
import { Socket } from 'socket.io';

// Socket.data는 기본적으로 any 타입
// 교차 타입으로 data에 구체적인 타입 추가
type AuthedSocket = Socket & { data: { userId?: string } };
```

```txt
Socket & { data: { userId?: string } }:
  Socket의 모든 기능 + data.userId?: string 타입이 추가된 객체

왜 필요한가:
  기본 Socket 타입에서 client.data.userId는 any
  AuthedSocket을 쓰면 string | undefined로 명확해짐
  → 핸들러에서 타입 자동완성 + null 체크 강제

  userId? (optional)인 이유:
  handleConnection에서 검증 실패 시 disconnect하지만
  혹시 모를 경우를 위해 optional로 두고 핸들러에서 방어 코드 작성
```

---

# handleConnection — 연결 시 인증 ⭐️⭐️⭐️⭐️

```typescript
async handleConnection(client: AuthedSocket) {
  try {
    // ① 토큰 추출
    const token =
      (client.handshake.auth?.token as string | undefined) ??
      (client.handshake.query?.token as string | undefined);

    if (!token) { client.disconnect(); return; }

    // ② JWT 검증 — verifyAsync는 비동기, async 함수 안에서 자연스럽게 사용
    const payload = await this.jwtService.verifyAsync<JwtPayload>(token, {
      secret: this.configService.getOrThrow('JWT_SECRET'),
    });

    // ③ 검증 성공 → 이후 핸들러에서 꺼내 쓸 수 있도록 저장
    client.data.userId = payload.sub;

  } catch {
    // ④ 검증 실패 → 연결 강제 종료
    client.disconnect();
  }
}
```

```txt
handleConnection이 실행되는 시점:
  클라이언트가 연결하는 순간 자동 실행
  HTTP Controller의 어떤 미들웨어보다도 먼저 실행되는 "첫 번째 문"
  여기서 인증을 통과한 소켓만 이후 @SubscribeMessage 핸들러에 도달 가능

client.disconnect():
  실패 시 즉시 연결 종료
  클라이언트는 connect_error 이벤트를 받음

verifyAsync vs verify:
  verify  = 동기 (블로킹) — async 함수 안에서는 불필요한 블로킹
  verifyAsync = 비동기 → handleConnection이 async이므로 자연스럽게 사용
```

---

# 룸 — 그룹에게 메시지 보내기 ⭐️⭐️⭐️⭐️

```txt
룸(Room) = 소켓들의 그룹
  특정 룸에 join한 소켓들에게 한 번에 emit 가능
  채팅방 = 하나의 룸
  어드민 채널, 개인 알림 등도 각각 다른 룸으로 관리
```

```typescript
// 룸 입장
await client.join('room:abc');

// 룸 퇴장
await client.leave('room:abc');

// 현재 입장한 룸 확인
client.rooms  // Set {'socketId', 'room:abc', 'room:xyz'}
              // 자기 자신의 socketId는 항상 포함됨
```

## 룸 네이밍 — prefix 관행 ⭐️⭐️⭐️

```typescript
// ❌ ID만으로 룸 이름 사용 — 충돌 위험
client.join('123');  // roomId 123? userId 123? 알 수 없음

// ✅ prefix로 용도 구분
client.join(`room:${roomId}`);    // 채팅방
client.join(`user:${userId}`);    // 개인 알림
client.join('admin:dashboard');   // 관리자 채널
```

```txt
prefix를 붙이는 이유:
  roomId '123'과 userId '123'이 같을 수 있음 → 다른 목적인데 같은 룸 이름
  prefix로 용도를 구분하면 충돌 없음
  로그에서도 'room:123'만 봐도 채팅방임을 바로 알 수 있음
```

---

# 이벤트 발신 패턴 ⭐️⭐️⭐️⭐️

```typescript
// 이 클라이언트에게만
client.emit('event', data);

// 같은 룸, 발신자 제외
client.to(`room:${roomId}`).emit('event', data);

// 같은 룸 전체 (발신자 포함)
this.server.to(`room:${roomId}`).emit('event', data);

// 특정 소켓 ID에게
this.server.to(socketId).emit('event', data);

// 모든 클라이언트 (전체 브로드캐스트)
this.server.emit('event', data);
```

```txt
client.to vs server.to:
  client.to → 발신자 자신 제외 (내가 방금 보낸 메시지를 나는 다시 안 받음)
  server.to → 발신자 포함 전체 (REST에서 브로드캐스트할 때)

  채팅 메시지를 REST로 저장 후 브로드캐스트할 때는 server.to 사용
  → "나도 포함해서 방 전체에 새 메시지 알림"
```

---

# payload — emit의 두 번째 인자 ⭐️⭐️⭐️

```typescript
// server.to(room).emit(이벤트명, payload)
//                              ↑ 클라이언트에 전달되는 실제 데이터

emitMessageReaction(
  roomId: string,
  payload: {
    messageId: string;
    userId:    string;
    emoji:     string;
    removed:   boolean;  // true = 삭제됨, false = 추가됨
  },
) {
  this.server.to(`room:${roomId}`).emit('message:reaction', payload);
}
```

```txt
payload = emit의 두 번째 인자 = REST 응답 body와 같은 역할, 소켓으로 전달

REST → WS 반응 토글 흐름:
  ① 클라이언트 → POST /reaction
  ② 서버 → DB 토글 → { messageId, userId, emoji, removed } 반환
  ③ Controller → gateway.emitMessageReaction(roomId, result)
  ④ Gateway → server.to('room:xxx').emit('message:reaction', payload)
  ⑤ 방 안의 모든 클라이언트 → 'message:reaction' 수신
  ⑥ 클라이언트 → applyReactionLocal(payload) 로 state 직접 수정

removed 필드:
  클라이언트가 이모지를 추가할지/제거할지 알 수 있도록
  → [[NestJS_Prisma_Patterns]] 토글 패턴
  → [[React_AsyncUI]] applyLocal 패턴
```

---

# @SubscribeMessage — 이벤트 핸들러 ⭐️⭐️⭐️⭐️

```typescript
@SubscribeMessage('join')
async onJoin(
  @ConnectedSocket() client: AuthedSocket,
  @MessageBody() body: { roomId: string },
): Promise<{ ok: boolean }> {

  const userId = client.data.userId;
  if (!userId || !body?.roomId) return { ok: false };

  // 멤버인지 확인
  await this.roomsService.assertMember(body.roomId, userId);

  // 룸 입장
  await client.join(`room:${body.roomId}`);

  return { ok: true };  // acknowledgement — 클라이언트 콜백으로 전달
}

@SubscribeMessage('leave')
async onLeave(
  @ConnectedSocket() client: AuthedSocket,
  @MessageBody() body: { roomId: string },
): Promise<{ ok: boolean }> {
  if (!body?.roomId) return { ok: false };
  await client.leave(`room:${body.roomId}`);
  return { ok: true };
}
```

## acknowledgement — 이벤트에 대한 응답 ⭐️⭐️⭐️

```txt
HTTP에서 Controller가 응답 body를 반환하듯
@SubscribeMessage 핸들러가 return한 값이 클라이언트에게 전달됨 = acknowledgement

없으면: 클라이언트가 성공/실패 여부를 알 수 없음
있으면: 클라이언트 emit의 콜백으로 전달됨
```

```typescript
// 클라이언트에서 acknowledgement 받는 방법
socket.emit('join', { roomId: 'abc' }, (response) => {
  //                                    ↑ 서버의 return 값이 여기로 옴
  if (response.ok) console.log('입장 성공');
  else console.log('입장 실패');
});
```

---

# REST + WebSocket 브로드캐스트 연결 ⭐️⭐️⭐️⭐️

```txt
가장 흔한 패턴:
  POST /rooms/:id/messages (REST)
  → DB에 메시지 저장
  → 같은 방의 WebSocket 클라이언트들에게 새 메시지 브로드캐스트

이 패턴이 필요한 이유:
  DB 저장은 REST (신뢰성, 검증, 트랜잭션)
  실시간 전달은 WebSocket
  둘을 같이 써야 "저장도 되고 즉시 보이기도 함"
```

## 순환 참조 문제와 해결 ⭐️⭐️⭐️

```txt
❌ 순환 참조 발생:
  RoomsGateway → RoomsService (join 시 멤버 확인)
  RoomsService → RoomsGateway (저장 후 emit)
  → 서로를 주입하면 Circular Dependency 에러

✅ 해결 — Controller가 중계:
  Gateway  →  Service  (단방향)
  Controller  →  Service  (저장)
              →  Gateway  (emit)
  Controller에서 두 가지를 순서대로 호출
```

## Gateway — emit 메서드 추가

```typescript
@WebSocketGateway({ namespace: '/chat', cors: { origin: true, credentials: true } })
export class RoomsGateway implements OnGatewayConnection {

  @WebSocketServer()
  server: Server;

  // ... handleConnection, @SubscribeMessage ...

  // Controller에서 호출할 emit 메서드들
  emitMessage(roomId: string, message: unknown) {
    this.server.to(`room:${roomId}`).emit('message', message);
  }

  emitMessageDeleted(roomId: string, messageId: string) {
    this.server.to(`room:${roomId}`).emit('message:deleted', { messageId });
  }

  emitRoomUpdated(roomId: string, payload: RoomUpdatedPayload) {
    this.server.to(`room:${roomId}`).emit('room:updated', payload);
  }
}
```

```txt
Gateway에 emit 메서드를 래핑하는 이유:
  Controller가 이벤트 이름('message', 'room:updated')을 직접 알 필요 없음
  이벤트 이름이 바뀌어도 Gateway 안에서만 수정
  의미 있는 메서드 이름(emitMessage, emitRoomUpdated)으로 의도 명확
```

## Controller — REST 처리 후 브로드캐스트

```typescript
@Controller('rooms')
export class RoomsController {
  constructor(
    private readonly roomsService:  RoomsService,
    private readonly roomsGateway:  RoomsGateway,  // Gateway 주입
  ) {}

  @Post(':id/messages')
  async sendMessage(
    @UserId() userId: string,
    @Param('id', ParseUUIDPipe) roomId: string,
    @Body() dto: CreateRoomMessageDto,
  ) {
    // ① DB 저장
    const message = await this.roomsService.createMessage(roomId, userId, dto);

    // ② WebSocket 브로드캐스트
    this.roomsGateway.emitMessage(roomId, message);

    return message;  // ③ HTTP 응답
  }

  @Delete(':id/messages/:messageId')
  async deleteMessage(
    @UserId() userId: string,
    @Param('id', ParseUUIDPipe) roomId: string,
    @Param('messageId', ParseUUIDPipe) messageId: string,
  ) {
    await this.roomsService.deleteMessage(messageId, userId);
    this.roomsGateway.emitMessageDeleted(roomId, messageId);
    return { ok: true };
  }
}
```

## 서비스에서 emit ⭐️⭐️⭐️⭐️

```typescript
// rooms.service.ts — Gateway를 직접 주입해서 emit
@Injectable()
export class RoomsService {
  constructor(
    private readonly prisma:   PrismaService,
    private readonly gateway:  RoomsGateway,  // 서비스가 Gateway 앎
  ) {}

  async createMessage(roomId: string, userId: string, dto: CreateRoomMessageDto) {
    const message = await this.prisma.message.create({ data: { roomId, userId, ...dto } });
    this.gateway.emitMessage(roomId, message);  // DB 커밋 직후 바로 emit
    return message;
  }

  // 컨트롤러를 타지 않는 cron도 같은 메서드로 emit 가능
  async closeForNight() {
    const updated = await this.prisma.room.updateMany({ data: { status: 'closed' } });
    this.gateway.emitRoomClosed();  // cron에서도 정상 방송
  }
}

// rooms.controller.ts — 얇게 유지
@Controller('rooms')
export class RoomsController {
  constructor(private readonly roomsService: RoomsService) {}

  @Post(':id/messages')
  sendMessage(@UserId() userId: string, @Param('id') roomId: string, @Body() dto: CreateRoomMessageDto) {
    return this.roomsService.createMessage(roomId, userId, dto);
    // emit은 서비스 안에서 처리 — 컨트롤러는 한 줄
  }
}
```

```txt
컨트롤러에서 emit vs 서비스에서 emit:

  컨트롤러에서 emit:
    DB 저장 → 방송 → 응답 흐름이 한눈에 보임
    컨트롤러가 Gateway도 알아야 함
    cron 등 컨트롤러를 안 타는 경우 → emit 누락 가능
    같은 emit 로직을 여러 컨트롤러에서 반복

  서비스에서 emit:
    DB 커밋 직후, 성공한 경우만 방송
    cron·REST 둘 다 같은 서비스 메서드를 탐 → emit 통일
    컨트롤러가 얇게 유지됨 (한 줄 호출)
    단점: 서비스가 Gateway(인프라)를 앎 → 의존성 방향 주의

  선택 기준:
    cron이나 이벤트에서도 같은 emit이 필요하면 → 서비스
    REST만 있고 흐름을 명확히 보여주고 싶으면 → 컨트롤러
```

---

# 특정 유저 타겟팅 ⭐️⭐️⭐️⭐️

```txt
문제: "강퇴된 사람에게만" 알림을 보내야 할 때
  server.to(userId) 는 동작 안 함 — socket.id와 userId는 다른 것
  → 연결된 소켓 중 data.userId가 일치하는 것을 찾아야 함
```

## 방법 1 — sockets 순회

```typescript
emitMemberKicked(roomId: string, targetUserId: string) {
  for (const socket of this.server.sockets.sockets.values()) {
    const client = socket as AuthedSocket;
    if (client.data.userId === targetUserId) {
      void client.leave(`room:${roomId}`);
      client.emit('room:kicked', { roomId });
    }
  }
}
```

```txt
this.server.sockets.sockets:
  현재 연결된 모든 소켓의 Map<socketId, Socket>
  .values() → 모든 소켓 인스턴스 이터레이터

  Gateway에 namespace가 있으면 this.server는 그 네임스페이스 기준
  → 다른 네임스페이스의 소켓은 포함 안 됨

한 유저가 여러 탭/기기로 접속하면:
  같은 userId의 소켓이 여러 개 존재
  → for...of 순회로 전부 처리됨
```

## 방법 2 — user 룸 패턴 (권장) ⭐️⭐️⭐️

```typescript
// handleConnection에서 개인 룸 입장
async handleConnection(client: AuthedSocket) {
  // ... JWT 검증 ...
  client.data.userId = payload.sub;
  await client.join(`user:${payload.sub}`);  // 개인 룸 자동 입장
}

// 이후 순회 없이 한 줄로 전송
emitMemberKicked(roomId: string, targetUserId: string) {
  this.server.to(`user:${targetUserId}`).emit('room:kicked', { roomId });
}
```

```txt
순회 vs user 룸:
  sockets 순회  → 추가 설정 불필요, 접속자 많으면 매번 전체 순회
  user 룸       → handleConnection에 join 한 줄 추가, 이후 server.to() 한 줄

  알림이 자주 발생하거나 동시 접속자가 많으면 user 룸 패턴 권장
```

---

# 모듈 등록 ⭐️⭐️⭐️

```typescript
@Module({
  providers:   [RoomsGateway, RoomsService],  // ⚠️ Gateway는 providers에 (controllers 아님)
  controllers: [RoomsController],
})
export class RoomsModule {}
```

```txt
Gateway를 providers에 등록하는 이유:
  Gateway는 HTTP 요청을 처리하는 게 아니라 DI 컨테이너에서 관리되는 서비스
  같은 모듈 안에 있으면 RoomsController가 RoomsGateway를 주입받을 수 있음
  다른 모듈에서 쓰려면 exports: [RoomsGateway] 추가 필요
```

---

# 이벤트 이름 규칙

```txt
단순 동사       message, join, leave
리소스:동작     room:updated, room:kicked, message:deleted, message:reaction

prefix 방식(room:, message:)을 쓰는 이유:
  이벤트가 많아져도 어떤 리소스에 관한 이벤트인지 이름만 보고 파악 가능
  클라이언트에서 on('room:kicked') → "아, 룸에서 강퇴됐을 때 처리구나"

서버 이벤트명 = 클라이언트 이벤트명 (정확히 같아야 함)
  서버: server.to(room).emit('room:updated', payload)
  클라이언트: socket.on('room:updated', handler)
  이름이 다르면 조용히 무시 (에러 없음, 이벤트 수신 안 됨)
```

---

# Gateway 전체 구현 ⭐️⭐️⭐️⭐️

```typescript
import {
  ConnectedSocket, MessageBody, OnGatewayConnection,
  SubscribeMessage, WebSocketGateway, WebSocketServer,
} from '@nestjs/websockets';
import { JwtService }    from '@nestjs/jwt';
import { ConfigService } from '@nestjs/config';
import { Server, Socket } from 'socket.io';
import type { JwtPayload } from '../auth/jwt-payload';
import { RoomsService } from './rooms.service';

type AuthedSocket = Socket & { data: { userId?: string } };

@WebSocketGateway({
  namespace: '/chat',
  cors:      { origin: true, credentials: true },
})
export class RoomsGateway implements OnGatewayConnection {

  @WebSocketServer()
  server: Server;

  constructor(
    private readonly jwtService:    JwtService,
    private readonly configService: ConfigService,
    private readonly roomsService:  RoomsService,
  ) {}

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
      await client.join(`user:${payload.sub}`);  // 개인 룸 입장
    } catch {
      client.disconnect();
    }
  }

  @SubscribeMessage('join')
  async onJoin(
    @ConnectedSocket() client: AuthedSocket,
    @MessageBody() body: { roomId: string },
  ): Promise<{ ok: boolean }> {
    const userId = client.data.userId;
    if (!userId || !body?.roomId) return { ok: false };

    await this.roomsService.assertMember(body.roomId, userId);
    await client.join(`room:${body.roomId}`);
    return { ok: true };
  }

  @SubscribeMessage('leave')
  async onLeave(
    @ConnectedSocket() client: AuthedSocket,
    @MessageBody() body: { roomId: string },
  ): Promise<{ ok: boolean }> {
    if (!body?.roomId) return { ok: false };
    await client.leave(`room:${body.roomId}`);
    return { ok: true };
  }

  // Controller에서 호출할 emit 메서드들
  emitMessage(roomId: string, message: unknown) {
    this.server.to(`room:${roomId}`).emit('message', message);
  }

  emitMessageDeleted(roomId: string, messageId: string) {
    this.server.to(`room:${roomId}`).emit('message:deleted', { messageId });
  }

  emitMemberKicked(roomId: string, targetUserId: string) {
    this.server.to(`user:${targetUserId}`).emit('room:kicked', { roomId });
  }
}
```

---

# Gateway 책임 분리 — 순환 참조 없이 구조화하기 ⭐️⭐️⭐️⭐️

```txt
문제:
  RoomsModule과 DmsModule이 각자 Gateway를 가지고
  Controller에서 서로의 Gateway를 주입받으면 순환 참조 발생

  RoomsModule → (emit 필요) → DmsModule
  DmsModule   → (emit 필요) → RoomsModule
  → 서로가 서로를 import → forwardRef 필요 → 설계 문제 신호

해결:
  Gateway의 책임을 두 가지로 분리
  ① 연결·인증·emit  → 공통 Gateway (RealtimeModule)
  ② 이벤트 구독      → 기능별 Gateway (RoomsModule, DmsModule)

  기능 모듈들은 공통 Gateway만 import → 서로를 몰라도 됨 → 순환 없음
```

## 구조

```txt
Before (순환):
  ModuleA ──import──▶ ModuleB
     ▲                    │
     └────── 서로 emit ◀──┘

After (공통 분리):
  SharedModule  ← SharedGateway (연결 + 인증 + emit만 담당)
       ▲                ▲
   ModuleA          ModuleB
  (이벤트 A)        (이벤트 B)
```

## SharedGateway — 공통 Gateway (연결·emit만)

```typescript
// shared/shared.gateway.ts
@WebSocketGateway({ namespace: '/chat', cors: { origin: true, credentials: true } })
export class SharedGateway implements OnGatewayConnection {

  @WebSocketServer()
  server: Server;

  constructor(
    private readonly jwtService:    JwtService,
    private readonly configService: ConfigService,
  ) {}

  // 연결·인증만 담당 — 비즈니스 로직(Service) 주입 없음
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
      await client.join(`user:${payload.sub}`);
    } catch {
      client.disconnect();
    }
  }

  // emit 메서드들만 — @SubscribeMessage 없음
  emitToRoom(roomId: string, event: string, payload: unknown) {
    this.server.to(`room:${roomId}`).emit(event, payload);
  }

  emitToUser(userId: string, event: string, payload: unknown) {
    this.server.to(`user:${userId}`).emit(event, payload);
  }
}
```

## 기능별 Gateway — @SubscribeMessage만

```typescript
// feature-a/feature-a-events.gateway.ts
@WebSocketGateway({ namespace: '/chat' })  // 같은 namespace — 가능
export class FeatureAGateway {
  constructor(
    private readonly sharedGateway:  SharedGateway,
    private readonly featureAService: FeatureAService,
  ) {}

  @SubscribeMessage('featureA:join')
  async onJoin(
    @ConnectedSocket() client: AuthedSocket,
    @MessageBody() body: { resourceId: string },
  ): Promise<{ ok: boolean }> {
    const userId = client.data.userId;
    if (!userId || !body?.resourceId) return { ok: false };
    await this.featureAService.assertAccess(body.resourceId, userId);
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
}

// feature-b/feature-b-events.gateway.ts
@WebSocketGateway({ namespace: '/chat' })  // 같은 namespace 공유
export class FeatureBGateway {
  constructor(private readonly featureBService: FeatureBService) {}

  @SubscribeMessage('featureB:join')
  async onJoin(
    @ConnectedSocket() client: AuthedSocket,
    @MessageBody() body: { resourceId: string },
  ): Promise<{ ok: boolean }> {
    if (!body?.resourceId) return { ok: false };
    await client.join(`featureB:${body.resourceId}`);
    return { ok: true };
  }
}
```

## 모듈 구성

```typescript
// shared/shared.module.ts
@Module({
  providers: [SharedGateway],
  exports:   [SharedGateway],   // 기능 모듈들이 inject해서 emit 호출
})
export class SharedModule {}

// feature-a/feature-a.module.ts
@Module({
  imports:     [SharedModule],                          // SharedGateway만 import
  providers:   [FeatureAService, FeatureAGateway],
  controllers: [FeatureAController],
})
export class FeatureAModule {}

// feature-b/feature-b.module.ts
@Module({
  imports:     [SharedModule],                          // SharedGateway만 import
  providers:   [FeatureBService, FeatureBGateway],
  controllers: [FeatureBController],
})
export class FeatureBModule {}
```

```txt
핵심:
  ModuleA와 ModuleB가 서로를 전혀 모름 → 순환 참조 없음
  둘 다 SharedModule만 import → SharedModule은 누구도 import 안 함

같은 namespace에 여러 Gateway:
  @WebSocketGateway({ namespace: '/chat' }) 를 여러 클래스에 붙여도 됨
  Socket.IO가 같은 namespace를 공유하므로 client.join/leave가 동일하게 동작
  @SubscribeMessage 핸들러가 어떤 Gateway에 있든 클라이언트 입장에서는 동일한 서버

Gateway는 providers에 등록 (controllers 아님):
  SharedGateway  → SharedModule의 providers + exports
  FeatureAGateway → FeatureAModule의 providers (export 불필요)
  FeatureBGateway → FeatureBModule의 providers (export 불필요)
```