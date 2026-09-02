---
aliases: [시맨틱 태그, article, aside, div soup, footer, header, main, nav, section, semantic HTML]
tags: [HTML, React]
relations:
  - "[[00_JS_Ecosystem_HomePage]]"
  - "[[HTML_ARIA]]"
  - "[[NextJS_Metadata]]"
---
# HTML_Semantics — 시맨틱 태그

>[!info]
> 시맨틱(Semantic) = "의미 있는". `<div>`는 의미가 없고, `<nav>`는 "내비게이션"이라는 의미를 내포.
> 브라우저·검색엔진·스크린 리더가 HTML 구조를 이해하는 방식이 달라진다.

---

# div soup — 왜 시맨틱이 필요한가 ⭐️⭐️⭐️⭐️

```html
<!-- ❌ div soup — 의미를 알 수 없음 -->
<div class="header">
  <div class="nav">
    <div class="nav-item">홈</div>
  </div>
</div>
<div class="main">
  <div class="article">
    <div class="title">제목</div>
  </div>
</div>
<div class="footer">...</div>
```

```html
<!-- ✅ 시맨틱 태그 — 구조가 의미를 직접 전달 -->
<header>
  <nav>
    <a href="/">홈</a>
  </nav>
</header>
<main>
  <article>
    <h1>제목</h1>
  </article>
</main>
<footer>...</footer>
```

```txt
시맨틱 태그의 이점:
  SEO        → 검색 엔진이 페이지 구조를 이해 → 중요한 콘텐츠를 파악
  접근성     → 스크린 리더가 nav/main/aside를 구분해서 읽어줌
  가독성     → 코드만 봐도 레이아웃 구조가 보임
  CSS 타겟   → class 없이도 태그로 스타일 지정 가능

  → [[HTML_ARIA]] role과 연결:
    <nav>           = role="navigation" 내포
    <main>          = role="main" 내포
    <header>        = role="banner" 내포 (최상위일 때)
    <footer>        = role="contentinfo" 내포 (최상위일 때)
    → 시맨틱 태그가 있으면 role 속성 불필요
```

---

# 레이아웃 태그 — 언제 뭘 쓰는가 ⭐️⭐️⭐️⭐️

```txt
판단 질문:
  "이 영역이 페이지에서 몇 번 나오는가?"
    한 번만 → header / footer / main / nav / aside
    여러 번 가능 → section / article

  "이 영역이 다른 맥락에 꺼내도 독립적으로 의미가 있는가?"
    있다  → article
    없다  → section (페이지 문맥 안에서만 의미 있음)

  "레이아웃 / 스타일만을 위한 래퍼인가?"
    그렇다 → div
```

## 레이아웃 태그 비교표

|태그|역할|페이지 내 개수|
|---|---|---|
|`<header>`|페이지·섹션의 머리말 (로고, nav, 제목)|보통 1개|
|`<nav>`|내비게이션 링크 묶음|1~2개 (상단/하단)|
|`<main>`|페이지의 주요 콘텐츠|**정확히 1개**|
|`<footer>`|페이지·섹션의 바닥글 (저작권, 링크)|보통 1개|
|`<aside>`|본문과 관련 있지만 부가적인 콘텐츠 (사이드바, 광고)|여러 개 가능|
|`<section>`|주제별로 묶인 콘텐츠 블록|여러 개|
|`<article>`|독립적으로 배포·구독 가능한 콘텐츠|여러 개|
|`<div>`|의미 없는 래퍼 (레이아웃·스타일 목적)|제한 없음|

---

# 자주 헷갈리는 쌍

## section vs article ⭐️⭐️⭐️⭐️

```txt
article — 독립적으로 의미가 완결되는 콘텐츠
  블로그 게시글, 뉴스 기사, 댓글 하나, 제품 카드
  "이걸 RSS 피드로 뽑아도 의미가 있는가?" → article

section — 주제별 그룹, 페이지 문맥이 있어야 의미 있음
  "소개", "특징", "후기" 같은 페이지 내 구획
  heading(h2~h6)을 반드시 가져야 함

  → 둘 다 아니면 div
```

