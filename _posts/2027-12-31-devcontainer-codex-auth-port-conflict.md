---
layout: post
title: Dev Container에서 Codex를 Autopilot으로 써도 안전한 이유 + 로그인 에러 해결
subtitle: Permission Level과 localhost 콜백 문제, 다중 Dev Container 포트 충돌까지 정리
tags: [vscode, devcontainer, codex, openai, docker, troubleshooting]
cover-img: /assets/img/develop.jpeg
thumbnail-img: /assets/img/vscode.png
share-img: /assets/img/develop.jpeg
author: 전경원
---

VSCode Dev Container 안에서 Codex를 사용하면서 두 가지를 동시에 경험했다.

1. Autopilot을 마음 놓고 써도 되는 환경  
2. 로그인에서 막히는 예상 밖의 문제  

그런데 나중에 보니 로그인 문제도 한 종류가 아니었다.  
처음에는 단순한 포트 포워딩 문제라고 생각했지만, Dev Container를 여러 개 띄운 상태에서는 **포트 충돌** 때문에 같은 증상이 다시 발생할 수 있었다.

## Dev Container에서 Codex를 쓰는 진짜 장점

Codex를 제대로 쓰려면 결국 이 선택을 하게 된다.

- Ask for Approval
- Bypass Approvals
- Autopilot

처음에는 승인 기반으로 시작하는 것이 안전해 보인다.  
하지만 작업이 조금만 길어져도 흐름이 계속 끊긴다.

- 파일 수정할 때 승인
- 명령 실행할 때 승인
- 다시 수정할 때 또 승인

이런 식이면 AI를 붙여도 속도가 잘 안 나온다.

그래서 결국 이런 고민을 하게 된다.

> Autopilot으로 올리고 싶은데, 괜찮을까?

## 로컬 환경에서는 Autopilot이 부담스럽다

Codex는 생각보다 강한 권한을 가진다.

- 파일 생성 / 수정 / 삭제
- 패키지 설치
- 쉘 명령 실행
- 프로젝트 구조 변경

즉, 로컬 환경에서 권한을 크게 열어두면 심리적으로 불안하다.

- 환경이 오염될 수 있고
- 설정이 꼬일 수 있고
- 원하지 않는 변경이 누적될 수 있다

그래서 기능은 강력하지만, 실제로는 조심스럽게 쓰게 된다.

## Dev Container에서는 상황이 달라진다

Dev Container 안에서는 전제가 바뀐다.

> Codex가 건드리는 대상이 내 컴퓨터 전체가 아니라, 컨테이너 내부다.

이 차이가 결정적이다.

### 격리된 환경

Codex가 어떤 작업을 하더라도 영향 범위는 컨테이너 안으로 제한된다.

- 호스트 운영체제는 그대로 유지되고
- 다른 프로젝트 환경과도 분리되며
- 문제가 생겨도 범위가 제한적이다

### 언제든지 리셋 가능

이 부분이 가장 크다.

문제가 생기면:

```bash
Rebuild Container
```

사실상 여기서 정리가 된다.

- 잘못 설치한 패키지
- 꼬여버린 설정
- 실험하다 망가진 환경

이런 것들을 로컬에서는 하나씩 복구해야 하지만, 컨테이너에서는 다시 만드는 쪽이 더 쉽다.

### 그래서 Autopilot을 과감하게 쓸 수 있다

로컬에서는:

```text
Ask for Approval = 안전
Autopilot = 부담
```

Dev Container에서는:

```text
Autopilot = 빠르고, 비교적 안전
```

즉, Dev Container + Codex 조합의 핵심 장점은  
**권한을 높여도 감당 가능한 범위 안에서 움직인다는 점**이다.

## 로그인 문제 발생

환경을 다 구성하고 Codex 로그인을 진행했다.

- ID 입력
- 비밀번호 입력
- OTP 입력

여기까지는 정상적으로 진행됐다.  
그런데 마지막 단계에서 브라우저가 열리더니 로그인 완료 대신 에러가 떴다.

```text
ERR_CONNECTION_REFUSED
```

주소는 이런 형태였다.

```text
http://localhost:1455/auth/callback?code=...
```

![](/assets/img/login-error-connection-refused.png)

## 첫 번째 원인: Dev Container의 localhost는 호스트와 다르다

처음 문제는 구조적인 것이었다.

- 브라우저는 호스트 OS에서 실행되고
- Codex 확장은 Dev Container 안에서 실행된다

즉, 같은 `localhost`처럼 보여도 실제로는 다르다.

```text
브라우저 → localhost:1455 (호스트)
Codex   → localhost:1455 (컨테이너)
```

그래서 브라우저 입장에서는:

> localhost:1455에 아무것도 없는데?

이 상태가 되어 `ERR_CONNECTION_REFUSED`가 발생한다.

## 첫 번째 해결: 1455 포트를 그대로 열어준다

이 경우 해결 방법은 명확하다.

`.devcontainer/devcontainer.json`에 콜백 포트를 포워딩한다.

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

