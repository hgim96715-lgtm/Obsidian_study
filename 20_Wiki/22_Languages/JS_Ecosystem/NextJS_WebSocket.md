---
aliases:
  - client
  - emit
  - off
  - on
  - socket.io
  - WebSocket
tags:
  - NextJS
related:
  - "[[00_JS_Ecosystem_HomePage]]"
  - "[[JS_FunctionPatterns]]"
  - "[[JS_Operators]]"
  - "[[NestJS_WebSocket]]"
  - "[[React_Types]]"
---
# NextJS_WebSocket — Socket.IO 클라이언트 패턴

>[!info]
>Next.js에서 Socket.IO 클라이언트로 NestJS Gateway와 통신. 
>`socket.io-client`의 `io()`로 연결, `on`으로 구독, `emit`으로 발신. 
>타입 안전한 `Socket<ServerToClient, ClientToServer>` 제네릭으로 이벤트 타입 정의. `
>useEffect`로 연결·구독·해제를 커스텀 훅으로 분리. NestJS 서버 → [[NestJS_WebSocket]]

---

# WebSocket이란 — HTTP와 무엇이 다른가 ⭐️⭐️⭐️⭐️

```txt
HTTP:
  클라이언트가 요청 → 서버가 응답 → 연결 끊김
  서버가 먼저 데이터를 보낼 수 없음

WebSocket:
  한 번 연결하면 양쪽이 언제든 메시지를 보낼 수 있음
  서버 → 클라이언트 push 가능
  연결 유지 (실시간 채팅, 알림, 라이브 업데이트)

언제 WebSocket:
  채팅·댓글·좋아요 실시간 반영
  다른 유저의 행동이 내 화면에 즉시 반영
  서버에서 주기적으로 데이터를 밀어야 할 때
```

---

# emit · on — 양방향 통신의 핵심 ⭐️⭐️⭐️⭐️

```typescript
// emit — 상대에게 이벤트 보내기
socket.emit('이벤트명', 데이터);

// on — 이벤트 받기 (구독)
socket.on('이벤트명', (데이터) => { /* 처리 */ });

// off — 구독 해제
socket.off('이벤트명', 핸들러);
```

```txt
이벤트 기반:
  HTTP처럼 path가 아닌 "이벤트 이름"으로 구분
  socket.emit('join', { roomId })  → 서버의 @SubscribeMessage('join')
  server.emit('message', data)     → 클라이언트의 socket.on('message', ...)
```

---

# 설치 및 타입 안전한 소켓 ⭐️⭐️⭐️⭐️

```bash
pnpm add socket.io-client
```

```typescript
import { io, Socket } from 'socket.io-client';

// 서버 → 클라이언트 이벤트 타입 (서버 emit, 클라이언트 on)
type ServerToClient = {
  message:    (payload: MessageItem) => void;
  userJoined: (payload: { userId: string }) => void;
};

// 클라이언트 → 서버 이벤트 타입 (클라이언트 emit, 서버 @SubscribeMessage)
type ClientToServer = {
  join:        (roomId: string) => void;
  sendMessage: (dto: CreateMessageDto) => void;
};

// 타입이 적용된 소켓 타입
type ChatSocket = Socket<ServerToClient, ClientToServer>;
//                       ↑ .on() 타입   ↑ .emit() 타입
```

```txt
Socket<ServerToClient, ClientToServer> 효과:
  socket.on('message', (p) => ...)  → p가 MessageItem 자동완성
  socket.emit('join', roomId)       → 인자 타입 체크
  socket.on('없는이벤트', ...)       → TS 에러

  ServerToClient 이름 ↔ NestJS gateway emit 이름 일치 필요
  ClientToServer 이름 ↔ @SubscribeMessage('이름') 일치 필요
```

## 연결 팩토리 함수

```typescript
// lib/chat-socket.ts
const API_URL = process.env.NEXT_PUBLIC_API_URL ?? 'http://localhost:3050';

export function connectChatSocket(token: string): ChatSocket {
  return io(`${API_URL}/chat`, {
    auth:       { token },       // handshake.auth.token → 서버 인증
    transports: ['websocket'],   // polling 없이 바로 WebSocket
  }) as ChatSocket;
}
```

