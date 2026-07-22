---
aliases:
  - Email
  - Nodemailer
  - SMTP
  - Render
  - Amazon SES
tags:
  - NestJS
related:
  - "[[00_NestJS_Ecosystem_HomePage]]"
  - "[[NestJS_Module]]"
  - "[[NestJS_Env_Config]]"
  - "[[Web_Email]]"
  - "[[NestJS_Logger]]"
---
# NestJS_Email — 서버 이메일 전송

> [!info] 
> 서버 이메일 전송 3가지: **Resend**(API, 빠른 시작) · **Nodemailer**(SMTP, 어떤 서버든 연결) · **Amazon SES**(대량 발송, 월 62,000통 무료). 
> 인증 메일·비밀번호 재설정은 Resend로 시작하고, 대량 발송이 필요해지면 SES로 전환.

---
# 흐름도

```mermaid
flowchart LR
  subgraph CLIENT["클라이언트"]
    FORM["폼 제출<br/>문의 · 인증 · 재설정"]
  end

  subgraph NEST["NestJS"]
    CTL["Controller<br/>DTO 검증"]
    SVC["Domain Service"]
    MAIL["Mail / Email Service"]
    CTL --> SVC --> MAIL
  end

  subgraph PROVIDER["발송 통로 · 하나만"]
    SMTP["Nodemailer<br/>SMTP"]
    RESEND["Resend<br/>API"]
    SES["Amazon SES<br/>API / SMTP"]
  end

  INBOX["수신함<br/>유저 또는 운영자"]

  FORM -->|"POST /…"|" CTL
  MAIL --> SMTP --> INBOX
  MAIL --> RESEND --> INBOX
  MAIL --> SES --> INBOX

  NOTE["mailto ❌<br/>유저 메일 앱이 보냄 · Nest 안 탐"]
```

> **핵심:** Web/앱은 폼만 · Nest가 발송 · 통로(SMTP / Resend / SES)는 **한 번에 하나**. `mailto`는 Nest 경로가 아님.

---

# 방식 비교

| |Nodemailer (SMTP)|Resend SDK|Amazon SES|
|---|---|---|---|
|방식|SMTP 직접 연결|REST API|SMTP 또는 REST API|
|설정|HOST / PORT / 인증 필요|API Key만|도메인 인증 + SMTP 자격증명|
|템플릿|직접 HTML 문자열|React 컴포넌트 가능|직접 HTML 또는 SES 템플릿|
|무료 플랜|Gmail 하루 500통|월 3,000통|월 62,000통 (EC2 발송 시)|
|대량 발송|❌ Gmail 한도 있음|유료 플랜 필요|✅ 대량 가능|
|도메인 평판 관리|❌|제한적|✅ 반송/스팸 자동 처리|
|사용 사례|기존 SMTP 서버 연동|빠르게 시작|운영 대량 발송|

```txt
선택 기준:
  개발/소규모    → Resend (설정 최소, 빠른 시작)
  Gmail/iCloud  → Nodemailer + SMTP (기존 계정 활용)
  운영/대량      → Amazon SES (월 62,000통 무료, 도메인 평판 관리)

  Nodemailer은 SMTP 클라이언트 라이브러리 — 어떤 SMTP 서버든 연결 가능
  Amazon SES SMTP 엔드포인트에 Nodemailer를 붙여 쓰는 것도 가능
  → 대량 발송이 필요해지면 Gmail → Amazon SES 로 HOST만 바꾸면 됨
```

---

# Resend — API 방식 (권장) ⭐️⭐️⭐️⭐️

```bash
pnpm add resend
```

## 환경 변수

```bash
# .env
RESEND_API_KEY=re_xxxxxxxxxxxx
MAIL_FROM=no-reply@example.com
```

## EmailService

