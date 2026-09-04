---
aliases:
  - 외부 AI
  - 연동
  - SDK
tags:
  - NestJS
related:
  - "[[00_NestJS_Ecosystem_HomePage]]"
  - "[[JS_Operators]]"
---
# NestJS_AiProvider — AI SDK 연동 패턴

> [!info] 
> Claude·GPT 같은 외부 AI를 NestJS에 연동하는 패턴. 
> `IAiProvider` 인터페이스 + `Symbol` DI 토큰 + `useClass`로 어떤 AI를 쓰는지 AiService는 모르게 설계 (Strategy Pattern). 
> 제공사를 바꾸려면 모듈 한 줄만 수정. `Symbol` 개념 → [[JS_Operators]]

---

# 왜 인터페이스로 감싸는가 ⭐️⭐️⭐️⭐️

```txt
직접 ClaudeService를 주입하면:
  AiService → ClaudeService 직접 의존
  GPT로 바꾸려면 AiService 코드를 수정해야 함

인터페이스 + DI 토큰으로 감싸면:
  AiService → IAiProvider (인터페이스)에만 의존
  ClaudeService / GptService 중 무엇이 주입될지 모듈에서 결정
  GPT로 바꾸려면 모듈 한 줄만 수정

이 패턴 = Strategy Pattern (전략 패턴)
  "어떤 전략(AI 제공사)을 쓸지" 런타임에 주입으로 교체
```

---

# 인터페이스 + DI 토큰 — 교체 가능한 구조 ⭐️⭐️⭐️⭐️

```typescript
// ai/ai.interface.ts
export interface IAiProvider {
  translateOverview(titleEn: string, overviewEn: string): Promise<string | null>;
  koreanTitle(titleEn: string, year: string): Promise<string | null>;
}

export const AI_PROVIDER = Symbol('AI_PROVIDER');
// Symbol → 전역 유일 DI 토큰 → [[JS_Operators]] Symbol 섹션
```

```typescript
// ai/claude.service.ts
@Injectable()
export class ClaudeService implements IAiProvider {
  private readonly logger = new Logger(ClaudeService.name);
  private readonly claudeClient: Anthropic;
  private readonly model: string;              // ← 환경변수로 모델 교체 가능

  constructor(private readonly configService: ConfigService) {
    this.claudeClient = new Anthropic({
      apiKey: this.configService.getOrThrow(EnvKeys.CLAUDE_KEY),
    });
    this.model =
      this.configService.get<string>(EnvKeys.CLAUDE_MODEL) ?? 'claude-haiku-4-5';
  }

  async translateOverview(titleEn: string, overviewEn: string): Promise<string | null> {
    try {
      const msg = await this.claudeClient.messages.create({
        model: this.model, max_tokens: 1024,   // ← 하드코딩 제거
        messages: [{ role: 'user', content: `번역해줘:\n${overviewEn}` }],
      });
      const block = msg.content[0];
      return block.type === 'text' ? block.text : null;
    } catch (err) {
      return null;
    }
  }

  async koreanTitle(titleEn: string, year: string): Promise<string | null> {
    try {
      const msg = await this.client.messages.create({
        model: 'claude-sonnet-4-5', max_tokens: 100,
        messages: [{ role: 'user', content: `"${titleEn}" (${year}) 한국어 제목만 답해줘.` }],
      });
      const block = msg.content[0];
      return block.type === 'text' ? block.text.trim() : null;
    } catch { return null; }
  }
}
```

```typescript
// ai/ai.service.ts — 인터페이스만 알고 있음
@Injectable()
export class AiService {
  constructor(
    @Inject(AI_PROVIDER)
    private readonly ai: IAiProvider,
  ) {}

  translate(titleEn: string, overviewEn: string) {
    return this.ai.translateOverview(titleEn, overviewEn);
  }
}
```

```txt
@Inject(AI_PROVIDER):
  일반 @Injectable() 클래스를 주입할 때는 타입만으로 충분
  → constructor(private readonly claudeService: ClaudeService)

  커스텀 토큰(Symbol)으로 등록된 것을 주입할 때는 @Inject() 필요
  → @Inject(AI_PROVIDER) private readonly ai: IAiProvider

  이렇게 하면 ClaudeService·GptService 어느 것이 오든
  IAiProvider 인터페이스로 사용 가능
```

