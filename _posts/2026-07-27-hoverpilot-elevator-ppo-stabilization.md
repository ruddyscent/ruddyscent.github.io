---
layout: post
title: HoverPilot - 엘리베이터 하나로 PPO 호버 드리프트 잡기
subtitle: 관측의 부호와 목표 프레임을 바로잡고, 정책 구조를 대칭으로 짜서 PPO가 스스로 복원하게 했다
tags:
- hoverpilot
- reinforcement-learning
- ppo
- realflight
- python
cover-img: /assets/img/develop.jpeg
thumbnail-img: /assets/img/toy-plane.webp
share-img: /assets/img/develop.jpeg
author: 전경원
mathjax: true
---

[이전 글](/2026-07-25-hoverpilot-tensorboard-logging/)에서는 PPO 학습에 TensorBoard 로깅을 붙여서, 정책이 꼬리부터 착지하는 이유를 지표로 들여다볼 수 있게 만들었다. 관측 도구는 갖췄지만, 정작 4개 채널을 한꺼번에 학습하다 보니 실패의 원인을 하나로 짚기가 어려웠다. 자세가 무너진 건지, 보상 설계가 잘못된 건지, 부호가 뒤집힌 건지, 아니면 통신이 끊긴 건지 — 후보가 너무 많았다.

그래서 제어 채널을 엘리베이터 하나로 줄였다. 이 글은 그 좁은 문제 안에서 PPO가 실제로 호버를 세우기까지, 무엇을 관찰하고 어떤 가설을 어떻게 고쳐 나갔는지에 대한 기록이다. 핵심은 두 가지다. 하나는 관측의 부호를 물리적으로 일관되게 맞추는 일이고, 다른 하나는 학습해야 할 자유도를 정책 구조 자체로 줄이는 일이었다.

## 문제를 단순하게 바꾸자

aileron, elevator, throttle, rudder 네 채널을 동시에 학습하면, 뭔가 잘못됐을 때 원인이 다음 중 무엇인지 구분하기 어렵다.

- PPO 구현 오류
- action distribution 오류
- 보상 설계 오류
- 좌표계 또는 부호 오류
- RealFlight 통신 문제
- 여러 축이 서로 얽히는 결합 문제

엘리베이터만 남기면 문제가 사실상 종방향 자세와 위치 복원 문제로 줄어든다. aileron과 rudder는 `0`으로 고정하고, throttle은 `0.55`로 둔다. 이 설정의 목적은 완전한 4축 호버가 아니다. PPO 정책, 환경 lifecycle, 관측과 보상의 정의가 서로 제대로 연결되는지를 먼저 확인하는 것이다.

성공 조건은 명확하게 정했다. 기수가 하늘을 향하고 동체가 지면에 수직인 자세를 올바른 호버로 보고, 이 자세의 signed inclination error를 `0°`로 삼는다. 기체가 한쪽으로 이동하면 반대편으로 기울여 원점 방향 속도를 만들고, 원점에 가까워지면 지나쳐 버리기 전에 미리 제동한다. 그러면서 reset 없이 300 step 구간을 반복해 유지하고, 원통 경계에 닿기 전에 제어를 포기하지 않아야 한다.

## 처음 본 문제: 기울면 돌아오지 못한다

초기 정책은 약 1~2초 동안은 균형을 잡는 것처럼 보였다. 그런데 동체 기울기가 조금 커지기 시작하면 늘 같은 패턴으로 무너졌다. 기체가 한쪽으로 기울고, 그 방향으로 수평 이동이 시작되고, 정책은 다시 수직 자세로 돌아가려 하지만 원점으로 되돌아올 만큼 반대편으로 넘기지는 못한다. 남은 수평 속도가 그대로 기체를 훈련 경계까지 밀어내 충돌로 끝났다.

