---
aliases:
  - HTML 테이블
  - table
  - thead
  - tbody
  - th
  - scope
  - caption
tags:
  - HTML
  - React
relations:
  - "[[00_JS_Ecosystem_HomePage]]"
  - "[[HTML_Semantics]]"
  - "[[HTML_ARIA]]"
---
# HTML_Table — 데이터 테이블

>[!info]
> `<table>`은 데이터의 행·열 관계를 표현할 때 쓴다.
> 레이아웃 목적(CSS Grid/Flexbox 대신)으로 table을 쓰는 건 안티 패턴.
> 스크린 리더는 `<th scope>`를 보고 "3행 2열 = 홍길동의 나이" 같은 관계를 읽어준다.

---

# 테이블 기본 구조 ⭐️⭐️⭐️⭐️

```html
<table>
  <caption>2026년 상반기 영화 관람 기록</caption>  <!-- 테이블 제목 -->

  <thead>                    <!-- 헤더 행 묶음 -->
    <tr>
      <th scope="col">제목</th>
      <th scope="col">감독</th>
      <th scope="col">관람일</th>
      <th scope="col">평점</th>
    </tr>
  </thead>

  <tbody>                    <!-- 데이터 행 묶음 -->
    <tr>
      <td>듄: 파트 2</td>
      <td>드니 빌뇌브</td>
      <td>2026-01-15</td>
      <td>9.0</td>
    </tr>
    <tr>
      <td>오펜하이머</td>
      <td>크리스토퍼 놀란</td>
      <td>2026-02-03</td>
      <td>8.5</td>
    </tr>
  </tbody>

  <tfoot>                    <!-- 합계·요약 행 (선택) -->
    <tr>
      <td colspan="3">총 관람 수</td>
      <td>2편</td>
    </tr>
  </tfoot>
</table>
```

## 태그 역할 정리

|태그|역할|
|---|---|
|`<table>`|테이블 루트|
|`<caption>`|테이블 제목 (스크린 리더가 먼저 읽음)|
|`<thead>`|헤더 행 묶음 — 스크롤 시 고정 가능|
|`<tbody>`|데이터 행 묶음 (여러 개 가능)|
|`<tfoot>`|합계·요약 행 묶음|
|`<tr>`|행 (table row)|
|`<th>`|헤더 셀 — bold + 가운데 정렬 (기본)|
|`<td>`|데이터 셀|

---

# scope — 헤더와 데이터의 관계 ⭐️⭐️⭐️⭐️

```html
<thead>
  <tr>
    <th scope="col">이름</th>   <!-- 이 열 전체의 헤더 -->
    <th scope="col">나이</th>
    <th scope="col">역할</th>
  </tr>
</thead>
<tbody>
  <tr>
    <th scope="row">홍길동</th>  <!-- 이 행 전체의 헤더 -->
    <td>30</td>
    <td>admin</td>
  </tr>
</tbody>
```

```txt
scope 값:
  "col" → 이 th는 같은 열(column)의 td들의 헤더
  "row" → 이 th는 같은 행(row)의 td들의 헤더

왜 scope가 필요한가:
  스크린 리더는 셀을 읽을 때 어느 헤더에 속하는지 함께 읽음
  scope 없으면 "30" / scope 있으면 "홍길동, 나이, 30"
  복잡한 테이블일수록 scope가 없으면 데이터 의미를 파악 불가
```

---

# colspan · rowspan — 셀 병합 ⭐️⭐️⭐️

```html
<table>
  <thead>
    <tr>
      <th scope="col" rowspan="2">이름</th>   <!-- 2행 차지 -->
      <th scope="col" colspan="2">점수</th>   <!-- 2열 차지 -->
    </tr>
    <tr>
      <!-- 이름 th가 rowspan으로 이미 차지 → 여기 없음 -->
      <th scope="col">1분기</th>
      <th scope="col">2분기</th>
    </tr>
  </thead>
  <tbody>
    <tr>
      <th scope="row">홍길동</th>
      <td>85</td>
      <td>90</td>
    </tr>
  </tbody>
</table>
```

```txt
colspan="n" → 가로로 n칸 차지
rowspan="n" → 세로로 n칸 차지

colspan/rowspan을 쓰면 병합된 만큼 다음 행/열의 셀 개수가 줄어듦
→ 셀 수가 안 맞으면 레이아웃이 깨짐
```

---

# React에서 테이블 ⭐️⭐️⭐️⭐️

## 기본 패턴

```tsx
type Movie = {
  id: number;
  title: string;
  director: string;
  watchedAt: string;
  rating: number;
};

function MovieTable({ movies }: { movies: Movie[] }) {
  return (
    <table>
      <caption className="sr-only">관람 기록</caption>
      <thead>
        <tr>
          <th scope="col">제목</th>
          <th scope="col">감독</th>
          <th scope="col">관람일</th>
          <th scope="col">평점</th>
        </tr>
      </thead>
      <tbody>
        {movies.map((movie) => (
          <tr key={movie.id}>
            <td>{movie.title}</td>
            <td>{movie.director}</td>
            <td>
              <time dateTime={movie.watchedAt}>
                {formatDate(movie.watchedAt)}
              </time>
            </td>
            <td>{movie.rating}</td>
          </tr>
        ))}
      </tbody>
    </table>
  );
}
```

