---
aliases:
  - translator
  - library
  - deep-translator
tags:
  - Python
related:
  - "[[00_Python_HomePage]]"
  - "[[Sklearn_TF_IDF]]"
  - "[[Python_Library_Faker]]"
---


> 한국어 ↔ 영어 번역 API 래퍼 라이브러리 이 과제에서 쓴 이유: NCS 수행준거(한국어) → O*NET 비교용 영어 변환

---

## 설치

```bash
pip install deep-translator
```

> 공식 문서: https://deep-translator.readthedocs.io/en/latest/usage.html

---

## 기본 사용법

```python
from deep_translator import GoogleTranslator

# 단일 문장 번역
translated = GoogleTranslator(source='ko', target='en').translate("사업관리")
print(translated)  # Business management
```

### 파라미터

```
source : 원본 언어 코드  ('ko', 'en', 'ja', 'zh-CN' ...)
target : 번역할 언어 코드
'auto' : source를 'auto'로 하면 언어 자동 감지
```

---

## 실전 코드 - 리스트 전체 번역

```python
from deep_translator import GoogleTranslator

# NCS 수행준거 전체 번역 → 하나의 영어 텍스트로 합치기
ncs_kos = df_business["수행준거"].dropna().unique()

en_kos = ''
for ko in ncs_kos:
    translated = GoogleTranslator(source='ko', target='en').translate(str(ko))
    en_kos += ' ' + translated  # 공백으로 이어붙이기

print(en_kos[:300])  # 확인
```

### str(ko)를 쓰는 이유

```python
# 수행준거 컬럼에 nan(float)이 섞일 수 있음
# translate()는 문자열만 받음 → str()로 강제 변환

# dropna()로 nan 제거했어도 str() 감싸는 게 안전
translated = GoogleTranslator(...).translate(str(ko))
```

---

## 긴 텍스트 분할 번역 (글자 수 제한 대응)

Google 번역은 한 번에 **5000자** 제한이 있어요.

```python
def translate_long_text(text, chunk_size=4000):
    """긴 텍스트를 chunk_size 단위로 잘라서 번역"""
    chunks = [text[i:i+chunk_size] for i in range(0, len(text), chunk_size)]
    translated_chunks = [
        GoogleTranslator(source='ko', target='en').translate(chunk)
        for chunk in chunks
    ]
    return ' '.join(translated_chunks)

# 사용
long_korean_text = "매우 긴 한국어 텍스트..."
result = translate_long_text(long_korean_text)
```

---

## 주의사항

```
1. 인터넷 연결 필요 (Google 서버 호출)
2. 무료지만 너무 많이 호출하면 일시 차단될 수 있음
3. 번역 품질은 완벽하지 않음 (전문 용어 오역 가능)
4. 상업용으로 쓰면 Google 이용약관 확인 필요
```

---

## 다른 번역 엔진

```python
from deep_translator import (
    GoogleTranslator,       # 가장 많이 씀
    DeepLTranslator,        # 품질 좋음 (API 키 필요)
    PapagoTranslator,       # 한국어 특화 (API 키 필요)
    MicrosoftTranslator,    # Azure (API 키 필요)
)
```

---
