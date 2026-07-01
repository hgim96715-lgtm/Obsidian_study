---
aliases: [영상 처리, fluent-ffmpeg, NestJS FFmpeg]
tags:
  - NestJS
related:
  - "[[00_NestJS_Ecosystem_HomePage]]"
  - "[[NestJS_FileUpload]]"
  - "[[NestJS_Queue]]"
---

# NestJS_FFmpeg — fluent-ffmpeg 연동

```txt
fluent-ffmpeg = Node.js 에서 FFmpeg 명령어를 코드로 실행하는 라이브러리
영상 변환 / 압축 / 썸네일 추출을 NestJS 서비스에서 처리
```

---

---

# 설치 ⭐️

```bash
pnpm add @ffmpeg-installer/ffmpeg fluent-ffmpeg ffprobe-static
pnpm add -D @types/fluent-ffmpeg
```

```txt
@ffmpeg-installer/ffmpeg   ffmpeg 바이너리를 npm 패키지로 제공
                           서버에 ffmpeg 별도 설치 없이 사용 가능
fluent-ffmpeg              FFmpeg 명령어를 JS/TS 코드로 실행
ffprobe-static             ffprobe 바이너리 (미디어 정보 분석용)
@types/fluent-ffmpeg       TypeScript 타입 정의
```

```bash
Queue 와 같이 쓰는 경우:
  영상 변환은 오래 걸리는 작업
#  → [[NestJS_Queue]] 로 백그라운드 처리 권장

  pnpm add @nestjs/bullmq bullmq
  영상 업로드 → Queue 에 변환 작업 추가 → Worker 에서 ffmpeg 실행
```

---

---

# FFmpeg 경로 설정 ⭐️

```typescript
//main.ts
import * as ffmpeg from 'fluent-ffmpeg';
import * as ffmpegInstaller from '@ffmpeg-installer/ffmpeg';
import * as ffprobeStatic from 'ffprobe-static';

// 바이너리 경로 지정 (서비스 초기화 시 1번)
ffmpeg.setFfmpegPath(ffmpegInstaller.path);
ffmpeg.setFfprobePath(ffprobeStatic.path);
```

```txt
@ffmpeg-installer/ffmpeg 을 쓰는 이유:
  서버 OS 에 ffmpeg 설치 여부와 무관하게 동작
  npm 패키지 안에 바이너리 포함
  → path 로 경로를 명시적으로 지정해야 함
```

---

---

# 썸네일 추출 

```typescript
@Injectable()
export class FfmpegService {
  constructor() {
    ffmpeg.setFfmpegPath(ffmpegInstaller.path);
    ffmpeg.setFfprobePath(ffprobeStatic.path);
  }

  async createThumbnail(inputPath: string, outputPath: string): Promise<void> {
    return new Promise((resolve, reject) => {
      ffmpeg(inputPath)
        .screenshots({
          count: 1,              // 썸네일 1장
          timemarks: ['00:00:01'], // 1초 지점에서 캡처
          filename: 'thumbnail.jpg',
          folder: outputPath,
        })
        .on('end', () => resolve())
        .on('error', (err) => reject(err));
    });
  }
}
```


## 썸네일 추출 — Processor 실전 코드 ️

```typescript
// worker/thumbnail-generation.worker.ts
@Processor('thumbnail-generation')
export class ThumbnailGenerationProcessor extends WorkerHost {

  async process(job: Job): Promise<string[]> {
    const { videoId, videoPath } = job.data as {
      videoId: string[];
      videoPath: string[];
    };

    // 1. 폴더 먼저 생성 ⭐️
    const outputDirectory = join(process.cwd(), 'public', 'thumbnail');
    await mkdir(outputDirectory, { recursive: true });
    //                            ↑ 이미 있어도 에러 없음

    // 2. 개별 썸네일 생성 함수
    const createThumbnail = (id: string, path: string): Promise<string> => {
      const outputName = `${id}.png`;
      const outputPath = join(outputDirectory, outputName);

      return new Promise((resolve, reject) => {
        ffmpeg(path)
          .screenshots({
            count: 1,
            filename: outputName,
            folder: outputDirectory,
            size: '320x240',
          })
          .on('end', () => {
            resolve(outputPath);   // ⭐️ 반드시 resolve() 해야 Job 완료
          })
          .on('error', (err) => {
            reject(err);
          });
      });
    };

    // 3. 여러 영상 병렬 처리
    const thumbnails = await Promise.all(
      videoPath.map((path, i) => createThumbnail(videoId[i], path))
    );

    return thumbnails;
  }
}
```