```typescript
// email/email.service.ts
import { Injectable } from '@nestjs/common';
import { ConfigService } from '@nestjs/config';
import { Resend } from 'resend';

@Injectable()
export class EmailService {
  private readonly resend: Resend;
  private readonly from: string;

  constructor(private readonly configService: ConfigService) {
    this.resend = new Resend(
      this.configService.getOrThrow<string>('RESEND_API_KEY'),
    );
    this.from = this.configService.getOrThrow<string>('MAIL_FROM');
  }

  async sendVerification(to: string, token: string) {
    const link = `${this.configService.getOrThrow('FRONTEND_URL')}/verify?token=${token}`;
    await this.resend.emails.send({
      from:    this.from,
      to,
      subject: '이메일 인증',
      html:    `<p>아래 링크를 클릭해 이메일을 인증해주세요.</p>
                <a href="${link}">${link}</a>
                <p>링크는 24시간 후 만료됩니다.</p>`,
    });
  }

  async sendPasswordReset(to: string, token: string) {
    const link = `${this.configService.getOrThrow('FRONTEND_URL')}/reset-password?token=${token}`;
    await this.resend.emails.send({
      from:    this.from,
      to,
      subject: '비밀번호 재설정',
      html:    `<p>비밀번호 재설정 링크입니다.</p>
                <a href="${link}">${link}</a>
                <p>요청하지 않았다면 무시하세요. 링크는 1시간 후 만료됩니다.</p>`,
    });
  }

  // 범용 전송 메서드
  async send(options: {
    to:      string | string[];
    subject: string;
    html:    string;
    cc?:     string | string[];
  }) {
    return this.resend.emails.send({
      from: this.from,
      ...options,
    });
  }
}
```

## EmailModule

```typescript
// email/email.module.ts
@Module({
  providers: [EmailService],
  exports:   [EmailService],  // 다른 모듈에서 주입 가능하게
})
export class EmailModule {}

// app.module.ts에 imports 추가
@Module({
  imports: [EmailModule, ...],
})
export class AppModule {}
```

## 다른 Service에서 사용

```typescript
@Injectable()
export class AuthService {
  constructor(
    private readonly emailService: EmailService,
  ) {}

  async register(email: string, password: string) {
    // 계정 생성 ...
    const token = generateVerificationToken();
    await this.emailService.sendVerification(email, token);
  }
}
```

---

# SMTP 설정 — HOST / PORT / 인증 ⭐️⭐️⭐️⭐️

```txt
SMTP(Simple Mail Transfer Protocol) = 이메일을 전송하는 표준 프로토콜
서버가 직접 이메일 서버(Gmail, iCloud, SES 등)에 연결해서 메일을 보냄
```

## PORT 선택 기준

|포트|방식|언제|
|---|---|---|
|`465`|SSL/TLS (즉시 암호화)|`secure: true` — 연결부터 암호화|
|`587`|STARTTLS (평문 → 암호화 업그레이드)|`secure: false` — 가장 범용적, **일반적으로 이걸 씀**|
|`25`|평문 (암호화 없음)|서버 간 전송 전용 — 개인 이메일 클라이언트에선 차단됨|

```txt
587을 기본으로 쓰는 이유:
  가장 많은 SMTP 서버가 지원
  방화벽에서 막힐 가능성이 가장 낮음
  STARTTLS로 연결 후 암호화되므로 보안 충분

secure 옵션:
  secure: true  → 포트 465 (SSL — 처음부터 암호화)
  secure: false → 포트 587 (STARTTLS — 연결 후 암호화 업그레이드)
  둘 다 암호화됨, 포트와 방식만 다름
```

---

## 제공자별 SMTP 설정

### Gmail

```bash
SMTP_HOST=smtp.gmail.com
SMTP_PORT=587
SMTP_USER=your@gmail.com
SMTP_PASS=xxxx xxxx xxxx xxxx   # 앱 비밀번호 (16자리, 공백 포함)
MAIL_FROM="서비스명 <your@gmail.com>"
```

```txt
Gmail 앱 비밀번호 발급:
  Google 계정 → 보안 → 2단계 인증 활성화 필수
  → 앱 비밀번호 → 앱 선택: 메일, 기기: 기타 → 생성
  → 16자리 코드 발급 (SMTP_PASS에 사용)

⚠️ Gmail 한도:
  하루 500통 (개인 계정) / 2,000통 (Google Workspace)
  대량 발송에는 부적합 → Resend / Amazon SES 사용
```

---

### iCloud

