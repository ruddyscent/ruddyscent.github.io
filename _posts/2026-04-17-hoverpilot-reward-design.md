---
layout: post
title: HoverPilot — 강화학습을 위한 Reward 설계
subtitle: RealFlight Link 상태 데이터를 기반으로 hover reward를 구성해보자
tags: [hoverpilot, reinforcement-learning, reward, realflight, realflight-link]
cover-img: /assets/img/develop.jpeg
thumbnail-img: /assets/img/toy-plane.webp
share-img: /assets/img/develop.jpeg
author: 전경원
---

이전 단계까지 진행하면서 state와 action은 이미 연결해 두었다. 이제 남은 것은 강화학습에서 가장 중요한 요소 중 하나인 **보상 함수(reward)**를 설계하는 일이다. reward는 에이전트가 **무엇을 잘한 것으로 판단할지**를 정의하는 기준이며, 학습이 어떤 방향으로 진행될지를 결정한다. 결국 reward를 어떻게 설계하느냐에 따라 최종 학습 결과도 크게 달라진다.

## Reward 설계의 관점

hover는 하나의 조건으로 정의되지 않는다. 기체가 공중에 떠 있다고 해서 곧바로 "잘하고 있다"고 말할 수는 없다. 자세가 무너지고 있을 수도 있고, 위치가 boundary 바깥으로 밀려나고 있을 수도 있고, 조종 입력이 지나치게 크거나 불안정할 수도 있다. 즉 hover를 잘한다는 것은 여러 조건을 동시에 만족하는 상태에 가깝다.

- 자세가 안정적인가
- 위치가 기준점에서 크게 벗어나지 않는가
- 고도가 급격하게 흔들리지 않는가
- 경계(boundary)에 과도하게 가까워지지 않는가

그래서 reward는 하나의 "정답 체크"가 아니라, 여러 기준을 합쳐 **원하는 비행 상태 쪽으로 에이전트를 유도하는 장치**로 설계해야 한다.

## Reward 구조

이번 구현에서는 reward를 하나의 거대한 공식으로 다루지 않고, **여러 penalty 항목으로 나눈 뒤 합산하는 방식**을 사용했다.

```python
reward = -(
    position_penalty
    + altitude_penalty
    + attitude_penalty
    + boundary_proximity_penalty
) + terminal_penalty
```

실제 구현에서도 같은 구조를 사용한다.  아래 코드는 `rflink-howver-reward` 태그의 `compute_reward()` 핵심 부분이다.

```python
def compute_reward(state: FlightAxisState, config: RewardConfig) -> RewardBreakdown:
    termination = compute_termination(state, config)

    x_error = ...
    y_error = ...
    altitude_error = ...

    position_penalty = ...
    altitude_penalty = ...
    attitude_penalty = ...
    boundary_proximity_penalty = _compute_boundary_proximity_penalty(state, config)
    terminal_penalty = config.terminal_failure_reward if termination.terminated else 0.0

    reward = -(
        position_penalty
        + altitude_penalty
        + attitude_penalty
        + boundary_proximity_penalty
    ) + terminal_penalty
```

이 구조의 핵심은 단순하다.

- 기준에서 멀어질수록 penalty가 커진다
- 위험한 상태일수록 reward가 빠르게 나빠진다
- episode가 실패 조건에 도달하면 큰 terminal penalty를 준다

즉 reward를 "칭찬 점수"라기보다, **얼마나 이상적인 hover 상태에서 멀어졌는지를 측정하는 값**으로 보는 편이 더 직관적이다.

## Penalty 중심 설계

hover 학습에서 중요한 것은 에이전트가 무엇을 해야 하는지보다, **무엇을 피해야 하는지**를 먼저 명확히 알려주는 일이다. 예를 들어 아래와 같은 현상은 초기에 자주 발생한다.

- 기체가 한쪽으로 계속 밀린다
- 기수가 빠르게 흔들린다
- 높이가 불안정하게 튄다
- 경계 근처에서 아슬아슬하게 버틴다

이런 상태에 각각 penalty를 두면 에이전트는 자연스럽게 다음 방향으로 학습하게 된다.

- 중심에 가까이 머무르기
- 급격한 자세 변화 줄이기
- 안전한 고도 범위 유지하기
- boundary 근처를 피하기

reward를 잘게 나눠두면, 나중에 어떤 행동이 학습을 방해하는지도 훨씬 선명하게 보인다.

## Boundary와 종료 조건

RealFlight의 Airplane Hover Trainer에서는 boundary를 벗어나면 episode가 끝난다.

