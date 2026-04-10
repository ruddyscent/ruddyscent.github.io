---
layout: post
title: HoverPilot — RC 비행기를 학습시키는 프로젝트를 시작했다
subtitle: RealFlight와 강화학습으로 hover를 배우는 실험
tags: [reinforcement-learning, rc-airplane, simulation, realflight, python]
cover-img: /assets/img/develop.jpeg
thumbnail-img: /assets/img/toy-plane.webp
share-img: /assets/img/develop.jpeg
author: 전경원
---

RC 비행에서 **hover**는 단순한 기술이 아니다.  기체를 수직으로 세운 상태에서 추력과 모든 조종면을 동시에 미세하게 제어해야 한다.

![RC 비행기가 수직 자세를 유지하며 hover를 수행하는 모습](/assets/img/hover-pilot/rc-airplane-hover-field.png)

사람 기준으로도 어려운 비행 기술이다.  그럼 이걸 **강화학습으로 학습시킬 수 있을까?**

## HoverPilot

이번에 시작한 프로젝트의 이름은 **HoverPilot**이다. 이 프로젝트의 목표는 다음과 같다.

> RealFlight 시뮬레이터 위에서 강화학습 에이전트가 RC 비행기의 hover를 스스로 학습하도록 만드는 것

시스템 구성은 SITL(Software In The Loop) 구조와 유사하다.  시뮬레이터에서 상태를 읽고, 학습된 에이전트가 action을 결정해서 조종 입력으로 전달하는 형태다.

![SITL 구조를 응용한 시스템 구성](/assets/img/hover-pilot/dev-setup-sketch.png)

## 왜 이걸 하는가

강화학습은 **게임 잘하는 AI**를 넘어서 **연속 제어 문제**에서 진짜 힘을 발휘한다. hover는 그 대표적인 문제다.

- 상태는 연속적이고
- action도 연속적이며
- 정답이 명확하지 않고
- 안정성이 핵심이다

이건 RL이 잘하는 영역이다.

## 접근 방식

전체 시스템은 Gymnasium 환경으로 래핑한다.

- 상태는 FlightAxis를 통해 읽고
- action은 Software Radio를 통해 전달한다
- reward는 hover 유지와 안정성을 기준으로 설계한다

그리고 처음부터 full control을 학습시키지 않는다.  RealFlight의 Hover Trainer 기능을 활용해서 조종면을 단계적으로 열어가는 방식으로 진행한다.

예를 들면:

1. Throttle only
2. Throttle + Elevator
3. Throttle + Rudder
4. Full control

이렇게 탐색 공간을 점진적으로 넓힌다.

## 앞으로 할 것

TODO 리스트는 다음과 같다.

- FlightAxis 인터페이스 구현
- Software Radio 구현
- Gymnasium 환경 완성
- baseline 정책 학습 (PPO 또는 SAC)

이후에는

- reward 튜닝
- curriculum learning
- 안정성 개선

순으로 진행할 예정이다.
