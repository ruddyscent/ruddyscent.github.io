---
layout: post
title: Ghostty 한글 입력, Herdr prefix, 원격 zsh 문제 해결
subtitle: 한글 입력부터 원격 zsh까지, Ghostty로 옮긴 뒤 막힌 것들
tags: [ghostty, herdr, macos, ime, ssh, zsh, ubuntu, troubleshooting]
cover-img: /assets/img/develop.jpeg
thumbnail-img: /assets/img/terminal.webp
share-img: /assets/img/develop.jpeg
author: 전경원
description: macOS의 Ghostty에서 한글 글꼴과 Ctrl 제어문자를 설정하고, Herdr prefix와 Ubuntu 원격 zsh Backspace 문제를 해결한 과정.
---

iTerm2 대신 Ghostty를 쓰기 시작하면서 설정은 가능한 한 적게 가져가기로 했습니다. 기본값만으로도 충분히 쓸 만했습니다. 다만 한글 입력 상태에서 셸과 Herdr를 쓰고 Ubuntu에 원격으로 붙자 서로 다른 문제가 차례로 드러났습니다.

한글 모양은 Ghostty의 글꼴 설정으로 고치고 한글 입력 중 셸의 제어문자는 물리 키 바인딩으로 보냅니다. Herdr에서는 한글 입력 중 prefix 모드에 들어갈 때 입력 소스를 잠시 바꾸고 원격 zsh의 Backspace 문제는 SSH integration으로 `xterm-ghostty` terminfo를 설치해 해결합니다.

## Ghostty 설정은 최소한으로

Ghostty는 Homebrew로 설치합니다.

```bash
brew install --cask ghostty
```