![](/assets/img/hover-pilot/boundary-violation-crash.png)

이 조건은 단순히 화면상의 안내가 아니라, 실제로 학습 환경에서 **반드시 반영해야 하는 규칙**이다. 그래서 이번 설계에서는 boundary를 두 단계로 다뤘다. 

첫째, 경계에 가까워질수록 `boundary_proximity_penalty`를 주었다.  둘째, 경계를 넘으면 즉시 종료하고 `terminal_penalty`를 적용했다. 이 부분도 구현에서는 penalty와 termination을 분리해서 계산한다.

```python
def compute_termination(state: FlightAxisState, config: RewardConfig) -> TerminationResult:
    x_error = ...
    if abs(x_error) > config.max_abs_x_m:
        return TerminationResult(True, "out_of_bounds_x")

    altitude_agl_m = ...
    if altitude_agl_m < config.min_altitude_agl_m:
        return TerminationResult(True, "altitude_too_low")
    if altitude_agl_m > config.max_altitude_agl_m:
        return TerminationResult(True, "altitude_too_high")

    ...
    return TerminationResult(False, None)


def _boundary_edge_penalty(distance_to_edge: float, limit: float, margin_ratio: float) -> float:
    margin = max(limit * margin_ratio, 1.0e-6)
    if distance_to_edge >= margin:
        return 0.0
    if distance_to_edge <= 0.0:
        return 1.0

    normalized = 1.0 - (distance_to_edge / margin)
    return normalized * normalized
```

이렇게 하면 에이전트는 두 가지를 동시에 배우게 된다.

- 경계에 가까워지는 것 자체가 좋지 않다
- 경계를 넘는 것은 명확한 실패다

이 차이는 중요하다. 종료 조건만 두면 에이전트는 경계 직전까지 가는 행동을 반복할 수 있다. 반대로 proximity penalty까지 함께 두면, **안전 영역 안쪽에서 안정적으로 hover하는 정책**을 더 잘 유도할 수 있다.

## Reward Breakdown

reward 설계는 대개 한 번에 끝나지 않는다. 처음 만든 공식이 그대로 정답이 되는 경우는 거의 없다. 실제 학습을 돌려보면 어떤 항목은 지나치게 크고, 어떤 항목은 거의 영향을 주지 못하는 경우가 자주 생긴다. 그래서 reward를 항목별로 분해해 두는 것이 중요하다. 예를 들어 다음과 같은 질문에 바로 답할 수 있어야 한다.

- position penalty가 너무 커서 자세 제어를 못 배우는 건 아닌가
- attitude penalty가 너무 약해서 기체가 흔들려도 reward 손실이 작은 건 아닌가
- boundary penalty가 너무 늦게 작동해서 종료 직전까지 위험 행동을 허용하는 건 아닌가

이런 분석을 하려면 reward를 한 줄짜리 숫자로만 보면 안 된다. 각 항목을 따로 계산하고 관찰할 수 있어야 한다. 즉 breakdown은 단순 디버그 로그가 아니라, **reward 튜닝 자체를 가능하게 하는 관측 도구**다. 그래서 실제 함수도 최종 reward만 반환하지 않고, 각 penalty를 함께 담은 `RewardBreakdown`을 반환한다.

```python
return RewardBreakdown(
    reward=reward,
    position_penalty=position_penalty,
    altitude_penalty=altitude_penalty,
    ...,
    termination_reason=termination.termination_reason,
)
```

## 이번 단계의 결과

여기까지 오면 HoverPilot의 강화학습 루프는 핵심 구성요소를 거의 갖추게 된다.

- state: RealFlight에서 읽어온다
- action: RC 채널 입력으로 보낸다
- reward: hover 기준으로 계산한다
- termination: boundary와 실패 조건으로 결정한다

이제 남은 일은 이 구조를 Gymnasium 인터페이스에 맞게 감싸서, 학습 알고리즘이 바로 사용할 수 있는 환경으로 정리하는 것이다.

## 다음 단계

다음 글에서는 지금까지 만든 state, action, reward, termination을 하나의 Gymnasium 환경으로 묶는다. 그 순간부터 HoverPilot은 단순한 시뮬레이터 제어 코드가 아니라, PPO나 SAC 같은 알고리즘에 바로 연결할 수 있는 **학습 가능한 environment**가 된다.

## 함께 보기

- GitHub 저장소: [hover-pilot](https://github.com/ruddyscent/hover-pilot)
- 이번 글 기준 구현: [rflink-howver-reward](https://github.com/ruddyscent/hover-pilot/releases/tag/rflink-howver-reward)
