---
layout: post
title: 프로그래머스의 코딩테스트 C# 버전 확인
subtitle: C# 환경의 버전 확인
cover-img: /assets/img/develop.jpeg
thumbnail-img: /assets/img/csharp.png
share-img: /assets/img/develop.jpeg
tags: [csharp, pccp]
author: 전경원
---

[프로그래머스의 코딩테스트](https://school.programmers.co.kr/learn/challenges) 문제를 풀다보면 최신 버전의 C# 기능을 쓸 수 없다는 것을 알게된다.

예를 들어서, 대표적으로 [`PriorityQueue`](https://learn.microsoft.com/ko-kr/dotnet/api/system.collections.generic.priorityqueue-2)를 쓸 수 없다.

안그래도 시험 시간이 빠듯한데, 지원하지 않는 문법 때문에 구현을 바꾸는 수고를 하지 않으려면, 지원되는 기능과 지원하지 않는 기능을 미리 파악해 놓을 필요가 있다.

> **지원되는 기능과 지원되지 않는 기능을 미리 파악해 두는 것**

문제는, 프로그래머스의 코딩테스트 환경에서는 터미널 접근이 불가능하다는 점이다.  `dotnet --version` 같은 명령으로 확인할 수 없다.  그렇다면 직접 코드로 확인하는 수밖에 없다.

## 들어가기 전에
사실은 프로그래머스는 C# 컴파일러 버전을 공개하고 있습니다. 코딩테스트 연습 인터페이스의 오른쪽 위에 있는 **컴파일 옵션**을 클릭하면, 지원하는 언어의 컴파일러 버전과 컴파일 커맨드를 확인할 수 있습니다. C#의 컴파일러와 컴파일 커맨드는 아래와 같습니다.
- **컴파일러**: Mono C# Compiler 6.10.0
- **컴파일 커맨드**: mcs -out:FILENAME -r:System.dll

## 런타임 버전 확인하기

먼저 간단한 코드를 실행해서 런타임 버전을 확인해보자.
```csharp
using System;

public class Solution {
    public int[] solution(int target) {
        int[] answer = new int[] {};
        Console.WriteLine(Environment.Version);
        return answer;
    }`
}
```
이 코드를 실행해보면, 
```
...
출력 〉	4.0.30319.42000
```
런타임 버전 번호로 추정해 볼 수 있는 .NET 프레임워크의 버전은 4.6에서 4.8 정도이다. 이 버전의 .NET 프레임워크에는 일반적으로 C# 6에서 7.3을 지원하는 컴파일러가 설치된다.

운좋게도 C# 7.1에서 지원하는 [default literal](https://learn.microsoft.com/ko-kr/dotnet/csharp/language-reference/operators/default#default-literal)을 시험해보다가 컴파일러가 지원하는 C# 버전을 확인할 수 있었다.

```csharp
using System;

public class Solution {
    public int[] solution(int target) {
        int x = default;      // C# 7.1
        return new int[] { x + target };
    }
}
```

```
/Solution0.cs(5,17): error CS1644: Feature `default literal' cannot be used because it is not part of the C# 7.0 language specification
```

> 지원되는 C# 버전은 **[7.0](https://learn.microsoft.com/ko-kr/dotnet/csharp/whats-new/csharp-version-history#c-version-70)**

## Tuple + Pattern Matching 문제

지원하는 C# 버전은 확인했지만, 프로그래머스 같은 온라인 저지는 종종 컴파일러, 런타임, 참조 어셈블리 등의 구성요소가 섞여 있어서, 일부 기능만 깨진지는 경우가 있다. 따라서, 주요 문법 및 지원 라이브러리를 확인할 필요가 있다. 

예를 들어, C# 7.0에서 “처음” 들어온 기능 중, 온라인 코딩테스트에서 실전으로 유용한 기능을 확인하던 중에 튜플과 패턴 매칭을 동시에 사용하면 문제가 발생하는 것을 확인할 수 있다.

```csharp
using System;

public class Solution {
    public int[] solution(int target) {
        (int x, int y) t = (target, 1);

        if (t.x is int n) {
            return new int[] { n };
        }

        return new int[] { 0 };
    }
}
```
```
/Solution0.cs(7,17): warning CS0183: The given expression is always of the provided (`int') type
/Solution0.cs(7,17): error CS0584: Internal compiler error: constant in type pattern matching
/Solution0.cs(8,32): error CS0841: A local variable `n' cannot be used before it is declared
```

`t.x`는 이미 `int`로 확정된 값이다. 이 경우 `is int n`은 항상 참이 되며, 특정 컴파일러 경로에서 내부 오류가 발생하기 때문에 문제가 된 것이다. 회피 방법으로는 패턴 매칭을 포기하거나, [박싱](https://learn.microsoft.com/en-us/dotnet/csharp/programming-guide/types/boxing-and-unboxing)을 이용할 수 있다.

### 방법 1 --- 패턴 매칭 제거

``` csharp
return new int[] { t.x };
```

### 방법 2 --- [object](https://learn.microsoft.com/en-us/dotnet/csharp/language-reference/builtin-types/reference-types#the-object-type)로 감싸기

``` csharp
object o = t.x;
if (o is int n) {
    return new int[] { n };
}
```

## 프로그래머스 C# 환경 정리

프로그래머스 C# 환경 요약을 요약해보자.

### ✔ 지원

-   C# 7.0 문법
-   Tuple 문법 및 분해
-   out var
-   local function
-   pattern matching
-   LINQ
-   Dictionary, HashSet, List, Queue, Stack
-   StringBuilder
-   BigInteger

### ✘ 미지원

-   C# 7.1 이상
-   default literal
-   PriorityQueue

## 결론

프로그래머스 C# 코딩테스트 환경은

> **C# 7.0 + .NET Framework 4.6 - 4.8 기반 환경**

핵심은 다음 두 가지다.

1.  C# 7.1 이상 기능은 쓰지 않는다.
2.  PriorityQueue는 직접 구현하거나 대체한다.
