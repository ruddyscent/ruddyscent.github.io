---
layout: post
title: Remote-SSH 환경에서 Codex 로그인이 멈출 때 해결하기
subtitle: Mac 브라우저와 원격 Jetson의 localhost가 다를 때 생기는 OAuth 콜백 문제
tags: [vscode, remote-ssh, codex, openai, oauth, ssh, jetson, troubleshooting]
cover-img: /assets/img/develop.jpeg
thumbnail-img: /assets/img/vscode.webp
share-img: /assets/img/develop.jpeg
author: 전경원
---

VSCode Remote-SSH로 Jetson Xavier NX에 접속해서 Codex를 실행했다. 로그인 과정에서 브라우저는 Mac에서 정상적으로 열렸고, 계정 로그인도 문제없이 진행됐다. 그런데 마지막 단계에서 아무 반응이 없었다. 에러가 크게 보이는 것도 아니고, 그냥 멈춘 것처럼 보였다.

처음에는 Codex나 브라우저 쪽 문제처럼 보였지만, 실제 원인은 더 단순했다. **브라우저가 돌아오는 `localhost`와 Codex가 기다리는 `localhost`가 서로 다른 머신을 가리키고 있었다.**

## 문제 상황

작업 흐름은 이렇다.

1. Mac의 VSCode에서 Remote-SSH로 Jetson에 접속한다.
2. Jetson의 원격 터미널에서 Codex 로그인을 시작한다.
3. 인증 URL이 열리고, Mac 브라우저에서 로그인을 완료한다.
4. 로그인 후 브라우저가 `http://localhost:1455/...` 같은 콜백 주소로 돌아온다.
5. Codex는 여전히 인증 완료를 기다린다.

여기서 중요한 점은 Codex 프로세스가 Jetson에서 실행 중이라는 것이다. 반면 브라우저는 Mac에서 실행 중이다.

```text
Codex   → Jetson에서 실행
Browser → Mac에서 실행
```

두 환경 모두 `localhost`라는 이름을 쓰지만, 실제로는 전혀 다른 네트워크 공간이다.

## 원인: OAuth 콜백이 원격 머신까지 가지 않는다

Codex 로그인은 OAuth 흐름을 사용한다. 터미널에서 로그인을 시작하면 Codex가 잠시 로컬 포트를 열어두고, 브라우저 로그인이 끝난 뒤 콜백 요청이 그 포트로 돌아오기를 기다린다.

문제는 Remote-SSH 환경에서 이 구조가 이렇게 갈라진다는 점이다.

```text
[Mac 브라우저] → 로그인 성공 → http://localhost:1455/auth/callback
                                      ↓
                              Mac의 localhost

[Jetson]      → Codex가 localhost:1455에서 콜백 대기
```

브라우저 입장에서 `localhost:1455`는 Mac 자신이다. 하지만 실제로 콜백을 받아야 하는 서버는 Jetson의 `localhost:1455`에 떠 있다.

결국 로그인은 성공했지만, 성공했다는 신호가 Jetson의 Codex 프로세스까지 전달되지 않는다. 그래서 터미널에서는 로그인이 끝나지 않은 것처럼 보인다.

## 해결: SSH 포트 포워딩으로 콜백 경로를 이어준다

해결 방법은 Mac의 `1455` 포트를 Jetson의 `1455` 포트로 연결하는 것이다. Mac 터미널에서 다음처럼 SSH 터널을 열어둔다.

```bash
ssh -L 1455:localhost:1455 <user>@<jetson-ip>
```

이 명령의 의미는 단순하다.

```text
Mac localhost:1455 → Jetson localhost:1455
```

이제 Mac 브라우저가 `http://localhost:1455/auth/callback`으로 돌아오면, 그 요청이 SSH 터널을 타고 Jetson의 `localhost:1455`로 전달된다.

전체 흐름은 이렇게 바뀐다.

```text
[Mac 브라우저]
    ↓
Mac localhost:1455
    ↓
SSH 터널
    ↓
Jetson localhost:1455
    ↓
Codex 인증 완료
```

터널을 열어둔 상태에서 다시 Codex 로그인을 시도하면 콜백이 정상적으로 도착하고, 터미널 쪽 로그인도 완료된다.

## 포트 번호는 실제 콜백 주소에 맞춘다

위 예시는 `1455` 포트를 기준으로 설명했지만, 핵심은 포트 번호 자체가 아니다. 브라우저에 표시되는 콜백 주소의 포트와 Codex가 기다리는 포트가 같아야 한다.

예를 들어 콜백 주소가 다음처럼 보이면:

```text
http://localhost:1455/auth/callback?code=...
```

터널도 같은 포트로 잡는다.

```bash
ssh -L 1455:localhost:1455 <user>@<jetson-ip>
```

다른 포트가 나온다면 앞뒤 숫자를 그 포트에 맞춘다.

```bash
ssh -L <callback-port>:localhost:<callback-port> <user>@<jetson-ip>
```

## 같은 패턴이 다른 도구에서도 나온다

이 문제는 Codex에만 특수한 현상이 아니다. 구조는 항상 비슷하다.

- 브라우저는 로컬 머신에서 열린다.
- 인증을 기다리는 CLI나 서버는 원격 머신에서 실행 중이다.
- 로그인 완료 후 브라우저가 `localhost`로 돌아온다.
- 그 `localhost`가 원격 머신이 아니라 로컬 머신을 가리킨다.

그래서 비슷한 문제가 다음 도구에서도 생길 수 있다.

- Jupyter Notebook
- VSCode Server
- GitHub CLI `gh`
- Hugging Face CLI
- 로컬 콜백을 사용하는 다른 OAuth 기반 CLI

원격 서버에서 CLI를 실행했는데 브라우저 로그인 이후 아무 일도 일어나지 않는다면, 먼저 콜백 포트가 어디에서 열려 있고 브라우저의 `localhost`가 어디를 가리키는지 확인하는 편이 좋다.

## 한 줄 정리

Remote-SSH에서 Codex 로그인이 멈춘 것처럼 보인다면, 대개 로그인 자체가 아니라 **OAuth 콜백 경로**가 끊긴 것이다. Mac 브라우저의 `localhost`를 원격 머신의 `localhost`로 이어주도록 SSH 터널을 열면 해결된다.
