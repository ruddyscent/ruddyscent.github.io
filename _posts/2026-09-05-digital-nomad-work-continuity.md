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

사무실이 없는 프리랜서로 지내다 보니 작업 시간과 장소가 자주 바뀝니다. 기본 작업 장소는 집이지만 노트북을 들고 나간 뒤에도 집에 있는 GPU 머신, Edge 머신, Windows 머신에 수시로 접속해야 합니다. 노트북 한 대에서 모든 작업을 실행하기에는 성능과 저장 공간이 부족하기도 합니다.

제가 원한 것은 어느 장소에서나 집의 머신에 접속하는 것만이 아니었습니다. 카페에서 노트북을 덮고 이동한 뒤 다시 열어도 실행하던 터미널과 AI 에이전트 세션을 찾을 수 있어야 했습니다. 이 환경에서는 네트워크 연결과 작업 세션의 수명을 따로 다룹니다.

```text
외부의 MacBook
      │
      │  1. DDNS로 집의 공유기를 찾음
      │  2. VPN으로 암호화된 통로를 만듦
      ▼
ASUS RT-AX86U
      │
      ├── GPU 머신     ── SSH ── Herdr ── 작업 세션
      ├── Edge 머신    ── SSH ── Herdr ── 작업 세션
      └── Windows 머신 ── RDP 또는 SSH

노트북이 잠들거나 접속 장소가 바뀜
      └── VPN 재연결 ── herdr --remote <host> ── 기존 세션으로 복귀
```

DDNS는 집을 찾는 주소를 제공하고 VPN은 집 안 네트워크까지 안전한 통로를 만듭니다. SSH는 각 머신의 셸로 들어가며 Herdr는 클라이언트 연결이 끊긴 동안에도 원격 머신의 작업 세션을 유지합니다. 어느 하나가 나머지 역할까지 대신하지는 않습니다.

## 시작하기 전에 확인할 조건

