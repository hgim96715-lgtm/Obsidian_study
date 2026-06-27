---
aliases:
  - UI 타입
  - types.ts
  - lib/types
tags:
  - NextJS
related:
  - "[[00_JS_Ecosystem_HomePage]]"
  - "[[NextJS_Concept]]"
  - "[[NextJS_Env_Config]]"
  - "[[NextJS_API_Mapper]]"
  - "[[NestJS_DTO]]"
---
# NextJS_UI_Types — UI 타입 먼저 설계하기

# 한 줄 요약

```
프로젝트 시작할 때 설치/env 다음으로 하는 일 — lib/types.ts(또는 types/) 에
"화면이 실제로 쓰는 모양" 의 타입을 먼저 정의
API 응답 타입을 그대로 베끼는 게 아니라, 화면 기준으로 새로 설계하는 것이 핵심
```

## 이 노트가 프로젝트 시작 순서에서 어디 들어가는지

| 단계                  | 내용                              | 참고                    |
| ------------------- | ------------------------------- | --------------------- |
| 설치                  | `create-next-app` 등             | [[NextJS_Concept]]    |
| env                 | `NEXT_PUBLIC_API_URL` 등         | [[NextJS_Env_Config]] |
| **UI 타입 설계 (이 노트)** | `lib/types.ts` — 화면이 쓰는 모양부터 정의 | —                     |
| API 연동              | fetchAPI, Mapper                | [[NextJS_API_Mapper]] |

```
백엔드가 아직 없거나 엔드포인트가 안 끝났어도, 화면에 "뭘 보여줄지" 는 먼저 정할 수 있음
→ UI 타입을 먼저 만들어두면 mock 데이터로 컴포넌트 개발을 먼저 진행할 수 있고
  나중에 API 가 붙을 때도 "이 타입에 맞게 변환만 하면 된다" 는 명확한 목표가 생김
```

---

---

# 왜 API 응답과 1:1 이 아닌가 ⭐️⭐️⭐️

```
화면(UI)이 필요한 모양과 API/DB 가 들고 있는 모양은 책임이 달라서, 보통 서로 다름
```

|이유|설명|일반화된 예시|
|---|---|---|
|API 엔 없는데 화면엔 필요한 필드|다른 테이블/관계를 합치거나 집계해야 함 — Mapper 가 채움|`author`(작성자 정보 join), `likeCount`(집계 결과)|
|있을 수도 없을 수도 있는 필드|로그인 여부 등 "맥락" 에 따라 달라짐|`likedByMe?`(비로그인이면 없음)|
|API/DB 엔 있지만 화면엔 안 보여줄 필드|내부 관리용이거나 이 화면에서 안 씀|`hidden`(노출 플래그), `updatedAt`, 상세 메타데이터|

---

---

# 일반화된 템플릿 ⭐️⭐️⭐️

```typescript
// lib/types.ts
/** 피드 카드/페이지가 쓰는 UI 타입 — 예시 도메인: Post */

// 관계 엔티티는 "화면이 필요한 만큼만" 축소된 형태로
export type Author = {
  id: string;
  nickname: string;
  image?: string;
};

/** 화면 표시용 — mapPost.ts 가 API 응답을 이 타입으로 변환 */
export type Post = {
  id: string;
  title: string;
  content: string;
  tags: string[];          // apiTypes 와 동일 — 허용값은 백엔드(단일 소스)가 검증, 지금은 string[] 로 충분(아래 참고)
  likeCount: number;       // API 엔 없을 수 있음 — Mapper 가 채움
  likedByMe?: boolean;     // 비로그인이면 없음 — optional
  author: Author;          // API 응답엔 authorId 만 있을 수도 — Mapper 가 객체로 채움
  createdAt: string;       // ISO 8601 문자열
  // hidden / updatedAt / reactions 같은 내부·상세 필드는 의도적으로 제외
};

/** POST 요청 body — 백엔드 DTO 와 필드를 맞춤 (쓰기 전용, 위 Post 와는 별도 타입) */
export type CreatePostBody = {
  title: string;
  content: string;
  tags: string[];
};
```

## 포인트 3개 — 일반화

```
① author / likeCount / likedByMe 처럼 API 에 없거나 다른 형태인 필드는 Mapper 가 나중에 채움
   ([[NextJS_API_Integration]] 의 "API 타입 vs UI 타입 — 매퍼 패턴" 참고)
② optional(?) 로 표시된 필드는 "항상 있는 게 아니다" 라는 신호 — 로그인 여부 등 맥락에 따라 갈림
③ 화면에 안 쓰는 API/DB 필드(내부 플래그, 상세 메타데이터 등)는 UI 타입에 일부러 안 넣음
   → 백엔드 스펙 문서(API 명세 등)를 기준으로 "이 화면에 필요한 필드만" 추려서 적음
```

---

---

# 고정된 선택지 필드 — 지금 union 으로 좁혀야 하나? ⭐️⭐️⭐️

