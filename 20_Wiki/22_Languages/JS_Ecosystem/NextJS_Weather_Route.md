---
aliases:
  - 날씨
  - 역지오코딩
  - geolocation
  - open-meteo
  - weather
  - WMO
tags:
  - NextJS
related:
  - "[[00_JS_Ecosystem_HomePage]]"
  - "[[JS_BrowserAPI]]"
  - "[[NextJS_Server_Actions]]"
---

# Next_Weather_Route — 날씨 API 프록시

# 전체 흐름

```bash
Home (서버 컴포넌트)
  └─ WeatherByLocation (클라이언트 컴포넌트)
        1. navigator.geolocation.getCurrentPosition
             → lat(위도), lon(경도) 획득
        2. fetch('/api/weather?lat=&lon=')
             → Next.js route.ts (서버)
        3. open-meteo API 호출
             → temperature / weather_code / humidity
        4. 날씨 카드 표시

# → [[JS_BrowserAPI#navigator.geolocation]] 참고
```

---

---

# Open-Meteo API ⭐️

```txt
https://api.open-meteo.com

무료 / API 키 불필요
위도·경도 기반 날씨 제공
WMO 표준 날씨 코드 사용
```

## URL 구조 ⭐️

```txt
https://api.open-meteo.com/v1/forecast
  ?latitude=37.5665          ← 위도 (서울)
  &longitude=126.9780        ← 경도 (서울)
  &current=temperature_2m,weather_code,relative_humidity_2m
  ↑ 현재 날씨 데이터 항목 콤마로 나열
  &timezone=Asia%2FSeoul     ← 타임존 (%2F = /)
```

## current 파라미터 항목 ⭐️

```txt
temperature_2m:
  지면에서 2m 높이의 기온 (섭씨)
  "2m" = 기상 관측 표준 높이 (사람 키 높이 정도)
  → 우리가 체감하는 온도와 가장 가까운 높이

weather_code:
  WMO(세계기상기구) 표준 날씨 코드
  숫자 하나로 맑음/흐림/비/눈 표현
  → weatherLabel() 함수로 한글로 변환

relative_humidity_2m:
  2m 높이의 상대 습도 (0~100%)
  relative_humidity = 상대 습도
```

## WMO 날씨 코드 ⭐️

```txt
0        맑음
1, 2, 3  주로 맑음 / 구름 조금 / 구름 많음
45, 48   안개
51~67    이슬비 / 비
71~77    눈
80~82    소나기
95~99    뇌우
```

```typescript
// WMO 코드 → 한글 변환
const weatherLabel = (code: number) => {
  if (code === 0)    return '맑음';
  if (code <= 3)     return '구름';
  if (code <= 48)    return '흐림';
  if (code <= 67)    return '비';
  if (code <= 77)    return '눈';
  if (code <= 82)    return '소나기';
  return '날씨';
};
```

---

---

# 역지오코딩 — 좌표 → 지역명 ⭐️

```txt
역지오코딩 (Reverse Geocoding):
  위도/경도 → 사람이 읽을 수 있는 주소로 변환
  lat: 37.5665 / lon: 126.9780 → "서울 강남구"

Nominatim (OpenStreetMap):
  무료 / API 키 불필요
  한국어 응답 지원 (accept-language=ko)
  ⚠️ User-Agent 헤더 필수 (없으면 요청 거부)
  캐시 권장 (같은 좌표 반복 호출 방지)
```

## Nominatim API URL 구조

```txt
https://nominatim.openstreetmap.org/reverse
  ?lat=37.5665          ← 위도
  &lon=126.9780         ← 경도
  &format=json          ← JSON 응답
  &accept-language=ko   ← 한국어 주소
```

## 지역명 추출 함수

