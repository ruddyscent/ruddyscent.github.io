---
layout: post
title: Ghostty 초기 설정과 SSH에서 깨지는 zsh Backspace 고치기
subtitle: 키 바인딩이 아니라 원격 Ubuntu에 없던 terminfo가 문제였다
tags: [ghostty, ssh, zsh, ubuntu, troubleshooting]
cover-img: /assets/img/develop.jpeg
thumbnail-img: /assets/img/develop.jpeg
share-img: /assets/img/develop.jpeg
author: 전경원
description: macOS에서 Ghostty를 설정하고 Ubuntu에 SSH로 접속했을 때 zsh Backspace 화면이 깨지는 원인을 terminfo에서 찾아 해결한 과정.
---

iTerm2 대신 Ghostty를 쓰기 시작하면서 설정은 가능한 한 적게 가져가기로 했다. Ghostty는 기본값만으로도 충분히 쓸 만했고 영문 글꼴과 한글 글꼴을 나누는 정도로 시작했다.

## Ghostty는 기본값에서 시작한다

Ghostty 프로젝트는 macOS용 `.dmg`에 서명·공증해 공식 배포한다. Homebrew에는 커뮤니티가 관리하는 cask도 있다. Homebrew를 쓴다면 다음 명령으로 설치할 수 있다.

```bash
brew install --cask ghostty
```

