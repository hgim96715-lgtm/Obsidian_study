---
aliases:
  - Canvas
  - API
  - HTML 요소
tags:
  - JavaScript
related:
  - "[[00_JS_Ecosystem_HomePage]]"
  - "[[JS_DOM]]"
  - "[[React_useRef]]"
---
# JS_Canvas — Canvas 2D API

> [!info] 
> `<canvas>`는 픽셀 단위로 직접 그림을 그리는 HTML 요소. 
> `getContext('2d')`로 2D 컨텍스트를 얻어 선·도형·이미지를 그린다. 
> React에서는 `useRef`로 canvas 요소를 참조하고 `useEffect` 안에서 조작한다.

---

# 기본 설정 ⭐️⭐️⭐️

```typescript
const canvas = document.querySelector('canvas') as HTMLCanvasElement;

// canvas 크기 = 부모 요소 크기에 맞춤
const parent = canvas.parentElement!;
canvas.width  = parent.clientWidth;
canvas.height = parent.clientHeight;

// 2D 컨텍스트 얻기
const ctx = canvas.getContext('2d');
if (!ctx) return;
```

```txt
canvas.width / canvas.height:
  CSS width/height 가 아닌 "픽셀 버퍼" 크기
  이 값을 설정하면 캔버스 내용이 초기화됨 (clearRect 불필요)
  CSS로 크기를 늘리면 흐릿해짐 → 항상 실제 픽셀 크기와 맞춰야 함

  // 레티나(HiDPI) 디스플레이 대응
  const dpr = window.devicePixelRatio ?? 1;
  canvas.width  = parent.clientWidth  * dpr;
  canvas.height = parent.clientHeight * dpr;
  ctx.scale(dpr, dpr);  // 이후 좌표는 CSS 픽셀 단위로 쓰면 됨
```

---

# 자주 쓰는 컨텍스트 속성

|속성|설명|
|---|---|
|`ctx.strokeStyle`|선 색상 (`'#ff0000'`, `'rgba(0,0,0,0.5)'`)|
|`ctx.fillStyle`|채우기 색상|
|`ctx.lineWidth`|선 두께 (px)|
|`ctx.lineCap`|선 끝 모양: `'butt'`(기본) / `'round'` / `'square'`|
|`ctx.lineJoin`|선 꺾임 모양: `'miter'`(기본) / `'round'` / `'bevel'`|
|`ctx.globalAlpha`|전체 투명도 (0~1)|

---

# 선 그리기 ⭐️⭐️⭐️⭐️

```typescript
ctx.beginPath();             // 새 경로 시작 — 이전 경로와 분리
ctx.moveTo(x1, y1);          // 시작점 이동 (선 없이)
ctx.lineTo(x2, y2);          // 직선 추가
ctx.stroke();                // 선 실제로 그리기 (stroke 호출 전까지 안 보임)

// 닫힌 도형
ctx.closePath();             // 시작점으로 선을 이어서 닫음
ctx.fill();                  // 내부 채우기
```

```txt
beginPath() 가 중요한 이유:
  beginPath() 없이 여러 번 stroke()를 호출하면
  이전 경로들도 다시 그려져서 선이 점점 두꺼워짐
  → 매번 새 경로를 시작하려면 beginPath() 먼저
```

## 부드러운 곡선 (손글씨 스트로크)

```typescript
// 여러 점을 연결해서 자연스러운 선 그리기
ctx.beginPath();
ctx.strokeStyle = stroke.color;
ctx.lineWidth   = stroke.width;
ctx.lineCap     = 'round';   // 선 끝을 둥글게 — 손글씨 느낌
ctx.lineJoin    = 'round';   // 꺾임도 둥글게

stroke.points.forEach((p, i) => {
  if (i === 0) ctx.moveTo(p.x, p.y);
  else         ctx.lineTo(p.x, p.y);
});
ctx.stroke();
```

---

# 지우기

```typescript
ctx.clearRect(0, 0, canvas.width, canvas.height);  // 전체 지우기
ctx.clearRect(x, y, width, height);                 // 부분 지우기
```

---

# React — StrokeLayer 패턴 ⭐️⭐️⭐️⭐️

```typescript
// 스트로크(획) 데이터를 받아서 canvas에 그리는 컴포넌트
function StrokeLayer({ strokes }: { strokes: Stroke[] }) {
  const ref = useRef<HTMLCanvasElement>(null);

  useEffect(() => {
    const c = ref.current;
    if (!c) return;

    // 부모 크기에 맞춰 canvas 픽셀 버퍼 설정
    const parent = c.parentElement;
    if (!parent) return;
    const w = parent.clientWidth;
    const h = parent.clientHeight;
    c.width  = w;   // ← 이 시점에 자동으로 canvas 초기화됨
    c.height = h;

    const ctx = c.getContext('2d');
    if (!ctx) return;

    ctx.clearRect(0, 0, w, h);  // 명시적 초기화

    for (const s of strokes) {
      if (s.points.length < 2) continue;  // 점 하나는 선이 안 됨

      ctx.beginPath();
      ctx.strokeStyle = s.color;
      ctx.lineWidth   = s.width;
      ctx.lineCap     = 'round';
      ctx.lineJoin    = 'round';

      s.points.forEach((p, i) => {
        // 정규화된 좌표(0~1) → 실제 픽셀로 변환
        const x = p.x * w;
        const y = p.y * h;
        if (i === 0) ctx.moveTo(x, y);
        else         ctx.lineTo(x, y);
      });
      ctx.stroke();
    }
  }, [strokes]);  // strokes가 바뀔 때마다 다시 그림

  return (
    <canvas
      ref={ref}
      style={{ position: 'absolute', inset: 0, pointerEvents: 'none' }}
    />
  );
}
```

```txt
정규화 좌표(0~1) 패턴:
  좌표를 픽셀 값으로 저장하면 canvas 크기가 달라질 때 위치가 틀려짐
  x: 0~1, y: 0~1 (비율)로 저장하면 어떤 크기에서도 올바른 위치

  저장: { x: e.clientX / width,  y: e.clientY / height }
  그리기: { x: p.x * canvasWidth, y: p.y * canvasHeight }

useEffect deps에 [strokes] 넣는 이유:
  strokes 배열이 바뀌면 (새 획 추가, 수정) 다시 전체를 그려야 함
  Canvas는 React처럼 "변경된 것만 업데이트"하지 않음
  → 항상 전체를 clearRect 후 다시 그리는 게 가장 단순하고 안전
```

---

# 한눈에

```txt
초기화:
  canvas.width = parent.clientWidth   픽셀 버퍼 크기 설정 (CSS와 별개)
  ctx = canvas.getContext('2d')

그리기 순서:
  ctx.beginPath()               새 경로 시작
  ctx.moveTo(x, y)              시작점
  ctx.lineTo(x, y)              이어서 직선
  ctx.stroke() / ctx.fill()     실제로 그리기

지우기:
  ctx.clearRect(0, 0, w, h)    전체 또는 부분 지우기

스타일:
  strokeStyle, fillStyle        색상
  lineWidth                     선 두께
  lineCap: 'round'              선 끝 둥글게 (손글씨 느낌)
  lineJoin: 'round'             꺾임 둥글게

React 패턴:
  useRef<HTMLCanvasElement>(null) + useEffect([data])
  데이터 바뀌면 clearRect → 전체 다시 그리기
  정규화 좌표(0~1) → 크기 변경에 강건

Pointer Events → [[JS_DOM]]
```