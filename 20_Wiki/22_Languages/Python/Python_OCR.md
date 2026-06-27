---
aliases:
  - OCR
  - 이미지 텍스트 추출
  - EasyOCR
  - Tesseract
tags:
  - Python
related:
  - "[[00_Python_HomePage]]"
  - "[[Python_Web_Scraping]]"
  - "[[Python_Selenium]]"
---


# Python_OCR — 이미지 텍스트 추출

## 한 줄 요약

```
OCR (Optical Character Recognition)
= 이미지 / PDF 안의 텍스트를 읽어서 문자열로 변환
```

---

---

# ① OCR 라이브러리 비교 ⭐️

|라이브러리|한글 정확도|속도|비용|설치|
|---|---|---|---|---|
|**EasyOCR**|⭐⭐⭐⭐|보통|무료|pip 만|
|**Tesseract**|⭐⭐⭐|느림|무료|시스템 설치 필요|
|**Google Vision**|⭐⭐⭐⭐⭐|빠름|유료|API Key|

```
선택 기준:
  빠르게 시작 / 무료 / 한글 필요   → EasyOCR  ← 추천
  서버에 설치 / 오프라인 / 무료     → Tesseract
  정확도 최우선 / 비용 OK          → Google Vision
```

---

---

# ② EasyOCR — 가장 쉽고 한글 잘 됨 ⭐️

## 설치

```bash
pip install easyocr pillow numpy
```

```
처음 실행 시 언어 모델 자동 다운로드 (수백 MB)
GPU 없어도 CPU 로 동작 (느리지만 됨)
```

## OCR 초기화 패턴 ⭐️

```python
import easyocr

# 기본 초기화
reader = easyocr.Reader(["ko", "en"], gpu=False)

# 클래스 안에서 초기화 — use_ocr 플래그로 제어
class Scraper:
    def __init__(self, use_ocr=True):
        if use_ocr:
            print("OCR 엔진 초기화 중...")
            self.ocr_reader = easyocr.Reader(["ko", "en"], gpu=False)
            print("OCR 엔진 준비 완료!")
        else:
            self.ocr_reader = None   # OCR 안 쓸 때

    def extract_text(self, img_array):
        if self.ocr_reader is None:
            return []
        return self.ocr_reader.readtext(img_array, detail=0)
```

```
초기화는 무겁기 때문에:
  __init__ 에서 한 번만 생성
  매번 Reader() 만들면 성능 저하
  use_ocr 플래그로 필요할 때만 초기화
```

## 기본 사용법

```python
import easyocr

reader = easyocr.Reader(["ko", "en"])
results = reader.readtext("image.png")

for bbox, text, confidence in results:
    print(f"텍스트: {text}")
    print(f"신뢰도: {confidence:.2f}")
```

## readtext() 옵션 완전 정리 ⭐️

```python
results = reader.readtext(
    image,              # 이미지 (경로 / numpy array / bytes)
    detail=1,           # 1=bbox+text+conf 전부 / 0=텍스트만 리스트
    paragraph=False,    # True=문장 단위로 합치기 / False=줄 단위
    mag_ratio=1.5,      # 이미지 확대 배율 (작은 글씨 정확도 향상)
    contrast_ths=0.1,   # 대비 임계값 (낮을수록 저대비 이미지 처리)
    adjust_contrast=0.5,# 대비 자동 조정 강도
    text_threshold=0.7, # 텍스트 감지 임계값
    low_text=0.4,       # 낮은 신뢰도 텍스트 포함 여부
)
```

```
자주 쓰는 조합:

# 빠르게 텍스트만
results = reader.readtext(img, detail=0)

# 문장 단위로 합쳐서 (긴 문장 읽을 때)
results = reader.readtext(img, detail=0, paragraph=True)

# 작은 글씨 / 저화질 이미지
results = reader.readtext(
    img,
    detail=0,
    paragraph=True,
    mag_ratio=1.5,        # 1.5배 확대
    contrast_ths=0.1      # 낮은 대비도 처리
)
```

## 결과 형식 이해

```python
results = reader.readtext("image.png")

# results 예시:
# [
#   ([[10,5],[200,5],[200,30],[10,30]], "안녕하세요", 0.95),
#   ([[10,40],[150,40],[150,60],[10,60]], "Hello", 0.89),
# ]

# bbox  = 텍스트 영역 좌표 (4개 꼭짓점)
# text  = 추출된 텍스트
# conf  = 신뢰도 (0~1)

# detail=0 이면 텍스트만
texts = reader.readtext("image.png", detail=0)
# ["안녕하세요", "Hello"]
```

## 신뢰도 필터링

```python
results = reader.readtext("image.png")
filtered = [text for _, text, conf in results if conf > 0.5]
```

## 이미지 입력 형식 ⭐️

```python
# 1. 파일 경로
reader.readtext("image.png")
reader.readtext("/path/to/image.jpg")

# 2. numpy 배열 (가장 많이 씀)
import numpy as np
from PIL import Image

img = Image.open("image.png")
img_array = np.array(img)          # PIL → numpy 변환
results = reader.readtext(img_array)

# 3. bytes
with open("image.png", "rb") as f:
    img_bytes = f.read()
results = reader.readtext(img_bytes)

# 4. URL
results = reader.readtext("https://example.com/image.jpg")
```

## PIL + BytesIO — URL 이미지 → numpy 변환 ⭐️

```python
import requests
import numpy as np
from PIL import Image
from io import BytesIO

# URL 에서 이미지 다운로드
response = requests.get(image_url)

# bytes → PIL Image (RGB 로 변환)
img = Image.open(BytesIO(response.content)).convert("RGB")

# PIL → numpy 배열
img_array = np.array(img)

# EasyOCR 에 전달
results = reader.readtext(img_array, detail=0)
```

