---
aliases:
  - S3 업로드
  - 클라우드 스토리지
  - 파일 저장소
  - Cloudflare R2
  - S3Client
  - R2
  - S3 호환
  - PutObjectCommand
  - 오브젝트 스토리지
tags:
  - NestJS
related:
  - "[[00_NestJS_Ecosystem_HomePage]]"
  - "[[NestJS_FileUpload_r]]"
  - "[[NestJS_Env_Config]]"
  - "[[NestJS_Excel]]"
---
# NestJS_S3Storage — S3 호환 클라우드 파일 저장

> [!info]
> Multer가 클라이언트에서 파일을 **받는** 역할 → [[NestJS_FileUpload_r]]
> 이 노트는 받은 파일을 클라우드 저장소(S3 / R2)에 **저장**하고 public URL을 반환하는 패턴.
>
> - `@aws-sdk/client-s3` → S3 호환 API로 업로드·삭제
> - `S3Client | null` 지연 초기화 패턴 (환경변수 없는 환경에서도 서비스 등록 가능)
> - 검증 순서: 파일 존재 → MIME 타입 → 크기 → 업로드
> - key = `폴더/${userId}/${randomUUID()}.ext` → 안전한 경로 서버에서 직접 생성

---

# 왜 클라우드 저장소인가

```txt
서버 디스크에 저장하면:
  재배포(Railway 등)할 때마다 파일 사라짐 ⚠️
  서버 인스턴스 여러 개로 늘리면 파일 위치 꼬임

S3 / R2에 저장하면:
  서버와 분리 → 영구 보존
  업로드된 파일의 public URL을 DB 컬럼에 바로 저장 가능
    예: VisitRecord.photoUrl = 'https://xxx.r2.dev/photo123.jpg'
```

---

# S3 vs Cloudflare R2

```txt
S3 호환(S3-compatible) API:
  여러 회사가 AWS S3 인터페이스를 동일하게 구현
  → 같은 SDK(@aws-sdk/client-s3)로 사용 가능, endpoint만 바꾸면 전환

  AWS S3         원조 / egress(다운로드) 비용 있음
  Cloudflare R2  egress 비용 없음 ⭐️ / 저렴 → 이미지 자주 보여주는 서비스에 유리
```

---

# 설치 & 환경변수

```bash
pnpm add @aws-sdk/client-s3
```

```typescript
// src/config/env.keys.ts
export const EnvKeys = {
  S3_ACCOUNT_ID:        'S3_ACCOUNT_ID',
  S3_ENDPOINT:          'S3_ENDPOINT',
  S3_BUCKET:            'S3_BUCKET',
  S3_ACCESS_KEY_ID:     'S3_ACCESS_KEY_ID',
  S3_SECRET_ACCESS_KEY: 'S3_SECRET_ACCESS_KEY',
  S3_PUBLIC_URL:        'S3_PUBLIC_URL',
} as const;
```

```properties
# .env (Cloudflare R2 예시)
S3_ACCOUNT_ID=cloudflare_계정_id
S3_ENDPOINT=https://<ACCOUNT_ID>.r2.cloudflarestorage.com
S3_BUCKET=my-bucket
S3_ACCESS_KEY_ID=발급받은_access_key
S3_SECRET_ACCESS_KEY=발급받은_secret_key
S3_PUBLIC_URL=https://photos.example.com
```

```txt
각 변수가 쓰이는 곳:
  S3_ACCOUNT_ID + S3_ENDPOINT  → S3Client 접속 주소
  S3_ACCESS_KEY_ID / SECRET    → S3Client 인증
  S3_BUCKET                    → 업로드할 버킷 지정
  S3_PUBLIC_URL                → 업로드 후 반환할 이미지 URL 베이스
```

## Cloudflare R2 발급 절차 ⭐️

