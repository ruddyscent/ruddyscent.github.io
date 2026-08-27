---
layout: post
title: Dev Container에서 Codex 로그인 에러의 두 가지 해결책
subtitle: 다중 컨테이너 포트 충돌과 자격증명 마운트까지
tags: [vscode, devcontainer, codex, openai, docker, oauth, troubleshooting]
cover-img: /assets/img/develop.jpeg
thumbnail-img: /assets/img/vscode.webp
share-img: /assets/img/develop.jpeg
author: 전경원
---

VSCode의 Dev Container 안에서 Codex를 쓰기 시작했습니다. 그런데 컨테이너 수가 늘면서 로그인 문제가 생겼습니다. 여러 Dev Container가 같은 OAuth 콜백 포트를 포워딩하려다 충돌하고 있었습니다.

## 에러 상황

여러 컨테이너가 돌아가는 상황에서도 로그인 과정은 똑같습니다.

- ID 입력
- 비밀번호 입력
- OTP 입력

여기까지는 잘 진행됐습니다. 마지막에 브라우저가 열리고 로그인 완료 화면이 나와야 했지만 화면에는 에러가 떴습니다.

```text
ERR_CONNECTION_REFUSED
```

주소는 이랬습니다.

```text
http://localhost:1455/auth/callback?code=...
```

![브라우저에서 Codex OAuth 콜백 주소인 localhost:1455에 접속하지 못해 ERR_CONNECTION_REFUSED가 표시된 화면](/assets/img/posts/2026-05-01-devcontainer-codex-login-troubleshooting/login-error-connection-refused.png)

이 에러를 분석하면서 **브라우저가 돌아올 호스트 포트와 컨테이너 안의 콜백 포트를 어떻게 연결하느냐**가 문제라는 것을 알게 됐습니다.

## 여러 컨테이너에서 1455 포트가 충돌한다

먼저 짚고 갈 점이 있습니다. 호스트의 `1455`와 컨테이너 내부의 `1455`는 같은 숫자지만 같은 포트는 아닙니다.

- 브라우저는 호스트 OS에서 돈다
- Codex는 Dev Container 안에서 돈다
- 컨테이너마다 내부 네트워크 공간은 따로 있다

같은 `localhost`처럼 보여도 가리키는 곳은 다릅니다.

```text
브라우저 → localhost:1455 (호스트)
Codex   → localhost:1455 (컨테이너)
```

그래서 Dev Container에서 Codex에 로그인하려면 보통 호스트의 `localhost:1455`를 컨테이너 내부의 `1455`로 이어야 합니다. `.devcontainer/devcontainer.json`에는 이렇게 적을 수 있습니다.

```json
{
  "forwardPorts": [1455],
  "portsAttributes": {
    "1455": {
      "label": "Codex Auth Callback",
      "onAutoForward": "openBrowser"
    }
  }
}
```

이 자체는 문제가 아닙니다. 호스트 `1455`와 컨테이너 `1455`를 연결하는 것은 정상적인 포트 포워딩입니다. 문제는 Dev Container를 동시에 여러 개 띄울 때 생깁니다. 상황은 이렇습니다.

- 첫 번째 Dev Container의 포워딩이 이미 호스트 1455를 잡고 있다
- 두 번째 Dev Container에서 또 로그인을 시도한다
- 브라우저는 여전히 `localhost:1455`로 돌아오려고 한다

컨테이너마다 내부 `1455`는 따로 존재합니다. 하지만 호스트의 `localhost:1455`는 하나뿐입니다.

```text
호스트 localhost:1455 → 컨테이너 A:1455
호스트 localhost:1455 → 컨테이너 B:1455
```

먼저 띄운 컨테이너가 포트를 잡고 있으면 두 번째 컨테이너의 콜백은 엉뚱한 곳으로 갑니다. 이 경우는 `forwardPorts` 추가만으로 풀리지 않습니다.

### 점유 확인과 세션 정리

#### 1. 누가 1455를 잡고 있는지 확인

호스트에서 다음 명령으로 확인합니다.

```bash
lsof -i :1455
```

다른 프로세스나 세션이 이미 잡고 있다면 두 번째 콜백은 정상적으로 붙기 어렵습니다.

#### 2. 이전 세션 정리

- 먼저 띄운 Dev Container의 Codex 세션 종료
- 필요하면 그 VSCode 창도 닫기
- 포워딩 세션 정리
- 다시 로그인 시도

필요하면 점유 중인 프로세스를 직접 종료해야 할 수도 있습니다.

```bash
kill -9 <PID>
```

