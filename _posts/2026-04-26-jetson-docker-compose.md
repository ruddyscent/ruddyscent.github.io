---
layout: post
title: Jetson Xavier NX에서 Docker Compose로 NGC 컨테이너 실행하기
subtitle: 권한 문제, Compose 플러그인, NVIDIA runtime까지
tags: [jetson, docker, docker-compose, nvidia, ngc, linux, xavier-nx]
cover-img: /assets/img/develop.jpeg
thumbnail-img: /assets/img/docker.webp
share-img: /assets/img/develop.jpeg
author: 전경원
---

Jetson에서 Docker 기반 개발 환경을 잡으려고 하면 초반부터 막히는 경우가 많습니다. 일반적인 Linux 서버처럼 바로 될 것 같지만, 실제로는 `docker` 권한 문제와 `docker compose` 플러그인 부재, NVIDIA runtime 설정이 한꺼번에 얽힙니다.

특히 NGC 컨테이너를 올리려면 GPU를 정상적으로 연결해야 하므로 Docker만 설치했다고 끝나지 않습니다. 이 글에서는 **Jetson Xavier NX에서 Docker Compose로 NGC 컨테이너를 실행할 때 필요한 핵심 설정**을 순서대로 정리합니다.

## 왜 Jetson에서 한 번에 안 되는가

문제는 보통 세 갈래로 나뉩니다.

- 현재 사용자가 Docker 데몬에 접근할 권한이 없습니다.
- `docker compose` 명령이 없거나 구버전 사용법이 섞여 있습니다.
- 컨테이너는 떠도 GPU를 잡지 못합니다.

Jetson에서 Compose가 안 되는 것처럼 보여도 실제 원인은 `Compose 자체`가 아니라 **권한**, **플러그인 설치 상태**, **NVIDIA runtime 설정** 가운데 하나인 경우가 많습니다.

## 1. Docker 권한 문제 해결

가장 먼저 현재 계정이 Docker 데몬에 접근할 수 있는지 확인합니다. 다음 오류가 나오면 권한 문제일 가능성이 높습니다.

```text
permission denied while trying to connect to the Docker daemon socket
```

이 경우 현재 사용자를 `docker` 그룹에 추가합니다.

```bash
sudo usermod -aG docker $USER
```

설정은 다음 둘 중 하나로 반영합니다.

```bash
newgrp docker
```

또는 SSH 세션을 끊고 다시 접속합니다. 적용한 뒤에는 Docker 정보가 정상적으로 보이는지 확인합니다.

```bash
docker info
```

여기서 더 이상 permission 오류가 나오지 않으면 첫 번째 문제는 해결된 것입니다.

## 2. Docker Compose 설치

Jetson 환경에서는 `docker compose`가 기본으로 바로 동작하지 않는 경우가 있습니다. 자주 보이는 증상은 두 가지입니다.

```text
docker-compose: command not found
```

또는 예전 사용법과 섞여 옵션 오류가 나기도 합니다.

```text
unknown shorthand flag: 'f' in -f
```

현재는 별도 바이너리인 `docker-compose`보다 **Docker Compose V2 플러그인**을 쓰는 편이 맞습니다. 다음처럼 설치합니다.

```bash
sudo apt-get update
sudo apt-get install docker-compose-plugin
```

설치한 뒤에는 버전을 확인합니다.

```bash
docker compose version
```

정상적으로 버전이 출력되면 이제 `docker compose ...` 형식으로 명령을 사용할 수 있습니다.

## 3. 플러그인 패키지 설치가 어려울 때 수동 설치

환경에 따라 패키지 저장소에서 `docker-compose-plugin`을 바로 설치하기 어려울 수도 있습니다. 이때는 사용자 로컬 CLI 플러그인 경로에 직접 넣을 수 있습니다.

```bash
mkdir -p ~/.docker/cli-plugins

curl -SL \
  https://github.com/docker/compose/releases/latest/download/docker-compose-linux-aarch64 \
  -o ~/.docker/cli-plugins/docker-compose

chmod +x ~/.docker/cli-plugins/docker-compose
```

