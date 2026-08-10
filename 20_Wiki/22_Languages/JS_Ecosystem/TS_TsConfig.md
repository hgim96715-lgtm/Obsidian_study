---
aliases:
  - Monorepo
  - tsconfig.json
  - "strictPropertyInitialization: false — NestJS DTO"
tags:
  - TypeScript
related:
  - "[[00_JS_Ecosystem_HomePage]]"
  - "[[TS_ImportType]]"
  - "[[NestJS_Concept]]"
  - "[[Monorepo_PNPM]]"
  - "[[NestJS_DTO]]"
---
# TS_TsConfig — tsconfig.json 설정

>[!info]
>tsconfig.json = TypeScript 컴파일러(tsc)에게 "이 프로젝트를 어떻게 컴파일할지" 알려주는 설정 파일.
> `strict`으로 타입 안전성 강도, `paths`로 `@/` 경로 별칭, `target`으로 출력 JS 버전 지정.
>  경로 별칭(`@/`) → [[TS_ImportType]]

---

# tsconfig.json이란 ⭐️⭐️⭐️⭐️

```txt
TypeScript = JavaScript를 타입 안전하게 만드는 언어
실제로 실행되는 것은 JavaScript
→ TypeScript 코드를 JavaScript로 변환(컴파일)하는 과정이 필요

tsconfig.json:
  TypeScript 컴파일러(tsc)에게 전달하는 설정
  "어떤 파일을 컴파일할지"
  "어떤 버전의 JS로 변환할지"
  "얼마나 엄격하게 타입 검사할지"
  "경로 별칭(@/)을 어떻게 해석할지"
  등을 정의
```

---

# 핵심 옵션 ⭐️⭐️⭐️⭐️

## compilerOptions — 컴파일러 동작 설정

### target — 출력 JS 버전

```json
{
  "compilerOptions": {
    "target": "ES2022"
  }
}
```

```txt
target:
  TypeScript를 어떤 버전의 JavaScript로 변환할지
  "ES5"   → 구형 브라우저 지원 (화살표 함수, const 등이 구문으로 변환됨)
  "ES2020"→ 최신 브라우저/Node 지원 (대부분 그대로 유지)
  "ES2022"→ 더 최신 (class fields, top-level await 등)

  Node.js 18+ → ES2022 이상 사용 가능
  Next.js → Next.js가 내부적으로 번들링하므로 상대적으로 덜 중요
```

### strict — 타입 검사 엄격도

```json
{
  "compilerOptions": {
    "strict": true
  }
}
```

```txt
strict: true — 아래 옵션들을 한 번에 켬:
  strictNullChecks       null/undefined를 별도 타입으로 취급
  noImplicitAny          타입 추론 불가 시 any 금지
  strictFunctionTypes    함수 타입 엄격 검사
  strictPropertyInitialization  클래스 프로퍼티 초기화 강제
  strictBindCallApply    bind/call/apply 타입 검사

strict: false면:
  null, undefined가 모든 타입에 할당 가능
  → 런타임 에러 방지를 TypeScript가 못 해줌
  → 항상 true 권장
```

### moduleResolution — 모듈 찾는 방식

```json
{
  "compilerOptions": {
    "moduleResolution": "bundler"
  }
}
```

```txt
moduleResolution:
  import { x } from './utils' 처럼 경로를 쓸 때
  실제 파일을 어떻게 찾을지 결정

  "node"    → Node.js 방식 (구형, .js 확장자 없어도 찾음)
  "bundler" → 번들러(Vite, webpack) 방식 (최신 권장)
  "node16"  → Node.js ESM 방식 (.js 확장자 명시 필요)
```

### paths — 경로 별칭 (@/)

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
paths:
  "@/components/Button" → "./src/components/Button"로 해석
  → import 경로에서 ../../../ 대신 @/ 사용 가능

  baseUrl:
  paths의 기준 경로 (paths에서 . = 이 파일이 있는 위치)

