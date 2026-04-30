---
layout: post
title: 내가 자주 쓰는 Git 명령어 정리
subtitle: 매번 검색하지 않으려고 모아둔 실전 Git 커맨드
tags: [git, github, version-control, developer-tools]
cover-img: /assets/img/develop.jpeg
thumbnail-img: /assets/img/git.webp
share-img: /assets/img/develop.jpeg
author: 전경원
---

Git 명령어는 종류가 많지만, 실제로 자주 쓰는 것들은 생각보다 많지 않다. 문제는 그 자주 쓰는 명령어조차 막상 필요할 때마다 다시 검색하게 된다는 점이다.

이번 글에서는 내가 실제 작업에서 반복해서 사용하는 Git 명령어를 **설정**, **변경 확인**, **브랜치 작업**, **원격 저장소 작업**처럼 목적별로 정리해 보려고 한다.  완전한 Git 문서라기보다, 자주 꺼내 보는 개인용 레퍼런스에 가깝다.

## 1. 사용자 정보와 기본 설정

처음 Git 환경을 잡을 때 가장 먼저 넣어 두는 설정들이다.

```bash
git config --global core.editor "vim"
git config --global user.name <사용자_이름>
git config --global user.email <사용자_이메일주소>
git config --global pull.rebase true
```

특히 `pull.rebase true`는 개인적으로 자주 켜 두는 옵션이다. `git pull` 할 때 불필요한 merge commit 생성을 줄여서 히스토리를 조금 더 단순하게 유지할 수 있다.

## 2. 파일 스테이징

변경한 파일을 커밋 대상으로 올릴 때 사용한다.

```bash
git add .
git add -p
```

`git add .`은 빠르게 전체 변경 사항을 스테이징할 때 편하다.  반면 `git add -p`는 변경 내용을 덩어리 단위로 나눠서 선택적으로 올릴 수 있어서, 커밋을 더 작고 명확하게 나누고 싶을 때 유용하다.

## 3. 변경 내용 확인

커밋 전에 무엇이 들어가는지 확인하는 습관은 생각보다 중요하다.

```bash
git diff --cached
git diff --cached | pbcopy
git diff --cached | xclip -selection clipboard
```

`git diff --cached`는 현재 스테이징된 변경 사항만 보여 준다. 리뷰 요청 전에 diff를 복사해야 할 때는 macOS에서는 `pbcopy`, Linux에서는 `xclip` 조합이 편하다.

macOS 터미널(iTerm2)에서 SSH로 Linux 서버에 접속해 작업할 때도 비슷하게 쓸 수 있다. 서버의 `~/.bashrc`에 아래 함수를 추가해 두면, 원격 서버에서 실행한 결과를 로컬 macOS의 클립보드로 보낼 수 있다.

```bash
pbcopy() {
  base64 | tr -d '\n' | sed 's/.*/\x1b]52;c;&\x07/'
}
```

## 4. 커밋

스테이징된 변경 사항을 기록할 때 사용한다.

```bash
git commit
git commit --amend
```

`git commit`은 기본 커밋이고, `git commit --amend`는 직전 커밋을 수정할 때 쓴다. 커밋 메시지 오타를 고치거나, 빠뜨린 파일 하나를 같은 커밋에 묶고 싶을 때 자주 사용한다.

## 5. 브랜치 관리

브랜치를 확인하고 이동하거나 새로 만들 때 가장 많이 쓰는 명령어들이다.

```bash
git branch
git branch -r
git switch
git switch -c <새로운_브랜치>
```

`git branch`는 로컬 브랜치 목록, `git branch -r`는 원격 브랜치 목록을 보여 준다. 브랜치 이동 목적이라면 `checkout`보다 `switch`가 의도가 더 명확해서 개인적으로 더 자주 쓰고 있다.

## 6. 원격 저장소

원격 저장소 주소를 확인하거나 연결을 바꿀 때 사용한다.

```bash
git remote -v
git remote set-url origin <새로운_URL>
git remote add origin git@github.com:ruddyscent/hoverpilot.git
```

저장소를 처음 연결할 때는 `git remote add origin ...`을 쓰고, HTTPS에서 SSH로 바꾸거나 저장소 주소가 변경됐을 때는 `git remote set-url origin ...`이 필요하다.

## 7. 푸시

로컬 커밋을 원격 저장소에 올릴 때 사용한다.

```bash
git push
git push --set-upstream origin <새로운_브랜치>
git push -u origin main
git push origin master
git push origin main --tags
```

새 브랜치를 처음 푸시할 때는 `--set-upstream` 또는 `-u` 옵션을 자주 쓴다. 한 번 upstream이 연결되면 이후에는 보통 `git push`만으로 충분하다.

## 8. pull & rebase

원격 변경 사항을 가져오면서 내 작업을 다시 정리할 때 쓴다.

```bash
git pull --rebase origin master
git rebase --continue
```

충돌이 발생하면 수정한 뒤 `git rebase --continue`로 이어서 진행한다. merge보다 rebase를 선호한다면 자주 만나게 되는 조합이다.

## 9. cherry-pick

특정 커밋 하나만 다른 브랜치로 가져오고 싶을 때 유용하다.

```bash
git cherry-pick <커밋_해시>
```

예를 들어 핫픽스 커밋 하나만 별도 브랜치에 반영하거나, 실수로 다른 브랜치에서 작업한 변경을 옮겨야 할 때 많이 쓴다.

## 10. 브랜치 이름 변경

기본 브랜치 이름을 바꾸는 상황에서 주로 사용한다.

```bash
git branch -M main
```

예전에는 `master`를 기본 브랜치로 쓰는 저장소가 많았지만, 지금은 `main`으로 맞추는 경우가 많다.

## 11. 태그 관리

릴리즈 시점을 이름 붙여 남기고 싶을 때 사용하는 명령어다.

```bash
git tag v0.3-rflink-state-read
git push origin main --tags
git push origin v0.3-rflink-state-read
git tag -d rflink-state-read
git push origin --delete tag rflink-state-read
```

태그를 만들고 원격에 올려 두면 특정 배포 시점을 다시 찾기 쉬워진다. 잘못 만든 태그는 로컬과 원격에서 각각 삭제해 주면 된다.

## 12. 로그

히스토리를 빠르게 훑어볼 때 가장 자주 쓰는 기본 명령어다.

```bash
git log
git log --oneline
```

`git log`는 자세한 정보를 확인할 때, `git log --oneline`은 흐름을 빠르게 볼 때 편하다. 개인적으로는 대부분 `--oneline`부터 확인한 뒤 필요할 때만 자세한 로그를 다시 본다.
