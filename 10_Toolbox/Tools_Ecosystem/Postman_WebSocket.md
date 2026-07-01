---
aliases:
  - Postman WebSocket
  - Postman Socket.IO
tags:
  - Postman
related:
  - "[[00_Tools_Ecosystem_HomePage]]"
---

# Postman_WebSocket — WebSocket 연결 테스트

```txt
Postman 은 HTTP 요청뿐만 아니라 WebSocket 연결도 지원
Socket.IO 서버와 직접 연결해서 이벤트 송수신 테스트 가능
```

---

---

# 연결 생성 ⭐️

```txt
Postman 좌측 상단 New → Socket.IO 선택

URL 입력:
  ws://localhost:3001
  ↑ ws:// 로 시작
  ↑ WebSocket 전용 포트 (HTTP 포트와 분리)

Connect 클릭 → 연결 성공 시 초록 불
```

## 환경변수로 관리

```txt
Environments 에 wsHost 변수 추가:
  wsHost = ws://localhost:3001

URL 입력창:
  {{wsHost}}

배포 환경으로 바꿀 때:
  wsHost = ws://43.202.2.225:3001
  → URL 수정 없이 환경변수만 변경
```

---

---

# 이벤트 송신 (emit) ⭐️

```txt
하단 입력창:
  Event name: message         ← @SubscribeMessage('message') 와 일치
  Message:    {"text": "안녕"} ← JSON 형태로 입력

Send 클릭 → 서버로 이벤트 전송
```

---

---

# 이벤트 수신 (listen)

```txt
Listen to events 탭 → + 버튼
  Event name: message  ← 서버가 emit 하는 이벤트명 등록

서버에서 해당 이벤트 emit 시 Postman 에서 수신 확인 가능
```

---

---

# 연결 확인 흐름

```txt
1. NestJS 서버 실행 (pnpm run start:dev)
2. Postman Environments 에 wsHost = ws://localhost:3001 추가
3. Postman → New → Socket.IO
4. URL: {{wsHost}}
5. Connect
6. Listen to events 에 수신할 이벤트명 등록
7. Event name + Message 입력 후 Send
8. 서버 로그 / Postman 수신 탭에서 확인
```