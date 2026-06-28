---
aliases: [설명 생성, AI, Anthropic, Claude]
tags:
  - NestJS
  - AI
related:
  - "[[00_NestJS_Ecosystem_HomePage]]"
  - "[[NestJS_Env_Config]]"
  - "[[NestJS_External_API]]"
  - "[[NestJS_Service_Provider]]"
---
# NestJS_AI — Claude AI 연동

# 한 줄 요약

```txt
@anthropic-ai/sdk 로 NestJS 에서 Claude API 호출
AI 기능을 Service 로 분리 / API 키 없어도 앱 실행 가능
```

---

---

# 개념 ⭐️

```txt
Anthropic = Claude 를 만든 회사
Claude    = Anthropic 의 AI 모델 (GPT 의 대안)

@anthropic-ai/sdk:
  Anthropic 이 공식 제공하는 Node.js SDK
  Claude API 를 쉽게 호출할 수 있는 라이브러리
  fetch 로 직접 호출하는 것보다 타입 안전 / 편리

사용 흐름:
  API 키 발급 → SDK 설치 → 클라이언트 생성 → 메시지 전송 → 응답 파싱
```

---

---

# 설치 & API 키 발급

```bash
# 루트에서 설치
pnpm add @anthropic-ai/sdk
```

```txt
API 키 발급:
  https://console.anthropic.com
  → API Keys → Create Key → 이름 입력 → 복사
  sk-ant-... 형식

  ⚠️ 한 번만 보여줌 → 바로 복사해서 저장
  ⚠️ 유료 서비스 → 사용량만큼 과금

키 이름 vs .env 변수명:
  Create Key 시 이름 (예: my-claude-key)
    → 대시보드에서 구분하는 라벨 / 코드와 무관

  .env 의 ANTHROPIC_API_KEY
    → 코드에서 읽는 변수명 / 키 이름과 달라도 됨
```

```properties
# 루트 .env (Git 제외)
ANTHROPIC_API_KEY=sk-ant-실제키값
```

```typescript
// src/config/env.keys.ts
ANTHROPIC_API_KEY: 'ANTHROPIC_API_KEY',

// src/config/env.validation.ts
[EnvKeys.ANTHROPIC_API_KEY]: Joi.string().optional(),
//                                        ↑ 필수 아님
// 키 없어도 앱 실행 / AI 기능만 503 으로 막음
```

---

---

# Anthropic 클라이언트 생성 ⭐️

```typescript
import Anthropic from '@anthropic-ai/sdk';

// 클라이언트 생성 — API 키를 전달해서 인증
const client = new Anthropic({ apiKey: 'sk-ant-...' });
```

```txt
new Anthropic({ apiKey }):
  Anthropic 서버와 통신하는 클라이언트 객체 생성
  모든 API 호출은 이 client 를 통해 이루어짐
  apiKey 로 "나는 이 계정이야" 인증

  직접 비유:
  axios.create({ baseURL, headers: { Authorization: apiKey } })
  와 비슷한 개념
```

## NestJS 에서 — null 패턴 ⭐️

```typescript
@Injectable()
export class AiService {
  private client: Anthropic | null = null;
  //              ↑ 처음에는 null (키 확인 전)

  constructor(private readonly config: ConfigService) {
    const apiKey = this.config.get<string>(EnvKeys.ANTHROPIC_API_KEY);

    if (apiKey) {
      this.client = new Anthropic({ apiKey });
      //             ↑ 키 있을 때만 클라이언트 생성
    }
    // 키 없으면 client 는 null 그대로
  }
}
```

```txt
왜 null 로 초기화하나:
  API 키가 없는 환경(로컬 테스트, 미설정 서버)에서도
  NestJS DI 컨테이너에 서비스가 등록되어야 함
  → 클라이언트 없이도 서비스 객체는 생성됨

  키 있는 환경  → client = Anthropic 인스턴스 → 정상 동작
  키 없는 환경  → client = null → 호출 시 503 / 나머지 API 정상

  타입: Anthropic | null
    null 가능성을 타입으로 명시
    → 메서드 안에서 null 체크 강제
```

---

---

# Claude API 호출 ⭐️

```typescript
const message = await this.client.messages.create({
  model:      'claude-sonnet-4-6',   // ← 현재 권장 모델
  max_tokens: 400,
  messages: [
    {
      role:    'user',
      content: '여기에 프롬프트',
    },
  ],
});
```

