# T[h]inkering 블로그 에이전트 가이드

## 블로그 개요

- **블로그명**: T[h]inkering
- **URL**: ruddyscent.github.io
- **저자**: 전경원 (Kyungwon Chun)
- **플랫폼**: GitHub Pages + Beautiful Jekyll
- **언어**: 한국어 (포스트 전체)

## 포스트 frontmatter 규칙

```yaml
---
layout: post
title: 제목
subtitle: 부제목
tags: [tag1, tag2]
cover-img: /assets/img/develop.jpeg
thumbnail-img: /assets/img/TOPIC.webp
share-img: /assets/img/develop.jpeg
author: 전경원
---
```

- `cover-img`와 `share-img`는 기본적으로 `/assets/img/develop.jpeg`를 쓴다.
- `thumbnail-img`는 주제에 따라 다르다:
  - C#: `csharp.webp`
  - Git: `git.webp`
  - Docker 자체 또는 일반 DevContainer 환경: `docker.webp`
  - 편집기 및 Codex 개발 환경 문제 해결: `editor.webp`
  - Python/ML: `python.webp`
  - PyTorch: `pytorch-wood.webp`
  - HoverPilot/RC: `toy-plane.webp`
  - 기타(기본): `develop.jpeg`
- SEO가 필요한 포스트는 `description` 필드를 추가한다.
- `author`는 항상 `전경원`.

## subtitle 작성 원칙

- 짧고 직접적으로, 사람이 쓴 것처럼 쓴다.
- 절차 설명("~부터 확인한다", "~하는 방법") 형식은 AI 슬롭처럼 보이므로 피한다.
- 포스트 본문 첫 문장의 맥락과 자연스럽게 이어지도록 쓴다.
- 좋은 예: "Xcode의 문제인 줄 알았지만, 원인은 Codex CLI 설정 파일에 있었다"
- 나쁜 예: "ChatGPT 계정 로그인만 실패할 때는 Codex CLI 전역 설정부터 확인한다"

## 본문 문체·시제 원칙

- 독자에게 차분하게 설명하는 자연스러운 존댓말을 쓴다.
- 설치, 설정, 명령 실행, 일반적인 동작처럼 독자가 그대로 따라 할 수 있는 설명은 현재형으로 쓴다.
- 실제로 겪은 증상, 관찰 결과, 원인을 좁혀 간 과정처럼 시간의 흐름이 중요한 대목은 과거형을 유지한다.
- 한 문단에서 현재형과 과거형을 섞을 때는 일반적인 설명과 과거의 경험이 분명히 구분되어야 한다.

## 용어 번역과 영문 병기

- 용어의 직역보다 문맥에 맞는 자연스러운 한국어 설명을 우선한다. 낯선 번역어 하나로 바꾸기보다 무엇을 하는지 풀어 쓴다.
- 전문 용어는 필요한 경우 포스트 전체의 첫 등장에만 `한국어(영어)`로 병기한다. 이후에는 같은 한국어 표현을 사용한다.
- 영어 발음을 그대로 옮긴 용어도 첫 등장에는 원어를 병기한다. 단, 그래프, 로그, 컴파일, 메모리, 스레드, 이슈, 프로그램은 병기하지 않는다. 저장 공간에도 원어를 붙이지 않는다.
- 코드, 변수명, API 이름, 명령어, 파일명, 로그 표식, URL은 번역하지 않는다. 코드 식별자와 본문의 설명을 구분한다.
- 용어를 바꿀 때는 본문뿐 아니라 제목, 표, 링크 표시 문구, 설명용 도식도 확인한다. 영문 병기가 반복되거나 문장 조사가 어색해지지 않았는지 검토한다.

### PyTorch·컴파일러 글의 표현 예시

다음은 이 문맥에서 사용하는 표현이다. 다른 분야에 기계적으로 적용하지 않는다.

