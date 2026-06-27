---
aliases:
  - selenium
  - 동적 크롤링
  - webdriver
tags:
  - Python
related:
  - "[[00_Python_HomePage]]"
  - "[[Python_Web_Scraping]]"
  - "[[Python_Requests_Methods]]"
---
# Python_Selenium — 동적 크롤링

## 한 줄 요약

```
BeautifulSoup  → 정적 HTML (서버가 이미 완성해서 준 HTML)
Selenium       → 동적 HTML (JavaScript 로 나중에 그려지는 페이지)
```

---

---

# ① BeautifulSoup vs Selenium 선택 기준

```
requests + BeautifulSoup:
  서버가 HTML 을 완성해서 줌
  빠름 / 가벼움
  JS 렌더링 데이터 못 가져옴

Selenium:
  실제 Chrome 브라우저를 코드로 제어
  JS 실행 → 동적 데이터 수집 가능
  로그인 / 클릭 / 스크롤 / 팝업 처리 가능
  느림 / 무거움

선택 기준:
  requests 로 HTML 가져왔는데 데이터가 비어있음 → JS 렌더링 → Selenium
  로그인 / 버튼 클릭 필요 → Selenium
  단순 HTML 파싱 → BeautifulSoup
```

---

---

# ② 설치 & import

```bash
pip install selenium webdriver-manager
```

```python
from selenium import webdriver
from selenium.webdriver.common.by import By                          # 요소 찾는 방법 지정
from selenium.webdriver.support.ui import WebDriverWait              # 명시적 대기
from selenium.webdriver.support import expected_conditions as EC      # 대기 조건
from selenium.webdriver.chrome.options import Options                 # 크롬 설정
from selenium.webdriver.chrome.service import Service                 # 크롬 서비스
from selenium.common.exceptions import TimeoutException, NoSuchElementException
from webdriver_manager.chrome import ChromeDriverManager              # 드라이버 자동 관리
```

---

---

# ③ 기본 흐름 — 순서대로 ⭐️

```
1. Options  설정 (headless 등)
2. driver   생성
3. get()    페이지 이동
4. wait     로딩 대기
5. find     요소 찾기
6. 작업     클릭 / 입력 / 텍스트 추출
7. quit()   반드시 종료
```

```python
from selenium import webdriver
from selenium.webdriver.common.by import By
from selenium.webdriver.support.ui import WebDriverWait
from selenium.webdriver.support import expected_conditions as EC
from selenium.webdriver.chrome.options import Options
from selenium.webdriver.chrome.service import Service
from webdriver_manager.chrome import ChromeDriverManager

# 1. 옵션 설정
options = Options()
options.add_argument("--headless")
options.add_argument("--no-sandbox")
options.add_argument("--disable-dev-shm-usage")
options.add_argument("--window-size=1920,1080")

# 2. 드라이버 생성
driver = webdriver.Chrome(
    service=Service(ChromeDriverManager().install()),
    options=options
)

# 3. 페이지 이동
driver.get("https://example.com")

# 4. 대기 객체 생성 (한 번만 만들어서 계속 재사용)
wait = WebDriverWait(driver, timeout=10)

try:
    # 5. 요소 대기 + 찾기
    element = wait.until(
        EC.presence_of_element_located((By.CLASS_NAME, "result"))
    )
    # 6. 텍스트 추출
    print(element.text)

finally:
    # 7. 반드시 종료
    driver.quit()
```

---

---

# ④ find_element vs find_elements ⭐️

```
find_element   → 첫 번째 1개 반환
                  없으면 NoSuchElementException 발생

find_elements  → 전부 리스트로 반환
                  없으면 빈 리스트 [] (에러 없음)
```

```python
# find_element — 1개
driver.find_element(By.ID, "username")       # 없으면 에러!

# find_elements — 전부 리스트
items = driver.find_elements(By.CLASS_NAME, "item")   # 없으면 []

# 개수 확인
print(len(items))

# 리스트 순회
for item in items:
    print(item.text)

# 인덱스로 특정 요소
first  = items[0]    # 첫 번째
last   = items[-1]   # 마지막
second = items[1]    # 두 번째
```

## By 종류

```python
driver.find_element(By.ID, "login-btn")
driver.find_element(By.CLASS_NAME, "item-title")     # 클래스 하나만 (공백 불가)
driver.find_element(By.CSS_SELECTOR, ".list > li")   # CSS 선택자 ← 추천
driver.find_element(By.XPATH, "//div[@id='main']") # XPath (복잡한 경우)
driver.find_element(By.TAG_NAME, "h1")
driver.find_element(By.NAME, "email")
driver.find_element(By.LINK_TEXT, "다음 페이지")
```

