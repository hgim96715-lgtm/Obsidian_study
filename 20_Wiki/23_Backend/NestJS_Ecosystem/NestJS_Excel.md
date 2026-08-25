---
aliases:
  - Excel
  - xlsx
  - 엑셀
  - 스프레드시트
tags:
  - NestJS
  - Excel
related:
  - "[[00_NestJS_Ecosystem_HomePage]]"
  - "[[NestJS_S3Storage]]"
---
# NestJS_Excel — xlsx 파일 생성 · 관리

> [!info]
> NestJS에서 xlsx 라이브러리로 엑셀 파일을 생성·저장·다운로드하는 패턴.
> 사본 생성 → 월별 폴더 축적 → 나중에 묶음 처리 흐름에 맞춰 정리.

---

# 설치 ⭐️⭐️⭐️⭐️

```bash
pnpm --filter api add xlsx
```

```txt
xlsx (SheetJS Community Edition)
  - Node.js · 브라우저 양쪽에서 동작
  - .xlsx · .xls · .csv · .ods 등 다양한 포맷 지원
  - 타입 선언 내장 (@types 별도 설치 불필요)
```

---

# 핵심 API ⭐️⭐️⭐️⭐️

```typescript
import * as XLSX from 'xlsx';

// 워크북 생성
const wb = XLSX.utils.book_new();

// 데이터 → 시트 (배열 방식)
const ws = XLSX.utils.aoa_to_sheet([
  ['이름', '금액', '날짜'],   // 헤더 행
  ['홍길동', 10000, '2025-01-01'],
]);

// 데이터 → 시트 (객체 배열 방식)
const ws2 = XLSX.utils.json_to_sheet([
  { 이름: '홍길동', 금액: 10000, 날짜: '2025-01-01' },
  { 이름: '김철수', 금액: 20000, 날짜: '2025-01-02' },
]);

// 시트를 워크북에 추가
XLSX.utils.book_append_sheet(wb, ws, '시트이름');

// 파일로 저장 (서버 디스크)
XLSX.writeFile(wb, '/path/to/output.xlsx');

// Buffer로 변환 (응답·S3 업로드 등)
const buf = XLSX.write(wb, { type: 'buffer', bookType: 'xlsx' });
```

---

# 월별 폴더 축적 패턴 ⭐️⭐️⭐️⭐️

```txt
구조:
  exports/
    2025-01/
      report_2025-01-05.xlsx
      report_2025-01-12.xlsx
    2025-02/
      report_2025-02-03.xlsx
    ...

흐름:
  ① 실행 시점의 연-월 계산
  ② 폴더 없으면 생성 (fs.mkdirSync)
  ③ 파일명에 날짜 포함해서 저장
  ④ 나중에 월별 폴더째로 묶음 처리
```

```typescript
import * as fs from 'fs';
import * as path from 'path';
import * as XLSX from 'xlsx';

export class ExcelService {
  private readonly baseDir = path.resolve('exports');

  generateMonthlyReport(data: ReportRow[]): string {
    const now = new Date();
    const yearMonth = `${now.getFullYear()}-${String(now.getMonth() + 1).padStart(2, '0')}`;
    const dateStr = now.toISOString().slice(0, 10); // YYYY-MM-DD

    // 월별 폴더 생성
    const dir = path.join(this.baseDir, yearMonth);
    fs.mkdirSync(dir, { recursive: true });

    // 파일명: report_2025-01-05.xlsx
    const filename = `report_${dateStr}.xlsx`;
    const filePath = path.join(dir, filename);

    // 워크북 생성 · 저장
    const wb = XLSX.utils.book_new();
    const ws = XLSX.utils.json_to_sheet(data);
    XLSX.utils.book_append_sheet(wb, ws, '리포트');
    XLSX.writeFile(wb, filePath);

    return filePath; // 저장된 경로 반환
  }
}
```

---

# 사본 생성 패턴 ⭐️⭐️⭐️

