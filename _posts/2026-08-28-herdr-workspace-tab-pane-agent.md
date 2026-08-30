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

## Agents 목록의 상태등은 무엇을 뜻할까?

Agents 목록에서 agent 이름 앞에 붙는 점은 해당 agent의 현재 상태를 나타냅니다. 기본 설정에서는 같은 모양의 점이 색으로 바뀌므로 처음 보면 서로 다른 상태를 구분하기 어렵습니다. [Herdr의 agent 상태](https://herdr.dev/docs/agents/)는 다음 다섯 가지입니다.

| <span style="white-space: nowrap;">기본 표시</span> | 상태 | 의미 |
| --- | --- | --- |
| <span style="white-space: nowrap;"><span style="color: #f9e2af; font-size: 1.25em; line-height: 1;" aria-label="노란색 채운 원">●</span> 노란색</span> | `working` | agent가 요청을 처리하고 있습니다. |
| <span style="white-space: nowrap;"><span style="color: #f38ba8; font-size: 1.25em; line-height: 1;" aria-label="빨간색 채운 원">●</span> 빨간색</span> | `blocked` | 권한 승인이나 질문에 대한 답처럼 사용자의 입력을 기다립니다. |
| <span style="white-space: nowrap;"><span style="color: #94e2d5; font-size: 1.25em; line-height: 1;" aria-label="청록색 채운 원">●</span> 청록색</span> | `done` | 작업을 마쳤지만 아직 결과를 확인하지 않았습니다. |
| <span style="white-space: nowrap;"><span style="color: #a6e3a1; font-size: 1.25em; line-height: 1;" aria-label="초록색 빈 원">○</span> 초록색</span> | `idle` | 결과를 확인했거나 새 요청을 받을 수 있는 대기 상태입니다. |
| <span style="white-space: nowrap;"><span style="color: #6c7086; font-size: 1.25em; line-height: 1;" aria-label="회색 가운데점">·</span> 회색</span> | `unknown` | Herdr가 agent 상태를 판별하지 못했습니다. 일반 셸이나 지원하지 않는 프로그램도 여기에 해당할 수 있습니다. |

표의 모양과 색은 Herdr의 기본 `dots` 설정과 Catppuccin Mocha 테마를 기준으로 재현했습니다. 다른 테마를 선택하거나 색을 직접 설정하면 색조는 달라지지만 상태별 의미는 같습니다. `done`과 `idle`의 차이는 agent 프로세스가 끝났는지가 아니라 결과를 확인했는지에 있습니다. 작업이 끝난 agent는 청록색 `done`으로 남아 있다가 해당 agent를 열어 결과를 보면 초록색 `idle`로 바뀝니다. 빨간색 `blocked`가 보이면 agent가 실패했다고 단정하기보다 pane을 열어 승인이나 답변을 기다리는지 확인합니다.

왼쪽 위 Spaces 목록의 상태등은 workspace 안에 있는 agent 상태를 모아서 보여 줍니다. [Herdr의 상태 집계](https://herdr.dev/docs/agents/#state-rollups)에서는 하나라도 `blocked`인 agent가 있으면 workspace도 빨간색으로 표시하고, 실행 중인 agent가 있으면 활성 상태로 표시합니다. 따라서 workspace의 점만 보고 끝내지 말고 Agents 목록에서 어느 agent에 입력이 필요한지 확인하는 편이 정확합니다.

색만으로 구분하기 어렵다면 상태마다 다른 기호를 쓰도록 바꿀 수 있습니다. `~/.config/herdr/config.toml`에 다음 설정을 추가한 뒤 `herdr server reload-config`를 실행합니다. [Herdr 설정 문서](https://herdr.dev/docs/configuration/#ui-and-sidebar)에 따르면 이 설정은 `blocked`, `working`, `done`, `idle`, `unknown`을 색뿐 아니라 서로 다른 모양으로 표시합니다.

```toml
[ui]
status_indicators = "symbols"
```

## 무엇을 나눠야 할까?

| 상황 | 사용할 단위 |
| --- | --- |
| 다른 저장소나 독립된 작업 | workspace |
| 같은 프로젝트의 다른 화면 구성 | tab |
| 계속 함께 볼 터미널 | pane |
| 같은 종류의 실행 중인 프로그램 | agent 이름 |

Agents 목록의 첫째 줄이 겹치면 workspace 구성을 확인합니다. `codex`가 여러 개라서 헷갈리면 agent 이름을 붙이고 분할된 터미널의 역할이 불분명하면 pane 이름을 붙입니다. 자세한 정의와 명령은 [Herdr Concepts](https://herdr.dev/docs/concepts/)와 [CLI Reference](https://herdr.dev/docs/cli-reference/)에서 확인할 수 있습니다.
