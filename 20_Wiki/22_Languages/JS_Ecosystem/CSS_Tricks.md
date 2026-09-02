---
aliases: [::after, CSS 실전 패턴, line-clamp, overflow]
tags: [CSS]
related:
  - "[[00_JS_Ecosystem_HomePage]]"
  - "[[CSS_Grid]]"
  - "[[CSS_Layout]]"
---
# CSS_Tricks — CSS 실전 패턴

>[!info]
>자주 마주치는 CSS 현상과 해결 패턴. 
>`line-clamp`로 텍스트 자르기, `overflow: hidden`이 `::after` 꼬리를 clip하는 문제, 말풍선 꼬리 구현 등.

---

# line-clamp — 텍스트 N줄에서 말줄임 ⭐️⭐️⭐️⭐️

```txt
line-clamp = 긴 텍스트를 N줄까지만 보여주고 나머지는 "…"으로 자르는 CSS

예전 WebKit 전용 방식:
  display: -webkit-box;
  -webkit-box-orient: vertical;
  -webkit-line-clamp: 2;
  overflow: hidden;   ← clamp가 overflow: hidden을 요구

현재 표준 + WebKit 병행 (브라우저 호환 위해 둘 다 씀):
  line-clamp: 2;        ← 표준
  -webkit-line-clamp: 2;← WebKit 접두사
  overflow: hidden;     ← 필수
```

```css
/* 2줄에서 자르기 */
.bio-text {
  display: -webkit-box;
  -webkit-box-orient: vertical;
  -webkit-line-clamp: 2;
  line-clamp: 2;
  overflow: hidden;
}

/* 1줄 말줄임 (단일 행) */
.title {
  white-space:   nowrap;
  overflow:      hidden;
  text-overflow: ellipsis;
  /* line-clamp 아닌 text-overflow — 단일 행에서는 이게 더 간단 */
}
```

```txt
line-clamp vs text-overflow: ellipsis:
  text-overflow: ellipsis → 1줄에서 자를 때 (white-space: nowrap 필요)
  line-clamp: N           → N줄에서 자를 때 (display: -webkit-box 세트 필요)
```

---

# overflow: hidden이 ::after 꼬리를 clip하는 문제 ⭐️⭐️⭐️⭐️

## 증상

```txt
말풍선 ::after 꼬리(다이아몬드/삼각형)가 안 보임
→ 꼬리가 잘림
```

## 원인

```txt
.speech-bubble 에 line-clamp용 overflow: hidden
→ 박스 밖으로 나간 ::after 꼬리까지 clip(잘라냄)

overflow: hidden = "이 박스 밖으로 나가는 것은 전부 자름"
::after로 만든 꼬리 = 부모 박스 밖에 위치 (position: absolute)
→ 부모에 overflow: hidden 있으면 꼬리도 잘림
```

## 해결 — clamp 요소와 ::after를 분리

```css
/* ❌ 문제: 말풍선 자체에 overflow: hidden */
.speech-bubble {
  -webkit-line-clamp: 2;
  overflow: hidden;     /* ← ::after 꼬리도 잘림 */
  position: relative;
}
.speech-bubble::after {
  content:  '';
  position: absolute;
  bottom:   -8px;       /* 박스 밖 → overflow: hidden에 잘림 */
}

/* ✅ 해결: 텍스트는 안쪽 span에서, 말풍선 자체는 overflow: visible */
.speech-bubble {
  overflow: visible;    /* ← 꼬리 허용 */
  position: relative;
}
.speech-bubble::after {
  content:  '';
  position: absolute;
  bottom:   -8px;       /* 이제 잘리지 않음 */
}

.speech-bubble-text {
  display:            -webkit-box;
  -webkit-box-orient: vertical;
  -webkit-line-clamp: 2;
  overflow:           hidden;   /* ← 텍스트만 자름 */
}
```

```html
<!-- JSX: bio 문구를 안쪽 span으로 감쌈 -->
<div class="speech-bubble">
  <span class="speech-bubble-text">{bio}</span>
</div>
```

```txt
핵심 규칙:
  clamp(overflow: hidden)이 필요한 요소와
  ::after 장식(꼬리·뱃지 등)을 같은 박스에 두지 말 것

  해결 방법:
  부모 → overflow: visible  (꼬리 허용)
  자식 span → overflow: hidden + line-clamp (텍스트만 자름)
  ::after → 부모에 붙임 (잘리지 않음)
```

---

# ::after / ::before — 가상 요소로 장식 ⭐️⭐️⭐️

```css
/* 말풍선 꼬리 — ::after로 구현 */
.speech-bubble {
  position: relative;
  background: #fff;
  border-radius: 8px;
}

.speech-bubble::after {
  content:      '';         /* 반드시 있어야 함 (비어도 됨) */
  position:     absolute;   /* 부모 기준으로 위치 */
  bottom:       -8px;       /* 아래로 8px 나오게 */
  left:         50%;
  transform:    translateX(-50%);
  width:        16px;
  height:       16px;
  background:   inherit;    /* 부모 배경색 상속 */
  clip-path:    polygon(50% 100%, 0 0, 100% 0);  /* 삼각형 */
}
```

```txt
::after / ::before:
  CSS로 DOM 요소를 추가하지 않고 장식 요소를 만드는 방법
  content: '' 없으면 렌더링 안 됨
  position: absolute → 부모에 position: relative 필요

  ::before  → 요소 안 맨 앞에 생성
  ::after   → 요소 안 맨 뒤에 생성

꼬리가 잘리면:
  부모에 overflow: hidden 없는지 확인
  → overflow: visible 로 바꾸고 텍스트 자르기는 자식 span으로 이동
```

---

# 정리 — clamp + ::after 패턴

```txt
clamp가 필요한 요소에 ::after 꼬리도 달아야 할 때:

  부모 .bubble:
    overflow: visible   ← 꼬리 허용
    position: relative  ← ::after 기준점

  자식 .bubble-text:
    overflow: hidden    ← 텍스트만 자름
    -webkit-line-clamp: N

  .bubble::after:
    꼬리 스타일        ← 잘리지 않음

JSX:
  <div className="bubble">
    <span className="bubble-text">{text}</span>
  </div>
```