여기서 얻은 첫 교훈은 단순했다. **자세를 `0°`로 되돌리는 것만으로는 이미 생긴 수평 속도를 없앨 수 없다.** 원점으로 돌아오려면 한동안은 오히려 반대편으로 기체를 넘겨서 원점 방향 가속을 만들어야 한다. 자세 안정화와 위치 복귀는 서로 다른 목표였다.

경계 모델도 처음엔 잘못 잡았다. 직육면체라고 가정했다가, 지면 위 반구라고 바꿨다가, 실제로 관찰해 보니 수직 원통이었다. 반구로 계산하면 고도가 올라갈수록 허용 수평 반경이 줄어드는데, 이 잘못된 모델 때문에 실제 충돌이 나기도 전에 환경이 경계를 벗어났다고 판단하고 제어를 끝내 버렸다. 최종 모델은 고도와 무관하게 수평 거리만 본다.

```text
horizontal distance = hypot(x - reset_x, y - reset_y)
terminate if horizontal distance > 6 m
```

너무 낮은 지상 고도(AGL, Above Ground Level)는 원통 경계와 별개로 추락 조건으로 따로 처리하고, 확인되지 않은 상단 한계(ceiling)는 넣지 않았다.

## 좌표계 부호가 틀리면 튜닝으로는 못 고친다

드리프트(기체가 원점에서 한쪽으로 계속 밀려나는 현상)를 잡으려고 learning rate와 보상 가중치를 한참 만졌지만 소용이 없었다. 문제는 학습이 아니라 관측의 부호에 있었다.

RealFlight는 수직 nose-up 자세를 `m_inclination = 90°`로 보고한다. 원하는 제어 오차는 이 자세에서 `0°`여야 하니, 단순하게는 이렇게 변환할 수 있다.

$$
\text{inclination\_error} = \text{inclination} - 90°
$$

문제는 수직 자세 부근이 Euler angle의 특이점이라는 데 있다. 수직에 가까워지면 방위각(azimuth)과 롤(roll)이 180° 뒤집힐 수 있어서, $$\text{inclination} - 90°$$만 쓰면 수직의 서로 반대편에 있는 tilt가 같은 부호로 해석된다. 왼쪽으로 기운 것과 오른쪽으로 기운 것을 정책이 구분하지 못하니, 복원 방향을 배울 도리가 없었다.

![수직 호버 자세 부근의 Euler 특이점을 보여주는 3차원 좌표계 그림. +x(기수 방위)를 향해 기운 경우 A와 반대편 -x로 기운 경우 B가 수직축에서 같은 각도 θ만큼 벌어져 있다. inclination - 90°만 쓰면 두 경우가 모두 -θ로 같아져 부호를 잃지만, reset heading에 cos로 투영하면 A는 -θ, B는 +θ로 반대 부호가 된다.](/assets/img/hover-pilot/euler-singularity-inclination-sign.png)

_수직 자세 부근에서는 서로 반대편으로 기운 A와 B가 같은 inclination을 갖는다. `inclination - 90°`만으로는 둘이 구분되지 않지만(왼쪽 빨간 상자), reset heading 기준 cos 투영을 곱하면 방위각이 180° 뒤집힌 B의 부호만 반대로 뒤집혀 좌우가 갈린다(오른쪽 파란 상자)._

그래서 최종 signed error는 reset 시점의 기수 방위(heading)를 기준으로 tilt를 투영해서 만든다.

$$
\text{heading\_error} = \text{azimuth} - \text{reset\_heading}
$$

$$
\text{signed\_error} = (\text{inclination} - 90°) \cdot \cos(\text{heading\_error})
$$

이제 수직 자세는 언제나 `0°`이고, 양쪽 tilt는 서로 반대 부호를 갖는다. 관측이 물리적으로 일관된 의미를 되찾자, 그제야 나머지 조정이 의미를 갖기 시작했다.

같은 종류의 부호 함정이 속도에도 있었다. 수직 호버 reset 전후로 RealFlight가 주는 world 좌표 U/V 속도 성분의 부호가 달라지는 경우가 있어서, 이 값을 그대로 쓰면 같은 실제 이동이 스텝마다 반대 부호로 읽혔다. 결국 시뮬레이터가 주는 속도를 믿는 대신, 연속한 위치와 물리 시간(physics time)으로 종방향 위치 변화율을 직접 계산했다.