→ 자세한 내용 [[TS_ImportType]] 경로 별칭 섹션
```

### 자주 쓰는 옵션 한눈에

|옵션|설명|일반값|
|---|---|---|
|`target`|출력 JS 버전|`ES2022`|
|`strict`|타입 검사 엄격도|`true`|
|`moduleResolution`|모듈 찾는 방식|`bundler`|
|`baseUrl`|경로 별칭 기준|`.`|
|`paths`|경로 별칭 매핑|`{"@/*": ["./src/*"]}`|
|`outDir`|컴파일 출력 폴더|`./dist`|
|`rootDir`|소스 루트 폴더|`./src`|
|`esModuleInterop`|CommonJS 모듈 import 편의|`true`|
|`skipLibCheck`|.d.ts 파일 타입 검사 생략|`true`|
|`declaration`|.d.ts 파일 생성|`true` (라이브러리)|
|`emitDecoratorMetadata`|데코레이터 메타데이터|`true` (NestJS 필수)|
|`experimentalDecorators`|데코레이터 사용|`true` (NestJS 필수)|

---

# include · exclude — 컴파일 대상 파일

```json
{
  "include": ["src/**/*"],        // 컴파일할 파일
  "exclude": ["node_modules", "dist"]  // 제외할 파일
}
```

```txt
include:
  컴파일할 파일 glob 패턴
  "src/**/*" = src 폴더 안의 모든 파일

exclude:
  포함하지 않을 파일/폴더
  node_modules는 기본으로 제외되지만 명시하는 것이 관례
  dist = 이미 컴파일된 결과물을 다시 컴파일하지 않도록
```

---

# extends — 설정 상속 ⭐️⭐️⭐️

```json
// tsconfig.json (공통 기반)
{
  "compilerOptions": {
    "strict": true,
    "esModuleInterop": true
  }
}

// apps/web/tsconfig.json
{
  "extends": "../../tsconfig.json",   // 루트 설정 상속
  "compilerOptions": {
    "target": "ES2022",
    "paths": { "@/*": ["./src/*"] }   // 추가·덮어쓰기
  }
}
```

```txt
모노레포에서 extends 활용:
  루트 tsconfig.json → 공통 설정 (strict, esModuleInterop 등)
  apps/api/tsconfig.json → extends 루트 + API 전용 설정
  apps/web/tsconfig.json → extends 루트 + Web 전용 설정

  공통 설정은 한 곳에서 관리 → 변경 시 전체 적용
```

---

# API(NestJS) vs Web(Next.js) 설정 비교 ⭐️⭐️⭐️⭐️

```json
// apps/api/tsconfig.json (NestJS — TypeScript 6 기준)
{
  "compilerOptions": {
    "module":                       "nodenext",
    "moduleResolution":             "nodenext",
    "resolvePackageJsonExports":    true,
    "esModuleInterop":              true,
    "isolatedModules":              true,
    "declaration":                  true,
    "removeComments":               true,
    "emitDecoratorMetadata":        true,
    "experimentalDecorators":       true,
    "allowSyntheticDefaultImports": true,
    "target":                       "ES2023",
    "sourceMap":                    true,
    "outDir":                       "./dist",
    "rootDir":                      "./src",
    "incremental":                  true,
    "skipLibCheck":                 true,
    "strictNullChecks":             true,
    "forceConsistentCasingInFileNames": true,
    "noImplicitAny":                false,
    "strictBindCallApply":          false,
    "noFallthroughCasesInSwitch":   false
  }
}
```

```txt
outDir + rootDir — 왜 둘 다 필요한가:
  outDir: "./dist"   → 컴파일 결과물을 dist 폴더에 저장
  rootDir: "./src"   → 소스 파일의 루트가 src 폴더임을 명시

  TypeScript 6부터 outDir만 있고 rootDir이 없으면 에러:
  "Option 'outDir' requires 'rootDir'"
  → rootDir을 추가해야 해결

  이유: outDir가 있으면 TS가 src 구조를 그대로 dist에 복사
  rootDir을 알아야 "dist 안에 src 폴더를 만들지 않고" 올바른 경로 생성 가능

module + moduleResolution — "nodenext":
  Node.js의 ESM(ECMAScript Module) 방식 사용
  .js 확장자를 import 경로에 명시해야 하는 엄격한 규칙
  NestJS는 과거 "commonjs"를 썼지만 최신 Node.js + ESM 지원으로 nodenext 사용

strictNullChecks: true vs strict: false:
  strict: true를 쓰면 아래 옵션들이 전부 켜짐
  이 설정은 strict 대신 필요한 것만 선택적으로 켬:
    strictNullChecks: true     → null/undefined 타입 분리 (필요)
    noImplicitAny: false       → any 추론 허용 (DTO 등에서 유연하게)
    strictBindCallApply: false → bind/call/apply 타입 검사 완화
  → NestJS의 데코레이터·DI 패턴과 충돌 없이 타입 안전성 확보