```txt
"원본 건드리지 않고 사본만 별도 저장"
  → 원본 파일 경로를 읽어서 워크북 로드
  → 필요한 셀만 수정
  → 다른 경로에 저장
```

```typescript
// 기존 엑셀 파일 복사 후 수정
cloneAndModify(srcPath: string, destPath: string, updates: Record<string, string>) {
  const wb = XLSX.readFile(srcPath);
  const ws = wb.Sheets[wb.SheetNames[0]];

  // 특정 셀 값 수정
  for (const [cell, value] of Object.entries(updates)) {
    if (ws[cell]) {
      ws[cell].v = value; // v = value
    }
  }

  XLSX.writeFile(wb, destPath);
}

// 사용
this.cloneAndModify(
  'templates/base.xlsx',
  `exports/2025-01/copy_${Date.now()}.xlsx`,
  { B2: '수정된 값', C5: '2025-01-10' },
);
```

---

# 응답으로 직접 다운로드 — StreamableFile ⭐️⭐️⭐️⭐️

```typescript
@Get('daily-excel')
async downloadYesterdayExcel(): Promise<StreamableFile> {
  const report = await this.adminDailyExcelService.createYesterdayWorkbook();

  return new StreamableFile(report.buffer, {
    type:        'application/vnd.openxmlformats-officedocument.spreadsheetml.sheet',
    disposition: `attachment; filename="${report.filename}"`,
    length:      report.buffer.length,
  });
}
```

## StreamableFile이란 ⭐️⭐️⭐️⭐️

```txt
StreamableFile = NestJS 내장 클래스
  Buffer 또는 Stream을 HTTP 응답으로 바꿔주는 래퍼

  일반 JSON 응답:  return { data }          → NestJS가 자동으로 JSON 직렬화
  파일 응답:       return new StreamableFile(buf) → NestJS가 바이너리 스트림으로 전송

왜 @Res() 없이 쓸 수 있는가:
  일반적으로 Response 객체를 직접 제어하려면 @Res()가 필요
  StreamableFile을 return하면 NestJS가 내부적으로
  res.set(headers) + buf를 pipe해서 응답을 완성해줌
  → @Res() 없이도 헤더·바디 모두 자동 처리

import { StreamableFile } from '@nestjs/common';
```

## StreamableFile 옵션 3가지 ⭐️⭐️⭐️⭐️

```typescript
new StreamableFile(buffer, {
  type:        '...',   // Content-Type 헤더
  disposition: '...',   // Content-Disposition 헤더
  length:      number,   // Content-Length 헤더
})
```

```txt
type — Content-Type 헤더
  브라우저에게 "이 파일이 어떤 종류인지" 알려줌
  브라우저는 이 값으로 파일을 어떻게 처리할지 결정

  자주 쓰는 MIME 타입:
    .xlsx  application/vnd.openxmlformats-officedocument.spreadsheetml.sheet
    .xls   application/vnd.ms-excel
    .csv   text/csv
    .pdf   application/pdf
    .png   image/png
    .zip   application/zip

  "vnd.openxmlformats-officedocument.spreadsheetml.sheet":
    vnd = vendor (특정 회사의 독점 포맷)
    openxmlformats = Microsoft Open XML 표준
    spreadsheetml.sheet = 스프레드시트 시트 파일
    → 풀어쓰면 "Microsoft Office의 xlsx 포맷"

disposition — Content-Disposition 헤더
  브라우저에게 "이 파일을 어떻게 처리할지" 알려줌

  attachment  → 다운로드 대화상자 열림 (파일로 저장)
  inline      → 브라우저에서 바로 표시 (PDF, 이미지 등)

  filename="cinemo-2026-08-25.xlsx":
    다운로드 대화상자에서 기본 파일명으로 제안됨
    사용자가 다른 이름으로 저장할 수 있음

length — Content-Length 헤더
  파일의 정확한 바이트 크기를 브라우저에 알려줌

  없어도 동작하지만 있으면:
    다운로드 진행률 표시 가능 (전체 크기를 알아야 % 계산 가능)
    브라우저가 연결 재사용(keep-alive) 판단에 활용
  Buffer면 .length로 항상 정확한 값을 알 수 있으므로 넣어주는 것이 좋음
```