```bash
SMTP_HOST=smtp.mail.me.com
SMTP_PORT=587
SMTP_USER=your@icloud.com
SMTP_PASS=xxxx-xxxx-xxxx-xxxx   # 앱 전용 암호
MAIL_FROM="서비스명 <your@icloud.com>"
```

```txt
iCloud 앱 전용 암호 발급:
  appleid.apple.com → 로그인 → 앱 전용 암호 → 생성
  → 16자리 코드 (4자리씩 - 로 구분)

⚠️ iCloud 제한:
  개인 계정 전용 — 대량 발송 금지
  일 1,000통 미만 권장
  Apple이 스팸으로 판단하면 계정 차단 가능
```

---

### Amazon SES SMTP ⭐️⭐️⭐️⭐️

```bash
SMTP_HOST=email-smtp.ap-northeast-2.amazonaws.com  # 서울 리전
SMTP_PORT=587
SMTP_USER=AKIAIOSFODNN7EXAMPLE          # SES SMTP 자격 증명 (IAM 키 아님!)
SMTP_PASS=BmNMjXExample/SMTP_CREDENTIAL  # SES SMTP 비밀번호
MAIL_FROM="서비스명 <no-reply@your-domain.com>"   # SES에 인증된 도메인 필요
```

```txt
Amazon SES SMTP 설정 순서:
  1. AWS SES 콘솔 → 도메인 인증 (Route53 또는 DNS TXT/DKIM 레코드 추가)
  2. SES 콘솔 → SMTP 설정 → SMTP 자격 증명 생성
     (IAM 사용자 자동 생성 + SMTP 전용 비밀번호 발급)
  3. 발신 주소도 SES에서 인증 필요 (이메일 또는 도메인 단위)
  4. 샌드박스 해제 — 처음엔 인증된 주소에만 발송 가능
     → 프로덕션 접근 요청 후 승인되면 외부 주소로 발송 가능

리전별 SMTP 호스트:
  서울(ap-northeast-2)  email-smtp.ap-northeast-2.amazonaws.com
  도쿄(ap-northeast-1)  email-smtp.ap-northeast-1.amazonaws.com
  버지니아(us-east-1)   email-smtp.us-east-1.amazonaws.com

SES 장점:
  월 62,000통 무료 (EC2에서 보낼 때)
  대량 발송 가능, 반송/스팸 처리 자동화
  도메인 평판 관리 도구 제공
```

---

### Outlook / Office 365

```bash
SMTP_HOST=smtp.office365.com
SMTP_PORT=587
SMTP_USER=your@outlook.com
SMTP_PASS=your-password
MAIL_FROM="서비스명 <your@outlook.com>"
```

---

## .env 전체 예시

```bash
# 어떤 SMTP를 쓸지 하나 고름 (주석으로 선택지 명시)
MAIL_PROVIDER=gmail   # gmail | icloud | ses

# 선택한 프로바이더에 맞는 값만 채움
SMTP_HOST=smtp.gmail.com
SMTP_PORT=587
SMTP_USER=your@gmail.com
SMTP_PASS=xxxx xxxx xxxx xxxx

# 발신자 표시 이름
MAIL_FROM=Music Community <your@gmail.com>

# 문의 수신 이메일 — FQDN (Fully Qualified Domain Name) 주소
MAIL_SUPPORT_TO=your@gmail.com
```

