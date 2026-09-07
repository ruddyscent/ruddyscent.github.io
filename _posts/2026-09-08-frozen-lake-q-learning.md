---
layout: post
title: "FrozenLake: Q 학습으로 얼음 호수 건너기"
subtitle: "목표에 도착한 경험을 표에 조금씩 쌓습니다"
tags: [python, reinforcement-learning, gymnasium, q-learning]
cover-img: /assets/img/develop.jpeg
thumbnail-img: /assets/img/parchment-map.webp
share-img: /assets/img/develop.jpeg
author: 전경원
---

Gymnasium이 제공하는 FrozenLake 환경의 풀이를 구현했습니다. 코드는 [gymnasium-playbook 저장소](https://github.com/ruddyscent/gymnasium-playbook)에 정리했습니다. 출발점부터 어느 방향으로 움직일지 직접 정해 주지 않고 여러 번 시도한 결과를 바탕으로 길을 찾게 했습니다.

이때 사용한 방법이 **Q 학습(Q-learning)**입니다. 현재 위치에서 각 행동이 얼마나 유리한지를 숫자로 기록하고 움직인 결과를 보고 그 숫자를 고칩니다. 이번 환경에서는 칸과 행동의 수가 적어서 신경망 없이 표 하나로 구현할 수 있습니다.

## 어떤 호수를 건널까?

![FrozenLake의 4×4 지도. 왼쪽 위에 출발점, 오른쪽 아래에 목표가 있고 중간에 네 개의 구멍이 있습니다.](/assets/img/frozen-lake/frozen-lake-window.png)

*FrozenLake 실행 화면. 파란 구멍을 피해 오른쪽 아래의 선물까지 이동합니다.*

지도는 가로와 세로가 각각 4칸입니다. 왼쪽 위에서 출발해 상하좌우로 움직이며 구멍에 빠지면 실패하고 오른쪽 아래 목표에 도착하면 성공합니다. 출발해서 성공하거나 실패할 때까지의 한 번의 시도를 **에피소드(episode)**라고 부릅니다. 이번 구현에서는 100번 움직여도 끝나지 않으면 그 시도를 중단합니다.

FrozenLake는 기본 설정에서 얼음 위를 미끄러질 수 있습니다. 오른쪽을 선택해도 다른 방향으로 이동할 수 있다는 뜻입니다. 이번에는 학습 과정을 살펴보기 쉽도록 `is_slippery=False`로 설정해 미끄러짐을 껐습니다. 따라서 선택한 방향으로 이동하며 지도 밖으로 나가려 하면 제자리에 머뭅니다. [환경 설정 코드](https://github.com/ruddyscent/gymnasium-playbook/blob/4a142954fb52ef8e052257407d85a1d7884d9af3/environments/toy_text/frozen_lake/q_learning.py#L57-L63)에서도 이 조건을 확인할 수 있습니다.

목표에 도착하면 **보상(reward)** 1을 받고 나머지 이동에서는 모두 0을 받습니다. 구멍에 빠져도 음수 보상은 없습니다. 이 보상 규칙과 지도 구성은 [Gymnasium의 FrozenLake 기본 규칙](https://gymnasium.farama.org/environments/toy_text/frozen_lake/)을 그대로 사용합니다.

목표에 도착하기 전까지는 보상만으로 어떤 이동이 도움이 되는지 바로 알기 어렵습니다. 목표 바로 앞까지 잘 움직여도 그동안 받는 보상은 계속 0입니다.

## 64개의 숫자로 행동을 고르기

프로그램에 주어지는 **상태(state)**는 현재 칸의 번호입니다. 화면의 그림을 읽는 대신, 왼쪽 위부터 차례대로 매긴 0~15 중 하나를 받습니다. 각 칸에서 선택할 **행동(action)**은 왼쪽, 아래, 오른쪽, 위의 네 가지입니다.

그래서 필요한 표의 크기는 `16 × 4`, 총 64개 숫자입니다. 이를 **Q 표(Q-table)**라고 부릅니다. 한 행은 현재 칸을, 네 열은 그 칸에서 선택할 방향을 나타냅니다. 각 숫자는 해당 방향으로 움직인 뒤 앞으로 얻을 보상의 추정값입니다.

처음에는 모든 값을 0으로 채웁니다. 이후 이동할 때마다 현재 칸에서 선택한 방향의 값 하나를 고칩니다. 학습한 표를 사용할 때는 현재 칸의 네 값을 비교해 가장 큰 값을 고르고 그 방향으로 움직입니다. 이렇게 상태에 따라 행동을 선택하는 규칙을 **정책(policy)**이라고 합니다.[[1]](#ref-sutton-barto)[[2]](#ref-coursera-fundamentals) [표의 생성과 학습 반복 코드](https://github.com/ruddyscent/gymnasium-playbook/blob/4a142954fb52ef8e052257407d85a1d7884d9af3/environments/toy_text/frozen_lake/q_learning.py#L103-L143)는 이 과정을 그대로 담고 있습니다.

## 성공한 경험은 어떻게 앞선 선택에 반영될까?

목표에 도착하면 바로 직전에 선택한 행동이 좋은 선택이었다는 것을 알 수 있습니다. Q 학습은 여기서 한 걸음 더 나아갑니다. 지금 보상을 받지 못하더라도, 이동한 칸에서 앞으로 좋은 결과를 기대할 수 있다면 방금 선택한 행동의 값도 높입니다.

예를 들어 처음으로 목표에 도착했다고 해 보겠습니다. 목표 바로 앞에서 선택한 행동의 값은 아직 0이고 받은 보상은 1입니다. 이 구현의 **학습률(learning rate)**은 기본값이 0.1이므로 기존 값에서 목표값까지 차이의 10%만 반영합니다. 값은 한 번에 1이 되지 않고 0.1로 올라갑니다.

이후 다시 그 앞 칸에서 같은 칸으로 이동하면, 이번에는 당장 받은 보상이 0이어도 다음 칸에 0.1이라는 값이 남아 있습니다. **할인율(discount factor)** 0.99를 곱해 미래 보상을 조금 낮게 평가하면 갱신할 목표값은 0.099가 됩니다. 현재 값이 0이라면 학습률을 적용한 새 값은 0.0099입니다.

이런 갱신을 반복하면서 목표 근처의 좋은 행동부터 출발점 쪽으로 값이 전해집니다. 한 번 성공했다고 지나온 경로 전체를 한꺼번에 바꾸는 것은 아닙니다. 같은 칸들을 다시 방문하며 조금씩 반영합니다.[[1]](#ref-sutton-barto)[[3]](#ref-coursera-sample)[[4]](#ref-udacity) 할인율이 1보다 작기 때문에 같은 보상이라면 더 적게 움직여 받는 쪽을 높게 평가합니다.

구현의 핵심을 설명용 코드로 줄이면 다음과 같습니다. 원래 함수는 [Q 값 갱신 코드](https://github.com/ruddyscent/gymnasium-playbook/blob/4a142954fb52ef8e052257407d85a1d7884d9af3/environments/toy_text/frozen_lake/q_learning.py#L77-L100)에서 볼 수 있습니다.

```python
if terminated:
    target = reward
else:
    target = reward + discount * q_table[next_state].max()

q_table[state, action] += learning_rate * (target - q_table[state, action])
```

구멍에 빠지거나 목표에 도착해 실제로 끝났다면 `terminated`가 참이므로 이후의 보상을 더하지 않습니다. 반면 100번 이동해서 중단되는 `truncated`는 수집을 끊는 조건입니다. 이번 구현에서는 남은 이동 횟수를 상태에 넣지 않으므로 시간 제한으로 중단되더라도 다음 칸의 값을 갱신에 사용합니다.

## 아는 길을 가면서도 다른 길을 시도하기

좋아 보이는 방향만 계속 선택하면 아직 가 보지 않은 길을 놓칠 수 있습니다. 특히 처음에는 Q 표가 전부 0이어서 어느 쪽이 나은지 구분조차 안 됩니다.

학습 중에는 일정 확률로 방향을 무작위로 고릅니다. 이를 **탐색(exploration)**이라고 합니다. 나머지 경우에는 현재 표에서 가장 값이 큰 방향을 선택합니다. 무작위로 고를 확률을 엡실론(epsilon)이라고 하며 이 선택 방식을 **엡실론 탐욕 정책(epsilon-greedy policy)**이라고 부릅니다.[[1]](#ref-sutton-barto)[[2]](#ref-coursera-fundamentals)

이번 구현은 처음에는 무작위로 움직이다가 그 비율을 점차 줄여 마지막에는 최소 5%를 유지합니다. 가장 큰 값이 여러 개일 때도 그중 하나를 무작위로 선택합니다. 모든 값이 0인 초기에 늘 첫 번째 방향인 왼쪽만 고르는 편향을 피하기 위해서입니다. [행동 선택 함수](https://github.com/ruddyscent/gymnasium-playbook/blob/4a142954fb52ef8e052257407d85a1d7884d9af3/environments/toy_text/frozen_lake/q_learning.py#L66-L74)와 [탐색 확률 계산](https://github.com/ruddyscent/gymnasium-playbook/blob/4a142954fb52ef8e052257407d85a1d7884d9af3/environments/toy_text/frozen_lake/q_learning.py#L113-L123)에 해당 내용이 들어 있습니다.

따라서 길을 충분히 익힌 뒤에도 학습 중에는 구멍에 빠질 수 있습니다. 배운 결과를 확인할 때는 무작위 탐색을 끄고 표도 고정합니다. [평가 코드](https://github.com/ruddyscent/gymnasium-playbook/blob/4a142954fb52ef8e052257407d85a1d7884d9af3/environments/toy_text/frozen_lake/q_learning.py#L146-L182)는 매번 가장 값이 큰 방향을 선택하고 값이 같으면 일정한 순서로 고릅니다.

## 학습 후에는 얼마나 잘 건널까?

난수의 시작값인 **시드(seed)**를 0, 1, 2로 바꿔 각각 10,000회 학습하고 학습한 정책과 무작위 정책을 각각 1,000회 평가했습니다. 평가 시드는 순서대로 10000, 10001, 10002를 사용했습니다. 아래는 커밋 [4a14295](https://github.com/ruddyscent/gymnasium-playbook/commit/4a142954fb52ef8e052257407d85a1d7884d9af3)를 2026년 9월 7일 macOS ARM64에서 Python 3.13.15, Gymnasium 1.3.0, NumPy 2.5.3으로 실행한 결과입니다.

| 학습 시드 | 학습한 정책의 성공률 | 성공까지 평균 이동 횟수 | 무작위 정책의 성공률 |
| --- | --- | --- | --- |
| 0 | 100% | 6회 | 1.5% |
| 1 | 100% | 6회 | 1.2% |
| 2 | 100% | 6회 | 1.1% |

세 경우 모두 학습한 정책은 구멍을 피해 6번 이동해 목표에 도착했습니다. 출발점에서 목표까지는 오른쪽으로 3칸, 아래로 3칸만큼 이동해야 하므로 6회는 가능한 최소 이동 횟수입니다.

다만 이 결과의 범위는 미끄러지지 않는 고정 지도입니다. 출발점도 같고 평가 중에는 행동도 고정되어 같은 경로를 반복합니다. 여기서의 100%는 새로운 지도나 미끄러운 얼음에서도 성공한다는 의미는 아닙니다.

## 직접 실행하고 영상으로 보기

이 글의 구현 설명과 실행 명령은 커밋 [4a14295](https://github.com/ruddyscent/gymnasium-playbook/commit/4a142954fb52ef8e052257407d85a1d7884d9af3)를 기준으로 합니다. 같은 버전으로 실행하려면 저장소를 별도 디렉터리에 복제한 뒤 해당 커밋을 체크아웃합니다.

```sh
git clone https://github.com/ruddyscent/gymnasium-playbook.git gymnasium-playbook-frozen-lake
cd gymnasium-playbook-frozen-lake
git checkout --detach 4a142954fb52ef8e052257407d85a1d7884d9af3
```

[해당 커밋의 환경 준비 안내](https://github.com/ruddyscent/gymnasium-playbook/blob/4a142954fb52ef8e052257407d85a1d7884d9af3/README.md#environment-setup)에 따라 uv와 Python을 준비한 뒤, 같은 디렉터리에서 다음 명령을 실행합니다.

```sh
uv sync --locked
uv run --locked python -m environments.toy_text.frozen_lake train --seed 0
uv run --locked python -m environments.toy_text.frozen_lake watch --q-table runs/frozen_lake/seed-0/q_table.npy
```

학습 결과는 `runs/frozen_lake/seed-0/q_table.npy`에 저장됩니다. `watch`는 저장한 표로 행동을 재생합니다. 창을 띄울 수 있는 환경에서 실행하며 기본 속도는 초당 두 번 이동입니다. 창을 닫거나 Escape를 누르면 종료됩니다.

같은 조건에서 성공률을 비교하려면 다음 명령을 실행합니다. 결과는 `runs/frozen_lake/benchmark/summary.csv`와 `summary.json`에 남습니다. 자세한 실행 옵션은 [FrozenLake 실행 안내](https://github.com/ruddyscent/gymnasium-playbook/blob/4a142954fb52ef8e052257407d85a1d7884d9af3/environments/toy_text/frozen_lake/README.md)에 정리되어 있습니다.

```sh
uv run --locked python -m environments.toy_text.frozen_lake benchmark --seeds 0 1 2 --episodes 10000 --eval-episodes 1000 --eval-seed 10000
```

실행 영상은 아래 YouTube 링크에서도 볼 수 있습니다.

<iframe
  src="https://www.youtube-nocookie.com/embed/3dsN-a2jc0Y?list=PLL-Q-OgP1MtU&index=1"
  title="FrozenLake Q 학습 실행 영상"
  width="960"
  height="540"
  style="width: 100%; aspect-ratio: 16 / 9; border: 0;"
  loading="lazy"
  allow="accelerometer; autoplay; clipboard-write; encrypted-media; gyroscope; picture-in-picture; web-share"
  referrerpolicy="strict-origin-when-cross-origin"
  allowfullscreen>
</iframe>

[YouTube에서 FrozenLake 영상과 재생목록 열기](https://www.youtube.com/watch?v=3dsN-a2jc0Y&list=PLL-Q-OgP1MtU&index=1)

## 함께 보기

1. <a id="ref-sutton-barto"></a>Richard S. Sutton, Andrew G. Barto. [*Reinforcement Learning: An Introduction*, 2판](https://mitpress.mit.edu/9780262039246/reinforcement-learning/). MIT Press, 2018. 탐색과 엡실론 탐욕 선택은 §2.2 “Action-value Methods”(27쪽), 할인된 보상과 에피소드는 §3.3 “Returns and Episodes”(54쪽), 정책과 가치 함수는 §3.5 “Policies and Value Functions”(58쪽), Q 학습의 갱신식은 §6.5 “Q-learning: Off-policy TD Control”(131쪽)을 참고합니다. 쪽수는 인쇄본 기준이며 [공식 목차](https://mitp-content-server.mit.edu/books/content/sectbyfn/books_pres_0/10094/Toc.pdf?dl=1)에서도 해당 절을 찾을 수 있습니다.
2. <a id="ref-coursera-fundamentals"></a>University of Alberta · Coursera. [*Reinforcement Learning Specialization*](https://www.coursera.org/specializations/reinforcement-learning)의 1번째 강좌, [*Fundamentals of Reinforcement Learning*](https://www.coursera.org/learn/fundamentals-of-reinforcement-learning/). “An Introduction to Sequential Decision-Making” 단원의 「Learning Action Values」, 「What is the trade-off?」와 탐색 실습에서 행동의 가치 추정과 엡실론 탐욕 선택을 다룹니다. “Value Functions & Bellman Equations” 단원의 「Specifying Policies」, 「Value Functions」는 본문의 정책과 Q 값 설명에 해당합니다.
3. <a id="ref-coursera-sample"></a>University of Alberta · Coursera. [*Reinforcement Learning Specialization*](https://www.coursera.org/specializations/reinforcement-learning)의 2번째 강좌, [*Sample-based Learning Methods*](https://www.coursera.org/learn/sample-based-learning-methods). “Temporal Difference Learning Methods for Prediction” 단원의 「What is Temporal Difference (TD) learning?」에서 다음 상태의 추정값을 이용하는 갱신을 설명합니다. “Temporal Difference Learning Methods for Control” 단원의 「What is Q-learning?」, 「How is Q-learning off-policy?」는 탐색하면서도 다음 상태의 최댓값을 목표로 학습하는 원리를 설명합니다.
4. <a id="ref-udacity"></a>Udacity. [*Deep Reinforcement Learning*, Course 1: Introduction to Deep Reinforcement Learning](https://www.udacity.com/course/deep-reinforcement-learning-nanodegree--nd893). 「The RL Framework: The Problem」과 「The RL Framework: The Solution」에서 문제와 해법의 기본 틀을, 「Temporal-Difference Methods」에서 Q 학습을 다룹니다. [공식 강의 실습 저장소](https://github.com/udacity/deep-reinforcement-learning/tree/master/temporal-difference)에는 Q 학습을 직접 구현하는 연습도 있습니다.
