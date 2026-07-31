---
aliases:
  - WebSocket
  - socket.io
  - client
tags:
  - NextJS
related:
  - "[[00_JS_Ecosystem_HomePage]]"
  - "[[NestJS_WebSocket]]"
  - "[[JS_Operators]]"
---
# NextJS_WebSocket — Socket.IO 클라이언트 패턴

> [!info] 
> WebSocket = 한 번 연결하면 서버도 클라이언트에게 언제든 데이터를 보낼 수 있는 실시간 통신 방식.
>  이 노트는 Next.js(클라이언트)에서 socket.io-client로 NestJS Gateway와 연결하는 패턴을 다룬다. 
>  서버 Gateway 구현 → [[NestJS_WebSocket]]

---

# WebSocket이란 — HTTP와 무엇이 다른가 ⭐️⭐️⭐️⭐️

```txt
HTTP:
  클라이언트가 요청 → 서버가 응답 → 연결 끊김
  서버가 먼저 데이터를 보낼 방법이 없음
  "새 메시지 있어?" 를 알려면 클라이언트가 계속 물어봐야 함 (폴링)

WebSocket:
  한 번 연결하면 연결이 유지됨
  서버도 클라이언트에게 언제든 데이터를 보낼 수 있음
  다른 사람이 메시지를 보내면 → 서버가 즉시 나에게 전달

채팅이 WebSocket이 필요한 이유:
  HTTP로는 서버가 먼저 "새 메시지 왔어"를 알릴 수 없음
  5초마다 "새 메시지 있어?"를 물어보는 방식 → 지연 발생, 서버 부하
  WebSocket → 서버가 메시지가 오는 순간 연결된 모든 클라이언트에게 즉시 전달
```

```txt
HTTP:
  클라이언트 ──요청──▶ 서버 ──응답──▶ 클라이언트  (연결 끊김)

WebSocket (연결 유지):
  클라이언트 ────────────────────── 서버
             ◀── 메시지 도착 알림 ──
             ──── "메시지 전송" ────▶
             ◀── 다른 사람 입력 ────
             ◀── 읽음 처리 알림 ────
```

---

# socket.io란 ⭐️⭐️⭐️

```txt
브라우저 기본 WebSocket API 위에 편의 기능을 얹은 라이브러리

socket.io가 추가해주는 것:
  자동 재연결   네트워크가 잠깐 끊겨도 자동으로 다시 연결
  room        채팅방처럼 특정 그룹에게만 메시지 보내기
  namespace   연결 단위 분리 (/chat, /notification 등)
  acknowledgement  emit하고 서버 응답을 콜백으로 받는 기능

서버(NestJS)와 클라이언트(Next.js) 양쪽 모두 socket.io를 씀:
  서버: socket.io (npm: socket.io)
  클라이언트: socket.io-client (npm: socket.io-client)
```

---

# emit / on — 양방향 통신의 핵심 ⭐️⭐️⭐️⭐️

```txt
emit = "이 이벤트를 보내다"
on   = "이 이벤트를 받으면 이 함수를 실행해"

서버와 클라이언트 둘 다 emit과 on을 가짐:
  클라이언트가 emit('join', { roomId })
  → 서버의 @SubscribeMessage('join') 핸들러가 받음

  서버가 socket.emit('message', data) 또는 room으로 broadcast
  → 클라이언트의 socket.on('message', handler)가 받음
```

```typescript
// 클라이언트 → 서버 방향
socket.emit('join', { roomId: '123' });  // 클라이언트가 보냄
// 서버의 @SubscribeMessage('join') 이 받음

// 서버 → 클라이언트 방향
socket.on('message', (data) => {         // 클라이언트가 받을 준비
  setMessages(prev => [...prev, data]);
});
// 서버가 socket.emit('message', ...) 또는 broadcast 하면 이 핸들러 실행
```

```txt
이벤트 이름은 서버와 클라이언트가 정확히 같아야 함:
  서버: this.server.to(roomId).emit('message', data)
  클라이언트: socket.on('message', handler)
              ↑ 같은 이름 'message'

  이름이 다르면 이벤트가 전달되지 않음 (에러도 안 남, 조용히 무시)
```

---

# 설치 및 기본 연결 ⭐️⭐️⭐️