Jetson Xavier NX는 ARM64 계열이므로 `docker-compose-linux-aarch64` 바이너리를 사용합니다. 설치한 뒤 다시 한 번 버전을 확인해 두는 편이 안전합니다.

```bash
docker compose version
```

## 4. NVIDIA runtime 확인

NGC 이미지를 쓰는 목적이 GPU 활용이라면 Compose가 실행되는 것만으로는 충분하지 않습니다. 컨테이너가 **NVIDIA runtime을 통해 GPU에 접근할 수 있어야** 합니다.

현재 Docker가 NVIDIA runtime을 인식하는지 확인합니다.

```bash
docker info | grep -i runtime
```

정상적인 경우에는 아래와 비슷한 정보가 보입니다.

```text
Runtimes: nvidia runc
Default Runtime: nvidia
```

`nvidia`가 보이지 않는다면 Compose 파일보다 먼저 Docker와 NVIDIA Container Runtime의 연동 상태를 확인해야 합니다. Jetson에서는 이 부분이 빠져 있으면 컨테이너는 떠도 GPU 관련 기능이 제대로 동작하지 않습니다.

## 5. Compose로 컨테이너 실행

기본 설정이 끝났다면 Compose 파일로 컨테이너를 실행할 수 있습니다.

```bash
docker compose -f compose.jetson.yml up -d
```

실행한 뒤 컨테이너 내부로 들어가려면 다음처럼 접속합니다.

```bash
docker compose -f compose.jetson.yml exec hoverpilot bash
```

여기서 `hoverpilot`은 서비스 이름 예시입니다. 실제 서비스 이름은 `compose.jetson.yml` 안의 정의에 맞게 바꾸면 됩니다.

## 6. Jetson용 Compose 파일에서 자주 챙기는 포인트

Jetson에서 GPU 관련 컨테이너를 안정적으로 올리려면 Compose 파일에서 몇 가지를 확인합니다.

- `host network` 사용 여부
- NVIDIA runtime 관련 설정
- NGC 이미지 사용 여부

예를 들어 Jetson 전용 Compose 파일은 아래처럼 구성할 수 있습니다.

```yaml
services:
  hoverpilot:
    image: nvcr.io/nvidia/l4t-pytorch:<tag>
    network_mode: host
    runtime: nvidia
    stdin_open: true
    tty: true
```

프로젝트에 따라 볼륨 마운트, 환경 변수, 장치 매핑이 더 들어갈 수 있지만 Jetson에서 먼저 확인할 핵심은 대체로 위 세 가지입니다.

## 7. 가장 흔한 실패 패턴

오류 메시지가 길어 보여도 실제 원인은 아래 셋 가운데 하나인 경우가 많습니다.

- `permission denied`가 나오면 대부분 Docker 권한 문제입니다.
- `docker compose`가 안 되면 Compose 플러그인 설치 여부를 먼저 봅니다.
- 컨테이너는 뜨는데 GPU가 잡히지 않으면 NVIDIA runtime 설정을 확인합니다.

이 순서대로 점검하면 Compose 파일 자체를 불필요하게 오래 의심하지 않아도 됩니다.

## 8. 정리

Jetson Xavier NX에서 Docker Compose로 NGC 컨테이너를 실행하려면 세 가지가 맞아야 합니다.

- 사용자가 Docker 데몬에 접근할 수 있어야 합니다.
- `docker compose` V2 플러그인이 준비되어 있어야 합니다.
- Docker가 NVIDIA runtime을 인식해야 합니다.

Jetson에서는 Docker만 설치했다고 바로 개발 환경이 완성되지는 않습니다. 하지만 위 세 가지를 순서대로 맞추면 이후에는 `compose.jetson.yml` 기반으로 재현성 있게 환경을 올릴 수 있습니다. 초기에 자주 막히는 부분을 정리해 두면, 다음부터 NGC 컨테이너를 실행하는 일은 단순해집니다.