## Service에서 Buffer 반환, Controller에서 StreamableFile ⭐️⭐️⭐️

```typescript
// ✅ 권장 패턴 — Service는 Buffer만 반환
// Service
async createYesterdayWorkbook(): Promise<{ date: string; filename: string; buffer: Buffer }> {
  const workbook = XLSX.utils.book_new();
  // ... 시트 생성 ...
  return {
    date:     dateKey,
    filename: `report-${dateKey}.xlsx`,
    buffer:   XLSX.write(workbook, { bookType: 'xlsx', type: 'buffer' }) as Buffer,
  };
}

// Controller
@Get('daily-excel')
async downloadYesterdayExcel(): Promise<StreamableFile> {
  const report = await this.adminDailyExcelService.createYesterdayWorkbook();
  return new StreamableFile(report.buffer, {
    type:        'application/vnd.openxmlformats-officedocument.spreadsheetml.sheet',
    disposition: `attachment; filename="${report.filename}"`,
    length:      report.buffer.length,
  });
}
```

```txt
역할 분리:
  Service  → 데이터 조합·워크북 생성·Buffer 반환 (HTTP 무관)
  Controller → Buffer를 StreamableFile로 감싸서 HTTP 응답 완성

  Service가 StreamableFile을 직접 만들지 않는 이유:
    Service는 HTTP 계층을 모르는 게 맞음
    Buffer 자체는 테스트·저장·S3 업로드 등 다양하게 재사용 가능
    StreamableFile은 HTTP 응답 전용 → Controller 책임
```

---


# Cron 전용 엔드포인트 — GitHub Actions ⭐️⭐️⭐️⭐️

```typescript
@Public()   // JWT Guard 완전 우회
@Post('daily-excel/cron')
async downloadYesterdayExcelForCron(
  @Headers('x-cron-secret') secret: string | undefined,
): Promise<StreamableFile> {
  // x-cron-secret 직접 검증 (JWT 대신)
  if (!process.env.CRON_SECRET || secret !== process.env.CRON_SECRET) {
    throw new UnauthorizedException('잘못된 cron secret입니다.');
  }
  const report = await this.adminDailyExcelService.createYesterdayWorkbook();
  return this.toDownload(report);
}

// StreamableFile 생성 공통 헬퍼 — DRY 패턴
private toDownload(report: { buffer: Buffer; filename: string }): StreamableFile {
  return new StreamableFile(report.buffer, {
    type:        'application/vnd.openxmlformats-officedocument.spreadsheetml.sheet',
    disposition: `attachment; filename="${report.filename}"`,
    length:      report.buffer.length,
  });
}
```

```txt
왜 @Public() + x-cron-secret 조합인가:
  GitHub Actions는 JWT 토큰이 없음 (사람이 아닌 자동화 프로세스)
  → 전역 JWT Guard가 막음

  @Public() → JWT Guard 완전 우회 (인증 없이 접근 허용)
  x-cron-secret → 대신 사전 공유 비밀 키로 "이 요청이 Actions에서 왔다"를 검증

  CRON_SECRET 환경변수:
    Railway API 서비스 → Variables에 CRON_SECRET 등록
    GitHub → Repository Secrets에 CRON_SECRET 등록 (Actions에서 참조)
    둘이 같은 값이어야 함

@Public() 없이 JWT Guard를 통과하는 방법이 없는 이유:
  전역 Guard는 모든 라우트에 적용됨
  @Public()은 "이 라우트는 Guard에서 skip" 마커 역할
  → MoviePool cron, AI 웹훅 등 자동화 엔드포인트 모두 같은 패턴
```

```txt
검증 순서:
  ① CRON_SECRET 환경변수 자체가 없으면 → 401 (서버 설정 누락 방어)
  ② secret !== CRON_SECRET → 401 (잘못된 키)
  ③ 통과 → 리포트 생성 · 반환

  !process.env.CRON_SECRET || secret !== process.env.CRON_SECRET
  ↑ 왼쪽부터 평가: 환경변수 없으면 오른쪽 비교도 스킵
  → 환경변수 미설정 상태로 실수로 배포해도 열리지 않음
```