```typescript
// ai/ai.module.ts — 제공사 선택
@Module({
  providers: [
    { provide: AI_PROVIDER, useClass: ClaudeService },
    // GPT로 바꾸려면:
    // { provide: AI_PROVIDER, useClass: GptService },
    AiService,
  ],
  exports: [AiService],
})
export class AiModule {}
```

---

# SDK란 ⭐️⭐️⭐️

```txt
SDK(Software Development Kit) = 외부 서비스를 쉽게 쓸 수 있게 만든 라이브러리

  직접 HTTP 요청을 만들면:
    fetch('https://api.anthropic.com/v1/messages', {
      method: 'POST',
      headers: { 'x-api-key': key, 'content-type': 'application/json', ... },
      body: JSON.stringify({ ... }),
    })
    → 헤더 직접 관리, 에러 파싱, 타입 없음

  SDK를 쓰면:
    const client = new Anthropic({ apiKey });
    await client.messages.create({ ... })
    → 인증·헤더 자동 처리, TypeScript 타입 완비, 에러 객체 구조화

@anthropic-ai/sdk:
  Anthropic이 공식 제공하는 Node.js/TypeScript SDK
  import Anthropic from '@anthropic-ai/sdk' 로 클라이언트 생성
  HTTP 요청·스트리밍·에러 처리를 래핑해서 간단하게 사용 가능

openai:
  OpenAI 공식 Node.js SDK
  import OpenAI from 'openai' 로 사용
```

---

# API Key 받는 방법 ⭐️⭐️⭐️

```txt
Claude (Anthropic):
  1. https://console.anthropic.com 접속
  2. 회원가입 후 Settings → API Keys → Create Key
  3. sk-ant-... 형태의 키 복사
  4. .env → CLAUDE_KEY=sk-ant-...

GPT (OpenAI):
  1. https://platform.openai.com 접속
  2. 회원가입 후 API keys → Create new secret key
  3. sk-... 형태의 키 복사
  4. .env → OPENAI_KEY=sk-...

주의:
  키는 절대 코드에 직접 쓰지 않음 → .env + ConfigService
  .gitignore에 .env 포함 확인
  사용량/비용은 각 콘솔 대시보드에서 확인
```

---

# 설치 ⭐️⭐️⭐️

```bash
# Claude
pnpm add @anthropic-ai/sdk

# OpenAI (GPT)
pnpm add openai
```

---

# .env + EnvKeys 설정

```dotenv
# apps/api/.env
CLAUDE_KEY=sk-ant-...
CLAUDE_MODEL=claude-haiku-4-5     # 생략 시 기본값 사용
OPENAI_KEY=sk-...
OPENAI_MODEL=gpt-5-mini           # 생략 시 기본값 사용
```

```typescript
// src/config/env.keys.ts
export const EnvKeys = {
  CLAUDE_KEY:   'CLAUDE_KEY',
  CLAUDE_MODEL: 'CLAUDE_MODEL',   // 없으면 claude-haiku-4-5 폴백
  OPENAI_KEY:   'OPENAI_KEY',
  OPENAI_MODEL: 'OPENAI_MODEL',   // 없으면 gpt-5-mini 폴백
} as const;
```

```typescript
// 서비스에서 사용
constructor(private readonly configService: ConfigService) {
  this.client = new Anthropic({
    apiKey: this.configService.getOrThrow(EnvKeys.CLAUDE_KEY),
    //                          ↑ 없으면 앱 시작 시 에러 → 빠른 실패
  });
}
```

---

```txt
src/ai/
  ai.interface.ts     IAiProvider + AI_PROVIDER 토큰
  ai.service.ts       @Inject(AI_PROVIDER) → 인터페이스만 앎
  ai.module.ts        useClass로 실제 구현체 선택
  claude.service.ts   Claude SDK 구현
  gpt.service.ts      GPT SDK 구현 (추가 시)
```

---

# Claude SDK 사용법 ⭐️⭐️⭐️⭐️

```typescript
import Anthropic from '@anthropic-ai/sdk';

const client = new Anthropic({ apiKey: process.env.CLAUDE_KEY });

const message = await client.messages.create({
  model:      'claude-sonnet-4-5',  // 직접 지정 or this.model로 env에서 주입
  max_tokens: 1024,                  // 최대 출력 토큰 수
  messages: [
    { role: 'user', content: '안녕하세요' },
  ],
});

// 응답 추출
const block = message.content[0];
if (block.type === 'text') {
  console.log(block.text);
}
```

## AI 응답 유효성 검증 ⭐️⭐️⭐️⭐️