| 원어 | 본문 표현 |
| --- | --- |
| functionalization / reinplacing | 함수형 변환 / 제자리 연산 복원 |
| mutation / in-place operation | 제자리 변경 / 제자리 연산 |
| tensor / view / buffer | 텐서 / 뷰 / 버퍼 |
| storage / metadata | 저장 공간 / 메타데이터 |
| clone / copy-back | 복제 / 원래 입력에 다시 복사 |
| fusion / materialize | 연산 융합 / 중간 결과를 실제 메모리에 저장 |
| kernel | 연산 함수(커널, kernel), 이후 연산 함수(커널) 또는 커널 |
| alias / liveness | 별칭 / 생존성 — 공유하는 메모리와 이후 값 사용 여부를 함께 설명 |
| reader / user | 값을 읽는 연산 / 결과를 사용하는 연산 |
| edge / base edge | 연결 / 갱신 대상 텐서로 이어지는 연결 |
| predicate / pass | 판정 조건 / 최적화 단계 |
| negative test | 최적화가 적용되면 안 되는 경우를 검증 |
| assertion | 검증 항목 또는 무엇을 확인하는지 풀어 쓴 문장 |
| eager / compiled | 즉시 실행 / 컴파일된 실행 |
| index | 지정한 위치, 위치 번호 등 문맥에 맞게 풀어 쓰기 |

예를 들어 “인덱스가 중복될 때”는 “같은 위치를 여러 번 지정할 때”로 쓴다. `clone`은 새 저장 공간을 만들어 복제하는 동작이고 `copy_`는 기존 저장 공간에 값을 복사하는 동작이라는 차이를 보존한다.

## 절 제목과 기술 설명

- 본문은 존댓말로 쓰되 절 제목에는 존칭을 쓰지 않는다.
- 같은 수준의 절 제목은 기존 형식을 유지한다. `표제: 설명 문장` 형식이면 일부만 명사구나 다른 형식으로 바꾸지 않는다.
- “대가”처럼 맥락 없이 추상적인 표제를 붙이기보다 해당 절의 내용을 드러내는 표현을 고른다. 아직 합의하지 않은 제목 후보를 확정된 선호로 기록하지 않는다.
- 절 사이에 장식용 가로줄을 넣지 않는다. Jekyll frontmatter의 `---`는 유지한다.
- “프로그램의 의미를 복구한다”, “ATen 수준에서 처리한다”처럼 추상적인 설명은 실제로 어떤 값이나 연산이 어떻게 바뀌는지 풀어 쓴다.
- `add`와 `add_`, `clone`과 `copy_`처럼 비슷해 보이지만 동작이 다른 API는 처음 필요한 곳에서 차이를 설명한다.
- 소스 구현을 설명할 때는 해당 문장 가까이에 GitHub 링크를 둔다. 분석한 커밋으로 고정한 permalink와 필요한 줄 범위의 하이라이트를 사용한다. 본문 링크로 충분하면 별도 참고 자료 절을 중복해서 붙이지 않는다.
- 다이어그램은 설명하려는 차이(서로 다른 노드, 데이터 흐름, 원래 입력에 반영되는 시점 등)가 보이도록 만든다. 코드와 설명용 도식을 구분한다.
- 로그 확인을 권할 때는 켜는 명령, 출력 파일, 확인할 단계·표식·연산을 함께 안내한다. 편집기로 파일을 여는 법이나 `less` 조작법 같은 일반 도구 사용 설명은 요청이 없으면 길게 늘이지 않는다.

## 편집과 확인

- 사용자가 지정한 문장이나 용어의 수정은 해당 범위에 집중한다. 전체 적용을 요청하면 포스트 전체를 검색하되 실행 코드와 URL은 보존한다.
- 윤문을 요청하면 사용 가능한 `humanize-korean` 스킬을 사용하고, 이미 합의한 용어와 병기 규칙을 유지한다. 단순 용어 치환마다 전체 윤문을 반복하지 않는다.
- 수정 전후의 차이에서 코드, 링크, 수치, frontmatter와 기술적 의미가 보존됐는지 확인한다.
- Jekyll 미리보기를 요청하면 작업 중인 worktree를 기준으로 실행한다. 서버가 이미 있으면 수정 반영 여부를 확인하고, 확인하지 않은 결과를 확인했다고 보고하지 않는다.

## 블로그 포스트 생성·윤문 프롬프트

새 포스트의 초안이나 기존 포스트를 윤문할 때는 다음 프롬프트를 사용한다. `{{SAMPLE_MD_CONTENT}}`에는 `sample.md` 전문을, `{{TARGET_POST_CONTENT}}`에는 수정할 포스트 전문을 넣는다. 이 프롬프트는 단독으로 붙여 넣어도 작동해야 하므로 위의 작성 규칙 일부를 의도적으로 반복한다.

