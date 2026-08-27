# Jetson Xavier NX Developer Kit 설정

## JetPack SDK 설치
[Jetson Xavier NX Developer Kit - Get Started](https://developer.nvidia.com/embedded/learn/get-started-jetson-xavier-nx-devkit)를 참고해 설정합니다. 다만 2026년 7월 15일 현재 Jetson Xavier NX Developer Kit은 EOL(End of Life) 상태라 최신 JetPack 릴리스에서 더 이상 지원되지 않습니다. [JetPack Archive](https://developer.nvidia.com/embedded/jetpack-archive)에서는 Jetson Xavier NX Developer Kit을 지원하는 JetPack 릴리스를 찾을 수 있습니다. 이 페이지에 따르면 마지막 지원 릴리스는 [JetPack 5.1.6](https://developer.nvidia.com/embedded/jetpack-sdk-516)입니다. 이 페이지에서 SD Card Image를 내려받아 설치합니다.

Mac에서 JetPack SDK 이미지를 microSD 카드에 쓰려면 Etcher를 쓰면 편합니다. Etcher는 [balenaEtcher](https://etcher.balena.io/)로 이름이 바뀌었으며, Homebrew에서는 다음처럼 설치합니다.

```bash
brew install --cask balenaetcher
```

## Root on SSD 설정
https://www.youtube.com/watch?v=ZK5FYhoJqIg
https://github.com/jetsonhacks/rootOnNVMe
