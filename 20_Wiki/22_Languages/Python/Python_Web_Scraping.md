---
aliases:
  - 크롤링
  - 웹스크래핑
  - BeautifulSoup
  - bs4
tags:
  - Python
related:
  - "[[00_Python_HomePage]]"
  - "[[Python_Requests_Methods]]"
  - "[[Python_Requests_Response]]"
  - "[[Python_ElementTree]]"
  - "[[Python_Selenium]]"
---

# Python_Web_Scraping — 웹 크롤링

## 한 줄 요약

```
requests  → 웹 페이지 HTML 가져오기 (데이터 요청)
BeautifulSoup → 가져온 HTML 에서 원하는 데이터 추출 (파싱)
```

---

---

# ① 크롤링이란

```
웹 크롤링 = 웹 페이지의 HTML 을 가져와서
            원하는 데이터를 추출하는 것

흐름:
  브라우저가 하는 일을 코드로 자동화

  브라우저: URL 입력 → HTML 받음 → 화면에 렌더링
  크롤링:   requests로 HTML 받음 → BeautifulSoup 으로 파싱 → 데이터 추출
```

---

---

# ② 설치

```bash
pip install requests beautifulsoup4 lxml
```

```
requests      HTTP 요청 (HTML 가져오기)
beautifulsoup4  HTML 파싱 라이브러리 (bs4 로 import)
lxml          빠른 파서 (parser)

requirements.txt:
  beautifulsoup4>=4.12.0
  lxml>=5.0.0
```

---

---

# ③ 기본 흐름

```python
import requests
from bs4 import BeautifulSoup

# 1. HTML 가져오기
url = "https://example.com"
response = requests.get(url)

# 2. 상태 코드 확인
print(response.status_code)   # 200 = 정상

# 3. HTML → BeautifulSoup 객체로 파싱
soup = BeautifulSoup(response.text, "lxml")

# 4. 원하는 데이터 추출
title = soup.find("h1")
print(title.text)
```

```
"lxml" = 파서(Parser) 종류
  lxml    빠름 / 외부 라이브러리 필요
  html.parser  느림 / 파이썬 내장 (설치 불필요)
  xml     XML 전용

  → 보통 lxml 권장
```

---

---

# ④ BeautifulSoup 핵심 메서드 ⭐️

## find() — 첫 번째 1개

```python
soup.find("태그명")              # 태그로 찾기
soup.find("태그명", class_="클래스")  # 클래스로 찾기
soup.find("태그명", id="아이디")      # id로 찾기
soup.find("태그명", {"속성": "값"})   # 속성으로 찾기
```

```python
# 예시 HTML:
# <h1 class="title">안녕하세요</h1>
# <p id="desc">설명입니다</p>

soup.find("h1")                      # <h1 class="title">안녕하세요</h1>
soup.find("h1", class_="title")      # 동일
soup.find("p", id="desc")            # <p id="desc">설명입니다</p>
```

## find_all() — 전부 리스트로

```python
soup.find_all("태그명")              # 해당 태그 전부
soup.find_all("태그명", limit=3)     # 최대 3개만
```

```python
# 모든 링크 추출
links = soup.find_all("a")
for link in links:
    print(link.text)         # 링크 텍스트
    print(link["href"])      # 링크 주소
```

## select() — CSS 선택자 ⭐️

```python
soup.select("태그")               # 태그 선택
soup.select(".클래스명")           # 클래스 선택 (점)
soup.select("#아이디")             # id 선택 (샵)
soup.select("부모 > 자식")         # 직계 자식
soup.select("부모 자손")           # 모든 자손
soup.select("태그.클래스")         # 태그 + 클래스 동시
```

```python
# find_all vs select 비교
soup.find_all("div", class_="item")    # find_all
soup.select("div.item")                # select (더 간결)

# 복잡한 선택
soup.select("ul.menu > li > a")        # ul.menu 의 직계 li 의 직계 a
soup.select_one("h1")                  # select 버전의 find() (첫 번째 1개)
```

