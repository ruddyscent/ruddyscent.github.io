---
layout: post
title: IonosphereFDTD — 불규칙한 구면 격자를 NumPy 배열로 계산하기
subtitle: 오각형과 육각형도 접속 관계를 표로 만들면 한 번에 계산할 수 있다
tags: [fdtd, simulation, ionosphere, numpy, python, numerical-methods]
cover-img: /assets/img/develop.jpeg
thumbnail-img: /assets/img/ionosphere-fdtd/ionosphere-fdtd-globe-simple.webp
share-img: /assets/img/develop.jpeg
author: 전경원
mathjax: true
---

2편에서 만든 측지 FDTD의 한 스텝은 값의 차이, 방향을 맞춘 순환, 길이와 면적을 반영한 나눗셈, 장의 갱신으로 이뤄진다. 수식만 보면 단순하지만 그대로 파이썬 반복문으로 옮기면 이야기가 달라진다. 모든 모서리와 면, 방사층을 시간 스텝마다 하나씩 훑는 동안 계산보다 파이썬 인터프리터가 더 바빠진다.

IonosphereFDTD의 NumPy 구현은 격자의 연결 관계를 **정수 배열**로 바꾼다. 그러면 불규칙한 구면 격자도 결국 "어느 값을 가져와 어떤 부호로 더할 것인가"라는 배열 연산으로 정리된다. NumPy가 지향하는 배열 프로그래밍처럼, 원소별 파이썬 코드를 컴파일된 다차원 배열 연산으로 밀어 넣는 것이다([Harris et al., 2020](https://doi.org/10.1038/s41586-020-2649-2)).

## 네 개의 장, 네 개의 배열

구면 위 꼭짓점 수를 $N_v$, 모서리 수를 $N_e$, 삼각형 면 수를 $N_f$, 방사 방향 셀 수를 $N_r$라고 하자. 네 장의 배열 모양은 이렇다.

```text
Er: (Nv, Nr + 1)
Ht: (Ne, Nr + 1)
Et: (Ne, Nr)
Hr: (Nf, Nr)
```

첫 축은 언제나 구면 위의 대상이다. $E_r$는 꼭짓점, $H_t$와 $E_t$는 모서리, $H_r$는 삼각형 면에 놓인다. 두 번째 축은 방사 방향 위치다. $E_r$와 $H_t$가 $N_r+1$개의 TM-r 평면을 차지하고, 그 사이 $N_r$개의 TE-r 층에 $E_t$와 $H_r$가 놓인다.

![방사 방향으로 엇갈린 TM-r 평면과 TE-r 층](/assets/img/ionosphere-fdtd/numpy-field-radial.svg)

![네 장을 구면 대상과 방사 위치의 두 축으로 저장한 NumPy 배열](/assets/img/ionosphere-fdtd/numpy-field-storage.svg)

이 축 순서 덕분에 한 번의 인덱싱으로 모든 방사층을 함께 처리한다. 예를 들어 방향이 정해진 모서리의 양 끝에서 $E_r$ 차이를 구하는 코드는 한 줄이다.

```python
er_head_minus_tail = er[edges[:, 1]] - er[edges[:, 0]]
```

`edges[:, 0]`은 꼬리 꼭짓점, `edges[:, 1]`은 머리 꼭짓점의 번호다. `er`가 `(Nv, Nr + 1)`이면 결과는 `(Ne, Nr + 1)`이 된다. 모서리마다, 방사층마다 돌던 두 겹 반복문이 NumPy의 고급 인덱싱과 뺄셈 하나로 바뀐다.

![원소마다 머리와 꼬리 값을 읽는 파이썬 반복문](/assets/img/ionosphere-fdtd/numpy-loop-scalar.svg)

![머리와 꼬리 행 전체를 모아 한 번에 빼는 NumPy 벡터화](/assets/img/ionosphere-fdtd/numpy-loop-vectorized.svg)

여기서 벡터화가 복사를 없애거나 CPU 코어를 자동으로 모두 쓴다는 뜻은 아니다. 위 식은 머리와 꼬리 행을 모은 임시 배열을 만든다. 이득은 격자 크기만큼 반복되는 파이썬 호출을 없애고 실제 산술을 NumPy 내부의 컴파일된 루프로 넘긴 데서 온다.

## 삼각형의 순환은 gather–sign–reduce

삼각형 면에는 언제나 모서리가 세 개 있다. 격자를 만들 때 다음 두 표를 함께 저장해 두면 면을 따라 도는 순환을 한 번에 구할 수 있다.

- `face_edges[face, corner]`: 각 면의 세 모서리 번호
- `face_edge_signs[face, corner]`: 면의 반시계 방향과 모서리 방향이 같으면 $+1$, 반대면 $-1$

```python
selected = edge_values[face_edges]
circulation = np.sum(
    selected * face_edge_signs[..., None],
    axis=1,
)
```

방사층이 포함된 `edge_values`를 넣으면 `selected`의 모양은 `(Nf, 3, Nr)`가 된다. 모서리를 모으고(gather), 방향 부호를 곱하고(sign), 세 항을 더한다(reduce). 2편에서 적분형 맥스웰 방정식으로 설명한 순환이 그대로 배열 연산이 된 셈이다.

## 오각형 하나 때문에 반복문으로 돌아가야 할까

쌍대 셀은 대부분 육각형이지만 12개는 오각형이다. 셀마다 이웃 수가 다르니 가변 길이 목록을 떠올리기 쉽다. 하지만 최대 이웃 수가 6으로 작고 고정되어 있다는 점을 이용하면 더 단순하게 만들 수 있다.

```text
vertex_edges:      (Nv, 6)
vertex_edge_signs: (Nv, 6)
```

육각형은 여섯 칸을 모두 쓰고 오각형은 남는 여섯 번째 칸의 부호를 0으로 둔다. 그 칸의 모서리 번호가 무엇이든 0을 곱하므로 결과에는 영향을 주지 않는다.

```python
sign_shape = (Nv,) + (1,) * (edge_values.ndim - 1)
result = edge_values[vertex_edges[:, 0]].copy()
result *= vertex_edge_signs[:, 0].reshape(sign_shape)

for slot in range(1, 6):
    result += edge_values[vertex_edges[:, slot]] * (
        vertex_edge_signs[:, slot].reshape(sign_shape)
    )
```

여기에도 반복문이 보이지만 격자나 방사층의 크기만큼 도는 반복문은 아니다. 딱 여섯 번만 돌며 매번 모든 꼭짓점과 모든 방사층을 한꺼번에 계산한다. 격자가 커져도 파이썬 반복 횟수는 그대로다.

![삼각형의 세 모서리를 gather–sign–reduce 연산으로 바꾸는 과정](/assets/img/ionosphere-fdtd/numpy-circulation-face.svg)

![오각형과 육각형을 여섯 개의 고정 슬롯으로 계산하는 과정](/assets/img/ionosphere-fdtd/numpy-circulation-dual.svg)

`np.add.at`으로 각 모서리 값을 양 끝 셀에 흩뿌리는 방식도 옳다. 실제 격자 객체에는 이해하기 쉬운 기준 구현으로 남아 있다. 시간 루프 안에서는 충돌하는 위치에 반복적으로 누적하는 scatter보다, 순서가 고정된 여섯 번의 gather가 성능과 재현성 면에서 다루기 좋았다. 최적화된 연산은 스칼라와 여러 차원의 배열 모두에서 이 기준 구현과 같은 결과를 내는지 테스트한다.

## 시간 루프 밖으로 밀어낼 것들

FDTD는 같은 모양의 계산을 수만 번 되풀이한다. 그러니 스텝 사이에 변하지 않는 값은 시작할 때 한 번만 만들어 둔다.

- 각 반경에서의 주·쌍대 모서리 길이와 셀 면적
- 방사 셀 너비와 중점 사이 거리
- 위치별 전도도와 유전율
- 손실을 반영하는 $C_a$, $C_b$ 계수
- 파원이 들어갈 꼭짓점, 방사층과 가중치

시간 루프에는 구면 거리나 물질 계수 계산도, 이웃을 찾는 코드도 없다. 남는 것은 2편의 네 갱신식뿐이다. 이는 속도뿐 아니라 코드를 검토할 때도 중요하다. 루프 안의 각 줄을 맥스웰 방정식의 어느 항과 대응하는지 확인하기 쉬워진다.

## NumPy로 한 스텝 굴리기

한 스텝에서는 먼저 현재 전기장으로 $H_t$와 $H_r$를 갱신하고 새 자기장으로 $E_r$와 $E_t$를 갱신한다.

![표면과 방사 방향의 전기장 차이로 접선 자기장 Ht를 갱신하는 흐름](/assets/img/ionosphere-fdtd/numpy-step-ht.svg)

![삼각형 순환으로 방사 자기장 Hr을 갱신하는 흐름](/assets/img/ionosphere-fdtd/numpy-step-hr.svg)

![쌍대 셀 순환과 파원 전류로 방사 전기장 Er을 갱신하는 흐름](/assets/img/ionosphere-fdtd/numpy-step-er.svg)

![자기장의 표면·방사 방향 차이로 접선 전기장 Et를 갱신하는 흐름](/assets/img/ionosphere-fdtd/numpy-step-et.svg)

$H_t$ 갱신은 앞서 본 $E_r$의 표면 방향 차이와 $E_t$의 방사 방향 차이를 합친다. $H_r$는 `Et * primal_lengths`를 삼각형 둘레로 더한 뒤 면적으로 나눈다. $E_r$는 `Ht * dual_lengths`를 오각형·육각형 둘레로 더하고 손실 계수와 파원 전류를 반영한다. 마지막 $E_t$는 모서리 좌우의 $H_r$ 차이와 위아래 $H_t$ 차이로 전진한다.

지속해서 보관할 네 장은 제자리 연산으로 갱신하고 차이와 순환을 담는 배열은 임시로 사용한다. 구현을 직접 확인하려면 [NumPy 백엔드](https://github.com/ruddyscent/ionosphere-fdtd/blob/main/src/ionosphere_fdtd/backends/numpy_backend.py)와 [풀이기의 시간 갱신](https://github.com/ruddyscent/ionosphere-fdtd/blob/main/src/ionosphere_fdtd/solver.py)을 함께 보면 된다.

## 기준 구현으로 실행하기

NumPy는 기본 백엔드이며 자동 자료형은 `float64`다.

```python
from ionosphere_fdtd import GeodesicFDTD, GaussianCurrent, SimulationConfig

simulation = GeodesicFDTD(
    SimulationConfig(subdivision=2, radial_cells=24),
    source=GaussianCurrent(carrier_frequency_hz=20.0),
    backend="numpy",
    dtype="float64",
)
simulation.step(1_000)
print(simulation.diagnostics())
```

명령행에서는 백엔드 옵션을 생략해도 된다.

```bash
uv run ionosphere --subdivision 2 --radial-cells 24 --steps 1000
```

공개된 장은 평범한 NumPy 배열이라 후처리도 바로 할 수 있다.

```python
surface_er = simulation.er[:, 12]
peak = np.max(np.abs(surface_er))
```

## 메모리는 얼마나 빨리 늘어날까

네 장만 세어도 저장할 실수의 수는 이렇다.

$$
N_v(N_r+1)+N_e(N_r+1)+N_eN_r+N_fN_r
$$

방사 셀 수에는 선형으로 비례하지만 구면 세분화 레벨 $L$에는 대략 $4^L$로 증가한다. 방사 셀 24개를 쓸 때 네 장의 `float64` 저장 공간은 세분화 4에서 약 4.30 MiB, 세분화 8에서 약 1,100 MiB가 된다. `float32`는 정확히 절반이지만 여기에 기하·물질·접속 배열과 임시 배열이 더 필요하다.

![세분화 레벨에 따라 증가하는 float32와 float64 장 배열의 저장 공간](/assets/img/ionosphere-fdtd/numpy-field-memory-scaling.svg)

그래서 NumPy 구현의 역할은 분명하다. 격자를 개발하고 식을 검토하며 작은·중간 크기 실험과 후처리를 하기 좋다. 무엇보다 배정밀도 기준 답을 제공한다. 정지한 장은 그대로 정지하는지, 손실장에서 $C_a$만큼 감쇠하는지, 파원의 위치와 총전류가 보존되는지, 최적화한 순환이 단순한 scatter 정의와 일치하는지 모두 이 경로에서 확인한다.

하지만 수십만 개의 표면 셀과 수만 스텝으로 올라가면 CPU 기준 구현만으로는 부족하다. 다음 4편에서는 알고리즘을 새로 쓰지 않고 이 배열 연산을 PyTorch 텐서로 옮겨 CPU·MPS·CUDA에서 실행한다. NumPy가 **계산을 묶어서 표현하는 단계**였다면, 그다음은 **묶인 계산과 데이터를 어디에 두고 어떻게 실행할지 정하는 단계**다.