#### 3. Ports 뷰에서 실제 매핑 확인

포트가 열렸다고 끝이 아닙니다. 브라우저는 `localhost:1455`로 돌아오기 때문에 호스트 쪽도 정확히 1455여야 합니다.

```text
1455 → 1455   정상 가능성 높음
1455 → 1456   콜백 실패 가능
```

포워딩이 된 것처럼 보여도 호스트 쪽 포트 번호가 달라지면 인증 경로가 깨집니다.

#### 4. 로그인은 한 컨테이너씩

여러 Dev Container를 동시에 띄워야 한다면 적어도 Codex 로그인만큼은 한 번에 하나씩 처리해야 합니다.

1. 첫 번째 컨테이너에서 로그인 완료
2. 필요 없는 세션 정리
3. 두 번째 컨테이너에서 로그인 진행

이렇게 해야 두 번째 컨테이너에서도 정상적으로 포트를 포워딩합니다.

![여러 Dev Container가 호스트의 localhost 1455 포트를 두고 충돌하는 구조](/assets/img/posts/2026-05-01-devcontainer-codex-login-troubleshooting/codex-devcontainer-port-conflict-handdrawn.svg)

## 더 나은 해결책: 자격증명을 마운트해서 OAuth 자체를 건너뛴다

> 애초에 컨테이너 안에서 OAuth를 다시 할 필요가 있을까?

호스트에서 이미 한 번 로그인했다면 그 자격증명을 컨테이너에서 그대로 쓰면 됩니다.

### Codex의 자격증명 위치

Codex CLI는 홈 디렉터리 아래에 인증 정보를 모아둡니다.

```text
~/.codex/
├── auth.json        # OAuth 토큰
├── config.toml      # 설정
└── sessions/        # 대화 히스토리
```

이 중 `auth.json` 하나가 로그인 상태를 결정합니다.

### devcontainer.json에 마운트 추가

호스트의 `~/.codex` 디렉터리를 컨테이너 홈 안으로 바인드 마운트합니다.

```jsonc
{
  "mounts": [
    "source=${localEnv:HOME}/.codex,target=/home/vscode/.codex,type=bind,consistency=cached"
  ]
}
```

여기서 `target`의 `/home/vscode`는 하드코딩해야 안전합니다. `mounts`는 컨테이너가 부팅되기 전에 Docker가 처리합니다. 이 시점에는 컨테이너 내부의 `$HOME`이 아직 설정되지 않아 변수를 그대로 쓰면 엉뚱한 경로로 마운트할 수 있습니다.

![호스트의 ~/.codex 디렉터리를 여러 Dev Container의 /home/vscode/.codex로 바인드 마운트해 OAuth 콜백 없이 같은 Codex 자격증명을 공유하는 구조](/assets/img/posts/2026-05-01-devcontainer-codex-login-troubleshooting/codex-devcontainer-auth-mount-handdrawn.svg)

이렇게 하면 컨테이너에서 `codex`를 실행해도 호스트의 토큰을 그대로 씁니다.

- OAuth 콜백 자체가 일어나지 않는다
- 1455 포트와 무관하게 돌아간다
- Rebuild 후에도 다시 로그인할 필요가 없다

### 알아둘 것: 권한과 토큰 갱신

#### 1. UID 차이로 인한 권한 문제

호스트(macOS)와 컨테이너(Linux)의 UID가 다르면 컨테이너에서 `auth.json`을 쓰지 못할 수 있습니다. 이 경우 컨테이너 안에서 한 번 소유자를 바꾸면 됩니다.

```bash
sudo chown -R $(id -u):$(id -g) ~/.codex
```

#### 2. 토큰 갱신 충돌

호스트와 컨테이너에서 동시에 Codex를 활발히 쓰면 양쪽이 각자 토큰을 갱신하다가 서로 덮어쓸 수 있습니다. 이 점이 신경 쓰이면 read-only로 마운트할 수 있습니다.

```jsonc
"mounts": [
  "source=${localEnv:HOME}/.codex,target=/home/vscode/.codex,type=bind,readonly"
]
```

읽기 전용이면 갱신은 못 하지만 충돌도 없습니다. access token이 만료되면 호스트에서 다시 로그인하면 됩니다.

## 한 줄 정리

Dev Container에서는 Autopilot을 과감하게 쓸 수 있습니다. 다만 Codex 로그인에 쓰는 `localhost:1455`는 단일 자원처럼 다루거나, 아예 호스트의 `~/.codex`를 마운트해 OAuth를 건너뛰는 편이 안전합니다.
