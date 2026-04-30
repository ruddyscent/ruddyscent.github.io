---
layout: post
title: HoverPilot — RealFlight Link 네트워크 설정하기
subtitle: RealFlight Link로 시뮬레이터와 통신하는 방법을 알아보자.
tags: [
  hoverpilot,
  realflight,
  realflight-link,
  flightaxis,
  sitl,
  rc-airplane,
  network
]
cover-img: /assets/img/develop.jpeg
thumbnail-img: /assets/img/toy-plane.webp
share-img: /assets/img/develop.jpeg
author: 전경원
---
HoverPilot 프로젝트의 첫 단계는 시뮬레이터의 상태를 읽어오거나 제어할 수 있는 환경을 구축하는 것이다.  RealFlight Link를 통해 시뮬레이터와 통신하는 방법을 알아보자.

## RealFlight Link
RealFlight Link는 RealFlight 시뮬레이터와 외부 프로그램이 통신할 수 있도록 해주는 인터페이스다.  이를 통해 시뮬레이터에서 상태를 읽고, 조종 입력을 전달할 수 있다.

> RealFlight Link는 기존 FlightAxis 인터페이스의 새 이름으로, RealFlight 9부터 명칭이 변경되었다. 아직도 많은 자료에서 FlightAxis라는 명칭을 사용하고 있다.

## RealFlight Link 활성화
RealFlight 9부터는 RealFlight Link가 기본적으로 활성화되어 있다.  설정(Settings)에서 **RealFlight Link Enabled** 옵션이 **Yes**로 되어 있는지 확인한다.  또한 **Pause Physics When in Background** 옵션을 **No**로 설정한다.  이렇게 하면 RealFlight 창이 포커스를 잃어도 시뮬레이션이 계속 진행되게 할 수 있다.

RealFlight Link를 활성화하려면 다음 단계를 따른다.
- 상단 메뉴바에서 Simulation > Settings로 이동한다.
- Settings 창에서 왼쪽 패널에서 Physics를 선택한다.
- 오른쪽 패널에서 **RealFlight Link Enabled** 옵션을 **Yes**로 설정한다.
- **Pause Physics When in Background** 옵션을 **No**로 설정한다.
- RealFlight를 재시작한다.

![RealFlight의 Settings 창](/assets/img/hover-pilot/rf-settings-rflink-enabled.png)

## RealFlight Link 연결 시험

macOS에서 Windows 안에 있는 RealFlight에 접속해서, 네트워크 설정이 올바르게 구성되어 있는지 확인해보자.  Parallels Desktop은 기본적으로 macOS와 Windows가 서로 통신할 수 있도록 설정이 되어 있다.

![RealFlight Link 접속 시험](/assets/img/hover-pilot/rflink-connection-test.png)

RealFlight에는 RealFlight Link의 포트를 알려주는 기능이 없다. 따라서 어떤 포트가 열려 있는지 직접 알아내야 한다.  다행히도 Microsoft에서 제공하는 [TCPView](https://learn.microsoft.com/ko-kr/sysinternals/downloads/tcpview)라는 도구를 활용하면 시스템에 있는 모든 네트워크 엔드포인트를 간편하게 확인할 수 있다.  Windows에서 TCPView를 내려받아 압축을 풀어 실행하면, 시스템에 열려 있는 모든 TCP 및 UDP 엔드포인트를 볼 수 있다. 상단의 검색창에 **realflight**를 입력하여 RealFlight가 사용하는 포트가 열려 있는지 확인해보자.

![TCPView에서 RealFlight 포트 확인](/assets/img/hover-pilot/tcpview-rf-port-18083.png)

> 과거 FlightAxis Link는 UDP 기반으로 포트 9000를 사용했지만, RealFlight Link는 TCP 기반으로 변경되었으며 기본 포트는 18083을 사용하도록 변경되었다.

Parallels Desktop 외부에서 Windows로 접근하려면, Windows의 IP 주소를 알아야 한다. **명령 프롬프트**를 열고, `ipconfig` 명령어를 입력하여 네트워크 어댑터의 IP 주소를 확인하자.  **IPv4 주소**가 Windows의 IP 주소다.  이 주소를 사용하여 Parallels Desktop 외부에서 Windows로 접근할 수 있다.

![Windows의 IP 주소 확인](/assets/img/hover-pilot/windows-ipconfig-network-check.png)

이제 RealFlight Link가 활성화되고, 포트가 열려 있으며, Windows의 IP 주소도 확인되었다.  이제 실제로 외부에서 접근이 가능한지 확인한다. macOS에서 `nc` 명령어를 사용하여 Windows의 TCP 포트 18083에 실제로 연결이 가능한지 확인한다.  터미널에서 다음 명령어를 입력한다.

```bash
nc -vz <Windows_IP_ADDRESS> 18083
```

`nc`는 "netcat"의 줄임말로, 네트워크 연결을 테스트하는 데 사용되는 도구다.  `-v` 옵션은 verbose 모드로, 연결 시도에 대한 자세한 정보를 출력한다.  `-z` 옵션은 포트 스캔 모드로, 실제로 데이터를 전송하지 않고 포트가 열려 있는지만 확인한다.

![nc를 사용하여 RealFlight 포트 연결](/assets/img/hover-pilot/nc-rf-port-18083-success.png)

이제 기본적인 연결이 성공적으로 이루어졌다. 이 단계까지 완료되면, 외부 프로그램에서 RealFlight를 직접 제어할 수 있는 기반이 마련된다.  다음 단계에서는 RealFlight Link를 사용하여 시뮬레이터에서 상태를 읽고, 제어 신호를 전달하는 방법을 알아보자.
