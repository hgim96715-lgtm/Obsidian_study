---
aliases: [클라우드 스토리지, 파일 저장소, Cloudflare R2, public URL, R2, S3]
tags:
  - NestJS
related:
  - "[[00_NestJS_Ecosystem_HomePage]]"
  - "[[NestJS_Deploy]]"
  - "[[NestJS_Env_Config]]"
  - "[[NestJS_FileUpload]]"
---
# NestJS_FileStorage — 클라우드 파일 저장소 (S3 호환)

# 한 줄 요약

```bash
Multer 가 파일을 "받는" 역할  #→ [[NestJS_FileUpload]] 참고
이 노트는 받은 파일을 어디에 "저장"하고 URL 을 어떻게 만드는지
@aws-sdk/client-s3 로 S3 호환 저장소(R2 등)에 업로드
```

---

---

# 왜 필요한가

```txt
서버 디스크에 그대로 저장하면:
  재배포(Railway 등)할 때마다 파일 사라짐 ⚠️
  서버 여러 대로 늘리면 파일 위치 꼬임

클라우드 객체 저장소(S3 / R2)에 저장하면:
  서버와 분리되어 영구 보존
  업로드된 파일의 public URL 을 그대로 DB 컬럼에 저장 가능
    예: VisitRecord.photoUrl = 'https://xxx.r2.dev/photo123.jpg'
```

---

---

# 저장소 선택 — S3 vs Cloudflare R2

```txt
S3 호환(S3-compatible) API:
  여러 회사가 AWS S3 인터페이스를 동일하게 구현
  → 같은 SDK(@aws-sdk/client-s3)로 사용 가능, endpoint 만 바꾸면 전환

  AWS S3        원조 / egress(다운로드) 비용 있음
  Cloudflare R2  egress 비용 없음 ⭐️ / 저렴 → 이미지 자주 보여주는 서비스에 유리
```

---

---

# 설치 & 환경변수

```bash
pnpm add @aws-sdk/client-s3
```

```typescript
// src/config/env.keys.ts
export const EnvKeys = {
  S3_ACCOUNT_ID:         'S3_ACCOUNT_ID',
  S3_ENDPOINT:           'S3_ENDPOINT',
  S3_BUCKET:             'S3_BUCKET',
  S3_ACCESS_KEY_ID:      'S3_ACCESS_KEY_ID',
  S3_SECRET_ACCESS_KEY:  'S3_SECRET_ACCESS_KEY',
  S3_PUBLIC_URL:         'S3_PUBLIC_URL',
} as const;
```

```properties
# .env (Cloudflare R2 예시)
S3_ACCOUNT_ID=cloudflare_계정_id
S3_ENDPOINT=https://<ACCOUNT_ID>.r2.cloudflarestorage.com
S3_BUCKET=artinerary-photos
S3_ACCESS_KEY_ID=발급받은_access_key
S3_SECRET_ACCESS_KEY=발급받은_secret_key
S3_PUBLIC_URL=https://photos.artinerary.com
```

## Cloudflare R2 발급 절차 — 요약표 ⭐️

|단계|위치|결과 → 환경변수|
|---|---|---|
|1. 계정 생성|dash.cloudflare.com|-|
|2. 버킷 생성|R2 → Create bucket|`S3_BUCKET`|
|3. 계정 ID 확인|R2 Overview 우측|`S3_ACCOUNT_ID` → `S3_ENDPOINT` 조합|
|4. API 토큰 발급|R2 Overview → Account Details → `{}` Manage → Create User API Token|`S3_ACCESS_KEY_ID` / `S3_SECRET_ACCESS_KEY`|
|5. 공개 URL 활성화|버킷 → Settings → 공개 개발 URL(Public Development URL)|`S3_PUBLIC_URL`|

