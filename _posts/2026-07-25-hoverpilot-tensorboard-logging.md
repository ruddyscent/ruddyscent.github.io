---
layout: post
title: HoverPilot - PPO 학습에 TensorBoard 로깅 붙이기
subtitle: 보상의 변화를 시각적으로 분석해보자
tags:
- hoverpilot
- reinforcement-learning
- ppo
- tensorboard
- python
cover-img: /assets/img/develop.jpeg
thumbnail-img: /assets/img/toy-plane.webp
share-img: /assets/img/develop.jpeg
author: 전경원
---

[이전 글](/2026-05-02-hoverpilot-ppo-first-try/)에서는 PPO 학습 루프를 돌렸을 때 정책이 hover 대신 꼬리부터 착지하는 방향으로 수렴하는 것을 보고 reward 구조 때문일 거라는 가설을 세웠습니다. 그런데 이 가설을 확인하려면 먼저 학습 도중 무슨 일이 벌어지는지 볼 수 있어야 했습니다.

강화학습을 돌릴 때 가장 답답한 순간은 학습이 되고 있는지 확신할 수 없을 때입니다. 콘솔에 찍히는 평균 보상 숫자 몇 줄만으로는 정책이 실제로 개선되는지, 특정 액션으로 쏠리는지, 손실값이 불안정한지, 환경 종료 이유가 무엇인지 파악하기 어렵습니다.

