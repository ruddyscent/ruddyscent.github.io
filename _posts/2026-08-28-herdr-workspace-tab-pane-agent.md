---
layout: post
title: Herdr의 workspace, tab, pane, agent 구분하기
subtitle: 다른 저장소는 workspace로, 같은 프로젝트의 화면은 tab과 pane으로 나눈다
tags: [codex, herdr, terminal, workflow]
cover-img: /assets/img/develop.jpeg
thumbnail-img: /assets/img/terminal.webp
share-img: /assets/img/develop.jpeg
author: 전경원
description: Herdr의 workspace, tab, pane, agent가 무엇을 구분하는지와 Agents 목록의 이름을 읽는 방법을 정리한다.
---

Herdr에서는 저장소나 독립된 작업을 workspace로 나누고 같은 프로젝트 안의 화면 구성은 tab과 pane으로 나눕니다. 각 pane에서 실행하는 Codex 같은 프로그램은 agent로 인식합니다. 이름이 비슷해 헷갈리기 쉬운 네 단위는 다음 순서로 이어집니다.

```text
Workspace
└── Tab
    └── Pane
        └── Agent
```

구분 기준은 간단합니다.

- 저장소나 독립된 작업이 다르면 **workspace**를 나눕니다.
- 같은 프로젝트에서 화면 구성을 전환하려면 **tab**을 만듭니다.
- 여러 터미널을 동시에 보려면 **pane**을 나눕니다.
- pane에서 실행 중인 Codex 같은 프로그램은 **agent**입니다.

## Herdr 화면에서 이름은 어디에 표시될까?

한 workspace의 tab을 두 pane으로 나누고 각 pane에서 Codex를 실행한 화면을 단순화하면 다음과 같습니다.

```text
+----------------------+--------------------------------------------------+
| spaces               | tab: coding                                      |
| * project-a          +--------------------------------------------------+
|                      | pane: writer                                      |
|                      | Codex                                             |
+----------------------+--------------------------------------------------+
| agents       grouped | pane: reviewer                                    |
| * project-a          | Codex                                             |
|   codex              |                                                   |
| * project-a          |                                                   |
|   codex              |                                                   |
+----------------------+--------------------------------------------------+
```

왼쪽 위 `spaces`는 workspace 목록입니다. 오른쪽에는 선택한 tab이 있고 그 안을 `writer`와 `reviewer`라는 두 pane으로 나눴습니다.

왼쪽 아래 Agents 목록에는 `project-a / codex`가 두 번 나타납니다. 두 agent가 같은 workspace에 속하고 둘 다 Codex이기 때문입니다. pane 이름과 현재 디렉터리는 이 목록의 이름을 정하는 데 쓰이지 않습니다.

| 화면에 보이는 이름 | 뜻 |
| --- | --- |
| Agents 목록 첫째 줄 | workspace 이름. tab이 여러 개면 tab 이름도 함께 표시됩니다. |
| Agents 목록 둘째 줄 | agent 이름 |
| pane 테두리 | pane 이름 |
| pane 안의 `cwd` | 현재 셸의 작업 디렉터리 |

따라서 pane에서 `cd`로 다른 저장소에 들어가도 workspace는 바뀌지 않습니다. 서로 다른 저장소의 Codex가 Agents 목록에서 같은 workspace 이름 아래 보인다면, 두 터미널을 하나의 workspace 안에 만든 경우입니다.

## Workspace: 저장소와 작업의 경계

Workspace는 Herdr의 최상위 작업 단위입니다. 저장소, Git worktree, 서로 독립된 작업은 별도 workspace로 두는 편이 좋습니다.

```text
Herdr
├── workspace: blog
│   └── tab: writing
└── workspace: gmes
    └── tab: benchmark
```

기본 키 바인딩에서는 `Ctrl-b Shift-n`으로 workspace를 만듭니다. CLI에서는 시작 디렉터리와 이름을 함께 지정할 수 있습니다.

```bash
herdr workspace create --cwd ~/Workspace/gmes --label gmes
```

## Tab: 같은 프로젝트의 화면 구성

Tab은 한 workspace 안에서 화면 구성을 전환하는 단위입니다. 같은 프로젝트의 코딩, 테스트, 서버, 리뷰 화면을 나눌 때 사용합니다.

```text
workspace: project-a
├── tab: coding
├── tab: tests
└── tab: server
```

새 tab은 `Ctrl-b c`로 만듭니다. tab을 바꾸면 그 안의 pane 배치 전체가 전환됩니다.

## Pane: 동시에 보는 터미널

Pane은 셸과 프로세스가 실행되는 실제 터미널입니다. Codex 옆에 테스트 결과나 개발 서버 로그를 계속 띄워 둘 때 적합합니다.

```text
Ctrl-b v    # 좌우 분할
Ctrl-b -    # 상하 분할
```

Pane 이름은 터미널의 역할을 표시합니다. `Ctrl-b Shift-p`로 현재 pane의 이름을 바꿀 수 있습니다. 터미널에서는 `herdr pane rename` 명령을 사용합니다.

```bash
herdr pane rename <pane_id> test-runner
```

이 이름은 pane 테두리에 표시되며 Agents 목록의 이름은 바꾸지 않습니다.

## Agent: pane에서 실행 중인 프로그램

Agent는 Herdr가 pane 안에서 감지한 Codex CLI, Claude Code, Pi 같은 프로그램입니다. 같은 workspace에서 Codex를 여러 개 실행하면 기본 이름인 `codex`가 반복될 수 있습니다.

Agents 목록에서 역할을 구분하려면 agent 이름을 바꿉니다.

```bash
herdr agent rename <target> writer
herdr agent rename <target> reviewer
```

`pane rename`은 화면의 터미널을 구분하고 `agent rename`은 Agents 목록과 agent 제어 대상을 구분합니다.

## 무엇을 나눠야 할까?

| 상황 | 사용할 단위 |
| --- | --- |
| 다른 저장소나 독립된 작업 | workspace |
| 같은 프로젝트의 다른 화면 구성 | tab |
| 계속 함께 볼 터미널 | pane |
| 같은 종류의 실행 중인 프로그램 | agent 이름 |

Agents 목록의 첫째 줄이 겹치면 workspace 구성을 확인합니다. `codex`가 여러 개라서 헷갈리면 agent 이름을 붙이고 분할된 터미널의 역할이 불분명하면 pane 이름을 붙입니다. 자세한 정의와 명령은 [Herdr Concepts](https://herdr.dev/docs/concepts/)와 [CLI Reference](https://herdr.dev/docs/cli-reference/)에서 확인할 수 있습니다.
