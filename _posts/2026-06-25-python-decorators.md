---
layout: post
title: 파이썬 장식자, 제대로 이해하기
subtitle: 함수의 동작을 깔끔하게 확장하는 파이썬 장식자 기본기
description: 파이썬 장식자의 기본 구조, functools.wraps, 인자 처리, 캐싱, 클래스 기반 장식자, 장식자 중첩과 인자를 받는 장식자 패턴을 정리한다.
tags: [python, decorator, functools, programming, devtips]
cover-img: /assets/img/develop.jpeg
thumbnail-img: /assets/img/python.webp
share-img: /assets/img/develop.jpeg
author: 전경원
---

데코레이터(decorator, 이하 장식자)는 파이썬에서 비교적 최근에 도입된 문법이다. 요즘도 장식자를 다루지 않는 파이썬 문법 책을 어렵지 않게 찾을 수 있다. 하지만 실제로 파이썬을 쓰다 보면 장식자는 꽤 자주 마주치게 된다. 이번 기회에 파이썬 장식자가 무엇인지, 어떻게 만들어지고 활용되는지 정리해 본다.

## 함수도 값이다

파이썬에서 함수는 **일급 객체(first-class object)**다. `int`나 `str`처럼 변수에 담거나, 다른 함수의 인자로 넘기거나, 반환값으로 돌려줄 수 있다.

```python
def say_hello(name):
    return f"Hello {name}"

def greet(greeter_func):
    return greeter_func("경원")

greet(say_hello)  # 'Hello 경원'
```

여기서 중요한 점은 `say_hello`를 실행한 것이 아니라, 함수 자체를 값처럼 넘겼다는 것이다. 함수 이름 뒤에 괄호가 붙으면 호출이고, 괄호 없이 이름만 쓰면 그 함수 객체를 가리킨다.

함수 안에 함수를 정의할 수도 있다.

```python
def outer():
    def inner():
        print("inner 함수입니다")
    inner()
```

`inner`는 `outer` 안에서만 존재하고, 바깥에서는 접근할 수 없다. 그리고 함수는 다른 함수를 반환값으로 내보낼 수도 있다.

```python
def make_greeter(lang):
    def korean():
        return "안녕하세요"
    def english():
        return "Hello"
    
    return korean if lang == "ko" else english

greeter = make_greeter("ko")
greeter()  # '안녕하세요'
```

이 세 가지를 알면 장식자를 이해할 수 있다.

## 장식자의 기본 구조

이제 장식자를 직접 만들어보자. 어떤 함수가 호출될 때 전후로 메시지를 출력하는 간단한 장식자다.

```python
def my_decorator(func):
    def wrapper():
        print("함수 실행 전")
        func()
        print("함수 실행 후")
    return wrapper

def say_hello():
    print("안녕하세요!")

say_hello = my_decorator(say_hello)
say_hello()
```

```
함수 실행 전
안녕하세요!
함수 실행 후
```

`say_hello = my_decorator(say_hello)` 이 한 줄이 장식자의 핵심이다. `say_hello`라는 이름이 이제 `wrapper` 함수를 가리키게 되고, `wrapper` 안에는 원래 `say_hello`가 `func`라는 이름으로 살아있다.

### `@` 문법

매번 `say_hello = my_decorator(say_hello)` 식으로 쓰는 건 번거롭다. 파이썬은 이걸 `@` 기호로 줄여 쓸 수 있게 해준다.

```python
@my_decorator
def say_hello():
    print("안녕하세요!")
```

`@my_decorator`는 `say_hello = my_decorator(say_hello)`와 완전히 동일하다. 간결한 표기(syntactic sugar)이면서, 장식자가 적용됐다는 것도 한눈에 드러난다.

## 인자를 받는 함수에 장식자 적용하기

앞서 만든 `wrapper`는 인자를 받지 않는다. 그러니 인자가 있는 함수에 적용하면 에러가 난다.

```python
@my_decorator
def greet(name):
    print(f"안녕하세요, {name}님!")

greet("경원")  # TypeError!
```

해결책은 `*args`와 `**kwargs`를 활용하는 것이다. 이 두 가지를 쓰면 wrapper가 어떤 형태의 인자든 그대로 받아서 원래 함수에 전달한다.

```python
def my_decorator(func):
    def wrapper(*args, **kwargs):
        print("함수 실행 전")
        result = func(*args, **kwargs)
        print("함수 실행 후")
        return result
    return wrapper
```

`return result`도 빠뜨리면 안 된다. wrapper에서 return을 빠뜨리면 원래 함수의 반환값이 사라지고 `None`이 된다. 이 패턴을 기본으로 깔고 가면 거의 모든 함수에 장식자를 무난하게 적용할 수 있다.

## `functools.wraps`로 함수 메타정보 보존하기