## toDownload() — DRY 헬퍼 패턴 ⭐️⭐️⭐️

```typescript
// ❌ 중복 — 엔드포인트마다 같은 StreamableFile 옵션 반복
@Get('daily-excel')
async download(): Promise<StreamableFile> {
  const report = await ...;
  return new StreamableFile(report.buffer, {
    type: 'application/vnd.openxmlformats-officedocument.spreadsheetml.sheet',
    disposition: `attachment; filename="${report.filename}"`,
    length: report.buffer.length,
  });
}

@Post('daily-excel/cron')
async cron(): Promise<StreamableFile> {
  const report = await ...;
  return new StreamableFile(report.buffer, {   // ← 완전 동일한 코드 중복
    type: 'application/vnd.openxmlformats-officedocument.spreadsheetml.sheet',
    disposition: `attachment; filename="${report.filename}"`,
    length: report.buffer.length,
  });
}

// ✅ private 헬퍼로 추출 — 두 엔드포인트 모두 재사용
private toDownload(report: { buffer: Buffer; filename: string }): StreamableFile {
  return new StreamableFile(report.buffer, {
    type:        'application/vnd.openxmlformats-officedocument.spreadsheetml.sheet',
    disposition: `attachment; filename="${report.filename}"`,
    length:      report.buffer.length,
  });
}
```

```txt
private 메서드로 분리하는 기준:
  같은 로직이 2곳 이상 반복되면 추출
  HTTP 응답 래핑처럼 "변하지 않는 규격"은 항상 헬퍼 후보

private인 이유:
  Controller 내부에서만 사용 → 외부(다른 서비스·컨트롤러)에 노출 불필요
  NestJS가 라우트로 등록하지 않음
```

## GitHub Actions 워크플로우 연동

```yaml
# .github/workflows/daily-excel.yml
name: Daily Excel Report

on:
  schedule:
    - cron: '0 15 * * *'   # UTC 15:00 = KST 자정 (00:00)
  workflow_dispatch:         # 수동 실행도 허용

jobs:
  download-report:
    runs-on: ubuntu-latest
    steps:
      - name: Download yesterday excel
        run: |
          curl -X POST "${{ secrets.API_URL }}/admin/daily-excel/cron"             -H "x-cron-secret: ${{ secrets.CRON_SECRET }}"             --output "report-$(date +%Y-%m-%d).xlsx"

      # 결과 파일을 Artifacts로 업로드 (Actions UI에서 다운로드 가능)
      - name: Upload to Actions Artifacts
        uses: actions/upload-artifact@v4
        with:
          name: daily-report-${{ github.run_id }}
          path: report-*.xlsx
          retention-days: 30
```

```txt
cron: '0 15 * * *' — UTC 기준
  UTC 15:00 = KST 00:00 (자정)
  어제 KST 날짜 데이터를 자정 직후에 수집

workflow_dispatch:
  수동으로도 실행 가능 → 테스트·재실행에 유용

secrets.API_URL / secrets.CRON_SECRET:
  GitHub Repository → Settings → Secrets and variables → Actions에 등록
  API_URL = Railway API 서비스의 공개 도메인
  CRON_SECRET = Railway Variables의 CRON_SECRET과 동일한 값

GitHub Actions Artifacts:
  다운로드한 xlsx를 Actions 실행 결과로 저장
  retention-days: 30 → 30일 후 자동 삭제
  Actions UI → 해당 run → Artifacts 섹션에서 다운로드 가능
```

# 열 너비 조정 ⭐️⭐️⭐️

```typescript
// !cols 배열로 열 너비 설정 (wch = 문자 수 기준)
ws['!cols'] = [
  { wch: 20 }, // A열
  { wch: 15 }, // B열
  { wch: 12 }, // C열
];
```

---