```txt
auth: { token }:
  서버 handleConnection에서 socket.handshake.auth.token 으로 꺼냄
  JWT 검증에 사용

transports: ['websocket']:
  기본값 ['polling', 'websocket'] → polling 먼저, 그 다음 upgrade
  ['websocket'] 단독 → 바로 연결 (더 빠름)
```

---

# 커스텀 훅 패턴 — useEffect로 소켓 관리 ⭐️⭐️⭐️⭐️

```txt
소켓 연결·구독·해제를 커스텀 훅으로 분리하는 것이 표준 패턴

이유:
  useEffect cleanup → 언마운트 시 socket.disconnect() 자동 실행
  deps 변경(토큰 갱신) 시 자동으로 재연결
  컴포넌트에서 소켓 로직 분리 → 컴포넌트가 얇아짐
```

## 훅 없이 — 컴포넌트에서 직접

```typescript
// 소켓 로직이 단순하거나 한 곳에서만 쓸 때
'use client';
import { useEffect } from 'react';
import { connectChatSocket } from '@/lib/chat-socket';

function ChatRoom({ roomId }: { roomId: string }) {
  const [messages, setMessages] = useState<MessageItem[]>([]);
  const accessToken = useAuthStore(s => s.accessToken);

  useEffect(() => {
    if (!accessToken) return;

    const socket = connectChatSocket(accessToken);

    socket.on('connect', () => {
      socket.emit('join', roomId);
    });

    socket.on('message', (payload) => {
      setMessages(prev => [...prev, payload]);
    });

    return () => {
      socket.disconnect();
    };
  }, [accessToken, roomId]);

  return <div>{messages.map(m => <p key={m.id}>{m.text}</p>)}</div>;
}
```

```txt
훅으로 분리 vs 컴포넌트에서 직접:

  컴포넌트에서 직접:
    소켓 이벤트가 2-3개로 단순할 때
    이 컴포넌트 하나에서만 소켓을 씀
    빠르게 구현이 필요할 때

  커스텀 훅으로 분리:
    이벤트가 많아서 컴포넌트가 복잡해질 때
    여러 컴포넌트에서 같은 소켓 로직을 씀
    args(setter들)가 많아서 하나로 묶고 싶을 때
    → 외부에서 재사용 가능, 컴포넌트가 얇아짐

  결과는 동일 — 구조의 차이만 있음
```

## 커스텀 훅으로 분리한 버전

```typescript
// hooks/useChatSocket.ts
'use client';
import { useEffect } from 'react';
import { connectChatSocket } from '@/lib/chat-socket';

type Args = {
  accessToken:  string | null;
  setMessages:  React.Dispatch<React.SetStateAction<MessageItem[]>>;
  setOnlineCount: React.Dispatch<React.SetStateAction<number>>;
};

export function useChatSocket({ accessToken, setMessages, setOnlineCount }: Args) {
  useEffect(() => {
    if (!accessToken) return;  // 토큰 없으면 연결 안 함
    //   ↑ early return → cleanup 함수도 없음 (연결 안 했으니 해제도 없음)

    const socket = connectChatSocket(accessToken);

    // 연결 완료 → 방 입장 emit
    socket.on('connect', () => {
      socket.emit('join', roomId);
    });

    // 서버 이벤트 구독
    socket.on('message', (payload) => {
      setMessages(prev => [...prev, payload]);
      //                    ↑ prev → 최신 상태 기반으로 업데이트
    });

    socket.on('userJoined', ({ userId }) => {
      setOnlineCount(prev => prev + 1);
    });

    // cleanup — 컴포넌트 언마운트 or accessToken 변경 시 실행
    return () => {
      socket.disconnect();
    };
  }, [accessToken, setMessages, setOnlineCount]);
  //  ↑ accessToken 바뀌면 이전 소켓 해제 후 새 소켓 연결
}
```

```typescript
// 컴포넌트에서 사용
function ChatRoom() {
  const [messages, setMessages] = useState<MessageItem[]>([]);
  const [onlineCount, setOnlineCount] = useState(0);
  const accessToken = useAuthStore(s => s.accessToken);

  useChatSocket({ accessToken, setMessages, setOnlineCount });
  //  ↑ 한 줄로 소켓 관리 완료 — 컴포넌트가 얇게 유지됨

  return (
    <div>
      {messages.map(m => <Message key={m.id} {...m} />)}
    </div>
  );
}
```