```html
<!-- 블로그 피드 — 각 게시글은 article -->
<main>
  <section>
    <h2>최신 글</h2>
    <article>
      <h3>React 상태 관리 정리</h3>
      <p>...</p>
    </article>
    <article>
      <h3>TypeScript 유틸 타입</h3>
      <p>...</p>
    </article>
  </section>
</main>

<!-- 제품 상세 페이지 — 구획이 section -->
<main>
  <section>
    <h2>제품 소개</h2>
    <p>...</p>
  </section>
  <section>
    <h2>사용 후기</h2>
    <article>
      <h3>홍길동의 후기</h3>
      <p>좋아요</p>
    </article>
  </section>
</main>
```

## header vs div.header ⭐️⭐️⭐️

```html
<!-- 페이지 전체 header -->
<body>
  <header>  ← role="banner" — 페이지 최상위 머리말
    <nav>...</nav>
  </header>
  <main>
    <article>
      <header>  ← article 안의 header — role="banner" 아님 (중첩 시 다름)
        <h2>게시글 제목</h2>
        <time>2026-01-01</time>
      </header>
      <p>본문...</p>
    </article>
  </main>
</body>
```

```txt
<header>는 페이지 최상위에서만 role="banner"
article/section 안에 쓰면 그 구획의 머리말 역할 (banner 아님)
→ 중첩해서 사용 가능
```

## aside ⭐️⭐️⭐️

```html
<main>
  <article>
    <h2>React 개념 정리</h2>
    <p>본문...</p>
  </article>

  <aside>
    <h3>관련 글</h3>
    <ul>
      <li><a href="...">useState 심화</a></li>
    </ul>
  </aside>
</main>
```

```txt
aside 용도:
  사이드바 (관련 글, 태그 목록)
  본문 내 주석·추가 설명 (callout)
  광고 영역

aside = "없어도 본문 이해에 지장 없는 부가 정보"
```

---

# 텍스트 계층 — h1~h6 ⭐️⭐️⭐️⭐️

```txt
h1~h6 규칙:
  h1은 페이지 전체에 하나 (페이지 제목)
  계층을 건너뛰지 말 것 — h1 다음 h3 (x)
  시각적 크기 때문에 태그를 선택하지 말 것
    → 크기는 CSS로, 태그는 의미로

  스크린 리더는 h 태그로 페이지를 탐색함
  검색 엔진은 h1~h2를 주요 키워드로 해석
```

```html
<!-- ✅ 올바른 계층 -->
<h1>홈 - 영화 기록 서비스</h1>        ← 페이지 전체 제목 (1개)
  <h2>최근 본 영화</h2>
    <h3>액션</h3>
    <h3>드라마</h3>
  <h2>내 캘린더</h2>
    <h3>2026년 1월</h3>

<!-- ❌ 계층 건너뜀 -->
<h1>제목</h1>
<h3>소제목</h3>   ← h2를 건너뜀
```

---

# 기타 시맨틱 태그 ⭐️⭐️⭐️

## 인라인 텍스트

|태그|의미|
|---|---|
|`<strong>`|중요한 강조 (bold — 스크린 리더가 강조해서 읽음)|
|`<em>`|약한 강조 (italic — 어조 변화)|
|`<time datetime="2026-01-15">`|날짜·시각 (기계가 파싱 가능한 datetime 속성)|
|`<abbr title="HyperText">`|약어 (full form은 title에)|
|`<mark>`|하이라이트 (검색 결과 강조)|
|`<code>`|인라인 코드|
|`<pre>`|공백 보존 (코드 블록)|

```html
<!-- time 태그 — 날짜를 시맨틱하게 -->
<time dateTime="2026-01-15">2026년 1월 15일</time>
<!-- 스크린 리더·SEO가 날짜임을 인식 -->

<!-- 강조 차이 -->
<strong>필수 입력</strong>   <!-- 중요한 것 → 스크린 리더 강조 -->
<em>선택 사항</em>           <!-- 어조 변화 → italic -->
<b>굵게</b>                  <!-- 시각적 굵음만 (의미 없음) -->
<i>기울임</i>                <!-- 시각적 기울임만 (의미 없음) -->
```

## 목록 태그

```html
<!-- ul — 순서 없는 목록 -->
<ul>
  <li>사과</li>
  <li>배</li>
</ul>

<!-- ol — 순서 있는 목록 (단계, 랭킹) -->
<ol>
  <li>npm install</li>
  <li>npm run dev</li>
</ol>

<!-- dl/dt/dd — 정의 목록 (용어:설명) -->
<dl>
  <dt>useState</dt>
  <dd>React 상태 관리 훅</dd>
  <dt>useEffect</dt>
  <dd>사이드이펙트 처리 훅</dd>
</dl>
```

