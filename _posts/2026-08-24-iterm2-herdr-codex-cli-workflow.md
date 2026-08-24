---
layout: post
title: iTerm2와 Herdr로 Codex CLI 작업 환경 나누기
subtitle: 터미널 창이 아니라, 코드가 있는 머신에서 AI 에이전트 세션을 유지한다
tags: [codex, herdr, iterm2, remote-development, workflow]
cover-img: /assets/img/develop.jpeg
thumbnail-img: /assets/img/chatgpt.webp
share-img: /assets/img/develop.jpeg
author: 전경원
description: iTerm2, Herdr, Codex CLI의 역할을 나누고 macOS·Ubuntu·Jetson에서 프로젝트별 AI 개발 세션을 유지하는 작업 환경 구성.
---

MacBook에서 개발하면서 Ubuntu 서버와 Jetson을 오가다 보면 터미널 창부터 늘어난다. 로컬 프로젝트용 탭, 서버에 접속한 SSH 탭, 빌드 로그를 보는 창, 그 안에서 실행한 Codex CLI 세션까지 생긴다. 잠깐 자리를 비우려고 창을 닫으면 어디까지 작업했는지 다시 찾아야 한다.

처음에는 tmux를 더 적극적으로 쓰면 해결될 문제라고 생각했다. 하지만 로컬과 원격에서 multiplexer를 겹쳐 쓰자 어느 계층의 세션을 보고 있는지부터 헷갈렸다. 지금은 **iTerm2는 터미널 UI, Herdr는 지속되는 작업 공간, Codex CLI는 실제 개발 작업**을 맡도록 역할을 나눴다.

```text
MacBook
└── iTerm2
    ├── Local  ── Herdr ── Codex CLI
    ├── Ubuntu ── remote Herdr ── Codex CLI
    └── Jetson ── remote Herdr ── Codex CLI
```

겉으로는 도구 하나를 더 넣은 구성이지만 핵심은 계층을 늘리는 데 있지 않다. 코드와 빌드 환경이 있는 머신마다 Herdr가 세션을 붙들고 있고 iTerm2는 그 세션으로 들어가는 입구만 제공한다.

## 도구별 역할

각 도구의 경계를 먼저 정하면 구성이 단순해진다.

| 도구 | 맡는 일 |
| --- | --- |
| <i class="fas fa-terminal fa-fw" aria-hidden="true"></i>&nbsp; iTerm2 | macOS의 창, 탭, 프로필 관리 |
| <i class="fas fa-key fa-fw" aria-hidden="true"></i>&nbsp; SSH | 원격 머신까지 연결 |
| <i class="fas fa-layer-group fa-fw" aria-hidden="true"></i>&nbsp; Herdr | 프로젝트 작업 공간과 터미널·에이전트 세션 유지 |
| <i class="fas fa-robot fa-fw" aria-hidden="true"></i>&nbsp; Codex CLI | 코드 분석, 수정, 명령 실행과 테스트 |
| <i class="fab fa-git-alt fa-fw" aria-hidden="true"></i>&nbsp; Git | 사람이 검토할 수 있는 변경 이력 관리 |

iTerm2에는 이미 탭과 split, profile이 있다. 따라서 macOS 전체의 창 배치까지 Herdr로 다시 감쌀 필요는 없다. Herdr는 각 머신에서 오래 살아 있어야 하는 개발 프로세스와 AI 에이전트 세션에 집중한다.