그 다음 순서대로 적용한다.

1. `devcontainer.json` 수정
2. **Rebuild Container**
3. VSCode 재접속
4. Codex 로그인 재시도

이렇게 하면 브라우저가 접근하는 호스트의 `localhost:1455`가  
컨테이너 내부의 1455와 연결되어 로그인이 완료된다.

## 그런데 여기서 끝이 아니었다

처음에는 이 설정만 하면 해결된다고 생각했다.

그런데 Dev Container를 하나만 쓸 때는 잘 되다가,  
여러 Dev Container를 동시에 띄운 상태에서는 같은 로그인 문제가 다시 나타났다.

겉으로 보면 똑같이 `ERR_CONNECTION_REFUSED`라서 처음에는 같은 원인처럼 보인다.  
하지만 이 경우는 **포트 포워딩이 아니라 포트 충돌**일 수 있다.

## 두 번째 원인: 여러 Dev Container + Codex 조합에서 1455 포트 충돌

문제 상황은 이렇다.

- 첫 번째 Dev Container에서 Codex가 이미 로그인용 포트를 사용 중
- 두 번째 Dev Container에서도 Codex 로그인을 시도
- 브라우저는 여전히 `localhost:1455`로 돌아오려고 함

하지만 호스트 입장에서는 `1455`는 하나다.

즉,

- 컨테이너마다 내부 1455는 따로 존재할 수 있어도
- 호스트의 `localhost:1455`는 동시에 하나만 제대로 연결될 수 있다

그래서 먼저 실행한 Dev Container + Codex 조합이 포트를 잡고 있으면  
두 번째 Dev Container 쪽 로그인 콜백이 꼬일 수 있다.

이 경우는 단순히 `forwardPorts`를 추가했다고 끝나지 않는다.

## 이런 경우 어떻게 해결할까

### 1. 누가 1455를 잡고 있는지 먼저 확인

호스트에서 다음처럼 확인할 수 있다.

```bash
lsof -i :1455
```

이미 다른 프로세스나 세션이 1455를 잡고 있다면,  
두 번째 Dev Container의 로그인 콜백이 정상적으로 붙지 않을 수 있다.

### 2. 이전 세션을 정리

- 먼저 띄운 Dev Container의 Codex 세션 종료
- 필요하면 해당 VSCode 창 종료
- 포워딩 세션 정리
- 다시 로그인 시도

상황에 따라서는 점유 중인 프로세스를 종료해야 할 수도 있다.

```bash
kill -9 <PID>
```

### 3. Ports 뷰에서 실제 매핑 상태 확인

중요한 건 단순히 포트가 열려 있는지가 아니다.

브라우저는 `localhost:1455`로 돌아오기 때문에  
실제로도 **1455 → 1455**로 잡혀 있어야 한다.

예를 들어:

```text
1455 → 1455  정상 가능성 높음
1455 → 1456  로그인 콜백 실패 가능
```

겉보기에는 포워딩이 된 것 같아도,  
호스트 쪽 포트 번호가 달라지면 인증 흐름이 깨질 수 있다.

### 4. 로그인은 한 컨테이너씩 처리

여러 Dev Container를 동시에 띄워야 한다면,  
적어도 Codex 로그인만큼은 한 번에 하나씩 처리하는 편이 안전하다.

즉:

1. 첫 번째 컨테이너에서 로그인 완료
2. 필요 없는 세션 정리
3. 두 번째 컨테이너에서 로그인 진행

이 방식이 가장 덜 꼬인다.

## 정리하면 원인은 두 가지였다

겉으로 같은 에러처럼 보여도 실제 원인은 달랐다.

### 케이스 1: 포트 포워딩 미설정

- 브라우저는 호스트
- Codex는 컨테이너
- `localhost`가 분리되어 인증 콜백 실패

해결:
- `forwardPorts`에 `1455` 추가

### 케이스 2: 여러 Dev Container에서 포트 충돌

- 이미 다른 Dev Container + Codex가 호스트의 1455 흐름을 점유
- 두 번째 컨테이너에서 로그인 콜백 실패

해결:
- 1455 점유 확인
- 기존 세션 정리
- 실제 포트 매핑 확인
- 로그인은 한 컨테이너씩 처리

## 결론

Dev Container + Codex 조합의 핵심 장점은 분명하다.

> Autopilot이나 Bypass Approvals 같은 높은 권한도 비교적 안심하고 사용할 수 있다.

하지만 이런 환경일수록  
**인증 콜백 포트가 어떻게 연결되는지**를 이해하고 있어야 한다.

특히 Dev Container를 여러 개 동시에 띄우는 순간부터는  
단순한 포워딩 문제를 넘어서 **호스트 포트 충돌**까지 고려해야 한다.

## 한 줄 정리

> Dev Container에서는 Autopilot을 과감하게 쓸 수 있다. 다만 Codex 로그인에 쓰는 `localhost:1455`는 단일 자원처럼 다루는 게 안전하다.
