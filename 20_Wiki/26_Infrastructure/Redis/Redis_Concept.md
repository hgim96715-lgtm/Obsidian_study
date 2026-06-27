---
aliases:
  - Redis 개념
  - 인메모리
  - Key-Value
tags:
  - Redis
related:
  - "[[00_Redis_HomePage]]"
  - "[[Redis_Install_Setup]]"
  - "[[NestJS_Caching]]"
---

# Redis_Concept — Redis 개념

## 한 줄 요약

```
Redis = Remote Dictionary Server
메모리에 데이터를 저장하는 Key-Value 저장소
빠른 읽기/쓰기가 필요한 곳에 사용
```

---

---

#  Redis 란 ⭐️

```
Redis = Remote Dictionary Server
  데이터를 메모리(RAM)에 저장 → 매우 빠름
  Key-Value 구조 → 키로 값을 저장하고 꺼냄
  오픈소스 / 가장 널리 쓰이는 인메모리 저장소

인메모리란:
  디스크(SSD/HDD) 가 아닌 RAM 에 저장
  RAM = 휘발성 (서버 꺼지면 데이터 사라짐)
  → 영속성 필요 시 RDB/AOF 설정 필요
  → 빠른 속도가 장점
```

---

---

#  DB vs Redis ⭐️

```
                  PostgreSQL / MySQL    Redis
저장 위치          디스크(SSD/HDD)       메모리(RAM)
속도               상대적으로 느림        매우 빠름 (마이크로초)
데이터 영속성       영구 저장             기본 휘발성 (설정으로 유지 가능)
데이터 구조         테이블 / 행 / 열      Key-Value / List / Hash / Set 등
주 용도            비즈니스 데이터 저장   캐싱 / 세션 / 임시 데이터
쿼리               SQL                  Redis 명령어 (GET, SET 등)

함께 쓰는 패턴:
  DB (영구 저장) + Redis (자주 쓰는 데이터 캐싱)
  DB 부하 줄이고 응답 속도 높이기
```

---

---

#  언제 쓰나 ⭐️

```
캐싱:
  DB 조회 결과를 Redis 에 저장 → 다음 요청 시 DB 안 거침
  영화 정보 / 랭킹 / 검색 결과

세션 / 인증:
  JWT 블랙리스트 / 로그인 세션 저장
  토큰 검증 결과 캐싱

Rate Limiting:
  요청 횟수를 Redis 에 카운팅
  초당 / 분당 요청 제한

실시간 기능:
  채팅 / 알림 (Pub/Sub)
  실시간 랭킹 (Sorted Set)

임시 데이터:
  이메일 인증번호 (TTL 3분)
  비밀번호 재설정 토큰
```

---

---

#  Key-Value 구조

```
Key   = 데이터를 찾는 식별자 (문자열)
Value = 저장할 데이터 (다양한 타입)

예시:
  Key: "movie:1"         Value: {id:1, title:"기생충"}
  Key: "user:session:abc" Value: {userId:1, role:"admin"}
  Key: "email:482910"    Value: "482910" (인증번호)

키 네이밍 관례:
  : 으로 계층 구분
  도메인:타입:식별자
  예: auth:block:access:해시값
```