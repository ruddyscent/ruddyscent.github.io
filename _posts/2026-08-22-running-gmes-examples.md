---
layout: post
title: GMES 예제 9개를 Python 3.14에서 다시 돌려봤다
subtitle: 공기 중 원통파부터 금 박막의 Fresnel 반사까지, 오래된 예제가 다시 움직였다
tags: [gmes, fdtd, simulation, electromagnetics, python, photonics]
cover-img: /assets/img/develop.jpeg
thumbnail-img: /assets/img/gmes-examples/fdtd-sinc-field.webp
share-img: /assets/img/develop.jpeg
author: 전경원
description: Python 3.14로 현대화한 GMES의 2D·3D FDTD 예제 9개를 실행하고 공기 중 파동 전파부터 도파관, TFSF 산란, 분산 물질과 플라즈모닉 구조까지 코드와 영상으로 살펴본다.
---

[지난 글]({% post_url 2026-08-21-reviving-gmes %})에서 10년 넘게 멈춰 있던 GMES를 다시 꺼낸 이유와 현대화 계획을 정리했습니다. 그때 가장 먼저 하겠다고 적은 일은 저장소의 예제를 하나씩 실행해 지금도 동작하는 기능과 이미 깨진 기능을 구분하는 것이었습니다.

이번에는 실제로 그 일을 했습니다. Python 2 시절에 작성한 예제 9개를 Python 3.14 환경에서 다시 실행하고 계산 영역과 재료 분포, 마지막 시점의 전자기장을 한 장의 그림과 짧은 영상으로 남겼습니다. 단순한 공기 중 파동부터 유전체 도파관, TFSF 산란, 광결정, 금속의 물질 분산과 3차원 구조까지 GMES의 주요 기능을 한 바퀴 도는 셈입니다.

![Python 3.14에서 다시 실행한 GMES 예제 9개의 재료 분포와 전자기장 결과.](/assets/img/gmes-examples/gallery.png)

아래 재생목록에는 이 글에서 다루는 9개 결과가 모두 들어 있습니다. 각 영상은 재료 분포에서 시작해 시뮬레이션이 진행된 뒤의 장 분포로 넘어갑니다.

<iframe
  src="https://www.youtube-nocookie.com/embed/videoseries?list=PLUyMddqyWSe0"
  title="GMES FDTD Simulation Examples"
  width="960"
  height="540"
  style="width: 100%; aspect-ratio: 16 / 9; border: 0;"
  loading="lazy"
  allow="accelerometer; autoplay; clipboard-write; encrypted-media; gyroscope; picture-in-picture; web-share"
  referrerpolicy="strict-origin-when-cross-origin"
  allowfullscreen>
</iframe>