$$
\text{velocity} = \frac{\text{current\_position} - \text{previous\_position}}{\Delta t_{\text{physics}}}
$$

첫 state이거나, 물리 시간이 뒤로 가거나, trainer reposition이라고 볼 만큼 큰 teleport가 있으면 속도를 0으로 초기화한다. 관측, 보상, telemetry, 복원 목표(recovery target) 모두 이 하나의 계산 결과를 재사용하도록 통일했다.

## 복원 목표: 위치를 보고 기울일 방향을 정한다

자세 오차만 줄이는 정책으로는 드리프트를 못 잡는다는 걸 알았으니, 남은 질문은 "그럼 목표 자세를 어떻게 정할 것인가"였다. 답은 목표를 고정하지 않고 위치와 속도에 따라 움직이게 하는 것이었다.

원점 오른쪽에 있으면 왼쪽 가속이 필요하고, 오른쪽으로 계속 이동 중이면 더 강한 왼쪽 tilt가 필요하다. 반대로 원점으로 빠르게 접근 중이면 통과하기 전에 tilt를 줄이거나 반대로 제동해야 한다. 이걸 하나의 복원 목표 자세로 묶었다.

$$
\begin{aligned}
\text{target\_inc\_error} = \text{clamp}\big(&-(2 \cdot \text{position\_error} + 3 \cdot \text{position\_rate}), \\
&-30°,\ +30°\big)
\end{aligned}
$$

정책이 실제로 추종하는 첫 번째 관측은 수직 자세 오차 그 자체가 아니라, 이 목표에 대한 tracking error다.

$$
\text{tracking\_error} = \text{signed\_inc\_error} - \text{target\_inc\_error}
$$

여기서 중요한 건 위치에 비례하는 elevator 명령을 action에 직접 더한 게 아니라는 점이다. 정책이 따라갈 **자세 목표 자체**를 상태에 따라 바꿨다. 위치 복원 로직을 보상이나 action이 아니라 관측의 기준 프레임에 심은 셈이다.

## 순수 PPO도 정책 구조에 대칭성을 넣을 수 있다

목표 프레임까지 맞췄는데도 한동안 정책은 한쪽으로 치우쳤다. 일반 MLP actor에 position feature와 복원 사전 항(prior)을 모두 주면, actor가 잔차(residual)로 복원 명령을 상쇄하는 방향을 학습해 버렸다. 문제가 없던 중간 구현과 비교해 보니, 당시엔 강한 복원 사전 항이 있었는데 이후 단순화 과정에서 그게 빠진 게 결정적인 차이였다.

사전 항을 다시 넣으면 간단하지만, 그러면 "순수 PPO가 스스로 배울 수 있는가"라는 원래 질문이 흐려진다. 대신 actor 구조 자체를 제약하기로 했다. `--policy-preset none`인 엘리베이터 모드에서는 actor MLP를 아예 쓰지 않고, 학습 가능한 양수 이득(gain) 두 개만 둔다.

$$
\text{latent\_elevator} = -k_{\text{attitude}} \cdot o_0 + k_{\text{rate}} \cdot o_1
$$

여기서 $$o_0$$는 복원 목표 기준 tracking error, $$o_1$$은 피치 각속도(pitch rate)다. 초기 이득은 `[0.55, 0.45]`로 두고, 두 이득 모두 $$\text{softplus}(\text{raw\_gain})$$으로 parameterize한다. softplus는 입력을 항상 양수로 매끄럽게 눌러 주는 함수($$\text{softplus}(x)=\log(1+e^{x})$$)라, 이렇게 두면 학습 중에도 이득이 음수가 될 수 없어서 복원 부호가 뒤집히지 않는다. 마지막에는 이 값을 tanh 함수에 통과시켜 `[-1, 1]` 범위로 눌러 준다.