[Herdr](https://herdr.dev/docs/quick-start/)는 프로젝트별 workspace 안에 pane과 agent를 두고 백그라운드 서버가 pane을 계속 실행하는 터미널 기반 세션 관리자다. 클라이언트에서 빠져나가도 작업은 멈추지 않으며 나중에 다시 붙을 수 있다. Codex integration을 설치하면 Herdr가 Codex의 세션 식별자를 기록해 서버 재시작 뒤에도 네이티브 세션 복원을 시도한다.

## 로컬 환경부터 시작한다

이 글의 macOS 명령은 모두 iTerm2에서 실행한다. 먼저 Homebrew로 Herdr를 설치한다.

```bash
brew install herdr
herdr --version
```

[Codex CLI 공식 문서](https://learn.chatgpt.com/docs/codex/cli)의 Homebrew 방식을 쓰면 Herdr와 함께 패키지 관리자를 하나로 맞출 수 있다. iTerm2에서 이어서 설치한다.

```bash
brew install --cask codex
codex --version
```

Codex를 한 번 실행해 설정 디렉터리가 만들어진 뒤 Herdr integration을 설치한다.

```bash
herdr integration install codex
herdr integration status
```

이 integration은 Codex의 세션 정보를 Herdr에 전달한다. Herdr 공식 문서에 따르면 Codex의 동작 상태는 화면을 바탕으로 감지하고 integration은 세션 복원에 필요한 식별자를 보고한다. 단순히 Codex 프로세스를 pane에 띄우는 것보다 한 단계 더 이어지는 셈이다.

프로젝트 디렉터리에서 `herdr`를 실행하면 로컬 세션을 시작하거나 기존 세션에 다시 붙는다.

```bash
cd ~/Workspace/my-project
herdr
```

내가 기본으로 쓰는 구성은 pane 두 개뿐이다.

```text
┌──────────────────────────┬──────────────────────┐
│                          │                      │
│          Codex           │        shell         │
│                          │                      │
│                          │ $ git status         │
│                          │ $ git diff           │
│                          │ $ pytest             │
│                          │                      │
└──────────────────────────┴──────────────────────┘
```

한쪽에서는 Codex와 대화하고 다른 쪽에서는 `git diff`와 테스트 결과를 직접 확인한다. pane을 많이 만들어 모든 일을 동시에 벌이기보다, 에이전트의 작업과 사람의 검증을 분리하는 정도가 시작점으로 적당했다.

## 원격 머신에도 같은 구조를 둔다

Ubuntu와 Jetson에서도 코드가 있는 머신에 Herdr와 Codex CLI를 설치한다. Herdr는 Linux와 macOS에서 공식 설치 스크립트를 제공한다.

```bash
curl -fsSL https://herdr.dev/install.sh | sh
herdr --version
```

이어서 OpenAI가 macOS와 Linux에 제공하는 standalone installer로 Codex CLI를 설치한다. Ubuntu 서버와 Jetson의 SSH 셸에서 같은 명령을 쓰면 된다.

```bash
curl -fsSL https://chatgpt.com/codex/install.sh | sh
codex --version
```

프로젝트 디렉터리에서 `codex`를 처음 실행하면 로그인 방식을 선택할 수 있다.

```bash
cd ~/Workspace/my-project
codex
```

Codex CLI 설치와 인증을 마친 다음에는 원격 머신에서도 integration을 설치한다.

```bash
herdr integration install codex
```

가장 단순한 접속 방법은 SSH로 서버에 들어가 Herdr를 실행하는 것이다.

```bash
ssh xavier
herdr
```

이 경로에서는 셸과 Herdr 클라이언트, 서버가 모두 원격 머신에서 실행된다. 평소 SSH 셸 안에서 작업하거나 휴대전화의 SSH 클라이언트로 접속할 때 이해하기 쉽다.

iTerm2에서 원격 세션을 로컬처럼 열고 싶다면 [Herdr의 remote attach](https://herdr.dev/docs/persistence-remote/)를 쓸 수 있다.

```bash
herdr --remote xavier
```

이때 로컬 Herdr는 얇은 클라이언트가 된다. SSH를 통해 원격 Herdr 서버를 시작하거나 기존 서버에 붙고 화면만 현재 터미널로 가져온다. 이미지 클립보드처럼 로컬 데스크톱에 의존하는 기능도 원격 세션으로 이어줄 수 있다는 점이 일반 SSH 접속과 다르다.

반복해서 접속할 머신은 `~/.ssh/config`에 별칭을 둔다.

```sshconfig
Host xavier
    HostName 192.168.50.10
    User myuser

Host orin
    HostName 192.168.50.21
    User myuser
```

그러면 로컬과 두 원격 머신을 같은 형태의 명령으로 오갈 수 있다.

```bash
# MacBook의 프로젝트
herdr

# Ubuntu의 프로젝트
herdr --remote xavier

# Jetson의 프로젝트
herdr --remote orin
```

`herdr --remote`에서 인증 문제가 나면 Herdr부터 의심하기보다 `ssh xavier`가 정상적으로 연결되는지 먼저 확인한다. 암호가 걸린 키를 비대화형 환경에서 사용할 때는 `ssh-agent`에 키가 올라가 있어야 한다.

## Codex는 코드가 있는 곳에서 실행한다

이 구성에서 가장 중요한 원칙은 Codex를 어느 머신에서 실행하느냐다.

> 코드와 빌드·테스트 환경이 실제로 존재하는 머신에서 Codex를 실행한다.

Ubuntu에 있는 프로젝트를 Mac의 Codex가 SSH 명령으로 조작하게 만들 수도 있다. 그러나 그렇게 하면 파일 접근과 명령 실행, 권한, 환경 변수의 경계가 두 머신에 걸친다. 반대로 Ubuntu에서 Codex를 실행하면 그 머신의 컴파일러와 Python 환경, Podman 컨테이너, 테스트 도구를 직접 사용한다.

Jetson에서는 이 차이가 더 분명하다. CUDA와 JetPack, 장치 접근이 필요한 테스트는 Mac에서 재현할 수 없다. Mac의 iTerm2는 화면을 보여주고 키 입력을 전달할 뿐이며 Codex와 소스 코드, 빌드와 테스트는 Jetson 안에 함께 둔다.

```text
iTerm2
  ↓
herdr --remote orin
  ↓
Jetson의 Herdr
  ↓
Jetson의 Codex
  ↓
source · CUDA · Podman · test
```

이렇게 경계를 잡으면 Codex에 별도의 원격 조작 규칙을 가르칠 필요가 줄어든다. 에이전트가 보는 파일 시스템과 사람이 기대하는 실행 환경이 처음부터 같다.

## Herdr를 두 번 겹치지 않는다

로컬 Herdr 안의 pane에서 SSH를 실행하고 다시 원격 Herdr를 여는 구조도 가능은 하다.

```text
iTerm2 → Mac Herdr → SSH → Ubuntu Herdr → Codex
```

하지만 이 구성에서는 detach와 키 바인딩, workspace의 의미가 두 단계로 겹친다. Herdr도 기본 설정에서는 중첩 실행을 허용하지 않는다. 나는 iTerm2 탭을 최상위 경계로 두고 각 탭에서 필요한 Herdr에 바로 붙는다.

```text
iTerm2
├── Local 탭  ── herdr
├── Ubuntu 탭 ── herdr --remote xavier
└── Jetson 탭 ── herdr --remote orin
```

tmux도 없애지 않았다. Herdr가 설치되지 않은 서버나 범용 시스템 관리에는 여전히 tmux가 편하다. 다만 평상시 AI 개발 경로에서 iTerm2, Herdr, tmux를 모두 중첩하지 않는다. 같은 역할을 맡은 도구가 한 경로에 둘 이상 나타나면 어느 세션을 유지해야 하는지 다시 살펴본다.

## 작업마다 Git worktree를 만든다

여러 작업을 동시에 진행할 때는 같은 디렉터리에서 브랜치를 계속 바꾸지 않는다. Git worktree로 작업 디렉터리를 따로 만들고, 그 안에서 Herdr와 Codex를 실행한다. 브랜치와 파일, 에이전트 세션의 경계가 같은 단위로 맞춰진다.

먼저 기본 저장소에서 새 브랜치와 worktree를 함께 만든다.

```bash
cd ~/Workspace/my-project

BASE_BRANCH=main # 저장소에 따라 master 등으로 변경
TASK_BRANCH=feat/login
WORKTREE_DIR=../my-project-login

git fetch origin
git worktree add -b "$TASK_BRANCH" "$WORKTREE_DIR" "$BASE_BRANCH"
```

이제 iTerm2에서 worktree 디렉터리로 이동해 Herdr를 연다. Codex도 이 디렉터리를 기준으로 코드를 읽고 명령을 실행한다.

```bash
cd ~/Workspace/my-project-login
herdr
```

작업 흐름은 단순하다.

```text
Git branch: feat/login
└── worktree: ../my-project-login
    └── Herdr workspace
        ├── Codex
        └── shell · test · git diff
```

Codex가 작업을 마치면 옆 pane에서 변경 내용과 테스트 결과를 직접 확인한 뒤 커밋한다.

```bash
git status
git diff
git add -p
git commit
```

브랜치를 병합한 뒤에는 기본 저장소로 돌아가 worktree를 정리한다. `git status`에 남은 변경이 없는지 먼저 확인하고, 아직 병합하지 않은 브랜치는 삭제하지 않는다.

```bash
cd ~/Workspace/my-project
git worktree remove ../my-project-login
git branch -d feat/login
git worktree prune
```

이 절차를 따르면 작업마다 별도의 저장소를 복제할 필요가 없다. Git 객체는 공유하면서 작업 디렉터리와 브랜치, Herdr 세션은 서로 섞이지 않는다.

## 창을 닫아도 작업은 남는다

Herdr 클라이언트는 `Ctrl-b q`로 detach할 수 있다. pane과 그 안의 에이전트는 백그라운드에서 계속 실행된다. 로컬에서는 다시 `herdr`, 원격에서는 다시 `herdr --remote <host>`를 실행하면 기존 작업 공간으로 돌아간다.

```bash
# 원격 세션에서 빠져나온 뒤 다시 접속
herdr --remote xavier
```

detach는 세션을 끝내는 명령이 아니다. pane까지 종료하려면 `herdr server stop`을 사용해야 한다. 이 차이를 알고 있으면 터미널 창의 수명과 작업의 수명을 분리할 수 있다.

대규모 코드 분석이나 리팩터링처럼 시간이 오래 걸리는 작업에서 특히 유용하다. MacBook의 덮개를 닫거나 네트워크가 끊기더라도 원격 머신의 Herdr 서버와 pane은 그곳에 남는다. 다시 연결했을 때 새 Codex 대화를 열고 맥락부터 복원하는 대신 기존 세션을 이어간다.

## 머신이 늘어나도 입구만 하나 추가한다

최종 구조에서 기억할 것은 세 문장이다.

- iTerm2는 로컬의 창과 탭을 관리한다.
- Herdr는 각 머신에서 프로젝트와 에이전트 세션을 유지한다.
- Codex는 코드와 빌드 환경이 있는 머신에서 실행한다.

새 Linux 머신을 추가할 때도 SSH와 Herdr, Codex를 준비하고 SSH 별칭 하나를 더하면 된다.

```bash
herdr --remote new-server
```

그 뒤의 사용법은 Ubuntu나 Jetson과 다르지 않다. 개발 머신은 늘어나지만 사람이 기억해야 할 작업 흐름은 늘어나지 않는다.

AI 코딩 에이전트를 오래 사용할수록 터미널 창을 몇 개 열었는지보다 **어느 프로젝트의 에이전트가 어떤 머신에서 작업 중인지**가 중요해진다. iTerm2와 Herdr, Codex CLI를 함께 쓰는 이유도 여기에 있다. 도구를 더 많이 쌓기 위해서가 아니라, 화면과 세션과 실행 환경의 경계를 분명하게 나누기 위해서다.
