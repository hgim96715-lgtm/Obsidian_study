---
aliases:
  - FFmpeg
  - 미디어 변환
tags:
  - Linux
related:
  - "[[00_Linux_HomePage]]"
  - "[[NestJS_FFmpeg]]"
---

# Linux_FFmpeg — FFmpeg 개념 & 명령어

```
FFmpeg = 영상·음성·이미지를 변환·편집·분석하는 CLI 도구
오픈소스 / 크로스플랫폼 / 거의 모든 미디어 포맷 지원
```

---

---

# FFmpeg 이란 ⭐️

```
FFmpeg (Fast Forward MPEG):
  영상 / 음성 / 이미지 파일을 다루는 모든 것을 할 수 있는 도구
  변환 / 압축 / 자르기 / 합치기 / 스트리밍 / 썸네일 추출

구성 요소:
  ffmpeg    파일 변환·처리 (가장 많이 사용)
  ffprobe   파일 정보 분석 (메타데이터 확인)
  ffplay    미디어 재생

왜 서버에서 쓰나:
  영상 업로드 → 서버에서 자동으로 포맷 변환 / 압축 / 썸네일 생성
  사용자가 업로드한 영상을 웹에서 재생 가능한 형태로 변환
  → 브라우저는 모든 포맷 재생 불가 → mp4/h264 로 변환 필요
```

---

---

# 설치

```bash
# Ubuntu / Debian
sudo apt update
sudo apt install ffmpeg -y

# 설치 확인
ffmpeg -version
ffprobe -version
```

---

---

# 기본 구조

```bash
ffmpeg [입력 옵션] -i 입력파일 [출력 옵션] 출력파일
#                  ↑ -i = input
```

---

---

# 주요 명령어 ⭐️

## 파일 정보 확인

```bash
# 영상 정보 확인 (해상도 / 코덱 / 길이 / 비트레이트)
ffprobe -v quiet -print_format json -show_streams video.mp4
ffprobe video.mp4
```

## 포맷 변환

```bash
# mp4 → avi
ffmpeg -i input.mp4 output.avi

# avi → mp4 (H.264 코덱)
ffmpeg -i input.avi -c:v libx264 -c:a aac output.mp4
```

## 영상 압축

```bash
# 파일 크기 줄이기 (CRF: 낮을수록 고화질 / 18~28 권장)
ffmpeg -i input.mp4 -crf 23 output.mp4

# 해상도 변경 (1920x1080 → 1280x720)
ffmpeg -i input.mp4 -vf scale=1280:720 output.mp4
```

## 썸네일 추출 ⭐️

```bash
# 특정 시간(00:00:01)에서 이미지 1장 추출
ffmpeg -i input.mp4 -ss 00:00:01 -vframes 1 thumbnail.jpg

# 옵션 설명:
# -ss  탐색 시작 시간 (HH:MM:SS)
# -vframes 1  프레임 1장만 추출
```

## 영상 자르기

```bash
# 10초부터 30초까지 자르기
ffmpeg -i input.mp4 -ss 00:00:10 -to 00:00:30 -c copy output.mp4

# -ss  시작 시간
# -to  종료 시간
# -c copy  재인코딩 없이 복사 (빠름)
```

## 음성 추출

```bash
ffmpeg -i input.mp4 -vn -c:a copy output.aac
# -vn = 비디오 제거 / -c:a copy = 오디오 그대로 복사
```

## 영상에서 음성 제거

```bash
ffmpeg -i input.mp4 -an output.mp4
# -an = 오디오 제거
```

---

---

# 주요 옵션 한눈에

|옵션|의미|
|---|---|
|`-i`|입력 파일|
|`-c:v`|비디오 코덱 (libx264, copy 등)|
|`-c:a`|오디오 코덱 (aac, copy 등)|
|`-vf`|비디오 필터 (scale, crop 등)|
|`-ss`|시작 시간|
|`-to`|종료 시간|
|`-t`|지속 시간 (초)|
|`-crf`|품질 (18~28, 낮을수록 고화질)|
|`-vframes N`|추출할 프레임 수|
|`-vn`|비디오 스트림 제거|
|`-an`|오디오 스트림 제거|
|`-y`|출력 파일 덮어쓰기 확인 없이|

---

---

# NestJS 에서 사용

```
→ [[NestJS_FFmpeg]] 참고
fluent-ffmpeg 패키지로 NestJS 에서 FFmpeg 명령어를 코드로 실행
```