```bash
pnpm add socket.io-client
```

```typescript
import { io } from 'socket.io-client';

const socket = io('http://localhost:3000');        // 기본 연결 (/ 네임스페이스)
const socket = io('http://localhost:3000/chat');   // /chat 네임스페이스
```

```txt
io(url):
  서버에 WebSocket 연결을 시작하고 Socket 인스턴스를 반환
  기본적으로 호출과 동시에 연결 시도

URL 구조:
  io('http://localhost:3000/chat')
       ↑ 서버 주소 (포트 포함)  ↑ 네임스페이스
  
  네임스페이스 = 연결 단위 분리
  서버의 @WebSocketGateway({ namespace: '/chat' }) 와 일치해야 함
  없으면 기본값 '/'
```

## 연결 관련 이벤트

```typescript
socket.on('connect', () => {
  console.log('연결됨, id:', socket.id);   // 서버가 부여한 고유 ID
});

socket.on('disconnect', (reason) => {
  console.log('연결 끊김:', reason);
  // 'io server disconnect' → 서버가 끊음
  // 'transport close'      → 네트워크 문제
});

socket.on('connect_error', (err) => {
  console.error('연결 에러:', err.message);
  // 토큰 만료나 인증 실패가 여기서 잡힘
});
```

---

# 왜 싱글턴이 필요한가 ⭐️⭐️⭐️⭐️

```txt
❌ 컴포넌트 안에서 io()를 직접 호출하면:
```

```typescript
function ChatRoom({ roomId }) {
  useEffect(() => {
    const socket = io('http://localhost:3000/chat');  // ❌
    // 이 컴포넌트가 렌더링될 때마다 새 연결이 생성됨
    // roomId가 바뀔 때마다, 부모 리렌더 때마다 새 소켓
    // → 서버에 연결이 누적됨 (10개, 100개...)
  }, [roomId]);
}
```

```txt
✅ 해결 — 모듈 스코프 싱글턴:
  파일 레벨의 변수로 소켓 하나만 유지
  getRoomSocket()으로 있으면 재사용, 없으면 생성
  io()는 앱 전체에서 딱 한 번만 호출됨
```

---

# 싱글턴 소켓 유틸 구현 ⭐️⭐️⭐️⭐️

```typescript
// lib/roomSocket.ts
import { io, type Socket } from 'socket.io-client';

let socket: Socket | null = null;  // 모듈 스코프 — 앱 전체에서 하나

export function getRoomSocket(): Socket {
  // 이미 연결된 소켓이 있으면 그대로 반환
  if (socket?.connected) return socket;

  const token = getApiAccessToken();
  if (!token) throw new Error('로그인이 필요합니다.');

  if (!socket) {
    // 소켓이 없으면 새로 만들기
    socket = io(`${getApiBaseUrl()}/chat`, {
      autoConnect:     false,         // io() 호출과 동시에 연결 안 함
      auth:            { token },     // 서버 handleConnection에서 검증
      withCredentials: true,          // 쿠키 포함
    });
  } else {
    socket.auth = { token };          // 기존 소켓 — 토큰만 갱신
  }

  if (!socket.connected) socket.connect();  // 직접 연결 시작
  return socket;
}

export function disconnectRoomSocket() {
  socket?.disconnect();
  socket = null;   // 인스턴스도 초기화 — 다음 로그인 시 새로 만들기 위해
}
```

```txt
autoConnect: false 이유:
  io()를 호출하는 시점과 실제 연결 시점을 분리
  토큰을 확인하고 auth에 세팅한 뒤 직접 socket.connect()를 호출하기 위해

socket.auth = { token } 이유:
  토큰이 만료되어 갱신된 경우, 기존 소켓 인스턴스에 새 토큰을 세팅하고 재연결
  socket = null 후 새로 만드는 것보다 효율적

disconnectRoomSocket()이 socket = null 까지 하는 이유:
  disconnect()만 하면 인스턴스는 남아있음
  다음 로그인 시 getRoomSocket()이 끊긴 인스턴스를 재사용하려 해서 문제 생길 수 있음
  null로 초기화하면 다음 호출 시 새 인스턴스 생성

withCredentials: true:
  쿠키 포함 요청 — 서버 CORS credentials: true 와 세트로 필요
```

