박사학위 주제는 FDTD와 관련이 깊습니다. 몇몇 기능은 학위 기간에 다루지 못했는데, 요즘 바이브 코딩 기술 덕분에 큰 시간을 들이지 않고도 관심 있던 구현을 시도할 수 있게 되었습니다.

PHANTOM (Pythonic High-performance Adaptive Nonorthogonal Time-domain Object-oriented Maxwell solver) is a Python-first electromagnetic simulation framework for FDTD and related time-domain methods, with PyTorch-based acceleration.

GMES -> Fork -> 오류 점검 -> 파이썬 3.14 적용 -> C++ 표준 적용 -> 테스트 프레임워크 적용 -> PyTorch 적용 -> INUG 

그중 가장 해보고 싶은 일은 두 가지입니다.
하나는 GPU 가속입니다. GMES에는 MPI를 이용해 리눅스 클러스터에서 병렬 시뮬레이션을 하는 기능이 있습니다. 요즘 인공지능 분야에서는 PyTorch로 GPU 가속 기능을 구현할 수 있으므로, GMES의 FDTD 시뮬레이터에도 GPU 가속을 적용해보고 싶습니다.
- https://github.com/flaport/fdtd

다른 하나는 irregular nonorthogonal structured grid 지원입니다. GMES는 Yee 격자에서 자유로운 물질 배치에 상당한 강점을 갖지만, 직교 격자만 지원하기 때문에 사용하는 메모리 용량에 비해 해상도가 낮습니다. 비정형 격자를 지원하면 메모리 효율이 크게 개선될 것입니다.
- A. Taflove 11.5장
