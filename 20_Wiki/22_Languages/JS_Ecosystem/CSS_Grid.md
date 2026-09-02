---
aliases: [align-content, align-self, auto-fill, auto-fit, CSS Grid, fr 단위, grid-area, grid-column, grid-row, grid-template-columns, grid-template-rows, minmax, repeat, span]
tags: [CSS]
relations:
  - "[[00_JS_Ecosystem_HomePage]]"
  - "[[CSS_Layout]]"
  - "[[CSS_Tricks]]"
---
# CSS_Grid — 그리드 레이아웃

>[!info]
> CSS Grid = 2차원(행 + 열) 레이아웃 시스템.
> `grid-template-columns`로 열 정의 → `grid-column`으로 아이템 배치.
> 계산 단위 `fr`, 함수 `repeat()` · `minmax()` 이해가 핵심.

---

# Grid 기본 개념 — 격자 구조 ⭐️⭐️⭐️⭐️⭐️

```txt
Grid = 행(row) × 열(column) 격자를 만들고 아이템을 원하는 칸에 배치

용어:
  Grid Container  → display: grid를 선언한 부모
  Grid Item       → 직계 자식
  Grid Line       → 격자를 나누는 선 (1번부터 시작, 오른쪽 끝 = -1)
  Grid Track      → 두 선 사이의 행 또는 열
  Grid Cell       → 행 × 열이 만나는 단일 칸
  Grid Area       → 여러 셀을 합친 영역
```

```txt
  grid-template-columns: repeat(4, 1fr)
  ┌──────────┬──────────┬──────────┬──────────┐
  │    1fr   │    1fr   │    1fr   │    1fr   │
  └──────────┴──────────┴──────────┴──────────┘
  ↑          ↑          ↑          ↑          ↑
 선1        선2        선3        선4       선5(=-1)

  grid-column: 1 / -1   ├──────────────────────────┤  전체 너비
  grid-column: 2 / 4             ├──────────────────┤  2~3열
  grid-column: 4                            ├──────────┤  4열만
  grid-row: 2 / span 2  → 2번 행에서 시작, 2행 차지
```

```txt
선 번호 규칙:
  왼쪽→오른쪽: 1, 2, 3, 4, 5 ...
  오른쪽→왼쪽: -1, -2, -3 ...
  4열 그리드 = 선이 5개 (1~5, 또는 -1~-5)

  grid-column: 1 / -1  → 1번 선 ~ 끝 선 (전체 너비)
  grid-column: 2 / 4   → 2번 선 ~ 4번 선 (2~3번 열 차지)
  grid-column: 4       → 4번 열 (= 4 / 5)
```

---

# grid-template-columns / rows — 격자 정의 ⭐️⭐️⭐️⭐️⭐️

## repeat()

```css
/* repeat(횟수, 크기) */
grid-template-columns: repeat(4, 1fr);
/* = 1fr 1fr 1fr 1fr — 같은 크기 4열 */

grid-template-columns: repeat(2, minmax(0, 1fr));
/* = minmax(0,1fr) minmax(0,1fr) — 2열, 각 열 최소 0 최대 동등 분배 */

grid-template-rows: auto;
/* 행 높이 = 내용에 맞게 자동 */
```

## fr 단위 — 남은 공간 분배

```css
/* 컨테이너 너비 900px, gap 없다고 가정 */
grid-template-columns: 1fr 2fr 1fr;
/*
  전체 fr 합계 = 1+2+1 = 4fr
  1fr = 900 / 4 = 225px
  2fr = 900 / 4 × 2 = 450px
  1fr = 225px
  → 225 : 450 : 225
*/

grid-template-columns: 200px 1fr;
/*
  200px 고정 후 남은 공간(700px)을 1fr이 차지
*/

grid-template-columns: repeat(3, 1fr);
/* gap: 1rem이 있으면:
   사용 가능 너비 = 900 - (gap × 2) = 900 - 32 = 868px
   1fr = 868 / 3 ≈ 289px
*/
```

## minmax(최소, 최대)

```css
grid-template-columns: repeat(2, minmax(0, 1fr));
/*
  최소: 0px (콘텐츠가 없어도 0까지 줄어들 수 있음)
  최대: 1fr (남은 공간을 동등 분배)

  minmax(0, 1fr) vs 1fr:
    1fr         → 내부 콘텐츠 크기 이상으로 줄어들지 않음 (min-content 하한)
    minmax(0,1fr)→ 0까지 줄어들 수 있음 → overflow 방지에 유리
    → Tailwind의 min-w-0 문제와 같은 맥락
*/

grid-template-columns: repeat(auto-fill, minmax(200px, 1fr));
/* 열 너비 최소 200px, 최대 1fr — 컨테이너에 맞게 열 수 자동 결정 */
```