```
왜 np.array() 가 필요한가:
  PIL Image 는 EasyOCR 이 직접 못 받는 경우 있음
  np.array() 로 변환하면 어떤 이미지 형식이든 통일됨
  OpenCV (cv2) 도 numpy 배열로 다룸

  PIL Image → np.array → EasyOCR  (가장 안전한 흐름)
```

## 이미지 슬라이싱 후 OCR

```python
from PIL import Image
import numpy as np
import easyocr

reader = easyocr.Reader(["ko", "en"], gpu=False)

img = Image.open("screenshot.png").convert("RGB")

# 특정 영역 자르기 (left, upper, right, lower)
sliced_img = img.crop((100, 200, 500, 350))

# PIL → numpy 변환 (필수)
img_array = np.array(sliced_img)

# OCR 실행
results = reader.readtext(
    img_array,
    detail=0,
    paragraph=True,
    mag_ratio=1.5,
    contrast_ths=0.1
)
print(results)
```

## GPU 설정

```python
reader = easyocr.Reader(["ko", "en"], gpu=True)   # GPU 사용 (빠름)
reader = easyocr.Reader(["ko", "en"], gpu=False)  # CPU (GPU 없을 때)
```

---

---

# ③ Tesseract — 오프라인 / 서버 설치형

## 설치

```bash
# Ubuntu / Debian
sudo apt-get install tesseract-ocr
sudo apt-get install tesseract-ocr-kor   # 한국어 언어팩

# Mac
brew install tesseract
brew install tesseract-lang

# Python 바인딩
pip install pytesseract Pillow
```

## 기본 사용법

```python
import pytesseract
from PIL import Image

# 이미지 열기
img = Image.open("image.png")

# 텍스트 추출 (영어 기본)
text = pytesseract.image_to_string(img)
print(text)

# 한국어
text = pytesseract.image_to_string(img, lang="kor")

# 한국어 + 영어
text = pytesseract.image_to_string(img, lang="kor+eng")
```

## 설치 경로 지정 (Windows)

```python
import pytesseract

pytesseract.pytesseract.tesseract_cmd = r"C:\Program Files\Tesseract-OCR\tesseract.exe"
```

## 이미지 전처리 → 정확도 향상

```python
import cv2
import numpy as np
from PIL import Image

img = cv2.imread("image.png")

# 그레이스케일 변환
gray = cv2.cvtColor(img, cv2.COLOR_BGR2GRAY)

# 이진화 (흑백으로)
_, binary = cv2.threshold(gray, 128, 255, cv2.THRESH_BINARY)

# 노이즈 제거
denoised = cv2.fastNlMeansDenoising(binary)

# pytesseract 에 전달
pil_img = Image.fromarray(denoised)
text = pytesseract.image_to_string(pil_img, lang="kor")
```

---

---

# ④ EasyOCR 실전 패턴

## Selenium + EasyOCR — 화면 캡처 후 OCR

```python
from selenium import webdriver
import easyocr
import base64

driver = webdriver.Chrome(...)
driver.get("https://example.com")

# 스크린샷 캡처
driver.save_screenshot("screenshot.png")

# OCR 로 텍스트 추출
reader = easyocr.Reader(["ko", "en"])
results = reader.readtext("screenshot.png", detail=0)
print(results)

driver.quit()
```

## 특정 영역만 OCR

```python
from PIL import Image
import easyocr

reader = easyocr.Reader(["ko", "en"])

# 이미지 열기
img = Image.open("screenshot.png")

# 특정 영역 자르기 (left, upper, right, lower)
cropped = img.crop((100, 200, 500, 350))
cropped.save("cropped.png")

# 자른 영역만 OCR
results = reader.readtext("cropped.png", detail=0)
print(results)
```

## 결과 DataFrame 으로 정리

```python
import easyocr
import pandas as pd

reader = easyocr.Reader(["ko", "en"])
results = reader.readtext("image.png")

df = pd.DataFrame(results, columns=["bbox", "text", "confidence"])
df = df[df["confidence"] > 0.5]   # 신뢰도 필터링
print(df[["text", "confidence"]])
```

---

---

# ⑤ 언어 코드 목록

```
"ko"    한국어
"en"    영어
"ja"    일본어
"ch_sim" 중국어 간체
"ch_tra" 중국어 번체

# 여러 언어 동시
reader = easyocr.Reader(["ko", "en", "ja"])
```

---

---

# 자주 하는 실수

|실수|원인|해결|
|---|---|---|
|처음 실행이 너무 느림|모델 다운로드 중|기다리면 됨 (한 번만)|
|한글이 깨짐|언어 설정 안 함|`Reader(["ko", "en"])`|
|정확도 낮음|이미지 품질 낮음|`mag_ratio=1.5` / 이진화 전처리|
|`detail=0` 결과가 빈 리스트|텍스트 없음 or 신뢰도 낮음|`detail=1` 로 confidence 확인|
|GPU 없는데 gpu=True|CUDA 없음|`gpu=False` 설정|
|Tesseract 경로 에러|설치 경로 못 찾음|`tesseract_cmd` 직접 지정|
|`Reader()` 를 매번 만듦|초기화 비용 큼|`__init__` 에서 한 번만 생성|
|PIL Image 바로 전달 에러|형식 불일치|`np.array(img)` 로 변환 후 전달|
|URL 이미지 OCR 안 됨|bytes 변환 안 함|`Image.open(BytesIO(response.content)).convert("RGB")`|