```text
당신은 한국어 기술 블로그 전문 편집자입니다.

첨부한 `sample.md`를 문체와 구성의 기준으로 삼아, 대상 블로그 포스트를 자연스럽게 윤문해 주세요. 샘플의 내용을 가져오거나 흉내 낸 표현을 반복하는 것이 아니라, 다음과 같은 글쓰기 특징을 대상 글에 적용하는 작업입니다.

## 목표

- 기술적으로 정확하면서도 비전공자가 흐름을 따라갈 수 있는 글로 다듬습니다.
- 딱딱한 설명문이나 AI가 요약한 듯한 문체를 피합니다.
- 원문의 정보, 주장, 경험, 코드와 기술적 의미는 보존합니다.
- 새로운 사실이나 경험을 임의로 만들어 넣지 않습니다.

## `sample.md`에서 적용할 문체

- 독자에게 차분하게 설명하는 자연스러운 존댓말을 사용합니다.
- 핵심 개념을 먼저 짧게 설명한 뒤 세부 내용으로 들어갑니다.
- 한 문단에는 하나의 중심 내용만 담습니다.
- 문장은 지나치게 길게 늘이지 않고, 원인과 결과가 분명하게 드러나게 씁니다.
- 어려운 용어는 처음 등장할 때 한국어와 영어를 함께 표기합니다.
  - 예: `순환 신경망(Recurrent Neural Network, RNN)`
- 영문 병기는 포스트 전체에서 첫 등장에만 쓰고, 이후에는 한국어 표현을 유지합니다. 그래프, 로그, 컴파일, 메모리, 스레드, 이슈, 프로그램, 저장 공간에는 영문을 병기하지 않습니다.
- 직역보다 문맥에 맞게 동작을 풀어 쓰는 표현을 우선합니다. 코드 식별자와 URL은 번역하지 않습니다.
- 전문 용어를 나열하는 데 그치지 말고, 왜 필요한지와 어떤 역할을 하는지 설명합니다.
- “먼저”, “이어서”, “하지만”, “그래서” 같은 연결 표현은 논리 전개에 필요할 때만 자연스럽게 사용합니다.
- 추상적인 설명 뒤에는 구체적인 동작, 예시 또는 결과를 덧붙입니다.
- 독자의 이해를 돕는 질문형 소제목을 사용할 수 있지만 남용하지 않습니다.
- 과도한 감탄, 홍보성 표현, 상투적인 도입과 결론은 피합니다.

## 블로그 작성 규칙

- 기존 Markdown 구조와 Jekyll frontmatter를 보존합니다.
- 제목, 링크, 이미지, 이미지 설명, 표, 인용문, 코드 블록, 수식은 삭제하거나 손상하지 않습니다.
- 명령어, 코드, 파일명, 옵션, API 이름은 정확히 유지합니다.
- `cover-img`와 `share-img`는 특별한 이유가 없으면 `/assets/img/develop.jpeg`를 사용합니다.
- `author`는 항상 `전경원`으로 유지합니다.
- `subtitle`은 짧고 직접적으로 작성하며, 본문 첫 문장과 자연스럽게 이어지게 합니다.
- subtitle에 “~하는 방법”, “~부터 확인한다”와 같은 기계적인 절차형 표현을 사용하지 않습니다.
- 설치, 설정, 명령 실행, 일반적인 동작은 현재형으로 설명합니다.
- 실제로 겪은 증상이나 원인을 찾아간 과정은 과거형을 유지합니다.
- 한 문단 안에서 현재형과 과거형이 뒤섞이지 않도록 정리합니다.
- 절 제목에는 존칭을 쓰지 않고 같은 수준의 기존 형식(예: `표제: 설명 문장`)을 유지합니다. 절 사이에 장식용 가로줄을 추가하지 않습니다.
- 추상적인 기술 설명은 실제 값과 연산의 변화로 풀어 쓰고, 비슷한 API의 동작 차이를 설명합니다. 소스 링크는 관련 설명 가까이에 커밋과 줄 범위를 고정해 둡니다.
- 로그 확인 방법에는 실행 명령과 살펴볼 출력을 포함하되 일반 편집기나 터미널 조작 설명은 필요한 만큼만 씁니다.

## 반드시 지킬 사항

1. 원문의 기술적 의미와 사실관계를 바꾸지 않습니다.
2. 원문에 없는 성능 수치, 원인, 결과, 경험을 추가하지 않습니다.
3. 코드와 명령어의 동작을 임의로 수정하지 않습니다.
4. 필요한 전문 용어와 정확한 기술 표현을 단순화한다는 이유로 제거하지 않습니다.
5. 같은 내용을 여러 표현으로 반복하지 않습니다.
6. “간단히 말해”, “본질적으로”, “주목할 점은”, “강력한”, “혁신적인”처럼 AI 글에서 자주 보이는 상투어를 습관적으로 사용하지 않습니다.
7. 모든 문단을 비슷한 길이와 구조로 맞추지 않습니다.
8. 원문에 없는 결론이나 교훈을 억지로 덧붙이지 않습니다.
9. 정보가 부족하거나 사실 확인이 필요한 부분은 추측해서 고치지 말고 `[확인 필요: 이유]`로 표시합니다.
10. 원문의 개성과 경험담은 살리되, 불필요하게 장황한 부분만 정리합니다.

## 작업 순서

1. `sample.md`에서 문장 길이, 설명 순서, 용어 소개 방식, 문단 리듬을 파악합니다.
2. 대상 포스트의 핵심 주장과 기술 정보를 식별합니다.
3. 중복되거나 흐름을 끊는 문장을 정리합니다.
4. 문단 사이의 논리적 연결을 보완합니다.
5. 샘플과 어울리는 자연스러운 한국어 문체로 전체 글을 윤문합니다.
6. 코드, 링크, 이미지, frontmatter가 원문과 일치하는지 마지막으로 확인합니다.

## 출력 형식

- 설명이나 작업 보고를 덧붙이지 말고, 수정된 Markdown 전문만 출력합니다.
- 완성된 글은 별도의 수정 없이 `.md` 파일에 붙여 넣을 수 있어야 합니다.

## 참고 문체

<sample>
{{SAMPLE_MD_CONTENT}}
</sample>

## 수정할 블로그 포스트

<target>
{{TARGET_POST_CONTENT}}
</target>
```

