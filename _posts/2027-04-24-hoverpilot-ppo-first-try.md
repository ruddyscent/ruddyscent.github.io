---
layout: post
title: HoverPilot - PPO를 실제 학습 루프로 구현하기
subtitle: Hover 환경 위에 PPO를 올릴 때 필요했던 정책 구조, reset 처리, 학습 진단
tags:
- hoverpilot
- reinforcement-learning
- ppo
- realflight
- gymnasium
- python
cover-img: /assets/img/develop.jpeg
thumbnail-img: /assets/img/toy-plane.png
share-img: /assets/img/develop.jpeg
author: 전경원
---

[이전 글](/2026-04-18-hoverpilot-gymnasium-interface/)에서는 HoverPilot을 Gymnasium 환경과 호환되는 형태로 맞추었다. `reset()`과 `step()`이 있고, observation과 reward, termination도 연결되어 있다. 하지만 그 상태만으로는 아직 "학습이 된다"고 말하기는 어렵다.

실제로 강화학습을 돌리려면 여기에 더해 **정책 네트워크**, **rollout 수집**, **advantage 계산**, **업데이트 루프**, **episode 복구 로직**, 그리고 **훈련 과정을 관찰할 수 있는 진단 정보**가 함께 갖춰져야 한다. RealFlight의 Airplane Hover Trainer 같은 **다소 까다로운 환경에 PPO를 적용하려면 무엇이 필요한가**를 살펴보자.

## 왜 PPO인가

HoverPilot의 action은 네 개의 연속 제어값으로 이루어진다.

- aileron
- elevator
- throttle
- rudder

