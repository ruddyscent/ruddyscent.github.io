---
layout: post
title: IonosphereFDTD - PyTorch로 FDTD 계산을 가속하기
subtitle: 전자기장 값과 계산 루틴을 같은 장치에 배치해야 GPU 활용도가 높아진다
tags: [fdtd, simulation, ionosphere, pytorch, gpu, cuda, numerical-methods]
cover-img: /assets/img/develop.jpeg
thumbnail-img: /assets/img/ionosphere-fdtd-globe-simple.webp
share-img: /assets/img/develop.jpeg
author: 전경원
mathjax: true
---

4편의 NumPy 구현은 꼭짓점과 모서리를 하나씩 도는 반복문을 배열 연산으로 바꿨다. PyTorch 버전은 여기서 FDTD 알고리즘을 다시 만들지 않는다. 격자, 장의 배치, 물질 계수, 파원, 쿠랑 조건과 네 갱신식은 그대로 두고 작은 배열 연산 집합만 텐서 연산으로 교체한다.

이 경계를 미리 나눠 둔 덕분에 NumPy는 읽기 쉬운 배정밀도 기준 구현으로 남고 PyTorch는 같은 시간 적분기를 멀티코어 CPU와 Apple MPS, NVIDIA CUDA에서 실행할 수 있다. 고정된 모양의 한 스텝 전체는 `torch.compile`로 묶을 수도 있다.

## 벡터화와 GPU 가속은 다른 층의 일이다

NumPy 벡터화는 원소별 파이썬 반복을 큰 배열 연산으로 바꾼다. PyTorch 최적화는 이미 묶인 연산을 어느 장치에서 실행하고 장치 사이 이동과 커널 호출을 어떻게 줄일지를 다룬다.

| 구분 | NumPy 벡터화 | PyTorch 실행 최적화 |
|---|---|---|
| 줄이려는 비용 | 원소마다 반복되는 파이썬 실행 | 장치 전송, 커널 호출, 동기화, 반복 그래프 실행 |
| 상태가 있는 곳 | CPU 메모리 | CPU, MPS 또는 CUDA 장치 |
| 실행 단위 | 개별 배열 연산 | 개별 텐서 연산 또는 컴파일된 한 스텝 |
| 남는 비용 | 임시 배열과 CPU 메모리 대역폭 | 컴파일 준비, 장치 메모리, 전송과 동기화 |
| 이 프로젝트의 역할 | 검토 가능한 `float64` 기준 | 큰 계산을 위한 실행 경로 |

벡터화됐다고 저절로 GPU 코드가 되지는 않는다. 반대로 GPU를 골랐다고 잦은 데이터 이동이 사라지는 것도 아니다. IonosphereFDTD에서는 먼저 계산을 텐서 단위로 만들고 그 텐서와 반복 갱신을 선택한 장치에 계속 머물게 한다.

## 사용할 장치는 시작할 때 정한다

PyTorch 의존성은 선택 사항이다.

```bash
uv sync --extra pytorch
```

`cpu`, `mps`, `cuda`, `cuda:N`을 직접 지정할 수 있고, `gpu`는 CUDA의 별칭이다. `auto`는 CUDA, MPS, CPU 순서로 사용 가능한 장치를 고른다.

```python
simulation = GeodesicFDTD(
    config,
    source=source,
    backend="torch",
    device="auto",
)
```

명시한 MPS나 CUDA가 없으면 CPU로 조용히 물러서지 않고 오류를 낸다. 존재하지 않는 CUDA 번호도 거부한다. 큰 계산을 걸어 두고 나중에야 CPU에서 돌았다는 사실을 발견하는 일을 막기 위해서다.

초기화할 때 장과 기하, 물질 계수는 요청한 자료형과 장치의 텐서가 된다. 모서리·면 번호는 같은 장치의 `torch.long` 텐서로 옮긴다. 스텝을 밟는 동안 격자 정보가 CPU와 GPU 사이를 오갈 필요는 없다.

![CPU 실행에서 장과 접속 표가 호스트 메모리에 머무는 구조](/assets/img/ionosphere-fdtd/pytorch-memory-cpu.svg)

![GPU 실행에서 초기화와 관측만 장치 경계를 건너는 구조](/assets/img/ionosphere-fdtd/pytorch-memory-gpu.svg)

