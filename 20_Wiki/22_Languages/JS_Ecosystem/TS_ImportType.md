---
aliases:
  - Barrel
  - import type
  - export type
  - 경로 별칭
tags:
  - TypeScript
related:
  - "[[00_JS_Ecosystem_HomePage]]"
  - "[[TS_TsConfig]]"
  - "[[NestJS_JwtGuard]]"
---
# TS_ImportType — import · export · 경로 별칭

>[!info]
>`import type` = 타입만 import — 컴파일 후 런타임 번들에서 완전히 제거됨.
> 값(함수·클래스)과 타입을 구분해서 import하면 번들 크기가 줄고 순환 참조 문제도 예방된다. 
> `@/` 경로 별칭은 tsconfig.json의 `paths`로 설정.
>  Barrel(index.ts) = 여러 모듈을 하나로 묶어 export.
>  `express.d.ts`의 `declare global`·`declare module`로 라이브러리 타입을 프로젝트 안에서 확장한다.

---

# import type — 타입만 import ⭐️⭐️⭐️⭐️

```typescript
// 일반 import — 값과 타입 모두
import { User, fetchUser } from './users';
//       ↑타입  ↑함수(값)

// import type — 타입만 (런타임에 존재하지 않음)
import type { User } from './users';
import { fetchUser }  from './users';
```

```txt
TypeScript의 타입은 컴파일 후 사라짐:
  TypeScript 코드 → tsc 컴파일 → JavaScript 코드
  타입 정보는 컴파일 타임에만 존재, 런타임(JS)에는 없음

  import { User } from './users'
  → JS로 컴파일되면 이 줄이 남아있음 (번들에 포함)
  → User가 실제 값(클래스 등)이 아니라면 빈 import가 됨

  import type { User } from './users'
  → 컴파일 후 이 줄 자체가 완전히 제거됨 (번들에 포함 안 됨)
```

## 언제 import type을 쓰는가 ⭐️⭐️⭐️⭐️

```typescript
// ✅ 타입만 사용할 때 — import type
import type { User }        from './types';
import type { ApiResponse }  from './api';
import type { ButtonProps }  from './Button';

// ✅ 값도 사용할 때 — 일반 import
import { fetchUser }         from './api';     // 함수 (런타임에 실행됨)
import { UserService }       from './service'; // 클래스 (런타임에 인스턴스화)
import { STATUS }            from './const';   // 상수 (런타임에 값을 읽음)

// ✅ 같은 파일에서 타입과 값을 같이 가져올 때
import { fetchUser }         from './api';
import type { ApiUser }      from './api';

// 또는 한 줄에서 구분
import { fetchUser, type ApiUser } from './api';
//                  ↑ 인라인 type 키워드
```

```txt
구분 기준:
  타입 선언에만 쓰인다면  → import type
  런타임에 실제로 호출/참조한다면 → 일반 import

  type ApiUser = { ... }           → import type
  const user: ApiUser = ...        → import type
  const service = new UserService  → 일반 import (런타임에 new 호출)
  fetchUser(id)                    → 일반 import (런타임에 함수 호출)
  if (status === STATUS.ACTIVE)    → 일반 import (런타임에 값 참조)
```

## import type의 장점

```txt
① 번들 크기 절감:
  타입 전용 파일을 import type으로 가져오면
  해당 파일이 번들에 포함되지 않음

② 순환 참조 방지:
  A가 B를, B가 A를 서로 import하면 순환 참조 에러
  타입만 필요하면 import type → 런타임 의존성을 끊음

③ 의도 명확화:
  "이건 타입으로만 쓴다"는 것을 코드로 표현
  팀원이 읽을 때 런타임 영향 없음을 바로 파악
```

---

# export type — 타입만 재수출 ⭐️⭐️⭐️

```typescript
// 타입만 export
export type { User, ApiResponse };

// 선언과 동시에 export
export type UserId = string;
export type Status = 'active' | 'inactive';

// 다른 파일에서 타입 재수출
export type { WithdrawResult } from './apiTypes';
export type { ButtonProps }    from './Button';
```

---

# 경로 별칭 — @/ ⭐️⭐️⭐️⭐️

```typescript
// 상대 경로 — 깊이에 따라 ../../../ 가 계속 늘어남
import { fetchUser } from '../../../lib/api/users';
import { Button }    from '../../components/Button';

// 경로 별칭 — 어디서 import해도 동일한 경로
import { fetchUser } from '@/lib/api/users';
import { Button }    from '@/components/Button';
```

## tsconfig.json 설정

```json
{
  "compilerOptions": {
    "baseUrl": ".",
    "paths": {
      "@/*": ["./src/*"]
    }
  }
}
```

