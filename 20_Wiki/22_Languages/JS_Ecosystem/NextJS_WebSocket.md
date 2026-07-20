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
  - "[[React_useMemo_useCallback_useEffect]]"
  - "[[JS_Promise]]"
  - "[[JS_WebStorage]]"
---
# NextJS_WebSocket — Socket.IO 클라이언트 패턴

> [!info] 
> 서버 Gateway 구현 → [[NestJS_WebSocket]].
>  이 노트는 Next.js/React에서 socket.io-client를 쓰는 클라이언트 패턴만 다룬다.

---

# 설치

```bash
pnpm add socket.io-client
```

---

# io() — 서버에 연결하기 ⭐️⭐️⭐️⭐️

```typescript
import { io } from 'socket.io-client';

const socket = io('http://localhost:3000');        // 기본 연결
const socket = io('http://localhost:3000/chat');   // 네임스페이스 연결
```

```txt
io(url, options):
  서버에 WebSocket 연결을 시작하고 Socket 인스턴스를 반환
  기본적으로 호출과 동시에 연결 시도 (autoConnect: true)

URL 구조:
  io('http://localhost:3000/chat')
       ↑ 서버 주소 (포트 포함)  ↑ 네임스페이스 (없으면 '/')
  
  네임스페이스는 서버 @WebSocketGateway({ namespace: '/chat' })와 일치해야 함
  같은 서버에 여러 네임스페이스 연결 가능 — 각각 독립적인 연결

반환값:
  Socket 인스턴스 — 이 연결을 대표하는 객체
  서버와 이 소켓으로 이벤트를 주고받음
```

---

# 클라이언트 Socket 객체 ⭐️⭐️⭐️⭐️

```typescript
import { type Socket } from 'socket.io-client';
// 서버의 Socket(socket.io)과 다른 타입 — 클라이언트 전용

const socket = io('http://localhost:3000/chat');

socket.id           // 이 연결의 고유 ID (서버가 부여, 재연결 시 바뀜)
socket.connected    // 현재 연결 상태 boolean
socket.auth         // 연결 시 전달한 auth 객체 (수정 가능)
```

```txt
서버 Socket vs 클라이언트 Socket:
  둘 다 이름이 Socket이지만 전혀 다른 타입
  서버: import { Socket } from 'socket.io'      — 서버에 연결된 클라이언트 하나
  클라이언트: import { Socket } from 'socket.io-client' — 서버와의 연결
```

---

# 이벤트 — on / emit / off ⭐️⭐️⭐️⭐️

## 이벤트 수신 — socket.on()

```typescript
// 서버가 보낸 이벤트 받기
socket.on('message', (data) => {
  console.log(data);
});

// 연결 관련 시스템 이벤트
socket.on('connect', () => {
  console.log('서버에 연결됨, id:', socket.id);
});
socket.on('disconnect', (reason) => {
  console.log('연결 끊김:', reason);
});
socket.on('connect_error', (err) => {
  console.error('연결 에러:', err.message);
  // 토큰 만료나 인증 실패가 여기서 잡힘
});
```

## 이벤트 발신 — socket.emit()

```typescript
// 서버에 이벤트 보내기
socket.emit('join', { roomId: '123' });

// acknowledgement — 서버 응답 받기 (세 번째 인자: 콜백)
socket.emit('join', { roomId: '123' }, (response) => {
  console.log(response);  // 서버가 return한 값
});
```

## 리스너 제거 — socket.off()

```typescript
const handler = (data) => setMessages(p => [...p, data]);

socket.on('message', handler);   // 등록
socket.off('message', handler);  // 제거 — 같은 핸들러 참조 필요

// handler를 변수에 저장하지 않으면 off로 제거 불가
// ❌ 이렇게 하면 제거 못 함
socket.on('message', (data) => setState(data));
socket.off('message', (data) => setState(data));  // 다른 함수 참조
```

```txt
off()에 핸들러를 안 넘기면:
  socket.off('message')  → 'message' 이벤트의 모든 리스너 제거
  socket.off()           → 모든 이벤트의 모든 리스너 제거
```

## 연결 관리