[YouTube에서 GMES 예제 재생목록 열기](https://www.youtube.com/playlist?list=PLUyMddqyWSe0)

## 예제는 모두 같은 순서로 움직인다

GMES 예제는 다루는 물리가 달라도 코드의 뼈대는 거의 같습니다.

1. `Cartesian`으로 계산 영역과 해상도를 정합니다.
2. `DefaultMedium`, `Block`, `Cylinder`, `Sphere` 같은 객체로 재료와 형상을 배치합니다.
3. `PointSource`나 `TotalFieldScatteredField`로 파동을 넣습니다.
4. 문제에 맞는 FDTD 클래스를 만들고 `init()`으로 격자와 갱신 계수를 준비합니다.
5. 관측할 장이나 probe를 등록하고 `step_until_t()`로 시간을 전진시킵니다.

가장 작은 [`air2d.py`](https://github.com/ruddyscent/gmes/blob/blog/examples/air2d.py)가 이 순서를 그대로 보여 줍니다.

```python
from gmes import *

space = Cartesian(size=(10, 10, 0), resolution=20)
geometry = [
    DefaultMedium(material=Dielectric()),
    Shell(material=Cpml()),
]
sources = [
    PointSource(
        src_time=Continuous(freq=0.8),
        center=(0, 0, 0),
        component=Ez,
    )
]

simulation = TMzFDTD(space, geometry, sources)
simulation.init()
simulation.show_field(Ez, Z, 0)
simulation.step_until_t(10)
```

`size=(10, 10, 0)`은 z 방향 두께가 없는 2차원 영역입니다. 가운데 놓인 `Ez` 점 소스가 주파수 0.8로 진동하고 `TMzFDTD`는 `Ez`, `Hx`, `Hy`를 번갈아 갱신합니다. 바깥의 CPML(Convolutional Perfectly Matched Layer)은 계산 영역 끝에서 파동이 되돌아오는 것을 줄입니다.

![공기 중 점 소스에서 퍼져 나가는 2차원 TMz 원통파.](/assets/img/gmes-examples/air2d.png)

아무 구조도 없는 이 예제는 화려하지 않지만 가장 쓸모 있는 기준 문제입니다. 패키지와 네이티브 확장을 제대로 불러오는지, 시간 갱신 뒤에 `NaN`이나 무한대가 생기지 않는지, CPML 부근에서 큰 반사가 보이지 않는지를 몇 초 안에 확인할 수 있습니다. 이번 전체 실행은 Apple silicon MacBook에서 약 3.2초 걸렸습니다.

## 재료 하나가 파동의 길을 만든다

다음 단계는 공기 중에 유전체를 놓는 것입니다. [`slab_waveguide.py`](https://github.com/ruddyscent/gmes/blob/blog/examples/slab_waveguide.py)는 상대 유전율 12인 폭 1의 슬래브를 계산 영역 가운데 배치합니다.

```python
space = Cartesian(size=(16, 8, 0), resolution=10)
geometry = [
    DefaultMedium(material=Dielectric()),
    Block(material=Dielectric(12), size=(inf, 1, inf)),
    Shell(material=Cpml()),
]
sources = [
    PointSource(
        src_time=Continuous(freq=0.15),
        component=Ez,
        center=(-7, 0, 0),
    )
]
```

왼쪽의 점 소스에서 나온 파동은 굴절률이 높은 슬래브 부근에 모여 오른쪽으로 진행합니다. 공기 중에서 사방으로 퍼지던 원통파가 재료 하나를 추가하자 도파 모드로 바뀌는 모습이 분명합니다.

![상대 유전율 12인 슬래브를 따라 진행하는 Ez 도파 모드.](/assets/img/gmes-examples/slab-waveguide.png)

다만 새 모델에서는 예제처럼 `inf`를 형상 크기로 쓰기보다 계산 영역보다 충분히 큰 유한한 값을 쓰는 편이 안전합니다. 현재 GMES는 `numpy.inf`를 모든 형상 경계에서 일관되게 무한대로 처리하지 못합니다.

[`phc_waveguide.py`](https://github.com/ruddyscent/gmes/blob/blog/examples/phc_waveguide.py)는 같은 생각을 주기 구조로 확장합니다. 상대 유전율 8.9인 원기둥을 격자로 늘어놓되 가운데 한 줄만 비웁니다.

```python
geometry.extend(
    [
        Cylinder(
            material=Dielectric(8.9),
            radius=0.38,
            center=(x, y, 0),
        )
        for x in range(-8, 9)
        for y in range(-4, 5)
        if y != 0
    ]
)
```

핵심은 `if y != 0` 한 줄입니다. 규칙적인 광결정에서 가운데 행을 제거해 선결함을 만들고 빛이 그 통로를 따라 진행하게 합니다. 복잡한 형상도 Python의 리스트 컴프리헨션으로 읽기 좋게 표현할 수 있다는 점은 예전 GMES에서 지금까지 남아 있는 장점입니다.

![원기둥 배열의 가운데 행을 비워 만든 2차원 광결정 선결함 도파관.](/assets/img/gmes-examples/phc-waveguide.png)

이런 반복 구조에서는 장을 보기 전에 `show_permittivity()`로 재료 분포부터 확인해야 합니다. 원기둥의 위치나 결함 행을 잘못 지정해도 코드는 정상적으로 실행될 수 있기 때문입니다.

## 같은 평면파, 산란체 하나의 차이

점 소스는 파동을 사방으로 내보냅니다. 특정 방향으로 진행하는 평면파를 넣고 입사장과 산란장을 구분하려면 TFSF(total-field/scattered-field)를 사용합니다. [`tfsf.py`](https://github.com/ruddyscent/gmes/blob/blog/examples/tfsf.py)의 소스는 다음과 같습니다.

```python
TotalFieldScatteredField(
    src_time=Continuous(freq=0.8),
    center=(0, 0, 0),
    size=(3, 3, 1),
    direction=(1, -1, 0),
    polarization=(0, 0, 1),
)
```

`direction`은 입사 방향, `polarization`은 편광 방향입니다. 점선으로 표시된 TFSF 경계 안에는 입사장과 산란장을 합한 전체장이 있고 바깥에는 산란장만 남습니다. 진공에는 산란체가 없으므로 바깥쪽 장은 거의 사라져야 합니다.

![진공의 TFSF 영역 안에서 대각선으로 진행하는 z 편광 평면파.](/assets/img/gmes-examples/tfsf.png)

[`tfsf_with_scatterer.py`](https://github.com/ruddyscent/gmes/blob/blog/examples/tfsf_with_scatterer.py)는 같은 설정의 중앙에 상대 유전율 3, 반지름 1인 원기둥 하나를 추가합니다.

```python
Cylinder(
    material=Dielectric(3),
    center=(0, 0, 0),
    radius=1,
    axis=(0, 0, 1),
)
```

![유전체 원기둥에서 산란된 평면파가 TFSF 경계 바깥으로 퍼져 나가는 모습.](/assets/img/gmes-examples/tfsf-with-scatterer.png)

두 결과를 나란히 보면 TFSF의 역할이 선명해집니다. 첫 그림에서는 경계 안의 평면파가 핵심이고 두 번째 그림에서는 원기둥이 만든 산란파가 경계 밖까지 이어집니다. 반지름, 유전율, 입사각과 주파수를 하나씩 바꾸면 산란 문제의 작은 실험 세트가 됩니다.

## 금속은 유전율 하나로 끝나지 않는다

일반 유전체는 일정한 상대 유전율로 시작할 수 있지만 금속은 파장에 따라 응답이 크게 달라집니다. [`fresnel_reflection.py`](https://github.com/ruddyscent/gmes/blob/blog/examples/fresnel_reflection.py)는 Drude pole 하나와 critical point 두 개를 조합한 `DcpPlrc` 모델로 얇은 금 층을 표현합니다.

```python
dp = DrudePole(omega=..., gamma=...)
cp1 = CriticalPoint(amp=..., phi=..., omega=..., gamma=...)
cp2 = CriticalPoint(amp=..., phi=..., omega=..., gamma=...)
gold = DcpPlrc(
    eps_inf=1.11683,
    mu_inf=1,
    dps=(dp,),
    cps=(cp1, cp2),
)
```

연속 Gaussian beam이 금 층을 향해 진행하고 층의 앞뒤에 둔 probe가 반사와 투과의 시간 신호를 기록합니다. 그림만 보면 왼쪽과 오른쪽의 장이 달라졌다는 정도를 알 수 있습니다. 정량적인 Fresnel 반사율을 얻으려면 금 층이 없는 기준 실행과 푸리에 변환, probe 위치에 따른 위상 보정이 더 필요합니다.

![Drude-critical-point 분산 모델로 표현한 얇은 금 층과 그 양쪽의 Ez 장.](/assets/img/gmes-examples/fresnel-reflection.png)

[`metal_array.py`](https://github.com/ruddyscent/gmes/blob/blog/examples/metal_array.py)는 분산 물질을 3차원으로 가져갑니다. 여섯 개의 은 나노구를 한 줄로 놓고 배열 방향의 `Jy` 점 소스로 종방향 플라즈몬 모드를 여기합니다.

```python
for y in range(-2, 4):
    geometry.append(
        Sphere(
            material=Silver(75 * NANO),
            radius=1.0 / 3,
            center=(0, y, 0),
        )
    )
```

![여섯 개 은 나노구로 만든 플라즈몬 도파관의 재료 분포와 Ey 장.](/assets/img/gmes-examples/metal-array.png)

전체 설정은 약 1.1GB의 메모리를 사용할 수 있습니다. 설치와 코드 경로만 확인할 때는 공간 해상도와 종료 시간을 줄이는 `--quick` 옵션이 낫습니다.

```sh
python examples/metal_array.py --quick
```

## 3차원 예제는 먼저 작게 확인한다

3차원 격자는 해상도를 조금만 높여도 셀 수와 메모리 사용량이 빠르게 늘어납니다. 그래서 이번 검증에서는 큰 예제 두 개를 원래 크기로 끝까지 실행했다고 포장하지 않았습니다. 축소 조건으로 형상 생성, 물질 매핑, 시간 갱신과 시각화 경로를 확인했습니다.

[`man.py`](https://github.com/ruddyscent/gmes/blob/blog/examples/man.py)는 `Block`, `Sphere`, `Cone`, `Cylinder`, `Ellipsoid`를 조합해 사람 모양을 만듭니다. 세 축의 유전율 단면을 함께 보면 3차원 객체의 중심, 크기와 방향 벡터가 어떻게 적용되는지 확인하기 좋습니다.

![여러 기하 객체를 조합한 사람 모양 유전체의 x, y, z 단면.](/assets/img/gmes-examples/man.png)

[`phc_slab.py`](https://github.com/ruddyscent/gmes/blob/blog/examples/phc_slab.py)는 실리콘-온-인슐레이터 슬래브에 삼각 격자 공기 구멍과 선결함을 만듭니다. 전체 모델은 약 1.3GB의 메모리를 요구하므로 이 예제도 먼저 축소 실행합니다.

```sh
python examples/man.py --quick
python examples/phc_slab.py --quick
```

![축소 해상도로 실행한 3차원 광결정 슬래브 도파관.](/assets/img/gmes-examples/phc-slab.png)

축소 실행의 통과는 원래 크기의 결과가 정확하다는 뜻이 아닙니다. 큰 계산을 시작하기 전에 설치, 형상 구성과 기본 시간 갱신이 무너지지 않았음을 확인하는 smoke test에 가깝습니다.

## 9개 예제의 실행 결과

검증은 Apple silicon MacBook, Python 3.14.6에서 진행했습니다. Matplotlib은 화면을 열지 않는 `Agg` 백엔드를 사용했고 출력은 저장소 밖의 임시 디렉터리에 기록했습니다.

| 예제 | 확인한 범위 | 결과 | 실행 시간 / 최대 메모리 | 영상 |
| --- | --- | --- | --- | --- |
| `air2d.py` | `t=10` 전체 실행 | 통과 | 3.20초 / 143MB | [원통파](https://www.youtube.com/watch?v=EuQ4U9MphuY) |
| `slab_waveguide.py` | `t=200` 전체 실행 | 통과 | 2.23초 / 130MB | [슬래브 도파관](https://www.youtube.com/watch?v=yLs-zd6pk78) |
| `phc_waveguide.py` | `t=200` 전체 실행 | 통과 | 9.93초 / 139MB | [2D 광결정](https://www.youtube.com/watch?v=msWHde4rzlQ) |
| `tfsf.py` | `t=200` 전체 실행 | 통과 | 4.50초 / 125MB | [TFSF 평면파](https://www.youtube.com/watch?v=dnFyxgWO6_M) |
| `tfsf_with_scatterer.py` | `t=200` 전체 실행 | 통과 | 4.96초 / 122MB | [원기둥 산란](https://www.youtube.com/watch?v=STJ3K1ijt44) |
| `fresnel_reflection.py` | `t=200` 전체 실행, probe 파일 6개 | 통과 | 21.67초 / 88MB | [금 박막 반사](https://www.youtube.com/watch?v=mDFX909jFGg) |
| `man.py` | `--quick` 형상 생성 | 축소 통과 | 2.17초 / 130MB | [3D 형상](https://www.youtube.com/watch?v=uAQ3rWTnloo) |
| `metal_array.py` | `--quick`, `t=1` | 축소 통과 | 3.52초 / 127MB | [은 나노구 배열](https://www.youtube.com/watch?v=3FFPlhN7q-U) |
| `phc_slab.py` | `--quick`, `t=1` | 축소 통과 | 2.34초 / 129MB | [3D 광결정 슬래브](https://www.youtube.com/watch?v=GB8235-JhbQ) |

재현하려면 GMES 저장소에서 Python 3.14 가상환경을 만들고 시각화 의존성을 함께 설치합니다. C++17 컴파일러와 SWIG 4도 필요합니다.

```sh
python3.14 -m venv .venv
source .venv/bin/activate
python -m pip install --upgrade pip
python -m pip install -e ".[plot]"

MPLBACKEND=Agg python examples/air2d.py
```

각 예제에서 실제로 확인한 범위와 필드 크기, peak 값, probe 샘플 수는 저장소의 [`examples/VERIFICATION.md`](https://github.com/ruddyscent/gmes/blob/blog/examples/VERIFICATION.md)에 남겼습니다.

## 예제가 움직인다는 것과 답이 맞는다는 것

이번 실행으로 확인한 것은 Python 3.14에서 예제 9개의 주요 코드 경로가 다시 이어졌다는 사실입니다. 계산 영역을 만들고 형상을 격자에 매핑하고 소스를 넣고 CPML과 분산 물질을 포함한 시간 갱신을 끝까지 수행할 수 있습니다. 생성된 필드는 모두 유한했고 각 문제에서 기대한 정성적인 형태도 나타났습니다.

하지만 그림이 자연스러워 보인다는 이유만으로 계산이 정확하다고 결론 내릴 수는 없습니다. 해상도를 바꾸었을 때 결과가 수렴하는지, CPML의 잔류 반사가 얼마나 작은지, Fresnel 반사율과 투과율이 이론값에 어느 오차까지 맞는지, 단일 프로세스와 MPI 실행이 같은 답을 내는지는 별도의 수치 검증이 필요합니다.

그래도 출발점은 확보했습니다. 예제 9개는 앞으로 변경이 기존 동작을 깨뜨렸는지 확인하는 회귀 기준이 되고 그중 `fresnel_reflection.py`는 그림에서 수치로 넘어가기 좋은 다음 문제입니다. 다음에는 probe 신호를 반사율과 투과율로 바꾸고 금 박막이 없는 기준 실행과 Fresnel 식을 이용해 GMES의 답을 정량적으로 비교해보려 합니다.