```txt
"둘 다 넣는 게 아니다":
  SMTP는 한 번에 하나의 서버에만 연결함
  Gmail 설정을 쓰면서 iCloud 설정을 같이 넣어도 아무 의미 없음
  → SMTP_HOST 하나, SMTP_USER 하나, SMTP_PASS 하나만 채우면 됨
  → 어떤 서버에 연결할지는 SMTP_HOST 값이 결정

MAIL_PROVIDER 주석의 역할:
  코드에서 읽어서 분기하는 값이 아님 (현재는)
  "지금 이 .env는 gmail을 쓰고 있다"는 것을 팀원/미래의 자신에게 알려주는 가이드
  iCloud로 바꾸려면 MAIL_PROVIDER=icloud로 바꾸고
  HOST/USER/PASS 값을 iCloud 값으로 교체하면 됨

FQDN (Fully Qualified Domain Name):
  이메일에서 FQDN = user@domain.com 처럼 완전한 이메일 주소
  'your@gmail.com' ✅  'your' ❌ (도메인 없으면 메일 서버가 못 찾음)
  MAIL_SUPPORT_TO는 문의가 실제로 도착할 완전한 이메일 주소여야 함

iCloud로 전환하려면:
  MAIL_PROVIDER=icloud
  SMTP_HOST=smtp.mail.me.com
  SMTP_PORT=587
  SMTP_USER=your@icloud.com
  SMTP_PASS=xxxx-xxxx-xxxx-xxxx   # 앱 전용 암호

Amazon SES로 전환하려면:
  MAIL_PROVIDER=ses
  SMTP_HOST=email-smtp.ap-northeast-2.amazonaws.com
  SMTP_PORT=587
  SMTP_USER=AKIAIOSFODNN7EXAMPLE   # SES SMTP 자격증명
  SMTP_PASS=BmNMjX...              # SES SMTP 비밀번호
  MAIL_FROM=Music Community <no-reply@your-verified-domain.com>
```

---

# Nodemailer — SMTP 방식 ⭐️⭐️⭐️

```bash
pnpm add nodemailer
pnpm add -D @types/nodemailer
```

## MailService 전체 구현

```typescript
// mail/mail.service.ts
import { Injectable, Logger } from '@nestjs/common';
import { ConfigService } from '@nestjs/config';
import * as nodemailer from 'nodemailer';
import type { Transporter } from 'nodemailer';
import { EnvKeys } from '../config/env.keys';

export type MailProvider = 'gmail' | 'icloud';

export type SupportContactMail = {
  fromEmail: string;
  nickname?: string;
  subject:   string;
  body:      string;
};

// 프로바이더별 기본값 — SMTP_HOST/PORT를 .env에 안 넣으면 이 값으로 폴백
const PROVIDER_DEFAULTS: Record<MailProvider, { host: string; port: number }> = {
  gmail:  { host: 'smtp.gmail.com',   port: 587 },
  icloud: { host: 'smtp.mail.me.com', port: 587 },
};

@Injectable()
export class MailService {
  private readonly logger     = new Logger(MailService.name);
  private readonly transporter: Transporter;
  private readonly mailFrom:   string;
  private readonly supportTo:  string;
  private readonly provider:   MailProvider;

  constructor(private readonly configService: ConfigService) {
    // 1. 프로바이더 결정 (유효성 검사 포함)
    this.provider = this.resolveProvider();
    const defaults = PROVIDER_DEFAULTS[this.provider];

    // 2. HOST — env에 있으면 그것 사용, 없으면 프로바이더 기본값
    const host =
      this.configService.get<string>(EnvKeys.SMTP_HOST)?.trim() || defaults.host;

    // 3. PORT — env에 있으면 파싱 + 검증, 없으면 프로바이더 기본값
    const port = this.resolvePort(
      this.configService.get<string | number>(EnvKeys.SMTP_PORT),
      defaults.port,
    );

    const user = this.configService.getOrThrow<string>(EnvKeys.SMTP_USER);
    const pass = this.configService.getOrThrow<string>(EnvKeys.SMTP_PASS);

    this.mailFrom  = this.configService.getOrThrow<string>(EnvKeys.MAIL_FROM);
    this.supportTo = this.configService.getOrThrow<string>(EnvKeys.MAIL_SUPPORT_TO);

    this.transporter = nodemailer.createTransport({
      host,
      port,
      secure: port === 465,
      auth: { user, pass },
    });

    // 기동 시 어떤 SMTP로 연결됐는지 확인용 로그
    this.logger.log(`mail ready · provider=${this.provider} · host=${host}`);
  }

  async sendSupportContact(input: SupportContactMail): Promise<void> {
    const nickname = input.nickname?.trim() || '익명';
    const subject  = `[문의] ${input.subject.trim()}`;
    const text     = [
      `닉네임: ${nickname}`,
      `회신 이메일: ${input.fromEmail.trim()}`,
      '',
      input.body.trim(),
    ].join('\n');

    try {
      await this.transporter.sendMail({
        from:    this.mailFrom,
        to:      this.supportTo,
        replyTo: input.fromEmail.trim(),
        subject,
        text,
      });
    } catch (error) {
      this.logger.error('support contact mail failed', error);
      throw error;
    }
  }

  /** MAIL_PROVIDER 유효성 검사 — 모르는 값이면 기동 시 즉시 에러 */
  private resolveProvider(): MailProvider {
    const raw = this.configService
      .getOrThrow<string>(EnvKeys.MAIL_PROVIDER)
      .trim()
      .toLowerCase();
    if (raw === 'gmail' || raw === 'icloud') return raw;
    throw new Error(`MAIL_PROVIDER must be "gmail" or "icloud" (got: ${raw})`);
  }

  /** SMTP_PORT 파싱 + 검증 — 없거나 빈 문자열이면 fallback */
  private resolvePort(raw: string | number | undefined, fallback: number): number {
    if (raw === undefined || raw === '') return fallback;
    const port = typeof raw === 'number' ? raw : Number(raw);
    if (!Number.isFinite(port) || port <= 0) {
      throw new Error(`SMTP_PORT is invalid: ${String(raw)}`);
    }
    return port;
  }
}
```