```typescript
socket.connect();      // 연결 (autoConnect: false일 때)
socket.disconnect();   // 연결 해제

// 연결 상태 확인
if (socket.connected) { ... }
if (!socket.connected) socket.connect();
```

---

# io() 옵션 ⭐️⭐️⭐️

```typescript
const socket = io('http://localhost:3000/chat', {
  // 연결 옵션
  autoConnect:     false,         // 기본 true — false면 connect()를 직접 호출
  reconnection:    true,          // 기본 true — 끊기면 자동 재연결
  reconnectionDelay: 1000,        // 재연결 시도 간격 (ms)

  // 인증
  auth:            { token: '...' },  // handshake.auth 로 서버에 전달

  // CORS
  withCredentials: true,          // 쿠키 포함 요청
});
```

---

# 싱글턴 소켓 유틸 ⭐️⭐️⭐️⭐️

```txt
컴포넌트 안에서 io()를 직접 호출하면:
  렌더링마다 새 소켓이 만들어져 연결이 중복됨
  서버 쪽에 대량의 연결/해제 발생

해결 — 모듈 스코프 싱글턴:
  파일 레벨 변수로 소켓 하나만 유지
  getRoomSocket()으로 있으면 재사용, 없으면 생성
  io()는 앱 전체에서 딱 한 번만 호출됨
```

```typescript
// lib/roomSocket.ts
import { io, type Socket } from 'socket.io-client';
import { getApiAccessToken } from './authToken';
import { getApiBaseUrl } from './fetchApi';
import type { ApiRoomMessage } from './rooms';

// 모듈 스코프 — 앱 전체에서 하나의 인스턴스 공유
let socket: Socket | null = null;

export function getRoomSocket(): Socket {
  if (socket?.connected) return socket;   // 이미 연결 중이면 바로 반환

  const token = getApiAccessToken();
  if (!token) throw new Error('로그인이 필요합니다.');

  if (!socket) {
    socket = io(`${getApiBaseUrl()}/chat`, {
      autoConnect:     false,   // 생성과 동시에 연결 안 함
      auth:            { token },
      withCredentials: true,
    });
  } else {
    socket.auth = { token };    // 기존 소켓 — 토큰만 갱신
  }

  if (!socket.connected) socket.connect();
  return socket;
}

export function disconnectRoomSocket() {
  socket?.disconnect();
  socket = null;                // 로그아웃 시 인스턴스도 초기화
}
```

## io() URL 구조

```typescript
io(`${getApiBaseUrl()}/chat`)
//   ↑ 서버 주소          ↑ 네임스페이스
// 예: io('http://localhost:3000/chat')
```

```txt
getApiBaseUrl():
  process.env.NEXT_PUBLIC_API_URL 같은 환경변수의 래퍼 함수
  NEXT_PUBLIC_ 접두사 없으면 브라우저에서 undefined

// lib/fetchApi.ts
export function getApiBaseUrl(): string {
  return process.env.NEXT_PUBLIC_API_URL ?? 'http://localhost:3000';
}

/chat:
  서버 @WebSocketGateway({ namespace: '/chat' }) 와 정확히 일치해야 함

autoConnect: false:
  io() 호출과 동시에 연결하지 않음
  토큰 확인 후 socket.connect()를 직접 호출

socket.auth = { token }:
  끊긴 소켓에 새 토큰을 세팅 후 재연결
  토큰 만료 → 갱신 → 재연결 흐름에서 사용

withCredentials: true:
  쿠키 포함 요청 — 서버 CORS credentials: true 와 세트
```

---

# emit — Promise 래핑 (acknowledgement) ⭐️⭐️⭐️⭐️

```typescript
// 기본 — 이미 연결된 경우
export function socketLeaveRoom(roomId: string): Promise<{ ok: boolean }> {
  const s = getRoomSocket();
  return new Promise((resolve) => {
    s.emit('leave', { roomId }, (res: { ok: boolean }) => resolve(res));
  });
}

// 개선 — 연결 중이면 연결 완료 후 emit ⭐️⭐️⭐️⭐️
export function socketJoinRoom(roomId: string): Promise<{ ok: boolean }> {
  const s = getRoomSocket();
  return new Promise((resolve) => {
    const doJoin = () => {
      s.emit('join', { roomId }, (res: { ok: boolean }) =>
        resolve(res ?? { ok: false }),
      );
    };
    if (s.connected) doJoin();       // 이미 연결됨 → 즉시 emit
    else s.once('connect', doJoin);  // 연결 중 → 연결되면 한 번만 실행
  });
}
```