이번 [`1e4c7d0`](https://github.com/ruddyscent/hover-pilot/commit/1e4c7d0) 커밋에서는 HoverPilot의 PPO 학습 루프에 [TensorBoard](https://www.tensorflow.org/tensorboard) 로깅을 추가해 이 문제를 풀었습니다. `episode_reward` 하나만 남기는 데 그치지 않고 보상 흐름, 액션 분포, PPO 업데이트 지표, 평가 결과, 종료 원인까지 한 번에 볼 수 있게 만들었습니다. PPO 하이퍼파라미터도 CLI에서 직접 조정할 수 있도록 열어 두어 학습을 실행하는 데서 그치지 않고 관찰하며 튜닝하는 워크플로우로 발전시켰습니다.

## 이번 커밋에서 바뀐 것

핵심 변경점은 `src/hoverpilot/rl/ppo.py`에 `torch.utils.tensorboard.SummaryWriter`를 연결한 것입니다. 학습이 시작되면 writer를 생성하고 학습 중에는 주요 지표를 scalar로 기록하며 종료 시에는 안전하게 `flush()`와 `close()`를 호출합니다. 실행 설정도 `run/config` 텍스트로 남겨 나중에 어떤 설정으로 실험했는지 다시 확인할 수 있게 했습니다.

이번 커밋에서 TensorBoard에 기록하는 대표 지표는 이렇습니다.

- 학습 에피소드 지표
  - `train/episode_reward`
  - `train/episode_length`
  - `train/avg_reward`
  - `train/avg_length`
- 롤아웃 요약 지표
  - `train/reward_mean`
  - `train/reward_min`
  - `train/reward_max`
  - `train/done_rate`
  - `train/return_mean`
  - `train/return_std`
  - `train/advantage_mean`
  - `train/advantage_std`
- 액션 통계
  - `train/action/aileron_mean`
  - `train/action/elevator_mean`
  - `train/action/throttle_mean`
  - `train/action/rudder_mean`
  - 각 액션의 `std`
- PPO 업데이트 지표
  - `train/policy_loss`
  - `train/value_loss`
  - `train/entropy`
  - `train/ratio`
- 종료 원인 분석
  - `train/termination/<reason>`
  - `train/termination_rate/<reason>`
- 평가 지표
  - `eval/avg_reward`
  - `eval/avg_length`

이 구성이 중요한 이유는 단순히 "보상이 올랐다/내렸다"를 보는 수준을 넘어 학습 상태를 여러 각도에서 진단할 수 있기 때문입니다.

## 왜 TensorBoard가 중요한가

TensorBoard의 가장 큰 장점은 학습 과정을 시계열로 구조화해 보여준다는 점입니다. 강화학습에서는 결과가 불안정하고 분산도 크기 때문에 텍스트 로그만으로 패턴을 읽기 어렵습니다. TensorBoard에서는 지표의 추세를 시각적으로 확인할 수 있어 다음 질문에 빠르게 답할 수 있습니다.

정책이 실제로 좋아지고 있는지 살펴봅니다. `train/episode_reward`, `train/avg_reward`, `eval/avg_reward`를 보면 학습 보상과 평가 보상이 함께 개선되는지 확인할 수 있습니다. 학습 보상만 오르고 평가 보상이 정체된다면 과적합이나 불안정한 정책 업데이트를 의심할 수 있습니다.

PPO 업데이트가 안정적인지도 확인합니다. `train/policy_loss`, `train/value_loss`, `train/entropy`, `train/ratio`는 PPO의 내부 상태를 보여줍니다. 예를 들어 entropy가 너무 빨리 떨어지면 탐험이 급격히 줄어든다는 신호일 수 있고, ratio가 지나치게 흔들리면 update 강도가 과한 것일 수 있습니다.

셋째, 정책이 특정 조작에 쏠리고 있는가?
HoverPilot는 aileron, elevator, throttle, rudder 네 개 축을 제어합니다. `train/action/*_mean`, `train/action/*_std`를 보면 throttle이 지나치게 높게 유지되는지, rudder가 거의 죽어 있는지, 특정 축의 분산이 비정상적으로 작은지 같은 문제를 바로 발견할 수 있습니다.

넷째, 왜 에피소드가 끝나는가?
`train/termination/<reason>`과 `train/termination_rate/<reason>`는 특히 실전적인 지표입니다. 보상만 보면 "학습이 안 된다"로 보일 수 있지만 실제로는 `parked_on_ground`나 boundary 계열 종료가 대부분일 수 있습니다. 이 경우 문제는 모델 구조가 아니라 환경 보상 설계, 시작 조건, termination threshold일 가능성이 큽니다.

TensorBoard는 단순한 시각화 도구가 아니라 강화학습 디버깅 도구입니다.

## HoverPilot에서 TensorBoard를 붙인 방식

이번 커밋은 "있으면 쓰고, 필요 없으면 끌 수 있는" 형태로 구현했다는 점도 중요합니다.

기본적으로는 `PPOConfig`의 `tensorboard_log_dir`가 `runs/hoverpilot-ppo`로 설정되어 있어 학습 시 자동으로 로그가 남습니다. TensorBoard 로깅이 필요 없으면 `--disable-tensorboard` 옵션으로 비활성화할 수 있습니다. 테스트 코드도 기본적으로 TensorBoard를 끄도록 조정해 테스트 환경에서 불필요한 로그 파일이 생기지 않게 했습니다.

또 하나 중요한 변화는 PPO 튜닝 파라미터를 CLI로 노출한 것입니다. 이제 아래 항목을 실행 시점에 직접 조절할 수 있습니다.

- `--n-steps`
- `--batch-size`
- `--epochs`
- `--learning-rate`
- `--gamma`
- `--gae-lambda`
- `--clip-epsilon`
- `--value-coef`
- `--entropy-coef`
- `--max-grad-norm`

이 설정은 TensorBoard와 궁합이 좋습니다. 하이퍼파라미터를 바꿔 가며 실험하고 결과를 TensorBoard에서 비교하면 어떤 설정이 실제로 더 안정적인지 빠르게 판단할 수 있기 때문입니다.

## 실제로 어떻게 사용하는가

먼저 RL 의존성을 설치합니다.

```bash
uv sync --extra rl
```

학습은 평소처럼 실행하면 됩니다.

```bash
uv run hoverpilot-ppo train --timesteps 50000 --save-path ppo_hoverpilot.pt
```

이렇게 실행하면 기본적으로 `runs/hoverpilot-ppo` 아래에 TensorBoard 로그가 쌓입니다.

다른 실험과 분리하고 싶다면 로그 디렉터리를 명시할 수 있습니다.

```bash
uv run hoverpilot-ppo train --timesteps 50000 \
  --n-steps 2048 \
  --batch-size 128 \
  --learning-rate 3e-4 \
  --tensorboard-log-dir runs/hoverpilot-ppo-exp1 \
  --seed 42
```

TensorBoard는 다음처럼 띄웁니다.

```bash
uv run tensorboard --logdir runs
```

그다음 브라우저에서 `http://localhost:6006`으로 접속하면 TensorBoard 대시보드를 볼 수 있습니다.

![브라우저에서 연 TensorBoard 대시보드. eval/avg_reward, eval/avg_length와 train/action/aileron_mean 등 scalar 카드가 5,000 스텝 시점 기준으로 나열되어 있습니다.](/assets/img/hover-pilot/tensorboard-dashboard.png)

_TIME SERIES 탭에서 eval과 train 지표가 카드 형태로 나열됩니다. 왼쪽에서 run을 선택하고 위쪽 검색창으로 태그를 필터링해 원하는 지표만 골라볼 수 있습니다._

로그를 남기지 않고 빠르게 학습만 돌리고 싶다면 아래처럼 실행할 수 있습니다.

```bash
uv run hoverpilot-ppo train --timesteps 50000 --disable-tensorboard
```

## TensorBoard에서 우선적으로 봐야 할 것

HoverPilot 같은 hover 제어 문제에서는 다음 순서로 보는 것이 효율적입니다.

1. `train/episode_reward`, `eval/avg_reward`
   학습이 전체적으로 개선되는지 먼저 확인합니다.
2. `train/policy_loss`, `train/value_loss`, `train/entropy`
   업데이트가 지나치게 불안정하지 않은지 확인합니다.
3. `train/action/throttle_mean`과 각 액션 표준편차
   정책이 throttle 한 축에만 과도하게 의존하거나 액션 다양성이 사라지지 않는지 봅니다.
4. `train/termination_rate/<reason>`
   실패의 주된 원인이 무엇인지 확인합니다. 여기서 원인이 보이면 보상 함수나 termination 조건을 손볼 근거가 생깁니다.

## 마무리

이번 `1e4c7d0` 커밋의 의미는 단순히 "TensorBoard를 붙였다"가 아닙니다. HoverPilot의 PPO 학습을 블랙박스 실행에서 관측 가능한 실험 루프로 바꿨다는 데 의미가 있습니다. 이제 학습 결과만 보는 것이 아니라 학습이 어떤 경로로 진행되는지, 어디서 흔들리는지, 무엇을 조정해야 하는지를 더 명확하게 볼 수 있습니다.

강화학습에서는 모델을 잘 만드는 것만큼 학습을 잘 관찰하는 일도 중요합니다. 이번 변경은 그 관찰 가능성을 코드베이스 안으로 끌어들인 작업입니다.

## 함께 보기

- GitHub 저장소: [hover-pilot](https://github.com/ruddyscent/hover-pilot)
- 이번 글 기준 구현: [1e4c7d0](https://github.com/ruddyscent/hover-pilot/commit/1e4c7d0)
