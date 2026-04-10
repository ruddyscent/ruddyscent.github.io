---
layout: post
title: "C#에서 2차원 배열: int[,] vs int[][]"
subtitle: 2차원 배열의 차이점과 사용법
tags: [csharp, array, 2d-array, jagged-array, multidimensional-array]
cover-img: /assets/img/develop.jpeg
thumbnail-img: /assets/img/csharp.png
share-img: /assets/img/develop.jpeg
author: 전경원
---

C#에서 2차원 배열을 다룰 때는 크게 두 가지 방법이 있다.

- `int[,]`: 다차원 배열 (Rectangular Array)
- `int[][]`: 가변 배열 (Jagged Array)

겉보기에는 비슷하지만, 내부 구조와 사용 방식에는 큰 차이가 있다.

## 1. 다차원 배열 (Rectangular Array) - `int[,]`

다차원 배열은 모든 행이 같은 열 수를 가지는 고정된 크기의 배열이다. 메모리 상에서 연속적으로 저장되며, 2차원 이상의 배열도 지원한다.

```csharp
int[,] matrix = new int[3, 4]; // 3행 4열
matrix[0, 0] = 1;
matrix[0, 1] = 2;
// ...
```

배열을 순회하려면 다음과 같은 이중 루프를 사용한다.

```csharp
for (int i = 0; i < matrix.GetLength(0); i++)
{
    for (int j = 0; j < matrix.GetLength(1); j++)
    {
        Console.Write(matrix[i, j] + " ");
    }
    Console.WriteLine();
}
```

특히 큰 데이터에서 반복 접근이 많을 경우, `int[,]`는 메모리 지역 (locality) 덕분에 CPU 캐시를 더 효율적으로 활용할 수 있어 성능면에서 유리하다.

> 참고: `int[,]`는 `Length`가 전체 요소 개수를 반환하고,  
각 차원의 길이는 `GetLength(dimension)`으로 가져와야 한다.

행렬 계산에 적합한 형태이며, 모든 행이 같은 열 개수를 가지는 경우에 사용하기 좋다.

## 2. 가변 배열 (Jagged Array) - `int[][]`

가변 배열은 배열의 배열로, 각 행이 다른 열 수를 가질 수 있다. 각 행은 별도의 배열로 존재하며, 메모리 상에서 연속적으로 저장되지 않는다.

```csharp
int[][] jaggedArray = new int[3][];

jaggedArray[0] = new int[2]; // 첫 번째 행은 2열
jaggedArray[1] = new int[3]; // 두 번째 행은 3열
jaggedArray[2] = new int[1]; // 세 번째 행은 1열
```

배열을 순회하려면 다음과 같은 이중 루프를 사용한다.

```csharp
for (int i = 0; i < jaggedArray.Length; i++)
{
    for (int j = 0; j < jaggedArray[i].Length; j++)
    {
        Console.Write(jaggedArray[i][j] + " ");
    }
    Console.WriteLine();
}
```

가변 배열은 유연성이 뛰어나지만, 메모리 비연속 구조로 인해  
캐시 효율이 떨어질 수 있고 성능이 다소 불리할 수 있다.

> 참고: `int[][]`는 일반 배열이므로 `Length`와 `arr[i].Length`를 사용한다.

그래프(Adjacency List), 트리, 가변 데이터 구조에 사용하기 좋다.

## 결론

- 단순 2D → `int[,]`
- 유동 구조 → `int[][]`
- 그래프 → `int[][]` 또는 `List[]`