```txt
messages.create():
  Claude 에게 메시지를 보내고 응답 받는 메서드
  HTTP POST 를 내부적으로 처리해줌

role:
  'user'      사람이 보내는 메시지
  'assistant' Claude 가 보낸 메시지 (이전 대화 이어갈 때)

model ⭐️:
  'claude-sonnet-4-6'       현재 권장 / 속도·품질 균형
  'claude-opus-4-6'         가장 강력 / 느리고 비쌈
  'claude-haiku-4-5'        가장 빠르고 저렴

  ⚠️ EOL (End of Life) 모델 주의:
    'claude-sonnet-4-20250514' → 2026-06-15 에 EOL
    → 요청 거절됨 (Anthropic 서버에서 400/404 반환)
    → NestJS 가 이 에러를 안 잡으면 500 Internal Server Error
    → 화면에는 "Internal server error" 만 보임

    모델명은 Anthropic 공식 문서에서 항상 확인
    https://docs.anthropic.com/en/docs/about-claude/models

max_tokens:
  Claude 가 생성할 최대 토큰 수 (글자 수 제한)
  400 ≈ 한국어 200~300자 정도
  너무 적으면 문장이 중간에 잘림
  너무 많으면 비용 증가
```

## 여러 턴 대화 (멀티턴)

```typescript
// 이전 대화를 messages 배열에 추가해서 맥락 유지
const message = await this.client.messages.create({
  model:      'claude-sonnet-4-20250514',
  max_tokens: 1024,
  messages: [
    { role: 'user',      content: '안녕!' },
    { role: 'assistant', content: '안녕하세요!' },
    { role: 'user',      content: '오늘 날씨 어때?' },
    // ↑ 마지막이 user 로 끝나야 함
  ],
});
```

---

---

# 응답 구조 — message.content ⭐️

```typescript
// Claude 응답 구조
message = {
  id:         'msg_xxx',
  type:       'message',
  role:       'assistant',
  model:      'claude-sonnet-4-20250514',
  content: [
    { type: 'text', text: 'Claude 가 생성한 텍스트...' }
    // type 이 'text' 외에 'tool_use' 등도 있을 수 있음
  ],
  stop_reason: 'end_turn',
  usage: {
    input_tokens:  50,    // 프롬프트 토큰 수
    output_tokens: 120,   // 응답 토큰 수
  },
}
```

```txt
content 가 배열인 이유:
  응답이 여러 블록으로 올 수 있음
  텍스트 + 도구 호출 동시에 올 수도 있음
  → type 으로 어떤 블록인지 구분
```

## 텍스트 추출 패턴 ⭐️

```typescript
// content 배열에서 text 타입 블록 찾기
const block = message.content.find((b) => b.type === 'text');
const text  = block?.type === 'text' ? block.text.trim() : '';
```

```txt
왜 이렇게 쓰나:

  1. find((b) => b.type === 'text')
     content 배열에서 첫 번째 text 블록 찾기
     없으면 undefined 반환

  2. block?.type === 'text'
     TypeScript 타입 좁히기
     block 이 undefined 면 → '' 반환
     block.type 이 'text' 여야 → .text 속성 접근 가능
     이 조건 없으면 TypeScript 가 .text 접근 에러

  3. block.text.trim()
     앞뒤 공백 / 줄바꿈 제거

더 간단한 버전:
  const text = message.content
    .filter(b => b.type === 'text')
    .map(b => b.type === 'text' ? b.text : '')
    .join('') .trim();
```

---

---

# Anthropic 에러 핸들링 ⭐️

```txt
Anthropic SDK 에서 발생하는 에러는 NestJS 가 자동으로 잡지 않음
→ 처리 안 하면 500 Internal Server Error 로 떨어짐

자주 발생하는 에러:
  EOL 모델 사용     → 400 / 404 (요청 거절)
  API 키 잘못됨     → 401 Unauthorized
  토큰 한도 초과    → 429 Too Many Requests
  Anthropic 서버 문제 → 500 / 503
```

```typescript
import Anthropic from '@anthropic-ai/sdk';

async generateSomething(input: string): Promise<string> {
  if (!this.client) {
    throw new ServiceUnavailableException('API 키가 없습니다.');
  }

  try {
    const message = await this.client.messages.create({
      model:      'claude-sonnet-4-6',
      max_tokens: 400,
      messages: [{ role: 'user', content: input }],
    });

    const block = message.content.find((b) => b.type === 'text');
    const text  = block?.type === 'text' ? block.text.trim() : '';

    if (!text) throw new ServiceUnavailableException('AI 응답이 비어 있습니다.');
    return text;

  } catch (e) {
    // Anthropic SDK 에러 구분
    if (e instanceof Anthropic.APIError) {
      throw new ServiceUnavailableException(
        `Claude API 오류: ${e.status} ${e.message}`,
      );
    }
    // NestJS HttpException 은 그대로 re-throw
    throw e;
  }
}
```