## 주요 콘텐츠 카테고리

### HoverPilot 시리즈

RC 비행기를 강화학습으로 조종하는 프로젝트. 태그: `hoverpilot`.

- RealFlight 시뮬레이터 연동 (RFLink, SOAP)
- Gymnasium 환경 래핑
- Reward 설계
- PPO 구현 및 TensorBoard 로깅

### C# / 코딩테스트

프로그래머스 PCCP 기반. 태그: `csharp`, `pccp`, `codingtest`.

- 런타임 환경, 배열, 정렬, Length vs Count 등

### Codex / AI 도구 문제 해결

Codex CLI 로그인 이슈 3부작. 태그: `codex`, `troubleshooting`.

- DevContainer (포트 충돌, 자격증명 마운트)
- Remote-SSH (OAuth 콜백 localhost 불일치)
- Xcode (Codex CLI 전역 설정 파일 충돌)

### 개발 환경

- Jekyll DevContainer 로컬 개발 환경
- Jetson Xavier NX Docker Compose / NGC
- Apple Silicon PyTorch + uv 설치

### Python / ML

- 파이썬 장식자
- nbdev + git hooks (Jupyter 노트북 diff 정리)
- RL Book 코세라 강의 동반 정리

### Git

- 커밋 접두사(Conventional Commits) 규칙
- 자주 쓰는 Git 명령어

### 기타

- Zed vs VSCode (ML 엔지니어 관점)
- FDTD 시뮬레이션 (GPU 가속, PHANTOM 프로젝트)
- 점프와 순간이동 — 그리디 알고리즘

## 파일 구조

```text
_posts/         # YYYY-MM-DD-slug.md 형식
assets/img/     # 이미지 (webp 권장)
aboutme.md      # 소개 페이지
_config.yml     # 사이트 설정
```

## .gitignore 관리