---

# acknowledgement — emit하고 서버 응답 받기 ⭐️⭐️⭐️⭐️

```txt
일반 HTTP: 요청 → 응답이 자연스러움
WebSocket emit: 기본적으로 "쏘고 끝" — 응답이 없음

acknowledgement:
  emit의 세 번째 인자로 콜백을 넘기면
  서버 핸들러가 return한 값을 그 콜백으로 받을 수 있음
  → "쏘고 응답 대기"가 가능해짐
```

```typescript
// 기본 — 이미 연결된 경우
export function socketLeaveRoom(roomId: string): Promise<{ ok: boolean }> {
  const s = getRoomSocket();
  return new Promise((resolve) => {
    s.emit('leave', { roomId }, (res: { ok: boolean }) => resolve(res));
    //                          ↑ 서버가 return한 값이 여기로 옴
  });
}
```

```typescript
// 개선 — 연결 중이면 연결 완료 후 emit ⭐️⭐️⭐️
export function socketJoinRoom(roomId: string): Promise<{ ok: boolean }> {
  const s = getRoomSocket();
  return new Promise((resolve) => {
    const doJoin = () => {
      s.emit('join', { roomId }, (res: { ok: boolean }) =>
        resolve(res ?? { ok: false }),
      );
    };

    if (s.connected) doJoin();        // 이미 연결됨 → 즉시 emit
    else s.once('connect', doJoin);   // 연결 중 → 연결되면 한 번만 실행
  });
}
```

```txt
s.once('connect', doJoin) 가 필요한 이유:
  getRoomSocket()이 connect()를 호출하지만 연결은 비동기로 완료됨
  연결 완료 전에 emit을 보내면 서버가 못 받음
  → connected 여부를 확인하고, 아직이면 'connect' 이벤트를 한 번만 기다림

s.once vs s.on:
  s.on   → 이벤트마다 계속 실행
  s.once → 딱 한 번만 실행 후 자동 제거
  연결 대기용으로는 once가 적합 (매번 join이 실행되면 안 됨)
```

---

# 이벤트 구독 패턴 — on/off ⭐️⭐️⭐️⭐️

```typescript
// 클린업 함수를 반환하는 패턴
export function onRoomMessage(
  handler: (message: ApiRoomMessage) => void,
): () => void {
  const s = getRoomSocket();
  s.on('message', handler);
  return () => s.off('message', handler);  // 언마운트 시 구독 해제
}
```

```typescript
// useEffect에서 사용
useEffect(() => {
  const offMessage = onRoomMessage((msg) => {
    setMessages(prev => [...prev, msg]);
  });
  return offMessage;  // 언마운트 시 자동으로 s.off() 실행
}, []);
```

```txt
왜 클린업 함수를 반환하는가:
  컴포넌트가 언마운트될 때 이벤트 리스너를 제거해야 함
  제거하지 않으면 언마운트된 컴포넌트의 setState가 호출되어 메모리 누수
  useEffect의 return 함수 = cleanup → 언마운트 시 자동 호출

off()에 반드시 같은 handler 참조를 넘겨야 함:
  ❌ socket.off('message', (data) => ...)  // 새 함수 → 다른 참조 → 제거 안 됨
  ✅ socket.on('message', handler)
     socket.off('message', handler)         // 같은 변수 → 정상 제거
```

## 재연결 시 자동 재입장 ⭐️⭐️⭐️

```typescript
export function onSocketConnect(handler: () => void): () => void {
  const s = getRoomSocket();
  s.on('connect', handler);
  return () => s.off('connect', handler);
}

// 사용
useEffect(() => {
  const offConnect = onSocketConnect(() => {
    void socketJoinRoom(roomId);  // 재연결 시 자동 재입장
  });
  return offConnect;
}, [roomId]);
```

```txt
재연결 처리가 필요한 이유:
  네트워크가 잠깐 끊겼다 다시 연결되면 서버는 이 소켓이 어느 룸에 있었는지 모름
  (소켓 연결이 새로 되어 서버 메모리의 룸 정보가 초기화됨)
  → 'connect' 이벤트에서 다시 join을 보내야 메시지 수신이 정상 작동

  이 처리를 안 하면:
  재연결 후 새 메시지가 와도 이 클라이언트에 전달되지 않음
```

