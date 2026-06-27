---
aliases:
  - implements
  - extends
  - readonly
  - parameter properties
tags:
  - TypeScript
related:
  - "[[00_JS_Ecosystem_HomePage]]"
  - "[[NestJS_JwtGuard]]"
  - "[[NestJS_Response]]"
---

# TS_Class_Patterns — interface/implements, extends, readonly

> [!info] 
>  `implements`는 "이 클래스가 반드시 이 모양을 갖추겠다"는 약속, `extends`는 "기존 클래스의 기능을 이어받는다"는 상속이다. 
>  둘은 비슷해 보여도 목적이 다르다. `readonly`와 생성자 매개변수 축약은 클래스 필드를 더 짧고 안전하게 선언하는 문법이다.

---

# interface + implements — "이 모양을 갖추겠다"는 약속 ⭐️⭐️⭐️⭐️

```typescript
interface CanActivate {
  canActivate(context: ExecutionContext): boolean;
}

@Injectable()
export class RolesGuard implements CanActivate {
  canActivate(context: ExecutionContext): boolean {
    // ...
    return true;
  }
}
```

```
interface CanActivate는 "canActivate라는 메서드가 있어야 한다"는 모양(계약)만 정의함 — 구현은 없음
implements CanActivate를 쓰면, RolesGuard는 반드시 그 모양대로 canActivate 메서드를
구현해야 함 — 안 구현하면 그 자리에서 컴파일 에러

→ NestJS가 "Guard로 등록된 모든 클래스는 canActivate를 가지고 있다"고 믿고 호출할 수 있는 이유가
  바로 이 implements 계약 덕분임 (런타임에 진짜로 있는지 매번 확인 안 해도 됨)
```

```
같은 발상이 반복되는 다른 예:
  ExceptionFilter   → catch(exception, host) 메서드를 약속
  CanActivate       → canActivate(context) 메서드를 약속
  → "이런 역할을 하는 클래스라면 반드시 이 메서드가 있다"는 약속을 interface로 강제
```

---

# extends — 기존 클래스의 기능을 이어받기 ⭐️⭐️⭐️

```typescript
export class ApiError extends Error {
  constructor(message: string, public readonly status: number) {
    super(message); // 부모(Error)의 생성자를 먼저 실행
    this.name = 'ApiError';
  }
}
```

```
extends Error — Error가 원래 갖고 있던 기능(message, stack trace 등)을 그대로 물려받고,
거기에 status라는 새 필드를 추가한 "확장판" 클래스를 만드는 것

super(message) — 부모 클래스(Error)의 생성자를 먼저 호출해야 함
  (자식 클래스 생성자 안에서 this를 쓰기 전에, 부모가 먼저 자기 몫의 초기화를 끝내야 하기 때문)
```

|구분|언제 쓰나|
|---|---|
|`implements`|"이런 메서드를 가진 모양"이라는 약속만 강제 — 코드(구현)는 안 물려받음|
|`extends`|기존 클래스의 실제 코드(필드/메서드 구현)까지 그대로 물려받음|

```
ApiError extends Error를 쓰는 이유:
  완전히 새로 클래스를 만드는 대신, JS/TS에 이미 있는 Error의 기능(스택 트레이스 등)을
  그대로 누리면서 status라는 필요한 정보만 추가하는 게 더 안전하고 적은 코드로 가능함
```

---

# readonly — 생성자에서만 정하고 그 뒤로는 못 바꾸게 ⭐️⭐️

```typescript
export class ApiError extends Error {
  constructor(
    message: string,
    public readonly status: number,  // 생성 시점에만 값이 정해지고, 이후 재할당 불가
  ) {
    super(message);
  }
}

const err = new ApiError('실패', 404);
err.status;       // 404 — 읽기는 가능
err.status = 500; // ❌ 컴파일 에러 — readonly 필드는 재할당 불가
```

```
왜 readonly를 붙이나: status처럼 "한 번 정해지면 그 객체가 살아있는 동안 절대 안 바뀌어야 하는 값"에
표시해두면, 나중에 누군가(또는 미래의 나)가 실수로 바꾸는 코드를 컴파일 단계에서 미리 막아줌
```

---

# 생성자 매개변수 축약 — public/readonly가 생성자에 바로 붙는 이유 ⭐️⭐️⭐️⭐️

```typescript
// 길게 쓰면 이렇게 — 필드 선언 + 생성자에서 대입, 두 단계
class ApiErrorLong extends Error {
  public readonly status: number;  // ① 필드 선언

  constructor(message: string, status: number) {
    super(message);
    this.status = status;          // ② 생성자에서 대입
  }
}

// 축약하면 — 생성자 매개변수 앞에 public/readonly를 붙이는 것만으로 ①②가 한 줄에 합쳐짐
class ApiErrorShort extends Error {
  constructor(message: string, public readonly status: number) {
    super(message);
  }
}
```

```
constructor(message: string, public readonly status: number) 처럼
매개변수 앞에 public(또는 private/protected)을 붙이면:
  1. 클래스에 그 이름의 필드를 자동으로 선언하고
  2. 생성자 실행 시 그 매개변수 값을 그 필드에 자동으로 대입까지 해줌

→ "필드 선언"과 "생성자에서 대입"을 한 줄로 합쳐주는 TS만의 축약 문법(parameter properties)
  message는 public을 안 붙였으니 필드로 안 남고, 그냥 super()에 전달되는 일반 매개변수로만 쓰임
```

---

# 한눈에

| 문법                                  | 역할                                      |
| ----------------------------------- | --------------------------------------- |
| `interface X { ... }`               | 메서드/필드의 "모양"만 정의 — 구현 없음                |
| `implements X`                      | 이 클래스가 그 모양을 반드시 갖추겠다는 약속, 안 지키면 컴파일 에러 |
| `extends X`                         | 기존 클래스의 실제 구현(필드/메서드)까지 물려받음            |
| `super(...)`                        | 부모 클래스의 생성자를 먼저 실행                      |
| `readonly 필드`                       | 생성 시점에만 값이 정해지고 이후 재할당 불가               |
| `constructor(public readonly x: T)` | 필드 선언 + 생성자 대입을 한 줄로 합치는 축약 문법          |