Apple Silicon은 통합 메모리를 쓰지만 PyTorch에는 여전히 논리적인 MPS 장치 경계가 있다. 핵심은 "GPU 메모리가 무조건 빠르다"가 아니라, 정적인 접속 표와 계속 바뀌는 장을 계산 커널 곁에 두고 불필요한 동기화를 피하는 데 있다.

## PyTorch에서도 접속 표는 그대로다

모서리 양 끝과 좌우 면의 차이는 NumPy와 거의 같은 코드로 계산한다.

```python
vertex_values[edges[:, 1]] - vertex_values[edges[:, 0]]
face_values[left_faces] - face_values[right_faces]
```

삼각형 순환은 세 모서리를 차례로 모아 부호를 곱하고 `add_`로 누적한다. 오각형·육각형 순환도 4편의 `(Nv, 6)` 패딩 접속 표를 그대로 쓴다.

```python
sign_shape = (n_vertices,) + (1,) * (edge_values.ndim - 1)
result = edge_values[vertex_edges[:, 0]]
result.mul_(vertex_edge_signs[:, 0].reshape(sign_shape))

for slot in range(1, vertex_edges.shape[1]):
    term = edge_values[vertex_edges[:, slot]]
    term.mul_(vertex_edge_signs[:, slot].reshape(sign_shape))
    result.add_(term)
```

반복 횟수는 격자 크기가 아니라 위상 구조가 정한다. 삼각형은 세 번, 쌍대 셀은 여섯 번이며 각 반복은 모든 셀과 방사층을 함께 처리한다. 원자적 scatter로 같은 위치에 경쟁적으로 더하지 않고 순서가 고정된 gather를 쓰므로 CUDA에서도 결과 순서를 재현하기 쉽고 `torch.compile`이 정적인 텐서 그래프로 잡기에도 알맞다.

![장치에 올라간 접속 표로 삼각형 순환을 계산하는 PyTorch 커널](/assets/img/ionosphere-fdtd/pytorch-topology-face.svg)

![여섯 슬롯으로 오각형·육각형 순환을 계산하는 PyTorch 커널](/assets/img/ionosphere-fdtd/pytorch-topology-dual.svg)

## Eager로 시작하고 긴 계산만 컴파일한다

작은 실험은 eager 실행으로 충분하다.

```bash
uv run --extra pytorch ionosphere \
  --backend torch --device mps --steps 200
```

격자 모양이 고정되고 스텝 수가 많다면 한 번의 장 갱신 전체를 컴파일할 수 있다.

```bash
uv run --extra pytorch ionosphere \
  --backend torch --device cuda --torch-compile --steps 20000
```

내부에서는 다음 설정을 쓴다.

```python
torch.compile(step, fullgraph=True, dynamic=False)
```

