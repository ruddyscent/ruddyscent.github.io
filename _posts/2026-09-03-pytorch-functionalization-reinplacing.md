---
layout: post
title: PyTorch Inductor의 불필요한 텐서 복사 줄이기
subtitle: PyTorch 함수형 변환과 제자리 연산 복원을 작은 예제로 이해하기
tags: [pytorch, compiler, optimization, python, inductor, functionalization, reinplacing]
cover-img: /assets/img/develop.jpeg
thumbnail-img: /assets/img/pytorch-wood.webp
share-img: /assets/img/develop.jpeg
author: 전경원
---

우리가 작성하는 프로그램에는 값을 직접 바꾸는 코드가 흔히 등장합니다. 다음은 PyTorch 텐서(tensor) `x`의 일부 원소에 10을 더하는 코드입니다.

```python
x[1:4] += 10
```

사람에게는 간단한 코드입니다. 하지만 컴파일러 입장에서는 질문이 많습니다.

- `x[1:4]`는 별도 배열인가, `x`의 일부를 바라보는 뷰(view)인가?
- 이 연산 뒤에서 누군가 변경 전의 `x`를 읽지는 않는가?
- `x`와 같은 메모리를 공유하는 다른 텐서가 있는가?
- 앞뒤 연산을 하나의 연산 함수(커널, kernel)로 합쳐도 되는가?