```typescript
// AI가 거절하거나 이상한 응답을 했을 때 null로 처리
function extractValidText(
  content: Anthropic.ContentBlock[],
  opts?: { maxLength?: number; rejectPatterns?: RegExp },
): string | null {
  const text = content
    .find((c) => c.type === 'text')
    ?.text?.trim();

  if (!text) return null;

  if (opts?.maxLength && text.length > opts.maxLength) return null;

  if (opts?.rejectPatterns?.test(text)) return null;

  return text;
}

// 사용 — 한국어 영화 제목 추출
const REJECT_KO = /죄송|알 수 없|확인|모르|없어/;

async koreanTitle(titleEn: string, year: string): Promise<string | null> {
  const msg = await this.client.messages.create({
    model: 'claude-sonnet-4-5', max_tokens: 100,
    messages: [{ role: 'user', content: `"${titleEn}" (${year}) 한국어 제목만 답해줘.` }],
  });

  return extractValidText(msg.content, {
    maxLength:      40,         // 40자 초과 = 불필요한 설명 붙은 응답
    rejectPatterns: REJECT_KO,  // "죄송합니다..." 같은 거절 응답
  });
}
```

```txt
왜 content[0] 대신 .find를 쓰는가:
  content 배열은 텍스트 외에 tool_use, 이미지 등 다른 타입이 올 수 있음
  .find(c => c.type === 'text') → 안전하게 텍스트 블록만 추출

유효성 검증이 필요한 이유:
  AI는 "죄송합니다, 모르겠습니다" 같은 거절 응답을 함
  이걸 제목으로 저장하면 안 됨 → null로 처리 → 폴백(원문 사용 등)

maxLength 체크:
  "싱 스트리트"              → 5자 → 통과
  "이 영화의 한국어 제목은..." → 40자 초과 → null

거절 패턴 직접 정의:
  서비스·언어마다 다름
  영어 거절: /sorry|unknown|unable|don't know/i
  한국어 거절: /죄송|알 수 없|모르/
```


## AI가 JSON을 반환할 때 — 파싱 + 런타임 타입 검증 ⭐️⭐️⭐️⭐️

```txt
문제:
  AI에게 JSON 배열을 요청해도 실제 응답에는
  ```json ... ``` 마크다운 코드 펜스가 붙어서 오는 경우가 많음

  그대로 JSON.parse하면 SyntaxError 발생

해결:
  정규식으로 코드 펜스 제거 → JSON.parse → 런타임 타입 검증
```

### 코드 펜스 제거 + JSON.parse

