# Jetson Xavier NX Developer Kit 설정 방법

## JetPack SDK 설치
[Jetson Xavier NX Developer Kit - Get Started](https://developer.nvidia.com/embedded/learn/get-started-jetson-xavier-nx-devkit)를 참고해서 설정한다. 다만, 2026년 7월 15일 현재 Jetson Xavier NX Developer Kit은 EOL(End of Life) 상태로 최신 JetPack 릴리스에서는 더 이상 지원되지 않는다. 다행히도 [JetPack Archive](https://developer.nvidia.com/embedded/jetpack-archive)에서 Jetson Xavier NX Developer Kit을 지원하는 JetPack 릴리스를 찾을 수 있다. 이 페이지에 따르면 Jetson Xavier NX Developer Kit을 지원하는 마지막 JetPack 릴리스는 [JetPack 5.1.6](https://developer.nvidia.com/embedded/jetpack-sdk-516)이다. 이 페이지에서 SD Card Image를 내려받아 설치한다.

맥에서 JetPack SDK 이미지를 microSD 카드에 쓰려면, Etcher를 사용하면 편한데, Etcher는 [balenaEtcher](https://etcher.balena.io/)로 이름이 변경되었다. balenaEtcher는 다음과 같이 brew를 이용해서 설치한다.

```bash
brew install --cask balenaetcher
```

## Root on SSD 설정
https://www.youtube.com/watch?v=ZK5FYhoJqIg
https://github.com/jetsonhacks/rootOnNVMe