```txt
dl/dt/dd 사용 상황:
  메타 정보 나열 (제목: 값 형태)
  용어집, 설정 화면의 레이블:값 쌍
  영화 상세 (감독: 홍길동 / 장르: 드라마 / 개봉: 2026)
```

## figure / figcaption

```html
<!-- 이미지 + 설명 -->
<figure>
  <img src="chart.png" alt="월별 가입자 추이" />
  <figcaption>2026년 월별 가입자 수 변화</figcaption>
</figure>

<!-- 코드 블록 + 설명 -->
<figure>
  <pre><code>const x = 1;</code></pre>
  <figcaption>예제 코드</figcaption>
</figure>
```

---

# React에서 시맨틱 태그 쓰기 ⭐️⭐️⭐️⭐️

```tsx
// ✅ 레이아웃 컴포넌트에 시맨틱 태그 적용
export default function Layout({ children }: { children: React.ReactNode }) {
  return (
    <>
      <header className="site-header">
        <nav>
          <Link href="/">홈</Link>
          <Link href="/search">검색</Link>
        </nav>
      </header>

      <main className="site-main">
        {children}
      </main>

      <footer className="site-footer">
        <p>© 2026</p>
      </footer>
    </>
  );
}
```

```tsx
// 영화 카드 — 독립적 콘텐츠 → article
function MovieCard({ movie }: { movie: Movie }) {
  return (
    <article className="movie-card">
      <h3>{movie.title}</h3>
      <time dateTime={movie.releaseDate}>{formatDate(movie.releaseDate)}</time>
      <p>{movie.overview}</p>
    </article>
  );
}

// 필터 섹션 — 페이지 맥락 안의 구획 → section
function FilterSection() {
  return (
    <section aria-labelledby="filter-title">
      <h2 id="filter-title">필터</h2>
      {/* 필터 옵션들 */}
    </section>
  );
}
```

```txt
React에서 주의:
  컴포넌트 이름이 Semantic하다고 태그도 자동으로 되는 건 아님
  <MovieCard /> 안에 실제로 <article>을 써야 함

  불필요한 div 래퍼 → Fragment (<>...</>) 또는 시맨틱 태그로 대체
  styled-component / tailwind도 태그 자체는 시맨틱하게 유지
```

---

# 결정 체크리스트

```txt
이 요소가:

  페이지 맨 위 로고·nav → <header>
  주요 탐색 링크 묶음   → <nav>
  페이지 핵심 콘텐츠    → <main> (페이지 당 1개)
  독립 배포 가능 콘텐츠 → <article> (블로그 글, 댓글, 카드)
  주제별 구획 (heading 有) → <section>
  부가 정보·사이드바    → <aside>
  바닥글 정보           → <footer>

  → 위에 해당 없으면 → <div>
```

---

# select / option / optgroup ⭐️⭐️⭐️⭐️

## 기본 구조

```html
<!-- 기본 select -->
<label for="genre">장르</label>
<select id="genre" name="genre">
  <option value="">-- 선택 --</option>   <!-- 빈 기본값 패턴 -->
  <option value="action">액션</option>
  <option value="drama">드라마</option>
  <option value="comedy">코미디</option>
</select>
```

```txt
<select>   → 드롭다운 컨테이너 (name 속성 = form 제출 시 키)
<option>   → 선택지 하나 (value = 실제 전송값, 텍스트 = 화면 표시)
            value 생략 시 → 텍스트 자체가 value로 전송됨
<label>    → for 속성으로 select의 id와 연결 (접근성 필수)

빈 기본값 패턴:
  <option value="">-- 선택 --</option>
  → value=""로 "아직 선택 안 함" 상태를 명시
  → form validation 에서 required + value="" → 미선택 감지 가능
```

## optgroup — 선택지 그룹핑

```html
<select id="movie-type" name="movie-type">
  <option value="">-- 유형 선택 --</option>

  <optgroup label="장르">
    <option value="action">액션</option>
    <option value="drama">드라마</option>
    <option value="comedy">코미디</option>
  </optgroup>

  <optgroup label="국가">
    <option value="ko">한국</option>
    <option value="us">미국</option>
    <option value="jp">일본</option>
  </optgroup>
</select>
```

