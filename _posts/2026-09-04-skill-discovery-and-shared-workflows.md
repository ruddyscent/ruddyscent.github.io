---
layout: post
title: 심볼릭 링크로 Claude Code와 Codex의 리뷰 스킬 공유하기
subtitle: PyTorch의 리뷰 지침은 하나로 두고 탐색 경로만 연결한다
tags: [codex, git, troubleshooting, skills, pytorch]
cover-img: /assets/img/develop.jpeg
thumbnail-img: /assets/img/pytorch-wood.webp
share-img: /assets/img/develop.jpeg
author: 전경원
description: PyTorch의 pr-review 스킬 공유 사례로 살펴보는 Codex 탐색 경로, 상대 심볼릭 링크와 Git 파일 모드, 검증 범위.
---

PyTorch 저장소는 코드 리뷰에 `pr-review` 스킬(skill)을 사용하도록 정하고 있습니다. 이 스킬은 Claude Code가 찾는 `.claude/skills`에 있지만 `.agents/skills`를 탐색하는 Codex는 찾지 못합니다. [Issue #195305](https://github.com/pytorch/pytorch/issues/195305)에 보고된 문제입니다.

아직 적용되지 않은 [PR #195421](https://github.com/pytorch/pytorch/pull/195421)은 기존 스킬 디렉터리를 가리키는 심볼릭 링크(symlink) 하나를 추가하자고 제안합니다. 리뷰 지침을 복사하지 않고 두 도구가 같은 원본을 읽도록 합니다.

PR에서 추가하는 [링크 파일](https://github.com/pytorch/pytorch/blob/e69336b285316416d9c19a19a51cb96c02347ad6/.agents/skills/pr-review#L1)의 변경 내역(diff)은 다음과 같습니다.

```diff
new file mode 120000
+++ b/.agents/skills/pr-review
+../../.claude/skills/pr-review
```

Codex는 현재 작업 디렉터리부터 저장소 루트까지 각 디렉터리의 `.agents/skills`를 탐색하며 그 안의 심볼릭 링크도 따라갑니다. [OpenAI 공식 문서](https://learn.chatgpt.com/docs/build-skills)

이 글은 PR의 [`e69336b` 커밋](https://github.com/pytorch/pytorch/commit/e69336b285316416d9c19a19a51cb96c02347ad6)을 기준으로 설명합니다. 진단 절차와 도구별 실행 방법에 관한 제안은 변경 내용을 이해하기 위해 덧붙인 설명입니다. PR에서 모두 구현하거나 검증한 내용은 아닙니다.

## 1. 해결책은 내용을 복사하는 것이 아니다

가장 간단해 보이는 해결책은 파일을 복사하는 것입니다.

```text
.claude/skills/pr-review/SKILL.md   — 원본
.agents/skills/pr-review/SKILL.md   — 복사본
```

처음에는 두 파일이 같습니다. 하지만 한 달 뒤 리뷰 규칙을 바꾼다고 가정해 보겠습니다. 한쪽에는 “회귀 테스트를 확인하라”를 추가하면서 다른 쪽에는 빠뜨리면 같은 저장소에서 사용하는 도구에 따라 리뷰 절차가 달라집니다.

복사 시점에는 같은 설정이 시간이 지나면서 서로 어긋나는 문제(configuration drift)입니다.

PR은 기존 디렉터리를 유지하고 다른 이름으로 접근하게 합니다.

```text
.claude/skills/pr-review/        실제 원본
.agents/skills/pr-review         원본을 가리키는 심볼릭 링크
```

이렇게 하면 리뷰 지침을 수정할 곳이 하나뿐입니다. 하나의 기준 원본(Single Source of Truth)을 유지하는 설계입니다.

중복을 피하자는 DRY(Don't Repeat Yourself) 원칙도 같습니다. 같은 텍스트를 두 번 쓰는 중복뿐 아니라 같은 정책을 두 군데에서 독립적으로 관리하는 문제도 피합니다.

원본 변경은 두 도구에 모두 영향을 줍니다. 중복 관리 비용은 줄지만 공통 원본을 수정할 때는 양쪽 도구의 동작을 고려해야 합니다.

## 2. 심볼릭 링크는 파일 내용이 아니라 목적지를 가리킨다

심볼릭 링크는 다른 경로를 가리키는 파일시스템 항목입니다. 링크를 통해 파일을 열면 운영체제가 대상 경로를 따라갑니다. 위 변경으로 기존 스킬 디렉터리에 접근할 경로가 하나 더 생깁니다. 리뷰 지침의 복사본을 추가하지는 않습니다.

파일 복사, 심볼릭 링크, 하드 링크(hard link)는 저장하는 대상과 원본 수정의 영향이 다릅니다.

| 방식 | 저장하는 것 | 원본 수정의 영향 |
|---|---|---|
| 파일 복사 | 별도의 내용 | 복사본에는 자동 반영되지 않음 |
| 심볼릭 링크 | 대상 경로 | 같은 원본에 접근하므로 반영됨 |
| 하드 링크 | 같은 파일 객체를 가리키는 디렉터리 항목 | 같은 파일 내용에 반영됨 |

이 사례에서는 스킬 디렉터리 전체를 공유해야 합니다. macOS와 Linux에서는 일반적으로 디렉터리에 하드 링크를 만들 수 없으므로, 기존 디렉터리를 가리키는 심볼릭 링크를 사용합니다.

심볼릭 링크는 대상의 이름을 저장하므로 원본을 옮기거나 삭제하면 끊어질 수 있습니다. 링크 자체가 남아 있어도 목적지에 도달하지 못하는 끊어진 심볼릭 링크(dangling symlink)가 됩니다.

## 3. `../../`는 어디를 기준으로 계산할까?

상대 심볼릭 링크를 읽을 때는 경로의 기준부터 확인합니다.

```text
../../.claude/skills/pr-review
```

이 경로는 명령을 실행한 셸(shell)의 현재 디렉터리가 아니라 **링크가 들어 있는 디렉터리**를 기준으로 해석합니다.

```text
링크 위치:       repo/.agents/skills/pr-review
링크의 부모:     repo/.agents/skills/
첫 번째 ..:     repo/.agents/
두 번째 ..:     repo/
나머지 경로:     .claude/skills/pr-review
최종 목적지:     repo/.claude/skills/pr-review
```

절대 경로를 저장하면 어떨까요?

```text
/home/alice/pytorch/.claude/skills/pr-review
```

Alice의 컴퓨터에서는 동작할 수 있지만 Bob이 저장소를 체크아웃(checkout)한 위치나 지속적 통합(Continuous Integration, CI) 환경에는 그 경로가 없을 수 있습니다. 상대 경로는 저장소 내부 구조가 유지되는 한 체크아웃의 절대 위치가 바뀌어도 동작합니다.

다만 “어디로 옮겨도 안전하다”는 뜻은 아닙니다. 저장소 전체를 이동하면 상대적 배치가 유지되지만 내부의 `.claude` 디렉터리만 이동하면 링크가 끊어질 수 있습니다.

## 4. 한 줄짜리 변경 내역에서 파일 모드가 중요한 이유

Git 변경 내역의 `120000`은 심볼릭 링크를 뜻합니다. Git은 링크의 대상 경로를 블롭(blob) 내용으로 저장하고 트리 항목(tree entry)의 모드(mode)로 심볼릭 링크임을 구분합니다. 링크 대상을 통째로 복사해 저장하지는 않습니다.

아래 두 파일은 겉으로 같은 경로 문자열을 보여도 의미가 다릅니다.

```text
A: 심볼릭 링크이며 대상이 ../../.claude/skills/pr-review
B: 일반 텍스트 파일에 그 문자열만 적혀 있음
```

A에는 `A/SKILL.md`처럼 접근할 수 있습니다. B는 디렉터리가 아니므로 그렇게 접근할 수 없습니다.

이 차이는 체크아웃 환경에서도 중요합니다. Git의 `core.symlinks`가 `false`이면 심볼릭 링크가 대상 경로를 담은 작은 일반 파일로 체크아웃될 수 있습니다. 저장소에 심볼릭 링크가 올바르게 기록돼 있어도 모든 작업 환경에서 같은 형태로 복원되는 것은 아닙니다. [Git 공식 문서: core.symlinks](https://git-scm.com/docs/git-config#Documentation/git-config.txt-coresymlinks)

그래서 이런 변경을 읽을 때는 추가된 텍스트와 파일 모드를 함께 봐야 합니다.

## 5. 테스트 두 줄은 서로 다른 사실을 증명한다

PR 본문에는 다음 검사가 기록되어 있습니다.

```bash
test -L .agents/skills/pr-review
test -f .agents/skills/pr-review/SKILL.md
```

첫 줄은 해당 경로 자체가 심볼릭 링크인지 검사합니다. 대상이 사라져도 링크는 남아 있을 수 있으므로 첫 검사만 통과할 수 있습니다.

둘째 줄은 링크를 따라가 `SKILL.md`라는 일반 파일에 도달할 수 있는지 검사합니다. 다만 이 검사만으로 메타데이터(metadata)가 올바르거나 도구가 실제로 스킬을 선택한다고 결론 내릴 수는 없습니다. [PR의 테스트 기록](https://github.com/pytorch/pytorch/pull/195421)

각 검사가 확인하는 범위를 나누면 다음과 같습니다.

| 계층 | 질문 | 예시 |
|---|---|---|
| 파일 구조 | 진짜 링크이며 목적지에 도달하는가? | `test -L`, `test -f` |
| 형식 | 스킬 메타데이터가 유효한가? | 형식 검증 도구(validator) |
| 발견 | 실제 도구의 스킬 목록에 나타나는가? | 해당 환경에서 목록 확인 |
| 선택 | 적절한 요청에 이 스킬이 선택되는가? | 명시적·암시적 호출 테스트 |
| 수행 | 요구한 리뷰 절차가 실제로 지켜지는가? | 실행 기록과 결과 검토 |

파일 구조를 확인하는 테스트로 선택이나 수행 단계의 성공까지 입증할 수는 없습니다. 링크 검사와 정적 검사(lint)가 통과했더라도 “모든 도구에서 리뷰 절차를 끝까지 검증했다”고 쓰면 실제로 확인한 범위를 넘어섭니다.

## 6. 작은 실습: 링크와 목적지를 따로 확인하기

다음은 macOS나 Linux의 일반적인 셸에서 실행할 수 있는 실습입니다. 실제 저장소 설정을 바꾸지 않고 새 임시 디렉터리에 확인용 빈 파일(marker)과 링크를 만듭니다. 스킬의 형식 검증은 하지 않고 파일시스템 동작만 확인합니다.

```bash
demo_dir=$(mktemp -d)
mkdir -p "$demo_dir/.claude/skills/pr-review"
mkdir -p "$demo_dir/.agents/skills"
touch "$demo_dir/.claude/skills/pr-review/SKILL.md"

ln -s ../../.claude/skills/pr-review "$demo_dir/.agents/skills/pr-review"

readlink "$demo_dir/.agents/skills/pr-review"
test -L "$demo_dir/.agents/skills/pr-review" && echo "link exists"
test -f "$demo_dir/.agents/skills/pr-review/SKILL.md" && echo "target exists"
```

`readlink`는 링크에 저장된 상대 경로를 보여 줍니다. 두 `test`는 각각 링크와 목적지 파일을 확인합니다.

이제 임시 디렉터리 안의 원본 이름만 바꿔 보겠습니다.

```bash
mv "$demo_dir/.claude/skills/pr-review" "$demo_dir/.claude/skills/pr-review-renamed"

test -L "$demo_dir/.agents/skills/pr-review" && echo "link still exists"
if ! test -f "$demo_dir/.agents/skills/pr-review/SKILL.md"; then
    echo "target is no longer reachable"
fi
```

링크는 여전히 존재하지만 목적지에는 도달할 수 없습니다. 이름을 되돌리면 복구됩니다.

```bash
mv "$demo_dir/.claude/skills/pr-review-renamed" "$demo_dir/.claude/skills/pr-review"
```

이 차이를 직접 확인하면 `test -L`만으로는 충분하지 않은 이유가 분명해집니다. 실습 후 임시 디렉터리는 남아 있으므로 필요가 없으면 경로를 확인한 뒤 정리하면 됩니다.

## 7. 스킬을 찾고 선택하고 실행하는 과정

Codex는 스킬의 이름과 설명을 먼저 확인하고, 사용할 스킬을 선택하면 전체 `SKILL.md`를 읽습니다. 사용자가 스킬을 직접 지정하거나 요청 내용이 `description`과 맞으면 선택될 수 있습니다. [OpenAI 공식 문서](https://learn.chatgpt.com/docs/build-skills)

```yaml
---
name: example-review
description: Review a code change for correctness and missing regression tests.
---
```

문제가 생기면 어느 단계에서 막히는지 확인합니다. 스킬 목록에 없다면 탐색 경로를, 목록에 있지만 선택되지 않으면 설명과 호출 조건을 살펴봅니다. 선택된 뒤 절차를 수행하지 못하면 지침 본문과 필요한 도구, 실행 환경을 확인합니다.

심볼릭 링크로 같은 지침을 읽더라도 도구마다 제공하는 명령이나 승인 방식은 다를 수 있습니다. 이는 해당 PR에서 확인한 결함이 아니라 스킬을 여러 도구에서 공유할 때 살펴볼 조건입니다. 실제 차이가 생기면 공통 리뷰 절차는 유지하고 도구별 실행 방법만 나누면 됩니다.

## 맺음말

이번 사례에서는 한 줄로 기존 원본을 Codex의 스킬 탐색 경로에 연결합니다. 여러 도구에 같은 내용을 복사하지 않아도 되지만 파일에 도달하는지 확인하는 검사만으로 리뷰 수행까지 검증할 수는 없습니다.

AI 도구가 지침을 따르지 못할 때는 문장을 고치기 전에 경로를 확인해 보면 좋겠습니다. 그 지침은 어디에 있고 도구는 어디를 찾으며 실제로 무엇을 읽을까요? 지침을 작성할 때는 도구가 그 지침에 도달하는 구조도 개발 환경의 일부로 살펴봅니다.

> 작성 시점인 2026년 9월 4일 기준 PR #195421은 아직 병합되지 않은 상태입니다. 세부 구현과 테스트는 리뷰 과정에서 변경될 수 있습니다.
