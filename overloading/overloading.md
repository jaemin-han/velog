# 파이썬 연산자 오버로딩과 우선순위

## 목적
연산자 오버로드와 우선순위를 부여하는 방법을 나만의 방식으로 정리하자

---
## 특별 메소드 `__add__`
파이썬에서 클래스 간 연산자를 적용할 때, 각 클래스에 구현된 **특별 메소드**를 실행하는 방식으로 실행된다.  
**+** 의 경우 `__add__`, **\*** 의 경우는 `__mul__` 이런 식이다. 아래 예시를 보자.
```python
a = 2
b = 3

c = a + b

print(c)
# 5

# 특별 메소드 사용
d = a.__add__(b)

print(d)
# 5
```
## operator overloading
파이썬을 사용하다가 단순히 *int*, *float* 같은 기본 클래스가 아닌 홈메이드 클래스를 사용하는 상황이라고 가정하자.  

```python
class A:
    def __init__(self, value) -> None:
        self.value = value

a1 = A(2)
a2 = A(3)

a3 = a1 + a2
print(a3.value)
# 5가 출력되면 좋겠지만, 아래와 같은 에러가 발생한다.
# TypeError: unsupported operand type(s) for +: 'A' and 'A'
```

A는 int, float 같은 클래스들과 달리 `__add__` 메소드가 구현되어 있지 않다.  
문제를 해결하기 위해 간단하게 구현해 보자
```py
class A:
    def __init__(self, value) -> None:
        self.value = value

    def __add__(self, other):
        result = self.value + other.value
        return A(result)

a1 = A(2)
a2 = A(3)

a3 = a1 + a2
print(a3.value)
# 5
```
`__add__`가 A 클래스를 리턴하게 만들어, A 끼리의 합이 A 클래스가 되게 하였다.
이렇게 **+** 라는 연산자가 A라는 클래스에도 적용될 수 있게 특별 메소드를 구현하는 행위를 **연산자 오버로드**(operator overload)라 한다.

그런데 이렇게 같은 클래스들 뿐만이 아니라, 서로 다른 클래스 간에도 + 연산이 가능하면 좋지 않을까? 다음 예제를 보자
```py
import numpy as np

a_int = 1
b_float = 2.0
c_ndarray = np.array(3)

int_plus_float = a_int + b_float
print(int_plus_float, type(int_plus_float))
# 3.0 <class 'float'>

float_plus_ndarray = b_float + c_ndarray
print(float_plus_ndarray, type(float_plus_ndarray))
# 5.0 <class 'numpy.float64'>
```

`int` + `float` 는 `float`가 되고, `float` + `np.ndarray` 는 `numpy.float64`가 되었다. 우리의 클래스도 비슷한 행동을 할 수 있게 수정해보자

```py
class A:
    def __init__(self, value) -> None:
        self.value = value

    def __add__(self, other):
        other = as_A(other) # 추가된 코드
        result = self.value + other.value
        return A(result)
    
def as_A(x):
    # A와 다른 클래스의 경우  A클래스에 값을 담아줌
    if not isinstance(x, A):
        return A(x)
    return x

a1 = A(2)
a2 = 3

a3 = a1 + a2
print(a3.value, type(a3))
# 5 <class '__main__.A'>
```
`__add__`에서 *self* 는 + 왼쪽의 값, *other* 은 오른쪽 값을 의미한다.  

## 특별 메소드 `__radd__`
하지만 + 의 순서가 변경된다면 어떨까?. 
```py
a3 = a2 + a1
print(a3.value, type(a3))
# TypeError: unsupported operand type(s) for +: 'int' and 'A'
```

이처럼 A 클래스가 **+** 의 오른쪽에 위치할 경우, 에러가 발생한다. 이유는 파이썬 오퍼레이터의 작동 방식에 있다.

파이썬에서 **+** 연산자는 우선 **+** 왼쪽의 클래스의 `__add__`를 호출하고, 만약 없다면 오른쪽 클래스의 `__radd__` 를 호출한다. `__radd__` 는 `__add__` 와 유사하지만, `self`, `other` 의 순서가 바뀌었다고 생각하면 편하다. 다음과 같이 구현할 수 있다.
```py
class A:
    def __init__(self, value) -> None:
        self.value = value

    def __add__(self, other):
        other = as_A(other)
        result = self.value + other.value
        return A(result)
    
    def __radd__(self, other):
        other = as_A(other)
        result = self.value + other.value
        return A(result)
    
a1 = A(2)
a2 = 3

a3 = a2 + a1
print(a3.value, type(a3))
# 5 <class '__main__.A'>
```

## 우선순위 부여
여기서 또 하나의 궁금한 점이 생긴다. 서로 다른 두 클래스를 더할 수 있게 하는건 알겠다. 그럼 두 클래스 중 어떤 클래스를 살아남게, 즉 **우선순위**를 부여할 것인가?