```typescript
// 특별시 / 광역시 등 긴 행정명 축약
const shortenRegion = (name: string) =>
  name.replace(/특별시|광역시|특별자치시|특별자치도/g, '').trim();
// "서울특별시" → "서울"
// "부산광역시" → "부산"

const resolveLocationName = async (
  lat: string,
  lon: string
): Promise<string | null> => {
  try {
    const geoUrl =
      `https://nominatim.openstreetmap.org/reverse?lat=${lat}&lon=${lon}` +
      `&format=json&accept-language=ko`;

    const geoRes = await fetch(geoUrl, {
      headers: {
        // ⚠️ Nominatim 정책: User-Agent 필수
        'User-Agent': 'Artinerary/1.0 (https://artinerary-web.vercel.app)',
      },
      next: { revalidate: 3600 },   // 1시간 캐시 (좌표 → 주소는 잘 안 바뀜)
    });

    if (!geoRes.ok) return null;

    const geo = (await geoRes.json()) as {
      address?: {
        city?:    string;   // 서울특별시 / 부산광역시 (광역)
        town?:    string;   // 중소 도시
        county?:  string;   // 군 단위
        borough?: string;   // 자치구 (강남구 / 종로구)
        suburb?:  string;   // 동 단위
        state?:   string;   // 도 단위 (경기도)
      };
    };

    const { city, town, county, borough, suburb, state } = geo.address ?? {};

    const region   = city ?? town ?? county ?? state;   // 큰 단위
    const district = borough ?? suburb;                  // 작은 단위

    // "서울 강남구" 형태로 조합
    if (region && district && district !== region) {
      return `${shortenRegion(region)} ${district}`;
    }
    if (region)   return shortenRegion(region);
    if (district) return district;
    return null;
  } catch {
    return null;    // 실패해도 날씨는 표시 (위치명만 없음)
  }
};
```

```txt
address 필드 우선순위:
  region   city → town → county → state  (큰 단위부터)
  district borough → suburb              (작은 단위)

  서울 강남구:  city = "서울특별시" / borough = "강남구"
               → shortenRegion("서울특별시") + " " + "강남구"
               → "서울 강남구"

  경기 성남시:  state = "경기도" / city = "성남시"
               → "성남시" (city 가 있으면 state 무시)

  결과 없으면 null 반환 → 날씨는 표시 / 위치명만 숨김

User-Agent 필수 이유:
  Nominatim 이용 약관: 앱 식별자 포함 필수
  없으면 403 또는 요청 차단
  형식: "앱이름/버전 (URL)"
```

## route.ts 에서 함께 사용

```typescript
// app/api/weather/route.ts
export async function GET(req: NextRequest) {
  const lat = req.nextUrl.searchParams.get('lat');
  const lon = req.nextUrl.searchParams.get('lon');

  if (!lat || !lon) {
    return NextResponse.json({ message: 'lat, lon 필요' }, { status: 400 });
  }

  // 날씨 + 위치명 병렬 요청
  const [weatherRes, locationName] = await Promise.all([
    fetch(`https://api.open-meteo.com/v1/forecast?...`, { next: { revalidate: 300 } }),
    resolveLocationName(lat, lon),   // 실패해도 null 반환
  ]);

  const data = await weatherRes.json();

  return NextResponse.json({
    temperature:  data.current.temperature_2m,
    weather:      weatherLabel(data.current.weather_code),
    humidity:     data.current.relative_humidity_2m,
    locationName,   // "서울 강남구" 또는 null
  });
}
```

```txt
Promise.all 로 병렬 요청:
  날씨 API + 역지오코딩 동시에
  순차 요청보다 빠름

resolveLocationName 이 null 반환해도:
  날씨 데이터는 정상 응답
  locationName: null → 클라이언트에서 위치명만 숨김
```

```typescript
// app/api/weather/route.ts
import { NextRequest, NextResponse } from 'next/server';

