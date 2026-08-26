---
layout: post
title: Ghostty와 Herdr의 한글 입력, 원격 zsh Backspace 문제 해결
subtitle: 증상은 모두 터미널에 있었지만 원인은 서로 다른 층에 있었다
tags: [ghostty, herdr, macos, ime, ssh, zsh, ubuntu, troubleshooting]
cover-img: /assets/img/develop.jpeg
thumbnail-img: /assets/img/chatgpt.webp
share-img: /assets/img/develop.jpeg
author: 전경원
description: macOS의 Ghostty와 Herdr에서 한글 글꼴과 prefix 입력을 설정하고, Ubuntu 원격 zsh의 Backspace 문제를 terminfo로 해결한 과정.
---

iTerm2 대신 Ghostty를 쓰기 시작하면서 설정은 가능한 한 적게 가져가기로 했다. 기본값만으로도 충분히 쓸 만했지만 한글을 입력하고 Ubuntu의 Herdr에 원격으로 붙자 서로 다른 문제가 차례로 드러났다.

한글 모양은 글꼴 fallback의 문제였다. 한글 입력 상태에서 Herdr의 `Ctrl-b` prefix가 먹지 않는 문제는 macOS IME와 키 전달 방식에 걸려 있었다. SSH로 접속한 Ubuntu에서 zsh의 Backspace 화면이 깨진 원인은 원격에 없는 `xterm-ghostty` terminfo였다. 증상은 모두 터미널에서 나타났지만 손봐야 할 위치는 달랐다.

## Ghostty 설정은 최소한으로

Ghostty 프로젝트는 macOS용 `.dmg`에 서명·공증해 공식 배포한다. Homebrew에는 커뮤니티가 관리하는 cask도 있다. Homebrew를 쓴다면 다음 명령으로 설치할 수 있다.

```bash
brew install --cask ghostty
```

