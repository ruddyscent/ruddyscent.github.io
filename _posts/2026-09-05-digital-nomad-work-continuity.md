---
layout: post
title: 장소가 바뀌어도 작업을 이어가는 원격 개발 환경
subtitle: 노트북은 접속점으로 두고 실제 작업은 집의 여러 머신에 남겨 둔다
tags: [remote-development, vpn, ssh, herdr, digital-nomad, security]
cover-img: /assets/img/develop.jpeg
thumbnail-img: /assets/img/editor.webp
share-img: /assets/img/develop.jpeg
author: 전경원
description: ASUS 공유기의 DDNS와 WireGuard VPN, SSH 공개키 인증, Herdr를 연결해 외부에서도 집의 GPU·Edge·Windows 머신에 안전하게 접속하고 작업 세션을 이어가는 구성.
---

사무실이 없는 프리랜서로 지내다 보니 작업 시간과 장소가 자주 바뀝니다. 기본 작업 장소는 집이지만 외부에서도 집에 있는 GPU 머신, Edge 머신, Windows 머신에 접속해야 합니다. 노트북 한 대에서 모든 작업을 실행하기에는 성능과 저장 공간이 부족합니다.

여기서 중요한 것은 접속 자체보다 작업의 연속성입니다. 노트북의 연결은 끊겨도 괜찮지만 실제 작업은 집의 머신에서 계속되어야 합니다. 그래서 네트워크 연결과 작업 세션의 수명을 따로 관리합니다.

```text
외부의 MacBook
      │
      ├── WireGuard ── 집 안 네트워크에 연결
      ├── SSH       ── 작업할 머신을 선택
      ▼
ASUS RT-AX86U
      │
      ├── GPU 머신  ── Herdr ── 작업 세션
      ├── Edge 머신 ── Herdr ── 작업 세션
      └── Windows 머신 ── RDP 또는 SSH
```

동적 도메인 이름 시스템(Dynamic DNS, DDNS)은 집의 공유기를 찾는 주소를 제공합니다. WireGuard는 집 안 네트워크까지 암호화된 통로를 만들고 SSH는 그 안에서 작업할 머신을 고릅니다. Herdr는 노트북 연결이 끊겨도 원격 머신의 터미널과 AI 에이전트 세션을 유지합니다.

## DDNS와 WireGuard로 집에 들어간다

공유기 VPN 서버에 외부에서 접속하려면 인터넷 서비스 제공자(ISP)가 공유기에 공인 WAN IP를 할당해야 합니다. 공유기 WAN 주소가 사설 대역이거나 외부에서 조회한 공인 IP와 다르다면 이중 NAT나 CGNAT 환경일 수 있습니다. 이 경우에는 공인 IP를 신청하거나 메시 VPN 같은 다른 구성이 필요합니다.

설정하기 전에 집의 LAN도 확인합니다. 카페나 숙소와 같은 `192.168.1.0/24` 대역을 사용하면 VPN이 연결돼도 운영체제가 목적지를 잘못 판단할 수 있습니다. 집은 비교적 덜 흔한 사설 대역을 사용하고 GPU·Edge·Windows 머신에는 DHCP 예약으로 내부 IP를 고정합니다. 공유기와 작업 머신에는 최신 안정 펌웨어와 보안 업데이트를 적용합니다.

RT-AX86U의 `WAN > DDNS`에서 DDNS를 켜고 호스트 이름을 등록합니다. 공인 IP가 바뀌어도 이 이름으로 공유기를 찾을 수 있습니다. DDNS는 주소록일 뿐 접근 제어 장치는 아니므로 실제 호스트 이름은 공개하지 않습니다. 공유기의 `Web Access from WAN`과 SSH 포트 포워딩도 끄고 외부에 공개하는 입구를 VPN 하나로 줄입니다.