## 주요 패턴 설명

```txt
PROVIDER_DEFAULTS (Record 룩업 테이블):
  Record<MailProvider, { host: string; port: number }>
  새 프로바이더 추가 시 MailProvider 유니온 + PROVIDER_DEFAULTS에 추가
  → TypeScript가 누락 항목을 컴파일 에러로 잡아줌

nodemailer.createTransport(options):
  SMTP 서버와의 "연결 설정을 담은 Transporter 객체"를 반환
  이 시점에 실제 서버에 연결하는 게 아님 — 설정만 등록
  transporter.sendMail()을 호출할 때 비로소 SMTP 서버에 연결 → 메일 전송 → 연결 해제

  options 주요 필드:
    host    SMTP 서버 주소 (smtp.gmail.com 등)
    port    포트 번호 (587 or 465)
    secure  true  → SSL 연결 (포트 465)
            false → STARTTLS 연결 (포트 587, 평문 → 암호화 업그레이드)
    auth    { user: 이메일, pass: 비밀번호/앱비밀번호 }

  반환값: Transporter 객체
    transporter.sendMail(mailOptions) → 실제 메일 전송
    transporter.verify()             → SMTP 연결 테스트 (선택)

  constructor에서 한 번만 만드는 이유:
    매번 createTransport()를 호출하면 설정 파싱 비용이 반복됨
    NestJS Injectable에서 private readonly로 만들어두면
    서비스 인스턴스당 하나만 유지 → 재사용

폴백 패턴:
  config.get()?.trim() || defaults.host
  SMTP_HOST를 .env에 안 써도 프로바이더 기본값으로 동작
  → SMTP_USER, SMTP_PASS, MAIL_PROVIDER 세 가지만 필수

resolveProvider() — 기동 시 즉시 에러:
  모르는 MAIL_PROVIDER 값이면 서버 시작할 때 바로 에러
  런타임 중간에 에러나는 것보다 훨씬 빠르게 문제 발견

resolvePort() — Number.isFinite:
  Number.isNaN만으로는 Infinity가 통과할 수 있음
  isFinite가 더 안전 (NaN + Infinity 둘 다 걸러냄)

기동 로그:
  this.logger.log('mail ready · provider=gmail · host=smtp.gmail.com')
  배포 후 로그에서 설정이 제대로 읽혔는지 즉시 확인 가능

replyTo: fromEmail:
  from = 서비스 주소 (no-reply@example.com)
  replyTo = 사용자 이메일 → 운영자가 답장하면 사용자에게 감
```

