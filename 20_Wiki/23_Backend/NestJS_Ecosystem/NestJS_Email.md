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
  - "[[Web_Email]]"
  - "[[NestJS_Scheduling]]"
---
# NestJS_Email — 이메일 발송

>[!info]
>이메일 발송 = SMTP 서버를 통해 이메일을 전달. 
>`nodemailer`가 Node.js에서 SMTP 연결을 처리하는 라이브러리. 
>`Transporter`로 SMTP 설정을 한 번 구성하고 `sendMail()`로 발송한다.

---

# 이메일 발송 흐름 ⭐️⭐️⭐️⭐️

```txt
"이메일을 보낸다"는 것은:
  내 앱 → (SMTP) → 메일 서버 → 수신자

SMTP (Simple Mail Transfer Protocol):
  이메일을 전달하는 통신 규약 (프로토콜)
  편지를 우체국에 맡기는 것처럼
  내 앱이 SMTP 서버에 "이 편지를 저 사람에게 전달해줘"라고 요청

SMTP 서버:
  Gmail SMTP — smtp.gmail.com:587
  iCloud SMTP — smtp.mail.me.com:587
  SendGrid, AWS SES 등 전문 이메일 서비스

nodemailer:
  Node.js에서 SMTP 통신을 쉽게 처리해주는 라이브러리
  직접 SMTP 프로토콜을 구현하지 않아도 됨
```

---

# 이메일 구조 — 필드가 뭔지 ⭐️⭐️⭐️⭐️

```typescript
await transporter.sendMail({
  from:    'noreply@myapp.com',   // 발신자 주소 (보내는 사람)
  to:      'user@example.com',    // 수신자 주소 (받는 사람)
  subject: '회원가입을 축하합니다', // 제목 (이메일 목록에서 보이는 것)
  text:    '안녕하세요...',         // 본문 (일반 텍스트)
  html:    '<h1>안녕하세요</h1>',  // 본문 (HTML — text보다 우선)
});
```

```txt
from:    보내는 사람 주소
         실제로 이 주소로 보내진 것처럼 표시됨
         SMTP 인증 계정과 일치해야 함 (일치 안 하면 스팸 처리)

to:      받는 사람 주소
         여러 명: 'a@a.com, b@b.com' 또는 ['a@a.com', 'b@b.com']

subject: 이메일 제목
         받은 편지함 목록에서 보이는 한 줄

text:    일반 텍스트 본문
         HTML을 지원하지 않는 클라이언트를 위한 대체

html:    HTML 형식 본문
         text와 html 둘 다 있으면 html 우선 표시

cc:      참조 (Carbon Copy) — 다른 사람도 받음, 받는 사람에게 보임
bcc:     숨은 참조 — 다른 사람도 받지만, 받는 사람에게 안 보임
```

---

# 설치 ⭐️⭐️

```bash
pnpm add nodemailer
pnpm add -D @types/nodemailer
```

---

# MailService 구현 ⭐️⭐️⭐️⭐️

## 타입 정의

```typescript
// mail.types.ts
export type MailProvider = 'gmail' | 'icloud';

// 지원 이메일 제공업체별 SMTP 설정
const PROVIDER_DEFAULTS: Record<MailProvider, { host: string; port: number }> = {
  gmail:  { host: 'smtp.gmail.com',      port: 587 },
  icloud: { host: 'smtp.mail.me.com',    port: 587 },
};

// 발송할 이메일의 형태
export type SupportContactMail = {
  fromEmail:  string;      // 발신자 이메일
  nickname?:  string;      // 발신자 닉네임 (선택)
  subject:    string;      // 이메일 제목
  body:       string;      // 이메일 본문
};
```

## MailService

```typescript
// mail.service.ts
import * as nodemailer from 'nodemailer';
import type { Transporter } from 'nodemailer';
import { Injectable, Logger } from '@nestjs/common';
import { ConfigService } from '@nestjs/config';

@Injectable()
export class MailService implements OnModuleInit {
  private readonly logger = new Logger(MailService.name);

  private transporter:  Transporter;  // 이메일 발송 객체
  private mailFrom:     string;       // 발신 주소 (환경변수에서)
  private supportTo:    string;       // 운영 이메일 수신 주소
  private provider:     MailProvider; // 'gmail' | 'icloud'

  constructor(private readonly configService: ConfigService) {}

  onModuleInit() {
    // 모듈 초기화 시 SMTP 설정
    this.provider  = this.resolveProvider();
    this.mailFrom  = this.configService.getOrThrow<string>(EnvKeys.MAIL_FROM);
    this.supportTo = this.configService.getOrThrow<string>(EnvKeys.SUPPORT_EMAIL);

    const { host, port } = PROVIDER_DEFAULTS[this.provider];

    this.transporter = nodemailer.createTransport({
      host,
      port,
      secure: false,   // 587 포트는 STARTTLS (false)
      auth: {
        user: this.configService.getOrThrow<string>(EnvKeys.MAIL_USER),  // 이메일 계정
        pass: this.configService.getOrThrow<string>(EnvKeys.MAIL_PASS),  // 앱 비밀번호
      },
    });

    this.logger.log(`메일 서비스 초기화 provider=${this.provider}`);
  }

  // MAIL_PROVIDER 환경변수 유효성 검사
  private resolveProvider(): MailProvider {
    const raw = this.configService
      .getOrThrow<string>(EnvKeys.MAIL_PROVIDER)
      .trim()
      .toLowerCase();
    if (raw === 'gmail' || raw === 'icloud') return raw;
    throw new Error(
      `Mail provider는 "gmail" 또는 "icloud" 중 하나여야 합니다. (제공된 값: ${raw})`,
    );
  }
}
```