이어서 `VPN > VPN Server > WireGuard VPN`에서 MacBook용 클라이언트를 추가하고 구성 파일을 내보냅니다. ASUS의 [WireGuard 서버 설정 안내](https://www.asus.com/us/support/faq/1048280/)에 따르면 이 기능은 `3.0.0.4.388.xxxxx` 이후 펌웨어에서 지원됩니다. 장치마다 클라이언트를 따로 만들면 노트북을 잃어버렸을 때 해당 키만 폐기할 수 있습니다.

이 환경에서는 집의 LAN으로 가는 트래픽만 VPN에 넣는 분할 터널(split tunnel)을 사용합니다. 내보낸 구성의 `AllowedIPs`에 WireGuard 터널 대역과 집의 LAN 대역이 들어 있는지 확인합니다. `0.0.0.0/0, ::/0`이면 일반 인터넷 트래픽까지 집을 거치는 전체 터널입니다. WireGuard의 `.conf` 파일과 QR 코드에는 개인키가 있으므로 공개 저장소나 블로그에 올리지 않습니다.

Mac App Store에서 [공식 WireGuard 앱](https://apps.apple.com/us/app/wireguard/id1451685025?mt=12)을 설치한 뒤 `Import tunnel(s) from file`로 구성 파일을 가져옵니다. 집 Wi-Fi 대신 휴대전화 테더링에서 터널을 켭니다. 최근 handshake가 표시되고 고정한 내부 IP에 접근할 수 있으면 WireGuard 설정은 끝납니다. 일부 네트워크에서는 유휴 상태가 길어지면 연결이 반복해서 복구되지 않을 수 있습니다. 이때만 `PersistentKeepalive = 25`를 검토합니다.

## SSH 별칭으로 작업할 머신을 고른다

Passwordless SSH는 서버 계정의 암호를 비우는 설정이 아니라 공개키로 인증하는 방식입니다. MacBook에서 전용 키를 만들 때 개인키 암호를 설정하고, VPN에 연결한 상태에서 공개키를 각 서버에 등록합니다.

```bash
ssh-keygen -t ed25519 -a 64 -f ~/.ssh/id_ed25519_home_remote
ssh-copy-id -i ~/.ssh/id_ed25519_home_remote.pub myuser@192.168.80.10
ssh-copy-id -i ~/.ssh/id_ed25519_home_remote.pub myuser@192.168.80.20
```

macOS에 `ssh-copy-id`가 없다면 공개키 내용을 서버의 `~/.ssh/authorized_keys`에 직접 추가합니다. 개인키 파일은 전송하지 않습니다.

반복해서 접속할 머신은 `~/.ssh/config`에 별칭을 둡니다.

```sshconfig
Host gpu-home
    HostName 192.168.80.10
    User myuser
    IdentityFile ~/.ssh/id_ed25519_home_remote
    IdentitiesOnly yes
    AddKeysToAgent yes
    UseKeychain yes
    ServerAliveInterval 30
    ServerAliveCountMax 3

Host edge-home
    HostName 192.168.80.20
    User myuser
    IdentityFile ~/.ssh/id_ed25519_home_remote
    IdentitiesOnly yes
    AddKeysToAgent yes
    UseKeychain yes
    ServerAliveInterval 30
    ServerAliveCountMax 3
```

최초 접속 때 표시되는 서버 호스트 키 지문은 집에서 직접 확인합니다. 이후 호스트 키 변경 경고가 나오면 `known_hosts`를 바로 지우지 말고 서버가 재설치됐는지부터 확인합니다.

공개키 접속을 새 터미널에서 확인한 뒤 서버의 암호 로그인과 root 로그인을 끕니다. Ubuntu에서는 `/etc/ssh/sshd_config` 또는 `/etc/ssh/sshd_config.d/` 아래의 설정 파일에 다음 값을 둡니다.

```sshconfig
PubkeyAuthentication yes
PasswordAuthentication no
KbdInteractiveAuthentication no
PermitRootLogin no
AllowUsers myuser
```

기존 SSH 창은 닫지 않은 채 문법을 검사하고 설정을 다시 읽힙니다. `sshd -t`가 오류를 출력하면 reload하지 않고 파일부터 고칩니다.

```bash
sudo sshd -t
sudo systemctl reload ssh
```

배포판마다 서비스 이름과 설정 파일 구성이 다를 수 있습니다. 각 항목은 [OpenSSH `sshd_config` 매뉴얼](https://man.openbsd.org/sshd_config)에서 확인합니다. SSH 개인키는 서버 사이에 복사하지 않고 장치마다 따로 관리합니다. Windows 머신은 RDP나 OpenSSH로 접속하되 서비스 포트를 인터넷에 직접 열지 않고 VPN 안에서만 사용합니다.

## Herdr로 하던 작업에 돌아간다

VPN과 SSH를 다시 연결해도 일반 터미널에서 실행하던 프로세스까지 살아나지는 않습니다. 원격 머신마다 Herdr를 실행해 workspace와 pane을 그 머신에 남겨 두고 MacBook에서는 SSH 별칭으로 바로 붙습니다.

```bash
herdr --remote gpu-home
herdr --remote edge-home
```

카페에서 작업하다 노트북을 덮고 이동한다고 해보겠습니다. 새 장소에서 WireGuard를 다시 연결한 뒤 `herdr --remote gpu-home`을 실행하면 GPU 머신에 남아 있던 workspace로 돌아갑니다. 네트워크 연결은 새로 만들어지지만 작업 세션은 원격 머신에 그대로 남아 있습니다.

```text
MacBook             ── 잠자기 ── 이동 ── 다시 연결
WireGuard · SSH     ── 끊김 ──────────── 새 연결
원격 Herdr 작업 세션 ── 계속 실행
```

Herdr 설치와 Codex integration, detach 방법은 [Ghostty와 Herdr로 Codex CLI 작업 환경 나누기](/2026-08-24-ghostty-herdr-codex-cli-workflow/)에서 자세히 다뤘습니다. Herdr도 집의 머신이 꺼지는 상황까지 막지는 못합니다. 재부팅 뒤에도 다시 떠야 하는 서비스는 `systemd`나 컨테이너 재시작 정책으로 관리하고 긴 GPU 학습은 체크포인트를 남깁니다.

## 연결이 막히면 앞에서부터 확인한다

접속이 되지 않을 때는 도구를 한꺼번에 의심하지 않고 연결 순서대로 확인합니다.

```text
WireGuard handshake
        ↓
집의 내부 IP에 접근
        ↓
ssh gpu-home
        ↓
herdr --remote gpu-home
```

handshake가 없다면 DDNS, 공인 WAN IP, 공유기의 WireGuard 서버부터 봅니다. 내부 IP에는 접근되지만 SSH가 실패하면 서버 주소와 공개키를 확인합니다. SSH는 되는데 workspace가 보이지 않을 때 Herdr 상태를 살펴봅니다.

## 연결이 아닌 실패도 준비한다

원격 작업 환경은 집의 전원이나 인터넷, 공유기, 작업 머신 가운데 하나만 멈춰도 사용할 수 없습니다. 설정을 마쳤다면 다음 네 가지를 함께 준비합니다.

- GPU·Edge 머신이 절전으로 꺼지지 않도록 전원 정책과 정전 후 자동 켜기를 확인합니다.
- WireGuard 클라이언트와 SSH 키는 장치마다 분리하고 MacBook에는 FileVault와 화면 잠금을 설정합니다.
- 정전에 대비한 UPS에는 공유기와 핵심 머신뿐 아니라 통신사 광모뎀(Optical Network Terminal, ONT)도 연결합니다.
- 코드 변경은 원격 Git 저장소에 올리고 데이터와 모델 체크포인트는 작업 머신 밖에도 백업한 뒤 복원을 시험합니다.
- 노트북을 잃어버리면 WireGuard 클라이언트, SSH 공개키, 개발 서비스의 세션과 토큰을 차례로 폐기합니다. 이 작업을 MacBook 없이도 실행할 별도 관리 장치와 복구 코드를 준비합니다.
- IPv6를 사용한다면 공유기와 각 머신의 IPv6 인바운드 방화벽도 확인합니다.

마지막으로 집 Wi-Fi를 끄고 휴대전화 테더링에서 전체 경로를 시험합니다.

1. WireGuard를 켜고 최근 handshake를 확인합니다.
2. 고정한 내부 IP에 접근합니다.
3. `ssh gpu-home`으로 공개키 접속합니다.
4. `herdr --remote gpu-home`으로 기존 workspace에 들어갑니다.
5. MacBook을 잠자기 상태로 만들었다가 같은 workspace로 돌아옵니다.

이 다섯 단계를 통과하면 작업 연속성을 위한 기본 경로가 완성됩니다.