```txt
⚠️ 주의 3가지:

1. API 토큰은 버킷 안이 아니라 R2 Overview(계정 레벨) 에서 발급

2. 토큰 Permissions 선택 중요:
   "Object Read & Write" → S3 호환 키 (Access Key ID + Secret) 같이 나옴 ✅
   "Admin Read & Write"  → Cloudflare 전용 토큰 1개만 나옴 (S3 호환 아님) ❌

3. 키는 생성 시 한 번만 표시됨
   못 봤다면 계정/버킷 재생성 불필요 → 같은 위치에서 새 토큰만 다시 발급
```

```txt
어디에 쓰이는지:
  S3_ACCOUNT_ID + S3_ENDPOINT  → S3Client 접속 주소
  S3_ACCESS_KEY_ID / SECRET    → S3Client 인증
  S3_BUCKET                    → 업로드할 버킷 지정
  S3_PUBLIC_URL                → 업로드 후 반환할 이미지 URL 베이스
```

---

---

# StorageService — 지연 초기화 패턴 ⭐️

```bash
왜 생성자에서 바로 S3Client 를 안 만드나:

  환경변수가 없는 환경(로컬 테스트 등)에서도
  NestJS 가 이 서비스를 정상적으로 등록할 수 있어야 함
  → constructor 는 비워두고, 실제로 "업로드할 때"에만 client 생성/검증
#  → [[NestJS_AI#client: Anthropic | null 패턴]] 과 같은 개념
```

```typescript
import {
  BadRequestException,
  Injectable,
  ServiceUnavailableException,
} from '@nestjs/common';
import { PutObjectCommand, S3Client } from '@aws-sdk/client-s3';
import { ConfigService } from '@nestjs/config';
import { EnvKeys } from 'src/config/env.keys';
import { randomUUID } from 'crypto';

const MAX_BYTES     = 5 * 1024 * 1024;   // 5MB
const ALLOWED_MIME  = ['image/jpeg', 'image/png', 'image/webp'];

@Injectable()
export class S3StorageService {
  private client: S3Client | null = null;

  constructor(private readonly configService: ConfigService) {}

  // 환경변수를 한곳에서 모아서 읽고 검증
  private getConfig() {
    const accountId       = this.configService.getOrThrow<string>(EnvKeys.S3_ACCOUNT_ID);
    const accessKeyId     = this.configService.getOrThrow<string>(EnvKeys.S3_ACCESS_KEY_ID);
    const secretAccessKey = this.configService.getOrThrow<string>(EnvKeys.S3_SECRET_ACCESS_KEY);
    const endpoint        = this.configService.getOrThrow<string>(EnvKeys.S3_ENDPOINT);
    const bucket          = this.configService.getOrThrow<string>(EnvKeys.S3_BUCKET);
    const publicUrl       = this.configService.getOrThrow<string>(EnvKeys.S3_PUBLIC_URL);

    if (!accessKeyId || !secretAccessKey || !endpoint || !bucket || !publicUrl) {
      throw new ServiceUnavailableException('사진 업로드 설정이 되어 있지 않습니다.');
    }

    return { accountId, accessKeyId, secretAccessKey, endpoint, bucket, publicUrl };
  }

  // client 가 없을 때만 생성 — 한 번 만든 client 재사용
  private getClient() {
    if (!this.client) {
      const { accessKeyId, secretAccessKey, endpoint } = this.getConfig();
      this.client = new S3Client({
        region: 'auto',
        endpoint,
        credentials: { accessKeyId, secretAccessKey },
      });
    }
    return this.client;
  }
}
```

```txt
getConfig() 를 따로 분리한 이유:
  uploadVisitPhoto() 안에서도 bucket/publicUrl 이 필요해서
  설정 읽기 로직을 한곳에 모아두고 재사용

getClient() 가 매번 검사하는 이유:
  this.client 가 이미 있으면 그대로 반환 (재생성 안 함)
  없으면 그때 한 번만 생성 → 불필요한 S3Client 중복 생성 방지
```

## S3Client 가 뭔지 ⭐️

```txt
S3Client = @aws-sdk/client-s3 가 제공하는 클래스
  R2(또는 S3) 서버와 통신하는 "연결 통로" 역할
  이 객체를 통해서만 업로드/삭제/조회 명령(Command)을 보낼 수 있음

  비유: axios.create({ baseURL, headers }) 로 만든 axios 인스턴스와 비슷
        한 번 설정해두고 계속 재사용하는 통신 객체
```