## auto-fill vs auto-fit

```css
/* auto-fill — 빈 트랙도 유지 */
grid-template-columns: repeat(auto-fill, minmax(150px, 1fr));
/* 컨테이너 600px → 4열 (150×4=600), 아이템 2개면 나머지 2열은 빈 공간 유지 */

/* auto-fit — 빈 트랙을 접어 아이템이 공간 채움 */
grid-template-columns: repeat(auto-fit, minmax(150px, 1fr));
/* 아이템 2개면 빈 열 접힘 → 2개 아이템이 300px씩 차지 */

/*
  auto-fill → 카드 그리드에서 고정 너비 유지 원할 때
  auto-fit  → 아이템이 적을 때 꽉 채우고 싶을 때
*/
```

---

# grid-column / grid-row — 아이템 배치 ⭐️⭐️⭐️⭐️⭐️

## 기본 문법

```css
/* 단축: 시작선 / 끝선 */
grid-column: 1 / 3;     /* 1번~3번 선 → 1~2열 차지 (2열 span) */
grid-column: 2 / 4;     /* 2번~4번 선 → 2~3열 차지 */
grid-column: 1 / -1;    /* 1번~끝 선  → 전체 너비 */
grid-column: 4;         /* 4번 열 (= 4 / 5 축약) */

grid-row: 2 / span 2;   /* 2번 행에서 시작, 2행 span */
grid-row: 3;            /* 3번 행 */
```

## span — 몇 칸 차지

```css
/* span = "여기서부터 N칸" */
grid-column: 2 / span 3;  /* 2번 선에서 3칸 → 2,3,4열 */
grid-row: 1 / span 2;     /* 1번 행에서 2행 → 1,2행 */

/* 끝 선 기준 span */
grid-column: span 2 / 5;  /* 5번 선에서 역방향 2칸 → 3,4열 */
```

## 실전 예시 — room-scene-layout

```css
/*
  그리드 구조 가정:
  grid-template-columns: repeat(4, 1fr)  → 4열
  grid-template-rows: auto auto auto     → 3행
*/

/* 기본 배치 */
.room-me {
  grid-column: 2;    /* 2번 열 */
  grid-row: 3;       /* 3번 행 */
}

.room-others {
  grid-column: 4;        /* 4번 열 */
  grid-row: 2 / span 2;  /* 2번 행에서 2행 차지 (2~3행) */
}

/* 모바일 — 전체 너비 차지 */
@media (max-width: 30rem) {
  .room-scene-layout > .room-me {
    grid-column: 2;    /* 모바일에서도 2번 열 */
    grid-row: 3;
  }

  .room-panel {
    grid-column: 1 / -1;   /* 전체 너비 */
    grid-row: 2;
    grid-template-columns: repeat(2, minmax(0, 1fr));  /* 내부 2열 */
    grid-template-rows: auto;
  }
}
```

---

# 정렬 속성 — align · justify ⭐️⭐️⭐️⭐️

## 전체 구조

```txt
두 축:
  block axis  (세로, 행 방향) → align-*
  inline axis (가로, 열 방향) → justify-*

두 대상:
  아이템(셀 내부에서) → align-items / justify-items
  트랙(그리드 전체)   → align-content / justify-content
  개별 아이템        → align-self / justify-self (아이템에 직접 선언)
```

## 비교표

|속성|선언 위치|대상|효과|
|---|---|---|---|
|`align-items`|컨테이너|셀 내 아이템 (세로)|기본값 stretch|
|`justify-items`|컨테이너|셀 내 아이템 (가로)|기본값 stretch|
|`align-content`|컨테이너|행 트랙 전체 (세로)|여백 분배|
|`justify-content`|컨테이너|열 트랙 전체 (가로)|여백 분배|
|`align-self`|아이템|자기 자신 (세로)|개별 재정의|
|`justify-self`|아이템|자기 자신 (가로)|개별 재정의|