```txt
s.once('connect', doJoin) 가 필요한 이유:
  getRoomSocket()이 connect()를 호출하지만
  연결이 완료되기 전(비동기)에 emit을 호출하면 서버가 못 받음
  → connected 여부를 확인하고, 안 됐으면 'connect' 이벤트 한 번만 기다림

s.once vs s.on:
  s.on    → 이벤트마다 계속 실행
  s.once  → 딱 한 번만 실행 후 자동 제거
  → 연결 대기용으로는 once가 적합

Promise 패턴:
  s.emit('join', data, callback) — 세 번째 인자가 acknowledgement 콜백
  서버 @SubscribeMessage 핸들러가 return { ok: true } 하면 이 콜백이 호출됨
  콜백을 Promise로 감싸면 async/await로 사용 가능
```

---

# on/off — 이벤트 구독 패턴 ⭐️⭐️⭐️⭐️

## 연결 이벤트 — 재연결 시 재입장 처리

```typescript
export function onSocketConnect(handler: () => void): () => void {
  const s = getRoomSocket();
  s.on('connect', handler);
  return () => s.off('connect', handler);
}
```

```typescript
// 사용 — 재연결 시 룸 재입장
useEffect(() => {
  const offConnect = onSocketConnect(() => {
    void socketJoinRoom(roomId);  // 재연결 시 자동 재입장
  });
  return offConnect;
}, [roomId]);
```

```txt
왜 재연결 처리가 필요한가:
  네트워크가 잠깐 끊겼다 다시 연결되면 소켓이 재연결됨
  재연결 시 서버는 이 소켓이 어느 룸에 있었는지 모름 (룸 정보 초기화)
  → onSocketConnect에서 다시 join을 보내야 이벤트를 정상 수신

  이 처리를 안 하면:
  재연결 후 새 메시지가 와도 이 소켓에 전달 안 됨 (룸에서 나간 상태)
  → WS 이벤트 누락 발생
```

## 메시지 · 삭제 · 강퇴

```typescript
export function onRoomMessage(
  handler: (message: ApiRoomMessage) => void,
): () => void {
  const s = getRoomSocket();
  s.on('message', handler);
  return () => s.off('message', handler);
}

export function onRoomMessageDeleted(
  handler: (payload: { messageId: string }) => void,
): () => void {
  const s = getRoomSocket();
  s.on('message:deleted', handler);
  return () => s.off('message:deleted', handler);
}

export function onRoomKicked(
  handler: (payload: { roomId: string }) => void,
): () => void {
  const s = getRoomSocket();
  s.on('room:kicked', handler);
  return () => s.off('room:kicked', handler);
}
```

## 타입이 있는 payload 이벤트 ⭐️⭐️⭐️

```typescript
// payload 타입을 export → 사용하는 쪽에서 타입 자동완성 활용
export type RoomUpdatedPayload = {
  roomId:      string;
  description: string | null;
  name:        string;
  topicTags:   string[];
  updatedAt:   string;
};

export function onRoomUpdated(
  handler: (payload: RoomUpdatedPayload) => void,
): () => void {
  const s = getRoomSocket();
  s.on('room:updated', handler);
  return () => s.off('room:updated', handler);
}
```

```txt
payload 타입을 별도 export하는 이유:
  서버 Gateway의 emit과 클라이언트 handler 타입을 일치시킬 수 있음
  사용하는 쪽에서 RoomUpdatedPayload를 import해 타입 자동완성 활용

클린업 함수 반환 패턴:
  모든 onXxx 함수가 () => void 를 반환
  useEffect return에 그대로 사용 → 언마운트 시 자동 off

  useEffect(() => {
    const off = onRoomUpdated((payload) => setRoom(payload));
    return off;
  }, []);
```

