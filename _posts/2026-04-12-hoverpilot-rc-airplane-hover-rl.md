---
layout: post
title: HoverPilot — AI로 RC 비행기를 조종하는 프로젝트를 시작했다
subtitle: 강화학습으로 RealFlight 안에서 호버링을 해보자.
tags: [reinforcement-learning, rc-airplane, simulation, realflight, python]
cover-img: /assets/img/develop.jpeg
thumbnail-img: /assets/img/toy-plane.webp
share-img: /assets/img/develop.jpeg
author: 전경원
---

고정익 RC 비행에서 **호버링(hover)**이라는 기술이 있다.  기체를 수직으로 세운 상태에서 추력과 조종면을 미세하게 제어해 공중의 한 지점에 정지하는 기동을 말한다.

![RC 비행기가 수직 자세를 유지하며 호버링을 수행하는 모습](/assets/img/hover-pilot/rc-airplane-hover-field.png)

문제는 이게 **“불안정한 상태를 억지로 유지하는 기술”**이라는 점이다.  조금만 균형이 무너지면 바로 추락으로 이어진다. 그래서 RC 동호인 중에서도 웬만한 고인물이 아니고서는 흉내내기도 어려운 비행 기술이다.  그렇다면 이걸 **강화학습(Reinforcement Learning, RL)**으로 학습시킬 수 있을까?

## HoverPilot

[강화학습(Reinforcement Learning, RL)](https://spinningup.openai.com/en/latest/)은 에이전트가 환경과 상호작용하며 보상(reward)을 최대화하는 방향으로 학습하는 기계학습의 하나이다.  이 강화학습을 이용해서 RC 비행기의 호버링을 해내는 것이 **[HoverPilot](https://github.com/ruddyscent/hover-pilot)**의 목표이다.

실제로도 이러한 고난이도 비행을 실제 RC 비행기로 바로 연습하지는 않는다. RC 비행기가 실제 비행기보다 저렴하긴 해도 수천, 수만 번의 시행착오를 반복하는 것은 현실적으로 불가능하기 때문이다.  그래서 시뮬레이터를 활용한다.  수많은 시뮬레이터가 있지만, 이번 프로젝트에서는 [RealFlight](https://www.realflight.com/)를 선택했다. 

RealFlight는 RC 비행 시뮬레이터 중에서도 가장 널리 사용되는 상용 소프트웨어로, 정밀한 물리 엔진과 실제와 유사한 조종 감각을 제공한다. 또한 **RealFlight Link**라는 통신 인터페이스를 제공하여, 시뮬레이터 내부 상태를 외부 프로그램에서 읽고, 조종 입력을 다시 전달할 수 있다.  이 기능 덕분에 RealFlight를 단순한 훈련 도구가 아니라, **강화학습 환경으로 확장**하는 것이 가능하다.

시스템 구성은 [SITL(Software In The Loop)](https://ardupilot.org/dev/docs/sitl-simulator-software-in-the-loop.html) 구조와 유사하다.  시뮬레이터에서 상태를 읽고, 학습된 에이전트가 행동을 결정해서 조종 입력으로 전달하는 형태이다.  **HoverPilot**은 비행기가 호버링을 안정적으로 유지하도록 학습하는 것을 목표로 한다.

> SITL은 실제 하드웨어 없이, 시뮬레이터를 통해 소프트웨어를 테스트하는 방식이다.  

![SITL 구조를 응용한 시스템 구성](/assets/img/hover-pilot/dev-setup-sketch.png)

## 강화학습이 잘 해낼 수 있을까?

호버링은 불안정한 시스템을 안정되게 유지하는 문제이다. 불안정성이 커지는 방향의 움직임을 파악해서(**state**), 이 불안정성을 상쇄하는 힘을 가하여(**action**), 최대한 목표하는 상태를 유지(**reward**)하는 방식으로 정책을 만들어가야 한다.  

이 과정은 한 번의 판단으로 끝나지 않는다. 아주 작은 기울어짐도 시간이 지나면서 점점 커지고, 그걸 서둘러 보정하지 않으면 바로 무너진다. 호버링은 **연속적인 피드백 루프(feedback loop)**를 얼마나 잘 유지하느냐의 문제다.  

이 구조는 강화학습의 문제 정의와 거의 일치한다. 에이전트는 매 순간 상태를 관찰하고, 행동을 선택하며, 그 결과로 보상을 받는다. 그리고 이 과정을 반복하면서 점점 더 안정적인 정책을 만들어간다. 

## 접근 전략

RealFlight는 자체적으로 RealFlight Link라는 양방향 통신 인터페이스를 제공한다. 이를 이용하면 시뮬레이터 내부 상태를 직접 읽을 수 있어, 별도의 이미지 분석 없이 정확한 비행 정보를 얻을 수 있다. 양방향 통신이기에 비행기의 각 구동부에 대한 제어 신호도 보낼 수 있다. 

이 인터페이스 위에 [Gymnasium](https://gymnasium.farama.org/)과 같은 구조를 얹어 강화학습 환경으로 재구성한다. 에이전트는 상태(state)를 입력받아 행동(action)을 선택하고, 그 결과로 보상(reward)을 받는 방식으로 학습이 이루어진다.

문제는 탐색 공간이 너무 크다는 점이다. 호버링은 네 개의 조종 축이 동시에 얽혀 있는 문제라, 처음부터 전체를 학습시키면 거의 아무것도 배우지 못한 채 추락만 반복하게 될 것이다.  

그래서 RealFlight의 **Airplane Hover Trainer**가 제공하는 훈련 방식을 그대로 활용한다. 조종면을 단계적으로 열어가며 난이도를 점진적으로 높여간다.  예를 들면:

1. Throttle only
2. Throttle + Elevator
3. Throttle + Rudder
4. Full control

![RC 채널 제한 옵션을 보여주는 RealFlight 컨트롤 메뉴](/assets/img/hover-pilot/channel-restrictions.png)

이러한 방식으로 학습이 진행되면서 탐색 공간을 단계적으로 확장하여, 에이전트가 안정적인 정책을 더 빠르게 학습하기를 기대해 본다.

## 앞으로 할 것

TODO 리스트는 다음과 같다.

- RealFlight Link 인터페이스를 통한 비행 정보 파악
- RealFlight Link 인터페이스를 통한 비행면 및 모터 제어
- Gymnasium 환경 구성
- PPO 또는 SAC를 이용한 강화학습