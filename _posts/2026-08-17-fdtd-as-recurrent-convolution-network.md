---
layout: post
title: FDTD를 펼쳐 보면 RNN이 된다
subtitle: 공간 차분은 합성곱이고, 시간 갱신은 순환 구조다
tags: [fdtd, rnn, cnn, deep-learning, pytorch, numerical-methods]
cover-img: /assets/img/develop.jpeg
thumbnail-img: /assets/img/fdtd-sinc-field.webp
share-img: /assets/img/develop.jpeg
author: 전경원
mathjax: true
---

FDTD(Finite-Difference Time-Domain, 유한 차분 시간 영역법)는 1960년대부터 전산 전자기학에서 써 온 수치해석 기법이다([Yee, 1966](https://doi.org/10.1109/TAP.1966.1138693)). RNN(Recurrent Neural Network, 순환 신경망)은 1990년대부터 시계열을 다루는 데 쓰여 온 신경망 구조다([Elman, 1990](https://doi.org/10.1207/s15516709cog1402_1)). 출발점과 목적은 다르지만 계산 방식은 닮았다. FDTD의 시간 루프를 펼쳐 놓으면 합성곱 연산을 반복하는 RNN과 같은 형태가 된다. 2018년 홍콩대 연구진은 이 구조를 이용해 FDTD 결과를 예측하는 신경망을 학습했다. 이후에는 FDTD의 계수를 신경망의 고정 가중치로 해석하는 연구가 나왔다. 물리 법칙이 가중치의 값을 이미 정해 놓았으니 학습할 필요가 없다는 설명이다.

이러한 연관성은 FDTD 알고리즘에서 유용하게 사용할 수 있다. FDTD를 PyTorch나 TensorFlow로 작성하면 GPU 병렬화와 자동 미분을 바로 사용할 수 있기 때문이다. 합성곱과 순환으로 표현할 수 있는 계산이라 신경망 프레임워크의 연산 구조와도 잘 맞는다.

## FDTD의 한 스텝

맥스웰 방정식의 패러데이 법칙과 앙페르-맥스웰 법칙에 따르면 전기장의 시간 변화는 자기장의 공간 미분으로, 자기장의 시간 변화는 전기장의 공간 미분으로 정해진다. FDTD는 이 미분을 Yee 격자 위의 유한 차분으로 근사하고, 전기장과 자기장을 반 스텝씩 어긋나게 갱신한다. 1차원에서는 한 스텝을 다음 두 식으로 쓸 수 있다.

$$H^{\,n+1/2}_{i+1/2} = H^{\,n-1/2}_{i+1/2} - \frac{\Delta t}{\mu\,\Delta x}\left(E^{\,n}_{i+1} - E^{\,n}_{i}\right)$$

$$E^{\,n+1}_{i} = E^{\,n}_{i} - \frac{\Delta t}{\varepsilon\,\Delta x}\left(H^{\,n+1/2}_{i+1/2} - H^{\,n+1/2}_{i-1/2}\right)$$

한 스텝에 필요한 연산은 크게 두 가지다. 이웃한 격자점의 장을 빼는 **공간 차분**과, 그 결과로 다음 시각의 장을 계산하는 **시간 갱신**이다. 공간 차분은 합성곱에, 시간 갱신을 반복하는 구조는 RNN에 대응한다.

## 공간 미분은 합성곱이다

예를 들어 $$E^{\,n}_{i+1} - E^{\,n}_{i}$$는 커널 $$[-1, +1]$$을 격자에 적용하는 1차원 합성곱이다. 여기서 커널은 입력을 훑으며 곱셈과 덧셈을 수행하는 작은 가중치 배열을 말한다. 2차원 TM(Transverse Magnetic, 횡자기) 모드처럼 자기장이 진행 평면에 놓이고 전기장을 $$E_z$$ 성분 하나로 나타낼 수 있는 경우에도 회전 연산자($$\nabla \times$$)는 작은 커널 몇 개로 계산된다. CNN(Convolutional Neural Network, 합성곱 신경망)의 합성곱 층과 같은 연산이다.

![다섯 전기장 격자 중 이웃한 두 샘플의 공간 차분으로 H의 i+1/2 성분을 갱신하는 그림](/assets/img/fdtd-rnn/fdtd-rnn-convolution-five-samples-balanced-fields.png)

보통의 CNN과 다른 점은 커널을 학습하지 않는다는 것이다. $$[-1, +1]$$과 계수 $$\Delta t / (\mu\,\Delta x)$$는 물리 법칙과 격자 간격으로 정해진다. 따라서 이 합성곱 층의 가중치는 처음부터 고정돼 있으며 역전파로 갱신되지 않는다.

## 시간 전진은 순환이다

시각 $$n{+}1$$의 장은 시각 $$n$$의 장에서 계산되고, 시각 $$n$$의 장은 그 앞 시각의 장에서 계산된다. RNN도 이전 은닉 상태를 받아 다음 은닉 상태를 만든다. RNN 셀은 보통 다음과 같이 쓸 수 있다.

$$\mathbf{h}_{n+1} = \sigma\!\left(\mathbf{W}\,\mathbf{h}_{n} + \mathbf{U}\,\mathbf{x}_{n}\right)$$

FDTD에서는 격자 전체의 장 $$(E,\,H)$$가 은닉 상태 $$\mathbf{h}_n$$에 해당한다. 입력 $$\mathbf{x}_n$$은 그 시각에 주입되는 파원 $$s_n$$이고, 셀 함수는 앞에서 본 두 갱신식이다. 관측점에서 읽은 장은 출력 $$y_n$$이 된다.

![FDTD 시간 스텝을 RNN 셀로 펼친 그림](/assets/img/fdtd-rnn/fdtd-rnn-unroll.svg)

시간 루프를 펼쳐 그리면 셀 하나가 FDTD의 한 스텝에 대응한다. 격자 전체의 장은 은닉 상태로 다음 셀에 전달되고, 각 스텝에서는 같은 갱신식을 다시 쓴다. RNN이 모든 시점에 같은 가중치를 공유하는 것과 같다.

## 신경망에게 FDTD를 배우게 하기

2018년 홍콩대의 He Ming Yao와 Li Jun Jiang은 이 구조적 유사성을 실제 학습 문제로 다뤘다([Yao & Jiang, 2018](https://doi.org/10.1109/APUSNCURSINRSM.2018.8608745)). 이들은 시간축에 2차 차분을 적용한 2차원 스칼라 파동 방정식을 사용했다. 격자 전체를 한꺼번에 갱신하는 연산은 CNN으로, 그 연산을 시간에 따라 반복하는 과정은 RNN으로 모델링했다. FDTD 시뮬레이션 결과를 정답 데이터로 썼다.

시간축에 2차 차분을 적용한 FDTD는 다음 장을 계산할 때 직전 두 시각의 장이 필요하다. RNN에서는 한 스텝 전의 장을 은닉 상태에 보관하므로, 현재 시각의 국소 영역만 입력하면 된다.

![우리 2D 스칼라 파동 FDTD로 학습한 대리모형 비교. 위 행은 파동장, 아래 행은 정답과의 차이 지도다. 직전 프레임을 기억하면 정답을 거의 그대로 재현하고(차이 지도가 거의 0), 현재 프레임만 쓰면 파면을 따라 상대 오차 12%가 드러난다](/assets/img/fdtd-rnn/fdtd-rnn-fdtd-vs-surrogate.png)

다만 이 방법은 FDTD를 그대로 구현한 것이 아니라, 신경망이 FDTD 결과를 근사하도록 학습한 것이다. 학습에 없던 파원 위치에서는 RNN 모델의 평균 상대 오차가 15%까지 올랐고, 같은 문제를 학습한 CNN 모델은 3% 미만의 오차를 보였다. 구조가 닮았다는 점은 보여 줬지만 정확도는 학습 데이터와 모델에 달려 있었다.

## 상수 신경망

FDTD를 그대로 신경망 연산으로 옮기면 학습 과정이 필요 없다. 유한 차분 커널과 시간 갱신 계수는 물리 법칙과 격자가 정하므로 데이터로 맞출 가중치가 없기 때문이다. 칭화대 연구진은 2023년 이 구조를 RCNN(Recurrent Convolution Neural Network, 순환 합성곱 신경망)으로 정리했다([Guo et al., 2023](https://read.nxtbook.com/ieee/antennas_propagation/antennas_propagation_feb_2023/electromagnetic_modeling_usin.html)). FDTD의 각 요소는 다음과 같이 대응한다.

| RNN / CNN 구성 요소 | FDTD의 대응물 |
|---|---|
| 은닉 상태 $$\mathbf{h}_n$$ | 격자 전체의 장 $$(E,\,H)$$ |
| 셀(재귀 함수) | 한 시간 스텝의 갱신식 |
| 입력 $$\mathbf{x}_n$$ | 시각 $$n$$의 파원 |
| 출력 $$\mathbf{y}_n$$ | 관측점의 장 |
| 가중치 $$\mathbf{W},\ \mathbf{U}$$ | 물리 계수와 차분 커널(고정) |
| 합성곱 커널 | 유한 차분 스텐실 |
| 활성화 함수 $$\sigma$$ | 항등 함수(선형) |

보통의 딥러닝 모델과 다른 부분은 가중치와 활성화 함수다. 가중치는 물리 법칙이 정하며 학습 중에는 바뀌지 않는다. 여기서 다루는 맥스웰 방정식은 선형이므로 활성화 함수에는 항등 함수를 쓴다. PML(Perfectly Matched Layer, 완전 흡수 경계층)을 추가하면 은닉 상태에 보조 변수가 더 들어가지만 순환 구조는 유지된다.

## 그래서 무엇이 좋은가

FDTD를 신경망으로 표현해도 계산 결과는 달라지지 않는다. 대신 구현할 때 PyTorch와 TensorFlow의 GPU 연산과 자동 미분을 활용할 수 있다.

가장 직접적인 이점은 GPU 병렬화다. CUDA 커널을 직접 작성하지 않아도 PyTorch나 TensorFlow가 연산을 GPU에서 실행한다. 칭화대 연구진의 구현은 CPU 기반 병렬 FDTD보다 약 5.7배 빨랐고, 전통적인 FDTD와 같은 정확도를 보였다. 유한 차분은 같은 작은 연산을 격자 전체에서 반복하므로 GPU 병렬 처리에 잘 맞는다.

자동 미분은 역산과 설계에 유용하다. FDTD를 계산 그래프로 작성하면 시뮬레이션 결과를 물질 분포나 파원 같은 입력 파라미터에 대해 미분할 수 있다. 관측된 장으로부터 물질 분포를 추정하거나 원하는 방사 패턴을 만드는 문제를 경사 기반 최적화로 풀 수 있다. 순전파에서는 장을 계산하고, 역전파에서는 목표와 결과의 차이를 줄이는 방향으로 입력 파라미터를 갱신한다.

알려진 물리 법칙을 신경망 구조에 직접 넣을 수도 있다. 네트워크 전체를 블랙박스로 학습하는 대신, 물리로 계산할 수 있는 부분은 고정하고 알 수 없는 부분만 데이터로 학습한다.

## 파동 매질과 미분 가능한 FDTD

파동 매질 자체를 RNN으로 사용한 연구도 있다. 2019년 휴스 연구진은 FDTD로 이산화한 파동 방정식을 RNN의 상태 갱신식으로 해석했다([Hughes et al., 2019](https://doi.org/10.1126/sciadv.aay6946)). FDTD를 신경망 하드웨어에서 실행한 것이 아니라, 파동이 전파되는 매질에 RNN의 연산을 맡겼다. 수치 시뮬레이션에서는 역설계한 불균일 매질에 음성 파형을 입력해 영어 모음 세 가지를 분류했다.

![영어 모음 ae, ei, iy의 파형을 불균일 파동 매질에 넣고 세 출력 탐침으로 분류하며 매질 분포를 경사 기반으로 최적화하는 논문 도식](/assets/img/fdtd-rnn/hughes-2019-fig2.webp)

*Hughes et al., ["Wave physics as an analog recurrent neural network"](https://doi.org/10.1126/sciadv.aay6946), Fig. 2에서 잘라 사용. [CC BY-NC 4.0](https://creativecommons.org/licenses/by-nc/4.0/).*

FDTD에 자동 미분을 적용한 연구도 이어졌다. 각 스텝을 계산 그래프에 기록하면 전체 시간 전개를 역전파해 물질 분포나 파원에 대한 기울기를 구할 수 있다. 휴스턴 연구진은 시간 영역 전자기 해석을 미분 가능 프로그래밍 플랫폼으로 구현해 같은 코드로 순전파 시뮬레이션과 역산을 수행했다([Hu et al., 2022](https://ieeexplore.ieee.org/document/9496209/)). 광자 소자 설계에서도 미분 가능 FDTD와 FDFD(Finite-Difference Frequency-Domain, 유한 차분 주파수 영역법)를 사용한다. 자동 미분과 기존 수반법(adjoint)을 결합해 상용 FDTD 솔버를 계산 그래프에 넣는 방법도 제안됐다([Luce et al., 2024](https://doi.org/10.1088/2632-2153/ad5411)).

![미분 가능한 기하와 후처리 사이에 블랙박스 PDE 솔버를 넣고 순전파와 역전파를 연결한 논문 도식](/assets/img/fdtd-rnn/luce-2024-fig1.webp)

*Luce et al., ["Merging automatic differentiation and the adjoint method for photonic inverse design"](https://doi.org/10.1088/2632-2153/ad5411), Fig. 1에서 잘라 사용. [CC BY 4.0](https://creativecommons.org/licenses/by/4.0/).*

![경사 기반 최적화 전과 후의 마이크로 LED 단면. 출광 구조의 윗면 형상이 달라졌다](/assets/img/fdtd-rnn/luce-2024-fig4a.webp)

*Luce et al. (2024), Fig. 4(a)의 최적화 전·후 구조만 잘라 사용. [CC BY 4.0](https://creativecommons.org/licenses/by/4.0/).*

자동 미분에는 메모리 비용이 따른다. RNN의 BPTT(Backpropagation Through Time, 시간 역전파)는 각 시점의 은닉 상태를 저장해야 한다. FDTD 역산도 수천 스텝의 장을 보관해야 하므로 같은 문제가 생긴다. 이때는 RNN 학습에 쓰는 기울기 체크포인팅을 적용해 일부 상태만 저장하고, 필요한 값은 역전파 중에 다시 계산할 수 있다.

## 마무리

FDTD의 공간 차분은 고정 커널의 합성곱으로, 시간 갱신은 은닉 상태를 전달하는 순환 구조로 표현할 수 있다. 선형 맥스웰 방정식에서는 활성화 함수가 항등 함수가 되고, 물리 계수는 상수 가중치가 된다. 이 대응을 이용하면 FDTD의 시간 루프를 순환 합성곱 신경망으로 구현할 수 있다.

PyTorch나 TensorFlow로 구현한 FDTD는 CPU용과 GPU용 코드를 따로 작성하지 않고 두 환경에서 같은 갱신식을 사용한다. 공간 차분을 합성곱으로 작성하면 GPU 병렬화가 따라오고, 시간 전개를 계산 그래프에 담으면 물질 분포와 파원에 대한 기울기까지 얻는다. 하나의 갱신식으로 순전파 시뮬레이션과 역산을 함께 다룬다.