```txt
import type { Transporter } from 'nodemailer':
  Transporter는 타입으로만 쓰이고 런타임 값이 아님
  import type → JS 출력에서 완전히 사라짐
  * as nodemailer로 이미 값을 가져왔으므로 타입만 따로 import
  → [[TS_ImportType]] 참고

private readonly logger = new Logger(MailService.name):
  NestJS 내장 Logger (별도 설치 불필요)
  MailService.name → 클래스 이름 문자열 'MailService'
  → 로그 출력: [MailService] support contact mail failed +Error...
  logger.log()   → 일반 로그
  logger.error() → 에러 로그 (스택 트레이스 포함)
  logger.warn()  → 경고
  → [[NestJS_Logger]] 참고

secure: port === 465:
  포트를 읽어서 SSL 여부를 자동 결정
  환경변수 SMTP_PORT만 바꾸면 설정이 자동으로 따라옴
  Gmail/iCloud: 587 → false / SSL 전용: 465 → true

replyTo: input.fromEmail:
  from은 서비스 주소 (no-reply@example.com)
  replyTo는 사용자 이메일 → 운영자가 답장하면 사용자에게 감
  문의 폼 패턴의 핵심 — "사용자가 보냈지만 서비스 이름으로 수신"

text vs html:
  text  → 일반 텍스트 (문의 폼, 간단한 알림)
  html  → HTML 이메일 (인증 버튼, 디자인 있는 메일)
  둘 다 넣으면 클라이언트가 선택해서 표시
```

## MailModule

```typescript
// mail/mail.module.ts
@Module({
  providers: [MailService],
  exports:   [MailService],
})
export class MailModule {}
```

## 다른 Service에서 주입

```typescript
@Injectable()
export class SupportService {
  constructor(private readonly mailService: MailService) {}

  async createContact(dto: CreateSupportContactDto): Promise<{ ok: true }> {
    try {
      await this.mailService.sendSupportContact({
        fromEmail: dto.fromEmail,
        nickname:  dto.nickname,
        subject:   dto.subject,
        body:      dto.body,
      });
      return { ok: true };
    } catch {
      // 외부 서비스(SMTP) 실패 → 503으로 변환
      throw new ServiceUnavailableException(
        '문의 메일 전송에 실패했습니다. 잠시 후 다시 시도해 주세요.',
      );
    }
  }
}
```


## DTO + Controller 전체 패턴 ⭐️⭐️⭐️⭐️

```typescript
// support/dto/create-support-contact.dto.ts
export class CreateSupportContactDto {
  @IsEmail()
  fromEmail: string;

  @IsOptional()
  @IsString()
  @MaxLength(40)
  nickname?: string;

  @IsString()
  @MinLength(1)
  @MaxLength(120)
  subject: string;

  @IsString()
  @MinLength(1)
  @MaxLength(2000)
  body: string;
}
```

```typescript
// support/support.controller.ts
@ApiTags('support')
@Controller('support')
export class SupportController {
  constructor(private readonly supportService: SupportService) {}

  @ApiOperation({ summary: '고객지원 문의 메일 전송 (비로그인 OK)' })
  @Post('contact')
  @HttpCode(HttpStatus.OK)   // POST지만 리소스 생성이 아니라 액션 → 200
  async createContact(@Body() dto: CreateSupportContactDto) {
    return this.supportService.createContact(dto);
  }
}
```

## 패턴 설명

```txt
ServiceUnavailableException (503):
  SMTP 서버 다운, 네트워크 오류 등 "외부 서비스 장애"를 나타내는 HTTP 상태
  우리 서버 코드는 맞는데 외부 의존성이 실패한 경우
  클라이언트가 "서버 문제가 아니라 외부 서비스 문제"임을 알 수 있음

  에러 코드 선택 기준:
    400 BadRequest        클라이언트가 잘못된 입력을 보낸 경우
    401 Unauthorized      인증 필요
    403 Forbidden         권한 없음
    404 NotFound          리소스 없음
    409 Conflict          중복 등 충돌
    500 InternalServerError  예상치 못한 서버 에러
    503 ServiceUnavailable   외부 서비스 장애 ← 이메일 전송 실패에 적합

catch 에서 re-throw:
  MailService.sendSupportContact()가 던진 에러를 잡아서
  클라이언트에게 의미 있는 메시지로 바꿔서 다시 던짐
  (SMTP 내부 에러 메시지가 그대로 클라이언트에 노출되는 것 방지)

@HttpCode(HttpStatus.OK):
  NestJS의 POST는 기본적으로 201 Created를 반환
  문의 전송은 "리소스 생성"이 아니라 "액션 실행" → 200이 더 적절
  @HttpCode(HttpStatus.OK) 또는 @HttpCode(200) 으로 명시적으로 200 반환

return { ok: true }:
  "성공했다"는 최소한의 응답
  메일 전송은 응답에 보낼 생성된 리소스가 없음
  클라이언트는 ok: true 를 보고 성공으로 처리

Promise<{ ok: true }>:
  반환 타입을 { ok: true }(리터럴 타입)로 좁혀서
  성공 시 항상 ok가 true임을 컴파일 타임에 보장
  { ok: boolean } 이 아니라 { ok: true } — 성공 경로에서 false가 올 수 없음
```
---