이 구조의 좋은 점은 물리적 대칭성이 학습이 아니라 설계로 보장된다는 데 있다. 기체 상태를 좌우로 뒤집으면 엘리베이터 action도 정확히 반대로 나온다.

$$
\pi(\text{mirror}(o)) = -\pi(o)
$$

실제로 그런지는, 매 학습 구간이 끝날 때마다 대표 상태 몇 개에 정책을 넣어 어떤 복원 action이 나오는지 재보는 recovery probe로 확인했다.

![30,000 step 학습 동안 recovery probe의 좌우 action을 겹쳐 그린 손그림풍 그래프. attitude probe의 +side와 -side가 각각 약 -0.5와 +0.5에서, position probe의 +side와 -side가 약 -0.28과 +0.28에서 서로 부호만 반대인 대칭 쌍을 이루며 학습 내내 유지된다. symmetry_error는 0, effective_restoring_fraction은 1.0이다.](/assets/img/hover-pilot/recovery-probe-mirror-symmetry.png)

_같은 자극을 좌우로 뒤집어 넣었을 때 정책이 내는 복원 action이 학습 30,000 step 내내 정확히 부호만 반대다. 이 대칭은 학습으로 얻은 게 아니라 actor 구조로 보장되므로, `symmetry_error`는 계속 0, `effective_restoring_fraction`은 1.0으로 남는다._

위치와 속도는 앞서 만든 복원 목표를 통해 tracking error에 이미 녹아 있으므로, actor가 위치에 직접 비례하는 항을 따로 학습해 복원 목표를 상쇄할 여지도 없다. 외부 controller action을 섞지 않으면서도 좌우 대칭성, 복원 부호, 작은 actor, 상태 의존 목표 프레임을 정책 구조 안에 넣은 것이다. 학습해야 할 자유도를 문제에 맞게 줄이는 편이, 무작정 큰 MLP를 쓰는 것보다 훨씬 안정적이었다.

critic은 학습을 위해 기존의 비선형 MLP를 그대로 둔다. 다만 결정적(deterministic) play나 평가에서는 critic을 계산하지 않는다.

관측은 직접 제어 가능한 종방향 상태만 담은 6차원이다.

| 인덱스 | 값 | 정규화 scale |
|---:|---|---:|
| 0 | 복원 목표 기준 자세 추종 오차(inclination tracking error) | `15°` |
| 1 | 피치 각속도(pitch rate) | `30°/s` |
| 2 | reset 원점 기준 종방향 위치 오차(longitudinal position error) | `4 m` |
| 3 | 직접 계산한 종방향 위치 변화율(longitudinal position rate) | `5 m/s` |
| 4 | reset 지상 고도 기준 고도 오차(altitude error) | `1.5 m` |
| 5 | 수직 속도(vertical velocity) | `5 m/s` |

정규화 결과는 `[-5, 5]`로 clip한다.

비교 실험용으로 `--policy-preset elevator-pd`도 남겨 뒀다. 측정한 제어 부호를 쓰는 고정 사전 항에 PPO 잔차를 더하는 방식이다. PPO 자체의 능력을 보려면 `none`을, controller assistance와 비교하려면 `elevator-pd`를 쓴다.

## TensorBoard 로그로 살펴본 변화

보상 함수와 로깅 방식(logging schema)이 작업 도중 바뀌었기 때문에, 서로 다른 세대의 `reward_mean`을 절대값으로 직접 비교하면 안 된다. 대신 episode 길이, 종료 종류, 방사 거리(radial distance)가 더 믿을 만한 기준이었다.

