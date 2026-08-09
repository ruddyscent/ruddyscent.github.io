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

VSCode의 DevContainer를 활용하면 Jekyll 블로그 개발 환경을 손쉽게 구축할 수 있다. 이 글에서는 DevContainer로 Jekyll 블로그를 로컬에서 실행하는 방법을 단계별로 설명한다. 블로그는 Beautiful Jekyll 템플릿을 이용하고 GitHub Pages에서 호스팅한다고 가정한다.

## 1. DevContainer 설정 파일 생성

VSCode를 실행하고 F1을 눌러 커맨드 팔레트를 연다. "Dev Containers: Open Folder in Container"를 선택하여 Jekyll 블로그 프로젝트 폴더를 열면 된다.

다른 옵션은 모두 기본 상태로 두고 "Add Dev Container Configuration Files"만 "Jekyll"을 선택한다. 그러면 `.devcontainer` 폴더가 만들어지고 그 안에 `devcontainer.json`이 생성된다. 기본적인 설정은 아래와 같다. 커맨드 팔레트에서 옵션을 선택할 필요 없이 프로젝트 폴더에 `.devcontainer/devcontainer.json` 파일을 생성하고 아래 내용을 붙여넣어도 된다.

```json
{
	"name": "Jekyll",
	"image": "mcr.microsoft.com/devcontainers/jekyll:2-bullseye"
}
```

## 2. 컨테이너 빌드 및 실행
모든 선택을 마치고 나면 Docker 이미지가 빌드된다. 최초 실행 시 시간이 걸릴 수 있다. 빌드가 완료되면 컨테이너가 실행되고 VSCode가 자동으로 컨테이너에 연결된다.

## 3. Jekyll 서버 실행

Ctrl+` 키를 눌러 터미널을 열고 Jekyll 서버를 구동한다.

```bash
bundle exec jekyll serve --host 0.0.0.0 --livereload --future
```
사용한 옵션의 의미는 각각 이렇다.
- `--host 0.0.0.0`: 컨테이너 외부(로컬 호스트)에서 접속할 수 있도록 허용하는 필수 옵션이다.
- `--livereload`: 파일 수정 시 브라우저가 자동으로 새로고침한다.
- `--future`: 미래 날짜로 설정된 게시물도 포함하여 빌드한다.

## 4. 브라우저에서 접속

이제 브라우저에서 `http://localhost:4000`으로 접속하면 Jekyll 블로그가 정상적으로 실행된다. 간단히 VSCode 우측 하단에 나타난 녹색 "Open in Browser" 버튼을 클릭해도 블로그에 접속할 수 있다.