```css
.grid-container {
  display: grid;
  grid-template-columns: repeat(3, 200px);  /* 3열, 각 200px */
  /* 컨테이너가 900px이면 트랙이 꽉 참 → align-content 효과 없음 */
  /* 컨테이너가 1200px이면 300px 여백 → align-content로 분배 가능 */

  align-items: center;      /* 셀 내 아이템 세로 중앙 */
  justify-items: start;     /* 셀 내 아이템 가로 시작점 */

  align-content: stretch;   /* 행 트랙이 컨테이너 채움 (기본) */
  /* align-content: space-between → 행 사이 여백 균등 */
}

.grid-item {
  align-self: start;        /* 이 아이템만 세로 상단 정렬 */
  min-height: max-content;  /* 내용 크기만큼만 */
}
```

```txt
align-content vs align-items 헷갈리는 이유:
  align-items  → 각 셀 안에서 아이템이 어디에 붙느냐 (셀 내부 기준)
  align-content→ 행 트랙들이 컨테이너 안에서 어떻게 분배되느냐 (트랙 기준)
               → 트랙 총합 < 컨테이너 높이일 때만 효과 있음

align-self: start + min-height: max-content 조합:
  아이템이 행 높이에 stretch되지 않고 자기 콘텐츠 크기만큼만 차지
```

## 값 목록

```txt
start     → 시작점 정렬
end       → 끝점 정렬
center    → 중앙
stretch   → 늘려서 채움 (기본값)
space-between → 양 끝 붙이고 사이 균등 (content 계열만)
space-around  → 각 트랙 양쪽 균등 (content 계열만)
space-evenly  → 모든 간격 동일 (content 계열만)
```

---

# gap ⭐️⭐️⭐️

```css
gap: 1rem;              /* row-gap: 1rem; column-gap: 1rem; */
gap: 0.5rem 1rem;       /* row-gap: 0.5rem; column-gap: 1rem; */
row-gap: 0.5rem;
column-gap: 1rem;
```

```txt
gap은 트랙 사이 여백 (아이템 바깥 여백 아님)
→ fr 계산 시 gap이 먼저 제거되고 남은 공간을 fr로 분배
→ 첫/마지막 아이템 바깥 여백 없음 (padding으로 처리)
```

---

# grid-area — 이름 기반 배치 ⭐️⭐️⭐️

```css
.layout {
  display: grid;
  grid-template-areas:
    "header header header"
    "sidebar main   main"
    "footer  footer footer";
  grid-template-columns: 200px 1fr 1fr;
  grid-template-rows: auto 1fr auto;
}

header  { grid-area: header; }
.sidebar { grid-area: sidebar; }
main    { grid-area: main; }
footer  { grid-area: footer; }
```

```txt
. (점) = 빈 셀:
  grid-template-areas:
    "header header ."
    "sidebar main  main"

grid-area 단축 (선 번호):
  grid-area: row-start / col-start / row-end / col-end
  grid-area: 1 / 2 / 3 / -1;
  = grid-row: 1/3; grid-column: 2/-1;
```

---

# React에서 Grid ⭐️⭐️⭐️

```tsx
// Tailwind 그리드 패턴
function MovieGrid({ movies }: { movies: Movie[] }) {
  return (
    <div className="grid grid-cols-2 gap-4 md:grid-cols-3 lg:grid-cols-4">
      {movies.map((movie) => (
        <MovieCard key={movie.id} movie={movie} />
      ))}
    </div>
  );
}

// CSS Modules — 동적 그리드 배치
function RoomLayout() {
  return (
    <div className={styles.roomGrid}>
      <div className={styles.roomMe}>나</div>
      <div className={styles.roomOthers}>상대방</div>
    </div>
  );
}
```

```css
/* room.module.css */
.roomGrid {
  display: grid;
  grid-template-columns: repeat(4, 1fr);
  grid-template-rows: auto auto auto;
  gap: 1rem;
}
.roomMe     { grid-column: 2; grid-row: 3; }
.roomOthers { grid-column: 4; grid-row: 2 / span 2; }

@media (max-width: 30rem) {
  .roomGrid {
    grid-template-columns: repeat(2, minmax(0, 1fr));
  }
  .roomMe     { grid-column: 2; grid-row: 3; }
  .roomOthers { grid-column: 1 / -1; grid-row: 2; }
}
```

---

# Grid vs Flex 선택 기준

```txt
Flex  → 1차원 (행 또는 열 하나)
  네비게이션 바, 버튼 그룹, 인라인 아이템 나열
  → 주축 방향 정렬이 핵심

Grid → 2차원 (행 + 열 동시)
  캘린더, 대시보드, 갤러리, 복잡한 페이지 레이아웃
  → 행·열 위치를 명시적으로 제어

단일 요소 중앙 정렬:
  display: grid; place-items: center; ← grid가 더 간결
  display: flex; align-items: center; justify-content: center;
```