처음부터 큰 Ghostty 설정 파일을 만들 필요는 없습니다. [공식 설정 문서](https://ghostty.org/docs/config/)도 기본값으로 먼저 사용해보기를 권합니다. 현재 설정 파일 이름은 `config.ghostty`이며 macOS에서는 다음 두 경로를 읽습니다.

```text
~/.config/ghostty/config.ghostty
~/Library/Application Support/com.mitchellh.ghostty/config.ghostty
```

새 설정 파일은 `config.ghostty`라는 이름으로 만듭니다. macOS에서 `Cmd-Shift-,`를 누르면 설정을 다시 읽습니다. 일부 옵션은 새 터미널에만 적용되거나 앱을 다시 시작해야 반영될 수 있습니다.

## 한글 글꼴부터 예쁘게

Ghostty를 실행한 뒤 가장 먼저 눈에 들어온 것은 한글이었다. 영문은 또렷했지만 한글의 획과 자간은 기대와 달랐다. 기본 영문 글꼴은 유지하고 한글 코드 포인트만 D2Coding에 연결합니다.

```ini
font-family = "Hack Nerd Font Mono"
font-codepoint-map = U+1100-U+11FF,U+3130-U+318F,U+A960-U+A97F,U+AC00-U+D7AF,U+D7B0-U+D7FF="D2Coding"
```

첫 줄은 기본 영문 글꼴을 지정합니다. 두 번째 줄은 한글 자모와 완성형 한글 영역을 D2Coding에 연결합니다.

설정에 적을 수 있는 정확한 font family 이름은 Ghostty가 인식한 목록에서 확인합니다.

```bash
ghostty +list-fonts
```

D2Coding을 Homebrew로 설치한다면 다음 cask를 쓸 수 있습니다.

```bash
brew install --cask font-d2coding
```

글꼴을 새로 설치한 뒤 Ghostty가 찾지 못한다면 앱을 완전히 종료했다가 다시 실행합니다. 다음처럼 한글과 영문이 섞인 문장을 출력하면 모양과 열 정렬을 함께 확인하기 쉽습니다.

```text
Ghostty에서 한글 글꼴 테스트 123 ABC
경로: ~/문서/프로젝트
```

한글을 별도 글꼴로 강제하지 않고 macOS의 fallback에 맡기려면 `font-codepoint-map`을 빼고 결과를 비교해도 됩니다. 이 설정은 Ghostty가 글리프를 그리는 방법만 바꾼다. 키 입력이나 원격 터미널의 동작과는 관계가 없습니다.

## 한글 입력 중에도 Ctrl 제어문자를 보낸다

macOS에서 한글 입력기가 활성화된 동안에는 `Ctrl-a`, `Ctrl-c` 같은 조합이 셸에서 기대하는 ASCII 제어문자로 전달되지 않을 수 있습니다. Ghostty에서 물리 키 기준 바인딩을 명시해 두면 입력 소스가 한글이어도 같은 제어문자를 보낼 수 있습니다.

Ghostty의 `key_*` 표기는 입력 소스가 만든 글자가 아니라 물리 키를 기준으로 매칭합니다. `text:\xNN`은 해당 ASCII 제어 바이트를 터미널에 직접 보냅니다.

```ini
# Control-key passthrough for the macOS Korean IME.
keybind = ctrl+key_a=text:\x01
keybind = ctrl+key_b=text:\x02
keybind = ctrl+key_c=text:\x03
keybind = ctrl+key_d=text:\x04
keybind = ctrl+key_e=text:\x05
keybind = ctrl+key_f=text:\x06
keybind = ctrl+key_k=text:\x0b
keybind = ctrl+key_l=text:\x0c
keybind = ctrl+key_n=text:\x0e
keybind = ctrl+key_p=text:\x10
keybind = ctrl+key_r=text:\x12
keybind = ctrl+key_u=text:\x15
keybind = ctrl+key_w=text:\x17
keybind = ctrl+key_z=text:\x1a
```

zsh의 기본 Emacs 키맵에서 자주 쓰는 동작은 다음과 같다.

| 키 | 제어 바이트 | 일반적인 동작 |
| --- | --- | --- |
| `Ctrl-a` | `0x01` | 줄의 처음으로 이동 |
| `Ctrl-b` | `0x02` | 커서를 한 글자 뒤로 이동 |
| `Ctrl-c` | `0x03` | 실행 중인 프로세스 중단 |
| `Ctrl-d` | `0x04` | 다음 문자 삭제 또는 EOF |
| `Ctrl-e` | `0x05` | 줄의 끝으로 이동 |
| `Ctrl-f` | `0x06` | 커서를 한 글자 앞으로 이동 |
| `Ctrl-k` | `0x0b` | 커서부터 줄 끝까지 삭제 |
| `Ctrl-l` | `0x0c` | 화면 지우기 |
| `Ctrl-n` | `0x0e` | 다음 히스토리 |
| `Ctrl-p` | `0x10` | 이전 히스토리 |
| `Ctrl-r` | `0x12` | 히스토리 역방향 검색 |
| `Ctrl-u` | `0x15` | 커서부터 줄 처음까지 삭제 |
| `Ctrl-w` | `0x17` | 앞 단어 삭제 |
| `Ctrl-z` | `0x1a` | 프로세스 일시 중단 |

각 프로그램은 같은 제어문자를 다른 명령에 연결할 수 있습니다. 이 표는 zsh의 일반적인 기본 동작이며 모든 TUI 프로그램에서 똑같이 동작한다는 뜻은 아닙니다. 설정을 다시 읽은 뒤 새 터미널에서도 확인합니다.

## 한글 입력 상태에서는 Herdr prefix가 이어지지 않았다

Herdr의 기본 prefix는 tmux와 같은 `Ctrl-b`다. 영문 입력 상태에서는 `Ctrl-b`를 누르고 손을 뗀 뒤 `q`를 누르면 바로 detach됩니다. macOS 입력 소스를 한글 두벌식으로 바꾸면 이 두 단계가 매끄럽게 이어지지 않았다.

- 물리적인 `Ctrl-b` 자체를 Herdr가 알아보지 못할 때가 있었다.
- prefix가 인식돼도 다음 물리 키 `q`가 명령이 아니라 `ㅂ`으로 조합됐다.
- `prefix+ㅂ`을 별도 바인딩으로 만들면 IME 조합을 확정하려고 Enter를 한 번 더 눌러야 했다.
- 로컬 Herdr에서 고친 뒤에도 `herdr --remote xavier`에서는 같은 증상이 남았다.

문제는 두 단계입니다. 먼저 앞에서 추가한 `ctrl+key_b=text:\x02` 바인딩으로 한글 입력 상태의 물리적인 `Ctrl-b`를 Herdr에 보냅니다. Herdr가 prefix 모드에 들어간 다음에는 명령 키를 받는 동안 macOS 입력 소스를 잠시 ASCII로 바꿔야 합니다.

## Herdr가 명령을 받는 동안 입력 소스를 바꾼다

prefix만 인식해서는 충분하지 않습니다. 다음에 누르는 `q`, `h`, `j`, `k`, `l`도 IME가 한글로 조합하기 때문입니다. 로컬 Mac의 `~/.config/herdr/config.toml`에 입력 소스 전환 옵션을 켭니다.

```toml
onboarding = false

[keys]
prefix = "ctrl+b"

[experimental]
switch_ascii_input_source_in_prefix = true
```

`onboarding`과 `[keys]` 설정은 이미 초기 설정을 마쳤고 기본 prefix를 그대로 쓴다면 필수는 아닙니다. 핵심은 `switch_ascii_input_source_in_prefix`입니다.

[Herdr 설정 문서](https://herdr.dev/docs/configuration/#prefix-input-source-switching)에 따르면 이 옵션은 prefix 모드에 들어갈 때 현재 입력 소스를 기억하고 ASCII 입력 소스로 전환합니다. 명령 처리가 끝나 터미널 입력으로 돌아오면 이전 한글 입력 소스를 복원합니다. 따라서 `Ctrl-b`를 누른 뒤 물리적인 `q`를 눌러도 `ㅂ`이 조합되지 않고 detach 명령이 바로 실행됩니다.

설정을 저장한 뒤에는 실행 중인 Herdr 서버에 다시 불러올 수 있습니다.

```bash
herdr server reload-config
```

## 원격 Herdr에는 같은 옵션이 한 번 더 필요하다

로컬 세션은 여기까지로 해결되지만 Herdr 0.8.2에서 `herdr --remote xavier`로 접속하면 `q`가 다시 `ㅂ`으로 조합됩니다. remote attach는 기본적으로 로컬 키 바인딩을 사용하지만 `switch_ascii_input_source_in_prefix` 값까지 원격 서버에 전달하지는 않습니다.

원격 Ubuntu의 `~/.config/herdr/config.toml`에도 같은 옵션을 추가합니다.

```toml
[experimental]
switch_ascii_input_source_in_prefix = true
```

원격 서버에서 설정을 다시 읽습니다.

```bash
herdr server reload-config
```

서버나 pane을 종료할 필요는 없습니다. Ubuntu의 Herdr 서버가 prefix 모드 진입을 감지하면 foreground 클라이언트에 입력 소스 전환을 요청하고 실제 전환은 Mac에서 실행 중인 Herdr 클라이언트가 맡습니다. 바로 반영되지 않으면 원격 클라이언트만 다시 연결합니다.

이 동작은 연결할 서버마다 같은 옵션을 적어야 한다는 아쉬움이 있습니다. 로컬 클라이언트의 선호를 원격 서버에 전달하지 않는 문제는 [herdrdev/herdr#3271](https://github.com/herdrdev/herdr/issues/3271)에 정리했습니다. 입력 소스 전환을 foreground 클라이언트에서 처리하도록 바꾼 [herdrdev/herdr#1016](https://github.com/herdrdev/herdr/pull/1016)과 이어지는 문제입니다.

## 로컬과 원격에서 prefix를 확인한다

Mac의 입력 소스를 한글 두벌식으로 바꾼 뒤 로컬 Herdr에서 `Ctrl-b`를 눌렀다가 손을 떼고 물리적인 `q`를 누릅니다. `ㅂ`이 나타나지 않고 즉시 detach되며 입력 소스가 다시 한글로 돌아오면 정상입니다.

원격도 같은 순서로 확인합니다.

```bash
herdr --remote xavier
```

원격 세션에서도 `ㅂ`이나 조합 중인 글자가 나타나지 않아야 합니다. 명령을 실행하려고 Enter를 추가로 눌러야 한다면 IME가 여전히 `q`를 조합하고 있는 것입니다.

`detach = ["prefix+q", "prefix+ㅂ"]`처럼 한글 글자를 별도 키 바인딩으로 추가하는 방법은 쓰지 않습니다. `ㅂ`은 터미널에 곧바로 들어오는 물리 키가 아니라 IME가 조합 중인 문자열이라 자연스러운 prefix 동작을 만들 수 없습니다.

## SSH에서는 zsh Backspace가 깨졌다

한글 입력을 정리한 뒤 Ubuntu 서버에 접속하자 다른 문제가 나타났다. Bash에서는 Backspace가 멀쩡해 보이는데 zsh에서는 문자를 지워도 화면이 제대로 갱신되지 않았다. 처음에는 zsh 키 바인딩이나 TTY 설정을 의심했지만 원인은 Ghostty가 전달한 `TERM=xterm-ghostty`를 원격 Ubuntu가 이해하지 못하는 데 있었다.

사용자 설정을 읽지 않는 zsh에서도 증상이 같았다.

```bash
zsh -f
```

`.zshrc`와 플러그인을 배제했으니 키가 TTY에 어떻게 들어오고 zsh의 ZLE(Zsh Line Editor)가 어떤 명령에 연결했는지 차례로 확인했다.

## 키 입력부터 확인한다

먼저 TTY의 erase 설정을 봅니다.

```bash
stty -a
```

출력에는 `erase = ^?`가 있었고 실제 Backspace 입력도 `^?`였다. zsh의 키 바인딩도 확인했다.

```bash
bindkey '^?'
bindkey '^H'
bindkey -lL main
```

`^?`와 `^H`는 모두 `backward-delete-char`에 연결되어 있었고 `main` 키맵도 Emacs 모드였다. 여기까지 정상이면 Backspace 키가 잘못 전달됐거나 엉뚱한 편집 명령에 묶인 문제는 아닙니다.

화면 갱신에 필요한 터미널 기능 정보로 범위를 좁혔다.

```bash
echo "$TERM"
infocmp "$TERM"
```

문제가 있던 Ubuntu에서는 다음과 같이 나왔다.

```text
xterm-ghostty
infocmp: couldn't open terminfo file /usr/share/terminfo/x/xterm-ghostty.
```

터미널 프로그램은 `TERM` 환경 변수로 자신이 어떤 터미널 안에서 실행 중인지 확인합니다. Ghostty는 `xterm-ghostty`를 사용하며 원격 프로그램은 같은 이름의 terminfo 항목을 읽어 커서 이동과 문자 삭제, 색상 같은 기능을 제어합니다.

하지만 원격 Linux 배포판에 `xterm-ghostty`가 항상 설치되어 있는 것은 아닙니다. 로컬 Ghostty만 새로 설치하고 오래된 Ubuntu 서버에 접속하면 `TERM`은 전달됐는데 이를 설명할 terminfo가 없는 상태가 생긴다.

`TERM` 값은 원격에 도착했지만 해당 terminfo 항목은 없었다. Bash에서는 이 결손이 눈에 띄지 않았고 입력 줄을 적극적으로 다시 그리는 zsh ZLE에서 Backspace 화면 처리로 드러났다.

## TERM을 바꿔 원인을 확인한다

같은 SSH 세션에서 널리 설치된 terminfo 이름으로 잠시 바꾸고 zsh를 다시 실행했다.

```bash
export TERM=xterm-256color
zsh -f
```

Backspace가 정상으로 돌아왔다. zsh 설정과 키 바인딩은 그대로 두고 `TERM`만 바꿨으므로 원인은 `xterm-ghostty` terminfo 누락으로 좁혀졌다.

이 명령은 진단용입니다. `.zshrc`에 `TERM=xterm-256color`를 영구 설정하면 실제 터미널의 capability를 일부러 낮춰 알리는 셈입니다. Ghostty가 지원하는 기능을 숨기고 다른 프로그램에서 미묘한 호환성 문제를 만들 수 있습니다.

## SSH integration으로 terminfo 설치를 자동화한다

원인을 확인한 뒤에는 [Ghostty의 SSH integration](https://ghostty.org/docs/features/ssh/)으로 원격에 terminfo를 준비합니다. `config.ghostty`에 다음 한 줄을 추가합니다.

```ini
shell-integration-features = ssh-terminfo
```

Ghostty가 지원하는 대화형 셸에서 `ssh`를 shell function으로 감싸고 처음 연결할 때 원격의 `tic`로 terminfo를 설치합니다. 설정을 다시 읽은 뒤 새 셸에서 평소처럼 접속합니다.

```bash
ssh xavier
```

원격에서 terminfo와 zsh를 다시 확인합니다.

```bash
infocmp "$TERM"
zsh -f
```

`infocmp`가 `xterm-ghostty` 정보를 출력하고 `zsh -f`에서 Backspace를 눌렀을 때 화면과 입력 버퍼가 함께 갱신되면 해결된 것입니다.

다만 shell function은 Herdr, Git, `scp`, `rsync`, 스크립트처럼 `ssh` 실행 파일을 직접 호출하는 프로그램에 전달되지 않는다. 이 설정은 대화형 셸에서 `ssh`로 접속하는 경로에 적용됩니다.

## 확인한 환경

- macOS 26.5.2 (25F84), Apple silicon
- Ghostty 1.3.1 (build 15212)
- Herdr 0.8.2 stable, protocol 20
- macOS 기본 한글 두벌식 입력 소스
- 원격 호스트: Ubuntu (`xavier`)