```
By.CLASS_NAME 주의:
  클래스가 "item active" 처럼 여러 개면 에러
  → By.CSS_SELECTOR(".item.active") 로 대체

추천 순서:
  1. By.ID           빠르고 명확
  2. By.CSS_SELECTOR 유연하고 강력
  3. By.XPATH        마지막 수단
```

## 요소에서 할 수 있는 것

```python
el = driver.find_element(By.ID, "title")

el.text                    # 텍스트 내용
el.get_attribute("href")   # 속성값
el.get_attribute("class")
el.click()                 # 클릭
el.send_keys("입력값")     # 텍스트 입력
el.clear()                 # 입력창 초기화
el.is_displayed()          # 화면에 보이는지
el.is_enabled()            # 활성화 상태인지
```

---

---

# ⑤ 대기 — WebDriverWait ⭐️

```
JS 렌더링은 시간이 걸림
바로 find_element 하면 아직 안 그려진 상태 → 에러
→ 요소가 준비될 때까지 반드시 대기
```

```python
wait = WebDriverWait(driver, timeout=10)   # 최대 10초

# DOM 에 존재할 때까지
element = wait.until(
    EC.presence_of_element_located((By.CLASS_NAME, "result"))
)

# 화면에 보일 때까지
element = wait.until(
    EC.visibility_of_element_located((By.ID, "content"))
)

# 클릭 가능할 때까지
button = wait.until(
    EC.element_to_be_clickable((By.CSS_SELECTOR, ".btn-submit"))
)
button.click()

# URL 변경 대기
wait.until(EC.url_contains("/dashboard"))
```

```
presence  → DOM 에 존재 (눈에 안 보여도 됨)
visibility → 화면에 보이는 상태
clickable  → 보이고 + 클릭 가능

대부분: presence_of_element_located
클릭 전: element_to_be_clickable
```

---

---

# ⑥ execute_script — JavaScript 직접 실행 ⭐️

```
Selenium 으로 안 되는 것을 JS 로 직접 실행
  .click() 이 막힐 때
  스크롤 제어
  숨겨진 요소 조작
  형제 / 부모 / 자식 요소 접근
```

## 기본 구조

```python
# JS 실행
driver.execute_script("JS 코드")

# 반환값 받기 (return 필수)
result = driver.execute_script("return document.title")

# 파이썬 요소 전달
element = driver.find_element(By.ID, "btn")
driver.execute_script("JS코드", element)
#                              ↑
#                   JS 안에서 arguments[0] 으로 접근
```

## arguments[0] 완전 정리 ⭐️

```
execute_script("JS코드", 파이썬요소1, 파이썬요소2)
                          ↑           ↑
                    arguments[0]  arguments[1]

파이썬 요소 → JS arguments 배열로 전달
JS 안에서 DOM 요소처럼 자유롭게 조작
```

```python
btn = driver.find_element(By.ID, "submit")

# JS 클릭 — .click() 이 막혔을 때
driver.execute_script("arguments[0].click();", btn)

# 요소 스크롤
driver.execute_script("arguments[0].scrollIntoView(true);", btn)

# 속성 변경
driver.execute_script("arguments[0].style.display = 'block';", btn)

# 텍스트 읽기
text = driver.execute_script("return arguments[0].textContent;", btn)

# 요소 2개 전달
el1 = driver.find_element(By.ID, "source")
el2 = driver.find_element(By.ID, "target")
driver.execute_script(
    "arguments[1].value = arguments[0].textContent;",
    el1, el2    # el1=arguments[0] / el2=arguments[1]
)
```

## nextElementSibling — 형제 요소 ⭐️

```
HTML:
  <div class="tab active">탭1</div>   ← arguments[0]
  <div class="tab">탭2</div>           ← arguments[0].nextElementSibling

Selenium 으로 찾기 어려운 경우
→ JS 로 형제 요소에 직접 접근
```

```python
tab = driver.find_element(By.CSS_SELECTOR, ".tab.active")

# 다음 탭 클릭
driver.execute_script("arguments[0].nextElementSibling.click();", tab)

# 다음 형제 텍스트
next_text = driver.execute_script(
    "return arguments[0].nextElementSibling.textContent;", tab
)
```

## DOM 탐색 전체 패턴