장식자를 적용하면 한 가지 문제가 생긴다. 함수의 이름이나 docstring 같은 메타 정보가 wrapper로 덮여버린다.

```python
@my_decorator
def greet(name):
    """인사를 건네는 함수"""
    print(f"안녕하세요, {name}님!")

print(greet.__name__)  # 'wrapper'
help(greet)            # wrapper에 대한 도움말이 나옴
```

원래 함수처럼 보여야 할 `greet`가 사실은 `wrapper`라고 소개되는 셈이다. `functools.wraps`를 쓰면 이 문제를 막을 수 있다.

```python
import functools

def my_decorator(func):
    @functools.wraps(func)
    def wrapper(*args, **kwargs):
        print("함수 실행 전")
        result = func(*args, **kwargs)
        print("함수 실행 후")
        return result
    return wrapper
```

```python
print(greet.__name__)  # 'greet'
help(greet)            # 원래 docstring이 그대로 나옴
```

실무에서 장식자를 쓸 때는 `@functools.wraps(func)`를 빠뜨리지 않는 게 좋다. 특히 다른 사람이 함수를 `help()`로 조회하거나 디버거에서 추적할 때 그 차이가 두드러진다.

### 기본 틀

앞서 다룬 내용을 모으면 다음 패턴이 된다.

```python
import functools

def my_decorator(func):
    @functools.wraps(func)
    def wrapper(*args, **kwargs):
        # 함수 실행 전 처리
        result = func(*args, **kwargs)
        # 함수 실행 후 처리
        return result
    return wrapper
```

## 실전 예시

앞서 살펴본 구조를 실제 코드에 적용해보자.

### 실행 시간 측정

```python
import functools
import time

def timer(func):
    @functools.wraps(func)
    def wrapper(*args, **kwargs):
        start = time.perf_counter()
        result = func(*args, **kwargs)
        end = time.perf_counter()
        print(f"{func.__name__}() 실행 시간: {end - start:.4f}초")
        return result
    return wrapper

@timer
def heavy_computation(n):
    return sum(i**2 for i in range(n))

heavy_computation(1_000_000)
# heavy_computation() 실행 시간: 0.1823초
```

### 디버깅용 로그

함수가 어떤 인자로 호출됐고 무엇을 반환했는지 자동으로 출력해주는 장식자다. 복잡한 로직을 추적할 때 `print`를 여기저기 박는 것보다 훨씬 편하다.

```python
import functools

def debug(func):
    @functools.wraps(func)
    def wrapper(*args, **kwargs):
        args_repr = [repr(a) for a in args]
        kwargs_repr = [f"{k}={repr(v)}" for k, v in kwargs.items()]
        signature = ", ".join(args_repr + kwargs_repr)
        print(f"호출: {func.__name__}({signature})")
        result = func(*args, **kwargs)
        print(f"반환: {func.__name__}() → {repr(result)}")
        return result
    return wrapper

@debug
def add(x, y):
    return x + y

add(3, y=5)
# 호출: add(3, y=5)
# 반환: add() → 8
```

### 결과 캐싱

이미 계산한 결과를 저장해두고 같은 인자로 다시 호출될 때 재사용하는 패턴이다. 피보나치처럼 재귀 호출이 폭발하는 함수에 적용하면 효과가 극적이다.

```python
import functools

def cache(func):
    @functools.wraps(func)
    def wrapper(*args, **kwargs):
        key = args + tuple(kwargs.items())
        if key not in wrapper.cache:
            wrapper.cache[key] = func(*args, **kwargs)
        return wrapper.cache[key]
    wrapper.cache = {}
    return wrapper

@cache
def fibonacci(n):
    if n < 2:
        return n
    return fibonacci(n - 1) + fibonacci(n - 2)

fibonacci(30)  # 캐싱 없이는 수십만 번 호출될 것을 11번으로 해결
```

사실 파이썬 표준 라이브러리에는 이미 `functools.lru_cache`가 있다. 직접 구현할 필요 없이 `@functools.lru_cache` 또는 파이썬 3.9부터 추가된 `@functools.cache`를 바로 갖다 쓰면 된다.

```python
@functools.cache
def fibonacci(n):
    if n < 2:
        return n
    return fibonacci(n - 1) + fibonacci(n - 2)
```

## 클래스 기반 장식자

지금까지는 함수를 반환하는 방식으로 장식자를 만들었다. 같은 일을 클래스로도 할 수 있다. 핵심은 클래스의 인스턴스를 함수처럼 호출할 수 있게 만드는 `__call__` 메서드다.

```python
import functools

class CountCalls:
    def __init__(self, func):
        functools.update_wrapper(self, func)
        self.func = func
        self.count = 0

    def __call__(self, *args, **kwargs):
        self.count += 1
        print(f"{self.func.__name__} 호출 횟수: {self.count}")
        return self.func(*args, **kwargs)

@CountCalls
def say_hello(name):
    print(f"안녕하세요, {name}님!")

say_hello("경원")
say_hello("지민")
```

