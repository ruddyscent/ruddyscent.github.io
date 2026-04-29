---
layout: post
title: 노트북을 Git에 깔끔하게 올리기
subtitle: 노트북 diff에서 출력·메타데이터를 자동으로 걷어내기
tags: [nbdev, jupyter, git, git-hooks, uv, pre-commit, python]
cover-img: /assets/img/develop.jpeg
thumbnail-img: /assets/img/git.png
share-img: /assets/img/develop.jpeg
author: 전경원
---

Jupyter 노트북은 분석과 실험에는 더할 나위 없이 좋은 도구지만, **Git과는 궁합이 좋지 않다**. 셀 출력, 실행 카운트, 셀 ID, 커널 메타데이터처럼 코드와 무관한 정보가 매 실행마다 새로 끼어들어 diff를 지저분하게 만들고, 협업 시에는 머지 충돌을 손으로 풀기가 거의 불가능하다.

이런 문제를 깔끔하게 풀어 주는 도구가 [`nbdev`](https://nbdev.fast.ai/)다. nbdev는 노트북을 저장하거나 머지할 때 메타데이터·출력을 자동으로 정리해 주는 hook과 노트북 전용 머지 드라이버를 함께 제공한다. 공식 문서의 [Git-Friendly Jupyter](https://nbdev.fast.ai/tutorials/git_friendly_jupyter.html) 튜토리얼은 `pip install nbdev`로 시작하지만, macOS의 Homebrew Python처럼 외부에서 관리하는 환경에서는 다음과 같은 에러부터 만나게 된다.

```text
error: externally-managed-environment

× This environment is externally managed
╰─> To install Python packages system-wide, try brew install
    xyz, where xyz is the package you are trying to install.
```

아래에서는 `pip` 대신 [`uv`](https://docs.astral.sh/uv/)로 PEP 668 제약을 우회하지 않고 nbdev를 설치한 뒤, 기존 git 저장소에 hook을 붙이고 pre-commit으로 안전망까지 깔아보자.

## 1. 왜 `pip install nbdev`가 막히는가

macOS에서 `brew install python`으로 설치한 Python이나 일부 리눅스 배포판의 시스템 Python은 [PEP 668](https://peps.python.org/pep-0668/)에 따라 **시스템이 직접 관리하는 환경**에 해당한다. 시스템 패키지 매니저가 관리하는 Python에 사용자가 임의로 패키지를 끼워 넣으면 시스템이 망가질 수 있어서, `pip`가 이를 사전에 차단한다.

해결 방법은 크게 세 가지다.

- `--break-system-packages` 플래그를 강제로 붙인다 (권장하지 않음).
- `python -m venv`로 가상환경을 만들어 그 안에서 설치한다.
- `pipx`나 `uv tool`로 **격리된 환경에 CLI 도구로 설치**한다.

`nbdev`는 보통 `nbdev-install-hooks`, `nbdev-clean` 같은 커맨드라인 명령으로 다룬다. 라이브러리로 import해서 쓰는 경우보다 **CLI 도구**로 호출하는 경우가 훨씬 많아서, `uv tool`로 깔아 두는 방식이 가장 잘 맞는다.

## 2. uv 설치

macOS에서는 `Homebrew`로 한 줄로 설치할 수 있다.

```bash
brew install uv
```

설치 후 버전을 확인한다.

```bash
uv --version
```

## 3. nbdev를 uv로 설치

`nbdev`를 시스템 전역에서 CLI 도구처럼 사용할 수 있게 `uv tool`로 설치한다.

```bash
uv tool install nbdev
```

`uv`가 nbdev 전용 가상환경을 따로 만들고 그 안에 nbdev를 설치해 준다. 동시에 `nbdev-install-hooks`·`nbdev-clean` 같은 실행 파일을 `~/.local/bin`(또는 `uv`가 관리하는 shim 경로)에 연결해 어디서든 호출할 수 있게 해 준다. 시스템 Python은 전혀 건드리지 않는다.

설치가 끝나면 PATH에 nbdev 명령어가 잡혀 있는지 확인한다.

```bash
which nbdev-install-hooks
nbdev-install-hooks --help
```

명령어가 보이지 않는다면 PATH를 한 번 갱신해 준다.

```bash
uv tool update-shell
```

> nbdev를 **특정 프로젝트에서만** 쓰고 싶다면 `uv tool install` 대신 프로젝트 디렉터리에서 `uv add --dev nbdev`로 개발 의존성에 추가해도 된다. 이때는 nbdev 명령을 `uv run nbdev-install-hooks`처럼 `uv run`을 앞에 붙여 실행하면 된다.

## 4. 기존 git 저장소에 hook 설치

이제 nbdev의 hook을 적용하고 싶은 git 저장소로 이동해서 다음 명령을 실행한다.

```bash
cd /path/to/your-repo
nbdev-install-hooks
```

이 한 줄이 아래 세 가지를 자동으로 설치해 준다.

- **Jupyter pre-save hook** — Jupyter Notebook/Lab에서 노트북을 **저장할 때마다** 실행 카운트, 셀 ID, 불필요한 메타데이터, 출력을 자동으로 정리한다.
- **Git merge driver** — 노트북 머지 충돌을 노트북 구조에 맞게 해결해 주고, 그래도 남는 충돌은 Jupyter에서 그대로 열어 볼 수 있는 형태로 남긴다. 이때 `.gitattributes`에 `*.ipynb`용 merge 속성도 함께 적어 둔다.
- **Git post-merge hook** — 머지·풀·리베이스 직후에 `nbdev-trust`를 실행해서 노트북을 다시 신뢰 상태로 만든다. 머지 후 Jupyter에서 노트북을 열었을 때 "untrusted" 경고가 뜨는 문제를 막아 준다.

세 가지가 잘 자리 잡았는지는 아래로 확인한다.

```bash
# .gitattributes에 *.ipynb용 merge 규칙이 들어갔는지
cat .gitattributes

# post-merge git 훅이 만들어졌는지
ls .git/hooks/post-merge

# Jupyter pre-save hook이 활성화됐는지 (jupyter_server_config.py 등)
ls ~/.jupyter
```

## 5. 이미 더러워진 노트북 정리하기

훅을 처음 설치하기 전에 커밋해 둔 노트북에는 여전히 출력과 메타데이터가 남아 있을 수 있다. 저장소 전체의 노트북을 한 번에 정리하려면 다음 명령을 사용한다.

```bash
nbdev-clean --fname . --clear_all
```

- `--fname .` 은 현재 디렉터리 아래 모든 `*.ipynb`를 대상으로 한다. 특정 파일만 정리하려면 파일 경로를 직접 적으면 된다.
- `--clear_all` 은 셀 출력까지 모두 지운다. 출력은 살려 두고 메타데이터만 정리하고 싶다면 이 옵션은 빼면 된다.

정리 후에는 평소처럼 커밋하면 된다.

```bash
git add .
git commit -m "Clean notebooks with nbdev hooks"
```

이 시점부터는 다른 사람이 같은 저장소를 클론해 작업해도 노트북 diff가 코드 변경 위주로 깔끔하게 남는다.

## 6. VSCode·PyCharm 사용자를 위한 안전망: pre-commit

`nbdev-install-hooks`가 깔아 두는 Jupyter pre-save 훅은 **Jupyter Notebook/Lab 서버를 거쳐 저장한 경우에만** 동작한다. VSCode의 Jupyter 확장, PyCharm, papermill처럼 노트북 파일을 직접 수정·저장하는 도구를 쓰면 이 훅을 우회하므로 출력과 메타데이터가 그대로 커밋되어 버린다.

이 빈틈은 [`pre-commit`](https://pre-commit.com/)으로 메운다. nbdev는 pre-commit용 hook을 함께 배포하므로, 한두 줄 설정만 추가하면 **`git commit` 시점**에 `nbdev-clean`이 자동으로 돌면서 정리를 한 번 더 해 준다. 어느 에디터에서 저장했든 git에는 깨끗한 상태만 들어가도록 잡아 주는 안전망인 셈이다.

### 6.1. pre-commit을 uv로 설치

`pre-commit`도 CLI 도구이므로 `uv tool`로 설치하면 깔끔하다.

```bash
uv tool install pre-commit
```

특정 프로젝트에서만 쓰고 싶다면 `uv add --dev pre-commit`으로 개발 의존성에 추가해도 된다. 이때는 nbdev와 마찬가지로 `uv run pre-commit ...`처럼 `uv run`을 앞에 붙여 실행한다.

### 6.2. `.pre-commit-config.yaml` 작성

저장소 루트에 다음과 같은 `.pre-commit-config.yaml` 파일을 만든다.

```yaml
repos:
  - repo: https://github.com/AnswerDotAI/nbdev
    rev: 3.0.15  # GitHub release에서 최신 태그 확인 후 채우기
    hooks:
      - id: nbdev-clean
      - id: nbdev-export
```

- **`nbdev-clean`** — 커밋 직전에 노트북에서 출력·실행 카운트·불필요한 메타데이터를 정리한다. 이 hook 하나만 있어도 git diff에 끼는 노이즈는 거의 사라진다.
- **`nbdev-export`** — nbdev로 패키지를 빌드하는 프로젝트에서, 노트북에 정의한 `#| export` 셀을 Python 모듈로 다시 내보낸다. 노트북만 쓰는 분석 저장소라면 이 줄은 빼도 된다.

> `rev`에는 실제로 사용하는 nbdev 버전을 적는다. 현재 설치된 버전은 `uv tool list`로, 최신 릴리스 태그는 [GitHub releases](https://github.com/AnswerDotAI/nbdev/releases)에서 확인하면 된다. nbdev 2.x 문서나 오래된 예제에서는 hook id가 `nbdev_clean`처럼 밑줄 형태로 나오기도 하지만, nbdev 3.x pre-commit hook id는 `nbdev-clean`처럼 하이픈 형태다.

### 6.3. hook 활성화

`.pre-commit-config.yaml`을 만든 뒤 한 번만 실행하면 git의 pre-commit 훅으로 자리 잡는다.

```bash
pre-commit install
```

(프로젝트 의존성으로 설치한 경우에는 `uv run pre-commit install`)

이제 `git commit`을 실행하면 pre-commit이 스테이징된 파일에 hook을 차례로 돌린다. **hook이 파일을 수정하면 pre-commit이 커밋을 중단하고, 수정한 변경분을 unstaged 상태로 둔다.** 그러니 노트북에 출력이 남아 있었다면 첫 커밋은 일단 멈추고, `git add`로 다시 스테이징한 뒤 커밋하면 정리된 상태로 들어간다.

기존 노트북에 hook을 한 번 일괄 적용해 보고 싶다면 다음을 쓴다.

```bash
pre-commit run --all-files
```

### 6.4. `.pre-commit-config.yaml`을 커밋할지 말지

이 파일은 협업 전략에 따라 처리 방식이 갈린다.

- **모든 협업자가 pre-commit을 쓰기로 합의한 경우**: `.pre-commit-config.yaml`을 저장소에 함께 커밋한다. 각자 클론한 뒤 한 번씩 `pre-commit install`만 실행하면 동일한 hook이 적용된다.
- **개인적으로만 쓰고 싶은 경우**: `.pre-commit-config.yaml`을 `.gitignore`에 추가한다.

## 7. nbdev hook과 pre-commit의 역할 정리

도구가 둘이 되었으니 각자 어디까지 책임지는지 한 번 짚어 두자.

| 책임 영역 | `nbdev-install-hooks` | `pre-commit` + `nbdev-clean` |
| --- | --- | --- |
| Jupyter Notebook/Lab에서 저장 시 자동 정리 | ✅ pre-save hook | ➖ |
| VSCode·PyCharm 등 다른 에디터에서 저장한 노트북 정리 | ❌ | ✅ commit 시점에 정리 |
| 머지·풀·리베이스 충돌 해결 (merge driver) | ✅ | ❌ |
| 머지 후 노트북 자동 trust (`nbdev-trust`) | ✅ post-merge hook | ❌ |
| 협업자에게 자동 적용 | ❌ (각자 한 번씩 실행 필요) | △ (`.pre-commit-config.yaml`을 커밋하고 `pre-commit install` 실행) |

요약하면 **`nbdev-install-hooks`는 머지 쪽을, pre-commit은 커밋 쪽을 보강**한다. 둘을 함께 켜 두면 어떤 경로로 노트북을 수정하든 git에는 정리된 상태만 남는다.

## 8. 협업자가 같은 저장소를 클론했을 때

`nbdev-install-hooks`가 만들어 두는 merge driver 정의와 post-merge 훅, Jupyter pre-save 훅 설정은 모두 **각자의 로컬 환경에만 들어간다**. 다른 협업자가 저장소를 새로 클론하면 `.gitattributes`는 따라오지만 driver 정의와 훅 스크립트는 비어 있어, 자동 정리도 머지 충돌 해소도 동작하지 않는다.

`pre-commit`도 같은 맥락이다. `.pre-commit-config.yaml`은 저장소를 따라오지만, hook 자체는 각자 한 번씩 `pre-commit install`을 돌려야 git에 자리 잡는다.

그래서 README에 다음 한 줄을 적어 두면 깔끔하다.

```bash
# Clone 후 한 번만 실행
uv tool install nbdev
uv tool install pre-commit
nbdev-install-hooks
pre-commit install
```

## 9. 자주 만나는 문제

### `nbdev-install-hooks: command not found`

`uv tool install` 직후에는 셸이 새 PATH를 인식하지 못할 수 있다. 새 터미널을 열거나 다음을 실행한다.

```bash
uv tool update-shell
exec $SHELL -l
```

### VSCode 등 다른 에디터에서 저장한 노트북이 정리되지 않는다

`nbdev-install-hooks`가 깔아 두는 clean 훅은 **Jupyter의 pre-save hook**이라, Jupyter Notebook/Lab 서버를 거쳐 저장할 때만 동작한다. 정공법은 6번 항목처럼 pre-commit을 함께 거는 것이다. 일회성으로 한 파일만 정리하고 싶다면 다음 명령을 직접 돌린다.

```bash
nbdev-clean --fname path/to/notebook.ipynb
```

### `nbdev_clean` is not present in repository

pre-commit 설정에서 nbdev 3.x `rev`를 쓰면서 hook id를 예전 방식인 `nbdev_clean`으로 적으면 이런 에러가 난다. `pre-commit autoupdate`도 hook id 이름까지 바꿔 주지는 못하므로 먼저 `id: nbdev-clean`으로 직접 고쳐야 한다. 반대로 nbdev 2.x를 꼭 써야 한다면 `rev: 2.3.34`처럼 2.x 버전에 고정하고 `id: nbdev_clean`을 쓰면 된다.

### `pre-commit`이 매번 커밋을 멈춘다

이건 hook이 잘 동작하고 있다는 뜻이다. `nbdev-clean`이 노트북에서 출력·메타데이터를 지우면서 파일이 바뀌니, pre-commit은 그 변경분을 unstaged 상태로 두고 일단 커밋을 멈춘다. `git add`로 다시 스테이징한 뒤 커밋하면 깔끔하게 들어간다.

### 머지 시에도 충돌이 그대로 남는다

merge driver가 동작하려면 `.gitattributes`의 `*.ipynb merge=...` 규칙과 로컬 git 설정의 driver 정의가 모두 있어야 한다. 저장소를 새로 클론한 뒤 `nbdev-install-hooks`를 한 번도 실행하지 않았다면 로컬 driver 정의가 비어 있어서 적용되지 않는다. 8번 항목을 참고해 클론 직후 한 번 돌려 주면 된다.

### 머지 후 Jupyter에서 "untrusted notebook" 경고가 뜬다

보통 post-merge 훅이 제대로 자리잡지 않았거나 `nbdev-trust`가 PATH에 없는 상태다. `which nbdev-trust`로 명령이 잡히는지 확인하고, 필요하면 `uv tool update-shell`로 PATH를 갱신한 뒤 새 셸에서 다시 머지를 시도한다.

## 10. 정리

- macOS의 Homebrew Python에서는 `pip install nbdev`가 PEP 668 때문에 막힌다.
- `uv tool install nbdev`로 격리된 환경에 CLI 형태로 설치하면 시스템 Python을 건드리지 않으면서 동일한 nbdev 명령을 그대로 쓸 수 있다.
- 기존 git 저장소에서는 `nbdev-install-hooks` 한 번이면 **Jupyter pre-save 훅, git merge driver, post-merge `nbdev-trust` 훅** 세 가지가 한 번에 잡힌다.
- VSCode·PyCharm처럼 Jupyter 서버를 거치지 않는 에디터를 함께 쓴다면 **pre-commit + `nbdev-clean`** 조합으로 커밋 쪽 안전망까지 깔아 두면 좋다.
- 이미 출력이 박혀 있는 노트북은 `nbdev-clean --fname . --clear_all`로 한 번 일괄 정리하면 된다.

`nbdev`를 풀스택 패키지 빌드 도구로 쓰지 않더라도, **노트북을 git 저장소에 넣기 시작하는 순간 git hook 만큼은 켜 두는 편**이 거의 항상 이득이다. `nbdev-install-hooks`로 머지 쪽을, pre-commit으로 커밋 쪽을 막아 두면 노트북 diff가 코드 변경 중심으로 정리되어, 코드 리뷰와 협업이 한결 수월해진다.