```python
el = driver.find_element(By.CSS_SELECTOR, ".item")

# 형제
driver.execute_script("return arguments[0].nextElementSibling;", el)
driver.execute_script("return arguments[0].previousElementSibling;", el)

# 부모 / 자식
driver.execute_script("return arguments[0].parentElement;", el)
driver.execute_script("return arguments[0].firstElementChild;", el)
driver.execute_script("return arguments[0].lastElementChild;", el)
driver.execute_script("return arguments[0].children[1];", el)

# 속성 읽기 / 쓰기
driver.execute_script("return arguments[0].getAttribute('data-id');", el)
driver.execute_script("arguments[0].setAttribute('class', 'active');", el)
```

## 스크롤 패턴

```python
# 맨 아래 / 맨 위
driver.execute_script("window.scrollTo(0, document.body.scrollHeight);")
driver.execute_script("window.scrollTo(0, 0);")

# 특정 픽셀
driver.execute_script("window.scrollTo(0, 500);")

# 요소 보이도록
el = driver.find_element(By.ID, "target")
driver.execute_script("arguments[0].scrollIntoView(true);", el)
driver.execute_script("arguments[0].scrollIntoView({behavior:'smooth'});", el)

# 현재 높이 (무한스크롤 판단용)
height = driver.execute_script("return document.body.scrollHeight")
```

## 무한 스크롤

```python
import time

last_height = driver.execute_script("return document.body.scrollHeight")

while True:
    driver.execute_script("window.scrollTo(0, document.body.scrollHeight);")
    time.sleep(2)

    new_height = driver.execute_script("return document.body.scrollHeight")
    if new_height == last_height:
        break
    last_height = new_height
```

---

---

# ⑦ 실전 패턴

## 로그인 후 데이터 수집

```python
driver.get("https://example.com/login")

driver.find_element(By.ID, "username").send_keys("myid")
driver.find_element(By.ID, "password").send_keys("mypassword")
driver.find_element(By.CSS_SELECTOR, "button[type='submit']").click()

wait.until(EC.url_contains("/dashboard"))

items = driver.find_elements(By.CSS_SELECTOR, ".data-row")
for item in items:
    print(item.text)
```

## page_source + BeautifulSoup

```python
driver.get(url)
wait.until(EC.presence_of_element_located((By.CLASS_NAME, "result")))

html = driver.page_source
from bs4 import BeautifulSoup
soup = BeautifulSoup(html, "lxml")

for item in soup.select(".result .item"):
    print(item.text)
```

## 탭 처리

```python
driver.execute_script("window.open('https://example.com', '_blank');")
tabs = driver.window_handles
driver.switch_to.window(tabs[1])   # 두 번째 탭
driver.switch_to.window(tabs[0])   # 첫 번째 탭으로 복귀
```

## 예외 처리 템플릿

```python
try:
    driver.get(url)
    el = wait.until(EC.presence_of_element_located((By.ID, "result")))
    print(el.text)
except TimeoutException:
    print("로딩 시간 초과")
except NoSuchElementException:
    print("요소 없음")
except Exception as e:
    print(f"에러: {e}")
finally:
    driver.quit()
```

---

---

# ⑧ 크롬 옵션

```python
options = Options()
options.add_argument("--headless")                                    # 창 안 띄움
options.add_argument("--no-sandbox")                                  # 샌드박스 비활성화
options.add_argument("--disable-dev-shm-usage")                      # 메모리 문제 방지
options.add_argument("--window-size=1920,1080")                      # headless 시 필수
options.add_argument("--disable-blink-features=AutomationControlled") # 봇 감지 우회
options.add_argument("user-agent=Mozilla/5.0 (Windows NT 10.0; Win64; x64)")
options.add_experimental_option("detach", True)                      # 종료 후 창 유지
```

---

---

# 자주 하는 실수

|실수|원인|해결|
|---|---|---|
|`find_element` 에러|JS 렌더링 대기 안 함|`WebDriverWait` + `EC`|
|`driver.quit()` 안 함|크롬 프로세스 누적|`try-finally` 로 항상 quit|
|`By.CLASS_NAME` 에 공백|클래스 여러 개|`By.CSS_SELECTOR` 로 대체|
|암묵적 + 명시적 대기 혼용|타이밍 꼬임|명시적 대기만 사용|
|headless 동작 안 함|창 크기 없음|`--window-size=1920,1080`|
|`.click()` 에러|요소 가려짐|`execute_script("arguments[0].click();", el)`|
|`execute_script` 반환 None|return 없음|JS 코드에 `return` 추가|
|`nextElementSibling` null|다음 형제 없음|JS 에서 null 체크|
|`find_elements` 빈 리스트|아직 로딩 중|`wait.until` 후 호출|