```typescript
private client: S3Client | null = null;
```

```txt
S3Client | null 로 선언하는 이유:
  처음엔 아직 연결 객체가 없는 상태 (null)
  → getClient() 호출 시점에 실제로 생성해서 채워넣음
  → [[NestJS_AI#client: Anthropic | null 패턴]] 과 동일한 지연 초기화 패턴
```

## new S3Client({ ... }) 옵션 분해 ⭐️

```typescript
this.client = new S3Client({
  region: 'auto',
  endpoint,
  credentials: { accessKeyId, secretAccessKey },
});
```

```txt
new S3Client({ ... }) 에 넣는 설정 객체 — 옵션 3가지:

  region:
    저장소가 어느 지역에 있는지 (AWS S3 는 'ap-northeast-2' 등 실제 리전 필요)
    R2 는 리전 개념이 없는 글로벌 저장소 → 'auto' 로 고정해서 넘김

  endpoint:
    실제로 요청을 보낼 서버 주소
    AWS S3 사용 시엔 보통 생략 (SDK 가 자동으로 AWS 주소를 씀)
    R2 / 다른 S3 호환 저장소는 반드시 직접 지정
    → endpoint 하나만 바꾸면 S3 ↔ R2 전환 가능 (나머지 코드 동일)

  credentials:
    "내가 누구인지" 인증하는 키 묶음 — 객체 형태로 전달
    accessKeyId      신분증 같은 역할 (누구인지)
    secretAccessKey  비밀번호 같은 역할 (본인 확인)
    이 두 값이 EnvKeys.S3_ACCESS_KEY_ID / S3_SECRET_ACCESS_KEY 에서 옴

이 세 가지가 모여야 "어디에(endpoint) 누구로서(credentials) 어떤 지역 설정으로(region)
접속할지"가 결정되고, 그 결과로 만들어진 게 client 객체
```


---

---

# 업로드 메서드 — 검증 순서가 중요 ⭐️

```typescript
async uploadVisitPhoto(userId: number, file: Express.Multer.File) {
  // 1. 파일 존재 여부
  if (!file) {
    throw new BadRequestException('사진 파일이 없습니다.');
  }

  // 2. MIME 타입 검증 (허용된 이미지 형식인지)
  if (!ALLOWED_MIME.includes(file.mimetype)) {
    throw new BadRequestException(
      '지원하지 않는 파일 형식입니다. jpeg, png, webp만 업로드할 수 있습니다.',
    );
  }

  // 3. 파일 크기 검증
  if (file.size > MAX_BYTES) {
    throw new BadRequestException('파일 크기가 너무 큽니다. 최대 5MB까지 업로드할 수 있습니다.');
  }

  const { bucket, publicUrl } = this.getConfig();

  // 4. 확장자 결정 (MIME 타입 기반)
  const ext =
    file.mimetype === 'image/jpeg' ? 'jpg' :
    file.mimetype === 'image/png'  ? 'png' : 'webp';

  // 5. 저장 경로(key) 생성 — 사용자별 폴더 + UUID
  const key = `visits/${userId}/${randomUUID()}.${ext}`;

  // 6. 실제 업로드
  await this.getClient().send(
    new PutObjectCommand({
      Bucket:      bucket,
      Key:         key,
      Body:        file.buffer,
      ContentType: file.mimetype,
    }),
  );

  // 7. publicUrl 끝의 슬래시 정리 후 최종 URL 조합
  const base = publicUrl.replace(/\/$/, '');
  return { url: `${base}/${key}` };
}
```