---

## WS 이벤트 목록 — 룸별 용도 분리

|이벤트|룸|용도|
|---|---|---|
|`message`|`room:{roomId}`|새 채팅 메시지|
|`message:deleted`|`room:{roomId}`|채팅 메시지 삭제|
|`room:updated`|`room:{roomId}`|방 정보 변경 (이름·공지·태그 등)|
|`room:kicked`|`user:{userId}`|특정 유저 강퇴 (방 전체 브로드캐스트 ❌)|

```txt
이벤트 이름 관행:
  단순 동사         message, join, leave
  리소스:동작       room:updated, room:kicked, message:deleted
  → 이벤트가 많아져도 어떤 리소스에 관한 이벤트인지 명확

room:kicked를 user:{userId} 룸으로 보내는 이유:
  강퇴는 방 전체가 아닌 강퇴 당한 본인에게만 전달
  → 서버에서 sockets.values() 순회 또는 user: 룸으로 개인 전송
  → [[NestJS_WebSocket]] 참고
```

---

# 컴포넌트에서 사용 ⭐️⭐️⭐️

```typescript
'use client';
import { useEffect, useState } from 'react';
import {
  socketJoinRoom, socketLeaveRoom,
  onRoomMessage, onRoomMessageDeleted,
  onSocketConnect,
} from '@/lib/roomSocket';

function ChatRoom({ roomId }: { roomId: string }) {
  const [messages, setMessages] = useState<ApiRoomMessage[]>([]);

  useEffect(() => {
    let joined = false;

    void socketJoinRoom(roomId).then((res) => {
      if (res.ok) joined = true;
    });

    // 재연결 시 재입장
    const offConnect = onSocketConnect(() => {
      void socketJoinRoom(roomId);
    });

    const offMessage = onRoomMessage((msg) =>
      setMessages((prev) => {
        if (prev.some((m) => m.id === msg.id)) return prev;  // 중복 방지
        return [...prev, msg];
      })
    );

    const offDeleted = onRoomMessageDeleted(({ messageId }) =>
      setMessages((prev) => prev.filter((m) => m.id !== messageId))
    );

    return () => {
      offConnect();
      offMessage();
      offDeleted();
      if (joined) void socketLeaveRoom(roomId);
    };
  }, [roomId]);
}
```

# 로그아웃 시 완전 해제 ⭐️⭐️

```typescript
// AuthProvider 또는 로그아웃 핸들러에서
import { disconnectRoomSocket } from '@/lib/roomSocket';

function handleLogout() {
  disconnectRoomSocket();   // 소켓 연결 해제 + 인스턴스 초기화
  // ... 나머지 로그아웃 처리 (토큰 삭제, 라우팅 등)
}
```

```txt
disconnectRoomSocket()이 socket = null 까지 하는 이유:
  disconnect()만 하면 인스턴스는 남아있음
  다음에 로그인 시 getRoomSocket()이 끊긴 인스턴스를 재사용하려 해서 문제 생길 수 있음
  null로 초기화하면 다음 getRoomSocket() 호출 시 새 인스턴스 생성
```

---

# 한눈에

```txt
싱글턴 패턴:
  모듈 스코프 let socket: Socket | null = null
  getRoomSocket() — 있으면 재사용, 없으면 io()로 생성 후 connect()
  disconnectRoomSocket() — 로그아웃 시 disconnect + null 초기화

io() 옵션:
  autoConnect: false    생성과 동시에 연결 안 함 (토큰 확인 후 직접 connect())
  auth: { token }       handshake.auth.token 으로 전달 → 서버 handleConnection에서 검증
  withCredentials: true 쿠키 포함

socket.auth = { token }:
  끊긴 소켓에 새 토큰 세팅 → 토큰 갱신 후 재연결 시

emit → Promise:
  new Promise(resolve => s.emit('event', data, resolve))
  acknowledgement 콜백을 async/await로 사용 가능

on → 클린업 함수:
  s.on('event', handler)
  return () => s.off('event', handler)
  useEffect return에 그대로 사용

서버 Gateway 구현 → [[NestJS_WebSocket]]
```