| 로그 | Episode | 평균 길이 | 평균 보상 | 주요 실패 |
|---|---:|---:|---:|---|
| 초기 | 28 | 125.9 | -527.4 | 저고도 20, x/y 경계 8 |
| 초기 | 126 | 177.1 | -530.0 | 저고도 112, x/y 경계 8 |
| 중간 | 175 | 285.0 | -578.4 | 저고도 61, 경계 6 |
| 순수 PPO | 228 | 218.2 | -29.2 | 원통 경계 충돌 123 |
| recovery | 329 | 226.1 | -102.1 | 원통 경계 충돌 148 |
| recovery-v2 검증 | 100 | 300.0 | +298.3 | 충돌 0 |
| cleanup 검증 | 100 | 300.0 | +298.2 | 충돌 0 |

드리프트가 뚜렷했던 순수 PPO 로그에서는 방사 거리의 95백분위수(p95, 측정값의 95%가 이 아래에 들어오는 값)가 `4.8082 m`, 최댓값이 `6.7119 m`였고 원통 경계 충돌이 123회였다. 복원 구조를 넣은 최종 선형 actor 검증에서는 같은 지표가 이렇게 바뀌었다.

- 100/100 episode가 모두 300 step 완료, 충돌 `0`
- radial mean `0.5505 m`, p95 `0.6851 m`, max `0.8103 m`
- recovery probe 최소 복원 여유 `0.2648`, symmetry error `0`, effective restoring fraction `1.0`
- 평가 보상/step `0.99333`, 평가 위치 오차 `0.5248 m`

symmetry error(좌우로 뒤집은 입력에 대한 action의 비대칭 정도)가 정확히 `0`이고, effective restoring fraction(원점으로 되돌리는 방향으로 제대로 작용한 비율)이 `1.0`이라는 건, 앞에서 설계한 대칭 actor가 실제로 좌우 대칭인 복원력을 그대로 내고 있다는 뜻이다. 이 값들은 앞서 말한 recovery probe로 학습 내내 확인했다.

## reset lifecycle과 통신은 별도의 상태 머신으로

엘리베이터 문제와 직접 맞닿아 있진 않지만, 학습을 끝까지 돌리려면 두 가지 배관도 손봐야 했다.

첫째는 reset lifecycle이다. RealFlight Hover Trainer의 reset은 일반적인 Gym reset보다 복잡하다. 추락 직후와 trainer reposition 직후의 state flag가 늘 명확하지 않아서, 추락 중인 자세를 새 원점으로 잘못 anchor하기 쉬웠다. 그래서 충분한 지상 고도, 저속, 작은 기체 각속도(body rate), 수직 자세 `90°`에서 `0.5°` 이내 같은 조건을 모두 만족할 때만 새 episode를 시작하도록 했다. 또 300 step에 도달한 것은 충돌이 아니라 time-limit truncation이므로, 같은 원점과 마지막 state를 유지한 채 step counter만 초기화하고 다음 구간을 이어 간다. truncation마다 굳이 trainer reset을 기다리지 않는다.

둘째는 RFLink 통신이다. RealFlight endpoint는 정상 응답을 준 뒤 keep-alive socket을 닫는데, 초기 구현은 이 정상적인 transport rollover와 실제 통신 장애를 같은 실패로 취급했다. 그래서 멀쩡한 연결 갱신까지 매 frame 경고와 backoff 로그로 남았다. 최종 구현은 TCP transport lifecycle과 주입한 UAV controller lifecycle을 분리한다. 한 번 이상 정상 응답을 받은 socket이 다음 요청 전에 닫히면 이를 `RFLinkStaleConnectionError`로 분류해 조용히 새 socket을 열고, 필요할 때만 controller를 재주입한다. 진짜 장애(연결/timeout 오류)에만 3초 timeout에 4회 재시도와 지수 backoff를 적용한다.

## 검증 결과와 남은 한계

정리하면, 최종 구현은 30,000-step 검증에서 100개 episode가 모두 300 step을 완료했고 충돌은 없었다. 학습한 `.pt` 체크포인트를 `play` 명령으로 바로 RealFlight에 올려 5개 episode를 다시 돌린 결과도 모두 300 step을 채웠고, 평균 보상은 약 `297.9`였다.