```txt
Transporter란:
  nodemailer가 SMTP 연결 설정을 담은 객체
  "이 SMTP 서버(host·port·인증)로 이메일을 보낼 준비가 됐다"
  한 번 만들면 sendMail()을 여러 번 호출 가능

onModuleInit():
  NestJS 모듈이 초기화될 때 한 번 실행
  Transporter를 여기서 만들어두면 이후 sendMail()이 빠름

secure: false (포트 587):
  587 포트는 STARTTLS 방식 — 연결 후 암호화 협상
  465 포트였으면 secure: true (처음부터 암호화)
```

## 발송 메서드

```typescript
// 운영 알림 이메일 (에러, 신고 등 내부 알림)
async sendOpsAlert(subject: string, body: string): Promise<void> {
  try {
    await this.transporter.sendMail({
      from:    this.mailFrom,
      to:      this.supportTo,         // 운영팀 이메일
      subject: `[ops] ${subject.trim()}`,
      text:    body.trim(),
    });
  } catch (error) {
    this.logger.error(
      `ops 메일 전송 실패 · error=${error instanceof Error ? error.message : String(error)}`,
    );
    // 이메일 실패는 앱을 죽이지 않음 — 조용히 에러 기록만
  }
}

// 사용자에게 보내는 이메일 (문의 답변 등)
async sendSupportContact(mail: SupportContactMail): Promise<void> {
  const fromDisplay = mail.nickname
    ? `${mail.nickname} <${mail.fromEmail}>`  // "홍길동 <hong@example.com>"
    : mail.fromEmail;

  await this.transporter.sendMail({
    from:    fromDisplay,
    to:      this.supportTo,
    subject: mail.subject,
    text:    mail.body,
  });
}
```

---

# 환경변수 설정 ⭐️⭐️⭐️

```bash
# .env
MAIL_PROVIDER=gmail         # gmail | icloud
MAIL_FROM=noreply@myapp.com # 발신자 주소
MAIL_USER=myaccount@gmail.com  # SMTP 인증 계정
MAIL_PASS=xxxx xxxx xxxx xxxx  # 앱 비밀번호 (구글 계정 앱 비밀번호)
SUPPORT_EMAIL=support@myapp.com # 운영팀 수신 주소
```

```txt
MAIL_PASS = 앱 비밀번호:
  Gmail의 경우 계정 비밀번호를 그대로 쓰면 안 됨
  Google 계정 설정 → 2단계 인증 활성화 → 앱 비밀번호 생성
  "xxxx xxxx xxxx xxxx" 형태의 16자리 비밀번호
  → 이 비밀번호를 MAIL_PASS에 사용

iCloud의 경우:
  Apple ID 설정 → 앱 비밀번호 생성
  → 동일하게 앱 전용 비밀번호 사용
```

---

# MailModule 등록 ⭐️⭐️⭐️

```typescript
// mail.module.ts
@Module({
  providers: [MailService],
  exports:   [MailService],   // 다른 모듈에서 주입받으려면 exports 필요
})
export class MailModule {}

// app.module.ts
@Module({
  imports: [MailModule],
})
export class AppModule {}

// 사용하는 서비스에서
@Injectable()
export class ReportService {
  constructor(private readonly mailService: MailService) {}

  async handleReport(report: Report) {
    await this.mailService.sendOpsAlert(
      '새 신고 접수',
      `신고자: ${report.userId}\n사유: ${report.reason}`,
    );
  }
}
```

---

# 언제 이메일을 보내는가 ⭐️⭐️⭐️

```txt
사용자에게 보내는 이메일:
  회원가입 환영 이메일
  비밀번호 재설정 링크
  이메일 인증 코드
  결제 영수증
  문의 답변

운영팀 내부 알림:
  새 신고 접수
  결제 에러
  스케줄 작업 실패
  시스템 에러 (ops alert)

이메일 발송 주의사항:
  try/catch 필수 — SMTP 서버 장애 시 앱이 죽으면 안 됨
  실패해도 핵심 기능은 계속 동작 (이메일은 부수 기능)
  대량 발송은 전문 서비스 사용 (SendGrid, AWS SES)
  Gmail 무료 계정은 하루 500개 제한
```