---
aliases:
  - 외부 API 호출
  - XML 파싱
  - fast-xml-parser
tags:
  - NestJS
related:
  - "[[00_NestJS_Ecosystem_HomePage]]"
  - "[[Linux_Curl_API]]"
  - "[[NestJS_Env_Config]]"
  - "[[JS_Fetch_API]]"
  - "[[JS_Date]]"
---
# NestJS_External_API — 외부 API 호출 & XML 파싱

```bash
외부 API (공공 API 등) 를 NestJS 에서 호출하는 패턴
curl 로 구조 확인 → 타입 정의 → 파서 분리 → 서비스에서 사용
#  → [[Linux_Curl_API]] 참고
```

---

---

# 설치

```bash
pnpm add fast-xml-parser
```

---

---

# 폴더 구조

```
src/
  exhibition/
    culture-api.types.ts    API 응답 타입 정의
    culture-api.parser.ts   XML 파싱 함수 모음
    exhibition.service.ts   실제 API 호출
```

```
파서를 서비스에서 분리하는 이유:
  파싱 로직이 복잡해질 수 있음
  타입 캐스팅 / 에러 처리 등을 한 곳에서 관리
  서비스는 "무엇을 가져올지" 에만 집중
  파서는 "XML 을 어떻게 변환할지" 만 담당
```

---

---

# culture-api.types.ts — 타입 정의 ⭐️

```typescript
// 공공 API 응답 필드를 타입으로 정의
// curl 로 확인한 XML 구조 기반으로 작성

export type CultureListItem = {
  seq:         number;
  title:       string;
  startDate:   string;
  endDate:     string;
  area:        string;
  thumbnail:   string;
  serviceName: string;
};

export type CultureDetailItem = CultureListItem & {
  contents1:  string | null;
  price:      string | null;
  url:        string | null;
  placeAddr:  string | null;
  venueName:  string | null;
};
```

---

---

# culture-api.parser.ts — 파서 ⭐️

```typescript
import { XMLParser } from 'fast-xml-parser';
import { CultureDetailItem, CultureListItem } from './culture-api.types';

const parser = new XMLParser({
  ignoreAttributes:  true,    // 태그 속성 무시
  trimValues:        true,    // 앞뒤 공백 제거
  parseTagValue:     false,   // 값 자동 변환 비활성화 → 모두 string 유지
  isArray: (tagName) => tagName === 'item',
  //                         ↑ item 은 항상 배열로 강제
});
```

```
XMLParser 옵션:
  ignoreAttributes: true
    <tag attr="값"> 처럼 태그 속성은 무시
    공공 API 는 보통 속성 안 씀

  trimValues: true
    "  서울  " → "서울" 앞뒤 공백 자동 제거

  parseTagValue: false ⭐️
    기본값 true = XML 값을 타입에 맞게 자동 변환
      "123"   → 123 (number)
      "true"  → true (boolean)
      "00"    → 0 (number)   ← ⚠️ 위험

    false 로 설정하면 모든 값을 string 그대로 유지

    공공 API 에서 false 가 필요한 이유:
      resultCode "00" → 0 으로 변환되면 !== '00' 비교 실패
      날짜 "20260601" → 숫자 변환되면 parseYmd 에서 문제
      → 타입 변환은 직접 명시적으로 (Number() / parseYmd 등)

  isArray: (tagName) => tagName === 'item'
    item 이 1개 → 객체로 옴 (기본 동작)
    item 이 여러 개 → 배열로 옴
    → 개수마다 타입이 달라지는 문제 방지
    → 항상 배열로 강제해서 .map() 안전하게 사용
```

```typescript
// 공통 응답 파싱 + 에러 처리
export const parseCultureResponse = (xml: string) => {
  const parsed = parser.parse(xml) as {
    response?: {
      header?: { resultCode?: string; resultMsg?: string };
      body?:   Record<string, unknown>;
    };
  };

  const header = parsed.response?.header;
  const body   = parsed.response?.body;

  if (!header || header.resultCode !== '00') {
    throw new Error(
      `Culture API error: ${header?.resultCode ?? 'unknown'} ${header?.resultMsg ?? ''}`,
    );
  }

  return body ?? {};
};
```

```
parser.parse(xml) as { response?: ... }:
  fast-xml-parser 는 반환 타입이 any
  → as 로 예상 구조를 TypeScript 에 알려줌
  → optional chaining (?.) 으로 안전하게 접근

resultCode !== '00':
  공공 API 성공 코드 = '00'
  실패면 에러 throw → 서비스에서 try-catch 로 처리
```