| 단계 | 위치 | 결과 → 환경변수 |
|---|---|---|
| 1. 계정 생성 | dash.cloudflare.com | - |
| 2. 버킷 생성 | R2 → Create bucket | `S3_BUCKET` |
| 3. 계정 ID 확인 | R2 Overview 우측 | `S3_ACCOUNT_ID` → `S3_ENDPOINT` 조합 |
| 4. API 토큰 발급 | R2 Overview → Manage R2 API Tokens → Create API Token | `S3_ACCESS_KEY_ID` / `S3_SECRET_ACCESS_KEY` |
| 5. 공개 URL 활성화 | 버킷 → Settings → Public Development URL | `S3_PUBLIC_URL` |

> [!warning] 주의 3가지
> 1. API 토큰은 버킷 안이 아닌 **R2 Overview(계정 레벨)** 에서 발급
> 2. Permissions 선택 중요:
>    - `Object Read & Write` → S3 호환 키 (Access Key ID + Secret) 같이 나옴 ✅
>    - `Admin Read & Write` → Cloudflare 전용 토큰만 나옴 (S3 호환 아님) ❌
> 3. 키는 생성 시 **한 번만** 표시됨 — 못 봤으면 계정 재생성 불필요, 같은 위치에서 새 토큰만 다시 발급

---

# S3StorageService — 지연 초기화 패턴 ⭐️⭐️⭐️⭐️

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

const MAX_BYTES    = 5 * 1024 * 1024;                        // 5MB
const ALLOWED_MIME = ['image/jpeg', 'image/png', 'image/webp'];

@Injectable()
export class S3StorageService {
  private client: S3Client | null = null;  // ← 지연 초기화: 처음엔 null

  constructor(private readonly configService: ConfigService) {}

  // 환경변수를 한 곳에서 모아 읽고 검증
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

  // client가 없을 때만 생성 — 이미 있으면 재사용
  private getClient() {
    if (!this.client) {
      const { accessKeyId, secretAccessKey, endpoint } = this.getConfig();
      this.client = new S3Client({
        region: 'auto',   // R2는 리전 개념 없음 → 'auto' 고정
        endpoint,
        credentials: { accessKeyId, secretAccessKey },
      });
    }
    return this.client;
  }
}
```

```txt
왜 생성자에서 바로 S3Client를 안 만드나:
  환경변수가 없는 환경(로컬 테스트 등)에서도
  NestJS가 이 서비스를 정상적으로 등록할 수 있어야 함
  → constructor는 비워두고, 실제 "업로드할 때"에만 client 생성·검증
  → NestJS_AI의 client: Anthropic | null 패턴과 동일

getConfig() 분리 이유:
  uploadFile()과 deleteFile() 양쪽에서 bucket/publicUrl이 필요
  → 설정 읽기 로직을 한 곳에 모아 재사용

getClient() 재진입 안전:
  this.client가 이미 있으면 그대로 반환 → S3Client 중복 생성 방지
```

## S3Client — new S3Client({ ... }) 옵션 ⭐️

```typescript
new S3Client({
  region: 'auto',
  endpoint,
  credentials: { accessKeyId, secretAccessKey },
});
```

```txt
region:
  저장소가 어느 지역에 있는지 (AWS S3 → 'ap-northeast-2' 등 실제 리전 필요)
  R2는 글로벌 저장소라 리전 개념 없음 → 'auto' 고정

endpoint:
  실제 요청을 보낼 서버 주소
  AWS S3는 보통 생략 (SDK가 자동으로 AWS 주소 씀)
  R2 / 기타 S3 호환 저장소는 반드시 직접 지정
  → endpoint 하나만 바꾸면 S3 ↔ R2 전환 가능 (나머지 코드 동일)

credentials:
  accessKeyId      신분증 역할 (누구인지)
  secretAccessKey  비밀번호 역할 (본인 확인)