이런 문제를 다루는 컴파일러가 **PyTorch Inductor(TorchInductor)**입니다. 파이썬 함수나 모델에 [`torch.compile`](https://docs.pytorch.org/docs/stable/generated/torch.compile)을 적용하면 기본으로 사용하는 컴파일러로, 텐서 연산의 흐름을 나타낸 그래프를 최적화해 CPU나 GPU에서 실행할 코드를 만듭니다.

값을 제자리에서 바꾸는 **제자리 변경(mutation)**은 프로그램의 의미를 시간과 실행 순서에 의존하게 만듭니다. 그래서 그래프 컴파일러는 종종 제자리 변경을 일단 없앱니다. 그런데 제자리 변경을 없애려고 배열 전체를 복사하면 성능이 떨어질 수 있습니다. 결국 컴파일러는 분석하기 편하도록 제자리 변경을 제거한 다음, 안전하고 이득이 되는 곳에서는 다시 제자리 변경으로 되돌립니다.

이 글은 그 두 단계인 **함수형 변환(functionalization)**과 **제자리 연산 복원(reinplacing)**을 살펴봅니다. Inductor의 [이슈 #195285](https://github.com/pytorch/pytorch/issues/195285)와 [PR #195292](https://github.com/pytorch/pytorch/pull/195292)를 통해 불필요한 복사가 남는 이유와 이를 줄이는 과정을 따라가 보겠습니다.

## 1. 제자리 변경은 왜 분석하기 어려울까?

다음 파이썬 코드를 생각해 보겠습니다.

```python
x = [1, 2, 3, 4]
y = x
x[1] = 20
print(y[1])
```

출력은 `2`가 아니라 `20`입니다. `x`와 `y`가 같은 객체를 가리키기 때문입니다. `x`라는 이름만 보고는 변경의 영향을 알 수 없습니다. 같은 객체를 가리키는 모든 이름, 즉 별칭(alias)을 찾아야 합니다.

텐서 뷰에서도 같은 일이 일어납니다.

```python
import torch

x = torch.tensor([1, 2, 3, 4])
y = x[1:3]
y.add_(10)

print(x)  # tensor([1, 12, 13, 4])
```

여기서 `add_()` 끝의 밑줄(`_`)에 주목하면 좋습니다. PyTorch의 텐서 메서드는 이름 끝에 밑줄을 붙여 기존 값을 직접 수정하는 제자리 연산(in-place operation)임을 나타냅니다. [`y.add(10)`](https://docs.pytorch.org/docs/stable/generated/torch.Tensor.add.html)은 `y`를 그대로 두고 각 원소에 10을 더한 새 텐서를 반환합니다. 반면 [`y.add_(10)`](https://docs.pytorch.org/docs/stable/generated/torch.Tensor.add_.html)은 `y` 자체의 값을 바꿉니다. 위 코드에서는 `add_()` 대신 `add()`를 호출하고 반환값을 사용하지 않으면 덧셈 결과가 `y`에 반영되지 않습니다.

`y`는 데이터를 복사한 새 텐서가 아니라 `x`의 데이터 저장 공간 일부를 공유하는 뷰입니다. `y.add_(10)`을 실행하면 `x`까지 바뀝니다.

컴파일러가 `y.add_(10)`을 다른 연산보다 앞으로 옮기거나 두 연산을 하나의 연산 함수(커널)로 합쳐 실행하려면 그 사이에 이전 값을 읽는 연산이 없는지 확인해야 합니다. 제자리 변경이 있으면 데이터 의존성뿐 아니라 **시간에 따른 상태 변화**까지 추적해야 합니다.

## 2. 함수형 변환: 변경 대신 새 값을 만들기

함수형 변환은 제자리 변경을 함수형 연산으로 바꿉니다. 함수형 연산은 입력을 바꾸지 않고 새로운 출력을 반환합니다.

원래 코드가 다음과 같다면:

```python
x[1:3].add_(10)
```

개념적으로는 다음처럼 바꿀 수 있습니다.

```python
part = x[1:3]
new_part = part + 10
new_x = slice_scatter(x, new_part, start=1, end=3)
x.copy_(new_x)
```

중간 그래프에서는 변화 전의 값과 변화 후의 값이 서로 다른 노드가 됩니다.

```text
                 ┌──────────────┐
x ──────────────▶│ slice_scatter│────▶ new_x
                 └──────▲───────┘
                        │
x ─▶ slice ─▶ add(10) ──┘

x ◀──────────── copy_(x, new_x)
```

마지막 `copy_`는 새로 계산한 값을 원래 `x`에 복사합니다. 원래 코드처럼 호출자가 넘긴 `x` 자체에도 변경 결과를 반영하는 것입니다.

PyTorch 코드에서는 [`functionalize()`가 입력을 감싸고 함수를 실행하는 부분](https://github.com/pytorch/pytorch/blob/e4b9ce80a342100d49acab38bb4bf7ba0bf87118/torch/_functorch/eager_transforms.py#L1774-L1789)부터 살펴보면 이 과정을 이해하기 쉽습니다. 그 아래의 [입력 동기화와 `_propagate_functional_input_mutation()` 호출](https://github.com/pytorch/pytorch/blob/e4b9ce80a342100d49acab38bb4bf7ba0bf87118/torch/_functorch/eager_transforms.py#L1791-L1810)이 변경 결과를 원래 입력에 돌려주는 부분입니다.

이 파이썬 함수는 변환을 감싸는 진입점입니다. PyTorch는 내부의 텐서 연산 계층인 ATen에서 실제 변환을 수행합니다. 파이썬 코드의 문장을 직접 고치는 대신 실행되는 텐서 연산을 바꿉니다. 예를 들어 입력을 직접 바꾸는 `aten.add_`를 새 결과를 반환하는 `aten.add`로 바꿉니다.

이 표현에서 각 노드는 대체로 "입력을 받아 새 출력을 만든다"는 형태입니다. 덕분에 컴파일러가 연산 사이의 데이터 의존성을 파악하기 쉬워집니다. 공통 부분식과 사용하지 않는 값을 제거하고 연산을 재배치할 수도 있습니다. 여러 연산을 하나의 커널로 합치는 연산 융합(fusion), 자동 미분, 그래프 변환도 다루기 수월합니다.

함수형 변환은 실행을 바로 빠르게 만드는 최적화라기보다 **다른 최적화를 가능하게 하는 정규화 단계**에 가깝습니다.

## 3. 대가: 작은 변경 때문에 전체를 복사할 수 있다

함수형 업데이트는 원본을 보존해야 합니다. 가장 단순한 구현은 입력 전체를 복제(clone)한 뒤 복제본 일부를 바꾸는 것입니다.

```python
def functional_slice_update(x, new_values):
    out = x.clone()
    out[1:-1].copy_(new_values)
    return out
```

수백 MB 텐서에서 원소 몇 개만 바꾸더라도 전체 텐서 복제가 발생할 수 있습니다.

```text
변경할 영역:     99개 원소
복사할 전체 입력: 51,520개 원소
```

이슈 #195285의 예에서는 입력이 `float64` 51,520개이므로 한 번 복제하는 데이터의 크기는 다음과 같습니다.

```text
51,520 × 8 bytes = 412,160 bytes
```

이 예에서는 복사가 매 호출마다 발생합니다. 작은 산술 연산에서는 계산보다 메모리를 읽고 쓰는 비용이 더 커질 수 있습니다.

그렇다면 처음부터 제자리 변경을 유지하면 되지 않을까? 항상 그렇지는 않습니다. 컴파일러가 함수형 `slice_scatter`를 앞뒤 연산과 융합해 별도 중간 결과를 만들지 않는다면 제자리 변경보다 오히려 유리할 수 있습니다. 제자리 변경에는 텐서 데이터를 담는 메모리 공간인 버퍼(buffer)가 필요합니다. 이 때문에 중간 결과를 실제 메모리에 저장(materialize)하도록 강제하고 연산 융합의 선택지를 줄이기도 합니다.

컴파일러가 답해야 할 질문은 두 개입니다.

1. **안전한가?** 원본을 직접 바꿔도 프로그램 의미가 같은가?
2. **이득인가?** 직접 바꾸는 것이 연산 융합이나 다른 최적화를 포기할 만큼 유리한가?

이 둘은 같은 질문이 아닙니다.

## 4. 제자리 연산 복원: 안전할 때 제자리 변경을 복원하기

제자리 연산 복원은 함수형 연산을 다시 기존 텐서를 직접 수정하는 제자리 연산으로 바꾸는 최적화입니다.

다음 그래프를 보겠습니다.

```python
new_x = functional_update(x)
x.copy_(new_x)
```

`new_x`는 결국 `x`로 되돌아갑니다. 그 사이에 변경 전 `x`를 관찰하는 연산이 없다면 다음처럼 실행할 수 있습니다.

```python
inplace_update(x)
```

전체 텐서를 복제하는 작업과 계산 결과를 원래 입력에 다시 복사하는 작업이 모두 사라집니다.

하지만 아래 변환은 안전하지 않습니다.

```python
new_x = functional_update(x)
old_sum = x.sum()              # 변경 전 x가 필요하다
x.copy_(new_x)
```

이를 곧바로 다음처럼 바꾸면:

```python
inplace_update(x)
old_sum = x.sum()              # 이제 변경 후 x를 읽는다
```

결과가 달라집니다. 제자리 연산 복원에는 이후 연산이 이전 값을 사용하는지 확인하는 생존성(liveness) 분석과 같은 메모리를 공유하는 텐서를 추적하는 별칭 분석이 필요합니다.

### 안전성 검사의 핵심

후보 연산 `update(x)`를 제자리 연산으로 바꾸려면 적어도 다음을 확인해야 합니다.

- `x`와 저장 공간을 공유하는 뷰가 있는가?
- 그 뷰 또는 `x`의 이전 값을 제자리 변경 뒤에서 읽는가?
- 읽기가 최종 `copy_`보다 앞에 있는가?
- 겹치는 메모리 영역 때문에 쓰기 순서가 결과에 영향을 주는가?
- 최종 `copy_`가 결과를 정말 같은 그래프 입력으로 다시 복사하는가?

간단히 표현하면 다음과 같습니다.

```text
갱신 후보 ───────────────▶ 최종 copy_
     │                         │
     └─ 이 구간에서 변경 전 x의 값이 필요하면 제자리 연산 복원 불가
```

PyTorch Inductor의 실제 구현은 저장 공간별로 관련 노드를 모은 뒤 메모리 영역이 겹치는지, 이후에 해당 값을 사용하는 연산이 있는지 검사합니다. 그래프 입력은 최종 `copy_`가 있을 때만 제자리 연산 복원의 후보가 됩니다. 그렇지 않으면 함수가 입력을 변경하지 않는다는 원래 의미를 깨뜨릴 수 있기 때문입니다.

여기서 생존성 검사는 [`any_use_of_views_after_node()`](https://github.com/pytorch/pytorch/blob/e4b9ce80a342100d49acab38bb4bf7ba0bf87118/torch/_inductor/fx_passes/reinplace.py#L562-L599)에 대응합니다. 후보 연산 뒤부터 최종 `copy_` 앞까지 값을 읽는 연산을 살펴보는 코드입니다. 메모리 겹침과 그래프 입력으로 다시 복사하는 조건은 [`can_inplace()`의 검사](https://github.com/pytorch/pytorch/blob/e4b9ce80a342100d49acab38bb4bf7ba0bf87118/torch/_inductor/fx_passes/reinplace.py#L620-L648)에서 확인할 수 있습니다.

## 5. 실제 사례: 슬라이스를 제자리에서 변경한 뒤 지정한 위치의 값 바꾸기

이 절부터 살펴볼 기존 구현은 PR의 기준 커밋 `e4b9ce8`, 개선안은 PR 커밋 `c0ed45b`를 기준으로 합니다. 작성 시점에 개선안은 아직 병합되지 않은 초안이므로 기존 구현과 구분해서 읽으면 좋습니다.

이슈 #195285의 핵심 코드는 다음과 같습니다.

```python
def composed(field, delta, indices, gain):
    field[1:-1].add_(delta)
    values = torch.index_select(field, 0, indices) * gain
    field.index_copy_(0, indices, values)
```

이 코드는 두 번 값을 바꿉니다.

1. `field[1:-1]`에 `delta`를 더합니다.
2. 방금 갱신한 `field`에서 일부 값을 읽어 `indices` 위치에 다시 씁니다.

두 번째에 쓸 값은 첫 번째 변경 결과에 의존합니다. 함수형 변환 뒤의 그래프는 개념적으로 다음과 비슷합니다.

```text
field
  │
  ▼
generalized_scatter ───────▶ index_select ─▶ multiply ─┐
  │                                                    │ values
  └───────────────────────▶ index_put ◀────────────────┘
                                 │
                                 ▼
                         copy_(field, result)
```

`generalized_scatter`는 여러 종류의 뷰 갱신을 한 형태로 다루기 위한 Inductor 내부 표현입니다. 함수형 구현은 개념적으로 다음과 같습니다.

```python
def generalized_scatter(field, src, view_ops):
    out = field.clone()
    apply_views(out, view_ops).copy_(src)
    return out
```

실제 [`_generalized_scatter()`](https://github.com/pytorch/pytorch/blob/e4b9ce80a342100d49acab38bb4bf7ba0bf87118/torch/_inductor/fx_passes/reinplace.py#L119-L123)도 `inp.clone()`을 호출한 뒤 복제본을 갱신합니다. 함께 호출되는 [`_inplace_generalized_scatter()`](https://github.com/pytorch/pytorch/blob/e4b9ce80a342100d49acab38bb4bf7ba0bf87118/torch/_inductor/fx_passes/reinplace.py#L92-L116)는 뷰를 차례로 적용하고 마지막에 `tmp.copy_(src)`를 실행합니다. 위 의사 코드의 두 줄이 각각 어느 함수에 해당하는지 비교해 볼 수 있습니다.

기존 구현의 수익성 판단은 다음처럼 직접 연결된 형태를 인식합니다.

```text
generalized_scatter ─▶ copy_(original_input, ...)
```

그러나 이 예의 실제 그래프에서는 중간에 `index_put`이 있습니다.

```text
generalized_scatter ─▶ index_put ─▶ copy_(original_input, ...)
```

그래서 기존 구현은 안전성 분석이 제자리 연산 복원을 허용할 수 있는 경우에도 그 단계에 도달하기 전에 수익성 판정 조건을 통과하지 못한 후보를 제외합니다. 결과적으로 전체 버퍼 복제가 남습니다.

이 제한은 [기존 `should_reinplace_scatter()`의 검사](https://github.com/pytorch/pytorch/blob/e4b9ce80a342100d49acab38bb4bf7ba0bf87118/torch/_inductor/fx_passes/reinplace.py#L239-L247)에 그대로 드러납니다. `node.users` 가운데 원래 입력으로 직접 복사하는 `copy_`가 있는지만 확인합니다. 중간의 `index_put`을 따라가지는 않습니다.

## 6. 첫 해결책이 충분하지 않았던 이유

처음 생각하기 쉬운 해결책은 각 연산의 결과를 사용하는 다음 연산이 하나뿐인 경로를 따라가는 것입니다.

```text
scatter의 결과를 사용하는 연산이 정확히 하나인가?
  └─ index_put인가?
       └─ 원래 입력으로 다시 복사되는가?
```

하지만 실제 그래프에서는 두 연산이 `generalized_scatter`의 결과를 사용합니다.

```text
scatter ─▶ index_select
   └─────▶ index_put의 갱신 대상 텐서
```

갱신한 `field`에서 `values`를 계산하기 때문에 `scatter.users`의 개수는 하나가 아닙니다. “결과를 사용하는 연산이 여러 개다”라는 이유만으로 최적화 후보에서 제외하면 실제 문제를 해결하지 못합니다.

여기서 경로 인식과 안전성 검증을 구분해야 합니다.

- **경로 인식:** 앞선 연산의 결과를 받아 일부 위치의 값을 갱신하고 그렇게 갱신한 결과를 원래 입력에 다시 복사하는가?
- **안전성 검증:** 값을 읽는 다른 연산이 있어도 실행 순서와 별칭 관점에서 올바른가?

경로 인식 단계가 안전성 전체를 다시 증명하려고 지나치게 강한 조건을 붙일 필요는 없습니다. 안전성은 이미 별도의 별칭·생존성 검사가 담당합니다.

## 7. 개선된 알고리즘: 갱신 대상 텐서로 이어지는 연결만 따라가기

개선된 아이디어는 `index_put`의 특정 피연산자만 보는 것입니다.

```python
result = index_put(base, indices, values)
```

여기서 첫 번째 인자인 `base`가 실제 갱신 대상입니다. 그래프 탐색은 결과를 사용하는 모든 연산을 따라가지 않고 다음 연산이 그 결과를 갱신 대상 텐서로 받는 연결만 따라갑니다. 아래 코드의 `user`는 현재 결과를 사용하는 연산을 뜻하며 `user.args[0] is current`가 이 연결을 확인하는 조건입니다.

```python
user.target in {index_put, unsafe_index_put}
and user.args[0] is current
```

개념적인 알고리즘은 다음과 같습니다.

```python
pending = [scatter]
seen = set()

while pending:
    current = pending.pop()

    if current in seen:
        continue
    seen.add(current)

    if copied_back_to_original_input(current):
        return True

    for user in current.users:
        if is_allowed_indexed_update(user) and user.args[0] is current:
            pending.append(user)

return False
```

여기서는 그래프 이론의 **방향 그래프(directed graph)**와 **도달 가능성(reachability)** 개념을 사용합니다. 연산을 노드로 보고, 다른 연산이 한 연산의 결과를 입력으로 받는 관계를 방향이 있는 연결로 봅니다. 위 코드가 묻는 것은 허용된 연결을 따라 `scatter`에서 출발했을 때, 원래 입력으로 다시 복사하는 연산에 도달할 수 있느냐는 것입니다. 가장 짧은 경로를 찾을 필요는 없고, 그런 경로가 하나라도 있는지만 확인합니다.

이를 확인하는 알고리즘은 **깊이 우선 탐색(depth-first search, DFS)**입니다. `pending.pop()`으로 마지막에 넣은 노드부터 꺼내므로 한 경로를 따라 먼저 탐색합니다. `seen`은 여러 경로가 같은 노드로 합쳐질 때 이미 방문한 노드를 다시 처리하지 않게 합니다. 다만 일반적인 그래프 탐색과 달리, 다음 연산이 허용 목록에 있고 현재 결과를 갱신 대상 텐서로 받는 연결만 따라갑니다.

이 탐색은 `index_select`처럼 값을 읽는 연산을 따라가지 않습니다. 그렇다고 값을 읽는 연산이 없다고 가정하지도 않습니다. 단지 우리가 찾는 **원래 입력으로 다시 복사하는 경로**가 아니므로 경로 탐색 대상에서 제외합니다. 경로의 존재는 그래프 탐색으로 확인하지만, 값을 읽는 연산 때문에 제자리 변경이 위험한지는 뒤의 별칭·생존성 분석이 판단합니다.

PR에서는 이 탐색을 [`_scatter_copied_back_through_indexed_updates()`](https://github.com/pytorch/pytorch/blob/c0ed45b28595c6e30e37a211c14cc63228227ef9/torch/_inductor/fx_passes/reinplace.py#L240-L258)로 구현합니다. `pending`과 `seen`, 그리고 `user.args[0] is current` 조건을 위 의사 코드와 나란히 보면 대응 관계가 분명합니다. [바로 위의 허용 목록과 보조 함수](https://github.com/pytorch/pytorch/blob/c0ed45b28595c6e30e37a211c14cc63228227ef9/torch/_inductor/fx_passes/reinplace.py#L222-L237)에서는 탐색을 허용하는 두 연산을 정의합니다. 원래 입력으로 다시 복사하는지를 판정하는 코드도 이곳에 있습니다.

이 구조에는 몇 가지 전형적인 컴파일러 설계 원칙이 들어 있습니다.

- 연산 이름만 보지 않고 피연산자의 역할까지 검사합니다.
- 인식 가능한 연산을 허용 목록으로 제한합니다.
- 처리할 노드 목록과 `seen` 집합으로 그래프를 탐색합니다.
- 후보 발견과 의미 보존 증명을 서로 다른 단계로 둡니다.

## 8. 안전성과 수익성을 분리해야 하는 이유

두 판단을 한 함수에 섞으면 흔히 두 종류의 문제가 생깁니다.

첫째, 너무 보수적이 되어 가능한 최적화를 놓칩니다.

```text
값을 읽는 추가 연산이 있음
→ 위험할지도 모름
→ 무조건 거절
```

이슈 #195285의 첫 접근이 이 문제를 보여 줍니다. 값을 읽는 추가 연산이 있어도 제자리 연산으로 안전하게 바꿀 수 있는 경우가 있습니다.

둘째, 반대로 너무 공격적이 되어 잘못된 코드를 만들 수 있습니다.

```text
원래 입력으로 돌아가는 경로를 찾음
→ 이득이 있음
→ 즉시 제자리 연산으로 변환
```

중간에 이전 값을 읽는 별칭이 있다면 결과가 달라질 수 있습니다.

두 판단을 분리한 구조는 다음과 같습니다.

```text
패턴/수익성 검사
  "이 후보는 복제를 없앨 가능성이 있는가?"
              │
              ▼
별칭·생존성 안전성 검사
  "원본을 일찍 바꿔도 누구도 차이를 관찰하지 않는가?"
              │
              ▼
제자리 연산 복원
```

수익성 검사는 최적화를 할 가치가 있는 후보를 찾고 안전성 검사는 그 변환을 허가합니다.

## 9. 왜 제자리 연산을 항상 복원하지 않을까?

전체 텐서 복제가 보이면 없애는 것이 언제나 옳아 보입니다. 하지만 컴파일러의 비용 모델에서는 다음 두 실행 전략이 경쟁합니다.

### 함수형 전략

```text
값을 만드는 연산 ─▶ slice_scatter ─▶ 값을 사용하는 연산
```

- 중간 연산과 융합할 가능성이 있습니다.
- 외부 상태를 바꾸는 동작이 적어 실행 순서를 자유롭게 정할 수 있습니다.
- 경우에 따라 중간 텐서를 실제로 만들지 않을 수 있습니다.

### 제자리 변경 전략

```text
실제 메모리에 만들어진 버퍼 ─▶ 뷰 ─▶ copy_
```

- 전체 텐서 복제를 피할 수 있습니다.
- 동일 버퍼를 재사용합니다.
- 하지만 버퍼를 실제 메모리에 만드는 것과 실행 순서를 강제할 수 있습니다.
- 연산 융합 기회를 줄일 수 있습니다.

Inductor는 입력과 출력을 어차피 실제 메모리에 저장하거나 결과를 결국 원래 입력으로 다시 복사해야 하는 상황을 제자리 변경이 수익성 있는 신호로 사용합니다. 완벽한 성능 예측 모델이라기보다 제한된 정보로 내리는 보수적인 경험적 판단 기준입니다.

이 판단 순서는 [PR의 `should_reinplace_scatter()`](https://github.com/pytorch/pytorch/blob/c0ed45b28595c6e30e37a211c14cc63228227ef9/torch/_inductor/fx_passes/reinplace.py#L261-L289)에서 읽을 수 있습니다. Inductor가 이미 실제 메모리에 버퍼를 만드는 경우와 원래 입력으로 다시 복사하는 경로가 있는 경우에는 `True`를 반환합니다. 나머지는 연산 융합이 유리할 것으로 보고 함수형 표현을 유지합니다.

## 10. 테스트는 결과와 최적화 구조를 함께 검증해야 한다

컴파일러 최적화 테스트에서는 결과가 올바른지와 의도한 최적화가 실제로 적용됐는지를 모두 확인해야 합니다.

### 의미가 보존되는가?

```python
expected = field.clone()
actual = field.clone()

composed(expected, delta, indices, gain)
compiled(actual, delta, indices, gain)

torch.testing.assert_close(actual, expected, rtol=0, atol=0)
```

이 검사는 원래 함수를 그대로 실행한 결과(eager)와 컴파일한 함수를 실행한 결과(compiled)가 같은지 확인합니다. 하지만 결과가 같다는 것만으로는 불필요한 복제가 제거됐는지 알 수 없습니다.

### 의도한 최적화가 실제로 일어났는가?

```python
self.assertNotIn("aten.slice_scatter.default", post_grad_graph)
```

최종 값이 맞더라도 선택한 위치에 값을 반영하는 함수형 연산(scatter)이 비효율적으로 남을 수 있습니다. 그래서 컴파일러가 프로그램을 분석하고 변환할 때 사용하는 중간 표현(Intermediate Representation, IR)도 확인합니다. 단순히 `aten.clone` 문자열 개수만 검사하면 다른 최적화 단계에서 복제 연산을 변형하거나 제거했을 때 테스트가 잘못 통과할 수 있습니다. 문제를 대표하는 중간 표현인 `slice_scatter`가 없는지도 함께 검사하면 회귀 테스트의 의도를 더 정확하게 표현할 수 있습니다.

최적화가 적용되면 안 되는 경우도 검증해야 합니다.

- 결과를 원래 입력이 아닌 다른 입력으로 다시 복사하는 경우
- 제자리 변경 이후 이전 값을 읽는 별칭이 있는 경우
- 텐서 메타데이터(metadata)가 맞지 않는 경우
- 허용되지 않은 연산이 중간 경로에 있는 경우

최적화 테스트는 "언제 적용되는가"뿐 아니라 "언제 적용되면 안 되는가"를 정의합니다.

PR의 [결과 비교와 복제·`slice_scatter` 검사](https://github.com/pytorch/pytorch/blob/c0ed45b28595c6e30e37a211c14cc63228227ef9/test/inductor/test_inplacing_pass.py#L539-L555)가 두 종류의 검증을 한곳에서 수행합니다. 별도의 [수익성 판정 테스트](https://github.com/pytorch/pytorch/blob/c0ed45b28595c6e30e37a211c14cc63228227ef9/test/inductor/test_inplacing_pass.py#L583-L598)는 값을 읽는 추가 연산이 있어도 후보로 인정하는 경우와 다른 입력으로 다시 복사하면 거절하는 경우를 확인합니다. 이 테스트가 위에서 나열한 모든 안전성 조건을 검증하는 것은 아닙니다.

## 11. 직접 실험해 보기

다음처럼 작은 함수에서 시작할 수 있습니다.

```python
import torch


def composed(field, delta, indices, gain):
    field[1:-1].add_(delta)
    values = torch.index_select(field, 0, indices) * gain
    field.index_copy_(0, indices, values)


field = torch.linspace(0.25, 1.25, 51_520, dtype=torch.float64)
delta = torch.linspace(-0.01, 0.01, 51_518, dtype=torch.float64)
indices = torch.arange(0, 198, 2, dtype=torch.int64)
gain = torch.linspace(0.99, 1.01, 99, dtype=torch.float64)

expected = field.clone()
actual = field.clone()

compiled = torch.compile(composed, fullgraph=True)

for _ in range(5):
    composed(expected, delta, indices, gain)
    compiled(actual, delta, indices, gain)

torch.testing.assert_close(actual, expected, rtol=0, atol=0)
```

위 예제를 `example.py`로 저장한 뒤 macOS나 Linux 터미널에서 다음 명령으로 실행하면 그래프 로그를 확인할 수 있습니다.

```bash
TORCH_LOGS="aot_graphs,post_grad_graphs,output_code" \
python example.py 2> torch-compile.log
```

`TORCH_LOGS`는 PyTorch에서 출력할 로그를 선택하는 환경 변수입니다. `2>`는 PyTorch가 표준 오류로 출력하는 로그를 `torch-compile.log` 파일에 저장합니다. 같은 이름의 파일이 있으면 덮어쓰므로 이전 결과와 비교하려면 파일명을 다르게 지정합니다.

각 설정은 서로 다른 처리 단계의 출력을 보여 줍니다.

- `aot_graphs`: 함수형 변환을 거쳐 Inductor에 전달하는 연산 그래프
- `post_grad_graphs`: Inductor의 그래프 최적화 이후 남은 연산
- `output_code`: Inductor가 최종 생성한 실행 코드

`torch-compile.log`를 편집기로 열고 `slice_scatter`, `aten.clone`, `index_put`, `copy_`를 검색해 보면 됩니다.

로그 파일에는 최적화 전후의 출력이 함께 들어 있습니다. `aten.clone`이나 `slice_scatter`가 검색된다는 사실만으로 최적화에 실패했다고 판단하면 안 됩니다. 로그 줄의 `[__aot_graphs]`, `[__post_grad_graphs]`, `[__output_code]` 표식을 확인해 어느 단계의 출력인지 구분합니다. 특히 `post_grad_graphs`에서 복제나 함수형 갱신 연산이 남는지 보고 `output_code`에서는 Inductor가 최종 실행 코드를 어떻게 만들었는지 확인하면 됩니다.

PyTorch는 주로 컴파일할 때 이 로그를 출력합니다. 이미 만들어진 실행 코드를 재사용하는 호출에서는 같은 로그가 반복되지 않을 수 있습니다. 각 로그 설정의 의미는 [PyTorch의 로그 등록 코드](https://github.com/pytorch/pytorch/blob/e4b9ce80a342100d49acab38bb4bf7ba0bf87118/torch/_logging/_registrations.py#L106-L129)에서도 확인할 수 있습니다.

이렇게 처리 단계를 구분한 뒤 다음 항목을 비교해 보면 좋습니다.

- 함수형 변환 이후 제자리 변경이 어떤 연산으로 표현되는가?
- `slice_scatter` 또는 복제 연산이 남는가?
- `index_copy`가 어느 시점에 `index_put`으로 분해되는가?
- 지정한 위치에 쓸 값이 `field`에 의존할 때 그 값을 사용하는 연산 사이의 연결 관계가 어떻게 달라지는가?
- 지정한 위치가 처음 잘라낸 구간과 겹칠 때도 즉시 실행과 컴파일된 실행의 결과가 정확히 같은가?

성능을 측정할 때는 다음을 분리해야 합니다.

- 컴파일 시간과 실행 시간
- 첫 호출과 준비 실행(warm-up) 이후 호출
- 연산 함수(커널) 자체 성능과 프로그램 전체 성능
- 메모리 할당 감소와 실제 경과 시간 단축
- 단일 스레드와 다중 스레드 환경

이슈 #195285에서 보고된 약 2.48배의 차이는 작은 연산 함수(커널)를 따로 측정한 결과입니다. 이를 프로그램 전체가 같은 비율로 빨라진다는 뜻으로 해석하면 안 됩니다.

## 맺음말

함수형 변환과 제자리 연산 복원은 서로 반대 방향으로 움직입니다.

```text
사용자가 작성한 제자리 변경
        │
        ▼
함수형 변환
분석하기 쉬운 함수형 그래프
        │
        ▼
별칭·생존성·수익성 분석
        │
        ▼
제자리 연산 복원
복사 없이 버퍼를 재사용하는 실행
```

함수형 변환은 프로그램의 상태 변화를 명시적인 데이터 흐름으로 바꾸어 컴파일러가 이해할 수 있게 합니다. 제자리 연산 복원은 그렇게 얻은 정보로 의미 보존을 증명한 뒤 함수형 표현이 초래할 수 있는 복사 비용을 회수합니다.

컴파일러는 먼저 제자리 변경을 제거해 프로그램을 이해한 뒤 **아무도 차이를 관찰할 수 없으며 실제로 이득인 곳에서만** 제자리 변경을 복원합니다. 이슈 #195285와 PR #195292가 보여 주는 핵심은 바로 이 조정 과정입니다.

> 작성 시점인 2026년 9월 3일 기준 PR #195292는 초안 상태입니다. 세부 구현과 테스트는 리뷰 과정에서 변경될 수 있습니다.