가장 직관적인 방식으로 구현해 보자. 각 클래스가 클래스 속성으로 비교할 수 있는 값을 가지고, 그 값이 큰 클래스로 통합되는 식이다. 다음 코드를 보자.
```py
def as_A(x):
    if not isinstance(x, A):
        try:
            x = x.value
            return A(x)
        except AttributeError:
            return A(x)
    return x

def as_B(x):
    if not isinstance(x, B):
        try:
            x = x.value
            return B(x)
        except AttributeError:
            return B(x)
    return x

def is_bigger(a, b):
    return True if a.__array_priority__ >= b.__array_priority__ else False

class A:
    __array_priority__ = 2
    def __init__(self, value) -> None:
        self.value = value

    def __add__(self, other):
        if is_bigger(self, other):
            other = as_A(other)
            result = self.value + other.value
        else:
            return other.__radd__(self)
        return A(result)
    
    def __radd__(self, other):
        other = as_A(other)
        result = self.value + other.value
        return A(result)
    
class B:
    __array_priority__ = 3
    def __init__(self, value) -> None:
        self.value = value

    def __add__(self, other):
        if is_bigger(self, other):
            other = as_B(other)
            result = self.value + other.value
        else:
            return other.__radd__(self)
        return B(result)
    
    def __radd__(self, other):
        other = as_B(other)
        result = self.value + other.value
        return B(result)
    
a1 = A(2)
b1 = B(3)
c = a1 + b1
d = b1 + a1
print(c.value, type(c))
# 5 <class '__main__.B'>
print(d.value, type(d))
# 5 <class '__main__.B'>
``` 
`as_A` A, B 클래스 모두 value 속성으로 값을 처리하는 것을 염두에 두고 수정되었다. 만약 A 클래스로 만들 객체가 value 속성을 가진다면 그 속성을 사용하라는 의미이다.

`is_bigger` 두 객체의 *\_\_array_priority\_\_* 속성을 비교하여 참/거짓을 반환하는 함수이다

각 클래스에서는 비교하는 기준이 되는 속성 *\_\_array_priority\_\_* 을 정의하였다. 이후 `__add__`가 호출되는 과정에서 `is_bigger` 결과가 참이면 그대로 진행하고, 거짓이면 *other* 객체의 `__radd__` 를 호출하였다.

그 결과 A, B 클래스를 어떻게 더하든 모두 B 클래스가 되는 모습을 확인할 수 있다.

## numpy

우선순위를 구하기 위해 *\_\_array_priority\_\_* 클래스 속성을 활용하였다. numpy 클래스에도 비슷한 값이 존재한다.
```py
a = np.array(3)
print(type(a), a.__array_priority__)
# <class 'numpy.ndarray'> 0.0
```

**+** 의 왼쪽에 ndarray, 오른쪽에 type A가 있는 상황을 가정하자.

```py
def as_A(x):
    if not isinstance(x, A):
        try:
            x = x.value
            return A(x)
        except AttributeError:
            return A(x)
    return x

class A:
    def __init__(self, value) -> None:
        self.value = value

    def __add__(self, other):
        print('A add excuted')
        other = as_A(other)
        result = self.value + other.value
        return A(result)
    
a1 = np.array(3)
a2 = A(2)

a3 = a1 + a2
print(a3.value, type(a3))
# TypeError: unsupported operand type(s) for +: 'int' and 'A'

```
a1 의 `__add__`, a2 의 `__radd__` 모두 작동하지 않기 때문에 에러가 발생하는 모습이다. 그러면 클래스에 *\_\_array_priority\_\_* 속성을 추가해 주면 어떻게 될까?

```py
A.__array_priority__ = 10
a1 = np.array(3)
a2 = A(2)

a4 = a1 + a2
print(a3.value, type(a3))
# TypeError: Concatenation operation is not implemented for NumPy arrays, use np.concatenate() instead. Please do not rely on this error; it may not be given on all Python implementations.
```
에러의 내용이 달라졌다. concatnation 이야기가 나오는 걸 보니 python list 끼리의 **+** 연산을 생각하지 말라는 뜻인 것 같다.
```py
a = [1,2,3]
b = [4,5,6]
c = a+b
print(c)
# [1, 2, 3, 4, 5, 6]
```

즉 array_priority 속성이 추가됨에 따라 numpy array 끼리의 연산이라고 생각하여 우측 A 의 `__radd__` 함수를 우선적으로 호출하려고 했지만, 존재하지 않아 에러를 발생하는 것으로 이해할 수 있다.

# 결론
- 파이썬의 특별 메소드 중 연산자와 관련된 `__add__` 같은 메소드를 직접 구현하여 클래스 간 상호작용을 직관적으로 구현할 수 있다.
- 두 값이 필요한 특별 메소드는 앞에 `r`을 붙이면 뒤쪽 클래스의 특별 메소드를 사용하라는 의미이다.
- `numpy` 클래스의 *\_\_array_priority\_\_* 속성을 사용하여 클래스 간 특별 메소드의 우위를 결정할 수 있다.