# XLSX.write() type 옵션 — Buffer란 ⭐️⭐️⭐️⭐️

```txt
XLSX.write(wb, { type: '...', bookType: 'xlsx' })
                       ↑
                       반환 형식을 결정하는 옵션
```

## Node.js Buffer

```txt
Buffer = Node.js의 원시 바이너리 데이터 컨테이너
  - 메모리에 올라간 바이트 배열 (Uint8Array의 서브클래스)
  - 이진 파일(이미지·엑셀·PDF 등)을 메모리에 올릴 때 사용
  - 문자 인코딩 없이 순수 바이트로 처리

엑셀에서 Buffer가 필요한 이유:
  .xlsx 파일은 텍스트가 아닌 이진 파일 (ZIP 안에 XML 묶음)
  → 문자열로 다루면 바이트가 손상됨
  → Buffer(바이트 배열)로 다뤄야 온전히 전달·저장 가능
```

## type 옵션 종류

| `type` 값     | 반환 타입        | 언제 쓰는가                             |
|--------------|-----------------|---------------------------------------|
| `'buffer'`   | `Buffer`        | HTTP 응답 · fs.writeFileSync · S3 업로드 |
| `'base64'`   | `string`        | JSON 페이로드에 포함, data URI          |
| `'binary'`   | `string`        | 구형 브라우저 호환 (레거시)               |
| `'array'`    | `Uint8Array`    | 브라우저 Blob 생성                       |
| `'file'`     | 없음 (저장만)    | 경로를 직접 지정해서 저장 (writeFile 동일) |

```typescript
// HTTP 응답으로 직접 내려줄 때 → Buffer
const buf = XLSX.write(wb, { type: 'buffer', bookType: 'xlsx' }) as Buffer;
return new StreamableFile(buf);

// 서버 디스크에 저장할 때 → writeFile (type 옵션 불필요)
XLSX.writeFile(wb, '/path/to/file.xlsx');

// S3 업로드할 때 → Buffer
await s3.putObject({ Body: buf, ... }).promise();
```

---

# 파일 포맷 비교 — xlsx · xls · csv · ods ⭐️⭐️⭐️

```txt
.xlsx — Open XML (권장)
  Excel 2007+ 표준, ZIP 안에 XML 파일들
  최대 1,048,576행 · 16,384열
  서식·수식·다중 시트·차트 모두 지원
  SheetJS의 기본 타겟 포맷

.xls — 구형 Binary 포맷 (비권장)
  Excel 97~2003 시대 형식, BIFF 이진 포맷
  최대 65,536행 · 256열 (1/16 규모)
  최신 Excel도 읽지만 새로 만들 이유 없음

.csv — 순수 텍스트 (경량·범용)
  쉼표로 구분된 평문, 어떤 도구에서도 열림
  서식·수식·다중 시트 없음
  대용량 원본 데이터 내보내기 · 데이터 파이프라인에 적합

.ods — OpenDocument Spreadsheet
  LibreOffice/OpenOffice 표준, 오픈소스 포맷
  Excel에서도 열리지만 호환성 완전 보장 안 됨
  특별한 이유 없으면 잘 쓰지 않음
```

## 선택 기준

```txt
비즈니스 리포트 (열리는 순간 예쁘게 보여야)
  → .xlsx  (서식·시트 구성 가능, Excel 바로 오픈)

데이터 파이프라인 · DB 임포트 · 로그 내보내기
  → .csv   (가볍고 어디서나 파싱 가능)

```

# 자주 만나는 에러

| 에러 | 원인 | 해결 |
|---|---|---|
| `Cannot find module 'xlsx'` | 설치 안 됨 | `pnpm --filter api add xlsx` |
| `ENOENT: no such file or directory` | 폴더 없음 | `fs.mkdirSync(dir, { recursive: true })` |
| 한글 파일명 깨짐 | Content-Disposition 인코딩 | `encodeURIComponent(filename)` |
| 셀 수정 후 반영 안 됨 | ws[cell]이 undefined | 셀 존재 여부 확인 후 수정 |