```txt
검증 순서가 위→아래인 이유:
  가벼운 검사(파일 존재) 부터 먼저 끝내고
  무거운 작업(실제 업로드) 은 가장 마지막에
  → 어차피 거부될 요청이면 S3 까지 보내기 전에 빨리 차단

key = `visits/${userId}/${randomUUID()}.${ext}` 의 의미:
  visits/         용도별 폴더 구분 (나중에 다른 종류 파일과 섞이지 않게)
  ${userId}/      사용자별 폴더 → 같은 유저 파일끼리 묶임
  randomUUID()    파일명 중복 절대 방지 (Date.now() 보다 더 안전 — 같은 ms 충돌 없음)
  .${ext}         MIME 타입 기준으로 직접 결정 (원본 파일명 신뢰 안 함)

  ⚠️ file.originalname 을 그대로 안 쓰는 이유:
    사용자가 올린 파일명에 이상한 문자/경로가 섞여 있을 수 있음
    → 서버에서 직접 안전한 이름을 만들어서 사용

publicUrl.replace(/\/$/, '') 의 의미:
  $ 는 문자열 끝 / \/ 는 슬래시 한 글자
  publicUrl 끝에 사람이 실수로 '/' 를 붙여놨어도 안전하게 제거
  → 'https://photos.com/' + '/visits/...' 처럼 // 중복되는 것 방지
```

---

---

# Controller 연결 — Multer + StorageService ⭐️

```typescript
@Controller('uploads')
@UseGuards(JwtAuthGuard, RolesGuard)
@Roles(Role.USER)
export class UploadController {
  constructor(private readonly s3StorageService: S3StorageService) {}

  @Post('visit-photo')
  @UseInterceptors(
    FileInterceptor('file', {
      storage: memoryStorage(),
      limits: { fileSize: 5 * 1024 * 1024 },
    }),
  )
  uploadVisitPhoto(
    @Req() req: { user: JwtPayload },
    @UploadedFile() file: Express.Multer.File,
  ) {
    return this.s3StorageService.uploadVisitPhoto(req.user.sub, file);
  }
}
```

```bash
FileInterceptor / memoryStorage / limits 가 무엇인지는
# → [[NestJS_FileUpload]] 참고 (Multer 자체의 개념)

여기서 중요한 건 "흐름":
  1. FileInterceptor 가 multipart/form-data 를 가로채서 file 객체로 변환
  2. @UploadedFile() 로 컨트롤러가 file 받음
  3. Service 의 uploadVisitPhoto() 로 그대로 넘김 (S3 업로드는 Service 책임)

⚠️ 크기 제한이 두 군데 있는 이유:
  FileInterceptor 의 limits.fileSize  → Multer 단계에서 먼저 차단 (더 빠름)
  Service 안의 MAX_BYTES 체크         → 한 번 더 확실하게 검증 (이중 안전장치)
  둘 다 5MB 로 맞춰서 일관성 유지
```

---

---

# 파일 삭제 — DeleteObjectCommand

```typescript
async deleteFile(key: string): Promise<void> {
  await this.getClient().send(
    new DeleteObjectCommand({
      Bucket: this.getConfig().bucket,
      Key:    key,
    }),
  );
}
```

```txt
key 보관 방법:
  업로드 시 반환한 key 자체를 DB 에 별도 컬럼으로 저장해두면
  나중에 삭제할 때 publicUrl 에서 역산할 필요 없이 바로 사용 가능
```

---

---

# 한눈에

```txt
설치:
  pnpm add @aws-sdk/client-s3

환경변수:
  S3_ACCOUNT_ID / S3_ENDPOINT / S3_BUCKET
  S3_ACCESS_KEY_ID / S3_SECRET_ACCESS_KEY / S3_PUBLIC_URL

S3StorageService 패턴:
  client: S3Client | null = null   지연 초기화
  getConfig()                      환경변수 읽기 + 검증
  getClient()                      없으면 생성, 있으면 재사용

업로드 검증 순서:
  파일 존재 → MIME 타입 → 파일 크기 → 업로드

key 생성 규칙:
  `폴더/사용자ID/${randomUUID()}.확장자`
  원본 파일명 대신 서버가 직접 안전한 이름 생성

흐름:
  FileInterceptor(Multer) → Controller → Service.upload() → public URL → DB 저장
  → [[NestJS_FileUpload]] (Multer 부분 상세)
```