```

---

# 파일 업로드 — 검증 순서가 중요 ⭐️⭐️⭐️⭐️

```typescript
async uploadVisitPhoto(userId: number, file: Express.Multer.File) {
  // 1. 파일 존재 여부
  if (!file) throw new BadRequestException('사진 파일이 없습니다.');

  // 2. MIME 타입 검증 (허용된 이미지 형식인지)
  if (!ALLOWED_MIME.includes(file.mimetype)) {
    throw new BadRequestException('jpeg, png, webp만 업로드할 수 있습니다.');
  }

  // 3. 파일 크기 검증
  if (file.size > MAX_BYTES) {
    throw new BadRequestException('최대 5MB까지 업로드할 수 있습니다.');
  }

  const { bucket, publicUrl } = this.getConfig();

  // 4. 확장자 결정 (MIME 타입 기반 — 원본 파일명 신뢰 안 함)
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

  // 7. publicUrl 끝 슬래시 정리 후 최종 URL 조합
  const base = publicUrl.replace(/\/$/, '');
  return { url: `${base}/${key}` };
}
```

```txt
검증 순서 — 가벼운 것부터:
  파일 존재 → MIME 타입 → 파일 크기 → 업로드
  어차피 거부될 요청은 S3 전송 전에 빨리 차단

key = `visits/${userId}/${randomUUID()}.${ext}`:
  visits/       용도별 폴더 (다른 종류 파일과 섞이지 않게)
  ${userId}/    사용자별 폴더 → 같은 유저 파일끼리 묶임
  randomUUID()  파일명 중복 절대 방지 (Date.now() 보다 안전 — 같은 ms 충돌 없음)
  .${ext}       MIME 타입 기준으로 직접 결정

  ⚠️ file.originalname을 그대로 쓰지 않는 이유:
    사용자가 올린 파일명에 이상한 문자/경로가 포함될 수 있음
    → 서버에서 안전한 이름을 직접 만들어서 사용

publicUrl.replace(/\/$/, ''):
  환경변수 끝에 '/'가 실수로 붙어도 안전하게 제거
  → 'https://photos.com/' + '/visits/...' → '//' 중복 방지
```

---

# Controller 연결 ⭐️⭐️⭐️

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

```txt
흐름:
  ① FileInterceptor가 multipart/form-data를 가로채서 file 객체로 변환
  ② @UploadedFile()로 컨트롤러가 file 받음
  ③ Service의 uploadVisitPhoto()로 넘김 (S3 업로드는 Service 책임)

크기 제한이 두 곳에 있는 이유:
  FileInterceptor limits.fileSize  → Multer 단계에서 먼저 차단 (더 빠름)
  Service 안의 MAX_BYTES 체크      → 한 번 더 확실하게 검증 (이중 안전장치)
  둘 다 5MB로 맞춰 일관성 유지

FileInterceptor / memoryStorage / limits 상세 → [[NestJS_FileUpload_r]]
```

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
  업로드 시 반환한 key 자체를 DB에 별도 컬럼으로 저장해두면
  삭제 시 publicUrl에서 역산할 필요 없이 바로 사용 가능
  예: VisitRecord.photoKey = 'visits/42/abc-uuid.jpg'
```

---

# 전체 흐름 ⭐️⭐️⭐️

```txt
FileInterceptor(Multer)
  → Controller @UploadedFile()
  → S3StorageService.uploadVisitPhoto()
      ① 검증 (파일 존재 → MIME → 크기)
      ② key 생성 (visits/${userId}/${uuid}.ext)
      ③ PutObjectCommand → R2/S3 업로드
      ④ { url: publicUrl + key } 반환
  → DB에 url 저장

환경변수:
  S3_ENDPOINT / S3_BUCKET / S3_ACCESS_KEY_ID / S3_SECRET_ACCESS_KEY / S3_PUBLIC_URL

S3StorageService 핵심 패턴:
  client: S3Client | null  지연 초기화
  getConfig()              환경변수 읽기 + 검증
  getClient()              없으면 생성, 있으면 재사용
```