## 이벤트 목록 패턴

```typescript
// payload 타입을 export — 사용하는 쪽에서 타입 자동완성
export type RoomUpdatedPayload = {
  roomId:      string;
  name:        string;
  description: string | null;
  topicTags:   string[];
  updatedAt:   string;
};

export function onRoomUpdated(handler: (payload: RoomUpdatedPayload) => void) {
  const s = getRoomSocket();
  s.on('room:updated', handler);
  return () => s.off('room:updated', handler);
}

export function onRoomMessageDeleted(
  handler: (payload: { messageId: string }) => void,
) {
  const s = getRoomSocket();
  s.on('message:deleted', handler);
  return () => s.off('message:deleted', handler);
}

export function onRoomKicked(handler: (payload: { roomId: string }) => void) {
  const s = getRoomSocket();
  s.on('room:kicked', handler);
  return () => s.off('room:kicked', handler);
}
```

```txt
이벤트 이름 규칙:
  단순 동사         message, join, leave
  리소스:동작       room:updated, room:kicked, message:deleted
  → 이벤트가 많아져도 어떤 리소스에 관한 이벤트인지 명확

room:kicked를 특정 유저에게만 보내는 이유:
  강퇴는 방 전체가 아닌 강퇴 당한 본인에게만 전달해야 함
  → 서버에서 user:{userId} 룸으로 개인 전송
  → [[NestJS_WebSocket]] 참고
```

---

# 컴포넌트에서 전체 조립 ⭐️⭐️⭐️

```typescript
'use client';

function ChatRoom({ roomId }: { roomId: string }) {
  const [messages, setMessages] = useState<ApiRoomMessage[]>([]);

  useEffect(() => {
    let joined = false;

    // ① 룸 입장
    void socketJoinRoom(roomId).then((res) => {
      if (res.ok) joined = true;
    });

    // ② 재연결 시 자동 재입장
    const offConnect = onSocketConnect(() => {
      void socketJoinRoom(roomId);
    });

    // ③ 메시지 수신
    const offMessage = onRoomMessage((msg) =>
      setMessages((prev) => {
        if (prev.some((m) => m.id === msg.id)) return prev;  // 중복 방지
        return [...prev, msg];
      })
    );

    // ④ 메시지 삭제
    const offDeleted = onRoomMessageDeleted(({ messageId }) =>
      setMessages((prev) => prev.filter((m) => m.id !== messageId))
    );

    // ⑤ 언마운트 시 정리
    return () => {
      offConnect();
      offMessage();
      offDeleted();
      if (joined) void socketLeaveRoom(roomId);  // 입장했을 때만 퇴장
    };
  }, [roomId]);
}
```

```txt
joined 플래그가 필요한 이유:
  socketJoinRoom이 비동기라 마운트 즉시 언마운트될 경우
  join도 안 됐는데 leave를 보내면 서버에서 오류
  join 성공 확인 후에만 leave를 보내야 안전

void ... 앞의 void:
  async 함수가 반환하는 Promise를 "의도적으로 버린다"는 표시
  여기서는 join 결과를 then으로 처리하고 있으므로 await 대신 void 사용
  → [[JS_Operators]] void 섹션 참고
```

---

# 로그아웃 시 완전 해제 ⭐️⭐️

```typescript
import { disconnectRoomSocket } from '@/lib/roomSocket';

function handleLogout() {
  disconnectRoomSocket();   // 소켓 연결 해제 + 인스턴스 초기화
  // ... 나머지 로그아웃 처리 (토큰 삭제, 라우팅 등)
}
```

```txt
로그아웃 시 소켓 처리가 필요한 이유:
  소켓이 연결된 채로 로그아웃하면 서버에 인증 없는 연결이 남아있음
  다음 다른 계정으로 로그인 시 이전 계정의 소켓이 재사용될 수 있음

disconnectRoomSocket이 socket = null 까지 하는 이유:
  disconnect()만 하면 인스턴스가 남아 다음 getRoomSocket() 호출 시 재사용 시도
  null로 초기화 → 다음 로그인 시 새 토큰으로 완전히 새 인스턴스 생성
```