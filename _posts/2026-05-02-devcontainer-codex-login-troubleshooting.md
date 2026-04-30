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

VSCode Dev Container 안에서 Codex를 쓰기 시작했다. 그런데 여러 컨테이너에서 Codex를 쓰기 시작하면서 로그인 문제가 발생했다. 처음에는 단순히 `localhost`가 호스트와 컨테이너에서 다르게 해석되는 문제처럼 보였다. 하지만 실제로는 여러 Dev Container가 같은 OAuth 콜백 포트를 포워딩하려다 충돌한 쪽에 가까웠다.

## 에러 재현

여러 컨테이너가 돌아가는 상황에서도 로그인 과정은 똑같다.

- ID 입력
- 비밀번호 입력
- OTP 입력

여기까지는 잘 진행됐다. 마지막에 브라우저가 열리고 로그인 완료 화면이 나와야 했지만, 화면에는 에러가 떴다.

```text
ERR_CONNECTION_REFUSED
```

주소는 다음과 같았다.

```text
http://localhost:1455/auth/callback?code=...
```

![브라우저에서 Codex OAuth 콜백 주소인 localhost:1455에 접속하지 못해 ERR_CONNECTION_REFUSED가 표시된 화면](/assets/img/posts/2026-05-02-devcontainer-codex-login-troubleshooting/login-error-connection-refused.png)

이 에러를 파고들면서 알게 된 건, 핵심이 단순한 로그인 실패가 아니라 **브라우저가 돌아올 호스트 포트와 컨테이너 안의 콜백 포트가 어떻게 연결되느냐**에 있다는 점이었다.

## 여러 컨테이너에서 1455 포트가 충돌한다

먼저 짚고 갈 점이 있다. 호스트의 `1455`와 컨테이너 내부의 `1455`는 같은 숫자지만 같은 포트는 아니다.

- 브라우저는 호스트 OS에서 돈다
- Codex는 Dev Container 안에서 돈다
- 컨테이너마다 내부 네트워크 공간은 따로 있다

같은 `localhost`처럼 보여도 가리키는 곳이 다르다.

```text
브라우저 → localhost:1455 (호스트)
Codex   → localhost:1455 (컨테이너)
```

그래서 Dev Container에서 Codex 로그인을 하려면 보통 호스트의 `localhost:1455`를 컨테이너 내부의 `1455`로 이어줘야 한다.

`.devcontainer/devcontainer.json`에는 이렇게 적을 수 있다.

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

이 자체는 문제가 아니다. 호스트 `1455`와 컨테이너 `1455`를 연결하는 것은 정상적인 포트 포워딩이다.

문제는 Dev Container를 여러 개 동시에 띄울 때 생긴다.

상황은 이렇다.

- 첫 번째 Dev Container의 포워딩이 이미 호스트 1455를 잡고 있다
- 두 번째 Dev Container에서 또 로그인을 시도한다
- 브라우저는 여전히 `localhost:1455`로 돌아오려고 한다

컨테이너마다 내부 `1455`는 따로 존재한다. 하지만 호스트의 `localhost:1455`는 하나뿐이다.

```text
호스트 localhost:1455 → 컨테이너 A:1455
호스트 localhost:1455 → 컨테이너 B:1455
```

먼저 띄운 컨테이너가 포트를 잡고 있으면 두 번째 컨테이너의 콜백은 엉뚱한 곳으로 간다. 이 경우는 `forwardPorts` 추가만으로 풀리지 않는다.

### 점유 확인과 세션 정리

**1. 누가 1455를 잡고 있는지 확인**

호스트에서 다음 명령으로 확인한다.

```bash
lsof -i :1455
```

다른 프로세스나 세션이 이미 잡고 있다면 두 번째 콜백은 정상적으로 붙기 어렵다.

**2. 이전 세션 정리**

- 먼저 띄운 Dev Container의 Codex 세션 종료
- 필요하면 그 VSCode 창도 닫기
- 포워딩 세션 정리
- 다시 로그인 시도

상황에 따라서는 점유 중인 프로세스를 직접 종료해야 할 수도 있다.

```bash
kill -9 <PID>
```

**3. Ports 뷰에서 실제 매핑 확인**

핵심은 단순히 포트가 열려 있느냐가 아니다. 브라우저는 `localhost:1455`로 돌아오기 때문에 호스트 쪽도 정확히 1455여야 한다.

```text
1455 → 1455   정상 가능성 높음
1455 → 1456   콜백 실패 가능
```

포워딩이 된 것처럼 보여도 호스트 쪽 포트 번호가 달라지면 인증 흐름이 깨진다.

**4. 로그인은 한 컨테이너씩**

여러 Dev Container를 동시에 띄워야 한다면, 적어도 Codex 로그인만큼은 한 번에 하나씩 처리하는 편이 안전하다.

1. 첫 번째 컨테이너에서 로그인 완료
2. 필요 없는 세션 정리
3. 두 번째 컨테이너에서 로그인 진행

이 순서가 가장 덜 꼬인다.

![여러 Dev Container가 호스트의 localhost 1455 포트를 두고 충돌하는 구조](/assets/img/posts/2026-05-02-devcontainer-codex-login-troubleshooting/codex-devcontainer-port-conflict-handdrawn.svg)

## 더 나은 해결책: 자격증명을 마운트해서 OAuth 자체를 건너뛴다

케이스 1의 해결책은 기본적으로 같은 방향이다.

> OAuth 콜백을 어떻게든 컨테이너 안까지 잘 도달하게 만들자.