```txt
@/* → ./src/* 매핑:
  @/lib/api     → src/lib/api
  @/components  → src/components
  @/types       → src/types

Next.js:
  next.config.js와 tsconfig.json 둘 다 설정해야 할 수 있음
  Next.js는 기본으로 @/를 지원하는 경우도 있음 (버전에 따라 다름)

NestJS:
  tsconfig.json + webpack/esbuild 설정이 필요할 수 있음
  nest-cli.json에 webpack 옵션 추가

모노레포:
  각 앱의 tsconfig.json에서 별도 설정
  루트 tsconfig에서 extends해서 공통 설정 관리
```

---

# Barrel — index.ts로 묶어서 export ⭐️⭐️⭐️⭐️

```txt
Barrel = 여러 모듈의 export를 index.ts 하나에서 다시 내보내는 패턴
여러 파일에서 각각 import하는 대신 하나의 경로에서 전부 가져올 수 있음
```

```typescript
// ❌ Barrel 없이
import { fetchUser }           from '@/lib/api/users';
import { fetchFriends }        from '@/lib/api/friends';
import { login }               from '@/lib/api/auth';
import { fetchNotifications }  from '@/lib/api/notifications';

// ✅ Barrel 사용
import { fetchUser, fetchFriends, login, fetchNotifications } from '@/lib/api';
```

## index.ts 작성

```typescript
// src/lib/api/index.ts
/**
 * API client barrel — 도메인은 @/lib/api/users 등으로도 import 가능.
 * 기존 @/lib/api import 호환 유지.
 */
export * from './auth';
export * from './friends';
export * from './recommendations';
export * from './users';
export * from './notifications';
export * from './support';

export type { WithdrawResult } from './apiTypes';  // 타입만 재수출
```

```txt
export type { WithdrawResult } from './apiTypes':
  WithdrawResult는 임의 이름이 아님
  apiTypes.ts 안에 실제로 이렇게 정의된 타입:
    export type WithdrawResult = { success: boolean; message: string; }

  export * from './apiTypes' 대신 export type { WithdrawResult }만 쓰는 이유:
  → apiTypes.ts 전체를 노출하고 싶지 않고, 필요한 타입만 선택적으로 공개
  → "이 파일에서 꺼낼 수 있는 것은 WithdrawResult 타입뿐이야"를 명확히

  자신의 코드에서 쓸 때:
    export type { MyResult }   from './myTypes';  // myTypes.ts에 정의된 타입
    export type { LoginResult } from './auth';     // auth.ts에 정의된 타입
```


```txt
export * from './module':
  해당 파일의 모든 named export를 그대로 재수출
  default export는 포함 안 됨

export type { Xxx }:
  타입만 재수출 — 런타임 번들에 포함 안 됨

파일 구조:
  src/lib/api/
  ├── index.ts        ← barrel
  ├── auth.ts
  ├── users.ts
  ├── friends.ts
  └── apiTypes.ts     ← 공통 타입
```

## 이름 충돌 처리

```typescript
// ❌ 두 파일에서 같은 이름을 export하면 충돌
export * from './users';  // fetchAll 포함
export * from './posts';  // fetchAll 포함 → 충돌!

// ✅ 별칭으로 해결
export { fetchAll as fetchAllUsers } from './users';
export { fetchAll as fetchAllPosts } from './posts';

// ✅ 또는 필요한 것만 명시
export { fetchUser, updateUser }  from './users';
export { fetchPost, createPost }  from './posts';
```

---

# .d.ts — 타입 선언 파일 ⭐️⭐️

```typescript
// types.d.ts 또는 global.d.ts — 전역 타입 선언
declare global {
  interface Window {
    analytics: Analytics;  // window.analytics 타입 추가
  }
}

// 외부 모듈 타입 추가
declare module 'some-library' {
  export function doSomething(): void;
}
```

```txt
.d.ts 파일:
  TypeScript 타입 정보만 담은 파일 (실제 코드 없음)
  라이브러리에 타입이 없을 때 직접 작성
  전역 타입을 추가할 때 (Window, process.env 등)

@types/ 패키지:
  pnpm add -D @types/node       → Node.js 타입
  pnpm add -D @types/react       → React 타입
  → 라이브러리의 .d.ts를 담은 패키지
```

---
# 타입 선언 병합 — express.d.ts 패턴 ⭐️⭐️⭐️⭐️

## 왜 필요한가

```txt
라이브러리(Express, express-session 등)의 타입이
내 코드의 실제 동작과 맞지 않을 때 타입을 "확장"하는 방법

예시:
  request.user → Express는 타입을 Express.User로 정의
  JwtAuthGuard가 request.user = { sub: userId } 로 저장
  → request.user.sub를 쓰면 TypeScript 에러
  → Express.User에 sub가 없기 때문

해결:
  express.d.ts 파일에서 Express.User 타입에 sub를 추가
  → 이후 request.user.sub가 타입 에러 없이 작동
```