![같은 30,000-step 검증 학습의 두 지표를 손그림풍으로 나란히 그린 그래프. 왼쪽은 episode 길이가 내내 300 step 상한에 붙어 있는 모습, 오른쪽은 원점에서의 방사 거리가 6m 원통 경계 아래 1m 미만에서 움직이는 모습이다.](/assets/img/hover-pilot/elevator-validation-length-radial.png)

_30,000-step 검증 학습에서 뽑은 로그. 왼쪽은 모든 episode가 추락 없이 300 step 상한까지 채워 truncation으로만 끝났음을, 오른쪽은 방사 거리가 6m 원통 경계에 한참 못 미쳐(최대 `0.88 m`) 경계 충돌이 없었음을 보여준다._

다만 과장하지 않고 그대로 남겨 둘 지표도 있다. 마지막에 중복 계산과, 매 step 돌지만 쓰이지도 않던 critic 계산(hot path)을 걷어낸 cleanup 이후, 전체 radial p95는 `0.6851 → 0.7472 m`로 약 9.1% 늘었다. 검증 통과 기준(gate)인 `0.75 m` 이내였고 deterministic 평가 위치 오차는 오히려 소폭 좋아졌지만, 마지막 10,000 step만 보면 p95가 약 `0.772 m`였다. 더 긴 300,000-step 학습에서 이 추세를 계속 지켜볼 항목으로 남겼다.

지금 실험은 어디까지나 엘리베이터 전용이다. aileron, rudder, throttle까지 확장하면 각 축의 좌표계, 제어 가능성(controllability), 대칭성, 복원 목표를 축마다 따로 정의해야 한다. RealFlight 실행이 완전히 결정적이지도 않으므로, 한 번의 p95보다는 충돌률(collision rate)과 여러 seed의 결정적 평가, recovery probe를 함께 놓고 판단하는 편이 안전하다.

## 재현 명령

RL 의존성을 설치한다.

```bash
uv sync --extra rl
```

학습 전에 양수/음수 엘리베이터 pulse가 반대 피치 각속도를 만드는지부터 확인한다.

```bash
uv run hoverpilot-ppo diagnose-elevator \
  --elevator-fixed-throttle 0.55 \
  --pulse 0.1
```

30,000-step 순수 PPO 검증은 이렇게 돌린다.

```bash
uv run hoverpilot-ppo train \
  --control-mode elevator \
  --policy-preset none \
  --timesteps 30000 \
  --eval-episodes 5 \
  --save-path ppo_hoverpilot_elevator_validation.pt \
  --tensorboard-log-dir runs/hoverpilot-ppo-elevator-validation \
  --seed 44
```

학습한 체크포인트는 `play`로 바로 적용한다.

```bash
uv run hoverpilot-ppo play \
  --checkpoint ppo_hoverpilot_elevator_validation.pt \
  --episodes 5
```

TensorBoard는 `uv run tensorboard --logdir runs`로 띄우면 된다.

## 마무리

엘리베이터 하나로 좁혀 놓고 보니, PPO를 세운 힘은 알고리즘 튜닝보다 문제를 정책이 배우기 쉬운 모양으로 바꾸는 쪽에서 나왔다. 관측의 부호를 물리적으로 일관되게 맞추고, 위치 복원을 목표 프레임으로 옮기고, 학습해야 할 자유도를 대칭 actor로 줄이는 일이 대부분을 차지했다. 다음 단계는 이 구조를 나머지 축으로 넓히면서, 각 축이 같은 종류의 부호와 대칭 문제를 어떻게 다시 꺼내 놓는지 확인하는 일이다.

## 함께 보기

- GitHub 저장소: [hover-pilot](https://github.com/ruddyscent/hover-pilot) (태그: [elevator-ppo-realflight](https://github.com/ruddyscent/hover-pilot/releases/tag/elevator-ppo-realflight))
- [이전 글: PPO 학습에 TensorBoard 로깅 붙이기](/2026-07-25-hoverpilot-tensorboard-logging/)
