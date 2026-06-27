---
aliases:
  - 가상환경
  - venv
  - requirements.txt
  - pip
  - 의존성관리
  - python환경
tags:
  - Python
  - Linux
related:
  - "[[Linux_Fundamental_Rules]]"
  - "[[Spark_Installation_Local_Docker]]"
  - "[[Dockerfile_Basics]]"
  - "[[Python_Database_Connect]]"
  - "[[Linux_Python_Env]]"
---
## 개념 한 줄 요약

**"프로젝트마다 독립된 '작업실(Workspace)'을 만들어서, 서로 다른 라이브러리 버전이 뒤섞이는 대참사를 막는 기술."**

* **`venv`:** 파이썬이 제공하는 가상 환경 생성 도구. (내 프로젝트만의 전용 방)
* **`pip`:** 파이썬 라이브러리 설치 도구. (가구 배달 기사)
* **`requirements.txt`:** 설치해야 할 라이브러리 목록이 적힌 **주문서(영수증)**.

---

## 왜 필요한가? (Why)

**문제점 (의존성 지옥, Dependency Hell):**
- "A 프로젝트는 `pandas` 1.0 버전을 써야 하고, B 프로젝트는 `pandas` 2.0을 써야 해요."
- 만약 내 컴퓨터(Global)에 그냥 설치하면? **버전 충돌**이 나서 둘 다 안 돌아갑니다.

**해결책:**
- A 방(`venv_A`)과 B 방(`venv_B`)을 따로 만듭니다.
- 각 방에 서로 다른 버전의 `pandas`를 설치하면 싸울 일이 없습니다.
- **Docker**는 이 개념을 운영체제(OS) 수준까지 확장한 것입니다.

---
## Code Core Points: ① 가상 환경 만들기 (`venv`)

터미널에서 딱 3단계만 기억하세요. **(생성 -> 진입 -> 설치)**

```bash
# 1. 가상 환경 생성 (방 만들기)
# 관례적으로 이름을 .venv 또는 venv 라고 짓습니다.
python3 -m venv .venv

# 2. 가상 환경 진입 (방 들어가기) ⭐️
# 실행하면 프롬프트 앞에 (.venv) 라고 뜹니다.
source .venv/bin/activate

# 3. 라이브러리 설치 (가구 들여놓기)
pip install pandas numpy
```

(참고: 나올 때는 `deactivate` 명령어를 치면 됩니다.)

---
## ② `requirements.txt` 가 뭔가요?

**"제 컴퓨터랑 똑같은 환경으로 세팅해 주세요!"** 라고 할 때 주는 **구매 목록(Shopping List)** 입니다.

### 상황:

혼자 개발할 땐 `pip install`로 하나씩 깔면 되지만, 이 코드를 동료(또는 Docker)에게 줄 때는 **"내가 뭘 깔았는지"** 알려줘야 합니다.

### 사용법 1: 목록 만들기 (내보내기)

내가 현재 설치한 라이브러리들을 파일로 저장합니다.

```bash
# "현재 방에 있는 모든 패키지 이름과 버전을 얼려서(freeze) 파일에 적어라"
pip freeze > requirements.txt
```

(결과물 예시)

```text
pandas==2.0.3
numpy==1.24.3
pyspark==3.5.1
```

### 사용법 2: 목록대로 설치하기 (불러오기) 

Docker나 동료의 컴퓨터에서 이 파일을 보고 한 방에 설치합니다.

```bash
# "-r: 이 파일(requirements.txt)을 읽어서(read) 다 설치해줘"
pip3 install -r requirements.txt
```

---
## 6. 심화: 실무에서 쓰는 pip 옵션 꿀팁 (`-r`, `--no-cache-dir`)

협업이나 Docker 환경에서는 `pip install` 뒤에 옵션을 주렁주렁 달게 됩니다.
각각의 의미를 알아둡시다.

### ① `-r` (Requirement File)

* **의미:** "패키지 이름을 하나하나 치지 말고, **이 파일(requirements.txt)에 적힌 대로** 다 깔아줘."
* **사용법:** `pip install -r requirements.txt`
* **왜 쓰는가?** 사람이 일일이 타이핑하면 오타가 나거나 버전을 빼먹을 수 있으니까요.

### ② `--no-cache-dir` (No Cache) 

* **의미:** "다운로드 받은 설치 파일(.whl)을 캐시(임시 저장소)에 남기지 말고 **설치 끝나면 바로 지워.**"
* **사용법:** `pip install --no-cache-dir pandas`
* **왜 쓰는가?**
    * **Docker 이미지 만들 때 필수:** 이미지는 용량이 작을수록 좋습니다. 설치 파일 찌꺼기를 남겨서 용량을 늘릴 필요가 없죠.
    * **내 컴퓨터에서는?** 굳이 안 써도 됩니다. 캐시가 있으면 다음에 똑같은 걸 깔 때 다운로드 속도가 빨라지니까요.

### ③ 종합 예시 (Docker 단골 명령어)


```bash
# "캐시 남기지 말고(용량 절약), 리스트 파일 읽어서(한방에) 설치해라"
pip3 install --no-cache-dir -r requirements.txt
```


---
## 초보자가 자주 하는 실수 (Misconceptions)

### ① "`requirements.txt` 파일은 자동으로 생기나요?"

- 아니요! `pip install`만 한다고 파일이 생기지 않습니다.
- 반드시 **`pip freeze > requirements.txt`** 명령어로 직접 만들어줘야 합니다.

### ② "가상 환경 폴더(`.venv`)를 깃헙(Git)에 올려버렸어요."

- **절대 금지!** 🚫
- `.venv` 폴더는 용량이 크고(수백 MB), OS마다 파일이 다릅니다.
- Git에는 **`requirements.txt` 종이 한 장**만 올리는 겁니다. 다른 사람은 그 종이를 보고 각자 알아서 설치(`pip install -r ...`)하는 구조입니다.
- `.gitignore` 파일에 `.venv/`를 꼭 추가하세요.

### ③ "Docker 쓸 때도 가상 환경이 필요한가요?"

- Docker 컨테이너 자체가 거대한 가상 환경(격리 공간)이라서, 컨테이너 안에서는 굳이 `venv`를 또 만들지 않고 그냥 설치(`pip install`)하는 경우가 많습니다. 