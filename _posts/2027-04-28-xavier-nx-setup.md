# 1-3. No Assigned Problems


NVIDIA Jetson에서 JetPack 버전을 확인하는 가장 확실한 방법은 터미널에서 sudo apt-cache show nvidia-jetpack 명령어를 입력하는 것입니다. 또한, cat /etc/nv_tegra_release로 L4T(Linux for Tegra) 버전을 확인하거나, jtop 툴을 설치하여 확인할 수도 있습니다. 
블로그
블로그
 +3
주요 확인 방법:
메타 패키지 조회 (추천): sudo apt-cache show nvidia-jetpack 명령어를 입력하여 Version: 항목을 확인합니다 (예: 6.2-b123).
L4T 버전 확인: cat /etc/nv_tegra_release 명령어를 사용하여 JetPack의 기반이 되는 L4T 버전을 확인합니다.
jtop 모니터링 툴: sudo apt-get install python3-pip 및 sudo pip3 install -U jetson-stats 설치 후 jtop 명령어를 실행하여 GUI 화면에서 버전을 바로 확인합니다.
패키지 목록 조회: dpkg-query --show nvidia-l4t-core 명령어로 L4T 핵심 패키지 버전을 확인합니다. 




- Jetson Xavier NX Developer Kit
  - [Getting Started](https://developer.nvidia.com/embedded/learn/get-started-jetson-xavier-nx-devkit)

    The Jetson Xavier NX Developer Kit has reached EOL and is no longer available for purchase. The Jetson Xavier NX production module remains available; see the Jetson Product Lifecycle page for details.

    JetPack 5.x releases, built on the Jetson Linux r35 codeline, support Jetson Nano developer kits and modules. The most recent sustaining releases can be found at the JetPack Archive and the Jetson Linux Archive.

    If you have any questions, please visit the Jetson Xavier NX Forum.

  - [User Guide](https://developer.nvidia.com/embedded/downloads#?search=Jetson%20Xavier%20NX%20Developer%20Kit%20User%20Guide)
  - [Carrier Board Specifications](https://developer.nvidia.com/embedded/downloads#?search=Jetson%20Xavier%20NX%20Developer%20Kit%20Carrier%20Board%20Specification)


- [Jetson Xavier NX - SSD에서 실행](https://www.youtube.com/watch?v=ZK5FYhoJqIg)
- [rootOnNVMe](https://github.com/jetsonhacks/rootOnNVMe)
- [Boot Jetson Xavier from M.2 NVMe SSD](https://www.seeedstudio.com/blog/2020/06/22/boot-jetson-xavier-from-m-2-ssd/?srsltid=AfmBOoqpUxYX0ObJjIRQx5bKuApDK_yNxG3hp8C6ylTa0HY7Udi405bj)
- [Your First Jetson Container](https://developer.nvidia.com/embedded/learn/tutorials/jetson-container)
- [JetPack Archive](https://developer.nvidia.com/embedded/jetpack-archive)
- [JetPack SDK](https://developer.nvidia.com/embedded/jetpack-sdk-516)
- [Jetson Linux](https://developer.nvidia.com/embedded/jetson-linux-r3564)

JetPack 5.1.6
Jetson AGX Orin Series, Jetson Orin NX Series, Jetson Orin Nano Series, Jetson Xavier NX series, Jetson AGX Xavier Series, [L4T 35.6.4]

- Jetson Xavier NX Developer Kit
  - [Getting Started](https://developer.nvidia.com/embedded/learn/get-started-jetson-xavier-nx-devkit)
  - [User Guide](https://developer.nvidia.com/embedded/downloads#?search=Jetson%20Xavier%20NX%20Developer%20Kit%20User%20Guide)
  - [Carrier Board Specifications](https://developer.nvidia.com/embedded/downloads#?search=Jetson%20Xavier%20NX%20Developer%20Kit%20Carrier%20Board%20Specification)

