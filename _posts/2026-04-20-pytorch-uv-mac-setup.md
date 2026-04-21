---
layout: post
title: Apple Silicon 맥에서 PyTorch를 uv로 설치하고 사용하는 방법
subtitle: brew로 uv를 설치하고 Apple Silicon에서 PyTorch와 MPS까지 바로 확인하기
tags: [python, pytorch, uv, mac, apple-silicon, machine-learning]
cover-img: /assets/img/develop.jpeg
thumbnail-img: /assets/img/python.png
share-img: /assets/img/develop.jpeg
author: 전경원
---

요즘 `pip`의 대안으로 떠오르는 `uv`를 쓰면 가상환경 생성, Python 버전 관리, 패키지 설치까지 한 흐름으로 정리할 수 있다. 이 글에서는 uv`를 설치한 뒤, Apple Silicon 맥에서 PyTorch를 설치하고, 마지막으로 **MPS(Metal Performance Shaders)** 를 통해 GPU 사용 가능 여부까지 확인하는 과정을 순서대로 정리해 본다.

## 1. uv 설치

macOS에서는 `Homebrew`로 설치하는 방식이 가장 간단하다.

```bash
brew install uv
```

설치가 끝났으면 버전을 확인한다.

```bash
uv --version
```

정상적으로 버전이 출력되면 다음 단계로 진행하면 된다.

## 2. 프로젝트 생성

먼저 작업할 디렉터리를 만들고 `uv` 프로젝트를 초기화한다.

```bash
mkdir pytorch-m1-uv
cd pytorch-m1-uv
uv init
```

이렇게 하면 `pyproject.toml`을 포함한 기본 Python 프로젝트 구조가 생성된다.

## 3. Python 준비

PyTorch를 설치하기 전에 사용할 Python 버전을 준비한다. 여기서는 `3.12`를 기준으로 진행한다.

```bash
uv python install 3.12
```

필요하다면 이후 `uv run`이 해당 버전을 사용하도록 프로젝트에 맞춰 관리할 수 있다. 새 프로젝트를 시작할 때 이 단계를 먼저 해 두는 편이 깔끔하다.

## 4. PyTorch 설치

이제 PyTorch 관련 패키지를 설치한다.

```bash
uv add torch torchvision torchaudio
```

설치가 끝나면 `pyproject.toml`과 `uv.lock`에 의존성이 기록된다. 이 점이 `pip install`만 쓰는 방식보다 재현성이 좋은 이유 중 하나다.

## 5. 설치 확인

PyTorch가 정상적으로 설치됐는지 먼저 버전부터 확인해 보자.

```bash
uv run python -c "import torch; print(torch.__version__)"
```

버전 문자열이 출력되면 기본 설치는 끝난 것이다.

## 6. MPS 사용 가능 여부 확인

Apple Silicon 맥에서는 CUDA 대신 `MPS`를 통해 GPU 가속을 사용할 수 있다. 아래처럼 간단한 코드를 만들어 확인할 수 있다.

```python
import torch

print("MPS built:", torch.backends.mps.is_built())
print("MPS available:", torch.backends.mps.is_available())
```

예를 들어 `main.py`에 저장한 뒤 실행하면 된다.

```bash
uv run python main.py
```

두 값이 모두 `True`라면 현재 환경에서 MPS를 사용할 수 있다는 의미다.

## 7. MPS로 텐서 실행

이제 실제로 텐서를 MPS 디바이스에 올려 보자.

```python
import torch

device = torch.device("mps")

x = torch.ones(5, device=device)
y = x * 2

print(x)
print(y)
print(x.device)
```

출력된 디바이스가 `mps:0`처럼 보인다면 GPU 경로로 실행되고 있는 것이다.

## 8. 간단한 모델 실행

이번에는 아주 작은 신경망 레이어를 MPS에서 실행해 보자.

```python
import torch
import torch.nn as nn

device = torch.device("mps" if torch.backends.mps.is_available() else "cpu")

model = nn.Linear(4, 2).to(device)
x = torch.randn(3, 4).to(device)
y = model(x)

print(y)
print(y.device)
```

이 정도만 확인해도 PyTorch 설치 자체뿐 아니라, 실제 모델 연산까지 Apple GPU 경로에서 문제없이 돌아가는지 빠르게 점검할 수 있다.

## 9. 최종 디렉터리 구조

여기까지 진행하면 대략 아래와 비슷한 구조가 된다.

```text
pytorch-m1-uv/
├─ pyproject.toml
├─ uv.lock
└─ main.py
```

실제 프로젝트를 시작하면 여기에 학습 코드, 데이터 처리 코드, 노트북 파일 등을 추가하면 된다.

## 10. 정리

Apple Silicon 맥에서 PyTorch를 쓰기 위한 과정은 예전보다 훨씬 단순해졌다.

- `uv`는 빠른 패키지 설치와 프로젝트 환경 관리를 함께 처리해 준다.
- Apple Silicon 맥에서는 `MPS`를 통해 GPU 가속을 사용할 수 있다.
- `brew install uv`부터 시작하면 macOS에서 가장 간단하게 환경을 잡을 수 있다.

새 Python 프로젝트를 만들 때 `pip`와 `venv`를 따로 조합하기보다 `uv`로 시작하는 편이 훨씬 편하다. PyTorch처럼 의존성이 많은 라이브러리도 같은 흐름 안에서 깔끔하게 관리할 수 있기 때문이다.