```
tags(또는 moods, status 처럼 "정해진 값 중 하나" 인 필드)를 types.ts 에서 처음부터
'A' | 'B' | 'C' 같은 union 으로 적고 싶을 수 있는데, 보통 그 시점에는 아직 필요 없음
```

|시점|`tags` 의 타입|이유|
|---|---|---|
|처음 — 피드에 그냥 표시만 할 때|`string[]`|허용값의 진짜 출처는 백엔드(DTO + 상수 파일) — 프론트에 똑같이 다시 적으면 두 곳을 항상 같이 고쳐야 함(중복)|
|나중 — 칩/선택 UI 등 "값 하나하나를 알아야" 할 때|`Tag[]`(union)|버튼을 값마다 하나씩 그리려면, 그 값들이 정확히 뭔지 프론트도 알아야 함 — 이때가 진짜로 필요해지는 시점|

```
"나중" 이 오면 — 그 시점에 프론트 전용 상수 파일을 따로 만듦(백엔드 걸 의도적으로 복사):
```

```typescript
// lib/tags.ts — 칩 UI 를 만들 때가 되면 추가 (백엔드 constants/tags.ts 의 복사본)
/** 백엔드 TAGS 와 동일하게 유지할 것 — 칩 UI 렌더링용 */
export const TAGS = ['A', 'B', 'C'] as const;
export type Tag = (typeof TAGS)[number];
```

```
이렇게 나누는 이유:
  types.ts 의 Post.tags: string[] 는 "이 값이 무엇이든 화면에 표시는 가능하다" 는 최소 가정만 함
  → 백엔드 허용값이 바뀌어도(태그 추가/삭제) types.ts 를 고칠 일이 없음

  반대로 lib/tags.ts 의 TAGS 는 "내가 정확히 이 값들을 알아야 버튼을 그릴 수 있다" 는 화면의 진짜 필요
  → 이건 명백히 "복사본" 이라는 걸 주석으로 남겨두고, 백엔드 값이 바뀌면 같이 고쳐야 함(자동 동기화 안 됨)

⚠️ 단일 소스(source of truth)는 항상 백엔드 — DTO 의 @IsIn 이 실제로 검증하는 쪽이 "진짜" 기준
  프론트의 복사본은 화면을 그리기 위한 파생물일 뿐, 어긋나면 백엔드 기준으로 맞춰야 함
  (백엔드 쪽 단일 소스 패턴은 [[NestJS_DTO]] 의 "허용 값을 별도 상수 파일로 분리하기" 참고)
```

# 읽기 타입 vs 쓰기 타입을 분리하는 이유 ⭐️

```
Post(읽기/표시용)와 CreatePostBody(쓰기용)를 같은 타입으로 합치면:
  생성 시 필요 없는 필드(id, likeCount, createdAt 등)까지 다 채워야 하는 것처럼 보임
→ 두 타입을 분리하면 "이 요청에 진짜 필요한 필드" 만 명확하게 드러남
```

|타입|용도|특징|
|---|---|---|
|`Post`|화면 표시(읽기)|서버가 만들어낸 값(id, 집계값, 관계) 포함|
|`CreatePostBody`|생성 요청(쓰기)|사용자가 입력하는 값만, 보통 백엔드 DTO 와 필드 1:1|

---

---

# 언제 만드나 — API 연동 전에 먼저 ⭐️

```
1. Post 타입만 먼저 만들어둠 (백엔드 안 기다려도 됨)
2. mock 데이터([{ id: '1', title: '...', ... }])로 컴포넌트부터 개발
3. API 가 준비되면 mapPost(apiResponse) 가 이 타입에 맞게 변환만 해주면 끝
   (Mapper 작성법은 [[NextJS_API_Integration]] 참고)
```

---

---

# 한눈에

```
프로젝트 시작 순서: 설치 → env → UI 타입(이 노트) → 컴포넌트(mock 으로 먼저 가능) → API 연동(Mapper)

UI 타입은 API 응답의 부분집합이 아님 — 화면 기준으로 새로 설계, Mapper 가 그 차이를 메움
optional(?) = 맥락에 따라 있을 수도 없을 수도 있는 필드라는 신호
읽기 타입과 쓰기(Body) 타입은 분리 — 생성 시 불필요한 필드까지 강제로 채우는 걸 방지
고정된 선택지 필드(tags 등)는 처음엔 string[] 로 충분 — 칩 UI 등 값 하나하나가 필요해질 때만 union 으로 좁히기
  (그 시점에 만드는 프론트 상수 파일은 복사본일 뿐, 단일 소스는 항상 백엔드)

매퍼 패턴 자체의 자세한 내용 → [[NextJS_API_Integration]]
apiTypes.ts 작성법과 매퍼 함수의 의존 방향 규칙 → [[NextJS_API_Mapper]]
lib/, types/ 폴더 위치 자체 → [[NextJS_Concept]]
이 타입이 실제로 컴포넌트 props 가 되는 패턴({Component}Props) → [[React_Component]]
```