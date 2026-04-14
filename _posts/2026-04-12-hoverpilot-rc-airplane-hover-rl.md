---
layout: post
title: HoverPilot — RC 비행기를 학습시키는 프로젝트를 시작했다
subtitle: RealFlight와 강화학습으로 hover를 배우는 실험
tags: [reinforcement-learning, rc-airplane, simulation, realflight, python]
cover-img: /assets/img/develop.jpeg
thumbnail-img: /assets/img/toy-plane.png
share-img: /assets/img/develop.jpeg
author: 전경원
---

고정익 RC 비행에서 **호버링(hover)**이라는 기술이 있다.  기체를 수직으로 세운 상태에서 추력과 모든 조종면을 미세하게 제어해 공중의 한 지점에 정지하는 기동을 말한다.

![RC 비행기가 수직 자세를 유지하며 호버링을 수행하는 모습](/assets/img/hover-pilot/rc-airplane-hover-field.png)

RC 동호인 중에서도 웬만한 고인물이 아니고서는 흉내내기도 어려운 비행 기술이다.  그럼 이걸 **강화학습(Reinforcement Learning, RL)**으로 학습시킬 수 있을까?

> [강화학습](https://spinningup.openai.com/en/latest/)은 에이전트가 환경과 상호작용하며 보상(reward)을 최대화하는 방향으로 학습하는 기계학습 방법이다.

## HoverPilot

이번에 시작한 프로젝트의 이름은 **HoverPilot**이다. 이 프로젝트의 목표는 다음과 같다.

> [RealFlight](https://www.realflight.com/) 시뮬레이터 위에서 강화학습 에이전트가 RC 비행기의 hover를 스스로 학습하도록 만드는 것

시스템 구성은 [SITL(Software In The Loop)](https://ardupilot.org/dev/docs/sitl-simulator-software-in-the-loop.html) 구조와 유사하다.  시뮬레이터에서 상태(state)를 읽고, 학습된 에이전트가 행동(action)을 결정해서 조종 입력으로 전달하는 형태이다.

> SITL은 실제 하드웨어 없이, 시뮬레이터를 통해 소프트웨어를 테스트하는 방식이다.  

![SITL 구조를 응용한 시스템 구성](/assets/img/hover-pilot/dev-setup-sketch.png)

## 강화학습이 잘 해낼 수 있을까?

호버링은 불안정한 시스템을 안정되게 유지하는 문제이다. 불안정성이 커지는 방향의 움직임을 파악해서(**state**), 이 불안정성을 상쇄하는 힘을 가하여(**action**), 최대한 목표하는 상태를 유지(**reward**)하는 방식으로 정책을 만들어가야 한다.  

이 과정은 한 번의 판단으로 끝나지 않는다. 아주 작은 기울어짐도 시간이 지나면서 점점 커지고, 그걸 서둘러 보정하지 않으면 바로 무너진다. 호버링은 **연속적인 피드백 루프(feedback loop)**를 얼마나 잘 유지하느냐의 문제다.  

이 구조는 강화학습의 문제 정의와 거의 일치한다. 에이전트는 매 순간 상태를 관찰하고, 행동을 선택하며, 그 결과로 보상을 받는다. 그리고 이 과정을 반복하면서 점점 더 안정적인 정책을 만들어간다. 

## 접근 전략

RealFlight는 자체적으로 RealFlight Link라는 양방향 통신 인터페이스를 제공한다. 이를 이용하면 시뮬레이터 내부 상태를 직접 읽을 수 있어, 별도의 이미지 분석 없이 정확한 비행 정보를 얻을 수 있다. 양방향 통신이기에 비행기의 각 구동부에 대한 제어 신호도 보낼 수 있다. 

이 인터페이스 위에 [Gymnasium](https://gymnasium.farama.org/) 구조를 얹어 강화학습 환경으로 재구성한다. 에이전트는 상태(state)를 입력받아 행동(action)을 선택하고, 그 결과로 보상(reward)을 받는 방식으로 학습이 이루어진다.

문제는 탐색 공간이 너무 크다는 점이다. 호버링은 네 개의 조종 축이 동시에 얽혀 있는 문제라, 처음부터 전체를 학습시키면 거의 아무것도 배우지 못한 채 추락만 반복하게 된다.  

그래서 RealFlight의 **Airplane Hover Trainer**가 제공하는 훈련 방식을 그대로 활용한다. 조종면을 단계적으로 열어가며 난이도를 점진적으로 높여간다.

예를 들면:

1. Throttle only
2. Throttle + Elevator
3. Throttle + Rudder
4. Full control

학습이 진행되면서 점차 자유도를 늘린다. 이렇게 탐색 공간을 단계적으로 확장하면, 에이전트가 안정적인 정책을 더 빠르게 학습할 수 있다.

## 앞으로 할 것

TODO 리스트는 다음과 같다.

- RealFlight Link 인터페이스를 통한 비행 정보 파악
- RealFlight Link 인터페이스를 통한 비행면 및 모터 제어
- Gymnasium 환경 구성
- PPO 또는 SAC를 이용한 강화학습