## 값 추출

```python
tag = soup.find("h1")

tag.text          # 텍스트만 (태그 제거)
tag.get_text()    # text 와 동일 (공백 옵션 가능)
tag.get_text(strip=True)   # 앞뒤 공백 제거

tag["href"]       # 속성값 가져오기
tag.get("href")   # 없어도 에러 안 남 (None 반환)
tag.get("href", "기본값")  # 없으면 기본값

tag.name          # 태그 이름 ("h1")
tag.attrs         # 모든 속성 딕셔너리
```

---

---

# ⑤ 실전 패턴

## 뉴스 제목 수집

```python
import requests
from bs4 import BeautifulSoup

url = "https://news.ycombinator.com"
response = requests.get(url, timeout=10)
soup = BeautifulSoup(response.text, "lxml")

# 모든 뉴스 제목 추출
titles = soup.select(".titleline > a")
for title in titles:
    print(title.text, "→", title["href"])
```

## 테이블 데이터 추출

```python
table = soup.find("table")
rows = table.find_all("tr")

data = []
for row in rows:
    cols = row.find_all("td")
    cols = [col.text.strip() for col in cols]
    if cols:
        data.append(cols)

# 판다스로 변환
import pandas as pd
df = pd.DataFrame(data)
```

## 여러 페이지 수집 (페이지네이션)

```python
results = []

for page in range(1, 6):   # 1~5 페이지
    url = f"https://example.com/list?page={page}"
    response = requests.get(url, timeout=10)
    soup = BeautifulSoup(response.text, "lxml")

    items = soup.select(".item-title")
    for item in items:
        results.append(item.text.strip())

    time.sleep(1)   # ⚠️ 서버 부하 방지 — 반드시 딜레이!
```

## 헤더 설정 (봇 차단 우회)

```python
headers = {
    "User-Agent": "Mozilla/5.0 (Windows NT 10.0; Win64; x64) AppleWebKit/537.36"
}

response = requests.get(url, headers=headers, timeout=10)
```

```
일부 사이트는 봇 접근 차단
User-Agent 를 브라우저처럼 설정하면 우회 가능
```

---

---

# ⑥ HTML 구조 이해

```
크롤링 전에 HTML 구조 파악 필수
브라우저에서 F12 → Elements 탭

<div class="product-list">         ← soup.select(".product-list")
  <div class="item">               ← soup.select(".item")
    <h2 class="name">상품명</h2>   ← soup.select(".item .name")
    <span class="price">10000</span>
    <a href="/detail/1">상세보기</a>
  </div>
</div>
```

```
찾는 순서:
  1. F12 → 원하는 요소 오른쪽 클릭 → 검사
  2. 태그명 / class / id 확인
  3. find() 또는 select() 로 추출
```

---

---

# ⑦ requests vs BeautifulSoup 역할 분리

```
requests:
  인터넷에서 HTML 가져오기
  GET / POST 요청
  쿠키 / 헤더 / 세션 관리

BeautifulSoup:
  가져온 HTML 파싱
  원하는 데이터 추출
  로컬 HTML 파일도 파싱 가능

→ 둘은 항상 같이 씀
→ API 가 있으면 requests 만으로 충분 (JSON 응답)
→ HTML 페이지 파싱이 필요할 때 BeautifulSoup 추가
```

---

---

# 자주 하는 실수

|실수|원인|해결|
|---|---|---|
|`tag.text` 에서 None 에러|find() 가 None 반환|`if tag:` 체크 후 사용|
|`tag["href"]` KeyError|속성 없는 태그|`tag.get("href")` 사용|
|한글 깨짐|인코딩 문제|`response.encoding = "utf-8"`|
|403 Forbidden|봇 차단|`headers` 에 User-Agent 추가|
|데이터가 비어 있음|JavaScript 로 렌더링|Selenium / Playwright 필요|
|서버 과부하|딜레이 없이 반복 요청|`time.sleep(1)` 추가|