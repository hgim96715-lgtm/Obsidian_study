---
aliases:
  - 역직렬화
  - 직렬화
  - JSON
  - pasrse
  - stringify
tags:
  - JavaScript
related:
  - "[[00_JS_Ecosystem_HomePage]]"
  - "[[NextJS_API_Client]]"
---
# JS_JSON — JSON

>[!info]
>JSON = 데이터를 텍스트로 표현하는 형식. 
>`JSON.stringify(obj)` = 객체 → 문자열, `JSON.parse(str)` = 문자열 → 객체. 
>fetch API body 직렬화, localStorage 저장, 깊은 복사에 사용.
> `undefined`·함수·`Date`는 stringify 시 주의 필요.

---

# JSON이란 ⭐️⭐️⭐️⭐️

```txt
JSON (JavaScript Object Notation):
  데이터를 텍스트(문자열)로 표현하는 형식
  언어에 상관없이 어디서나 읽고 쓸 수 있는 표준 데이터 교환 형식

  서버 ↔ 클라이언트가 데이터를 주고받을 때 거의 항상 JSON 사용

  JavaScript 객체와 비슷하게 생겼지만 다름:
  JavaScript 객체 = 코드, 메모리에 존재
  JSON 문자열    = 텍스트, 파일·네트워크로 전송 가능
```

```json
// JSON 형태
{
  "id": 1,
  "email": "user@example.com",
  "active": true,
  "tags": ["admin", "user"],
  "profile": {
    "name": "홍길동"
  }
}
```

```txt
JSON 규칙 (JavaScript 객체와 다른 점):
  키는 반드시 큰따옴표("") — 작은따옴표 불가
  문자열 값도 큰따옴표만
  undefined 값 없음 (null은 있음)
  함수 없음
  주석 없음
  마지막 항목 뒤에 쉼표 없음 (trailing comma 금지)
```

---

# 언제 JSON이 필요한가 ⭐️⭐️⭐️⭐️

```typescript
// 1. fetch API body — 객체를 서버로 보낼 때
fetch('/api/login', {
  method: 'POST',
  headers: { 'Content-Type': 'application/json' },
  body: JSON.stringify({ email, password }),
  //   ↑ 객체 → 문자열 변환 필수 (body는 문자열이어야 함)
});

// 2. localStorage — 브라우저 저장소는 문자열만 저장 가능
localStorage.setItem('user', JSON.stringify({ id: 1, name: '홍길동' }));
const user = JSON.parse(localStorage.getItem('user') ?? '{}');

// 3. 깊은 복사 (간단한 경우)
const original = { a: { b: 1 } };
const copy = JSON.parse(JSON.stringify(original));
copy.a.b = 2;
original.a.b;  // 1 — 영향 없음

// 4. API 응답 → 화면에 표시 (자동으로 처리됨)
const data = await res.json();
// res.json() 내부에서 JSON.parse를 호출함
// 개발자가 직접 JSON.parse를 쓸 필요 없음
```

---

# JSON.stringify — 객체 → 문자열 ⭐️⭐️⭐️⭐️

```typescript
const user = { id: 1, name: '홍길동', email: 'hong@example.com' };

JSON.stringify(user)
// '{"id":1,"name":"홍길동","email":"hong@example.com"}'
// → 공백 없는 한 줄 문자열

JSON.stringify(user, null, 2)
// null = replacer (아래 설명), 2 = 들여쓰기 칸 수
// {
//   "id": 1,
//   "name": "홍길동",
//   "email": "hong@example.com"
// }
// → 사람이 읽기 좋은 형태 (디버깅·로그용)
```

## stringify 시 사라지는 값들 ⭐️⭐️⭐️⭐️

```typescript
const obj = {
  name:      '홍길동',
  age:       undefined,      // ← undefined는 사라짐
  greet:     () => 'hello',  // ← 함수는 사라짐
  createdAt: new Date(),     // ← Date는 문자열로 변환
  score:     NaN,            // ← NaN은 null로 변환
  inf:       Infinity,       // ← Infinity는 null로 변환
};

JSON.stringify(obj)
// '{"name":"홍길동","createdAt":"2024-01-15T00:00:00.000Z","score":null,"inf":null}'
// age, greet 사라짐
```

```txt
⚠️ 주의:
  undefined → 키 자체가 사라짐
  함수 → 키 자체가 사라짐
  Date → ISO 문자열로 변환 ("2024-01-15T00:00:00.000Z")
         → JSON.parse 해도 Date 객체가 아닌 문자열로 복원됨
  NaN, Infinity → null로 변환
```

