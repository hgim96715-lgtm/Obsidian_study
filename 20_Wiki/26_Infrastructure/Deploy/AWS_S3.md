---
aliases:
  - S3
  - AWS S3
  - Presigned URL
tags:
  - AWS
  - Deploy
related:
  - "[[00_Deploy_HomePage]]"
  - "[[AWS_Concept]]"
  - "[[AWS_IAM]]"
---

# AWS_S3 — S3 파일 업로드 & Presigned URL

```txt
S3 = Simple Storage Service
파일 / 이미지 / 영상을 클라우드에 저장하는 객체 스토리지

Presigned URL = 서버가 서명한 임시 URL
클라이언트가 서버를 거치지 않고 S3 에 직접 업로드/다운로드 가능
```

---

---

# Presigned URL 흐름 ⭐️

```txt
일반 업로드:
  클라이언트 → 서버 → S3
  파일이 서버를 거침 → 서버 부하 / 대역폭 낭비

Presigned URL 업로드:
  클라이언트 → 서버(URL 요청) → 서버가 서명된 URL 반환
                    ↓
  클라이언트 → S3 직접 업로드 (서명된 URL 사용)
  → 서버 부하 없음 / 대용량 파일에 적합

PUT Presigned URL  → 업로드 (클라이언트가 S3 에 파일 올림)
GET Presigned URL  → 다운로드 (클라이언트가 S3 에서 파일 받음)
```

---

---

# IAM 사용자 생성 ⭐️

```txt
S3 에 접근할 IAM 사용자를 만들어서 액세스 키 발급
→ NestJS 가 이 키로 S3 에 접근

IAM → 사용자 → 사용자 생성
  사용자 이름: NextJS (또는 원하는 이름)
  직접 정책 연결:
    AmazonS3FullAccess  ← S3 전체 권한

→ 생성 후 액세스 키 만들기
  사용 사례: AWS 외부에서 실행되는 애플리케이션
  → Access Key ID / Secret Access Key 발급
  → 반드시 저장 (이후 다시 볼 수 없음)
```

---

---

# S3 버킷 생성

```txt
S3 → 버킷 만들기

  버킷 이름:  nestjs-bucket-gong (전 세계 고유해야 함)
  리전:       ap-northeast-2 (서울)

  ACL 활성화됨  ← 객체 단위 접근 제어 필요시
```

---

---

# 설치 & .env 설정

```bash
# AWS SDK v3 — S3 클라이언트
pnpm add @aws-sdk/client-s3

# Presigned URL 생성
pnpm add @aws-sdk/s3-request-presigner
```

```properties
# .env
AWS_REGION=ap-northeast-2
AWS_S3_BUCKET=nestjs-bucket-gong(버킷생성이름)

AWS_ACCESS_KEY_ID=발급받은_액세스_키_ID
AWS_SECRET_ACCESS_KEY=발급받은_시크릿_키
```

---

---

# 폴더 구조

```txt
src/
└── aws/
    ├── aws.module.ts
    ├── aws.controller.ts
    ├── s3.service.ts
    └── dto/
        └── presigned-url.dto.ts
```

---

---

# 코드 ⭐️

### DTO

```typescript
// src/aws/dto/presigned-url.dto.ts
import { IsOptional, IsString } from 'class-validator';

export class PresignedUrlDto {
  @IsOptional()
  @IsString()
  key?: string;

  @IsOptional()
  @IsString()
  contentType?: string;
}

export class DownloadPresignedUrlDto {
  @IsString()
  key!: string;
}
```

### S3Service

