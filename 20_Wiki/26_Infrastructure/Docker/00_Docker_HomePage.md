

```txt
애플리케이션을 컨테이너로 패키징해서 어디서든 동일하게 실행
개발 / 테스트 / 배포 환경 통일
```

---

---

## Level 0. 개념 잡기

```txt
Docker 가 뭔지, 왜 쓰는지
```

| 노트                   | 핵심 개념                                                                       |
| -------------------- | --------------------------------------------------------------------------- |
| [[Docker_Concept]] ⭐ | Container / Image / Docker Hub / Docker Engine / docker run / docker images |

---

---

## Level 1. 기본 명령어

| 노트                    | 핵심 개념                                                                 |
| --------------------- | --------------------------------------------------------------------- |
| [[Docker_Commands]] ⭐ | 기본 명령어/이미지 관련/create/docker run/컨테이너 목록&생명주기/컨테이너 조작/명령어 한눈에          |
| [[Docker_Image]]      | 이미지 버전(TAG) / 레이어 개념 / Dockerfile FROM·RUN / 이미지 빌드 / tag / 삭제 순서     |
| [[Docker_Registry]]   | docker login / push / pull / Docker Hub / docker search / 로컬 레지스트리 구축 |


---

---

## Level 2. 컨테이너 관리

| 노트                         | 핵심 개념                                                                                        |
| -------------------------- | -------------------------------------------------------------------------------------------- |
| [[Docker_Container]] ⭐     | -d/-it 실행모드/sleep infinity / logs / exec / cp / inspect / 환경변수 / 리소스 제한/attach·Ctrl+P+Q ⭐️   |
| [[Docker_Port]] | EXPOSE vs -p 차이 ⭐️ / 포트 바인딩 문법 / ExposedPorts vs Ports (inspect) / docker port               |
| [[Docker_Volume]]          | bind mount / named volume / inspect 해석 / 컨테이너 간 공유 ⭐️ / 백업·복구 ⭐️                             |
| [[Docker_Network]]         | bridge·host·none / network create·connect / --network-alias / DNS 자동해석 / ping 해석 / 서비스 디스커버리 |


---

---

## Level 3. Dockerfile

| 노트                      | 핵심 개념                                                                         |
| ----------------------- | ----------------------------------------------------------------------------- |
| [[Docker_Dockerfile]] ⭐ | FROM / WORKDIR / COPY·ADD 차이·공백 주의 ⭐️ / build / ENV / CMD·ENTRYPOINT / VOLUME |
| [[Docker_Build]]        | docker build / 수정후재빌드 ⚠️ / 레이어캐시 / 멀티스테이지 / .dockerignore(**패턴)               |

---

---

## Level 4. Docker Compose

```txt
여러 컨테이너를 한번에 관리
```

| 노트                          | 핵심 개념                                                  |
| --------------------------- | ------------------------------------------------------ |
| [[Docker_Compose]] ⭐        | docker-compose.yml / up / down / services / depends_on |


---
---
## Level 5. 클러스터 & 오케스트레이션

|노트|핵심 개념|
|---|---|
|[[Docker_Swarm]] ⭐|Swarm init / node ls / stack deploy / service scale / 수평 확장 / REPLICAS|

---
---

## 프로젝트 적용