---

# JSON.parse — 문자열 → 객체 ⭐️⭐️⭐️⭐️

```typescript
const str = '{"id":1,"name":"홍길동"}';

const user = JSON.parse(str);
user.id;    // 1 (number)
user.name;  // '홍길동' (string)
```

## try/catch 필수 ⭐️⭐️⭐️⭐️

```typescript
// ❌ 에러 처리 없이 쓰면
JSON.parse('invalid json');  // SyntaxError: Unexpected token i

// ✅ try/catch로 감싸야 함
function safeParseJson<T>(str: string, fallback: T): T {
  try {
    return JSON.parse(str) as T;
  } catch {
    return fallback;
  }
}

// 사용
const user = safeParseJson(localStorage.getItem('user') ?? '', null);
//                          ↑ null이면 빈 문자열, parse하면 에러
//                          → fallback인 null 반환
```

```txt
JSON.parse가 throw하는 경우:
  유효하지 않은 JSON 문자열
    JSON.parse('undefined')  → 에러
    JSON.parse('')            → 에러
    JSON.parse("hello")       → 에러 (따옴표 없는 문자열)

  localStorage.getItem이 null을 반환할 때
    JSON.parse(null)          → 에러는 아님, null 반환 (예외)
    JSON.parse(null)          → null (JavaScript가 toString 적용)
    → 안전하게 ?? '' 또는 ?? '{}' 로 기본값 설정 권장
```

---

# Date 변환 주의 ⭐️⭐️⭐️

```typescript
const original = { createdAt: new Date('2024-01-15') };

const str = JSON.stringify(original);
// '{"createdAt":"2024-01-15T00:00:00.000Z"}'

const parsed = JSON.parse(str);
parsed.createdAt;             // '2024-01-15T00:00:00.000Z' (문자열!)
parsed.createdAt instanceof Date;  // false

// Date로 복원하려면 직접 변환 필요
const date = new Date(parsed.createdAt);
```

```txt
API 응답에서도 날짜는 문자열로 옴
  → TypeScript 타입에 Date 대신 string 사용
  → 화면에 표시할 때 new Date(str) 또는 Intl로 포맷
  → [[JS_Intl]], [[JS_Date]] 참고
```

---

# 깊은 복사 — JSON 방식의 한계 ⭐️⭐️⭐️

```typescript
// 간단한 경우 작동
const copy = JSON.parse(JSON.stringify(original));

// ❌ 작동 안 하는 경우
const withFn = { fn: () => 'hello' };
JSON.parse(JSON.stringify(withFn));  // {}  함수 사라짐

const withDate = { date: new Date() };
JSON.parse(JSON.stringify(withDate)).date instanceof Date;  // false (문자열)

const withUndefined = { a: undefined };
JSON.parse(JSON.stringify(withUndefined));  // {}  undefined 사라짐
```

```txt
JSON 깊은 복사 사용 기준:
  ✅ 순수 데이터 객체 (number, string, boolean, null, array, plain object)
  ❌ 함수, Date, undefined, Map, Set, Class 인스턴스 포함 시

  완벽한 깊은 복사가 필요하면:
  structuredClone(obj)  ← 브라우저 내장 (모던 브라우저)
  lodash cloneDeep()    ← 라이브러리
```

---

# replacer · reviver — 고급 변환 ⭐️⭐️

```typescript
// replacer — stringify 시 값 변환
JSON.stringify(data, (key, value) => {
  if (typeof value === 'function') return undefined;  // 함수 제외
  if (value instanceof Date) return value.toISOString();
  return value;
});

// 특정 키만 포함
JSON.stringify(user, ['id', 'name']);  // id, name만 포함

// reviver — parse 시 값 변환
JSON.parse(str, (key, value) => {
  if (key === 'createdAt') return new Date(value);  // 날짜 복원
  return value;
});
```

---

# 자주 쓰는 패턴 ⭐️⭐️⭐️

```typescript
// fetch body
body: JSON.stringify({ email, password })

// localStorage 저장/읽기
localStorage.setItem('data', JSON.stringify(obj));
JSON.parse(localStorage.getItem('data') ?? '{}')

// 디버깅 — 객체를 보기 좋게 출력
console.log(JSON.stringify(complexObj, null, 2));

// 간단한 깊은 복사
const copy = JSON.parse(JSON.stringify(plainObj));

// API 응답 타입 확인
console.log(JSON.stringify(res.data, null, 2));
```