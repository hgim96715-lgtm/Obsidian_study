---
title: Portfolio — 김한경
cssclasses:
  - portfolio
related:
  - "[[00_Career_HomePage]]"
  - "[[이력서]]"
  - "[[00_Project_HomePage]]"
---



<div class="pf-hero">

# 김한경

<p class="pf-greeting">안녕하세요 🙂</p>

<p class="pf-headline">사람이 손으로 하던 일을 서비스로 옮기고,<br>데이터가 제대로 쌓이게 만드는 개발자</p>

</div>

<p class="pf-intro">퍼블리셔로 일하며 문제 데이터 구조화(Array/JSON)와 사용자 입력 검증 로직을 직접 구현했습니다. 화면을 만드는 일이었지만 결국 <strong>데이터를 어떻게 설계하고 처리할 것인가</strong>를 계속 고민했고, 그 질문이 지금의 개발 공부로 이어졌습니다.</p>

<p class="pf-intro pf-intro-sub">지금은 <strong>Next.js · NestJS · TypeScript</strong>로 UI와 API를 모노레포로 이어 붙이고, Prisma · PostgreSQL로 데이터가 끝까지 쌓이게 만드는 쪽에 집중합니다.</p>

<p class="pf-contact">📞 010-4737-1692 · ✉️ <a href="mailto:hgim96715@gmail.com">hgim96715@gmail.com</a> · 🗂 <a href="https://github.com/hgim96715-lgtm">GitHub</a> · 🖼 <a href="https://artinerary-web.vercel.app/">Artinerary Live</a></p>

---

## Experience

<div class="pf-exp">

### 일상이지 · 퍼블리셔

`2024.04 – 2025.08`

- 교육 콘텐츠 스토리보드 분석 후 HTML · CSS · JavaScript 기반 **인터랙티브 미니게임** 구현 — 클릭·입력·선택에 따른 화면 상태 변경
- Array/JSON 기반 **문제 데이터 구성**, 사용자 **정답 검증 로직** 직접 구현
- 기획서 요구사항을 화면 동작 단위로 쪼개며 **데이터 구조와 UI가 연결되는 흐름** 경험

<p class="pf-exp-note">→ 이 경험이 데이터 처리와 API 설계에 관심을 갖게 된 계기가 됐습니다.</p>

</div>

---

## Projects

<div class="pf-project">

### 🎵 Music Community

*추천 이유와 분위기로 곡을 나누는 소셜 음악 커뮤니티*

알고리즘 피드가 아닌 **추천 → embed 청취 → 반응** 흐름을 목표로 한 MVP. pnpm 모노레포에서 NestJS가 도메인·인증·집계를, Next.js가 UI·REST 호출을 담당합니다. 피드, JWT 가입/로그인, SavedCard 커스텀 앨범, Admin 운영(통계·사용자·DAU)까지 구현하고 **Vercel · Railway · Neon**으로 배포했습니다.

<p class="pf-stack"><code>pnpm</code> <code>NestJS</code> <code>Next.js</code> <code>TypeScript</code> <code>Prisma</code> <code>PostgreSQL</code> <code>JWT</code> <code>RBAC</code> <code>Docker</code> <code>Vercel</code> <code>Railway</code> <code>Neon</code></p>

<p class="pf-links"><a href="https://music-community-web.vercel.app/">Demo</a> · <a href="https://github.com/hgim96715-lgtm/music-community">GitHub</a></p>

</div>

---

<div class="pf-project">

### 🖼️ Artinerary

*공공 문화 API 기반 전시 큐레이션 서비스*

문화누리 XML을 수집·정규화한 **전시 큐레이션** 서비스. 관리자 보정·수집, 사용자 **찜·관람 기록**. 외부 XML API 연동, JWT(httpOnly) 인증, 역할 분리.

<p class="pf-stack"><code>NestJS</code> <code>Next.js</code> <code>TypeScript</code> <code>PostgreSQL</code> <code>Docker</code> <code>Public API (XML)</code> <code>JWT</code></p>

<p class="pf-links"><a href="https://artinerary-web.vercel.app/">Demo</a> · <a href="https://github.com/hgim96715-lgtm/artinerary">GitHub</a></p>

</div>

---

<div class="pf-project">

### 🎬 영화 도메인 NestJS API

*NESTJS-PROJECT-PIPELINE*

영화·감독·장르 REST API — 인증·인가, 트랜잭션, 파일 업로드, Socket.IO 채팅, **AWS 배포**까지. TypeORM 도메인 구현 후 **Prisma로 점진 전환** (두 ORM 공존 경험).

<p class="pf-stack"><code>NestJS</code> <code>TypeScript</code> <code>PostgreSQL</code> <code>TypeORM</code> <code>Prisma</code> <code>JWT</code> <code>Socket.IO</code> <code>Redis</code> <code>Docker</code> <code>AWS</code> <code>GitHub Actions</code> <code>Jest</code></p>

<p class="pf-links"><a href="https://github.com/hgim96715-lgtm/nestjs-project-pipeline">GitHub</a></p>

</div>

---

<div class="pf-project">

### 🚑 응급실 실시간 병상 현황

*역설계 프로젝트*

응급·소방 실습 경험을 바탕으로, 119–병원 간 **실시간 병상 공유**를 데이터 엔지니어 관점에서 역설계.

<p class="pf-stack"><code>Python</code> <code>PostgreSQL</code> <code>Public API</code> <code>Streamlit</code></p>

<p class="pf-links"><a href="https://github.com/hgim96715-lgtm/hospital-realtime-pipeline">GitHub</a></p>

</div>

---

<div class="pf-project">

### 🚆 Seoul Station Train Dashboard

*공공데이터 실시간 파이프라인*

서울역 열차 운행 데이터를 공공 API로 수집·적재하고, **실시간 현황·지연**을 시각화.

<p class="pf-stack"><code>Python</code> <code>PostgreSQL</code> <code>Public API</code> <code>Streamlit</code></p>

<p class="pf-links"><a href="https://github.com/hgim96715-lgtm/seoul-realtime-train-pipeline">GitHub</a></p>

</div>

---

## Skills

| | |
|---|---|
| **Backend** | NestJS · TypeScript · REST API · JWT · RBAC · Swagger · Jest |
| **Database** | PostgreSQL · Prisma · TypeORM · SQL · ER 모델링 · migrate |
| **Frontend** | Next.js · React · TypeScript · Tailwind |
| **Infra** | Docker · Vercel · Railway · Neon · AWS (EB/RDS/S3) · Git · GitHub Actions |
| **Data** | Python · Pandas · 공공 API · Streamlit · Kafka · Spark · Airflow (학습) |

---

## Education & Certificates

| | |
|---|---|
| **경일대학교** 응급구조학과 | `2014.02 – 2018.02` |
| **SQLD** · 한국데이터산업진흥원 | `2026.03` |
| **정보처리기능사** · 한국산업인력공단 | `2023.04` |

---

## Technical Writing

Obsidian으로 공부·정리하는 개인 위키 **gong_study**. 개념 암기보다 **동작 원리와 흐름** 중심으로, 프론트와 백엔드 연결·배포·운영까지 프로젝트에 반영하며 기록합니다.

<p class="pf-links"><a href="https://github.com/hgim96715-lgtm/Obsidian_study">GitHub — Obsidian_study</a></p>