```txt
Anthropic.APIError:
  Anthropic SDK 가 던지는 에러 클래스
  e.status   HTTP 상태코드 (400 / 401 / 429 등)
  e.message  에러 메시지

instanceof Anthropic.APIError 로 구분하는 이유:
  ServiceUnavailableException 은 다시 throw 하면 안 됨
  → NestJS HttpException 은 그대로 올려보내야 함
  → Anthropic 에러만 잡아서 503 으로 변환

EOL 모델 에러 예시:
  model: 'claude-sonnet-4-20250514'  (2026-06-15 EOL)
  → Anthropic.APIError { status: 400, message: 'model not found' }
  → NestJS 가 안 잡으면 500 Internal Server Error
  → try/catch + Anthropic.APIError 로 503 으로 변환
```

```typescript
import { ServiceUnavailableException } from '@nestjs/common';

async generate() {
  if (!this.client) {
    throw new ServiceUnavailableException('API 키가 설정되지 않았습니다.');
  }
  // ...
  if (!text) {
    throw new ServiceUnavailableException('AI 응답이 비어 있습니다.');
  }
}
```

```txt
HTTP 503 Service Unavailable:
  "서버는 동작하지만 이 서비스는 일시적으로 사용 불가"

AI 기능에 적합한 이유:
  API 키 미설정    → 이 기능은 사용 불가 (503)
  외부 AI 서버 문제 → 일시적으로 사용 불가 (503)
  앱 자체 에러      → 500 Internal Server Error

  404  리소스 없음
  400  잘못된 요청
  500  서버 내부 에러 (우리 코드 문제)
  503  서비스 사용 불가 (외부 의존성 문제)
```

---

---

# 다른 프로젝트에 적용하는 법

```typescript
// 1. 서비스 만들기
@Injectable()
export class YourAiService {
  private client: Anthropic | null = null;

  constructor(private readonly config: ConfigService) {
    const apiKey = this.config.get<string>(EnvKeys.ANTHROPIC_API_KEY);
    if (apiKey) this.client = new Anthropic({ apiKey });
  }

  async generateSomething(input: string): Promise<string> {
    if (!this.client) throw new ServiceUnavailableException('키 없음');

    const message = await this.client.messages.create({
      model:      'claude-sonnet-4-20250514',
      max_tokens: 500,
      messages: [{ role: 'user', content: input }],
    });

    const block = message.content.find((b) => b.type === 'text');
    const text  = block?.type === 'text' ? block.text.trim() : '';

    if (!text) throw new ServiceUnavailableException('응답 없음');
    return text;
  }
}
```

```txt
바꾸는 부분:
  generateSomething  → 목적에 맞는 메서드 이름
  max_tokens         → 응답 길이에 맞게
  content: input     → 프롬프트 내용

고정으로 쓰는 부분:
  client: Anthropic | null = null    → 항상 이 패턴
  find((b) => b.type === 'text')     → 텍스트 추출 항상 동일
  ServiceUnavailableException        → AI 에러는 항상 503
```

---

---

# 프롬프트 잘 쓰는 법

```txt
규칙을 명확하게:
  길이 제한 (2~4문장)
  언어 명시 (한국어)
  금지 사항 (지어내지 말 것)
  형식 (따옴표 없이 / 본문만)

정보를 구조화해서 전달:
  - 제목: ${title}
  - 기간: ${start} ~ ${end}
  - 가격: ${price ?? '미상'}

모르는 내용 처리:
  "사실만 써줘" 명시
  null / 미상 값은 '미상' 으로 전달
  → 없는 내용 지어내는 환각 방지
```

---

---

# 한눈에

```txt
설치:
  pnpm add @anthropic-ai/sdk
  ANTHROPIC_API_KEY=sk-ant-... (.env)

클라이언트:
  new Anthropic({ apiKey })
  private client: Anthropic | null = null  (키 없으면 null)

API 호출:
  client.messages.create({ model, max_tokens, messages })
  model: 'claude-sonnet-4-6'  ← EOL 모델 주의 ⚠️

응답 파싱:
  message.content.find(b => b.type === 'text')
  block?.type === 'text' ? block.text.trim() : ''

에러 처리:
  Anthropic.APIError → ServiceUnavailableException (503)
  키 없음 / 응답 없음 → ServiceUnavailableException (503)
  처리 안 하면 500 Internal Server Error

모델 확인:
  https://docs.anthropic.com/en/docs/about-claude/models
```