```typescript
// 목록 파싱
export const parseListItems = (xml: string) => {
  const body       = parseCultureResponse(xml);
  const totalCount = Number(body.totalCount ?? 0);
  const items      =
    (body.items as { item?: CultureListItem[] } | undefined)?.item ?? [];

  return { totalCount, items };
};

// 상세 파싱
export const parseDetailItem = (xml: string) => {
  const body = parseCultureResponse(xml);
  const item =
    (body.items as { item?: CultureDetailItem[] } | undefined)?.item?.[0] ?? null;

  return item;
};
```

```
body.items as { item?: CultureListItem[] }:
  body 는 Record<string, unknown> 타입
  → items 에 접근하려면 타입 단언 필요
  item?: CultureListItem[]  있을 수도 없을 수도 있음

?.[0]:
  배열의 첫 번째 요소
  배열이 비어있으면 undefined → ?? null 로 처리

parseDetailItem 이 [0] 만 반환하는 이유:
  상세 API 는 항상 1개만 옴
  isArray 로 배열이 됐으니 [0] 으로 꺼냄
```

---
---
# culture-api.client.ts — API 클라이언트 분리 ⭐️

```
서비스에서 fetch 를 직접 호출하면:
  URL 조합 / 에러 처리 / 파싱 로직이 서비스에 섞임
  엔드포인트가 늘어날수록 서비스가 복잡해짐

API Client 로 분리하면:
  URL 조합 / fetch / 에러 처리 → 클라이언트
  비즈니스 로직 → 서비스
  역할이 명확하게 분리됨
```

```typescript
// culture-api.client.ts
import { Injectable } from '@nestjs/common';
import { ConfigService } from '@nestjs/config';
import { EnvKeys } from 'src/config/env.keys';
import { parseDetailItem, parseListItems } from './culture-xml.parser';
import { CultureDetailItem, CultureListItem } from './culture-api.types';

@Injectable()
export class CultureApiClient {
  constructor(private readonly configService: ConfigService) {}

  // getter 로 반복 제거
  private get baseUrl() {
    return this.configService.getOrThrow<string>(EnvKeys.API_EXHIBITION_BASE_URL);
  }
  private get serviceKey() {
    return this.configService.getOrThrow<string>(EnvKeys.API_EXHIBITION_KEY);
  }

  // URL 조합 공통 메서드
  private buildUrl(endpoint: string, params: Record<string, string | number>) {
    const url = new URL(`${this.baseUrl.replace(/\/$/, '')}/${endpoint}`);
    url.searchParams.set('serviceKey', this.serviceKey);
    for (const [key, value] of Object.entries(params)) {
      url.searchParams.set(key, String(value));
    }
    return url.toString();
  }

  // 목록 조회
  fetchPeriod2Page = async (pageNo: number, numOfrows = 100) => {
    const url = this.buildUrl('period2', { pageNo, numOfrows, serviceTp: 'A' });
    const res = await fetch(url);
    if (!res.ok) {
      throw new Error(`period2 HTTP error! status: ${res.status}: ${url} ${res.statusText}`);
    }
    const xml = await res.text();
    return parseListItems(xml) as { totalCount: number; items: CultureListItem[] };
  };

  // 상세 조회
  fetchDetailItem = async (seq: string) => {
    const url = this.buildUrl('detail', { seq });
    const res = await fetch(url);
    if (!res.ok) {
      throw new Error(`detail HTTP error! status: ${res.status}: ${url} ${res.statusText}`);
    }
    const xml = await res.text();
    return parseDetailItem(xml) as CultureDetailItem | null;
  };
}
```

```
buildUrl 패턴 ⭐️:
  new URL(base) 로 URL 객체 생성
  url.searchParams.set() 으로 파라미터 추가
    → 특수문자 / 한글 자동 인코딩 처리
    → 직접 문자열 조합보다 안전

  baseUrl.replace(/\/$/, ''):
    baseUrl 끝에 / 가 있으면 제거
    https://api.example.com/ + /period2 → 중복 / 방지

  Object.entries(params):
    { pageNo: 1, numOfrows: 100 } → [['pageNo','1'], ['numOfrows','100']]
    한 번에 모든 파라미터 처리

res.ok 체크:
  fetch 는 4xx / 5xx 에서도 에러를 throw 하지 않음
  → res.ok (200~299) 직접 확인 필수
  → 실패 시 상세 정보 포함한 에러 throw
```