`fullgraph=True`는 그래프 중단 없이 한 스텝 전체를 잡도록 요구하고 `dynamic=False`는 고정된 모양에 특화한다([PyTorch `torch.compile` 문서](https://docs.pytorch.org/docs/stable/generated/torch.compile.html)). 격자와 배열 모양이 한 번 정해진 뒤 같은 계산을 수천 번 반복하는 FDTD와 잘 맞는다.

![Eager 실행에서 연산마다 프레임워크와 장치 커널을 호출하는 전반부](/assets/img/ionosphere-fdtd/pytorch-eager-first.svg)

![Eager 실행에서 개별 커널 호출이 이어지는 후반부](/assets/img/ionosphere-fdtd/pytorch-eager-second.svg)

![Compiled 실행의 첫 호출에서 한 스텝을 수집하고 준비하는 과정](/assets/img/ionosphere-fdtd/pytorch-compiled-warmup.svg)

![준비를 마친 compiled 장 갱신을 반복 실행하는 과정](/assets/img/ionosphere-fdtd/pytorch-compiled-steady.svg)

컴파일은 공짜가 아니다. 첫 호출에 그래프 수집과 코드 생성 시간이 든다. 꼭짓점 42개나 162개짜리 개발 격자처럼 작은 문제에서는 이 준비 비용을 회수하지 못할 수 있다. 먼저 eager로 재고 충분히 긴 계산에서만 컴파일을 켜는 편이 낫다.

## 한 장비에서 측정해 본 처리량

같은 조건으로 실행한 로컬 벤치마크에서는 최적화 층이 하나씩 더해지는 모습이 드러난다. 세분화 4 격자(쌍대 셀 2,562개, 모서리 7,680개, 삼각형 5,120개), 방사 셀 24개와 `float64`를 사용했다. 20스텝을 미리 돌린 뒤 동기화된 200스텝 묶음을 다섯 번 측정한 중앙값이며 격자 생성과 텐서 초기화, 첫 컴파일 시간은 제외했다.

| 구현 | 실행 방식 | 중앙값(steps/s) | NumPy 대비 |
|---|---|---:|---:|
| NumPy | CPU, eager | 77.4 | 1.00× |
| PyTorch | CPU, eager, 스레드 4개 | 261.4 | 3.38× |
| PyTorch | RTX 3060 CUDA, eager | 934.6 | 12.07× |
| PyTorch | RTX 3060 CUDA, compiled | 4,059.6 | 52.43× |

![NumPy CPU와 PyTorch CPU·CUDA eager·CUDA compiled의 스텝 처리량](/assets/img/ionosphere-fdtd/numpy-pytorch-throughput.svg)

마지막 숫자를 "PyTorch는 언제나 52배 빠르다"로 읽으면 안 된다. NumPy에서 PyTorch CPU로 옮길 때 백엔드 커널과 CPU 스레드 수가 달라졌고, CUDA에서는 하드웨어와 장치 상주가 더해졌으며, 마지막으로 컴파일이 반복 그래프를 최적화했다. 짧은 계산에는 표에서 뺀 컴파일 준비 시간이 지배적일 수 있다. 실제 세분화 레벨, 방사층, 자료형, 스텝 수와 관측 주기로 직접 재야 한다.

## 자료형은 장치와 관측량을 함께 본다

PyTorch의 자동 자료형은 모든 장치에서 `float32`다. `float64`는 CPU와 CUDA에서 쓸 수 있지만 이 백엔드의 MPS 경로에서는 거부한다.

```bash
# Apple GPU에서 빠르게 탐색
uv run --extra pytorch ionosphere \
  --backend torch --device mps --dtype float32 --steps 20000

# CUDA에서 배정밀도 장기 계산
uv run --extra pytorch ionosphere \
  --backend torch --device cuda --dtype float64 \
  --torch-compile --steps 35000
```

![PyTorch CPU에서 지원하는 자료형과 알맞은 작업](/assets/img/ionosphere-fdtd/pytorch-device-cpu.svg)

![Apple MPS에서 지원하는 자료형과 알맞은 작업](/assets/img/ionosphere-fdtd/pytorch-device-mps.svg)

![NVIDIA CUDA에서 지원하는 자료형과 알맞은 작업](/assets/img/ionosphere-fdtd/pytorch-device-cuda.svg)

단정밀도는 장 저장 공간을 절반으로 줄이고 대개 가속기 처리량도 높인다. 그렇다고 언제나 충분한 것도 아니고 배정밀도가 물리 모델의 오차를 해결해 주는 것도 아니다. 전 지구 검증에서는 MPS `float32`를 CUDA `float64`로 바꿔도 초기 감쇠 오차가 거의 달라지지 않았고 전리층 프로파일과 스펙트럼 창을 바로잡은 효과가 훨씬 컸다. 최종 생산 검증은 남은 차이를 산술 정밀도 탓으로 돌릴 여지를 줄이기 위해 CUDA 배정밀도로 수행했다.

따라서 자료형은 장의 최대값만 보고 고를 일이 아니다. 도달 시각, 위상, 감쇠처럼 실제로 판단하려는 관측량을 NumPy `float64` 기준과 비교해야 한다. PyTorch도 같은 수학식이 실행 순서와 장치에 따라 비트 단위로 같지 않을 수 있음을 명시한다([수치 정확도 문서](https://docs.pytorch.org/docs/stable/notes/numerical_accuracy.html)).

## 관측도 계산의 일부다

CUDA와 MPS의 장을 매 스텝 NumPy로 복사하면 호스트가 결과를 기다리면서 동기화한다. 빠른 커널을 만들어 놓고 관측 코드로 매번 브레이크를 거는 셈이다. CUDA 연산은 기본적으로 비동기이므로 호스트에서 값을 읽는 시점이 곧 기다림의 경계가 된다([CUDA semantics](https://docs.pytorch.org/docs/stable/notes/cuda.html)).

```python
er_on_host = simulation.to_numpy(simulation.er)
value = simulation.field_value("er", vertex, layer)
```

IonosphereFDTD는 이 변환을 명시적으로 드러낸다. 그림은 프레임을 그릴 때만 CPU로 옮기고 수신기 파형은 장치 텐서에 모아 두었다가 주기적으로 가져올 수 있다. 진단값도 최댓값 같은 수치를 실제로 요청할 때만 동기화한다.

![매 스텝 호스트가 값을 읽어 GPU 실행을 기다리게 하는 흐름의 전반부](/assets/img/ionosphere-fdtd/pytorch-observe-bad-first.svg)

![매 스텝 동기화가 반복되는 비효율적인 관측 흐름의 후반부](/assets/img/ionosphere-fdtd/pytorch-observe-bad-second.svg)

![수신값을 장치에 계속 모으는 관측 흐름의 전반부](/assets/img/ionosphere-fdtd/pytorch-observe-good-first.svg)

![필요한 시점에만 결과를 가져오는 관측 흐름의 후반부](/assets/img/ionosphere-fdtd/pytorch-observe-good-second.svg)

GPU 최적화에서 관측 주기는 부차적인 설정이 아니다. 계산 주기와 관측 주기를 분리해야 장치가 기다리지 않고 계속 일할 수 있다.

## 빠른 답이 같은 답인지 확인하기

가속 경로는 NumPy 기준과 비교해야 의미가 있다. 테스트는 NumPy와 PyTorch CPU 시뮬레이션을 모두 `float64`로 만들고 같은 파원으로 40스텝 전진시킨 다음 네 장을 비교한다.

```python
for field in ("er", "et", "hr", "ht"):
    np.testing.assert_allclose(
        torch_simulation.to_numpy(getattr(torch_simulation, field)),
        getattr(numpy_simulation, field),
        rtol=1e-11,
        atol=1e-12,
    )
```

이 밖에도 eager와 compiled 결과, 자동 장치 선택, CUDA 별칭, MPS의 자료형 제한, 순환 연산, 시각화 변환과 CPU 스레드 수를 검사한다. `float32` 경로는 단순히 유한한 값이 나오는 데 그치지 않고 NumPy `float64`에 대한 상대 $L_2$ 장 오차를 잰다. 구현은 [PyTorch 백엔드](https://github.com/ruddyscent/ionosphere-fdtd/blob/main/src/ionosphere_fdtd/backends/torch_backend.py), 검증 항목은 [백엔드 테스트](https://github.com/ruddyscent/ionosphere-fdtd/blob/main/tests/test_backends.py)에 있다.

실제 생산 규모의 한 사례는 세분화 8, 표면 셀 655,362개, 방사 셀 40개, 35,000스텝이다. RTX 3060에서 컴파일된 PyTorch와 `float64`로 2,677.5초가 걸렸다. 특정 GPU와 PyTorch 버전에서 얻은 기록이지 보편적인 성능 약속은 아니다. 다만 작은 NumPy 테스트에서 검증한 똑같은 풀이기를 코드 분기 없이 생산 규모 CUDA 계산까지 가져갔다는 점은 남는다.

## 다섯 편을 지나며

GPU 가속을 위해 맥스웰 방정식을 두 벌로 유지할 필요는 없었다. 필요한 것은 격자의 위상을 몇 개의 배열 연산으로 좁히고, 장과 접속 표를 백엔드가 소유하게 하며, 장치 이동과 동기화 시점을 드러내는 일이었다.

이로써 첫 다섯 편의 흐름이 닫힌다. 1편은 지구–전리층 도파관이라는 문제를 정했고 2편은 구를 방향 있는 주·쌍대 격자와 적분 연산으로 바꿨다. 3편은 그 공간 연산자 위에 네 장을 배치하고 시간 갱신을 완성했다. 4편은 갱신식을 NumPy의 읽을 수 있는 기준 구현으로 만들었고 이번 편은 같은 계산을 CPU·MPS·CUDA의 실행 경로로 옮겼다.

이제 더 중요한 일은 백엔드를 하나 더 붙이는 것이 아니다. 격자 수렴성을 더 촘촘히 확인하고, 전리층과 지각 모델을 개선하며, 수치 오차와 물리 입력의 불확실성을 계속 분리해 내는 일이다.