```text
say_hello 호출 횟수: 1
안녕하세요, 경원님!
say_hello 호출 횟수: 2
안녕하세요, 지민님!
```

`@CountCalls`는 `say_hello = CountCalls(say_hello)`와 같다. `__init__`이 함수를 받아 인스턴스에 저장하고, 이후 `say_hello("경원")`처럼 호출하면 `__call__`이 실행된다. 함수 기반 장식자가 wrapper라는 함수 객체를 반환하는 것과 달리, `__call__`은 함수를 직접 실행하고 그 결과를 반환한다. `__init__`이 호출 준비를 담당하고, `__call__`은 실제 호출이 일어나는 시점이기 때문이다.

클래스 기반 장식자가 빛을 발하는 경우는 상태를 유지해야 할 때다. 호출 횟수처럼 호출 간에 유지해야 하는 값을 인스턴스 변수로 자연스럽게 관리할 수 있다. 전후 처리만 추가하는 정도라면 함수 기반 장식자가 더 간결하다.

## 장식자에 인자 넘기기

`@timer` 대신 `@slow_down(rate=2)`처럼 장식자 자체에 인자를 넘기고 싶을 때가 있다. 이 경우에는 함수를 한 겹 더 감싸야 한다.

```python
import functools
import time

def slow_down(rate=1):
    def decorator(func):
        @functools.wraps(func)
        def wrapper(*args, **kwargs):
            time.sleep(rate)
            return func(*args, **kwargs)
        return wrapper
    return decorator

@slow_down(rate=2)
def fetch_data():
    print("데이터를 가져왔습니다")
```

구조가 세 단계가 됐다. `slow_down(rate=2)`를 호출하면 `decorator`가 반환되고, 그 `decorator`가 `fetch_data`를 받아서 `wrapper`로 감싼다. `@slow_down(rate=2)`라고 쓰는 순간 이 세 단계가 순서대로 실행된다.

같은 동작을 클래스로 구현할 수도 있다. `__init__`이 인자를 받고, `__call__`이 함수를 받아 wrapper를 반환하는 구조다. 앞서 본 `CountCalls`에서 `__call__`이 함수를 직접 실행했던 것과 달리, 여기서는 `__call__`이 wrapper를 반환한다는 점에 주목하자. `__init__`이 인자를 받는 역할을 맡으면서, `__call__`이 함수 기반의 `decorator`와 같은 자리를 차지하기 때문이다.

```python
import functools
import time

class SlowDown:
    def __init__(self, rate=1):
        self.rate = rate

    def __call__(self, func):
        @functools.wraps(func)
        def wrapper(*args, **kwargs):
            time.sleep(self.rate)
            return func(*args, **kwargs)
        return wrapper

@SlowDown(rate=2)
def fetch_data():
    print("데이터를 가져왔습니다")
```

## 장식자 쌓아 올리기

한 함수에 장식자를 여러 개 적용할 수 있다.

```python
@timer
@debug
def greet(name):
    print(f"안녕하세요, {name}님!")
```

적용 순서는 아래에서 위로 읽으면 된다. 먼저 `debug`가 `greet`를 감싸고, 그 결과를 다시 `timer`가 감싼다. `timer(debug(greet))`와 같다. 순서를 바꾸면 결과도 달라지니 주의해야 한다.

## 정리

장식자는 함수를 건드리지 않고 기능을 끼워 넣는 방법이다. 로깅, 타이밍, 인증, 캐싱처럼 여러 함수에 반복적으로 필요한 관심사를 한 곳에 모을 수 있어서, 코드가 하는 일 자체에 집중하기가 훨씬 편해진다.

실수 없이 쓰려면 몇 가지 습관을 들여두는 게 좋다. wrapper 안에는 `@functools.wraps(func)`를 항상 붙이고, 시그니처는 `*args, **kwargs`로 유연하게 잡고, 반환값은 반드시 돌려줘야 한다. 장식자에 인자가 필요하다면 함수를 한 겹 더 감싸면 된다. 

클래스 기반 장식자는 상태를 유지해야 할 때 유용하다. 호출 횟수처럼 호출 간에 유지해야 하는 값을 인스턴스 변수로 자연스럽게 관리할 수 있다. 전후 처리만 추가하는 정도라면 함수 기반 장식자가 더 간결하다.

## 참고 자료

- [DataCamp, Python Decorators Tutorial](https://www.datacamp.com/tutorial/decorators-python)
- [Real Python, Primer on Python Decorators](https://realpython.com/primer-on-python-decorators/)
- [YouTube, Python Decorators](https://www.youtube.com/watch?v=03r7sloAyOY)
