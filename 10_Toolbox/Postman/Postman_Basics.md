---
aliases:
  - Postman 기본 사용법
tags:
  - Postman
related:
  - "[[00_Postman_HomePage]]"
  - "[[NestJS_Concept]]"
---
# Postman_Basics — 기본 사용법

```
Postman 으로 API 를 테스트하는 기본 흐름
서버 실행 → 요청 전송 → 응답 확인
```

---

---

# 기본 사용 흐름

```
1. 서버 실행 (pnpm run start:dev)
2. Postman 에서 Method + URL 입력
3. 필요하면 Headers / Body 설정
4. Send 클릭 → 응답 확인
```

---

---

# 요청 구성 ⭐️

```
Method  GET / POST / PATCH / DELETE 선택

URL     요청 주소
        http://localhost:3000/movie

Params  Query Parameter
        ?title=기생충&take=10
        → Params 탭에서 key / value 로 입력

Headers 헤더 설정
        Authorization: Bearer {{accessToken}}
        Content-Type: application/json (Body 있을 때 자동)

Body    요청 데이터 (POST / PATCH 에서 사용)
        → raw → JSON 선택
        {
          "title": "기생충",
          "detail": "봉준호 감독"
        }
```

---

---

# HTTP Method 별 용도

|Method|용도|Body|
|---|---|---|
|`GET`|조회|없음|
|`POST`|생성|있음|
|`PATCH`|일부 수정|있음|
|`PUT`|전체 교체|있음|
|`DELETE`|삭제|없음|

---

---

# Path Variable vs Query Parameter

```bash
# Path Variable — 특정 리소스 식별
GET http://localhost:3000/movie/1
#                                ↑ :id = 1

# Query Parameter — 필터 / 검색
GET http://localhost:3000/movie?title=기생충&take=10
#                               ↑ Params 탭에서 입력
```

---

---

# JSON Body 입력 ⭐️

```
Body 탭 → raw → JSON 선택

{
  "title": "기생충",
  "detail": "봉준호 감독"
}

⚠️ 반드시 큰따옴표("") 사용
   작은따옴표('') 는 JSON 파싱 에러
```