strictPropertyInitialization: false — NestJS DTO 때문에 필수:
  TypeScript 기본(true)이면 클래스 프로퍼티를 constructor에서 반드시
  초기화하거나 ! (definite assignment assertion)를 붙여야 함

  NestJS DTO는 class-validator 데코레이터만 붙이고 초기화하지 않음
  → true이면 모든 DTO 프로퍼티에 ! 를 붙여야 함

  false 없이 (! 필요):           false 설정 (깔끔):
  email!: string;                 email: string;
  name!: string;                  name: string;

  class-validator + ValidationPipe가 런타임에 값을 채워주기 때문에
  TypeScript의 정적 검사에서만 "초기화 안 됨"처럼 보이는 것
  → false로 설정해서 불필요한 ! 를 없앰
```

```txt
나머지 옵션 설명:

  resolvePackageJsonExports: true
    패키지의 exports 필드를 우선 따름 (최신 Node.js 방식)

  isolatedModules: true
    각 파일을 독립적으로 변환 가능하도록 강제 (esbuild/SWC 호환)
    const enum, namespace 등 일부 TypeScript 전용 문법 금지

  declaration: true
    .d.ts 타입 선언 파일 생성 → 다른 패키지에서 타입 참조 가능

  removeComments: true
    컴파일된 JS에서 주석 제거 → dist 파일 크기 감소

  sourceMap: true
    .js.map 파일 생성 → 디버깅 시 원본 TS 코드 위치 추적

  incremental: true
    이전 컴파일 정보를 캐싱 → 재컴파일 시 변경된 파일만 처리 (빌드 속도 향상)
    .tsbuildinfo 파일 생성

  forceConsistentCasingInFileNames: true
    import 경로의 대소문자가 실제 파일과 일치해야 함
    macOS(대소문자 무시)에서 개발 → Linux(대소문자 구분) 배포 시 에러 방지
```


```json
// apps/web/tsconfig.json (Next.js)
{
  "compilerOptions": {
    "target":           "ES2022",
    "lib":              ["dom", "dom.iterable", "esnext"],
    "module":           "esnext",     // 번들러가 처리
    "moduleResolution": "bundler",    // Vite/Webpack 방식
    "strict":           true,
    "esModuleInterop":  true,
    "skipLibCheck":     true,
    "baseUrl":          ".",
    "paths": {
      "@/*": ["./src/*"]              // 경로 별칭
    },
    "jsx":              "preserve",   // Next.js가 JSX 처리
    "plugins": [{ "name": "next" }]   // Next.js TypeScript 플러그인
  }
}
```

```txt
핵심 차이:

  module:
  NestJS → "commonjs"   (Node.js 런타임, require() 방식)
  Next.js → "esnext"    (번들러가 처리, import 방식)

  moduleResolution:
  NestJS → "node"       (Node.js 모듈 해석 규칙)
  Next.js → "bundler"   (번들러 모듈 해석 규칙)

  emitDecoratorMetadata + experimentalDecorators:
  NestJS에서 필수 — @Injectable(), @Controller() 같은 데코레이터가
  의존성 주입(DI)에서 타입 정보를 읽으려면 이 두 옵션이 켜져 있어야 함

  lib: ["dom", ...]:
  Next.js에 필요 — 브라우저 API(window, document 등)의 타입 정의 포함
  NestJS는 브라우저 API 불필요 → lib 생략

  jsx: "preserve":
  Next.js — JSX 변환을 Next.js(SWC)가 담당 → tsc는 건드리지 않음
```

---

# 흔히 만나는 설정 관련 에러

| 에러                                   | 원인                              | 해결                                    |
| ------------------------------------ | ------------------------------- | ------------------------------------- |
| `Cannot find module '@/...'`         | paths 설정 없음 또는 baseUrl 누락       | tsconfig.json에 baseUrl + paths 추가     |
| `Option 'outDir' requires 'rootDir'` | TS 6에서 outDir만 있고 rootDir 없음    | `"rootDir": "./src"` 추가               |
| `Decorators are not enabled`         | experimentalDecorators 없음       | `"experimentalDecorators": true` 추가   |
| `Cannot use namespace 'X' as type`   | skipLibCheck 꺼져 있고 .d.ts 충돌     | `"skipLibCheck": true` 추가             |
| `Property has no initializer`        | strictPropertyInitialization 충돌 | NestJS DTO는 `false`로 설정 또는 `!` 사용     |
| `baseUrl` deprecation 경고             | TS 6 → 7에서 baseUrl 제거 예정        | `"ignoreDeprecations": "6.0"` 추가 (임시) |