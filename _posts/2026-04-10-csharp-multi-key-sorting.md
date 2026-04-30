---
layout: post
title: C#에서 다중 정렬 (Multi-Level Sorting) 제대로 이해하기
subtitle: 특정 인덱스를 기준으로 문자열 정렬하는 방법
tags: [csharp, sorting, linq, algorithm, coding-test]
cover-img: /assets/img/develop.jpeg
thumbnail-img: /assets/img/csharp.webp
share-img: /assets/img/develop.jpeg
author: 전경원
---

코딩테스트에서 정렬 문제는 자주 나오지만, 조건이 하나 더 붙는 순간 당황하게 된다. 예를 들어, "특정 인덱스를 기준으로 정렬하고, 같으면 사전순으로 정렬하라" 같은 문제가 그렇다. C#에서 이런 **다중 정렬(Multi-Level Sorting)**을 어떻게 처리하는지 알아보자.

## 🧩 문제

문자열 배열 `words`와 정수 `k`가 주어진다.

다음 규칙에 따라 배열을 정렬하라:

1.  각 문자열의 `k`번째 문자를 기준으로 오름차순 정렬한다.
2.  만약 해당 문자가 같은 경우, 문자열 전체를 기준으로 사전순 정렬한다.

### 💡 핵심 포인트

정렬 기준이 하나가 아니다.

- 1차 기준: k번째 문자
- 2차 기준: 문자열 전체

즉, 하나의 기준으로 끝나는 정렬이 아니라 **다중 정렬 문제다.**

## 📌 예시

### 입력

    words = ["sun", "bed", "car"]
    k = 1

### 출력

    ["car", "bed", "sun"]

### 입력

    words = ["abce", "abcd", "cdx"]
    k = 2

### 출력

    ["abcd", "abce", "cdx"]

## ✅ 방법 1: Array.Sort + 비교 함수
정렬의 내부 동작을 직접 제어하는 방법이다. `Array.Sort` 메서드에 커스텀 비교 함수를 전달하여, 원하는 정렬 기준을 구현할 수 있다.

``` csharp
public string[] solution(string[] words, int k)
{
    Array.Sort(words, (a, b) =>
    {
        if (a[k] == b[k])
            return a.CompareTo(b);    // 2차 정렬

        return a[k].CompareTo(b[k]);  // 1차 정렬
    });

    return words;
}
```

이 비교 함수는 두 값을 비교해서 **누가 앞에 와야 하는지**를 결정한다.

- 먼저 k번째 문자를 비교하고
- 같으면 문자열 전체를 비교한다 (`CompareTo`)

즉, 조건 분기를 통해 **다중 정렬을 직접 구현한 것**이다.

### `CompareTo` 이해하기

``` csharp
a.CompareTo(b)
```

- 음수: `a`가 앞
- 0: 동일
- 양수: `b`가 앞

문자열에 적용하면 사전순 비교를 수행한다.

## ✅ 방법 2: LINQ OrderBy
위 방법보다 압도적인 가독성으로 실수를 줄일 수 있는 방법이다. LINQ의 `OrderBy`와 `ThenBy`를 활용하면, 정렬 기준을 명확하게 표현할 수 있다.

``` csharp
using System.Linq;

public string[] solution(string[] words, int k)
{
    return words
        .OrderBy(w => w[k])  // 1차 정렬
        .ThenBy(w => w)      // 2차 정렬
        .ToArray();
}
```

`OrderBy`로 첫 번째 기준으로 정렬을 수행하고 나서, 값이 같은 경우에 추가로 `ThenBy`로 두 번째 기준을 적용하는 방식으로 동작한다. 즉, LINQ에서는 정렬 기준을 “순서대로 선언”하는 방식으로 다중 정렬을 구현한다.

`ThenBy`를 여러 번 사용해서 3차, 4차 기준까지도 쉽게 추가할 수 있다.

``` csharp
var result = users
    .OrderBy(u => u.Age)
    .ThenBy(u => u.Name)
    .ThenBy(u => u.Score);
```

기본적으로는 올림차순 정렬이지만, 내림차순 정렬이 필요하다면, `OrderByDescending`과 `ThenByDescending`을 사용할 수 있다.

``` csharp
var result = users
    .OrderByDescending(u => u.Age)
    .ThenByDescending(u => u.Name);
```