그런데 시야를 한 번 바꿔볼 수 있다.

> 애초에 컨테이너 안에서 OAuth를 다시 할 필요가 있을까?

호스트에서 이미 한 번 로그인했다면, 그 자격증명을 컨테이너에 그대로 가져다 쓰면 된다.

### Codex의 자격증명 위치

Codex CLI는 홈 디렉터리 아래에 인증 정보를 모아둔다.

```text
~/.codex/
├── auth.json        # OAuth 토큰
├── config.toml      # 설정
└── sessions/        # 대화 히스토리
```

이 중 `auth.json` 하나가 로그인 상태를 결정한다.

### devcontainer.json에 마운트 추가

호스트의 `~/.codex` 디렉터리를 컨테이너 홈 안으로 바인드 마운트한다.

```jsonc
{
  "mounts": [
    "source=${localEnv:HOME}/.codex,target=/home/vscode/.codex,type=bind,consistency=cached"
  ]
}
```

여기서 `target`의 `/home/vscode`는 일부러 명시했다. `mounts`는 컨테이너가 만들어질 때 적용되기 때문에, 컨테이너 안에서 나중에 결정되는 `HOME` 값에 기대기보다 실제 홈 경로를 적는 편이 안전하다.

`remoteUser`가 `vscode`가 아니라면 `target` 경로의 홈 부분만 바꿔주면 된다.

![호스트의 ~/.codex 디렉터리를 여러 Dev Container의 /home/vscode/.codex로 바인드 마운트해 OAuth 콜백 없이 같은 Codex 자격증명을 공유하는 구조](/assets/img/posts/2026-05-02-devcontainer-codex-login-troubleshooting/codex-devcontainer-auth-mount-handdrawn.svg)

이렇게 하면 컨테이너에서 `codex`를 실행해도 호스트의 토큰을 그대로 쓴다.

- OAuth 콜백 자체가 일어나지 않는다
- 1455 포트와 무관하게 동작한다
- Rebuild 후에도 다시 로그인할 필요가 없다

### 알아둘 것: 권한과 토큰 갱신

**1. UID 차이로 인한 권한 문제**

호스트(macOS)와 컨테이너(Linux)의 UID가 다르면 컨테이너에서 `auth.json`을 쓰지 못할 수 있다. 이 경우 컨테이너 안에서 한 번 소유자를 바꿔주면 된다.

```bash
sudo chown -R $(id -u):$(id -g) ~/.codex
```

**2. 토큰 갱신 충돌**

호스트와 컨테이너에서 동시에 Codex를 활발히 쓰면, 양쪽이 각자 토큰을 갱신하다가 서로 덮어쓸 수 있다. 이게 신경 쓰이면 read-only로 마운트하는 방법이 있다.

```jsonc
"mounts": [
  "source=${localEnv:HOME}/.codex,target=/home/vscode/.codex,type=bind,readonly"
]
```

읽기 전용이면 갱신은 못 하지만 충돌도 없다. access token이 만료되면 호스트에서 다시 로그인해주면 된다.

### 다중 컨테이너 환경에서는 이 방식이 가장 강력하다

이 접근의 진짜 장점은 케이스 1 같은 상황에서 드러난다.

여러 Dev Container를 동시에 띄울 때:

- 케이스 1 방식 → 컨테이너마다 1455 포트를 두고 경쟁한다
- 케이스 2 방식 → OAuth를 안 하므로 충돌할 자원 자체가 없다

컨테이너를 다섯 개 띄워도 모두 호스트 자격증명을 공유하면서 함께 동작한다.

## 두 가지 케이스 한눈에 보기

같은 에러처럼 보여도 접근 방법은 한 가지가 아니었다.

### 케이스 1: 여러 Dev Container에서 포트 충돌

- 호스트 포트와 컨테이너 포트는 서로 다르다
- 컨테이너마다 내부 `1455`는 따로 있다
- 하지만 호스트의 `localhost:1455`는 한 번에 하나만 제대로 연결된다

해결:
- `forwardPorts`에 `1455` 추가
- 1455 점유 확인
- 기존 세션 정리
- 실제 포트 매핑 확인
- 로그인은 한 컨테이너씩

### 케이스 2: OAuth 자체를 건너뛰기

- 호스트에는 이미 로그인되어 있다
- 컨테이너에서 또 OAuth를 할 이유가 없다

해결:
- 호스트의 `~/.codex`를 컨테이너에 바인드 마운트
- 권한과 토큰 갱신 충돌만 주의
- 다중 컨테이너 환경에서는 가장 안정적인 선택

## 결론

Dev Container + Codex 조합의 가치는 분명하다.

> Autopilot이나 Bypass Approvals 같은 높은 권한도 비교적 안심하고 쓸 수 있다.

다만 이 환경일수록 **인증 콜백 포트가 어떻게 연결되는지**를 알아둘 필요가 있다. 특히 Dev Container를 여러 개 동시에 띄우는 순간부터는, 단순한 포워딩을 넘어 **호스트 포트 충돌**까지 봐야 한다.

그리고 그 모든 포트 이슈가 부담스러워진다면, 한 발짝 물러나서 **OAuth 자체를 건너뛰는 케이스 2**가 가장 깔끔하다.

## 한 줄 정리

> Dev Container에서는 Autopilot을 과감하게 쓸 수 있다. 다만 Codex 로그인에 쓰는 `localhost:1455`는 단일 자원처럼 다루거나, 아예 호스트의 `~/.codex`를 마운트해서 OAuth를 건너뛰는 편이 안전하다.