```txt
setMessages(prev => [...prev, payload]):
  prev → 을 쓰는 이유:
  useEffect 클로저는 마운트 시점의 messages를 캡처
  직접 setMessages([...messages, payload]) 하면 클로저의 오래된 messages 참조
  prev → 콜백 형태는 항상 최신 상태를 받음 → 안전

accessToken을 deps에 넣는 이유:
  토큰이 갱신되면 이전 소켓을 끊고 새 토큰으로 재연결
  → cleanup(disconnect) → effect 재실행(새 연결)
```

---

# 이벤트 구독 패턴 — on / off ⭐️⭐️⭐️⭐️

```typescript
useEffect(() => {
  if (!socket) return;

  const handleMessage = (payload: MessageItem) => {
    setMessages(prev => [...prev, payload]);
  };

  // 구독
  socket.on('message', handleMessage);

  // 반드시 cleanup에서 off
  return () => {
    socket.off('message', handleMessage);
  };
}, [socket]);
```

```txt
off 할 때 같은 함수 참조 필요:
  socket.on('message', handleMessage)   ← 이름 있는 함수
  socket.off('message', handleMessage)  ← 같은 참조로 해제

  arrow 함수를 on/off에 직접 넣으면 참조가 달라서 off 안 됨:
  ❌ socket.on('message', (p) => setMessages(...))
     socket.off('message', (p) => setMessages(...))  ← 다른 함수!

useEffect 하나에 on/off 쌍으로:
  구독과 해제가 항상 쌍으로 묶임 → 메모리 누수 방지
```

---

# 싱글턴 소켓 — 연결 재사용 ⭐️⭐️⭐️

```typescript
// lib/room-socket.ts — 여러 컴포넌트가 같은 소켓 재사용
let socket: ChatSocket | null = null;

export function getRoomSocket(token: string): ChatSocket {
  if (socket?.connected) return socket;  // 이미 연결됨 → 재사용
  socket = connectChatSocket(token);
  return socket;
}

export function disconnectRoomSocket() {
  socket?.disconnect();
  socket = null;
}
```

```txt
왜 싱글턴이 필요한가:
  컴포넌트 A, B가 같은 소켓을 써야 할 때
  각자 io()를 부르면 연결이 두 개 생김 → 이벤트 두 번 수신

  커스텀 훅 방식:
  훅 하나가 소켓 전체를 관리 → 싱글턴 불필요
  훅이 여러 곳에 있으면 → 싱글턴 또는 Context로 공유

getRoomSocket() 부르는 이유:
  socket이 null이면 새로 연결
  이미 연결돼 있으면 기존 소켓 반환
  → 중복 연결 방지
```

---

# 재연결 시 자동 재입장 ⭐️⭐️⭐️

```typescript
useEffect(() => {
  if (!accessToken) return;

  const socket = connectChatSocket(accessToken);

  function joinRoom() {
    socket.emit('join', roomId);
  }

  // 최초 연결 + 재연결 시 모두 입장
  socket.on('connect', joinRoom);

  return () => {
    socket.off('connect', joinRoom);
    socket.disconnect();
  };
}, [accessToken, roomId]);
```

```txt
connect 이벤트에 joinRoom을 등록하는 이유:
  네트워크 끊김 후 socket.io가 자동 재연결함
  재연결 시 서버는 새 소켓으로 인식 → 방 정보 초기화
  → connect 이벤트마다 emit('join') 재실행 → 방 유지
```

---

# 로그아웃 시 완전 해제 ⭐️⭐️

```typescript
function logout() {
  clearSession();        // Zustand + localStorage 초기화
  disconnectRoomSocket(); // 소켓 연결 끊기
  router.push('/login');
}
```

```txt
로그아웃 시 소켓을 끊는 이유:
  연결이 살아있으면 서버는 계속 이벤트를 보냄
  다음 로그인 시 이전 이벤트가 남아있을 수 있음
  → 명시적으로 disconnect()
```