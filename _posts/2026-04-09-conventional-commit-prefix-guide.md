---
layout: post
title: 커밋 접두사 규칙 정리
subtitle: 커밋 메시지를 읽기 쉽게 만드는 가장 간단한 규칙
tags: [git, commit, conventional-commit-prefix, convention, collaboration, devtips]
cover-img: /assets/img/develop.jpeg
thumbnail-img: /assets/img/git.webp
share-img: /assets/img/develop.jpeg
author: 전경원
---

커밋 메시지는 팀마다, 사람마다 스타일이 다르다.  문제는 이게 쌓이면 히스토리를 읽는 데 시간이 너무 많이 든다는 점이다.  이걸 가장 간단하게 해결하는 방법이 있다.  커밋 메시지 앞에 **접두사(prefix)**를 붙이는 것이다.  예를 들어 다음과 같이 작성할 수 있다.

```text
fix: resolve emoji rendering issue in chat window
```

이 한 줄만 봐도 **버그 수정**이라는 맥락을 바로 이해할 수 있다. 커밋 메시지에 일관된 접두사를 사용하면, 히스토리를 빠르게 훑어보면서 어떤 커밋이 어떤 종류의 변경을 포함하는지 쉽게 파악할 수 있다.  또한, 팀원 간의 커뮤니케이션도 원활해지고, 코드 리뷰나 릴리즈 노트 작성 시에도 큰 도움이 된다.  자주 사용되는 접두사를 알아보자.

## 커밋 접두사 규칙 (Conventional Commit Prefixes)

### 가장 자주 사용하는 접두사
- `feat`: feature, 새로운 기능 추가  
- `fix`: 버그 수정  
- `refactor`: 동작 변경 없이 코드 구조 개선  
- `docs`: documentation, 문서 수정  
- `test`: 테스트 추가 또는 수정  
- `chore`: 기타 유지보수 작업  

### 추가로 알아두면 좋은 접두사
- `style`: 코드 포맷 변경 (로직 변경 없음)  
- `perf`: performance, 성능 개선  
- `build`: 빌드 시스템 또는 의존성 변경  
- `ci`: continuous integration, CI/CD 설정 변경  
- `revert`: 이전 커밋 되돌리기  

### 사용 예시
```text
feat: add churn prediction pipeline
fix: handle missing values in dataset
refactor: simplify feature engineering logic
perf: reduce training time with randomized search
docs: update README with usage instructions
test: add unit tests for encoder_helper
```