```txt
<optgroup label="...">
  → label은 화면에 표시되는 그룹 제목 (선택 불가)
  → 선택지가 많을 때 시각적·구조적 그룹핑
  → 스크린 리더도 그룹 이름을 읽어줌
```

## multiple — 다중 선택

```html
<select id="tags" name="tags" multiple size="4">
  <option value="react">React</option>
  <option value="ts">TypeScript</option>
  <option value="next">Next.js</option>
  <option value="node">Node.js</option>
</select>
```

```txt
multiple   → Ctrl/Cmd + 클릭으로 복수 선택
size="4"   → 한 번에 보이는 항목 수 (리스트박스 형태로 렌더링)
form 전송 → 같은 name으로 여러 값이 전송됨
```

## selected / disabled / hidden

```html
<select name="rating">
  <option value="" disabled hidden selected>-- 별점 선택 --</option>
  <option value="5">★★★★★</option>
  <option value="4">★★★★</option>
  <option value="3">★★★</option>
</select>
```

```txt
selected  → 초기 선택값 지정
disabled  → 선택 불가 (회색 표시, 클릭 안 됨)
hidden    → 드롭다운 열기 전에는 안 보임

세 가지를 조합한 플레이스홀더 패턴:
  disabled hidden selected → "선택하세요" 텍스트가 기본으로 보이되 선택 자체는 불가
```

---

## React — controlled select ⭐️⭐️⭐️⭐️

```tsx
// ✅ controlled select — value + onChange 필수
function GenreSelect() {
  const [genre, setGenre] = useState('');

  return (
    <div>
      <label htmlFor="genre">장르</label>
      <select
        id="genre"
        value={genre}
        onChange={(e) => setGenre(e.target.value)}
      >
        <option value="">-- 선택 --</option>
        <option value="action">액션</option>
        <option value="drama">드라마</option>
      </select>
    </div>
  );
}
```

```tsx
// ✅ optgroup + 동적 렌더링 패턴
type GroupedOptions = {
  label: string;
  options: { value: string; label: string }[];
};

const GENRE_GROUPS: GroupedOptions[] = [
  {
    label: '분위기',
    options: [
      { value: 'action', label: '액션' },
      { value: 'drama', label: '드라마' },
    ],
  },
  {
    label: '국가',
    options: [
      { value: 'ko', label: '한국' },
      { value: 'us', label: '미국' },
    ],
  },
];

function GroupedSelect() {
  const [value, setValue] = useState('');

  return (
    <select value={value} onChange={(e) => setValue(e.target.value)}>
      <option value="">-- 선택 --</option>
      {GENRE_GROUPS.map((group) => (
        <optgroup key={group.label} label={group.label}>
          {group.options.map((opt) => (
            <option key={opt.value} value={opt.value}>
              {opt.label}
            </option>
          ))}
        </optgroup>
      ))}
    </select>
  );
}
```

```txt
React controlled select 핵심:
  value prop   → 현재 선택된 값 (state와 연결)
  onChange     → e.target.value로 선택값 업데이트
  defaultValue → uncontrolled 방식 (권장 안 함)

  multiple select → value를 string[] state로 관리
    value={selectedArr}
    onChange={(e) => {
      const opts = Array.from(e.target.selectedOptions);
      setSelected(opts.map(o => o.value));
    }}
```

---

## 스타일링 한계와 커스텀 드롭다운

```txt
네이티브 <select> CSS 한계:
  드롭다운 패널(option 목록) → 브라우저/OS가 그림 → CSS 거의 안 먹힘
  화살표 아이콘 → appearance: none + 직접 그리기 가능하지만 한계 있음
  option 내부 → 글꼴 색상 외에는 스타일 불가

  → 디자인 자유도가 필요하면 커스텀 드롭다운:
    Headless UI <Listbox> (Tailwind 생태계)
    Radix UI <Select>
    shadcn/ui <Select>
    → 내부적으로 div + aria-expanded + role="listbox" + role="option"으로 구현
    → [[HTML_ARIA]] role 패턴과 연결
```

```html
<!-- 네이티브 select에서 화살표만 커스터마이징 -->
<style>
  .custom-select {
    appearance: none;            /* 네이티브 화살표 제거 */
    background-image: url("chevron.svg");
    background-repeat: no-repeat;
    background-position: right 0.75rem center;
    padding-right: 2.5rem;
  }
</style>
<select class="custom-select">
  <option value="action">액션</option>
</select>
```