처음부터 큰 Ghostty 설정 파일을 만들 필요는 없다. [공식 설정 문서](https://ghostty.org/docs/config/)도 기본값으로 먼저 사용해보기를 권한다. 현재 설정 파일 이름은 `config.ghostty`이며 macOS에서는 다음 두 경로를 읽는다.

```text
~/.config/ghostty/config.ghostty
~/Library/Application Support/com.mitchellh.ghostty/config.ghostty
```

Ghostty 1.2.3 이전에 사용하던 `config`라는 파일명도 호환된다. 새로 시작한다면 `config.ghostty`를 쓰는 편이 현재 문서와 맞다. macOS에서 `Cmd-Shift-,`를 누르면 설정을 다시 읽는다. 일부 옵션은 새 터미널에만 적용되거나 앱을 다시 시작해야 반영될 수 있다.

## 한글 글꼴부터 맞춘다

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

한글을 별도 글꼴로 강제하지 않고 macOS의 fallback에 맡기려면 `font-codepoint-map`을 빼고 결과를 비교해도 된다. 이 설정은 Mac에서 Ghostty가 글리프를 그리는 방법만 바꾼다. 뒤에서 다룰 원격 Ubuntu의 Backspace 문제와는 관계가 없다.

글꼴을 정리한 뒤 Ubuntu 서버에 접속하자 다른 문제가 나타났다. Bash에서는 Backspace가 멀쩡해 보이는데 zsh에서는 문자를 지워도 화면이 제대로 갱신되지 않았다. 처음에는 zsh 키 바인딩이나 TTY 설정을 의심했지만 원인은 Ghostty가 전달한 `TERM=xterm-ghostty`를 원격 Ubuntu가 이해하지 못하는 데 있었다.

## Bash는 괜찮고 zsh에서만 Backspace가 깨졌다

문제가 발생한 환경은 macOS의 Ghostty에서 Ubuntu 서버로 SSH 접속하고 원격 셸로 zsh를 쓰는 구성이었다. Bash에서는 Backspace가 정상처럼 보였지만 zsh에서는 입력 버퍼와 화면의 문자가 어긋났다.

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

## terminfo를 직접 설치해 해결한다

원인을 확인한 뒤에는 Mac의 Ghostty terminfo를 원격 Ubuntu로 직접 보냈다.

```bash
infocmp -x xterm-ghostty | ssh xavier -- tic -x -
```

명령의 앞부분은 Mac에 설치된 `xterm-ghostty` terminfo를 소스 형식으로 출력한다. SSH로 전달받은 Ubuntu의 `tic`는 이를 컴파일해 terminfo 데이터베이스에 설치한다. 시스템 경로에 쓸 권한이 없으면 보통 사용자별 `~/.terminfo`가 사용되므로 관리자 권한 없이 설치할 수 있다.

Ubuntu의 `tic`가 오래됐다면 설명 필드를 별칭으로 처리할 수 있다는 경고가 나올 수 있다. 경고만 보고 성공과 실패를 판단하지 말고 설치된 항목을 직접 조회한다.

```bash
infocmp xterm-ghostty
```

내용이 출력되면 새 SSH 세션을 열어 다시 확인한다.

```bash
echo "$TERM"
infocmp "$TERM" >/dev/null && echo "terminfo OK"
zsh -f
```

정상이라면 다음처럼 나온다.

```text
xterm-ghostty
terminfo OK
```

마지막으로 `zsh -f`에서 문자를 입력하고 Backspace로 지운다. 화면과 입력 버퍼가 함께 갱신되면 해결된 것이다.

## 사실 Ghostty 설정 한 줄이면 된다

여기까지는 원인을 확인하려고 키 입력부터 추적하고 terminfo도 직접 설치한 과정이다. 대화형 셸에서 평소처럼 `ssh`로 접속한다면 [Ghostty의 SSH integration](https://ghostty.org/docs/features/ssh/)으로 이 과정을 자동화할 수 있다. `config.ghostty`에 다음 한 줄을 추가한다.

```ini
shell-integration-features = ssh-env,ssh-terminfo
```

두 기능의 역할은 다르다.

| 설정값 | 역할 |
| --- | --- |
| `ssh-env` | `COLORTERM`, `TERM_PROGRAM`, `TERM_PROGRAM_VERSION` 전달 요청 |
| `ssh-terminfo` | 원격 호스트에 Ghostty terminfo 설치 |

Backspace 문제를 직접 해결하는 것은 `ssh-terminfo`다. Ghostty가 지원하는 대화형 셸에서 `ssh`를 shell function으로 감싸고 처음 연결할 때 원격의 `tic`로 terminfo를 설치한다. 이후에는 평소처럼 접속하면 된다.

```bash
ssh xavier
```

설치된 Ghostty가 `+ssh` action을 지원한다면 wrapper를 직접 호출할 수도 있다.

```bash
ghostty +ssh -- xavier
```

지원 여부와 옵션은 `ghostty +ssh --help`로 확인한다. `+ssh` action이 없는 버전에서도 `ssh-env`, `ssh-terminfo` shell integration은 사용할 수 있다.

`TERM`은 SSH가 원격 pseudo-terminal에 전달하지만 `ssh-env`가 다루는 추가 환경 변수는 원격 `sshd`의 허용을 받아야 한다. 서버 정책상 필요하고 변경 권한이 있다면 원격 `/etc/ssh/sshd_config`에 다음 항목을 검토할 수 있다.

```text
AcceptEnv COLORTERM TERM_PROGRAM TERM_PROGRAM_VERSION
```

이는 Ghostty 설정이 아니라 SSH 서버 설정이다. 관리형 서버에서 권한이 없거나 정책상 환경 변수를 허용하지 않는다면 억지로 바꿀 필요는 없다. Backspace 문제의 핵심은 `ssh-terminfo`가 해결한다.

다만 shell function은 Herdr, Git, `scp`, `rsync`, 스크립트처럼 `ssh` 실행 파일을 직접 호출하는 프로그램에 전달되지 않는다. 이런 경로에서도 같은 호스트에 접속한다면 앞의 수동 설치가 fallback이 된다.

한글 모양이 어색한 문제는 로컬의 `font-family`와 `font-codepoint-map`에서 해결했다. zsh의 Backspace 화면이 깨진 문제는 설정 한 줄로 원격 terminfo를 준비해 해결할 수 있었다. 둘을 나눠서 보면 Ghostty를 처음 설정할 때 어디부터 확인해야 할지 분명해진다.
