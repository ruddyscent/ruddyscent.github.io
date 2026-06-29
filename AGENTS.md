# T[h]inkering 블로그 에이전트 가이드

## 블로그 개요

- **블로그명**: T[h]inkering
- **URL**: ruddyscent.github.io
- **저자**: 전경원 (Kyungwon Chun)
- **플랫폼**: GitHub Pages + Beautiful Jekyll
- **언어**: 한국어 (포스트 전체)

## 포스트 frontmatter 규칙

```yaml
---
layout: post
title: 제목
subtitle: 부제목
tags: [tag1, tag2]
cover-img: /assets/img/develop.jpeg
thumbnail-img: /assets/img/TOPIC.webp
share-img: /assets/img/develop.jpeg
author: 전경원
---
```

- `cover-img`와 `share-img`는 기본적으로 `/assets/img/develop.jpeg`를 쓴다.
- `thumbnail-img`는 주제에 따라 다르다:
  - C#: `csharp.webp`
  - Git: `git.webp`
  - Docker/DevContainer: `docker.webp`
  - VSCode/IDE: `vscode.webp`
  - Python/ML: `python.webp`
  - HoverPilot/RC: `toy-plane.webp`
  - 기타(기본): `develop.jpeg`
- SEO가 필요한 포스트는 `description` 필드를 추가한다.
- `author`는 항상 `전경원`.

## subtitle 작성 원칙

- 짧고 직접적으로, 사람이 쓴 것처럼 쓴다.
- 절차 설명("~부터 확인한다", "~하는 방법") 형식은 AI 슬롭처럼 보이므로 피한다.
- 포스트 본문 첫 문장의 맥락과 자연스럽게 이어지도록 쓴다.
- 좋은 예: "Xcode의 문제인 줄 알았지만, 원인은 Codex CLI 설정 파일에 있었다"
- 나쁜 예: "ChatGPT 계정 로그인만 실패할 때는 Codex CLI 전역 설정부터 확인한다"

## 주요 콘텐츠 카테고리

### HoverPilot 시리즈

RC 비행기를 강화학습으로 조종하는 프로젝트. 태그: `hoverpilot`.

- RealFlight 시뮬레이터 연동 (RFLink, SOAP)
- Gymnasium 환경 래핑
- Reward 설계
- PPO 구현 및 TensorBoard 로깅

### C# / 코딩테스트

프로그래머스 PCCP 기반. 태그: `csharp`, `pccp`, `codingtest`.

- 런타임 환경, 배열, 정렬, Length vs Count 등

### Codex / AI 도구 문제 해결

Codex CLI 로그인 이슈 3부작. 태그: `codex`, `troubleshooting`.

- DevContainer (포트 충돌, 자격증명 마운트)
- Remote-SSH (OAuth 콜백 localhost 불일치)
- Xcode (Codex CLI 전역 설정 파일 충돌)

### 개발 환경

- Jekyll DevContainer 로컬 개발 환경
- Jetson Xavier NX Docker Compose / NGC
- Apple Silicon PyTorch + uv 설치

### Python / ML

- 파이썬 장식자
- nbdev + git hooks (Jupyter 노트북 diff 정리)
- RL Book 코세라 강의 동반 정리

### Git

- 커밋 접두사(Conventional Commits) 규칙
- 자주 쓰는 Git 명령어

### 기타

- Zed vs VSCode (ML 엔지니어 관점)
- FDTD 시뮬레이션 (GPU 가속, PHANTOM 프로젝트)
- 점프와 순간이동 — 그리디 알고리즘

## 파일 구조

```text
_posts/         # YYYY-MM-DD-slug.md 형식
assets/img/     # 이미지 (webp 권장)
aboutme.md      # 소개 페이지
_config.yml     # 사이트 설정
```

## 포스트 날짜

일부 포스트는 미래 날짜(2027-)로 예약되어 있다. GitHub Pages는 미래 날짜 포스트를 빌드 시점 기준으로 공개하므로, 날짜가 미래여도 포스트 파일이 `_posts/`에 있으면 빌드에 포함된다.