처음부터 큰 Ghostty 설정 파일을 만들 필요는 없다. [공식 설정 문서](https://ghostty.org/docs/config/)도 기본값으로 먼저 사용해보기를 권한다. 현재 설정 파일 이름은 `config.ghostty`이며 macOS에서는 다음 두 경로를 읽는다.

```text
~/.config/ghostty/config.ghostty
~/Library/Application Support/com.mitchellh.ghostty/config.ghostty
```

새 설정 파일에는 현재 문서에서 사용하는 `config.ghostty`라는 이름을 썼다. macOS에서 `Cmd-Shift-,`를 누르면 설정을 다시 읽는다. 일부 옵션은 새 터미널에만 적용되거나 앱을 다시 시작해야 반영될 수 있다.

## 한글 글꼴부터 예쁘게

Ghostty를 실행한 뒤 가장 먼저 눈에 들어온 것은 한글이었다. 영문은 또렷했지만 한글의 획과 자간은 기대와 달랐다. 기본 영문 글꼴은 유지하고 한글 코드 포인트만 D2Coding에 연결했다.

```ini
font-family = "Hack Nerd Font Mono"
font-codepoint-map = U+1100-U+11FF,U+3130-U+318F,U+A960-U+A97F,U+AC00-U+D7AF,U+D7B0-U+D7FF="D2Coding"
```

첫 줄은 기본 영문 글꼴을 지정한다. 두 번째 줄은 한글 자모와 완성형 한글 영역을 D2Coding에 연결한다.

설정에 적을 수 있는 정확한 font family 이름은 Ghostty가 인식한 목록에서 확인한다.

```bash
ghostty +list-fonts
```

D2Coding을 Homebrew로 설치한다면 다음 cask를 쓸 수 있다.

```bash
brew install --cask font-d2coding
```

글꼴을 새로 설치한 뒤 Ghostty가 찾지 못한다면 앱을 완전히 종료했다가 다시 실행한다. 다음처럼 한글과 영문이 섞인 문장을 출력하면 모양과 열 정렬을 함께 확인하기 쉽다.

```text
Ghostty에서 한글 글꼴 테스트 123 ABC
경로: ~/문서/프로젝트
```

한글을 별도 글꼴로 강제하지 않고 macOS의 fallback에 맡기려면 `font-codepoint-map`을 빼고 결과를 비교해도 된다. 이 설정은 Ghostty가 글리프를 그리는 방법만 바꾼다. 키 입력이나 원격 터미널의 동작과는 관계가 없다.

## 한글 입력 상태에서는 Herdr prefix가 이어지지 않았다

Herdr의 기본 prefix는 tmux와 같은 `Ctrl-b`다. 영문 입력 상태에서는 `Ctrl-b`를 누르고 손을 뗀 뒤 `q`를 누르면 바로 detach된다. macOS 입력 소스를 한글 두벌식으로 바꾸면 이 두 단계가 매끄럽게 이어지지 않았다.

- 물리적인 `Ctrl-b` 자체를 Herdr가 알아보지 못할 때가 있었다.
- prefix가 인식돼도 다음 물리 키 `q`가 명령이 아니라 `ㅂ`으로 조합됐다.
- `prefix+ㅂ`을 별도 바인딩으로 만들면 IME 조합을 확정하려고 Enter를 한 번 더 눌러야 했다.
- 로컬 Herdr에서 고친 뒤에도 `herdr --remote xavier`에서는 같은 증상이 남았다.

문제는 두 단계에 걸쳐 있었다. 먼저 Ghostty가 한글 입력 상태의 물리적인 `Ctrl-b`를 Herdr가 읽을 수 있는 키 시퀀스로 보내야 한다. Herdr가 prefix 모드에 들어간 다음에는 명령 키를 받는 동안 macOS 입력 소스를 잠시 ASCII로 바꿔야 한다.

## Ghostty에서 물리적인 Ctrl-b를 CSI-u로 보낸다

`~/.config/ghostty/config.ghostty`에 다음 키 바인딩을 추가했다.

```ini
# Keep Herdr's physical Ctrl-b prefix available while the Korean IME is active.
keybind = ctrl+key_b=csi:98;5u
```

Ghostty는 한글 입력 상태에서도 물리적인 `Ctrl-b`를 Kitty keyboard protocol의 CSI-u 시퀀스로 보낸다. Herdr는 이를 `ctrl+b` prefix로 인식한다. Ghostty 설정을 다시 읽은 뒤 한글 입력 상태에서 prefix 모드로 들어가는지 먼저 확인한다. `csi:` action의 형식은 [Ghostty keybind 설정 문서](https://ghostty.org/docs/config/reference#keybind)에서 확인할 수 있다.

## Herdr가 명령을 받는 동안 입력 소스를 바꾼다

prefix만 인식해서는 충분하지 않았다. 다음에 누르는 `q`, `h`, `j`, `k`, `l`도 IME가 한글로 조합하기 때문이다. 로컬 Mac의 `~/.config/herdr/config.toml`에 입력 소스 전환 옵션을 켰다.

```toml
onboarding = false

[keys]
prefix = "ctrl+b"

[experimental]
switch_ascii_input_source_in_prefix = true
```

`onboarding`과 `[keys]` 설정은 이미 초기 설정을 마쳤고 기본 prefix를 그대로 쓴다면 필수는 아니다. 핵심은 `switch_ascii_input_source_in_prefix`다. 기존 설정에 `[keys]`나 `[experimental]` 섹션이 있다면 같은 섹션을 하나 더 만들지 말고 항목만 추가한다.

[Herdr 설정 문서](https://herdr.dev/docs/configuration/#prefix-input-source-switching)에 따르면 이 옵션은 prefix 모드에 들어갈 때 현재 입력 소스를 기억하고 ASCII 입력 소스로 전환한다. 명령 처리가 끝나 터미널 입력으로 돌아오면 이전 한글 입력 소스를 복원한다. 따라서 `Ctrl-b`를 누른 뒤 물리적인 `q`를 눌러도 `ㅂ`이 조합되지 않고 detach 명령이 바로 실행된다.

설정을 저장한 뒤에는 실행 중인 Herdr 서버에 다시 불러올 수 있다.

```bash
herdr server reload-config
```

## 원격 Herdr에는 같은 옵션이 한 번 더 필요했다

로컬 세션은 여기까지로 해결됐지만 Herdr 0.8.2에서 `herdr --remote xavier`로 접속하면 `q`가 다시 `ㅂ`으로 조합됐다. remote attach는 기본적으로 로컬 키 바인딩을 사용하지만 `switch_ascii_input_source_in_prefix` 값까지 원격 서버에 전달하지는 않았다.

원격 Ubuntu의 `~/.config/herdr/config.toml`에도 같은 옵션을 추가했다.

```toml
[experimental]
switch_ascii_input_source_in_prefix = true
```

원격 서버에서 설정을 다시 읽는다.

```bash
herdr server reload-config
```

서버나 pane을 종료할 필요는 없었다. Ubuntu의 Herdr 서버가 prefix 모드 진입을 감지하면 foreground 클라이언트에 입력 소스 전환을 요청하고, 실제 전환은 Mac에서 실행 중인 Herdr 클라이언트가 맡는다. 바로 반영되지 않으면 원격 클라이언트만 다시 연결한다.

이 동작은 연결할 서버마다 같은 옵션을 적어야 한다는 아쉬움이 있다. 로컬 클라이언트의 선호를 원격 서버에 전달하지 않는 문제는 [herdrdev/herdr#3271](https://github.com/herdrdev/herdr/issues/3271)에 정리했다. 입력 소스 전환을 foreground 클라이언트에서 처리하도록 바꾼 [herdrdev/herdr#1016](https://github.com/herdrdev/herdr/pull/1016)과 이어지는 문제다.

## 로컬과 원격에서 prefix를 확인한다

Mac의 입력 소스를 한글 두벌식으로 바꾼 뒤 로컬 Herdr에서 `Ctrl-b`를 눌렀다가 손을 떼고 물리적인 `q`를 누른다. `ㅂ`이 나타나지 않고 즉시 detach되며 입력 소스가 다시 한글로 돌아오면 정상이다.

원격도 같은 순서로 확인한다.

```bash
herdr --remote xavier
```

원격 세션에서도 `ㅂ`이나 조합 중인 글자가 나타나지 않아야 한다. 명령을 실행하려고 Enter를 추가로 눌러야 한다면 IME가 여전히 `q`를 조합하고 있는 것이다.

`detach = ["prefix+q", "prefix+ㅂ"]`처럼 한글 글자를 별도 키 바인딩으로 추가하는 방법은 쓰지 않았다. `ㅂ`은 터미널에 곧바로 들어오는 물리 키가 아니라 IME가 조합 중인 문자열이라 자연스러운 prefix 동작을 만들 수 없다. Herdr를 쓰기 전에 매번 입력 소스를 영문으로 바꾸는 방법도 가능하지만 한글과 영문을 오갈 때마다 손이 멈춘다.

## SSH에서는 zsh Backspace가 깨졌다

한글 입력을 정리한 뒤 Ubuntu 서버에 접속하자 다른 문제가 나타났다. Bash에서는 Backspace가 멀쩡해 보이는데 zsh에서는 문자를 지워도 화면이 제대로 갱신되지 않았다. 처음에는 zsh 키 바인딩이나 TTY 설정을 의심했지만 원인은 Ghostty가 전달한 `TERM=xterm-ghostty`를 원격 Ubuntu가 이해하지 못하는 데 있었다.

사용자 설정을 읽지 않는 zsh에서도 증상이 같았다.

```bash
zsh -f
```

`.zshrc`와 플러그인을 배제했으니 키가 TTY에 어떻게 들어오고 zsh의 ZLE(Zsh Line Editor)가 어떤 명령에 연결했는지 차례로 확인했다.

## 키 입력부터 확인한다

먼저 TTY의 erase 설정을 본다.

```bash
stty -a
```

출력에는 `erase = ^?`가 있었고 실제 Backspace 입력도 `^?`였다. zsh의 키 바인딩도 확인했다.

```bash
bindkey '^?'
bindkey '^H'
bindkey -lL main
```

`^?`와 `^H`는 모두 `backward-delete-char`에 연결되어 있었고 `main` 키맵도 Emacs 모드였다. 여기까지 정상이면 Backspace 키가 잘못 전달됐거나 엉뚱한 편집 명령에 묶인 문제는 아니다.

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

터미널 프로그램은 `TERM` 환경 변수로 자신이 어떤 터미널 안에서 실행 중인지 확인한다. Ghostty는 `xterm-ghostty`를 사용하며 원격 프로그램은 같은 이름의 terminfo 항목을 읽어 커서 이동과 문자 삭제, 색상 같은 기능을 제어한다.

하지만 원격 Linux 배포판에 `xterm-ghostty`가 항상 설치되어 있는 것은 아니다. 로컬 Ghostty만 새로 설치하고 오래된 Ubuntu 서버에 접속하면 `TERM`은 전달됐는데 이를 설명할 terminfo가 없는 상태가 생긴다.

`TERM` 값은 원격에 도착했지만 해당 terminfo 항목은 없었다. Bash에서는 이 결손이 눈에 띄지 않았고 입력 줄을 적극적으로 다시 그리는 zsh ZLE에서 Backspace 화면 처리로 드러났다.

## TERM을 바꿔 원인을 확인한다

같은 SSH 세션에서 널리 설치된 terminfo 이름으로 잠시 바꾸고 zsh를 다시 실행했다.

```bash
export TERM=xterm-256color
zsh -f
```

Backspace가 정상으로 돌아왔다. zsh 설정과 키 바인딩은 그대로 두고 `TERM`만 바꿨으므로 원인은 `xterm-ghostty` terminfo 누락으로 좁혀졌다.

이 명령은 진단용이다. `.zshrc`에 `TERM=xterm-256color`를 영구 설정하면 실제 터미널의 capability를 일부러 낮춰 알리는 셈이다. Ghostty가 지원하는 기능을 숨기고 다른 프로그램에서 미묘한 호환성 문제를 만들 수 있다.

## SSH integration으로 terminfo 설치를 자동화한다

원인을 확인한 뒤에는 [Ghostty의 SSH integration](https://ghostty.org/docs/features/ssh/)으로 원격에 terminfo를 준비했다. `config.ghostty`에 다음 한 줄을 추가한다.

```ini
shell-integration-features = ssh-terminfo
```

Ghostty가 지원하는 대화형 셸에서 `ssh`를 shell function으로 감싸고 처음 연결할 때 원격의 `tic`로 terminfo를 설치한다. 설정을 다시 읽은 뒤 새 셸에서 평소처럼 접속한다.

```bash
ssh xavier
```

원격에서 terminfo와 zsh를 다시 확인한다.

```bash
infocmp "$TERM"
zsh -f
```

`infocmp`가 `xterm-ghostty` 정보를 출력하고 `zsh -f`에서 Backspace를 눌렀을 때 화면과 입력 버퍼가 함께 갱신되면 해결된 것이다.

다만 shell function은 Herdr, Git, `scp`, `rsync`, 스크립트처럼 `ssh` 실행 파일을 직접 호출하는 프로그램에 전달되지 않는다. 이 설정은 대화형 셸에서 `ssh`로 접속하는 경로에 적용된다.

## 확인한 환경

- macOS 26.5.2 (25F84), Apple silicon
- Ghostty 1.3.1 (build 15212)
- Herdr 0.8.2 stable, protocol 20
- macOS 기본 한글 두벌식 입력 소스
- 원격 호스트: Ubuntu (`xavier`)

한글 모양은 Ghostty의 글꼴 매핑에서 고쳤다. 한글 입력 중 Herdr의 prefix가 끊기는 문제는 Ghostty의 키 전달과 Herdr의 입력 소스 전환 설정으로 풀었다. 원격 zsh의 Backspace 문제는 Ubuntu에 Ghostty terminfo를 준비해 해결했다. 모두 Ghostty 화면 안에서 드러난 문제였지만 로컬 렌더링, macOS IME, 원격 터미널 정보라는 서로 다른 층에 원인이 있었다. 어느 층에서 문제가 생겼는지 나누고 나니 불필요하게 zsh 설정이나 키 바인딩을 건드릴 필요가 없었다.
