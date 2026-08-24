---
layout: post
title: Ghostty와 Herdr로 Codex CLI 작업 환경 나누기
subtitle: 터미널 창이 아니라 코드가 있는 머신에서 AI 에이전트 세션을 유지한다
tags: [codex, herdr, ghostty, remote-development, workflow]
cover-img: /assets/img/develop.jpeg
thumbnail-img: /assets/img/chatgpt.webp
share-img: /assets/img/develop.jpeg
author: 전경원
description: Ghostty, Herdr, Codex CLI의 역할을 나누고 macOS·Ubuntu·Jetson에서 프로젝트별 AI 개발 세션을 유지하는 작업 환경 구성.
---

MacBook에서 개발하면서 Ubuntu 서버와 Jetson을 오가다 보면 터미널 창부터 늘어난다. 로컬 프로젝트용 탭, 서버에 접속한 SSH 탭, 빌드 로그를 보는 창, 그 안에서 실행한 Codex CLI 세션까지 생긴다. 잠깐 자리를 비우려고 창을 닫으면 어느 세션에서 무엇을 하던 중이었는지 다시 찾아야 한다.

이 글에서 `xavier`는 Ubuntu 개발 서버, `orin`은 Jetson 개발 장치다. 두 장치 모두 Codex가 코드를 수정하고 빌드와 테스트를 실행하는 실제 작업 머신이다.

이 환경에서는 **Ghostty**가 로컬 터미널의 창과 탭을 맡고 **Herdr**가 각 머신의 작업 세션을 유지한다. **Codex CLI**는 코드와 빌드 환경이 있는 머신에서 실제 개발 작업을 수행한다. 세 도구를 함께 쓰되 역할이 겹치지 않게 나누는 것이 핵심이다.

```text
MacBook
└── Ghostty
    ├── Local           ── Herdr ── Codex CLI
    ├── Ubuntu (xavier) ── remote Herdr ── Codex CLI
    └── Jetson (orin)   ── remote Herdr ── Codex CLI
```

## 시작하기 전에

이 글은 MacBook에 Homebrew가 설치되어 있고 Mac에서 두 원격 머신에 SSH로 접속할 수 있다고 가정한다. `xavier`와 `orin`은 뒤에서 설정할 SSH 별칭이다.

작업할 Git 저장소도 Mac과 원격 머신에 각각 복제되어 있어야 한다. 예제에서는 원격 저장소 이름을 `origin`, 기본 분기를 `master`로 사용한다. 기본 분기가 `main`인 저장소라면 뒤에 나오는 `master`와 `origin/master`를 각각 `main`과 `origin/main`으로 바꾼다. GitHub 이슈를 읽고 PR을 만들려면 GitHub 계정과 해당 저장소에 접근할 권한도 필요하다.

## 도구별 역할

각 도구의 경계를 먼저 정하면 구성이 단순해진다.

| 도구 | 맡는 일 |
| --- | --- |
| <i class="fas fa-terminal fa-fw" aria-hidden="true"></i>&nbsp; Ghostty | macOS의 네이티브 터미널 창, 탭, split 제공 |
| <i class="fas fa-key fa-fw" aria-hidden="true"></i>&nbsp; SSH | 원격 머신까지 연결 |
| <i class="fas fa-layer-group fa-fw" aria-hidden="true"></i>&nbsp; Herdr | 프로젝트 작업 공간과 터미널·에이전트 세션 유지 |
| <i class="fas fa-robot fa-fw" aria-hidden="true"></i>&nbsp; Codex CLI | 코드 분석, 수정, 명령 실행과 테스트 |
| <i class="fab fa-git-alt fa-fw" aria-hidden="true"></i>&nbsp; Git | 코드 변경 이력과 분기(branch) 관리 |