```typescript
// src/aws/s3.service.ts
import { Injectable } from '@nestjs/common';
import { ConfigService } from '@nestjs/config';
import { GetObjectCommand, PutObjectCommand, S3Client } from '@aws-sdk/client-s3';
import { getSignedUrl } from '@aws-sdk/s3-request-presigner';
import { randomUUID } from 'crypto';
import { envVariableKeys } from 'src/common/const/env.const';
import { DownloadPresignedUrlDto, PresignedUrlDto } from './dto/presigned-url.dto';

@Injectable()
export class S3Service {
  public readonly client: S3Client;
  private readonly bucket: string;

  constructor(private readonly configService: ConfigService) {
    const region = this.configService.get<string>(envVariableKeys.awsRegion);
    const bucket = this.configService.get<string>(envVariableKeys.awsS3Bucket);

    if (!region) throw new Error(`${envVariableKeys.awsRegion} is required`);
    if (!bucket) throw new Error(`${envVariableKeys.awsS3Bucket} is required`);

    this.client = new S3Client({ region });
    this.bucket = bucket;
  }

  async createPutPresignedUrl(dto: PresignedUrlDto) {
    const key = `temp/${randomUUID()}`;          // temp/ 폴더에 저장
    const contentType = dto.contentType;

    const command = new PutObjectCommand({
      Bucket: this.bucket,
      Key: key,
      ...(contentType ? { ContentType: contentType } : {}),
    });

    const url = await getSignedUrl(this.client, command, { expiresIn: 60 * 5 });
    // expiresIn: 300초(5분) 후 만료

    return { method: 'PUT', url, key, bucket: this.bucket };
  }

  async createGetPresignedUrl(dto: DownloadPresignedUrlDto) {
    const command = new GetObjectCommand({
      Bucket: this.bucket,
      Key: dto.key,
    });

    const url = await getSignedUrl(this.client, command, { expiresIn: 60 * 5 });

    return { method: 'GET', url, key: dto.key, bucket: this.bucket };
  }
}
```

### Controller

```typescript
// src/aws/aws.controller.ts
import { Body, Controller, Get, Post, Query } from '@nestjs/common';
import { DownloadPresignedUrlDto, PresignedUrlDto } from './dto/presigned-url.dto';
import { S3Service } from './s3.service';

@Controller('presigned-url')
export class AwsController {
  constructor(private readonly s3Service: S3Service) {}

  @Get()
  async getPresignedUrl(@Query() dto: PresignedUrlDto) {
    return await this.s3Service.createPutPresignedUrl(dto);
  }

  @Post()
  async postPresignedUrl(@Body() dto: PresignedUrlDto) {
    return await this.s3Service.createPutPresignedUrl(dto);
  }

  @Get('download')
  async getPresignedDownloadUrl(@Query() dto: DownloadPresignedUrlDto) {
    return await this.s3Service.createGetPresignedUrl(dto);
  }
}
```

---

---

# Postman 으로 테스트 ⭐️

### 업로드 테스트

```txt
1. GET /presigned-url
   → url / key 반환

2. 반환된 url 복사
   → Postman 에서 새 요청 생성
   → Method: PUT  ← 반드시 PUT
   → URL: 복사한 presigned url 붙여넣기
   → Body → binary → 파일 선택

3. Send → 200 OK 면 업로드 성공
   → S3 버킷 temp/ 폴더에 파일 생성됨
```

### 다운로드 테스트

```txt
1. GET /presigned-url/download?key=temp/{uuid}
   → url 반환

2. 반환된 url 브라우저에 붙여넣기
   → 파일 다운로드 / 열기 가능
```

---

---

# ⚠️ Access Denied 에러

```xml
<Error>
  <Code>AccessDenied</Code>
  <Message>Access Denied</Message>
</Error>
```

```txt
원인:
  S3 버킷이 비공개(Private) 설정
  URL 을 직접 브라우저에 입력하면 접근 거부

이건 정상 동작:
  Presigned URL 이 아닌 일반 URL 로 접근하면 차단됨
  → 비공개 버킷이 맞음 (보안상 올바른 설정)

해결:
  파일 접근은 반드시 GET Presigned URL 을 통해서만
  GET /presigned-url/download?key=... 로 url 받아서 접근
```

---

---

# 핵심 흐름 정리

```txt
IAM → 사용자 생성 → S3FullAccess 정책 → 액세스 키 발급
    ↓
S3 → 버킷 생성 (nestjs-bucket-gong)
    ↓
.env 에 AWS_REGION / AWS_S3_BUCKET / 액세스 키 입력
    ↓
pnpm add @aws-sdk/client-s3 @aws-sdk/s3-request-presigner
    ↓
src/aws/ 폴더 생성 → S3Service / Controller / Module
    ↓
GET /presigned-url → PUT url / key 반환
    ↓
Postman PUT 요청 → binary 파일 → S3 temp/ 업로드
    ↓
GET /presigned-url/download?key=... → 다운로드 url 반환
```