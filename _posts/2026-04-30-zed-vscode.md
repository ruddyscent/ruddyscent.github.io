---
layout: post
title: ML 엔지니어가 본 Zed와 VSCode — 빠른 에디터와 검증된 개발 환경의 역할 분담
subtitle: Dev Container와 Jupyter가 기본인 작업 환경에서 Zed를 써본 기록
tags: [vscode, zed, editor, developer-tools, devcontainer, jupyter, machine-learning]
cover-img: /assets/img/develop.jpeg
thumbnail-img: /assets/img/vscode.webp
share-img: /assets/img/develop.jpeg
author: 전경원
---

최근 [달레줄레 팟캐스트](https://podcasts.apple.com/ca/podcast/%EB%8B%AC%EB%A0%88%EC%A4%84%EB%A0%88/id1632529597)를 듣다가 [Zed](https://zed.dev/)라는 에디터를 알게 됐습니다. "Rust로 만든 빠른 에디터"라는 한 줄 소개가 흥미로워서 곧장 설치해 며칠 동안 실제 작업에 써봤습니다. Zed는 속도와 협업 면에서 분명한 강점이 있었습니다. 다만 Dev Container와 Jupyter Notebook처럼 VSCode에서 검증된 워크플로우는 당분간 그대로 쓰는 편이 합리적이라는 판단이 들었습니다.

## 1. 첫인상은 단연 속도

Zed를 처음 실행하면 가장 먼저 느껴지는 건 **속도**입니다. 실행이 빠르고 타이핑 반응도 즉각적이며 전체적으로 가볍습니다. VSCode를 오래 쓰다 보면 익숙해진 미세한 끊김이 Zed에는 거의 없습니다.

VSCode는 Electron 위에서 동작하지만 Zed는 **Rust로 작성되었고 GPU 렌더링을 사용**합니다. 텍스트를 그리는 일까지 GPU에 맡기는 구조라 오래 켜둬도 무거워지는 느낌이 거의 없습니다. 평소 VSCode 창을 서너 개씩 띄워 두는 입장에서는 반가운 차이입니다.

## 2. 그래서 실제 작업에는 어떻게 나눠 쓸까

빠르다는 점은 분명한 장점이지만 에디터의 가치는 속도만으로 결정되지 않습니다. 특히 ML/DL 작업은 단순히 텍스트를 편집하는 일이 아니라 **실험 환경을 함께 다루는 일**에 가깝습니다. Zed로 며칠을 써보니, 몇몇 워크플로우에는 VSCode가 이미 갖춘 통합이 더 잘 맞았습니다.

## 3. Dev Container는 VSCode의 통합을 그대로 쓴다

요즘 제 프로젝트는 거의 전부 `.devcontainer`로 환경을 잡습니다. CUDA 버전, PyTorch 버전, 시스템 라이브러리까지 한데 묶어 두면 노트북을 바꾸거나 외부에 공개할 때 환경 차이로 시간을 쓰지 않아도 됩니다.

Zed도 `.devcontainer`를 인식하긴 합니다. 하지만 실제로 컨테이너 안에서 Zed를 띄우려 하면 사정이 다릅니다. 내부에서 `zed .`을 실행해도 결국 **호스트 쪽 Zed가 열립니다**. 그러다 보니 이런 차이가 생깁니다.

- 컨테이너 안에 설치된 Python, CUDA 툴체인이 에디터에 연결되지 않습니다.
- 파일 경로와 권한이 호스트 기준으로 잡힌다.
- [Language Server Protocol(LSP)](https://microsoft.github.io/language-server-protocol/), 디버거가 컨테이너 안의 인터프리터를 바로 찾지 못합니다.

VSCode는 **Dev Container, SSH, Codespaces**까지 한 묶음으로 통합합니다. Remote 확장만 켜면 에디터가 컨테이너 안에 들어가 있는 것처럼 동작하고 디버거와 터미널까지 모두 원격 환경을 바라봅니다. 그래서 컨테이너 기반 작업, 즉 **재현 가능한 실험 환경**이 필요할 때는 지금도 VSCode의 이 통합을 그대로 활용합니다.

## 4. Jupyter Notebook 작업은 VSCode에서 이어간다

강화학습 공부 노트, 실험 코드, 결과 시각화처럼 제 작업의 상당 부분은 Jupyter Notebook 위에서 돌아갑니다. 학습 곡선을 한 셀에서 그리고 모델 출력을 다음 셀에서 분석하는 흐름은 ML 작업에서 큰 비중을 차지합니다.

Zed는 현재 `.ipynb` 파일의 셀 단위 편집을 지원하지 않습니다. JSON 형태의 원본은 열 수 있지만 셀 단위 실행, 출력 확인, 그래프 인라인 표시 같은 노트북 고유의 경험은 아직 Zed의 기능 범위 밖입니다.

그래서 자연스럽게 역할을 나눠 씁니다.

- 일반 코드와 모듈은 Zed에서 편집합니다.
- 실험과 분석은 VSCode의 Jupyter 확장을 그대로 활용합니다.

두 에디터를 오가는 전환 비용은 있지만 각자 잘하는 영역에 맡긴다고 생각하면 크게 거슬리지 않습니다.

## 5. 기능 지형을 정리하면

지금까지 써본 인상을 정리하면 대략 이렇습니다.

| 항목 | VSCode | Zed |
|------|--------|-----|
| 실행 속도 | 보통 | 매우 빠름 |
| 확장 생태계 | 매우 강함 | 성장 중 |
| Dev Container | 완전 통합 | 호스트에서 열림 |
| Jupyter Notebook | 지원 | 미지원 |
| 실시간 협업 | 보통 | 강점 |

Zed는 속도와 협업이라는 뚜렷한 강점을 지닌 **에디터**입니다. VSCode는 확장과 원격 개발 기능이 합쳐진 **개발 플랫폼**에 가깝습니다. 서로 강점이 겹치지 않으니, 하나를 골라 다른 하나를 버리기보다 작업 성격에 따라 나눠 쓰는 쪽이 지금은 더 잘 맞습니다.

## 6. 지금 내 결론

컨테이너 기반 학습 코드, Jupyter 실험, 원격 GPU 머신 접속처럼 VSCode의 통합이 이미 잘 갖춰진 작업은 계속 VSCode 안에서 진행할 생각입니다. 재현 가능한 실험 환경을 유지할 때는 검증된 방식을 쓰는 편이 낫습니다.

반대로 잠깐 코드 한 조각을 열어보거나 마크다운 파일을 빠르게 다듬을 때는 Zed를 자주 꺼내게 됩니다. 실행 속도와 반응성만큼은 지금도 VSCode보다 확실히 낫습니다.

Zed에 Dev Container 통합과 Jupyter Notebook 지원이 더해지면 역할 분담의 경계도 달라질 것 같습니다. 지금은 두 에디터를 각자 강점에 맞게 병행하는 것이 가장 현실적인 선택입니다.
