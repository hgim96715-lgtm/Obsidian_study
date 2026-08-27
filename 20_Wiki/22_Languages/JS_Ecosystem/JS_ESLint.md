---
aliases: [eslint, eslint-disable, linting, no-floating-promises, no-img-element]
tags: [JavaScript, TypeScript, Tooling]
related:
  - "[[00_JS_Ecosystem_HomePage]]"
  - "[[HTML_Image]]"
  - "[[JS_Operators]]"
---
# JS_ESLint — ESLint 정적 분석 · disable 주석

> [!info]
> ESLint = JS/TS 코드를 실행하지 않고 문제를 찾는 정적 분석 도구.
> 규칙(rule)별로 경고/에러를 내고, 특정 줄·블록·파일 단위로 끌 수 있음.

---

# eslint-disable 주석 종류 ⭐️⭐️⭐️⭐️

```typescript
// 1. 다음 줄만 비활성화 (가장 자주 씀)
// eslint-disable-next-line @next/next/no-img-element
<img src={url} alt={title} />

// 2. 현재 줄 비활성화 (줄 끝에 붙임)
const x = eval(code); // eslint-disable-line no-eval

// 3. 블록 비활성화
/* eslint-disable @typescript-eslint/no-explicit-any */
function legacy(data: any) { ... }
/* eslint-enable @typescript-eslint/no-explicit-any */

// 4. 파일 전체 비활성화 (파일 맨 위)
/* eslint-disable */
// 이 파일 전체에서 ESLint 꺼짐 — 거의 쓰지 말 것
```

```txt
선택 기준:
  한 줄만 예외  → eslint-disable-next-line  (범위 가장 좁음)
  현재 줄 끝   → eslint-disable-line       (줄 길이 길어질 수 있음)
  여러 줄 블록  → disable/enable 쌍
  파일 전체    → /* eslint-disable */      (거의 쓰지 말 것, 남은 에러 숨겨짐)

범위를 최대한 좁게 쓰는 것이 원칙
```

---

# 자주 만나는 규칙 ⭐️⭐️⭐️⭐️

## @next/next/no-img-element

```tsx
// ❌ ESLint 경고 — Next.js에서 <img> 직접 사용
<img src={posterUrl} alt={title} />

// ✅ 권장 — Next.js Image 컴포넌트 (자동 최적화)
import Image from 'next/image';
<Image src={posterUrl} alt={title} width={185} height={278} />
```

```txt
이 규칙이 존재하는 이유:
  Next.js <Image>는 자동으로 적용:
  - WebP/AVIF 변환 (파일 크기 감소)
  - 크기 기반 최적화 (width/height 필수)
  - lazy loading 기본 적용
  - CLS(레이아웃 시프트) 방지

언제 <img> + disable을 써도 되는가:
  외부 CDN URL이라 도메인을 next.config에 추가하기 어려울 때
  동적 src인데 width/height를 알 수 없을 때 (TMDB 포스터 등)
  서드파티 라이브러리가 <img>를 요구할 때
```

```tsx
// eslint-disable-next-line @next/next/no-img-element
<img
  src={tmdbPosterUrl(movie.poster_path, 'w185') ?? undefined}
  alt={movie.title}
  loading="lazy"
  decoding="async"
/>
// → [[HTML_Image]] loading/decoding 참고
```

## @typescript-eslint/no-floating-promises

```typescript
// ❌ 처리하지 않은 Promise — ESLint 경고
markRoomRead(roomId);

// ✅ void로 의도적 무시 명시
void markRoomRead(roomId);

// ✅ await로 처리
await markRoomRead(roomId);
```

```txt
→ [[JS_Operators]] void 섹션 참고
```

## @typescript-eslint/no-explicit-any

```typescript
// ❌
function parse(data: any) { ... }

// ✅ unknown (사용 전 타입 좁히기 강제)
function parse(data: unknown) { ... }

// ✅ 제네릭
function parse<T>(data: T) { ... }
```

## react-hooks/exhaustive-deps

```typescript
// ❌ deps 배열에 빠진 의존성
useEffect(() => {
  fetchUser(userId);  // userId를 쓰는데 deps에 없음
}, []);

// ✅
useEffect(() => {
  fetchUser(userId);
}, [userId]);

// deps를 의도적으로 생략할 때 (마운트 시 1회만 실행)
// eslint-disable-next-line react-hooks/exhaustive-deps
}, []);
```

---

# ⚠️ disable 주석 남용 금지 ⭐️⭐️⭐️⭐️

```txt
disable은 "이 규칙이 이 상황에서는 맞지 않는다"고 판단할 때만 사용
에러가 귀찮아서 끄는 건 기술 부채

disable 해도 되는 경우:
  규칙의 의도는 알지만 이 특정 케이스는 예외임을 알 때
  외부 라이브러리/API 제약으로 어쩔 수 없을 때
  레거시 코드를 점진적으로 마이그레이션 중일 때

disable 하면 안 되는 경우:
  에러 원인을 모르면서 끄는 경우
  no-floating-promises를 끄고 Promise를 그냥 버리는 경우 (실제 버그)
  any를 쓰고 no-explicit-any를 끄는 경우 (타입 안전성 포기)
```

---

# eslint.config.js — 규칙 전체 설정 ⭐️⭐️⭐️

```javascript
// ESLint Flat Config (v9+)
export default [
  {
    rules: {
      '@typescript-eslint/no-floating-promises': 'error',
      '@typescript-eslint/no-explicit-any':      'warn',
      'react-hooks/exhaustive-deps':             'warn',
    },
  },
];
```

```txt
규칙 수준:
  'error'  → 빌드/CI 실패, 반드시 고쳐야 함
  'warn'   → 경고만, 빌드는 통과
  'off'    → 비활성화

파일별로 끄고 싶으면:
  overrides / files 패턴으로 특정 파일에만 규칙 적용/해제 가능
```
