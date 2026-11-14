---
layout: post
title: Neovim에서 Codex·Claude Code 쓰기
subtitle: 편집기와 에이전트를 Herdr pane에 하나씩 띄운다
tags: [neovim, codex, claude-code, herdr, workflow]
cover-img: /assets/img/develop.jpeg
thumbnail-img: /assets/img/editor.webp
share-img: /assets/img/develop.jpeg
author: 전경원
description: Neovim에서 Codex CLI와 Claude Code CLI를 연동하는 방법을 비교하고, Herdr의 tab 하나를 pane으로 나눠 편집기와 에이전트를 함께 쓰는 구성을 정리한다.
---

터미널에서 Codex CLI와 Claude Code를 쓰다 보면 vim 편집기를 쓸 일이 많습니다. 에이전트가 고친 코드를 바로 열어 보고, 직접 손볼 부분은 손으로 고치고, 다시 에이전트에게 넘기는 흐름이 한 화면에서 이어지기 때문입니다. 이 글에서는 [Neovim](https://neovim.io/)과 두 에이전트를 연동하는 방법을 정리하고, [Herdr](https://herdr.dev/docs/quick-start/)의 tab 하나를 pane으로 나눠 함께 쓰는 구성으로 마무리합니다.

이 글은 Neovim 0.12, Claude Code 2.1.325를 기준으로 합니다. Herdr와 Codex CLI 설치는 [Ghostty와 Herdr로 Codex CLI 작업 환경 나누기](/2026-08-24-ghostty-herdr-codex-cli-workflow/)에서, Herdr의 workspace·tab·pane 구분은 [Herdr의 workspace, tab, pane, agent 구분하기](/2026-08-28-herdr-workspace-tab-pane-agent/)에서 다뤘습니다.

## 연동 방식은 세 갈래다

Neovim과 터미널 에이전트를 묶는 방법은 크게 세 가지입니다.

| 방식 | 에이전트가 실행되는 곳 | Neovim이 알려 주는 정보 |
| --- | --- | --- |
| Neovim 내장 터미널 | Neovim의 `:terminal` 버퍼 | 없음 |
| 터미널 래퍼 플러그인 | 플러그인이 띄운 터미널 버퍼 | 플러그인에 따라 버퍼 갱신 정도 |
| IDE 연동 플러그인 | 플러그인 터미널 또는 바깥 터미널 | 열린 파일, 선택 영역, diff |

첫 번째와 두 번째는 에이전트를 Neovim 화면 안에 띄우는 데 초점이 있습니다. 세 번째는 에이전트가 어디서 실행되든 Neovim의 상태를 에이전트에 전달하는 데 초점이 있습니다. 이 글에서 실제로 설정하는 구성은 세 번째 방식에서 에이전트를 Neovim 밖, Herdr pane에 두는 형태입니다. 나머지 방식은 따라 할 설정 없이 장단점만 짚고 넘어갑니다.

## 세 방식의 장단점

**Neovim 내장 터미널**은 플러그인 없이 Neovim의 [터미널 기능](https://neovim.io/doc/user/terminal/)으로 바로 시작할 수 있다는 점이 장점입니다. 대신 키 입력과 프로세스를 Neovim과 함께 씁니다. 터미널 모드를 `Esc`로 빠져나오도록 매핑하는 설정이 많은데, 그러면 Claude Code에서 응답을 중단하는 `Esc`와 되감기를 여는 `Esc Esc`가 에이전트에 닿지 않습니다. 에이전트가 Neovim의 자식 프로세스이므로 `:restart`나 설정 실수로 Neovim이 종료되면 에이전트 세션도 함께 끝납니다. 에이전트가 파일을 고쳐도 이미 열린 버퍼는 자동으로 다시 읽히지 않습니다.

**터미널 래퍼 플러그인**은 터미널 버퍼를 띄우고 닫는 일을 명령 하나로 묶어 줍니다. [greggh/claude-code.nvim](https://github.com/greggh/claude-code.nvim)은 Claude Code 전용 터미널 버퍼를 만들고, Claude Code가 수정한 파일을 감지해 버퍼를 다시 읽습니다. [johnseth97/codex.nvim](https://github.com/johnseth97/codex.nvim)은 Codex CLI를 띄우는 창을 토글합니다. 내장 터미널보다 편하지만 에이전트가 Neovim 안에서 실행되므로, Neovim이 종료되면 세션도 함께 끝난다는 점은 같습니다.

**IDE 연동 플러그인**은 Neovim의 상태를 에이전트에 전달합니다. Claude Code는 VS Code와 JetBrains 확장과 통신하는 IDE 연동 기능을 갖고 있고, [coder/claudecode.nvim](https://github.com/coder/claudecode.nvim)은 이 확장들이 쓰는 WebSocket 기반 [Model Context Protocol(MCP)](https://modelcontextprotocol.io/) 통신을 Neovim에서 구현합니다. 연결되면 Claude Code가 Neovim에서 지금 보고 있는 파일과 Visual 모드로 고른 영역을 알 수 있고, Claude Code가 제안한 수정을 Neovim의 diff 화면에서 검토할 수 있습니다. 에이전트를 Neovim 밖에서 실행해도 이 연동이 유지되므로 Herdr pane과 조합할 수 있습니다.

Codex CLI용으로도 claudecode.nvim의 구조를 옮긴 [ishiooon/codex.nvim](https://github.com/ishiooon/codex.nvim)과 [rxwen/codex.nvim](https://github.com/rxwen/codex.nvim)이 있습니다. 둘 다 Visual 모드로 고른 코드를 Codex로 보내고 제안된 수정을 diff로 검토하는 흐름을 목표로 합니다. 다만 개인이 관리하는 프로젝트이고 Codex CLI도 빠르게 바뀌고 있으므로, 이 글의 구성에서는 Codex를 플러그인 없이 pane에서 실행하고 Neovim은 파일 자동 갱신으로 결과를 받아 봅니다.

## 결론: 한 tab을 pane으로 나눠 쓴다

에이전트를 Neovim 안에 넣는 대신 Herdr의 tab 하나를 pane으로 나눠 Neovim, Claude Code, Codex를 나란히 둡니다.

```text
workspace: my-project
└── tab: coding
    ├── pane: editor  ── nvim
    ├── pane: claude  ── claude --ide
    └── pane: codex   ── codex
```

이렇게 나누면 앞에서 본 제약이 대부분 사라집니다.

- 각 프로그램이 자기 키 입력을 그대로 받으므로 `Esc`를 두고 Neovim과 에이전트가 다투지 않습니다.
- 에이전트 세션은 Neovim과 따로 실행되므로 Neovim을 다시 시작해도 대화가 이어집니다.
- Herdr가 각 pane의 Claude Code와 Codex를 agent로 인식하므로 Agents 목록의 상태등으로 어느 에이전트가 작업 중인지, 어느 에이전트가 승인을 기다리는지 볼 수 있습니다.
- Herdr 클라이언트에서 detach해도 세 pane이 모두 서버에 남습니다.

### pane 만들기

프로젝트 workspace의 tab에서 `Ctrl-b v`로 좌우를 나누고, 오른쪽 pane을 `Ctrl-b -`로 다시 위아래로 나눕니다. 왼쪽 pane에서 `nvim`, 오른쪽 위에서 `claude --ide`, 오른쪽 아래에서 `codex`를 실행합니다. 세 pane 모두 같은 프로젝트 디렉터리에서 시작합니다.

pane 테두리의 이름은 `Ctrl-b Shift-p`로, Agents 목록의 이름은 `herdr agent rename`으로 바꿉니다. 같은 workspace에 Codex를 여러 개 띄울 때 역할을 구분하기 좋습니다.

### Claude Code를 Neovim에 붙이기

claudecode.nvim은 [Neovim 기본 패키지 기능](https://neovim.io/doc/user/pack/)으로 설치할 수 있습니다. [Tree-sitter와 GitHub 테마를 설치할 때](/2026-11-07-neovim-github-highlight/)와 같은 디렉터리 구조를 사용합니다. Neovim 안에 터미널 창을 띄우지 않으므로 터미널 기능을 보강하는 snacks.nvim은 설치하지 않아도 됩니다.

```sh
git clone https://github.com/coder/claudecode.nvim.git \
  ~/.config/nvim/pack/github/start/claudecode.nvim
```

이 플러그인은 기본적으로 Neovim 안에 Claude Code 터미널 창을 띄웁니다. `terminal.provider`를 `"none"`으로 두면 창이나 버퍼를 만들지 않고 WebSocket 서버만 실행하므로, Claude Code는 Herdr pane에서 따로 실행할 수 있습니다. `~/.config/nvim/init.lua`에 다음 설정을 추가합니다.

```lua
-- Run only the WebSocket server; Claude Code runs in its own Herdr pane
require("claudecode").setup({
  terminal = {
    provider = "none",
  },
})

-- Send the Visual selection to Claude Code
vim.keymap.set("v", "<leader>as", "<cmd>ClaudeCodeSend<cr>", {
  desc = "Send selection to Claude Code",
})
```

Neovim을 다시 시작한 뒤 Claude Code pane에서 [`/ide`](https://code.claude.com/docs/en/commands)를 실행하면 연결할 수 있는 편집기 목록에 Neovim이 나타납니다. Claude Code를 [`--ide`](https://code.claude.com/docs/en/cli-reference) 옵션으로 시작하면 사용할 수 있는 편집기가 하나일 때 자동으로 연결합니다. 플러그인은 연결 정보를 `~/.claude/ide/` 아래에 포트 번호로 된 lock 파일로 남기므로, 목록에 Neovim이 보이지 않으면 이 파일이 생겼는지 먼저 확인합니다.

연결한 뒤 자주 쓰는 명령은 다음과 같습니다. 플러그인에는 기본 키 매핑이 없으므로 필요한 명령에만 직접 키를 붙입니다. 위 설정에서는 `:ClaudeCodeSend`에만 키를 붙였습니다.

| 명령 | 동작 |
| --- | --- |
| `:ClaudeCodeSend` | Visual 모드에서 고른 영역을 Claude Code로 보내기 |
| `:ClaudeCodeAdd <file>` | 파일이나 줄 범위를 Claude Code의 맥락에 추가하기 |
| `:ClaudeCodeDiffAccept` / `:ClaudeCodeDiffDeny` | 대기 중인 diff 받아들이기·거절하기 |

Claude Code가 제안한 수정은 Neovim의 diff 화면으로 열립니다. diff는 `:w`로 받아들이고 `:q`로 거절하며, 받아들이기 전에 제안된 내용을 직접 고칠 수도 있습니다.

### 에이전트가 고친 파일을 다시 읽기

pane을 나누면 에이전트가 고친 파일을 Neovim이 알아서 다시 읽게 해야 합니다. `autoread` 옵션만으로는 부족할 수 있습니다. Neovim은 포커스가 돌아오거나 외부 명령을 실행한 뒤에 파일 변경을 확인하는데, 터미널 멀티플렉서를 거치면 포커스 이벤트가 전달되지 않을 수 있기 때문입니다. 버퍼에 들어가거나 커서를 잠시 멈췄을 때도 `:checktime`을 실행하도록 autocmd를 추가합니다.

```lua
-- Auto-reload files changed outside Neovim (e.g. by an agent in another pane)
vim.opt.autoread = true
vim.api.nvim_create_autocmd({ "FocusGained", "BufEnter", "CursorHold" }, {
  group = vim.api.nvim_create_augroup("AutoReload", {
    clear = true,
  }),
  callback = function()
    -- Skip in the command-line window (q:), where :checktime raises an error
    if vim.fn.getcmdwintype() == "" then
      vim.cmd.checktime()
    end
  end,
})
```

`CursorHold`는 커서가 `updatetime`(기본 4초) 동안 멈췄을 때 실행됩니다. 더 빨리 반영되길 원하면 `vim.opt.updatetime = 1000`처럼 값을 줄입니다.

Codex에는 Neovim의 선택 영역이 전달되지 않으므로, Codex에게 일을 맡길 때는 Codex 입력창에서 `@`로 파일 경로를 붙여 범위를 알려 줍니다.

### 오래 걸리는 작업은 tab을 따로 둔다

한 tab에 pane을 너무 많이 두면 Neovim 화면이 좁아집니다. 지금 보고 있는 코드와 바로 주고받는 에이전트는 같은 tab의 pane에 두고, 테스트 로그나 오래 걸리는 리팩터링처럼 결과만 확인하면 되는 작업은 같은 workspace의 다른 tab에 둡니다. 다른 저장소나 이슈별 worktree는 [이전 글](/2026-08-24-ghostty-herdr-codex-cli-workflow/)처럼 workspace를 따로 만듭니다.

## Codex CLI, Claude Code CLI, Neovim 치트시트

이 구성에서 자주 쓰는 키와 명령을 치트시트로 정리했습니다. 세 자료 모두 영문 2페이지이며 내용은 같고 판형만 다릅니다. 세로형은 A4로 인쇄해 곁에 두기 좋고 가로형은 모니터 한쪽에 띄워 놓고 보기 좋습니다.

Codex CLI 치트시트에는 대화형 단축키와 slash command, 세션 복원, 코드 리뷰, `codex exec` 자동화, sandbox와 승인 정책, 설정과 플러그인 관련 명령을 정리했습니다.

- [Codex CLI 치트시트 - 세로형 PDF](https://www.dropbox.com/scl/fi/cj5fvo992vj7v7q6tkkzu/codex-cli-cheatsheet-en.pdf?rlkey=zrdw9cgvfbphx9ie7mvnoooxq&st=xso8ebe3&dl=0)
- [Codex CLI 치트시트 - 가로형 PDF](https://www.dropbox.com/scl/fi/pf63kpfo3ugucxsbv0s0e/codex-cli-cheatsheet-landscape-en.pdf?rlkey=cta9nagob08kl9xkj2hlgix42&st=lwa4ane2&dl=0)

Claude Code CLI 치트시트에는 대화형 단축키와 slash command, `claude -p` 자동화, 저장된 세션과 백그라운드 세션, 권한 모드, 설정 파일, MCP와 플러그인 명령, 스킬과 서브에이전트 파일 위치를 정리했습니다.

- [Claude Code CLI 치트시트 - 세로형 PDF](https://www.dropbox.com/scl/fi/jctbemncxlixrn6ccecer/claude-code-cli-cheatsheet-en.pdf?rlkey=y5qbvkkmnodebv6qrupt4jw81&st=e7k12nz5&dl=0)
- [Claude Code CLI 치트시트 - 가로형 PDF](https://www.dropbox.com/scl/fi/c730zadvn5somvh4ohgh6/claude-code-cli-cheatsheet-landscape-en.pdf?rlkey=g2ovop23eszrc2jmx59cd2gpw&st=ruu7xbey&dl=0)

Neovim 치트시트에는 모드와 이동, 연산자와 텍스트 객체, 검색과 치환, 창·버퍼·탭, 레지스터와 매크로, Neovim 0.12의 기본 LSP 키, `init.lua` 설정과 `vim.pack`을 정리했습니다.

- [Neovim 치트시트 - 세로형 PDF](https://www.dropbox.com/scl/fi/hiw5094cahljkckelbk85/neovim-cheatsheet-en.pdf?rlkey=ermdpfeul9s6a0iujzn1jczww&st=y3ougr45&dl=0)
- [Neovim 치트시트 - 가로형 PDF](https://www.dropbox.com/scl/fi/snd09h0jok54pmoi3tqx8/neovim-cheatsheet-landscape-en.pdf?rlkey=3d7nnaaxd36qu51fc3s5hb2su&st=hrlolpr5&dl=0)

키 바인딩과 명령은 도구가 업데이트되면서 달라질 수 있습니다. 현재 설치된 버전과 다르게 동작한다면 Codex CLI와 Claude Code에서는 `?`와 `/` 메뉴, Neovim에서는 `:help`를 확인합니다.