```typescript
const text = msg.content.find((item) => item.type === "text")?.text;
if (!text) return [];

const parsed: unknown = JSON.parse(
  text
    .replace(/^```json\s*/i, "")  // 앞의 ```json 제거
    .replace(/\s*```$/i, "")      // 뒤의 ``` 제거
    .trim(),
);
```

```txt
/^```json\s*/i
  ^         → 문자열 시작
  ```json   → 코드 펜스 시작 마커
  \s*       → 개행·공백 포함
  i         → 대소문자 무관 (```JSON 도 처리)

/\s*```$/i
  \s*       → 앞의 공백·개행
  ```       → 코드 펜스 끝 마커
  $         → 문자열 끝

JSON.parse의 반환 타입은 any → unknown으로 받는 게 안전
  unknown: 값은 있지만 타입은 모름 → 타입 좁히기(narrowing) 강제
  any:     타입 검사 없이 바로 사용 가능 → 런타임 에러 위험
```

### 런타임 타입 검증 — 타입 가드 filter ⭐️⭐️⭐️⭐️

```typescript
interface MovieQuoteSuggestion {
  text: string;
  isPopular: boolean;
}

if (!Array.isArray(parsed)) return [];

return parsed
  .filter(
    (item): item is MovieQuoteSuggestion =>
      typeof item === "object" &&
      item !== null &&
      typeof (item as { text?: unknown }).text === "string" &&
      typeof (item as { isPopular?: unknown }).isPopular === "boolean",
  )
  .map((item) => ({
    text: item.text.trim(),
    isPopular: item.isPopular,
  }))
  .filter((item) => item.text.length > 0 && item.text.length <= 80)
  .slice(0, 5);
```

```txt
(item): item is MovieQuoteSuggestion
  → 타입 서술어(type predicate) — filter의 반환 타입을 좁힘
  → filter 통과한 item은 MovieQuoteSuggestion 타입으로 추론됨
  → 이후 .map에서 item.text / item.isPopular 타입 안전하게 접근 가능

왜 (item as { text?: unknown }).text 처럼 단언하는가:
  item은 unknown → 직접 item.text 접근 불가 (TS 에러)
  { text?: unknown } 으로 단언 → text 프로퍼티 존재 가능성 인정
  → typeof ... === "string" 체크 → string으로 좁힘

검증 체인:
  Array.isArray(parsed)                    → 배열인지
  typeof item === "object" && item !== null → 객체인지 (null 제외)
  typeof item.text === "string"            → text 필드가 string인지
  typeof item.isPopular === "boolean"      → isPopular 필드가 boolean인지
  item.text.length > 0                     → 빈 문자열 제거
  item.text.length <= 80                   → 길이 제한
  .slice(0, 5)                             → 최대 5개
```

### 전체 흐름

```typescript
async recommendMovieQuotes(input: RecommendMovieQuotesInput): Promise<MovieQuoteSuggestion[]> {
  try {
    const msg = await this.claudeClient.messages.create({
      model: "claude-haiku-4-5",
      max_tokens: 512,
      messages: [{ role: "user", content: buildPrompt(input) }],
    });

    // ① 텍스트 블록 추출
    const text = msg.content.find((item) => item.type === "text")?.text;
    if (!text) return [];

    // ② 코드 펜스 제거 + JSON.parse
    const parsed: unknown = JSON.parse(
      text.replace(/^```json\s*/i, "").replace(/\s*```$/i, "").trim(),
    );

    // ③ 배열 여부 확인
    if (!Array.isArray(parsed)) return [];

    // ④ 런타임 타입 검증 + 정제
    return parsed
      .filter(
        (item): item is MovieQuoteSuggestion =>
          typeof item === "object" &&
          item !== null &&
          typeof (item as { text?: unknown }).text === "string" &&
          typeof (item as { isPopular?: unknown }).isPopular === "boolean",
      )
      .map((item) => ({ text: item.text.trim(), isPopular: item.isPopular }))
      .filter((item) => item.text.length > 0 && item.text.length <= 80)
      .slice(0, 5);

  } catch (error) {
    this.logger.warn(`실패 (${input.title}): ${(error as Error).message}`);
    return [];   // JSON.parse 실패 포함 — 항상 빈 배열로 폴백
  }
}
```

```txt
try/catch로 전체를 감싸는 이유:
  JSON.parse 자체가 SyntaxError를 던질 수 있음
  코드 펜스 제거 후에도 완전한 JSON이 아닐 수 있음 (응답 잘림 등)
  → catch에서 logger.warn + [] 반환으로 폴백

프롬프트 작성 시 JSON 응답을 강제하는 팁:
  "JSON 배열만 출력할 것" 명시
  형식 예시를 프롬프트에 포함 (few-shot)
  → 코드 펜스가 붙는 경우가 줄어들지만 완전히 없어지지는 않음
  → 항상 코드 펜스 제거 로직을 포함하는 게 안전
```


## Claude 모델 목록

```txt
claude-opus-4-5      가장 강력, 복잡한 추론·분석·코드
claude-sonnet-4-5    성능·비용 균형 → 실무 기본값
claude-haiku-4-5     가장 빠르고 저렴, 단순 작업

선택 기준:
  번역·요약·분류처럼 단순 → Haiku (저렴)
  코드 생성·복잡한 분석  → Sonnet (균형)
  고난이도 추론·긴 문서  → Opus (비쌈)

최신 모델 목록 → https://docs.anthropic.com/en/docs/about-claude/models
```

## messages 구조

```typescript
messages: [
  { role: 'user',      content: '사용자 입력' },
  { role: 'assistant', content: '이전 응답' },  // 대화 히스토리
  { role: 'user',      content: '다음 입력' },
]

// system prompt — 역할·제약 설정
const message = await client.messages.create({
  model:   'claude-sonnet-4-5',
  max_tokens: 1024,
  system: '너는 영화 번역 전문가야. 항상 자연스러운 한국어로 답해.',
  messages: [{ role: 'user', content: overviewEn }],
});
```

## max_tokens 기준

```txt
max_tokens = 응답의 최대 길이 (토큰 ≈ 단어 0.75개)

  짧은 답변 (제목, 분류, yes/no)  → 100~200
  중간 (요약, 번역 한 단락)       → 500~1024
  긴 답변 (보고서, 코드, 긴 번역) → 2048~4096
  Opus 최대                        → 32,000+

적게 설정하면 중간에 잘릴 수 있음 → 넉넉하게 설정 권장
```

---

# OpenAI SDK 사용법 ⭐️⭐️⭐️⭐️

## Chat Completions API (기존) ⭐️⭐️⭐️

```typescript
import OpenAI from 'openai';

const client = new OpenAI({ apiKey: process.env.OPENAI_KEY });

const res = await client.chat.completions.create({
  model:      'gpt-4o-mini',   // 사용할 모델
  max_tokens: 1024,
  messages: [
    { role: 'system', content: '너는 번역 전문가야.' },
    { role: 'user',   content: '번역해줘: ...' },
  ],
});

// 응답 추출
const text = res.choices[0]?.message.content ?? null;
```

## GPT 모델 목록

```txt
gpt-4o          가장 강력, 멀티모달(이미지 가능)
gpt-4o-mini     빠르고 저렴, 대부분 작업에 충분
gpt-4-turbo     긴 컨텍스트 (128k 토큰)
gpt-3.5-turbo   가장 저렴, 단순 작업

선택 기준:
  단순 번역·분류     → gpt-4o-mini
  복잡한 분석·코드   → gpt-4o
  이미지 처리        → gpt-4o

최신 모델 목록 → https://platform.openai.com/docs/models
```

## Claude vs GPT messages 차이

```typescript
// Claude — system을 별도 파라미터로
client.messages.create({
  system:   '역할 설정',
  messages: [{ role: 'user', content: '...' }],
});

// GPT — system을 messages 배열 안에
client.chat.completions.create({
  messages: [
    { role: 'system', content: '역할 설정' },
    { role: 'user',   content: '...' },
  ],
});
```

---

## Responses API (신규 2025~) ⭐️⭐️⭐️⭐️

```typescript
import OpenAI from 'openai';

const openai = new OpenAI({ apiKey: process.env.OPENAI_API_KEY });

const response = await openai.responses.create({
  model: process.env.OPENAI_MODEL ?? 'gpt-4o-mini',
  input: prompt,
});

// 텍스트 추출 — Chat Completions API와 다름
const text = response.output_text.trim();
```

```txt
Responses API vs Chat Completions API:
  responses.create()          → 신규 Responses API (2025~)
  chat.completions.create()   → 기존 Chat Completions API

텍스트 접근 방식:
  output_text                 → Responses API: 텍스트 바로 접근
  choices[0]?.message.content → Chat Completions API: 배열 인덱스로 접근

공식 문서 → https://platform.openai.com/docs/api-reference/responses
```

### Structured Outputs — JSON Schema 선언 ⭐️⭐️⭐️⭐️

```typescript
const response = await openai.responses.create({
  model: process.env.OPENAI_MODEL ?? 'gpt-4o-mini',
  input: prompt,
  text: {
    format: {
      type:   'json_schema',
      name:   'movie_quote_suggestions',
      strict: true,
      schema: {
        type: 'object',
        properties: {
          quotes: {
            type:  'array',
            items: {
              type: 'object',
              properties: {
                text:      { type: 'string' },
                isPopular: { type: 'boolean' },
              },
              required:             ['text', 'isPopular'],
              additionalProperties: false,
            },
          },
        },
        required:             ['quotes'],
        additionalProperties: false,
      },
    },
  },
});
```

```txt
text.format.type = 'json_schema'
  모델 출력 구조를 JSON Schema로 제한함
  strict: true                → 스키마 외 필드 생성 차단
  additionalProperties: false → 정의되지 않은 키 차단

Structured Outputs가 보장하는 것:   응답 구조 (타입·필드 존재 여부)
Structured Outputs가 보장 못하는 것: 데이터 신뢰성
  isPopular = AI의 임의 판단 → 커뮤니티 데이터·운영자 확인과 분리 필요
```

### Structured Outputs + Zod 검증 흐름 ⭐️⭐️⭐️⭐️

```typescript
import { z } from 'zod';

// Zod 스키마 — 업무 규칙 포함
const MovieQuoteSuggestionsSchema = z.object({
  quotes: z.array(
    z.object({
      text:      z.string().min(1).max(80),
      isPopular: z.boolean(),
    }),
  ).max(5),
});

type MovieQuoteSuggestions = z.infer<typeof MovieQuoteSuggestionsSchema>;

async recommendMovieQuotes(input: RecommendMovieQuotesInput): Promise<MovieQuoteSuggestion[]> {
  try {
    const response = await this.openai.responses.create({
      model: process.env.OPENAI_MODEL ?? 'gpt-4o-mini',
      input: buildPrompt(input),
      text: {
        format: {
          type:   'json_schema',
          name:   'movie_quote_suggestions',
          strict: true,
          schema: { /* 위 스키마와 동일 */ },
        },
      },
    });

    // ① output_text → JSON decoding
    const parsed = JSON.parse(response.output_text);

    // ② Zod 검증 — 구조 + 업무 규칙 동시 검증
    const result = MovieQuoteSuggestionsSchema.parse(parsed);

    return result.quotes;

  } catch (error) {
    this.logger.warn(`실패 (${input.title}): ${(error as Error).message}`);
    return [];
  }
}
```

```txt
Structured Outputs + Zod 두 레이어가 필요한 이유:
  Structured Outputs: 모델이 스키마 외 응답을 생성하지 못하게 차단 (모델 단)
  Zod:                애플리케이션 업무 규칙 검증 (길이·개수 등)
  → 두 레이어가 합쳐져야 진짜 안전한 AI 응답 처리

JSON.parse가 여전히 필요한 이유:
  SDK 버전에 따라 output_text가 JSON 문자열로 오는 경우가 있음
  자동 파싱 helper가 있다면 그것을 우선 사용하고, 없으면 JSON.parse → Zod 순서로 처리
```

```mermaid-beautiful
flowchart LR
  Input[input: 요청 데이터]       --> API
  Schema[text.format: JSON Schema] --> API[responses.create]
  API --> Output[output_text 문자열]
  Output --> Decode[JSON.parse]
  Decode --> Validate[Zod 검증]
  Validate --> Result[검증된 응답 타입]
```

### Claude JSON.parse 방식 vs OpenAI Structured Outputs 비교

| | Claude 방식 | OpenAI 방식 |
|---|---|---|
| 구조 강제 | 프롬프트로 요청 | JSON Schema → 모델 단 차단 |
| 코드 펜스 문제 | 발생 가능 → 제거 로직 필요 | 발생 안 함 |
| 타입 안전성 | 런타임 타입 가드 filter | Structured Outputs + Zod |
| 업무 규칙 검증 | filter + slice 직접 작성 | Zod schema |
| 구현 복잡도 | 낮음 (추가 설정 없음) | JSON Schema 정의 필요 |

```txt
핵심은 JSON.parse를 기계적으로 없애는 것이 아님.
자유 형식 응답을 받고 운에 기대어 파싱하는 패턴을
→ Structured Outputs → JSON decoding → Zod 검증 흐름으로 바꾸는 것
```

---

# Claude vs OpenAI — Provider 구현 전체 비교 ⭐️⭐️⭐️⭐️

> [!important]
> Provider를 바꾼다는 것은 서비스 이름만 바꾸는 작업이 아님.
> SDK 호출·응답 파싱·오류 처리는 각 API 규칙에 맞게 다시 작성해야 함.
> 공통 인터페이스(`IAiProvider`)는 유지하되, 구현체 내부 코드는 각 제공사 스펙에 맞게 새로 작성.

| 구분 | Claude | OpenAI GPT |
|------|--------|-----------|
| Provider 클래스 | `ClaudeService` | `OpenAiService` |
| SDK 패키지 | `@anthropic-ai/sdk` | `openai` |
| 요청 메서드 | `claudeClient.messages.create()` | `openai.responses.create()` |
| 입력 구조 | `messages[]` + `max_tokens` | `input` + `text.format` |
| 텍스트 추출 | `msg.content.find(...)?.text` | `response.output_text` |
| 구조화 응답 | 프롬프트로 JSON 요청 → 직접 파싱 | `text.format`에 JSON Schema 선언 |
| 인증 키 | `CLAUDE_KEY` | `OPENAI_API_KEY` |

---

# 공식 문서

```txt
Claude:
  API 레퍼런스  → https://docs.anthropic.com/en/api
  모델 목록     → https://docs.anthropic.com/en/docs/about-claude/models
  SDK (Node.js) → https://github.com/anthropics/anthropic-sdk-node

GPT (OpenAI):
  API 레퍼런스  → https://platform.openai.com/docs/api-reference
  모델 목록     → https://platform.openai.com/docs/models
  SDK (Node.js) → https://github.com/openai/openai-node

공통:
  요금 계산기   → 각 콘솔 대시보드
  사용량 한도   → API key 설정 페이지
```