## 정렬 가능한 테이블 헤더

```tsx
type SortKey = 'title' | 'watchedAt' | 'rating';
type SortDir = 'asc' | 'desc';

function SortableTable({ movies }: { movies: Movie[] }) {
  const [sortKey, setSortKey] = useState<SortKey>('watchedAt');
  const [sortDir, setSortDir] = useState<SortDir>('desc');

  const sorted = useMemo(() =>
    [...movies].sort((a, b) => {
      const aVal = a[sortKey];
      const bVal = b[sortKey];
      const cmp = typeof aVal === 'string'
        ? aVal.localeCompare(bVal as string)
        : (aVal as number) - (bVal as number);
      return sortDir === 'asc' ? cmp : -cmp;
    }),
    [movies, sortKey, sortDir]
  );

  const handleSort = (key: SortKey) => {
    if (key === sortKey) {
      setSortDir((d) => (d === 'asc' ? 'desc' : 'asc'));
    } else {
      setSortKey(key);
      setSortDir('asc');
    }
  };

  const SortTh = ({ colKey, label }: { colKey: SortKey; label: string }) => (
    <th
      scope="col"
      aria-sort={sortKey === colKey ? (sortDir === 'asc' ? 'ascending' : 'descending') : 'none'}
    >
      <button onClick={() => handleSort(colKey)}>
        {label}
        {sortKey === colKey && (sortDir === 'asc' ? ' ▲' : ' ▼')}
      </button>
    </th>
  );

  return (
    <table>
      <thead>
        <tr>
          <SortTh colKey="title"     label="제목" />
          <SortTh colKey="watchedAt" label="관람일" />
          <SortTh colKey="rating"    label="평점" />
        </tr>
      </thead>
      <tbody>
        {sorted.map((movie) => (
          <tr key={movie.id}>
            <td>{movie.title}</td>
            <td>{movie.watchedAt}</td>
            <td>{movie.rating}</td>
          </tr>
        ))}
      </tbody>
    </table>
  );
}
```

```txt
aria-sort:
  "ascending"  → 오름차순으로 정렬됨
  "descending" → 내림차순으로 정렬됨
  "none"       → 정렬 가능하지만 지금은 정렬 안 됨
  "other"      → 다른 방식으로 정렬됨

  스크린 리더: "제목, 열 헤더, 오름차순 정렬됨"
```

---

# 스타일링 패턴 ⭐️⭐️⭐️

```css
/* 기본 테이블 스타일 리셋 */
table {
  border-collapse: collapse;  /* 셀 사이 이중 테두리 제거 */
  width: 100%;
}

th, td {
  padding: 0.75rem 1rem;
  text-align: left;
  border-bottom: 1px solid #e5e7eb;
}

thead th {
  background-color: #f9fafb;
  font-weight: 600;
}

/* 홀짝 행 구분 */
tbody tr:nth-child(even) {
  background-color: #f9fafb;
}

/* 호버 */
tbody tr:hover {
  background-color: #f3f4f6;
}
```

```tsx
/* Tailwind 버전 */
<table className="w-full border-collapse text-sm">
  <thead>
    <tr className="border-b bg-gray-50">
      <th className="px-4 py-3 text-left font-semibold" scope="col">제목</th>
      <th className="px-4 py-3 text-left font-semibold" scope="col">평점</th>
    </tr>
  </thead>
  <tbody>
    {movies.map((m) => (
      <tr key={m.id} className="border-b hover:bg-gray-50">
        <td className="px-4 py-3">{m.title}</td>
        <td className="px-4 py-3">{m.rating}</td>
      </tr>
    ))}
  </tbody>
</table>
```

---

# 반응형 테이블 ⭐️⭐️⭐️

```txt
문제: 테이블은 좁은 화면에서 가로 스크롤 or 레이아웃 깨짐

해결 방법:
  1. 래퍼에 overflow-x: auto → 가로 스크롤
  2. 모바일에서 카드 레이아웃으로 전환
  3. 중요도 낮은 열 숨기기 (hidden sm:table-cell)
```

```tsx
{/* 방법 1 — 가로 스크롤 래퍼 */}
<div className="overflow-x-auto">
  <table className="w-full min-w-[600px]">
    ...
  </table>
</div>

{/* 방법 3 — 모바일에서 열 숨기기 */}
<th scope="col" className="hidden sm:table-cell">감독</th>
<td className="hidden sm:table-cell">{movie.director}</td>
```

---

# 언제 table vs div ⭐️⭐️⭐️⭐️

```txt
<table> 사용:
  행·열 관계가 있는 데이터 (스프레드시트, 통계, 비교표)
  "3행 2열이 무슨 의미인가"가 중요한 데이터
  정렬·필터 기능이 붙는 데이터 그리드

div/grid 사용:
  레이아웃 목적 (사이드바, 카드 그리드, 폼 정렬)
  시각적으로 표처럼 보이지만 행·열 관계가 없는 것
  카드 목록 (각 카드가 독립적)

판단 기준:
  "이 데이터를 엑셀에 붙여넣으면 의미가 있는가?" → table
  "그냥 나란히 배치하고 싶은 것" → div + CSS Grid/Flex
```