[Ghostty](https://ghostty.org/docs/features)는 macOS에서 네이티브 창과 탭, split을 제공하고 Metal로 터미널 화면을 그린다. Ghostty는 셸과 Herdr 클라이언트의 화면을 렌더링하고 키 입력을 전달한다. 개발 프로세스와 AI 에이전트 세션을 유지하는 일은 Herdr가 맡는다. 따라서 Ghostty 탭을 닫아 클라이언트 연결이 끊겨도 Herdr 서버의 원격 pane과 Codex 작업은 계속 실행될 수 있다.

[Herdr](https://herdr.dev/docs/quick-start/)는 프로젝트별 workspace 안에 pane과 agent를 두고 백그라운드 서버가 pane을 계속 실행하는 터미널 기반 세션 관리자다. 클라이언트에서 빠져나가도 작업은 멈추지 않으며 나중에 다시 붙을 수 있다. Codex integration을 설치하면 Herdr는 Codex의 세션 식별자를 기록하고 서버를 재시작한 뒤에도 네이티브 세션 복원을 시도한다.

## Ghostty에서 로컬 환경을 시작한다

Ghostty 프로젝트는 서명·공증된 macOS `.dmg`를 공식 배포한다. Homebrew에는 커뮤니티가 관리하는 cask도 있다. 여기서는 나머지 도구와 설치 방식을 맞추기 위해 Homebrew를 사용한다.

```bash
brew install --cask ghostty
```

Ghostty는 별도 설정 없이 바로 사용할 수 있다. macOS에서는 네이티브 탭과 split을 제공하고 zsh를 포함한 주요 셸의 integration도 자동으로 주입한다. 이 글의 macOS 명령은 모두 Ghostty에서 실행한다.

먼저 Homebrew로 Herdr를 설치한다.

```bash
brew install herdr
herdr --version
```

[Codex CLI 공식 문서](https://learn.chatgpt.com/docs/codex/cli)의 Homebrew 방식을 쓰면 Herdr와 함께 패키지 관리자를 하나로 맞출 수 있다. Ghostty에서 이어서 설치한다.

```bash
brew install --cask codex
codex --version
```

Codex를 한 번 실행해 로그인과 초기 설정을 마친다. Codex 화면에서 [`/exit`](https://learn.chatgpt.com/docs/developer-commands?surface=cli)를 입력해 Ghostty에서 실행 중인 셸로 돌아온 뒤 Herdr integration을 설치한다.

```bash
codex
herdr integration install codex
herdr integration status
```

이 integration은 Codex의 세션 정보를 Herdr에 전달한다. Herdr 공식 문서에 따르면 Codex의 동작 상태는 화면을 바탕으로 감지하고 integration은 세션 복원에 필요한 식별자를 보고한다. 단순히 Codex 프로세스를 pane에 띄우는 것보다 한 단계 더 이어지는 셈이다.

프로젝트 디렉터리에서 `herdr`를 실행하면 로컬 세션을 시작하거나 기존 세션에 다시 붙는다.

```bash
cd ~/Workspace/my-project
herdr
```

workspace가 열리면 그 안에서 `codex`를 실행한다.

## 원격 머신에도 환경을 구축한다

Ubuntu와 Jetson에서도 코드가 있는 머신에 Herdr와 Codex CLI를 설치한다. Herdr는 Linux와 macOS에서 공식 설치 스크립트를 제공한다.

```bash
curl -fsSL https://herdr.dev/install.sh | sh
herdr --version
```

설치 직후 `herdr`를 찾지 못하면 설치 프로그램이 출력한 PATH 안내를 확인하고 SSH 셸을 다시 연다.

이어서 OpenAI가 macOS와 Linux에 제공하는 standalone installer로 Codex CLI를 설치한다. Ubuntu 서버와 Jetson의 SSH 셸에서 같은 명령을 쓰면 된다.

```bash
curl -fsSL https://chatgpt.com/codex/install.sh | sh
codex --version
```

여기서도 `codex`를 찾지 못하면 SSH 셸을 다시 연 뒤 버전을 확인한다.

프로젝트 디렉터리에서 `codex`를 처음 실행하면 로그인 방식을 선택할 수 있다. 인증은 머신마다 따로 진행하며 로그인을 마쳤으면 `/exit`로 셸에 돌아온다.

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
cd ~/Workspace/my-project
herdr
```

처음 한 번은 코드가 있는 디렉터리에서 Herdr를 실행해 프로젝트 workspace를 만들고 그 안에서 `codex`를 실행한다. Orin에서도 같은 초기 설정을 해둔다. 이 경로에서는 셸과 Herdr 클라이언트, 서버가 모두 원격 머신에서 실행된다. 평소 SSH 셸 안에서 작업하거나 휴대전화의 SSH 클라이언트로 접속할 때 이해하기 쉽다.

Ghostty에서 원격 세션을 로컬처럼 열고 싶다면 [Herdr의 remote attach](https://herdr.dev/docs/persistence-remote/)를 쓸 수 있다.

```bash
herdr --remote xavier
```

이때 로컬 Herdr는 얇은 클라이언트가 된다. SSH를 통해 원격 Herdr 서버를 시작하거나 기존 서버에 붙고 화면만 현재 터미널로 가져온다. 이미지 클립보드처럼 로컬 데스크톱에 의존하는 기능도 원격 세션으로 이어줄 수 있다는 점이 일반 SSH 접속과 다르다.

반복해서 접속할 머신은 Mac의 `~/.ssh/config`에 별칭을 둔다. 아래의 `HostName`과 `User`는 실제 주소와 계정명으로 바꾼다.

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

## Ghostty 탭에서는 Herdr 클라이언트를 실행한다

Ghostty의 네이티브 탭과 split은 여러 터미널 화면을 배치하는 기능이다. 프로젝트와 Codex 작업 상태는 Ghostty가 아니라 Herdr가 관리한다. 각 Ghostty 탭에서는 필요한 Herdr 클라이언트를 실행한다.

```bash
# 첫 번째 Ghostty 탭
herdr

# 두 번째 Ghostty 탭
herdr --remote xavier

# 세 번째 Ghostty 탭
herdr --remote orin
```

Ghostty는 Herdr 클라이언트의 터미널 화면을 렌더링하고 키 입력을 전달한다. 어느 머신에 접속할지는 `herdr --remote`의 SSH 별칭이 결정한다. 프로젝트와 이슈별 작업 공간은 접속한 Herdr에서 전환하고 실제 코드 작업은 Codex agent 이름으로 구분한다.

```text
Ghostty tab      ── 로컬·원격 Herdr로 들어가는 접속점
Herdr workspace ── 프로젝트 · 이슈별 지속 작업 공간
Codex agent      ── 구현 · 분석 · 테스트를 수행하는 세션
```

Ghostty 탭을 닫으면 Herdr 클라이언트 연결은 끝나지만 Herdr 서버의 workspace와 pane은 남는다. 새 탭에서 같은 `herdr` 명령을 실행하면 유지 중인 workspace와 Codex 세션으로 돌아간다.

`herdr --remote`에서 인증 문제가 나면 Herdr부터 의심하기보다 `ssh xavier`가 정상적으로 연결되는지 먼저 확인한다. 암호가 걸린 키를 비대화형 환경에서 사용할 때는 `ssh-agent`에 키가 올라가 있어야 한다.

## Codex는 코드가 있는 곳에서 실행한다

이 구성에서 가장 중요한 원칙은 Codex를 어느 머신에서 실행하느냐다.

> 코드와 빌드·테스트 환경이 실제로 존재하는 머신에서 Codex를 실행한다.

Ubuntu에 있는 프로젝트를 Mac의 Codex가 SSH 명령으로 조작하게 만들 수도 있다. 그러나 그렇게 하면 파일 접근과 명령 실행, 권한, 환경 변수의 경계가 두 머신에 걸친다. 반대로 Ubuntu에서 Codex를 실행하면 그 머신의 컴파일러와 Python 환경, Podman 컨테이너, 테스트 도구를 직접 사용한다.

Jetson에서는 이 차이가 더 분명하다. CUDA와 JetPack, 장치 접근이 필요한 테스트는 Mac에서 재현할 수 없다. Mac의 Ghostty는 로컬 Herdr 클라이언트의 화면을 렌더링하고 키 입력을 전달할 뿐이다. Codex와 소스 코드, 빌드와 테스트는 Jetson 안에 함께 둔다.

```text
Ghostty
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

## Herdr는 한 번만 실행한다

로컬 Herdr 안에서 SSH로 접속한 뒤 원격 Herdr를 다시 실행하면 세션 경계와 키 바인딩이 겹친다. Ghostty 탭에서 필요한 Herdr에 바로 붙는 편이 단순하다.

```text
비권장: Ghostty → Mac Herdr → SSH → Ubuntu Herdr → Codex

권장:
Ghostty
├── Local 탭  ── herdr
├── Ubuntu 탭 ── herdr --remote xavier
└── Jetson 탭 ── herdr --remote orin
```

## Codex가 이슈별 worktree를 만든다

`herdr --remote orin`으로 접속했다면 이미 Jetson의 Herdr 안에 있다. `master` workspace에서 실행 중인 Codex에게 GitHub 이슈 번호를 주고 최신 `origin/master`를 기준으로 새 worktree와 Herdr workspace, Codex 세션까지 준비하도록 맡긴다.

여기서 Git worktree는 같은 저장소의 다른 분기를 별도 디렉터리에 꺼내 놓은 코드 작업 공간이다. Herdr workspace는 그 디렉터리에서 실행되는 터미널 묶음이고 pane은 그 안의 개별 터미널이다. agent는 pane에서 실행 중인 Codex처럼 Herdr가 상태를 추적하는 프로그램을 뜻한다.

작업을 시작하기 전에 Orin의 프로젝트 디렉터리에서 원격 저장소 이름과 현재 분기를 확인한다.

```bash
cd ~/Workspace/my-project
git remote -v
git branch --show-current
```

이 자동화에는 GitHub CLI인 `gh`와 JSON 결과에서 pane ID를 꺼내는 `jq`가 필요하다. Orin의 Ubuntu 셸에서 설치하고 GitHub 인증까지 확인한다. `gh auth setup-git`은 이후 `git push`에서도 GitHub CLI의 인증 정보를 사용하도록 Git을 설정한다.

```bash
sudo apt update
sudo apt install gh jq
gh auth login
gh auth setup-git
gh auth status
jq --version
```

Herdr pane 안에서 Codex가 workspace와 agent를 제어하려면 먼저 `herdr --skill`의 지침을 읽게 한다. 같은 이슈의 worktree나 agent가 이미 있으면 새로 만들지 않고 기존 작업으로 이동하도록 요청한다.

예를 들어 기존 Codex에는 이렇게 요청한다.

> 먼저 `herdr --skill`을 실행하고 그 지침을 따라줘. GitHub issue #123을 읽고 기존 `issue/123` worktree나 `issue_123` agent가 있는지 확인해. 있으면 그 agent로 이동하고 없으면 현재 `master` workspace가 깨끗한지 확인해. 커밋하지 않은 변경이 있으면 중단해서 알려주고 깨끗하면 최신 `origin/master` 기준으로 worktree를 새 Herdr workspace에 만들어. 첫 pane에서 `issue_123`이라는 새 Codex를 시작하고 이슈 구현과 테스트를 맡긴 뒤 새 agent로 포커스를 옮겨줘. 현재 `master` workspace의 파일은 수정하지 마.

[Herdr의 worktree 명령](https://herdr.dev/docs/cli-reference/#worktrees)은 checkout을 만들면서 별도의 workspace와 첫 pane도 함께 생성한다. 이어서 `agent start`로 그 pane에서 새 Codex를 실행할 수 있다.

```text
Ghostty
└── herdr --remote orin
    └── Jetson의 Herdr
        ├── master workspace
        │   └── Codex ── GitHub issue #123 확인
        └── issue/123 worktree workspace
            └── 새 Codex ── 구현 · 테스트
```

기존 Codex가 새 이슈를 처음 처리할 때 실행할 명령의 흐름은 다음과 같다. `gh`로 이슈를 확인하고 Herdr가 돌려준 pane ID를 이용해 새 Codex를 시작한다. 다음 명령은 Orin의 `master` workspace에서 실행한다.

```bash
ISSUE=123
AGENT_NAME="issue_$ISSUE"

herdr --skill
cd "$(git rev-parse --show-toplevel)"
gh issue view "$ISSUE"
git fetch origin
test "$(git branch --show-current)" = master || {
  echo "master 분기에서 실행해야 합니다."
  exit 1
}
test -z "$(git status --porcelain)" || {
  echo "master workspace에 커밋하지 않은 변경이 있습니다."
  exit 1
}

created=$(herdr worktree create \
  --cwd "$PWD" \
  --branch "issue/$ISSUE" \
  --base origin/master \
  --label "issue-$ISSUE" \
  --no-focus)

pane_id=$(printf '%s\n' "$created" |
  jq -r '.result.root_pane.pane_id')

herdr agent start "$AGENT_NAME" \
  --kind codex \
  --pane "$pane_id"

herdr agent prompt "$AGENT_NAME" \
  "GitHub issue #$ISSUE를 구현하고 관련 테스트를 실행해. 변경 내용과 테스트 결과를 보고해."

herdr agent focus "$AGENT_NAME"
```

같은 이슈를 다시 시작할 때는 먼저 `herdr worktree list --cwd "$PWD"`와 `herdr agent list`를 확인한다. 이미 실행 중인 agent가 있으면 `herdr agent focus issue_123`으로 돌아간다.

기존 Codex는 `master` workspace에 남고 실제 수정은 새 Codex가 worktree에서 진행한다. 작업 상태와 결과도 Herdr sidebar에서 이슈별 agent로 구분해 볼 수 있다.

작업이 끝나면 새 workspace에서 변경 내용과 테스트 결과를 확인하고 커밋한다. 아래 명령은 `issue/123` worktree 안에서 실행한다.

```bash
git status
git diff
git add -p
git commit -m "fix: resolve issue #123"
git push -u origin issue/123
gh pr create \
  --fill \
  --base master \
  --head issue/123 \
  --body "Closes #123"
```

PR을 검토하고 병합(merge)한 뒤에는 Herdr sidebar에서 묶인 `issue/123` 하위 workspace를 선택하고 `Delete worktree checkout...`을 실행한다. CLI에서는 `herdr worktree list`로 workspace ID를 확인한 뒤 다음처럼 삭제할 수도 있다.

```bash
herdr worktree list
# WORKSPACE_ID를 위 명령에서 확인한 실제 값으로 바꾼다.
herdr worktree remove --workspace WORKSPACE_ID
```

Git이 수정되거나 추적되지 않은 파일을 발견하면 안전한 제거를 거부하므로 강제 삭제 전에 남은 변경을 먼저 확인한다. Herdr는 checkout만 제거하며 분기는 자동으로 삭제하지 않는다. `master` workspace로 돌아가 병합된 로컬 분기를 지우고 GitHub에서 원격 분기가 남아 있다면 함께 정리한다.

```bash
git branch -d issue/123
git push origin --delete issue/123
```

## 창을 닫아도 작업은 남는다

Herdr 클라이언트는 `Ctrl-b q`로 detach할 수 있다. pane과 그 안의 에이전트는 백그라운드에서 계속 실행된다. 로컬에서는 다시 `herdr`, 원격에서는 다시 `herdr --remote <host>`를 실행하면 기존 작업 공간으로 돌아간다.

```bash
# 원격 세션에서 빠져나온 뒤 다시 접속
herdr --remote xavier
```

detach는 세션을 종료하지 않는다. pane 하나를 종료하려면 UI에서 닫거나 `herdr pane close <pane_id>`를 사용한다. `herdr server stop`은 모든 workspace의 작업을 끝낼 때만 사용한다. 이 차이를 알고 있으면 터미널 창의 수명과 작업의 수명을 분리할 수 있다.

대규모 코드 분석이나 리팩터링처럼 시간이 오래 걸리는 작업에서 특히 유용하다. MacBook의 덮개를 닫거나 네트워크가 끊기더라도 원격 머신의 Herdr 서버와 pane은 그곳에 남는다. 다시 연결했을 때 새 Codex 대화를 열고 맥락부터 복원하는 대신 기존 세션을 이어간다.

AI 코딩 에이전트를 오래 사용할수록 터미널 창을 몇 개 열었는지보다 **어느 프로젝트의 에이전트가 어떤 머신에서 작업 중인지**가 중요해진다. Ghostty는 Herdr 클라이언트가 출력한 터미널 화면을 빠르고 네이티브하게 렌더링한다. Herdr는 클라이언트 연결이 끊긴 뒤에도 세션을 유지하고 Codex CLI는 코드가 있는 머신에서 실제 변경과 검증을 수행한다. 이 조합의 장점은 도구를 더 많이 쌓는 데 있지 않다. 화면과 세션, 실행 환경의 경계를 분명하게 나누는 데 있다.