즉 이 문제는 본질적으로 연속 제어 문제다. [DQN(Deep Q-Network)](https://arxiv.org/abs/1312.5602)처럼 이산 action을 전제로 한 알고리즘은 바로 적용하기 어렵고, [SAC(Soft Actor-Critic)](https://spinningup.openai.com/en/latest/algorithms/sac.html)나 [TD3(Twin Delayed DDPG)](https://spinningup.openai.com/en/latest/algorithms/td3.html)는 장기적으로는 좋은 후보가 될 수 있지만 첫 구현에서 다루기에는 **구현 요소**가 너무 많다.

반면 [PPO(Proximal Policy Optimization)](https://spinningup.openai.com/en/latest/algorithms/ppo.html)는 첫 구현으로 시도해 볼 만하다. 이유는 다음과 같다.

- 연속 action을 Gaussian policy로 자연스럽게 다룰 수 있다
- actor-critic 구조가 비교적 단순하다
- clipped objective 덕분에 업데이트가 크게 흔들릴 가능성이 낮다
- baseline을 빠르게 만들고 디버깅하기 쉽다

HoverPilot의 현재 단계에서는 최고 효율보다 **신뢰할 수 있는 첫 학습 루프를 만드는 것**이 더 중요하다. 그 점에서 PPO는 꽤 실용적인 선택이다.

PPO를 이해할 때 핵심은 이름의 `Proximal`, 즉 **새 정책이 이전 정책에서 너무 멀리 벗어나지 않게 한다**는 점이다. 강화학습에서는 한 번의 업데이트가 좋아 보이는 action의 확률을 크게 밀어 올릴 수 있다. 문제는 그 action이 정말 좋은 선택이었는지, 아니면 rollout 도중 우연히 좋은 reward와 맞물렸을 뿐인지 불확실하다는 점이다. policy gradient를 너무 크게 적용하면 이전보다 나쁜 정책으로 한 번에 무너질 수 있다.

그래서 PPO는 policy를 직접 크게 바꾸기보다, 이전 policy가 action을 낼 확률과 새 policy가 같은 action을 낼 확률의 비율을 본다.

```text
ratio = new_policy_prob(action) / old_policy_prob(action)
```

이 ratio가 1에 가까우면 새 정책이 예전 정책과 비슷한 판단을 하고 있다는 뜻이고, 1보다 크면 그 action을 더 자주 선택하도록 바뀌었다는 뜻이다. 여기에 advantage를 곱하면 "좋았던 action은 더 자주 선택되도록, 나빴던 action은 덜 선택되도록"이라는 policy gradient의 기본 방향이 된다. PPO의 clip은 이 방향 자체를 없애는 것이 아니라, **한 번의 update에서 그 변화량이 과하지 않게 제한하는 장치**다.

## PPO 구현의 뼈대

HoverPilot의 PPO 구현은 전형적인 actor-critic 구조를 따른다. observation을 입력으로 받아 정책과 가치함수(value function)를 함께 추정하는 형태다.

```python
class ActorCritic(nn.Module):
    def __init__(self, observation_dim: int, action_dim: int):
        super().__init__()
        ...
        self.shared = nn.Sequential(...)
        self.policy_mean = nn.Linear(hidden_sizes[-1], action_dim)
        self.policy_log_std = nn.Parameter(torch.zeros(action_dim, dtype=torch.float32))
        self.value_head = nn.Linear(hidden_sizes[-1], 1)
```

구조 자체는 아주 복잡하지 않다.

- shared MLP 두 층
- action 차원만큼의 policy mean
- 학습 가능한 `log_std`
- scalar value head

즉 observation에서 공통 표현을 만든 뒤, 하나는 행동 분포의 평균으로, 다른 하나는 상태 가치 추정으로 보낸다. HoverPilot에서는 observation이 12차원으로 비교적 작기 때문에, 처음부터 깊고 무거운 네트워크를 쓰기보다는 **작고 해석하기 쉬운 구조**로 시작하는 편이 적절하다.

여기서 actor와 critic의 역할을 분리해서 보면 PPO가 조금 더 선명해진다.

- actor는 현재 상태에서 어떤 action을 선택할지에 대한 확률분포를 만든다
- critic은 현재 상태가 장기적으로 얼마나 괜찮은 상태인지 value를 예측한다
- advantage는 실제로 얻은 return이 critic의 예상보다 얼마나 좋거나 나빴는지를 나타낸다

즉 actor는 "어떤 조종 입력을 더 자주 낼 것인가"를 학습하고, critic은 "지금 상태를 얼마나 좋게 볼 것인가"를 학습한다. PPO update에서는 이 둘이 동시에 움직이지만, 역할은 다르다. 특히 Hover처럼 reward가 noisy하고 episode 실패 원인이 다양할 때는 critic이 어느 정도 안정되어야 actor update도 덜 흔들린다.

## 연속 action을 어떻게 다루나

PPO에서 정책은 action을 직접 내놓지 않고, 정규분포의 평균과 분산을 통해 샘플링한다.

```python
def get_action(self, obs: torch.Tensor):
    mean, log_std, value = self(obs)
    std = torch.exp(log_std)
    dist = Normal(mean, std)
    action = dist.rsample()
    log_prob = dist.log_prob(action).sum(-1)
    return action, log_prob, value
```

이 방식의 장점은 탐색과 정책 표현을 한 분포 안에서 함께 다룰 수 있다는 점이다. HoverPilot 같은 문제에서는 초기에 완전히 결정적인 정책보다, 어느 정도 무작위성(stochasticity)을 가진 정책이 더 유리하다. 그래야 기체를 회복시키는 방향의 조종 입력도 탐색 중에 우연히 발견할 수 있다.

물론 샘플링된 action을 그대로 쓰면 안 된다. RealFlight 쪽 action 범위는 명확하게 제한되어 있으므로, 실제 적용 전에는 환경의 action space로 clip한다.

```python
def _normalize_action(self, raw_action: np.ndarray) -> np.ndarray:
    low = self.env.action_space.low
    high = self.env.action_space.high
    return np.clip(raw_action, low, high)
```

즉 정책은 연속값을 자유롭게 제안하되, 최종적으로는 환경이 허용하는 조종 범위 안으로 집어넣는 구조다.

이때 `log_prob`를 함께 저장하는 이유도 중요하다. PPO는 rollout을 모을 때의 old policy와 update 시점의 current policy를 비교해야 한다. 따라서 action 자체만 저장해서는 부족하고, **그 action이 당시 정책에서 얼마나 그럴듯한 선택이었는지**도 함께 저장해야 한다. 나중에 같은 observation/action 쌍을 현재 policy에 다시 통과시켜 새 `log_prob`를 계산하고, 둘의 차이로 ratio를 만든다.

## Hover 문제에서는 throttle dead start를 피해야 한다

HoverPilot에서 PPO를 그냥 표준 형태로 붙이면 초반부터 한 가지 문제가 생긴다. policy mean과 std를 완전히 중립적으로 초기화하면, throttle까지 거의 0 근처에서 시작할 가능성이 높다. 그러면 비행기가 뜨기도 전에 episode가 일찍 끝나 버리고, 초반 rollout이 "추락 데이터"로만 채워질 수 있다.

그래서 정책 초기화에 아주 약한 bias를 넣는다.

```python
with torch.no_grad():
    self.policy_mean.bias.zero_()
    if action_dim >= 3:
        self.policy_mean.bias[2] = 0.55
        self.policy_log_std[2] = -1.0
```

여기서 핵심은 세 번째 action, 즉 throttle이다.

- 초기 mean을 `0.55` 부근으로 둔다
- throttle 분산은 너무 크지 않게 시작한다

이는 답을 미리 알려준다는 의미가 아니다. 그보다는 Hover Trainer의 시작 상태에서 **아예 무추력 상태로 죽어 버리는 비생산적인 탐색을 줄이는 장치**에 가깝다. 실제 제어 문제에서는 이런 종류의 초기화가 꽤 중요하다.

## rollout buffer는 단순하지만 꼭 필요한 정보만 담는다

PPO의 학습은 replay buffer 기반이 아니라, 짧은 길이의 trajectory를 모아서 한 번에 업데이트하는 방식이다. 그래서 rollout buffer에는 다음 정보가 들어간다.

- observation
- action
- reward
- done mask
- value estimate
- old log probability

```python
class RolloutBuffer:
    def __init__(...):
        self.observations = ...
        self.actions = ...
        self.rewards = ...
        self.dones = ...
        self.values = ...
        self.log_probs = ...
        self.advantages = ...
        self.returns = ...
```

중요한 점은 여기서 `done`을 단순 boolean으로만 보지 않고, bootstrap 계산에 바로 쓸 수 있는 mask 형태로 저장한다는 점이다. episode가 끝난 뒤에는 다음 state value를 이어서 쓰면 안 되기 때문이다.

## advantage는 GAE로 계산한다

PPO 구현에서 사실상 중심이 되는 부분은 advantage 계산이다. HoverPilot에서는 GAE(Generalized Advantage Estimation)를 사용한다.

```python
def compute_returns_and_advantages(self, last_value: float, gamma: float, lam: float):
    gae = 0.0
    ...
    for step in reversed(range(self.index)):
        next_value = last_value_tensor if step == self.index - 1 else self.values[step + 1]
        delta = self.rewards[step] + gamma * next_value * self.dones[step] - self.values[step]
        gae = delta + gamma * lam * self.dones[step] * gae
        self.advantages[step] = gae
```

이 방식은 reward-to-go만 그대로 쓰는 것보다 분산이 작고, value bootstrap을 함께 활용할 수 있다는 장점이 있다. Hover처럼 한순간의 제어 실패가 몇 step 뒤의 추락으로 이어지는 문제에서는, 단기 reward만 보는 것보다 이런 방식이 더 낫다.

그리고 PPO 업데이트 직전에는 advantage를 normalize한다.

```python
advantages = (advantages - advantages.mean()) / (advantages.std(unbiased=False) + 1e-8)
```

이 normalization은 구현상 사소해 보이지만, 실제로는 업데이트 scale을 안정화하는 데 꽤 도움이 된다.

직관적으로 말하면 advantage는 actor에게 주는 학습 신호다. 양수 advantage는 "이 action은 예상보다 좋았으니 확률을 올려도 된다"는 뜻이고, 음수 advantage는 "예상보다 나빴으니 확률을 낮추는 편이 낫다"는 뜻이다. 그런데 advantage의 절댓값 scale이 rollout마다 크게 달라지면, 같은 `learning_rate`와 `clip_epsilon`을 써도 실제 업데이트의 폭이 달라진다. normalize를 넣는 이유는 이 신호의 평균과 분산을 맞춰, PPO update가 reward scale에 덜 민감하게 만들기 위해서다.

## PPO 업데이트는 정석에 가깝게 유지한다

업데이트 단계에서는 old policy에서 모은 action log-prob와 현재 policy의 log-prob를 비교해 ratio를 만든다. 그리고 그 ratio를 clip한 surrogate objective를 사용한다.

```python
ratio = torch.exp(batch_log_probs - batch_old_log_probs)
surrogate1 = ratio * batch_advantages
surrogate2 = torch.clamp(
    ratio,
    1.0 - self.config.clip_epsilon,
    1.0 + self.config.clip_epsilon,
) * batch_advantages
policy_loss = -torch.min(surrogate1, surrogate2).mean()
value_loss = self.config.value_coef * (batch_returns - batch_values).pow(2).mean()
entropy_loss = -self.config.entropy_coef * batch_entropy.mean()
loss = policy_loss + value_loss + entropy_loss
```

이 구현은 의도적으로 정석에서 크게 벗어나지 않는다. HoverPilot에서 지금 중요한 것은 새로운 PPO 변형을 실험하는 것이 아니라, **환경과 학습 루프가 일관되게 연결되는 baseline을 확보하는 것**이기 때문이다.

여기에 추가한 안정화 장치는 두 가지다.

- `clip_epsilon`으로 과도한 policy 이동 제한
- `max_grad_norm`으로 gradient exploding 방지

즉 policy update를 너무 공격적으로 하지 않고, 조금씩 움직이게 설계한다.

clip objective가 특히 중요한 이유는 advantage의 부호에 따라 제한 방향이 달라지기 때문이다. advantage가 양수인 action은 새 policy에서 더 자주 선택되는 편이 좋지만, ratio가 `1 + clip_epsilon`을 넘을 정도로 커지는 것은 막는다. 반대로 advantage가 음수인 action은 덜 선택되는 편이 좋지만, ratio가 `1 - clip_epsilon` 아래로 과하게 내려가는 것도 막는다. PPO의 안정성은 이 단순한 제한에서 많이 나온다.

또 하나의 실용적인 포인트는 같은 rollout을 한 번만 쓰지 않고, minibatch로 나눠 여러 epoch 동안 업데이트한다는 점이다. policy gradient 계열은 on-policy라서 오래된 데이터를 계속 재사용하기 어렵지만, PPO는 clip objective 덕분에 **방금 수집한 rollout을 몇 차례 더 재사용하는 절충안**을 제공한다. 그래서 `n_steps`, `batch_size`, `epochs`가 서로 맞물린다. rollout을 너무 작게 잡으면 gradient가 noisy하고, epoch를 너무 많이 돌리면 old policy 기준으로 모은 데이터가 점점 낡아진다.

## 이 구현에서 더 어려운 문제는 reset이다

사실 HoverPilot에서 PPO 수식 자체보다 더 까다로운 부분은 episode lifecycle이다. 일반적인 Gym benchmark 환경은 `done=True`가 나오면 바로 다음 `reset()`으로 새로운 episode가 시작된다. 하지만 RealFlight Hover Trainer는 그렇지 않다.

- 충돌 직후에도 reset이 바로 반영되지 않을 수 있다
- trainer가 reposition 중일 수 있다
- episode는 끝났는데 환경은 아직 waiting-for-reset 상태일 수 있다

그래서 `done` 시점에 곧장 `env.reset()`을 호출하는 방식만으로는 부족하다. PPO 루프에서 별도의 helper를 두고, 다음 episode가 실제로 시작될 때까지 기다리는 경로를 만든다.

```python
def reset_env_with_wait(env, *, action=None, initial_action=None):
    if getattr(env, "_waiting_for_reset", False):
        ...
        return _wait_for_episode_start(env, poll_wait=poll_wait, action=action)

    try:
        reset_options = None
        if initial_action is not None:
            reset_options = {"initial_action": initial_action}
        return env.reset(options=reset_options)
    except TimeoutError as exc:
        ...
        return _wait_for_episode_start(env, poll_wait=poll_wait, action=action)
```

이 구조의 의미는 단순하다.

- 이미 waiting 상태라면 불필요한 `reset()`을 다시 호출하지 않는다
- `reset()`이 timeout 되면 polling으로 다음 episode 시작을 기다린다
- reset 동안 사용할 action과 episode 첫 시작에 사용할 action을 분리한다

즉 학습 루프가 RealFlight Trainer의 lifecycle에 맞춰 **조금 더 끈질기게 버티도록** 만든 셈이다.

## initial action과 wait action을 따로 두는 이유

이 구현에서 꽤 중요한 디테일은 `initial_action`과 `wait_action`을 분리한다는 점이다.

```python
DEFAULT_WAIT_ACTION = np.asarray([0.0, 0.0, 0.0, 0.0], dtype=np.float32)
DEFAULT_INITIAL_ACTION = np.asarray([0.0, 0.0, 0.55, 0.0], dtype=np.float32)
```

두 action은 역할이 다르다.

- `wait_action`은 reset 대기 중 trainer를 건드리지 않기 위한 안전한 idle 입력
- `initial_action`은 새 episode가 시작될 때 너무 죽은 상태로 출발하지 않기 위한 초기 입력

처음에는 이 둘을 같은 값으로 둬도 되지 않을까 생각할 수 있다. 하지만 실제 hover 문제에서는 그렇지 않다. 기다리는 동안에는 기체를 자극하지 않는 것이 중요하고, episode가 시작될 때는 어느 정도 추력을 확보하는 것이 중요하다. 이 둘은 목적이 다르다.

## 학습 로그는 숫자 하나로 끝내지 않는다

강화학습을 구현할 때 흔히 보는 출력은 `episode reward` 하나다. 하지만 HoverPilot에서는 그것만으로는 부족하다. 학습이 망가질 때 원인이 너무 많기 때문이다.

- action이 한쪽 채널로 쏠리는가
- termination reason이 특정 실패로 몰리는가
- entropy가 너무 빨리 줄어드는가
- return과 advantage scale이 비정상적인가

그래서 terminal log도 더 자세하게 넣는다. 예를 들면 episode 시작과 종료 시점의 debug state를 출력하고, rollout 단위로 action 통계와 termination 분포를 요약한다.

```python
print(
    f"[PPO] rollout steps={total_steps}/{self.config.timesteps} "
    f"samples={rollout.index} reward_mean={np.mean(rewards):+.3f} "
    ...
)
print(f"[PPO] rollout actions {self._format_action_stats(action_array)}")
print(f"[PPO] rollout terminations {reason_summary}")
```

이런 로그는 모델 성능을 예쁘게 보여주기 위한 것이 아니라, **학습 실패를 빠르게 진단하기 위한 도구**다. 예를 들어 `parked_on_ground` termination 비율이 비정상적으로 높다면, reward보다 reset 처리나 초기 추력 설정부터 다시 봐야 한다.

PPO 관점에서 추가로 보고 싶은 지표도 있다.

- `approx_kl`: old policy와 current policy가 얼마나 벌어졌는지
- `clip_fraction`: ratio가 clip 범위 밖으로 나간 sample 비율
- `entropy`: 정책이 너무 빨리 결정적으로 굳어지는지
- `explained_variance`: critic이 return을 어느 정도 설명하고 있는지
- `value_loss`: critic 학습이 발산하거나 reward scale을 못 따라가는지

예를 들어 `clip_fraction`이 계속 높다면 `clip_epsilon`이 작거나 learning rate가 너무 공격적일 수 있다. `approx_kl`이 갑자기 튀면 policy update가 한 번에 너무 멀리 간 것이다. `entropy`가 초반부터 빠르게 낮아지면 탐색이 충분하지 않을 수 있고, `explained_variance`가 계속 낮으면 actor보다 critic 쪽 문제를 먼저 의심해야 한다.

## CLI에서 하이퍼파라미터를 직접 노출한다

초기 실험 단계에서 하이퍼파라미터를 코드 안에서 수정하는 일은 오히려 불편하다. train CLI에서 PPO 주요 파라미터를 터미널에서 직접 조절할 수 있게 한다.

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

이렇게 해 두면 같은 환경에서 rollout 길이만 바꾸거나, entropy coefficient만 조정하는 실험을 빠르게 반복할 수 있다. 강화학습 초반 구현에서 중요한 것은 "정답 하이퍼파라미터"를 찾는 것이 아니라, **어떤 파라미터가 어떤 현상과 연결되는지 보이게 하는 것**이다.

PPO는 구현 자체보다도 이 연결 관계를 관찰하는 일이 중요하다. `clip_epsilon`은 policy가 한 번에 움직일 수 있는 폭이고, `entropy_coef`는 탐색을 유지하려는 압력이며, `value_coef`는 critic loss가 전체 loss에서 차지하는 비중이다. `gae_lambda`는 bias와 variance의 절충이고, `max_grad_norm`은 최종적으로 gradient step이 과격해지는 것을 막는 안전장치다. 숫자를 바꿀 때는 episode reward만 볼 것이 아니라, 위의 PPO 진단 지표가 같이 어떻게 움직이는지 봐야 한다.

## 이번 PPO 구현에서 중요한 것

정리하면 HoverPilot의 PPO 구현에서 중요했던 것은 수식보다도 오히려 다음 네 가지였다.

- Hover 문제에 맞는 연속 action 정책 구성
- throttle dead start를 줄이기 위한 초기화
- trainer lifecycle에 맞춘 reset-wait 복구 로직
- 학습을 관찰하고 진단하기 위한 상세 로그

HoverPilot처럼 외부 시뮬레이터와 연결된 환경에서는 PPO의 표준적인 구현만으로는 실제 학습 루프를 안정적으로 돌리기 어렵다. 정책 초기화, episode 경계 처리, 시작 action, waiting 상태 같은 구현상의 세부 사항도 함께 준비해야 한다.

## 첫술에 배가 차는가?

실행 테스트를 할 겸, PPO 훈련 루프를 돌려 보았다. 1000 rollout 정도면 뭔가 변화가 보이지 않을까 기대했는데, 실제로는 그보다 훨씬 짧은 시간 안에 모델이 뭔가를 배우는 듯한 행동을 보여줬다. 다만, 내가 바라던 기동은 아니었다. 모델은 에피소드가 시작하자마자 비행기 꼬리부터 시작해서 지면에 살짝 착륙해버렸다. Airplane Hover Trainer에서 이러한 기동이 가능하다는 것도 처음 알았다. 이제 왜 이러한 방향으로 학습이 이루어지는 지 디버깅을 할 시간이다.

![]()

## 함께 보기

- GitHub 저장소: [hover-pilot](https://github.com/ruddyscent/hover-pilot) (태그: [ppo-first-try](https://github.com/ruddyscent/hover-pilot/releases/tag/ppo-first-try))
- 참고 영상: [Proximal Policy Optimization(PPO) - How to train Large Language Models](https://www.youtube.com/watch?v=TjHH_--7l8g)