---

---
# culture.mapper.ts — API → Prisma 변환 ⭐️

```
API 에서 받은 데이터 형태와 DB 저장 형태가 다를 때
변환 로직을 mapper 에서 담당
```

```typescript
// culture.mapper.ts
export const mergeToExhibitionDate = (
  list:   CultureListItem,
  detail: CultureDetailItem | null,
): Prisma.ExhibitionCreateInput | null => {
  const merged = { ...list, ...(detail ?? {}) };

  const startDate = parseYmd(merged.startDate);  // "20260601" → Date
  const endDate   = parseYmd(merged.endDate);    // "20260630" → Date
  const seq       = merged.seq?.toString();

  // 필수 값 없으면 null 반환 → 수집 skip
  if (!seq || !startDate || !endDate) return null;

  return {
    seq,
    title:     merged.title ?? '',
    startDate,
    endDate,
    // ...
  };
};
```

```bash
parseYmd:
  공공 API 날짜 형식 "20260601" (YYYYMMDD) → Date 객체
  new Date("20260601") 은 환경마다 결과가 다를 수 있어서
  직접 파싱하는 유틸 함수 사용
  # → [[JS_Date]] parseYmd 섹션 참고

{ ...list, ...(detail ?? {}) }:
  목록 데이터 + 상세 데이터 병합
  detail 없으면 {} 로 대체 (목록 데이터만 사용)
  같은 키는 detail 값이 덮어씀

if (!seq || !startDate || !endDate) return null:
  필수 값 없으면 null 반환
  서비스에서 null 이면 skip 처리
```


---
---
# exhibition.service.ts — 서비스에서 사용

```typescript
@Injectable()
export class ExhibitionService {
  private readonly baseUrl: string;
  private readonly apiKey: string;

  constructor(private configService: ConfigService) {
    this.baseUrl = this.configService.getOrThrow(EnvKeys.API_EXHIBITION_BASE_URL);
    this.apiKey  = this.configService.getOrThrow(EnvKeys.API_EXHIBITION_KEY);
  }

  async fetchList(page = 1, size = 10) {
    const url = `${this.baseUrl}/period2?serviceKey=${this.apiKey}&PageNo=${page}&numOfrows=${size}&serviceTp=A`;
    const xml  = await fetch(url).then(res => res.text());

    return parseListItems(xml);   // 파서에 위임
  }

  async fetchDetail(seq: number) {
    const url = `${this.baseUrl}/detail2?serviceKey=${this.apiKey}&seq=${seq}`;
    const xml  = await fetch(url).then(res => res.text());

    return parseDetailItem(xml);  // 파서에 위임
  }
}
```

```bash
서비스 역할:
  URL 조합 + fetch 호출
  파싱은 culture-api.parser.ts 에 위임

fetch(url).then(res => res.text()):
  외부 API 는 JSON 이 아닌 XML 을 텍스트로 받음
  res.json() 아닌 res.text() 사용  # => [[JS_Fetch_API]] 참고 
```
---
---

# 환경변수 설정

```typescript
// src/config/env.keys.ts 에 추가
export const EnvKeys = {
  // 기존 키들...
  API_EXHIBITION_BASE_URL: 'API_EXHIBITION_BASE_URL',
  API_EXHIBITION_KEY:      'API_EXHIBITION_KEY',
} as const;
```

```properties
# .env
API_EXHIBITION_BASE_URL=https://apis.data.go.kr/B553457/cultureinfo
API_EXHIBITION_KEY=여기에_실제_키
```

---

---

# curl 테스트 → NestJS 구현 흐름

```
1. curl 로 API 구조 확인            → [[Linux_Curl_API]]
   curl -sS "${API_URL}/period2?..."

2. XML 응답 구조 파악
   어떤 태그에 데이터 있는지
   item 1개일 때 vs 여러 개일 때 구조 차이 확인

3. types 파일에 응답 타입 정의
   curl 에서 확인한 필드 기반으로 작성

4. parser 파일에 파싱 함수 작성
   parseCultureResponse  공통 에러 처리
   parseListItems        목록 변환
   parseDetailItem       상세 변환

5. 서비스에서 호출
   fetch → res.text() → parser 함수 호출
```