## declare global — 전역 타입 확장

```typescript
// src/types/express.d.ts
import type { JwtPayload } from '../auth/jwt-payload';

declare global {
  namespace Express {
    interface User extends JwtPayload {}
    //              ↑ JwtPayload = { sub: string }
    //                Express.User에 sub 필드가 추가됨
  }
}

export {};  // ← 이게 없으면 작동 안 함 (아래 설명)
```

```typescript
// auth/jwt-payload.ts
export type JwtPayload {
  sub: string;  // userId
}
```

```txt
declare global:
  TypeScript에서 전역 스코프에 타입을 선언하는 방법
  이 파일 안에서만이 아니라 프로젝트 전체에 적용됨

namespace Express { interface User extends JwtPayload }:
  Express.User라는 기존 타입에 JwtPayload를 병합
  "선언 병합(Declaration Merging)" — 같은 이름의 interface는 합쳐짐
  → Express.User가 { sub: string }을 갖게 됨
  → request.user.sub 타입 에러 없음

Passport / @types/express 관례:
  request.user의 타입이 Express.User로 정해져 있음
  JwtAuthGuard에서 request.user = payload 저장 시
  payload 타입과 Express.User가 일치해야 에러 없음
```

## export {} — 왜 필요한가 ⭐️⭐️⭐️

```typescript
// export {}가 없으면
declare global { ... }   // ← 이 경우 파일이 "ambient 모듈"
                         // → declare global이 작동하지 않을 수 있음

// export {}가 있으면
export {};               // 이 파일이 "모듈"로 인식됨
declare global { ... }   // → 모듈 안에서 declare global이 정상 작동
```

```txt
TypeScript 파일 구분:
  모듈:          import 또는 export가 하나라도 있는 파일
  스크립트(전역): import/export가 전혀 없는 파일

declare global은 모듈 안에서 써야 의도대로 작동
export {}는 실제 export하는 것 없이 파일을 모듈로 만드는 트릭
```

## declare module — 서드파티 모듈 타입 확장

```typescript
// src/types/express.d.ts
import type { JwtPayload }   from '../auth/jwt-payload';
import type { OAuthProfile } from '../auth/oauth-profile';
import type { Session }      from 'express-session';

// express-session 모듈의 SessionData 타입 확장
declare module 'express-session' {
  interface SessionData {
    oauthNext?: string;   // OAuth 완료 후 돌아갈 경로 저장용
  }
}

// Express 전역 타입 확장
declare global {
  namespace Express {
    interface User extends JwtPayload {}
    interface Request {
      session: Session;   // session 타입 명시
    }
  }
}

export {};
```

```txt
declare module '모듈이름':
  특정 라이브러리의 타입을 프로젝트 안에서만 확장
  해당 모듈의 타입 파일을 직접 수정하는 게 아님 (npm 업데이트로 날아가니까)

express-session의 SessionData에 oauthNext 추가:
  기본 SessionData에는 oauthNext가 없음
  declare module로 추가하면 → req.session.oauthNext 타입 에러 없음

declare global vs declare module:
  declare global   → 전역 네임스페이스(Express.User 등) 확장
  declare module   → 특정 라이브러리의 타입 확장
```

## 파일 위치와 tsconfig 연결

```typescript
// src/types/express.d.ts 위치
```

```json
// tsconfig.json — typeRoots 또는 include로 인식시킴
{
  "compilerOptions": {
    "typeRoots": ["./src/types", "./node_modules/@types"]
  }
}
// 또는 include에 types 폴더가 포함되면 자동 인식
// "include": ["src/**/*"]  ← src/types/*.d.ts 포함됨
```

```txt
.d.ts 파일이 tsconfig include 범위 안에 있으면 자동으로 적용
별도 import 없이도 프로젝트 전체에서 타입 확장이 반영됨
```
---

# 자주 쓰는 패턴

```typescript
// 1. 타입 파일 따로 분리
// types/api.ts
export type ApiUser = { id: string; nickname: string; };
export type ApiPost = { id: string; title: string; };

// 사용처
import type { ApiUser, ApiPost } from '@/types/api';

// 2. 컴포넌트 props 타입
// Button.tsx
export type ButtonProps = {
  children: React.ReactNode;
  onClick:  () => void;
  variant?: 'primary' | 'secondary';
};
export function Button({ children, onClick, variant = 'primary' }: ButtonProps) { ... }

// 사용처
import { Button }           from '@/components/Button';
import type { ButtonProps } from '@/components/Button';  // 타입만 필요할 때
// 또는
import { Button, type ButtonProps } from '@/components/Button';

// 3. 서드파티 타입 re-export
import type { Transporter } from 'nodemailer';
export type { Transporter };  // 내부 모듈에서 쓸 수 있게 재수출
```