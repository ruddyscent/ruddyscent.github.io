---
layout: post
title: VSCode vs Zed — 빠르지만 아직은 대체재는 아니다
subtitle: Zed를 직접 써보면서 느낀 점 정리
tags: [vscode, zed, editor, developer-tools]
cover-img: /assets/img/develop.jpeg
thumbnail-img: /assets/img/vscode.png
share-img: /assets/img/develop.jpeg
author: 전경원
---
최근에 [달레줄레 팟캐스트](https://podcasts.apple.com/ca/podcast/%EB%8B%AC%EB%A0%88%EC%A4%84%EB%A0%88/id1632529597)를 듣다가 **[Zed](https://zed.dev/)** 라는 에디터를 알게 되었다.  
“Rust로 만든 빠른 에디터”라는 이야기에 흥미가 생겨서 직접 사용해보기 시작했다.
결론부터 말하면, Zed는 분명 인상적인 도구다.

## Zed의 첫인상 — 빠르다
Zed를 처음 실행하면 바로 느껴지는 건 속도다.
- 실행 속도 빠름
- 타이핑 반응 빠름
- 전체적으로 가벼운 느낌
이건 우연이 아니다.  
VSCode는 Electron으로 개발되었지만, Zed는 **Rust + GPU 렌더링 기반**으로 만들어졌다. VSCode를 오래 쓰다 보면 느껴지는 “무거움”이 Zed에는 거의 없다.

## 그렇다면 Zed는 VSCode를 대체할 수 있을까?
빠르다는 점은 분명 매력적이지만, 실제로 개발 환경을 대체할 수 있을지는 다른 문제다.  Zed를 쓰면서 느낀 가장 큰 문제는 **개발 환경 전체**다. 내가 실제로 사용하면서 막혔던 부분은 크게 두 가지다.

## 1. Dev Container 지원 부족
요즘 개발 환경에서 Dev Container는 거의 필수다.  Zed도 `.devcontainer`를 인식하긴 한다.  
하지만 실제로 써보면 중요한 문제가 있다.
- Dev Container 내부에서 Zed를 실행하면
- **호스트의 Zed가 실행됨**
- 결과적으로:
  - 컨테이너 내부 환경을 제대로 사용하지 못함
  - 파일 접근/편집이 꼬임
즉,
> “컨테이너 안에서 개발한다”는 워크플로우가 깨진다.
반면 VSCode는:
- Dev Container 완전 통합
- Docker / SSH / Codespaces까지 자연스럽게 연결
👉 이 차이는 생각보다 크다.

## 2. Jupyter Notebook 미지원
이건 내 경우에 치명적이다.
- RL 공부
- 실험 코드
- 노트 정리
👉 대부분 **Jupyter Notebook 기반**
하지만 Zed는:
> ❌ Notebook 편집 자체가 불가능
결국:
- 코드 → Zed
- 노트북 → VSCode
이렇게 나눠 써야 하는데, 이건 생산성을 떨어뜨린다.
---
## 기능 vs 속도
정리하면 이렇게 된다.
| 항목 | VSCode | Zed |
|------|--------|-----|
| 성능 | 보통 | 매우 빠름 |
| 확장성 | 매우 강함 | 제한적 |
| Dev Container | 완벽 | 부족 |
| Jupyter | 지원 | 없음 |
| 협업 | 보통 | 강함 |

Zed는 분명 잘 만든 도구지만,  지금은 “에디터”에 가까운 느낌이다.
VSCode는:
> 에디터 + 플랫폼 + 생태계

## 그래서 지금 내 결론
아마도 진행 중인 중요한 작업은 모두 VSCode 안에 떠 있을 것 같다. 작업 중에 다른 일로 코드나 마크다운 파일을 열어야 한다면, Zed를 이용하게 될 것같다. 

핮만, 앞으로 Dev Container과 Notebook이 지원되는 날이 온다면, 아마도 Zed가 빠르게 VSCode의 자리를 대체하지 않을까 예측해본다.