---
aliases:
  - PATCH body
  - partial update
  - optional field
tags:
  - TypeScript
related:
  - "[[00_JS_Ecosystem_HomePage]]"
  - "[[TS_Utility_Types]]"
  - "[[NestJS_DTO]]"
  - "[[JS_Truthy_Falsy]]"
  - "[[JS_OptionalChaining]]"
---
# TS_PartialUpdate — 부분 수정(PATCH) 요청 객체 만들기

> [!info] 
> PATCH처럼 "일부 필드만 수정"하는 요청을 만들 때는, 모든 필드를 한 번에 객체 리터럴로 만들지 않고 빈 객체에서 시작해서 조건에 따라 하나씩 채워나간다. 
> 이렇게 하면 "건드리지 않은 필드"(키 자체가 없음)와 "명시적으로 지운 필드"(null)와 "새 값"을 구분할 수 있다.

---

# 인라인 타입 — { title?: string; description?: string | null } ⭐️⭐️⭐️⭐️

```typescript
const data: { title?: string; description?: string | null } = {};
```

```txt
이름 있는 interface/type을 따로 안 만들고, 변수 선언 자리에 바로 타입을 적은 것 —
이 변수 하나에만 쓰이는 일회성 모양이라 굳이 이름 붙일 가치가 없을 때 흔히 씀

? (옵셔널)이 필요한 이유:
  시작이 {}(빈 객체)이므로, "어떤 필드도 아직 없는 상태"도 이 타입이 허용해야 함
  필드를 옵셔널로 안 적으면, {} 자체가 이 타입과 안 맞는다고 TS가 에러를 냄

description?: string | null — 옵셔널(?)과 | null은 서로 다른 걸 표현함:
  ?         "이 키 자체가 객체에 없을 수도 있다"
  | null    "이 키가 있다면, 그 값이 string이거나 null일 수 있다"
  → "필드가 아예 없음"과 "필드는 있는데 값이 null임"은 서로 다른 상태이고, 이 둘을 같이 표현한 것
```

---

# 빈 객체로 시작해서 조건부로 채우는 이유 ⭐️⭐️⭐️⭐️

```typescript
const data: { title?: string; description?: string | null } = {};

if (title !== undefined) data.title = title;
data.description = description === '' ? null : description;
```

```txt
처음부터 { title, description } 로 한 번에 만들면 생기는 문제:
  title/description 둘 다 무조건 객체의 키로 들어감(값이 undefined여도 키 자체는 존재)
  → PATCH의 의미는 "보낸 필드만 바꿔라"인데, 안 바꾸고 싶은 필드까지 같이 보내게 되어버림

빈 객체에서 시작 + 조건을 만족할 때만 할당:
  → "이 요청에 실제로 포함시킬 필드만" 골라서 담을 수 있음
  → 조건을 안 만족한 필드는 data에 그 키 자체가 끝까지 안 생김
```

---

# null / undefined / 키 자체가 없음 — 세 가지 다른 의미 ⭐️⭐️⭐️⭐️⭐️

|상태|의미|
|---|---|
|`data`에 `description` 키 자체가 없음|"이 필드는 안 건드린다" — PATCH의 기본값|
|`data.description === null`|"이 필드를 명시적으로 비웠다" (지워라)|
|`data.description === '실제값'`|"이 필드를 이 값으로 바꿔라"|

```txt
data.description = description === '' ? null : description; 한 줄이 정확히 하는 일:

  사용자가 입력칸을 비웠다(description === '')  → "지워달라"는 의도로 해석 → null로 정규화해서 담음
  사용자가 입력칸에 값을 넣었다        → 그 값 그대로 담음

빈 문자열('')을 그대로 보내면 안 되는 이유:
  백엔드/DB 입장에서 ''(빈 문자열 자체)와 null은 보통 다른 의미로 처리됨
  ''는 "내용이 빈 문자열인 유효한 값"으로 보일 수 있어서 "지우려는 의도"가 명확히 안 드러남
  → null로 명시해야 "값이 없다"는 의도가 정확히 전달됨
  (falsy 값들 사이의 차이 자체는 [[JS_Truthy_Falsy]] 참고)
```

---

# 조건 패턴 변형들 ⭐️⭐️⭐️

```typescript
// 값이 undefined가 아닐 때만 포함 — "값이 전달됐을 때만 갱신"
if (title !== undefined) data.title = title;

// falsy 전체를 거를지, null/undefined만 거를지에 따라 다른 연산자 선택
data.description = description ?? null;          // description이 null/undefined일 때만 null로 (??는 falsy 전체가 아님)
data.description = description === '' ? null : description; // 빈 문자열만 특별히 null로 취급하고 싶을 때 (의도적으로 ===)
```

```txt
?? 와 === '' 의 차이를 의도에 맞게 고르는 게 중요함:
  description ?? null              description이 null/undefined일 때만 null — 빈 문자열('')은 그대로 통과시킴
  description === '' ? null : description  빈 문자열을 명시적으로 null 취급하고 싶을 때 — 의도가 다름

  (??가 falsy 전체가 아니라 null/undefined만 본다는 동작 자체는 [[JS_OptionalChaining]] 참고)
```

---

# 백엔드와의 짝 — PartialType + @IsOptional() ⭐️⭐️⭐️

```txt
프론트에서 이렇게 만든 부분 수정 객체는, 보통 NestJS의 UpdateDto(PartialType(CreateDto))가
받는 쪽임 — 모든 필드가 @IsOptional()로 선언돼 있어서, 보낸 필드만 검증/적용되고
안 보낸 필드(키 자체가 없는 필드)는 검증을 건너뛰고 그대로 안 바뀜
(자세한 PartialType 동작은 [[NestJS_DTO]]의 "Mapped Types" 참고)

→ 프론트의 "조건부로 키를 채우는" 패턴과 백엔드의 "옵셔널 필드만 검증하는" 패턴은
  사실 같은 의도(PATCH = 보낸 것만 바꾼다)를 양쪽에서 맞춰주는 한 쌍임
```

---

# 한눈에

| 패턴                                        | 핵심                                                              |
| ----------------------------------------- | --------------------------------------------------------------- |
| `{ field?: Type } = {}`                   | 이름 없는 인라인 타입 + 빈 객체로 시작 — 일회성 부분 객체에 흔함                         |
| `field?: Type \| null`                    | "키가 없을 수도"(`?`)와 "값이 null일 수도"(`\| null`)는 서로 다른 표현             |
| 조건부 할당                                    | 모든 필드를 한 번에 만들지 않고, 조건을 만족할 때만 키를 추가                            |
| 키 없음 vs `null` vs 값                       | "안 건드림" vs "명시적으로 지움" vs "새 값" — PATCH에서 세 가지 다른 의미             |
| `description === '' ? null : description` | 빈 문자열 입력을 "지우려는 의도"로 해석해 `null`로 정규화                            |
| 백엔드 짝                                     | NestJS `PartialType` + `@IsOptional()` — 같은 "보낸 것만 바꾼다" 의도의 반대편 |