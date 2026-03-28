---
layout: post
title: C#에서 Length, Count, Count()의 차이
subtitle: 배열, 컬렉션, LINQ에서 요소 개수를 구하는 올바른 방법과 성능 차이
description: C#의 Length, Count, Count() 차이를 정확히 정리합니다. 배열과 문자열, List와 Dictionary 같은 컬렉션, IEnumerable와 LINQ에서 어떤 방법을 써야 하는지와 O(1), O(n) 성능 차이를 설명합니다.
cover-img: /assets/img/develop.jpeg
thumbnail-img: /assets/img/csharp.png
share-img: /assets/img/develop.jpeg
tags: [csharp, linq, collection, performance, codingtest]
author: 전경원
---

# C#에서 Length, Count, Count()의 차이

C#에서 컬렉션의 크기를 구할 때 `Length`, `Count`, `Count()` 중 어떤 것을 사용해야 할지 헷갈리는 경우가 많습니다. 이 세 가지는 모두 "요소 개수"를 반환하지만, 적용 대상과 내부 동작, 성능이 서로 다릅니다.

정확히 구분하지 않으면 성능 문제로 이어질 수 있으므로, 각각의 차이를 명확히 이해할 필요가 있습니다.

## 1. Length
`Length`는 배열과 문자열에서 사용하는 **속성(property)**으로, 요소의 개수를 반환합니다.

```csharp
int[] numbers = { 1, 2, 3, 4, 5 };
Console.WriteLine(numbers.Length); // 출력: 5
```

내부적으로 길이를 이미 알고 있기 때문에 `Length`는 $O(1)$ 시간 복잡도를 보입니다.

## 2. Count
`Count`는 주로 `List<T>`, `HashSet<T>`, `Dictionary<TKey, TValue>` 등의 컬렉션에서 사용하는 **속성(property)**입니다. 

```csharp
List<int> numbers = new List<int> { 1, 2, 3, 4, 5 };
Console.WriteLine(numbers.Count); // 출력: 5
```

`Count`도 내부적으로 요소 수를 알고 있기 때문에 $O(1)$ 시간 복잡도를 보입니다.

## 3. Count()
`Count()`는 LINQ에서 제공하는, 컬렉션의 요소 수를 반환하는 **확장 메서드**입니다. `IEnumerable<T>` 인터페이스를 구현하는 모든 컬렉션에서 사용할 수 있습니다.

```csharp
List<int> numbers = new List<int> { 1, 2, 3, 4, 5 };
Console.WriteLine(numbers.Count()); // 출력: 5
```

`Count()`는 `ICollection<T>`에 대해서는 `Count` 속성을 사용하게 되지만, 그 외의 `IEnumerable<T>`에 대해서는 컬렉션을 순회하여 요소 수를 계산합니다.  이 경우 $O(n)$ 시간 복잡도를 보이므로, 가능하다면 `Count` 속성을 사용하는 것이 좋습니다.  특히, 반복문 안에서 `Count()` 쓰면 $O(n^2)$이 될 수 있으니 주의해야 합니다.
   
```csharp
List<int> numbers = new List<int> { 1, 2, 3, 4, 5 };
for (int i = 0; i < numbers.Count(); i++) // O(n^2)
{
    Console.WriteLine(numbers[i]);
}
```

## 결론
요약하자면, `Length`는 배열에서, `Count`는 컬렉션에서, 그리고 `Count()`는 LINQ 메서드로 사용합니다. 특히 `Count()`는 상황에 따라 $O(n)$이 될 수 있으므로, 습관적으로 사용하는 것은 피해야 합니다.

- 배열 / 문자열 → `Length`
- 컬렉션 → `Count`
- LINQ (`IEnumerable`) → 필요한 경우에만 `Count()`
