---
layout: post
title: HoverPilot — RealFlight Link로 상태 정보 읽어오기
subtitle: SOAP 요청을 통해 XML 응답에서 비행 상태를 추출해보자
tags: [hoverpilot, realflight, realflight-link, flightaxis, python, sitl, rc-airplane]
cover-img: /assets/img/develop.jpeg
thumbnail-img: /assets/img/toy-plane.png
share-img: /assets/img/develop.jpeg
author: 전경원
---

[이전 글](/2026-04-13-hoverpilot-rflink-setup/)에서는 RealFlight Link 포트를 확인하고 macOS에서 Windows의 RealFlight와 TCP 연결에 성공했다.  이제 시뮬레이터의 상태를 실제로 읽어오는 단계로 넘어가자.

## 단순 TCP 연결만으로는 상태를 읽을 수 없다

RealFlight Link는 스트리밍 소켓이 아니다.  단순히 연결만 해서는 상태 데이터를 보내주지 않는다.  TCP로 연결한 뒤, SOAP 형식의 XML 요청을 보내야 그에 대한 응답으로 상태 정보를 받을 수 있다.  즉, 상태를 읽으려면 다음과 같은 흐름이 필요하다.

1. TCP 연결
2. 요청 전송 (SOAP)
3. 응답 수신 (HTTP + XML)
4. 데이터 파싱

## 코드 구성

상태 정보를 읽기 위한 코드를 다음과 같이 나누어 구성했다.

```
src/hoverpilot/
├── main.py
├── config.py
└── rflink/
    ├── client.py
    ├── protocol.py
    └── models.py
```

각 스크립트의 역할은 다음과 같다.

- `client.py` → TCP 연결을 관리하고, 요청/응답 흐름을 제어한다.
- `protocol.py` →  SOAP 요청을 생성하고, HTTP/XML 응답을 파싱한다.
- `models.py` → 파싱된 상태 데이터를 구조화하는 dataclass를 정의한다.

## 상태를 읽어오는 코드

실제로 상태를 읽는 코드는 생각보다 단순하다.

```python
state = client.request_state()
print(state.summary())
```

`request_state()` 메서드는 다음 단계를 수행한다.

1. TCP 연결
2. 컨트롤러 인터페이스 초기화
3. `ExchangeData` 요청 전송
4. HTTP 응답 수신
5. XML 파싱 → 상태 객체 생성

이후 action을 보내는 구조도 이와 거의 동일하게 이루어진다.

## 컨트롤러 인터페이스 초기화

RealFlight Link는 컨트롤러 인터페이스가 활성화된 이후에만 정상적인 상태 값을 반환한다.  이를 위해 다음 두 요청을 먼저 보낸다.

- `RestoreOriginalControllerDevice`: 컨트롤러 상태를 초기 상태로 되돌린다.
- `InjectUAVControllerInterface`: 컨트롤러 인터페이스를 RealFlight에 연결한다.

이 단계는 이후 `ExchangeData`로 상태와 입력을 주고받기 위한 초기화 과정이다. 

## 상태 요청 방식

RealFlight Link는 우리가 요청을 보내야만 응답을 주는 구조다.  그 요청은 일반적인 JSON이 아니라 **XML 형식**으로 작성해야 한다.

```xml
<ExchangeData>
  <pControlInputs>
    <m-selectedChannels>4095</m-selectedChannels>
    ...
  </pControlInputs>
</ExchangeData>
```

이 XML을 HTTP POST 요청에 담아서 RealFlight로 보내면, 그에 대한 응답으로 현재 비행 상태가 XML 형태로 돌아온다.

> 내부적으로는 SOAP 프로토콜을 사용하지만, XML을 HTTP 요청에 담아 보내고 응답을 파싱하는 방식으로 이해해도 충분하다.

## 응답 처리 방식

RealFlight Link의 응답은 HTTP 형태로 전달되며, 그 안에 XML 형태의 상태 데이터가 포함되어 있다.  처리 흐름은 다음과 같다.