공유기 VPN 서버에 인터넷에서 직접 접속하려면 인터넷 서비스 제공자(ISP)가 공유기에 공인 WAN IP를 할당해야 합니다. ASUS의 [WireGuard 서버 설정 안내](https://www.asus.com/us/support/faq/1048280/)도 이 조건을 먼저 확인하도록 설명합니다. 공유기 WAN 주소와 외부에서 조회한 공인 IP가 다르거나 WAN 주소가 사설 대역이라면 통신사 공유기 아래에 이중 NAT가 있거나 CGNAT를 사용하는 환경일 수 있습니다. 이 경우 DDNS 이름을 만들더라도 외부 연결이 공유기까지 도착하지 않습니다.

통신사 장비를 브리지 모드로 바꾸거나 공인 IP를 신청할 수 있는지 먼저 확인합니다. 이를 바꿀 수 없다면 중계 서버를 쓰는 메시 VPN과 같은 다른 구성이 필요합니다. 이 글은 ASUS 공유기가 공인 WAN IP를 직접 받는 환경을 기준으로 합니다.

집과 외부 네트워크의 주소 대역이 겹치지 않게 잡는 것도 중요합니다. 집과 카페가 모두 `192.168.1.0/24`를 사용하면 VPN 연결 뒤에도 운영체제가 목적지를 어느 쪽으로 보낼지 잘못 판단할 수 있습니다. 집의 LAN 대역은 흔히 쓰지 않는 사설 주소 대역으로 정하고, GPU와 Edge 머신에는 DHCP 예약으로 주소를 고정합니다.

## DDNS로 집의 공유기를 찾는다

가정용 인터넷의 공인 IP는 바뀔 수 있습니다. 동적 도메인 이름 시스템(Dynamic DNS, DDNS)을 설정하면 현재 공인 IP를 기억하는 대신 일정한 호스트 이름으로 공유기를 찾을 수 있습니다.

RT-AX86U 관리 화면에서 `WAN > DDNS`로 이동해 DDNS를 켜고 호스트 이름을 등록합니다. 이 주소는 뒤에서 내보낼 VPN 프로필의 서버 주소로 들어갑니다. DDNS 등록 상태가 `Active`인지 확인한 뒤 휴대전화 테더링처럼 집 밖의 네트워크에서 이름이 조회되는지 시험합니다.

```text
고정해서 기억할 값                    바뀔 수 있는 값
vpn-home.example.invalid ── DDNS ──> 현재 집의 공인 IP
```

DDNS는 주소록일 뿐 접근 제어 장치가 아닙니다. 호스트 이름이 알려졌다고 바로 내부 머신에 들어갈 수 있는 것은 아니지만 공개된 주소는 스캔 대상이 될 수 있습니다. 실제 DDNS 호스트 이름은 블로그 화면이나 예제에 싣지 않습니다.

공유기의 `Web Access from WAN`은 끕니다. 외부에서 공유기 관리 화면을 직접 열 필요는 없습니다. VPN에 먼저 접속한 뒤 LAN 주소로 관리하면 됩니다. SSH 포트도 인터넷에 포워딩하지 않습니다. 인터넷에 공개하는 입구를 VPN 하나로 줄이는 편이 관리하기 쉽습니다.

## WireGuard로 집 안 네트워크에 들어간다

공유기 관리 화면에서 `VPN > VPN Server > WireGuard VPN`을 켭니다. WireGuard는 `3.0.0.4.388.xxxxx` 이후 펌웨어에서 지원되므로 설정 전에 최신 안정 버전으로 갱신합니다. `+` 버튼으로 MacBook용 클라이언트를 추가하고 일반 장치용 설정을 적용한 뒤 구성 파일을 내보냅니다. `macbook-work`처럼 어느 장치에 발급했는지 알 수 있는 이름을 쓰면 분실한 장치의 접근 권한만 찾아서 폐기하기 쉽습니다.

이 구성의 목적은 집 안 머신에 접속하는 것이므로 분할 터널(split tunnel)을 사용합니다. WireGuard 구성의 `AllowedIPs`에는 VPN 터널 주소와 집의 LAN 대역만 포함합니다. 예를 들어 집의 LAN이 `192.168.80.0/24`라면 이 대역으로 가는 트래픽은 VPN을 통하고 일반 인터넷 트래픽은 현재 Wi-Fi나 테더링을 그대로 사용합니다.

```ini
AllowedIPs = <VPN 터널 대역>, 192.168.80.0/24
```

내보낸 값을 임의로 바꾸기 전에 원본 구성으로 먼저 연결을 시험하고, `AllowedIPs`가 `0.0.0.0/0, ::/0`이라면 모든 인터넷 트래픽을 집으로 보내는 전체 터널이라는 점을 확인합니다. 전체 터널은 공용 Wi-Fi의 인터넷 트래픽까지 집을 경유하지만 집의 업로드 대역폭과 지연 시간이 추가됩니다.

WireGuard의 `.conf` 파일과 QR 코드에는 MacBook의 개인키가 들어 있습니다. 공개 저장소, 블로그 첨부 파일, 공용 클라우드 폴더에 올리지 않습니다. 장치마다 별도 클라이언트와 키를 발급하면 한 장치를 잃어버렸을 때 공유기에서 해당 클라이언트만 삭제할 수 있습니다. 사용하지 않는 PPTP, IPSec 같은 VPN 서버도 꺼 둡니다.

## MacBook에서 VPN을 연결한다

Mac App Store에서 [공식 WireGuard 앱](https://apps.apple.com/us/app/wireguard/id1451685025?mt=12)을 설치합니다. 앱에서 `Import tunnel(s) from file`을 선택하고 공유기에서 내보낸 `.conf` 파일을 가져온 뒤 macOS의 VPN 구성 추가를 승인합니다. 터널 이름은 공유기에 만든 클라이언트 이름과 맞춰 두면 어느 키가 어느 장치의 것인지 찾기 쉽습니다.

WireGuard는 사용자 이름과 암호를 매번 입력하지 않고 가져온 개인키로 MacBook을 인증합니다. 연결은 간단하지만 잠금이 풀린 노트북을 빼앗기면 VPN도 바로 켤 수 있습니다. FileVault와 화면 잠금을 함께 설정하고, 장치를 잃어버리면 공유기에서 해당 WireGuard 클라이언트를 즉시 삭제합니다.

설정을 마쳤으면 집 Wi-Fi를 끄고 휴대전화 테더링으로 바꿔 다음 순서대로 확인합니다.

1. DDNS 이름이 현재 집의 공인 IP로 조회되는지 확인합니다.
2. WireGuard 앱에서 터널을 켜고 최근 handshake가 표시되는지 확인합니다.
3. 공유기의 LAN 주소에 접근되는지 확인합니다.
4. GPU 머신과 Edge 머신의 고정 주소로 SSH 접속을 시험합니다.
5. 노트북을 잠자기 상태로 만들었다가 깨운 뒤 VPN과 SSH를 다시 연결합니다.

처음부터 카페에서 시험하면 DDNS, 방화벽, VPN, SSH 중 어디에서 막혔는지 구분하기 어렵습니다. 집에서 테더링으로 외부 조건을 만든 뒤 한 단계씩 확인하는 편이 빠릅니다.

## SSH는 암호 대신 공개키로 연결한다

흔히 Passwordless SSH라고 부르지만 서버 계정의 암호를 비워 둔다는 뜻은 아닙니다. MacBook의 개인키로 서명하고 서버에 등록한 공개키로 확인하는 공개키 인증입니다. 개인키에는 암호를 설정하고 macOS의 `ssh-agent`와 키체인이 필요할 때 잠금을 풀도록 구성합니다.

MacBook에서 서버 접속 전용 Ed25519 키를 만듭니다. 기존 기본 키를 무심코 덮어쓰지 않도록 별도 파일명을 사용합니다.

```bash
ssh-keygen -t ed25519 -a 64 -f ~/.ssh/id_ed25519_home_remote
```

VPN에 연결한 상태에서 공개키를 각 서버의 `~/.ssh/authorized_keys`에 등록합니다. 아래 이름과 주소는 예시이며 실제 값으로 바꿉니다.

```bash
ssh-copy-id -i ~/.ssh/id_ed25519_home_remote.pub myuser@192.168.80.10
ssh-copy-id -i ~/.ssh/id_ed25519_home_remote.pub myuser@192.168.80.20
```

MacBook의 `~/.ssh/config`에는 기억하기 쉬운 별칭을 둡니다.

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

서버 쪽 설정 파일 경로는  `/etc/ssh/sshd_config`입니다. Ubuntu처럼 `sshd_config.d`를 읽는 환경에서는 기존 파일을 크게 고치지 않고 `/etc/ssh/sshd_config.d/90-remote-work.conf`에 다음 설정을 둡니다.

```sshconfig
PubkeyAuthentication yes
PasswordAuthentication no
KbdInteractiveAuthentication no
PermitRootLogin no
AllowUsers myuser
```

암호 로그인을 끄기 전에 새 터미널에서 공개키 접속이 되는지 반드시 확인합니다. 기존 SSH 창은 닫지 않은 채 설정 문법을 검사하고 서비스를 다시 읽힙니다.

```bash
sudo sshd -t
sudo systemctl reload ssh
```

`sshd -t`가 오류를 출력하면 reload하지 않고 파일을 먼저 고칩니다. 배포판과 OpenSSH 버전에 따라 포함 파일이나 서비스 이름이 다를 수 있으므로 `sshd -T`로 최종 적용값도 확인합니다. `PasswordAuthentication`, `PubkeyAuthentication`, `PermitRootLogin` 등 각 항목의 의미는 [OpenSSH `sshd_config` 매뉴얼](https://man.openbsd.org/sshd_config)에서 확인할 수 있습니다.

SSH 개인키를 서버 사이에 복사하거나 모든 노트북이 같은 키를 공유하게 만들지는 않습니다. 장치마다 키를 따로 만들고 분실한 장치의 공개키 한 줄만 서버의 `authorized_keys`에서 제거합니다. 에이전트 포워딩도 필요한 경우가 아니면 켜지 않습니다.

## Herdr에 작업의 수명을 맡긴다

VPN과 SSH는 연결을 복구하지만 끊어진 터미널에서 실행하던 프로세스까지 유지하지는 않습니다. 원격 머신마다 Herdr를 실행해 workspace와 pane을 그 머신에 남겨 둡니다. MacBook에서는 SSH 별칭으로 원격 Herdr에 바로 붙습니다.

```bash
herdr --remote gpu-home
herdr --remote edge-home
```

노트북을 덮거나 Wi-Fi가 바뀌면 VPN과 SSH 연결은 끊어질 수 있습니다. 그래도 원격 Herdr 서버와 그 안의 pane은 원격 머신에서 계속 실행됩니다. 새 장소에서 VPN을 다시 연결하고 같은 `herdr --remote` 명령을 실행하면 기존 workspace로 돌아갑니다. 설치와 Codex integration, detach 방법은 앞서 작성한 [Ghostty와 Herdr로 Codex CLI 작업 환경 나누기](/2026-08-24-ghostty-herdr-codex-cli-workflow/)에서 자세히 다뤘습니다.

```text
MacBook의 수명        ── 열기 ── 이동 ── 잠자기 ── 다시 열기
VPN·SSH 연결의 수명   ── 연결 ── 끊김 ──────────── 재연결
Herdr 작업 세션의 수명 ── 시작 ─────────────────── 계속 실행
```

다만 Herdr는 전원 장애를 없애 주지 않습니다. 머신이 재부팅되면 실행 중이던 학습이나 빌드 프로세스 자체는 중단됩니다. 긴 GPU 학습은 체크포인트를 주기적으로 저장하고, 반드시 다시 떠야 하는 서비스는 `systemd`나 컨테이너의 재시작 정책으로 관리합니다.

## 연결보다 먼저 실패를 준비한다

원격 작업 환경은 집의 전원과 인터넷, 공유기, 각 머신 가운데 하나만 멈춰도 들어갈 수 없습니다. 다음 설정은 접속 방법만큼 중요합니다.

- 공유기와 서버 펌웨어 및 보안 업데이트를 주기적으로 적용합니다.
- 공유기 관리 계정에는 길고 고유한 암호를 사용하고 WireGuard 클라이언트는 장치마다 따로 발급합니다.
- GPU·Edge 머신은 절전으로 SSH가 끊기지 않도록 전원 정책을 확인합니다.
- BIOS의 정전 후 자동 켜기와 필요한 경우 Wake-on-LAN을 설정하되 VPN 안에서만 사용합니다.
- 잦은 정전이 있는 환경이라면 공유기와 핵심 머신에 UPS를 연결합니다.
- 코드 변경은 작은 단위로 커밋하고 원격 Git 저장소에도 올립니다.
- 데이터와 모델 체크포인트는 작업 머신 밖에도 백업하며 복원 절차를 직접 시험합니다.
- MacBook에는 FileVault, 짧은 화면 잠금 시간, 나의 찾기를 설정합니다. Apple은 [FileVault가 로그인 암호 없이는 저장된 데이터에 접근하지 못하도록 보호한다](https://support.apple.com/guide/mac-help/how-does-filevault-work-on-a-mac-flvlt001/mac)고 설명합니다.

노트북을 잃어버렸을 때의 순서도 미리 적어 둡니다. VPN 프로필을 폐기하고 각 서버의 `authorized_keys`에서 해당 노트북의 공개키를 제거한 뒤, GitHub를 비롯한 개발 서비스의 세션과 토큰을 취소합니다. 집 밖에 있는 동안 이 작업을 할 수 있도록 공유기 설정 백업과 비상 연락 경로도 준비합니다.
