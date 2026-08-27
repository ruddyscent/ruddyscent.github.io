---
layout: post
title: C#에서 Length, Count, Count()의 차이
subtitle: 배열, 컬렉션, LINQ에서 요소 개수를 구하는 올바른 방법과 성능 차이
description: C#의 Length, Count, Count() 차이를 정확히 정리한다. 배열과 문자열, List와 Dictionary 같은 컬렉션, IEnumerable와 LINQ에서 어떤 방법을 써야 하는지와 O(1), O(n) 성능 차이를 설명한다.
cover-img: /assets/img/develop.jpeg
thumbnail-img: /assets/img/csharp.webp
share-img: /assets/img/develop.jpeg
tags: [csharp, linq, collection, performance, codingtest]
author: 전경원
---

C#에서 컬렉션의 크기를 구할 때 `Length`, `Count`, `Count()` 가운데 무엇을 써야 할지 헷갈릴 때가 많습니다. 세 방법은 모두 "요소 개수"를 반환하지만 적용 대상과 내부 동작, 성능은 다릅니다.

정확히 구분하지 않으면 성능 문제가 생길 수 있으므로 차이를 알아 둘 필요가 있습니다.

## 1. Length
`Length`는 배열과 문자열에서 요소 수를 반환하는 **속성(property)**입니다.

```csharp
int[] numbers = { 1, 2, 3, 4, 5 };
Console.WriteLine(numbers.Length); // 출력: 5
```

내부에서 길이를 이미 알고 있으므로 `Length`의 시간 복잡도는 $O(1)$입니다.

## 2. Count
`Count`는 주로 `List<T>`, `HashSet<T>`, `Dictionary<TKey, TValue>` 같은 컬렉션에서 쓰는 **속성**입니다.

```csharp
List<int> numbers = new List<int> { 1, 2, 3, 4, 5 };
Console.WriteLine(numbers.Count); // 출력: 5
```

`Count`도 내부에서 요소 수를 알고 있으므로 시간 복잡도는 $O(1)$입니다.

## 3. Count()
`Count()`는 컬렉션의 요소 수를 반환하는 LINQ **확장 메서드**입니다. `IEnumerable<T>` 인터페이스를 구현한 모든 컬렉션에서 쓸 수 있습니다.

```csharp
List<int> numbers = new List<int> { 1, 2, 3, 4, 5 };
Console.WriteLine(numbers.Count()); // 출력: 5
```

`Count()`는 `ICollection<T>`에서는 `Count` 속성을 쓰지만, 그 밖의 `IEnumerable<T>`에서는 컬렉션을 순회해 요소 수를 계산합니다. 이 경우 시간 복잡도는 $O(n)$이므로 가능하면 `Count` 속성을 쓰는 편이 좋습니다. 특히 반복문 안에서 `Count()`를 쓰면 $O(n^2)$이 될 수 있으니 주의해야 합니다.
   
```csharp
List<int> numbers = new List<int> { 1, 2, 3, 4, 5 };
for (int i = 0; i < numbers.Count(); i++) // O(n^2)
{
    Console.WriteLine(numbers[i]);
}
```

## 결론
`Length`는 배열에서, `Count`는 컬렉션에서, `Count()`는 LINQ 메서드로 사용합니다. 특히 `Count()`는 상황에 따라 $O(n)$이 될 수 있으므로 습관적으로 쓰지 않는 편이 좋습니다.

- 배열 / 문자열 → `Length`
- 컬렉션 → `Count`
- LINQ (`IEnumerable`) → 필요한 경우에만 `Count()`