1. HTTP 응답을 받는다
2. 응답 본문(XML)을 추출한다
3. XML을 파싱해서 상태 값을 읽는다

이제 이 값들이 실제로 무엇을 의미하는지 해석해야 한다.

## 상태 데이터 구조화

응답 XML에는 수십 개의 값이 포함되어 있다.  속도, 고도, 자세, 각속도, 위치, quaternion, 배터리 상태 등 비행 상태를 나타내는 다양한 정보가 함께 들어온다.  이 값을 XML에서 직접 꺼내서 사용하는 것도 가능하지만, 필드가 많고 구조가 일정하지 않아 코드가 금방 복잡해진다.  

그래서 파싱된 값을 `FlightAxisState`라는 dataclass로 묶어서 관리하도록 한다.  이렇게 구조화하면 다음과 같은 장점이 있다.

- 필요한 값에 이름으로 바로 접근할 수 있어 디버깅이 쉽다.
- 각 값의 의미가 코드에 그대로 드러나 가독성이 좋다.
- 이후 RL observation 벡터로 변환하기도 수월하다.

## 사람이 읽을 수 있는 형태로 출력

값이 제대로 들어오는지 확인하는 가장 직관적인 방법은 **값이 실제로 변하는지 확인하는 것**이다.  XML 응답에는 너무 많은 값이 포함되어 있어서 모든 값을 다 출력하면 오히려 확인하기 어려우므로, 중요한 값만 골라서 출력하도록 한다.

```python
print(state.summary())
```

확인 포인트는 다음과 같다.

- 위치
- 고도
- 속도
- 자세

이 상태에서 RealFlight의 Airplane Hover Trainer를 실행하고 기체를 움직이면, 출력되는 값이 실제로 변하는 것을 확인할 수 있다.  정밀한 분석은 아니지만, 값이 0에서 벗어나서 변화하는 것을 보면 일단 상태를 제대로 읽어오고 있는 것을 알 수 있다.

## 초기 상태가 0인 이유

처음 상태를 받아보면, 모든 값이 0으로 채워진 것처럼 보이는 경우가 있다.  이 상황은 보통 다음 두 가지 이유에서 발생한다.

1. 컨트롤러 인터페이스가 아직 초기화되지 않은 경우  
2. 시뮬레이터가 아직 상태를 정상적으로 생성하지 못한 경우  

즉, **응답은 오지만 실제 데이터가 채워지지 않은 상태**다.  그래서 다음과 같은 처리를 추가했다.

- zero-state 감지
- 최초 응답 디버그 출력

이렇게 해두면 문제 상황을 빠르게 구분할 수 있다.

> “응답이 없다”와 “응답은 있지만 값이 비어 있다”는 완전히 다른 문제다.

## 🔧 코드로 직접 확인해보기

이번 단계에서 사용한 코드는 아래 저장소에서 확인할 수 있다.

- HoverPilot 저장소: [https://github.com/ruddyscent/hoverpilot](https://github.com/ruddyscent/hoverpilot)
- 태그: `rflink-state-read`

```bash
git clone https://github.com/ruddyscent/hoverpilot.git
cd hoverpilot
git checkout rflink-state-read
```

이 태그는 **RealFlight Link로부터 상태 정보를 읽어오는 최소 구현이 완료된 시점**을 가리킨다.  이 코드를 실행하면, 시뮬레이터 상태가 실제로 읽혀서 출력되는 것을 직접 확인할 수 있다.

## 여기까지 한 일

이 단계까지 오면서 RealFlight와 통신할 수 있는 기본 구조를 구축했다.  TCP 연결 위에 SOAP 요청을 얹고, HTTP 응답을 받아 XML을 파싱한 뒤, 상태 데이터를 코드에서 다룰 수 있는 형태로 정리했다.

## 다음 단계

이제 단순히 상태를 읽는 단계에서 나아가, 시뮬레이터에 직접 영향을 주는 단계로 넘어간다.  다음 글에서는 조종 입력을 보내고, 상태와 연결된 피드백 루프를 구성하는 과정을 다룬다.

