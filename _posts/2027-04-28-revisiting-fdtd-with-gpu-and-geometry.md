박사학위 주제가 FDTD와 관련이 깊다. 몇몇 기능은 학위 기간에 다루지 못했는데, 요즘 바이브 코딩 기술 덕에 큰 시간 들이지 않고도 관심었던 구현을 시도해볼 수 있게 되었다. 

PHANTOM (Pythonic High-performance Adaptive Nonorthogonal Time-domain Object-oriented Maxwell solver) is a Python-first electromagnetic simulation framework for FDTD and related time-domain methods, with PyTorch-based acceleration.

GMES -> Fork -> 오류 점검 -> 파이썬 3.14 적용 -> C++ 표준 적용 -> 테스트 프레임워크 적용 -> PyTorch 적용 -> INUG 

그중 가장 해보고 싶은 건 두 가지이다. 
먼저, GPU 가속이다. GMES에는 MPI를 이용해서 리눅스 클러스터에서 병렬 시뮬레이션을 하는 기능이 있다. 요즘 인공지능 덕분에 GPU 가속 기능이 PyTorch로 구현되어 있어서, GMES의 FDTD 시뮬레이터에도 GPU 가속을 적용해보고 싶다. 
- https://github.com/flaport/fdtd

다음으로 irregular nonorthogonal structured grid 지원이다. GMES는 Yee 격자에 상에서 자유로운 물질 배치에 상당한 강점을 갖지만, 직교 격자만을 지원하기에 사용하는 메모리 용량 대비 해상도가 낮은 문제가 있다. 비정형 격자 지원이 가능해지면, 메모리 효율이 크게 개선될 것이다.
- A. Taflove 11.5장