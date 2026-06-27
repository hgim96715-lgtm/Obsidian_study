
```
NoSQL 문서형 데이터베이스
JSON 형태로 데이터 저장 / 스키마 유연 / 수평 확장
```

---

---

## Level 0. 개념 잡기

```
MongoDB 가 뭔지, SQL 과 무엇이 다른지
```

|노트|핵심 개념|
|---|---|
|[[MongoDB_Concept]] ⭐|NoSQL 이란 / Document / Collection / SQL vs MongoDB 비교 / 언제 쓰나|
|[[MongoDB_Install_Setup]]|Docker 설치 / mongosh / 기본 명령어 / MongoDB Compass|

---

---

## Level 1. CRUD

```
데이터 넣고 꺼내고 바꾸고 지우기
```

|노트|핵심 개념|
|---|---|
|[[MongoDB_Insert]]|insertOne / insertMany / _id / ObjectId|
|[[MongoDB_Find]] ⭐|find / findOne / 조건 쿼리 / 비교 연산자($gt·$lt·$in) / 정렬·제한|
|[[MongoDB_Update]] ⭐|updateOne / updateMany / $set / $inc / $push / upsert|
|[[MongoDB_Delete]]|deleteOne / deleteMany / findOneAndDelete|

---

---

## Level 2. 쿼리 & 집계

|노트|핵심 개념|
|---|---|
|[[MongoDB_Query_Operators]] ⭐|$eq·$ne / $gt·$gte·$lt·$lte / $in·$nin / $and·$or·$not|
|[[MongoDB_Aggregation]] ⭐|파이프라인 / $match / $group / $sort / $project / $lookup(JOIN)|
|[[MongoDB_Index]] ⭐|인덱스 생성 / 복합 인덱스 / explain() / 성능 최적화|

---

---

## Level 3. 스키마 설계

```
NoSQL 이라도 설계는 중요하다
```

|노트|핵심 개념|
|---|---|
|[[MongoDB_Schema_Design]] ⭐|Embedding vs Referencing / 1:1·1:N·N:M 설계 패턴|
|[[MongoDB_Validation]]|$jsonSchema / 필드 타입·필수값 / 유효성 검사|

---

---

## Level 4. NestJS 연동

|노트|핵심 개념|
|---|---|
|[[MongoDB_Mongoose]] ⭐|Mongoose / Schema / Model / 설치 / NestJS 연동|
|[[MongoDB_NestJS_Module]]|MongooseModule.forRoot / forFeature / @InjectModel|
|[[MongoDB_NestJS_CRUD]] ⭐|@Schema / @Prop / Document 타입 / Repository 패턴|

---

---

## Level 5. 실전

|노트|핵심 개념|
|---|---|
|[[MongoDB_Transaction]]|트랜잭션 / session / withTransaction|
|[[MongoDB_Docker]]|Docker Compose / 볼륨 / 인증 설정|
|[[MongoDB_Atlas]]|클라우드 MongoDB / 무료 티어 / 연결 문자열|