- `.gitignore`는 [Toptal gitignore generator](https://www.toptal.com/developers/gitignore)의 생성 형식으로 관리한다.
- 템플릿을 추가하거나 변경할 때는 파일을 임의로 재구성하지 말고, 첫 줄의 API 템플릿 목록을 갱신해 Toptal에서 전체 파일을 다시 생성한다.
- 생성된 `# Created by`, `### Template ###`, `# End of` 표식과 템플릿 순서·주석을 유지한다.
- 저장소 전용 예외가 필요하면 생성 영역과 구분되는 별도 섹션에 최소한으로 추가한다.

## 포스트 날짜

일부 포스트는 미래 날짜(2027-)로 예약되어 있다. GitHub Pages는 미래 날짜 포스트를 빌드 시점 기준으로 공개하므로, 날짜가 미래여도 포스트 파일이 `_posts/`에 있으면 빌드에 포함된다.

## GitHub 작업 흐름

`master`는 GitHub의 `Protect master` 룰셋으로 보호한다. `master`에 직접 push하지 않고 PR로만 병합하며, squash 병합과 검증된 서명을 사용한다. GitHub Issues는 현재 비활성화되어 있으므로 이슈 생성을 작업 시작 조건으로 두지 않는다.

### 작업 브랜치와 worktree

- 새 작업은 최신 `origin/master`에서 전용 브랜치와 worktree를 함께 만든다. 기본 작업 디렉터리의 `master`에는 직접 작업하지 않는다.
- worktree를 만들기 전에 `git worktree list`로 같은 브랜치나 경로가 이미 있는지 확인한다.
- 새 브랜치와 worktree는 다음처럼 만든다. `<type>`에는 `docs`, `fix`, `feat`처럼 작업 성격을, `<short-name>`에는 짧은 작업 이름을 쓴다.

  ```bash
  git fetch origin
  git worktree add -b <type>/<short-name> ../ruddyscent.github.io-<short-name> origin/master
  ```

- 이미 만든 브랜치를 다른 worktree에서 열 때는 새 브랜치를 만들지 않는다.

  ```bash
  git worktree add ../ruddyscent.github.io-<short-name> <type>/<short-name>
  ```

### PR, 병합, 정리

- PR을 열기 전에 작업 브랜치를 `origin/master` 위로 rebase하고, 충돌과 검증 실패를 해결한다. rebase 뒤에 강제 push가 필요하면 본인 작업 브랜치에만 `--force-with-lease`를 사용하며, `master`에는 절대 사용하지 않는다.
- `master`에 들어가는 커밋은 모두 검증 가능한 서명이 있어야 한다. PR은 GitHub에서 squash 방식으로만 병합하고, 대화와 필수 검사를 모두 해결한 뒤 병합한다.
- 병합이 완료되었는지 `gh pr view <number> --json state,mergedAt`으로 확인한다. `state`가 `MERGED`가 아니거나 병합 과정에 문제가 있으면 worktree와 브랜치를 삭제하지 않는다.
- 병합을 확인했고 worktree에 미커밋 변경이 없으면 worktree, 로컬 브랜치, 남아 있는 원격 브랜치를 차례로 정리한다.

  ```bash
  git worktree remove ../ruddyscent.github.io-<short-name>
  git branch -D <type>/<short-name>
  git push origin --delete <type>/<short-name>
  git worktree prune
  ```

- squash 병합은 로컬 작업 브랜치의 커밋을 그대로 포함하지 않으므로 `git branch -d`가 실패할 수 있다. PR의 병합 상태와 추가 커밋이 없음을 확인한 경우에만 `git branch -D <type>/<short-name>`으로 로컬 브랜치를 정리한다. 원격 브랜치가 GitHub에서 이미 삭제됐다면 삭제 명령을 반복하지 않는다.

## 커밋 메시지 형식

- 형식: `type(scope): 요약` — `docs(blog)`, `docs`, `feat(blog)`, `fix`, `chore(devcontainer)` 등 Conventional Commits 접두사를 쓴다.
- 요약은 "무엇을 바꿨는지"보다 "왜 바꿨는지"에 가깝게 쓴다.
- 변경 사항이 여러 개면 본문을 문단이 아니라 하이픈 목록으로 정리한다. 각 항목은 하나의 변경/이유를 명령형으로 짧게 쓴다.
- 변경이 한 가지뿐이면 목록 없이 요약 한 줄로 끝낸다 (예: `docs: rewrite subtitle to sound less like AI slop`).

```
docs(blog): reframe Zed vs VSCode post as workflow split, not verdict

- Replace dismissive "Zed falls short" framing with role-based delegation between the two editors
- Route Dev Container and Jupyter workflows to VSCode's proven integration instead of calling them Zed gaps
- Link LSP to its spec on first mention
```
