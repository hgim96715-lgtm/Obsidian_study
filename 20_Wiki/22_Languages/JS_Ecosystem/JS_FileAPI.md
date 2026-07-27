---
aliases:
  - File
  - FileReader
  - file.size
  - file.onload
  - file.onerror
tags:
  - JavaScript
related:
  - "[[00_JS_Ecosystem_HomePage]]"
  - "[[JS_DOM]]"
  - "[[JS_BrowserAPI]]"
  - "[[React_useRef]]"
---
# JS_FileAPI — 파일 선택 & 읽기

> [!info] 
> 브라우저에서 파일을 다루는 두 가지: `<input type="file">`로 선택하고, `FileReader`로 읽는다. 
> 커스텀 버튼 → 숨겨진 input 클릭 패턴이 실무에서 표준이다.

---

# File 객체 — 선택된 파일 정보 ⭐️⭐️⭐️

```typescript
const file: File = e.target.files?.[0];

file.name      // 'photo.jpg'
file.type      // 'image/jpeg'  (MIME 타입)
file.size      // 1234567       (바이트 단위)
file.lastModified  // 타임스탬프 (ms)
```

## 파일 유효성 검사 ⭐️⭐️⭐️⭐️

```typescript
function validateImage(file: File): string | null {
  // 이미지 타입 확인
  if (!file.type.startsWith('image/')) {
    return '이미지 파일만 선택할 수 있어요.';
  }
  // 크기 제한 (1.2MB)
  if (file.size > 1.2 * 1024 * 1024) {
    return '이미지 파일은 1.2MB 이하여야 해요.';
  }
  return null;  // 통과
}
```

```txt
file.type.startsWith('image/'):
  'image/jpeg', 'image/png', 'image/webp', 'image/gif' 등을 한 번에 허용
  'image/' 접두사로 이미지 계열 전체를 체크

파일 크기 단위:
  file.size = 바이트(byte)
  1KB = 1024
  1MB = 1024 * 1024
  1.2MB = 1.2 * 1024 * 1024 = 1,258,291 bytes

  file.size > 5 * 1024 * 1024  → 5MB 초과 여부
```

---

# 숨겨진 input + 커스텀 버튼 패턴 ⭐️⭐️⭐️⭐️

```tsx
// React에서 커스텀 버튼으로 파일 선택 트리거
const galleryInputRef = useRef<HTMLInputElement>(null);

// 숨겨진 file input — 직접 클릭은 안 됨
<input
  ref={galleryInputRef}
  type="file"
  accept="image/*"          // 이미지만 허용 (파일 선택 창 필터링)
  className="hidden"        // 화면에서 완전히 숨김
  onChange={(e) => {
    onPickGallery(e.target.files?.[0]);  // 선택된 첫 번째 파일
    e.target.value = '';                  // 같은 파일 다시 선택 가능하게 초기화
  }}
/>

// 커스텀 버튼 — 클릭하면 input을 대신 클릭
<button
  type="button"
  onClick={() => galleryInputRef.current?.click()}
>
  갤러리에서 고르기
</button>
```

```txt
왜 hidden input + 버튼인가:
  <input type="file">의 기본 UI는 OS마다 달라서 커스텀 스타일링 불가
  → input을 완전히 숨기고, 커스텀 버튼이 input.click()을 대신 호출

  display:none vs className="hidden":
  둘 다 input을 안 보이게 하고 클릭도 안 됨 → ref로만 click() 호출 가능

e.target.value = '' 가 필요한 이유:
  같은 파일을 다시 선택하면 onChange가 발생하지 않음
  (이미 같은 값이라 브라우저가 change 이벤트를 안 보냄)
  → value를 빈 문자열로 초기화하면 다음 선택 시 항상 onChange 발생

accept="image/*":
  파일 선택 창에서 이미지 파일만 필터링 (사용자 편의)
  보안 검증은 아님 → 코드에서 file.type으로 반드시 별도 검증 필요

e.target.files?.[0]:
  files는 FileList — 여러 파일 선택 가능
  [0] → 첫 번째 파일만 (single 선택)
  ?.  → files가 null일 수 있음 (취소하면 null)
```

---