// WMO 코드 → 한글 변환
const weatherLabel = (code: number) => {
  if (code === 0)  return '맑음';
  if (code <= 3)   return '구름';
  if (code <= 48)  return '흐림';
  if (code <= 67)  return '비';
  if (code <= 77)  return '눈';
  if (code <= 82)  return '소나기';
  return '날씨';
};

export async function GET(req: NextRequest) {
  // 1. 쿼리 파라미터 추출
  const lat = req.nextUrl.searchParams.get('lat');
  const lon = req.nextUrl.searchParams.get('lon');

  if (!lat || !lon) {
    return NextResponse.json(
      { message: 'lat, lon 이 필요합니다.' },
      { status: 400 },
    );
  }

  // 2. Open-Meteo API URL 구성
  const url =
    `https://api.open-meteo.com/v1/forecast?latitude=${lat}&longitude=${lon}` +
    `&current=temperature_2m,weather_code,relative_humidity_2m` +
    `&timezone=Asia%2FSeoul`;

  // 3. 외부 API 호출 (5분 캐시)
  const res = await fetch(url, { next: { revalidate: 300 } });

  if (!res.ok) {
    return NextResponse.json(
      { message: '날씨 정보를 가져오는데 실패했습니다.' },
      { status: 502 },
    );
  }

  // 4. 응답 타입 명시
  const data = (await res.json()) as {
    current: {
      temperature_2m:       number;
      weather_code:         number;
      relative_humidity_2m: number;
    };
  };

  // 5. 필요한 데이터만 가공해서 반환
  return NextResponse.json({
    temperature: data.current.temperature_2m,
    weather:     weatherLabel(data.current.weather_code),
    humidity:    data.current.relative_humidity_2m,
  });
}
```

```txt
{ next: { revalidate: 300 } }:
  Next.js 캐시 옵션
  300초(5분) 동안 같은 응답 캐시
  → 같은 위치 반복 요청 시 외부 API 재호출 안 함

as { current: { ... } }:
  fetch 응답은 any 타입
  as 로 응답 구조 타입 명시
  → data.current.temperature_2m 자동완성 + 타입 안전

status: 502:
  502 Bad Gateway
  우리 서버는 정상이지만 외부 API(Open-Meteo) 가 실패했을 때
  → 우리 서버 에러 500 이 아님
```

---

---

# 클라이언트 컴포넌트

```typescript
// components/WeatherByLocation.tsx
'use client';

import { useEffect, useState } from 'react';

type Weather = {
  temperature: number;
  weather:     string;
  humidity:    number;
};

export default function WeatherByLocation() {
  const [weather, setWeather] = useState<Weather | null>(null);
  const [error,   setError]   = useState<string | null>(null);

  useEffect(() => {
    if (!navigator.geolocation) {
      setError('위치 정보를 지원하지 않는 브라우저입니다.');
      return;
    }

    navigator.geolocation.getCurrentPosition(
      async (pos) => {
        const { latitude: lat, longitude: lon } = pos.coords;
        const res  = await fetch(`/api/weather?lat=${lat}&lon=${lon}`);
        const data = await res.json();
        setWeather(data);
      },
      () => setError('위치 권한이 필요합니다.'),
    );
  }, []);

  if (error)    return <p className="text-sm text-gray-400">{error}</p>;
  if (!weather) return <p className="text-sm text-gray-400">날씨 로딩 중...</p>;

  return (
    <div className="text-sm text-gray-600">
      {weather.weather} {weather.temperature}°C · 습도 {weather.humidity}%
    </div>
  );
}
```

---

---

# 한눈에

|항목|값|
|---|---|
|API|Open-Meteo (무료 / 키 없음)|
|엔드포인트|`GET /api/weather?lat=&lon=`|
|캐시|`revalidate: 300` (5분)|
|온도|`temperature_2m` (2m 높이 기온)|
|날씨|`weather_code` (WMO 코드 → 한글 변환)|
|습도|`relative_humidity_2m`|
|에러|파라미터 없음 400 / 외부 API 실패 502|