# 이메일 템플릿 관리 ⭐️⭐️⭐️

## 인라인 HTML (간단)

```typescript
function verificationTemplate(link: string): string {
  return `
    <!DOCTYPE html>
    <html>
    <body style="font-family: sans-serif; padding: 20px;">
      <h2>이메일 인증</h2>
      <p>아래 버튼을 클릭해 이메일을 인증해주세요.</p>
      <a href="${link}"
         style="display:inline-block; padding:12px 24px;
                background:#0070f3; color:#fff;
                border-radius:6px; text-decoration:none;">
        이메일 인증하기
      </a>
      <p style="color:#666; font-size:12px;">
        링크는 24시간 후 만료됩니다.
        요청하지 않았다면 무시하세요.
      </p>
    </body>
    </html>
  `;
}
```

```txt
이메일 HTML 주의사항:
  CSS 클래스 안 됨 — 이메일 클라이언트가 <style> 태그 지원 안 하는 경우 많음
  인라인 style="" 만 사용
  외부 폰트 불가 — sans-serif 같은 시스템 폰트만 안전
  이미지는 절대 URL (http://... or https://...)

React 이메일 템플릿 (Resend + react-email):
  pnpm add @react-email/components react-email
  → JSX로 이메일 컴포넌트 작성 가능
  → Resend가 자동으로 인라인 스타일로 변환
```

---

# 재시도 / 에러 처리 ⭐️⭐️

```typescript
async sendWithRetry(
  options: EmailOptions,
  maxRetries = 3,
): Promise<void> {
  for (let i = 0; i < maxRetries; i++) {
    try {
      await this.send(options);
      return;
    } catch (err) {
      if (i === maxRetries - 1) {
        // 마지막 시도에서도 실패 → 로깅 후 에러
        this.logger.error('이메일 전송 실패', { to: options.to, err });
        throw err;
      }
      // 재시도 전 잠깐 대기 (지수 백오프)
      await new Promise(r => setTimeout(r, 1000 * 2 ** i));
    }
  }
}
```

```txt
이메일 전송 실패 처리:
  네트워크 오류, API 한도 초과 등으로 전송이 실패할 수 있음
  회원가입 인증 메일처럼 중요한 메일은 재시도 로직 필요
  실패 시 사용자에게 "메일 재전송" 버튼 제공

비동기 처리:
  이메일 전송이 느릴 때 요청을 블로킹하지 않으려면
  await 없이 void로 fire-and-forget 처리도 가능
  → 단, 에러를 잡을 방법이 없으므로 중요도에 따라 결정
```

---

# 한눈에

```txt
Resend (권장):
  pnpm add resend
  new Resend(API_KEY)
  resend.emails.send({ from, to, subject, html })
  API Key만 있으면 되고 도메인 등록 필요

Nodemailer (SMTP):
  pnpm add nodemailer @types/nodemailer
  createTransport({ host, port, auth })
  transporter.sendMail({ from, to, subject, html })
  Gmail: 앱 비밀번호 사용, 하루 500통 제한

공통 패턴:
  EmailService → EmailModule → exports: [EmailService]
  → 다른 모듈에서 imports: [EmailModule] 후 주입

HTML 이메일:
  CSS 클래스 안 됨 → 인라인 style="" 만
  이미지는 절대 URL 필수

프론트엔드 이메일 → [[Web_Email]]
```