# FileReader — 파일 내용 읽기 ⭐️⭐️⭐️⭐️

```typescript
const reader = new FileReader();

// 읽기 완료 시 호출
reader.onload = () => {
  const result = reader.result;  // string | ArrayBuffer | null
};

// 에러 발생 시 호출
reader.onerror = () => {
  console.error('파일 읽기 실패');
};

// 읽기 시작 (비동기)
reader.readAsDataURL(file);     // → base64 data URL
reader.readAsText(file);        // → 텍스트 문자열
reader.readAsArrayBuffer(file); // → ArrayBuffer (바이너리)
```

|메서드|결과 (`reader.result`)|언제|
|---|---|---|
|`readAsDataURL(file)`|`data:image/jpeg;base64,...`|이미지 미리보기, localStorage 저장|
|`readAsText(file)`|파일 내용 문자열|텍스트/CSV 파일 파싱|
|`readAsArrayBuffer(file)`|`ArrayBuffer`|바이너리 처리, 업로드|

---

# 전체 패턴 — 이미지 선택 → 검증 → 미리보기 ⭐️⭐️⭐️⭐️

```typescript
function onPickGallery(file: File | undefined) {
  if (!file) return;

  // 1. 유효성 검사
  if (!file.type.startsWith('image/')) {
    setError('이미지 파일만 선택할 수 있어요.');
    return;
  }
  if (file.size > 1.2 * 1024 * 1024) {
    setError('이미지 파일은 1.2MB 이하여야 해요.');
    return;
  }
  setError('');

  // 2. FileReader로 data URL 변환 (비동기)
  const reader = new FileReader();

  reader.onload = () => {
    // reader.result 타입: string | ArrayBuffer | null
    const dataUrl = typeof reader.result === 'string' ? reader.result : null;
    if (!dataUrl) return;

    setBackgroundUrl(dataUrl);  // 3. 미리보기 표시
    saveToStorage(dataUrl);     // 4. 로컬 저장
  };

  reader.onerror = () => {
    setError('이미지 파일을 읽는 중 오류가 발생했어요.');
  };

  reader.readAsDataURL(file);   // 5. 읽기 시작
}
```

```txt
typeof reader.result === 'string':
  reader.result 타입이 string | ArrayBuffer | null 이라서
  readAsDataURL은 string을 반환하지만 TS가 이를 보장 못 함
  → typeof 체크로 string임을 확인 후 사용

data URL 형식:
  data:image/jpeg;base64,/9j/4AAQSkZJRg...
  → <img src={dataUrl} /> 로 바로 미리보기 가능
  → localStorage.setItem(key, dataUrl) 로 저장 가능
  ⚠️ 크기가 커서 5~10MB localStorage 한도에 주의
     1.2MB 파일 → base64 인코딩 후 약 1.6MB

FileReader는 비동기:
  readAsDataURL()을 호출해도 결과가 바로 안 나옴
  → onload 콜백 안에서 reader.result를 읽어야 함
  → readAsDataURL() 호출 뒤에 reader.result를 바로 읽으면 null
```

---

# 한눈에

```txt
파일 선택 패턴:
  숨겨진 <input type="file" ref={ref} className="hidden" />
  커스텀 <button onClick={() => ref.current?.click()}>
  onChange: e.target.files?.[0]  (첫 번째 파일)
  e.target.value = ''  (같은 파일 재선택 허용)

File 객체:
  file.type    'image/jpeg' 등 MIME 타입
  file.size    바이트 단위 (1MB = 1024 * 1024)
  file.name    파일명

유효성 검사:
  file.type.startsWith('image/')  이미지 타입
  file.size > N * 1024 * 1024     크기 제한

FileReader:
  const reader = new FileReader()
  reader.onload = () => { reader.result }  // 비동기 완료
  reader.onerror = () => {}
  reader.readAsDataURL(file)   → data:image/...;base64,...  (미리보기, localStorage)
  reader.readAsText(file)      → 문자열
  reader.readAsArrayBuffer(file) → 바이너리

data URL:
  <img src={dataUrl} />  미리보기 가능
  localStorage 저장 가능 (용량 주의 — base64는 원본보다 약 33% 큼)
```