## 왜 이렇게 작성했나 ⭐️

```txt
await mkdir 먼저:
  ffmpeg 가 폴더보다 먼저 실행되면 간헐적 실패
  mkdir → ffmpeg 순서 보장
  recursive: true → 폴더가 이미 있어도 에러 없이 통과

Promise + await:
  ffmpeg 는 이벤트 기반 (콜백) 으로 동작
  Promise 로 감싸야 async/await 로 제어 가능
  await 없으면 Job 이 ffmpeg 완료 전에 끝남

.on('end', () => resolve()):
  반드시 있어야 함
  없으면 ffmpeg 가 끝나도 Job 이 완료되지 않음
  → Queue 가 Job 을 계속 대기 상태로 인식

videoPath.map((path, i) => createThumbnail(videoId[i], path)):
  job.data 가 배열 → 여러 영상을 한 Job 에서 처리
  videoId[i] / videoPath[i] 인덱스 일치 보장
  → 파일명과 ID 짝이 맞지 않으면 다중 업로드 시 버그

Promise.all:
  여러 썸네일을 순차가 아닌 병렬로 처리
  → 전체 처리 시간 단축
```


---

---

# 영상 변환 & 압축

```typescript
async convertVideo(inputPath: string, outputPath: string): Promise<void> {
  return new Promise((resolve, reject) => {
    ffmpeg(inputPath)
      .videoCodec('libx264')   // H.264 코덱
      .audioCodec('aac')
      .outputOptions([
        '-crf 23',             // 품질 (낮을수록 고화질 / 18~28 권장)
        '-preset fast',        // 인코딩 속도 (ultrafast ~ veryslow)
      ])
      .output(outputPath)
      .on('progress', (progress) => {
        console.log(`진행률: ${progress.percent?.toFixed(1)}%`);
      })
      .on('end', () => resolve())
      .on('error', (err) => reject(err))
      .run();
  });
}
```

---

---

# 미디어 정보 분석 (ffprobe)

```typescript
async getVideoInfo(filePath: string): Promise<ffmpeg.FfprobeData> {
  return new Promise((resolve, reject) => {
    ffmpeg.ffprobe(filePath, (err, metadata) => {
      if (err) return reject(err);
      resolve(metadata);
    });
  });
}

// 사용
const info = await this.ffmpegService.getVideoInfo('video.mp4');
console.log(info.format.duration);   // 영상 길이 (초)
console.log(info.streams[0].width);  // 가로 해상도
console.log(info.streams[0].height); // 세로 해상도
```

---

---

# fluent-ffmpeg 주요 메서드

|메서드|역할|
|---|---|
|`.videoCodec('libx264')`|비디오 코덱 설정|
|`.audioCodec('aac')`|오디오 코덱 설정|
|`.outputOptions([...])`|추가 옵션 설정|
|`.screenshots({...})`|썸네일 추출|
|`.output(path)`|출력 파일 경로|
|`.on('progress', fn)`|진행률 콜백|
|`.on('end', fn)`|완료 콜백|
|`.on('error', fn)`|에러 콜백|
|`.run()`|실행|

---

---

# 핵심 흐름

```bash
영상 업로드
    ↓
Queue 에 변환 작업 추가 ( # → [[NestJS_Queue]])
    ↓
Worker 에서 FfmpegService 호출
    ↓
변환 / 썸네일 추출 / 압축
    ↓
결과 파일 저장 (로컬 or S3)
```