---
layout: post
title: HoverPilot - AI로 RC 비행기를 조종하는 프로젝트를 시작했다
subtitle: 강화학습으로 RealFlight 안에서 호버링을 해보자.
tags: [hoverpilot, reinforcement-learning, rc-airplane, simulation, realflight, python]
cover-img: /assets/img/develop.jpeg
thumbnail-img: /assets/img/toy-plane.webp
share-img: /assets/img/develop.jpeg
author: 전경원
---

고정익 RC 비행에는 **호버링(hover)**이라는 기술이 있습니다. 기체를 수직으로 세운 뒤 추력과 조종면을 미세하게 제어해 공중 한 지점에 정지하는 기동입니다.

![RC 비행기가 수직 자세를 유지하며 호버링을 수행하는 모습](/assets/img/hover-pilot/rc-airplane-hover-field.png)

호버링은 **불안정한 상태를 유지하는 기술**입니다. 조금만 균형이 무너지면 바로 추락으로 이어집니다. 그래서 RC 동호인도 쉽게 익히기 어려운 비행 기술입니다. 이를 **강화학습(Reinforcement Learning, RL)**으로 학습시킬 수 있을까요?

## HoverPilot

[강화학습](https://spinningup.openai.com/en/latest/)은 에이전트가 환경과 상호작용하며 보상(reward)을 최대화하는 방향으로 학습하는 기계학습의 하나입니다. 강화학습으로 RC 비행기의 호버링을 해내는 일이 **[HoverPilot](https://github.com/ruddyscent/hover-pilot)**의 목표입니다.

이러한 고난이도 비행은 실제 RC 비행기로 바로 연습하지는 않습니다. RC 비행기가 실제 비행기보다 저렴하긴 해도 수십, 수백 번의 시행착오를 반복하다가는 지갑이 거덜나는 것은 시간 문제이기 때문입니다.  그래서 시뮬레이터를 활용합니다.  수많은 시뮬레이터가 있지만 이번 프로젝트에서는 [RealFlight](https://www.realflight.com/)를 선택했습니다.

RealFlight는 RC 비행 시뮬레이터 중에서도 가장 널리 사용되는 상용 소프트웨어로, 정밀한 물리 엔진과 실제와 유사한 조종 감각을 제공합니다. 무엇보다 **RealFlight Link**라는 통신 인터페이스가 있어서 시뮬레이터 내부 상태를 외부 프로그램에서 읽고 조종 입력을 다시 전달할 수 있습니다.  이 기능 덕분에 RealFlight는 훈련 도구를 넘어 **강화학습 환경**이 됩니다.

이 RealFlight Link를 써서 전체 시스템을 [SITL(Software In The Loop)](https://ardupilot.org/dev/docs/sitl-simulator-software-in-the-loop.html) 구조로 구성합니다. 실제 하드웨어 없이 시뮬레이터로 소프트웨어를 검증하는 방식입니다. 에이전트는 시뮬레이터가 보내는 상태를 받아 행동을 결정하고, 그 결과를 다시 조종 입력으로 돌려보냅니다.

![SITL 구조를 응용한 시스템 구성](/assets/img/hover-pilot/dev-setup-sketch.png)

## 강화학습이 잘 해낼 수 있을까?

호버링은 불안정한 시스템을 안정되게 유지하는 문제입니다. 기울어지는 방향과 정도를 파악하고 그 불안정성을 상쇄하는 힘을 가하면서 목표한 자세를 유지해야 합니다.

이 과정은 한 번의 판단으로 끝나지 않습니다. 아주 작은 기울어짐도 시간이 지나면서 점점 커지고 그걸 서둘러 보정하지 않으면 바로 무너집니다. 호버링은 **연속적인 피드백 루프(feedback loop)**를 얼마나 잘 유지하느냐의 문제입니다.

이 구조는 강화학습의 문제 정의와 거의 일치합니다. 에이전트는 매 순간 상태(**state**)를 관찰하고 그에 맞춰 행동(**action**)을 선택하며, 그 결과가 목표에 얼마나 가까운지에 따라 보상을 받습니다. 이 과정을 반복하면서 점점 더 안정적인 정책을 만들어갑니다.

## 접근 전략

RealFlight는 자체적으로 RealFlight Link라는 양방향 통신 인터페이스를 제공합니다. 이를 이용하면 시뮬레이터 내부 상태를 직접 읽을 수 있어 별도의 이미지 분석 없이 정확한 비행 정보를 얻습니다. 양방향 통신이기에 비행기의 각 구동부로 제어 신호도 보낼 수 있습니다.

이 인터페이스 위에 [Gymnasium](https://gymnasium.farama.org/)과 같은 구조를 얹어, 표준적인 강화학습 환경으로 재구성합니다.

문제는 탐색 공간이 너무 크다는 점입니다. 호버링은 네 개의 제어 입력이 동시에 얽혀 있는 문제라, 처음부터 전체를 학습시키면 거의 아무것도 배우지 못한 채 추락만 반복합니다.

그래서 RealFlight의 **Airplane Hover Trainer**가 제공하는 훈련 방식을 그대로 활용합니다. 조종면을 하나씩 열어가며 난이도를 점진적으로 높여갑니다.  예를 들면:

1. Throttle only
2. Throttle + Elevator
3. Throttle + Rudder
4. Full control

![RC 채널 제한 옵션을 보여주는 RealFlight 컨트롤 메뉴](/assets/img/hover-pilot/channel-restrictions.png)

이렇게 탐색 공간을 단계적으로 넓혀가면서 에이전트가 안정적인 정책을 더 빠르게 학습하기를 기대해 봅니다.

## 앞으로 할 것

남은 TODO는 다음과 같습니다.

- RealFlight Link 인터페이스를 통한 비행 정보 파악
- RealFlight Link 인터페이스를 통한 비행면 및 모터 제어
- Gymnasium 환경 구성
- PPO 또는 SAC를 이용한 강화학습
