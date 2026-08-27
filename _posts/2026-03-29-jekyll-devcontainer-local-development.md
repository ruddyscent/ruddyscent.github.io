---
layout: post
title: VSCode의 DevContainer로 Jekyll 블로그 로컬 서버 띄우기
subtitle: 환경 설정 없이 바로 실행하는 재현 가능한 블로그 개발 환경
description: VSCode DevContainer를 활용해 Jekyll 블로그를 로컬에서 실행하고 테스트하는 방법을 단계별로 정리한다.
cover-img: /assets/img/develop.jpeg
thumbnail-img: /assets/img/docker.webp
share-img: /assets/img/develop.jpeg
tags: [jekyll, devcontainer, vscode, beautiful-jekyll, github-pages]
author: 전경원
---

VSCode의 DevContainer를 활용하면 Jekyll 블로그 개발 환경을 쉽게 구성할 수 있습니다. 이 글에서는 DevContainer로 Jekyll 블로그를 로컬에서 실행합니다. 블로그는 Beautiful Jekyll 템플릿을 이용하고 GitHub Pages에서 호스팅한다고 가정합니다.

## 1. DevContainer 설정 파일 생성

VSCode를 실행하고 F1을 눌러 커맨드 팔레트를 엽니다. "Dev Containers: Open Folder in Container"를 선택해 Jekyll 블로그 프로젝트 폴더를 엽니다.

다른 옵션은 기본값으로 두고 "Add Dev Container Configuration Files"에서 "Jekyll"만 선택합니다. 그러면 `.devcontainer` 폴더와 그 안의 `devcontainer.json`이 생성됩니다. 기본 설정은 아래와 같습니다. 커맨드 팔레트에서 옵션을 고르지 않고 프로젝트 폴더에 `.devcontainer/devcontainer.json` 파일을 만든 뒤 아래 내용을 붙여넣어도 됩니다.

```json
{
	"name": "Jekyll",
	"image": "mcr.microsoft.com/devcontainers/jekyll:2-bullseye"
}
```

## 2. 컨테이너 빌드 및 실행
선택을 마치면 Docker 이미지가 빌드됩니다. 최초 실행에는 시간이 걸릴 수 있습니다. 빌드가 완료되면 컨테이너가 실행되고 VSCode가 자동으로 컨테이너에 연결됩니다.

## 3. Jekyll 서버 실행

Ctrl+` 키를 눌러 터미널을 열고 Jekyll 서버를 실행합니다.

```bash
bundle exec jekyll serve --host 0.0.0.0 --livereload --future
```
사용한 옵션의 의미는 각각 이렇다.
- `--host 0.0.0.0`: 컨테이너 외부(로컬 호스트)에서 접속할 수 있도록 허용하는 필수 옵션입니다.
- `--livereload`: 파일 수정 시 브라우저가 자동으로 새로고침합니다.
- `--future`: 미래 날짜로 설정된 게시물도 포함하여 빌드합니다.

## 4. 브라우저에서 접속

이제 브라우저에서 `http://localhost:4000`으로 접속하면 Jekyll 블로그가 실행됩니다. VSCode 오른쪽 아래에 나타나는 녹색 "Open in Browser" 버튼을 클릭해도 블로그에 접속할 수 있습니다.
