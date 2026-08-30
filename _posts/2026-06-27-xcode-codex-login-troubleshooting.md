---
layout: post
title: Xcode에서 Codex 로그인이 안 될 때 해결하기
subtitle: Xcode의 문제인 줄 알았지만, 원인은 Codex CLI 설정 파일에 있었다
tags: [xcode, codex, openai, oauth, troubleshooting, macos]
cover-img: /assets/img/develop.jpeg
thumbnail-img: /assets/img/editor.webp
share-img: /assets/img/chatgpt.webp
author: 전경원
---

Xcode의 Coding Intelligence에서 Codex를 쓰려고 **Sign in with ChatGPT**를 눌렀는데 인증이 끝나지 않았습니다. API Key는 잘 됐습니다. ChatGPT 계정 로그인만 계속 실패했습니다.

원인은 Xcode가 아니었습니다. **Codex CLI의 전역 설정 파일**이 문제였습니다.

## 증상

Xcode에서 Sign in with ChatGPT를 눌러도 인증이 완료되지 않습니다. API Key는 정상 동작합니다.

터미널에서 Codex CLI를 직접 실행하면 이런 에러가 나옵니다.

```bash
$ codex login

Error loading configuration:
~/.codex/config.toml

unknown variant `priority`,
expected `fast` or `flex`
```

또는

```bash
$ codex status

Error loading config.toml:
unknown variant `priority`,
expected `fast` or `flex`
```

## 원인

`~/.codex/config.toml`에 이런 항목이 들어 있는 것이 문제입니다.

```toml
service_tier = "priority"
```

최신 Codex CLI는 `priority`를 지원하지 않습니다. 유효한 값은 `fast`와 `flex`뿐입니다.

## 해결

설정 파일을 열고

```bash
open ~/.codex/config.toml
```

`service_tier = "priority"`를 다음 중 하나로 바꿉니다.

```toml
service_tier = "fast"
```

또는

```toml
service_tier = "flex"
```

해당 항목이 필요 없다면 그 줄 자체를 지워도 됩니다.

## 확인

```bash
codex login
```

에러 없이 브라우저에서 ChatGPT 인증 화면이 열립니다. 로그인을 완료하면 Xcode에서도 Codex를 쓸 수 있습니다.

## 한 줄 정리

Xcode에서 ChatGPT 계정 로그인만 안 된다면 Xcode 설정보다 `~/.codex/config.toml`부터 확인합니다. `service_tier = "priority"`가 남아 있으면 `fast` 또는 `flex`로 바꾸면 해결됩니다.
