# CHAPTER 3 Numbers, Dates, and Times

Python에서 정수와 부동소수점 수로 수학 계산을 수행하는 것은 쉽다. 하지만 분수, 배열, 날짜 및 시간으로 계산을 수행해야 한다면 약간 더 많은 작업이 필요하다. 이 장은 이러한 주제에 초점을 맞춘다.

## 3.1. Rounding Numerical Values

### 문제

부동소수점 수를 고정된 소수점 자리수로 반올림하고 싶다.

### 해법

간단한 반올림에는 내장 `round(value, ndigits)` 함수를 사용한다:

```python
>>> round(1.23, 1)
1.2
>>> round(1.27, 1)
1.3
>>> round(-1.27, 1)
-1.3
>>> round(1.25361, 3)
1.254
```

값이 두 선택지의 정확히 중간에 있을 때, `round`의 동작은 가장 가까운 짝수로 반올림하는 것이다. 즉, 1.5나 2.5 같은 값은 모두 2로 반올림된다.

`round()`에 주어지는 자릿수는 음수일 수 있으며, 이 경우 십, 백, 천 단위 등으로 반올림이 발생한다:

```python
>>> a = 1627731
>>> round(a, -1)
1627730
>>> round(a, -2)
1627700
>>> round(a, -3)
1628000
```

### 논의

반올림과 출력을 위한 값 포맷팅을 혼동하지 마라. 단순히 특정 소수점 자리수로 숫자를 출력하는 것이 목표라면, 일반적으로 `round()`를 사용할 필요가 없다. 대신 포맷할 때 원하는 정밀도를 지정하면 된다:

```python
>>> x = 1.23456
>>> format(x, '0.2f')
'1.23'
>>> format(x, '0.3f')
'1.235'
>>> 'value is {:0.3f}'.format(x)
'value is 1.235'
```

인지된 정확도 문제를 "수정"하기 위해 부동소수점 수를 반올림하려는 충동을 억제하라. 부동소수점과 관련된 대부분의 애플리케이션에서는 이것이 필요하지도 않고 권장되지도 않는다. 이러한 오류를 피하는 것이 중요하다면(
예: 금융 애플리케이션), `decimal` 모듈 사용을 고려하라.

## 3.2. Performing Accurate Decimal Calculations

### 문제

십진수로 정확한 계산을 수행해야 하며, 부동소수점 수에서 자연스럽게 발생하는 작은 오류를 원하지 않는다.

### 해법

부동소수점 수의 잘 알려진 문제는 모든 base-10 십진수를 정확하게 표현할 수 없다는 것이다:

```python
>>> a = 4.2
>>> b = 2.1
>>> a + b
6.300000000000001
>>> (a + b) == 6.3
False
```

이러한 오류는 underlying CPU와 부동소수점 유닛이 수행하는 IEEE 754 산술의 "특징"이다.

더 많은 정확도를 원한다면(그리고 약간의 성능을 포기할 용의가 있다면) `decimal` 모듈을 사용할 수 있다:

```python
>>> from decimal import Decimal
>>> a = Decimal('4.2')
>>> b = Decimal('2.1')
>>> a + b
Decimal('6.3')
>>> print(a + b)
6.3
>>> (a + b) == Decimal('6.3')
True
```

`Decimal`의 주요 특징은 자릿수와 반올림을 포함한 계산의 다양한 측면을 제어할 수 있다는 것이다. 이를 위해 local context를 생성하고 설정을 변경한다:

```python
>>> from decimal import localcontext
>>> a = Decimal('1.3')
>>> b = Decimal('1.7')
>>> print(a / b)
0.7647058823529411764705882353
>>> with localcontext() as ctx:
...     ctx.prec = 3
...     print(a / b)
0.765
>>> with localcontext() as ctx:
...     ctx.prec = 50
...     print(a / b)
0.76470588235294117647058823529411764705882352941176
```

### 논의

`decimal` 모듈은 IBM의 "General Decimal Arithmetic Specification"을 구현한다.

Python 초보자는 `float` 데이터 타입의 인지된 정확도 문제를 해결하기 위해 `decimal` 모듈을 사용하려 할 수 있다. 하지만 애플리케이션 도메인을 이해하는 것이 정말 중요하다. 과학이나 공학 문제,
컴퓨터 그래픽스 또는 대부분의 과학적 성격의 것들로 작업하는 경우, 일반적인 부동소수점 타입을 사용하는 것이 더 일반적이다.

`decimal` 모듈의 주요 용도는 금융과 같은 것들과 관련된 프로그램이다. 이러한 프로그램에서는 작은 오류가 계산에 침투하는 것이 극도로 성가시다.

## 3.3. Formatting Numbers for Output

### 문제

출력을 위해 숫자를 포맷하고, 자릿수, 정렬, 천 단위 구분자 포함 등을 제어해야 한다.

### 해법

단일 숫자를 출력용으로 포맷하려면 내장 `format()` 함수를 사용한다:

```python
>>> x = 1234.56789

>>> format(x, '0.2f')  # 소수점 두 자리 정확도
'1234.57'

>>> format(x, '>10.1f')  # 10자 우측 정렬, 한 자리 정확도
'    1234.6'

>>> format(x, '<10.1f')  # 좌측 정렬
'1234.6    '

>>> format(x, '^10.1f')  # 중앙 정렬
'  1234.6  '

>>> format(x, ',')  # 천 단위 구분자 포함
'1,234.56789'

>>> format(x, '0,.1f')
'1,234.6'
```

지수 표기법을 사용하려면 `f`를 `e` 또는 `E`로 변경한다:

```python
>>> format(x, 'e')
'1.234568e+03'
>>> format(x, '0.2E')
'1.23E+03'
```

일반적인 형식은 `'[<>^]?width[,]?(.digits)?'`이다. 동일한 format code가 문자열의 `.format()` 메서드에서도 사용된다:

```python
>>> 'The value is {:0,.2f}'.format(x)
'The value is 1,234.57'
```

### 논의

천 단위 구분자를 사용한 포맷은 locale을 인식하지 않는다. 이를 고려해야 한다면 `locale` 모듈의 함수를 조사하라. 문자열의 `translate()` 메서드를 사용하여 구분자 문자를 교환할 수도 있다:

```python
>>> swap_separators = { ord('.'):',', ord(','):'.' }
>>> format(x, ',').translate(swap_separators)
'1.234,56789'
```

## 3.4. Working with Binary, Octal, and Hexadecimal Integers

### 문제

이진, 팔진, 십육진 숫자로 표현된 정수를 변환하거나 출력해야 한다.

### 해법

정수를 이진, 팔진 또는 십육진 텍스트 문자열로 변환하려면 각각 `bin()`, `oct()`, `hex()` 함수를 사용한다:

```python
>>> x = 1234
>>> bin(x)
'0b10011010010'
>>> oct(x)
'0o2322'
>>> hex(x)
'0x4d2'
```

`0b`, `0o`, `0x` 접두사가 나타나지 않도록 하려면 `format()` 함수를 사용할 수 있다:

```python
>>> format(x, 'b')
'10011010010'
>>> format(x, 'o')
'2322'
>>> format(x, 'x')
'4d2'
```

정수는 signed이므로 음수로 작업하는 경우 출력에도 부호가 포함된다:

```python
>>> x = -1234
>>> format(x, 'b')
'-10011010010'
>>> format(x, 'x')
'-4d2'
```

Unsigned 값을 생성해야 한다면 비트 길이를 설정하기 위해 최댓값을 더해야 한다. 예를 들어, 32비트 값을 표시하려면:

```python
>>> x = -1234
>>> format(2**32 + x, 'b')
'11111111111111111111101100101110'
>>> format(2**32 + x, 'x')
'fffffb2e'
```

다른 base의 정수 문자열을 변환하려면 적절한 base와 함께 `int()` 함수를 사용한다:

```python
>>> int('4d2', 16)
1234
>>> int('10011010010', 2)
1234
```

### 논의

팔진수 값을 지정하는 Python 구문은 다른 많은 언어와 약간 다르다. 팔진수 값 앞에 `0o`를 붙여야 한다:

```python
>>> import os
>>> os.chmod('script.py', 0o755)
```

## 3.5. Packing and Unpacking Large Integers from Bytes

### 문제

Byte string이 있고 이를 정수 값으로 unpack해야 한다. 또는 큰 정수를 byte string으로 다시 변환해야 한다.

### 해법

128비트 정수 값을 보유하는 16요소 byte string으로 작업해야 한다고 가정하자:

```python
data = b'\x00\x124V\x00x\x90\xab\x00\xcd\xef\x01\x00#\x004'
```

Byte를 정수로 해석하려면 `int.from_bytes()`를 사용하고 byte ordering을 지정한다:

```python
>>> len(data)
16
>>> int.from_bytes(data, 'little')
69120565665751139577663547927094891008
>>> int.from_bytes(data, 'big')
94522842520747284487117727783387188
```

큰 정수 값을 byte string으로 다시 변환하려면 `int.to_bytes()` 메서드를 사용하고 바이트 수와 byte order를 지정한다:

```python
>>> x = 94522842520747284487117727783387188
>>> x.to_bytes(16, 'big')
b'\x00\x124V\x00x\x90\xab\x00\xcd\xef\x01\x00#\x004'
>>> x.to_bytes(16, 'little')
b'4\x00#\x00\x01\xef\xcd\x00\xab\x90x\x00V4\x12\x00'
```

### 논의

큰 정수 값을 byte string으로/에서 변환하는 것은 일반적인 작업은 아니다. 하지만 암호화나 네트워킹과 같은 특정 애플리케이션 도메인에서 때때로 발생한다.

Byte order 지정(little 또는 big)은 단순히 정수 값을 구성하는 바이트가 least significant에서 most significant로 나열되는지 아니면 그 반대인지를 나타낸다:

```python
>>> x = 0x01020304
>>> x.to_bytes(4, 'big')
b'\x01\x02\x03\x04'
>>> x.to_bytes(4, 'little')
b'\x04\x03\x02\x01'
```

필요한 경우 `int.bit_length()` 메서드를 사용하여 값을 저장하는 데 필요한 비트 수를 결정할 수 있다:

```python
>>> x = 523 ** 23
>>> x.bit_length()
208
>>> nbytes, rem = divmod(x.bit_length(), 8)
>>> if rem:
...     nbytes += 1
>>> x.to_bytes(nbytes, 'little')
b'\x03X\xf1\x82iT\x96\xac\xc7c\x16\xf3\xb9\xcf...\xd0'
```

## 3.6. Performing Complex-Valued Math

### 문제

복소수를 사용하여 계산을 수행해야 한다.

### 해법

복소수는 `complex(real, imag)` 함수 또는 `j` 접미사가 있는 부동소수점 수를 사용하여 지정할 수 있다:

```python
>>> a = complex(2, 4)
>>> b = 3 - 5j
>>> a
(2+4j)
>>> b
(3-5j)
```

Real, imaginary, conjugate 값을 쉽게 얻을 수 있다:

```python
>>> a.real
2.0
>>> a.imag
4.0
>>> a.conjugate()
(2-4j)
```

모든 일반적인 수학 연산자가 작동한다:

```python
>>> a + b
(5-1j)
>>> a * b
(26+2j)
>>> a / b
(-0.4117647058823529+0.6470588235294118j)
>>> abs(a)
4.47213595499958
```

사인, 코사인, 제곱근 같은 추가 복소수 함수를 수행하려면 `cmath` 모듈을 사용한다:

```python
>>> import cmath
>>> cmath.sin(a)
(24.83130584894638-11.356612711218174j)
>>> cmath.cos(a)
(-11.36423470640106-24.814651485634187j)
>>> cmath.exp(a)
(-4.829809383269385-5.5920560936409816j)
```

### 논의

Python의 대부분의 수학 관련 모듈은 복소수 값을 인식한다. 예를 들어, `numpy`를 사용하면 복소수 값의 배열을 만들고 연산을 수행하기 쉽다:

```python
>>> import numpy as np
>>> a = np.array([2+3j, 4+5j, 6-7j, 8+9j])
>>> a + 2
array([ 4.+3.j,  6.+5.j,  8.-7.j, 10.+9.j])
>>> np.sin(a)
array([  9.15449915 -4.16890696j, -56.16227422-48.50245524j, ...])
```

Python의 표준 수학 함수는 기본적으로 복소수 값을 생성하지 않는다. 복소수를 결과로 생성하려면 명시적으로 `cmath`를 사용해야 한다:

```python
>>> import math
>>> math.sqrt(-1)
ValueError: math domain error
>>> import cmath
>>> cmath.sqrt(-1)
1j
```

## 3.7. Working with Infinity and NaNs

### 문제

무한대, 음의 무한대 또는 NaN(not a number)의 부동소수점 값을 생성하거나 테스트해야 한다.

### 해법

Python은 이러한 특수 부동소수점 값을 나타내는 특별한 구문이 없지만 `float()`를 사용하여 생성할 수 있다:

```python
>>> a = float('inf')
>>> b = float('-inf')
>>> c = float('nan')
>>> a
inf
>>> b
-inf
>>> c
nan
```

이러한 값의 존재를 테스트하려면 `math.isinf()`와 `math.isnan()` 함수를 사용한다:

```python
>>> math.isinf(a)
True
>>> math.isnan(c)
True
```

### 논의

무한 값은 수학적 방식으로 계산에서 전파된다:

```python
>>> a = float('inf')
>>> a + 45
inf
>>> a * 10
inf
>>> 10 / a
0.0
```

그러나 특정 연산은 정의되지 않으며 NaN 결과를 생성한다:

```python
>>> a = float('inf')
>>> a/a
nan
>>> b = float('-inf')
>>> a + b
nan
```

NaN 값은 예외를 발생시키지 않고 모든 연산을 통해 전파된다:

```python
>>> c = float('nan')
>>> c + 23
nan
>>> c / 2
nan
>>> math.sqrt(c)
nan
```

NaN 값의 미묘한 특징은 절대 같다고 비교되지 않는다는 것이다:

```python
>>> c = float('nan')
>>> d = float('nan')
>>> c == d
False
>>> c is d
False
```

이 때문에 NaN 값을 테스트하는 유일하게 안전한 방법은 `math.isnan()`을 사용하는 것이다.

## 3.8. Calculating with Fractions

### 문제

분수와 관련된 계산을 수행해야 한다.

### 해법

`fractions` 모듈을 사용하여 분수와 관련된 수학 계산을 수행할 수 있다:

```python
>>> from fractions import Fraction
>>> a = Fraction(5, 4)
>>> b = Fraction(7, 16)
>>> print(a + b)
27/16
>>> print(a * b)
35/64

>>> c = a * b
>>> c.numerator  # 분자
35
>>> c.denominator  # 분모
64

>>> float(c)  # float으로 변환
0.546875

>>> print(c.limit_denominator(8))  # 분모 제한
4/7

>>> x = 3.75  # float을 분수로 변환
>>> y = Fraction(*x.as_integer_ratio())
>>> y
Fraction(15, 4)
```

### 논의

분수로 계산하는 것은 대부분의 프로그램에서 자주 발생하지 않지만, 사용하는 것이 합리적인 상황이 있다. 예를 들어, 프로그램이 분수 단위의 측정을 받아들이고 그 형태로 계산을 수행하도록 허용하면 사용자가 십진수나
float으로 수동 변환해야 하는 필요성이 완화될 수 있다.

## 3.9. Calculating with Large Numerical Arrays

### 문제

배열이나 그리드 같은 큰 수치 데이터셋에 대해 계산을 수행해야 한다.

### 해법

배열과 관련된 무거운 계산에는 NumPy 라이브러리를 사용한다. NumPy의 주요 특징은 표준 Python list보다 훨씬 더 효율적이고 수학 계산에 더 적합한 배열 객체를 Python에 제공한다는 것이다:

```python
>>> # Python lists
>>> x = [1, 2, 3, 4]
>>> y = [5, 6, 7, 8]
>>> x * 2
[1, 2, 3, 4, 1, 2, 3, 4]
>>> x + 10
TypeError: can only concatenate list (not "int") to list

>>> # Numpy arrays
>>> import numpy as np
>>> ax = np.array([1, 2, 3, 4])
>>> ay = np.array([5, 6, 7, 8])
>>> ax * 2
array([2, 4, 6, 8])
>>> ax + 10
array([11, 12, 13, 14])
>>> ax + ay
array([ 6,  8, 10, 12])
>>> ax * ay
array([ 5, 12, 21, 32])
```

배열과 관련된 기본 수학 연산은 다르게 작동한다. Scalar 연산(예: `ax * 2` 또는 `ax + 10`)은 요소별로 연산을 적용한다.

전체 배열에 걸쳐 함수를 계산하기 매우 쉽고 빠르다:

```python
>>> def f(x):
...     return 3*x**2 - 2*x + 7
>>> f(ax)
array([ 8, 15, 28, 47])
```

NumPy는 배열 연산을 허용하는 "universal function" 모음을 제공한다:

```python
>>> np.sqrt(ax)
array([ 1.        ,  1.41421356,  1.73205081,  2.        ])
>>> np.cos(ax)
array([ 0.54030231, -0.41614684, -0.9899925 , -0.65364362])
```

Universal function을 사용하면 배열 요소를 한 번에 하나씩 반복하고 `math` 모듈의 함수를 사용하여 계산을 수행하는 것보다 수백 배 빠를 수 있다.

### 논의

NumPy 배열은 C나 Fortran과 동일한 방식으로 할당된다. 즉, 동질적인 데이터 타입으로 구성된 크고 연속적인 메모리 영역이다. 10,000 x 10,000 float의 2차원 그리드를 만드는 것도 문제가
없다:

```python
>>> grid = np.zeros(shape=(10000,10000), dtype=float)
```

NumPy의 매우 주목할 만한 측면은 Python의 list indexing 기능을 확장하는 방식이다:

```python
>>> a = np.array([[1, 2, 3, 4], [5, 6, 7, 8], [9, 10, 11, 12]])
>>> a[1]  # row 1 선택
array([5, 6, 7, 8])
>>> a[:,1]  # column 1 선택
array([ 2,  6, 10])
>>> a[1:3, 1:3]  # 하위 영역 선택
array([[ 6,  7],
       [10, 11]])
>>> a[1:3, 1:3] += 10  # 변경
>>> a + [100, 101, 102, 103]  # 모든 행에 row vector broadcast
array([[101, 103, 105, 107],
       [105, 117, 119, 111],
       [109, 121, 123, 115]])
>>> np.where(a < 10, a, 10)  # 조건부 할당
array([[ 1,  2,  3,  4],
       [ 5, 10, 10,  8],
       [ 9, 10, 10, 10]])
```

## 3.10. Performing Matrix and Linear Algebra Calculations

### 문제

행렬 곱셈, 행렬식 찾기, 선형 방정식 풀기 등 행렬 및 선형 대수 연산을 수행해야 한다.

### 해법

NumPy 라이브러리에는 이 목적으로 사용할 수 있는 matrix 객체가 있다:

```python
>>> import numpy as np
>>> m = np.matrix([[1,-2,3],[0,4,5],[7,8,-9]])
>>> m
matrix([[ 1, -2,  3],
        [ 0,  4,  5],
        [ 7,  8, -9]])

>>> m.T  # Transpose
matrix([[ 1,  0,  7],
        [-2,  4,  8],
        [ 3,  5, -9]])

>>> m.I  # Inverse
matrix([[ 0.33043478, -0.02608696,  0.09565217],
        [-0.15217391,  0.13043478,  0.02173913],
        [ 0.12173913,  0.09565217, -0.0173913 ]])

>>> v = np.matrix([[2],[3],[4]])  # Vector 생성 및 곱셈
>>> m * v
matrix([[ 8],
        [32],
        [ 2]])
```

더 많은 연산은 `numpy.linalg` 서브패키지에서 찾을 수 있다:

```python
>>> import numpy.linalg

>>> numpy.linalg.det(m)  # Determinant
-229.99999999999983

>>> numpy.linalg.eigvals(m)  # Eigenvalues
array([-13.11474312,   2.75956154,   6.35518158])

>>> x = numpy.linalg.solve(m, v)  # mx = v에서 x 풀기
>>> x
matrix([[ 0.96521739],
        [ 0.17391304],
        [ 0.46086957]])
>>> m * x
matrix([[ 2.],
        [ 3.],
        [ 4.]])
```

## 3.11. Picking Things at Random

### 문제

Sequence에서 무작위로 항목을 선택하거나 난수를 생성하고 싶다.

### 해법

`random` 모듈에는 난수와 무작위 항목 선택을 위한 다양한 함수가 있다. Sequence에서 무작위 항목을 선택하려면 `random.choice()`를 사용한다:

```python
>>> import random
>>> values = [1, 2, 3, 4, 5, 6]
>>> random.choice(values)
2
>>> random.choice(values)
3
```

선택된 항목이 추가 고려에서 제거되는 N개 항목의 샘플을 취하려면 `random.sample()`을 사용한다:

```python
>>> random.sample(values, 2)
[6, 2]
>>> random.sample(values, 3)
[4, 3, 1]
```

Sequence의 항목을 제자리에서 섞으려면 `random.shuffle()`을 사용한다:

```python
>>> random.shuffle(values)
>>> values
[2, 4, 6, 5, 3, 1]
```

무작위 정수를 생성하려면 `random.randint()`를 사용한다:

```python
>>> random.randint(0,10)
2
>>> random.randint(0,10)
5
```

0에서 1 범위의 균등 부동소수점 값을 생성하려면 `random.random()`을 사용한다:

```python
>>> random.random()
0.9406677561675867
>>> random.random()
0.133129581343897
```

정수로 표현된 N개의 random bit를 얻으려면 `random.getrandbits()`를 사용한다:

```python
>>> random.getrandbits(200)
335837000776573622800628485064121869519521710558559406913275
```

### 논의

`random` 모듈은 Mersenne Twister 알고리즘을 사용하여 난수를 계산한다. `random.seed()` 함수를 사용하여 초기 seed를 변경할 수 있다:

```python
random.seed()  # 시스템 시간이나 os.urandom() 기반 seed
random.seed(12345)  # 주어진 정수 기반 seed
random.seed(b'bytedata')  # 바이트 데이터 기반 seed
```

암호화와 관련된 프로그램에서는 `random()`의 함수를 사용해서는 안 된다. 그러한 기능이 필요하다면 대신 `ssl` 모듈의 함수를 고려하라.

## 3.12. Converting Days to Seconds, and Other Basic Time Conversions

### 문제

일을 초로, 시간을 분으로 등 간단한 시간 변환을 수행하는 코드가 필요하다.

### 해법

다양한 시간 단위와 관련된 변환 및 산술을 수행하려면 `datetime` 모듈을 사용한다. 시간 간격을 나타내려면 `timedelta` 인스턴스를 생성한다:

```python
>>> from datetime import timedelta
>>> a = timedelta(days=2, hours=6)
>>> b = timedelta(hours=4.5)
>>> c = a + b
>>> c.days
2
>>> c.seconds
37800
>>> c.seconds / 3600
10.5
>>> c.total_seconds() / 3600
58.5
```

특정 날짜와 시간을 나타내려면 `datetime` 인스턴스를 생성하고 표준 수학 연산을 사용하여 조작한다:

```python
>>> from datetime import datetime
>>> a = datetime(2012, 9, 23)
>>> print(a + timedelta(days=10))
2012-10-03 00:00:00

>>> b = datetime(2012, 12, 21)
>>> d = b - a
>>> d.days
89
>>> now = datetime.today()
>>> print(now + timedelta(minutes=10))
2012-12-21 15:04:43.094063
```

계산을 할 때 `datetime`은 윤년을 인식한다는 점에 유의해야 한다:

```python
>>> a = datetime(2012, 3, 1)
>>> b = datetime(2012, 2, 28)
>>> (a - b).days
2
>>> c = datetime(2013, 3, 1)
>>> d = datetime(2013, 2, 28)
>>> (c - d).days
1
```

### 논의

대부분의 기본 날짜 및 시간 조작 문제에는 `datetime` 모듈이 충분하다. 시간대 처리, 모호한 시간 범위, 휴일 날짜 계산 등 더 복잡한 날짜 조작을 수행해야 한다면 `dateutil` 모듈을 살펴보라.

`dateutil.relativedelta()` 함수로 많은 유사한 시간 계산을 수행할 수 있다. 한 가지 주목할 만한 특징은 월 처리와 관련된 일부 공백을 채운다는 것이다:

```python
>>> from dateutil.relativedelta import relativedelta
>>> a = datetime(2012, 9, 23)
>>> a + relativedelta(months=+1)
datetime.datetime(2012, 10, 23, 0, 0)
>>> a + relativedelta(months=+4)
datetime.datetime(2013, 1, 23, 0, 0)

>>> b = datetime(2012, 12, 21)
>>> d = relativedelta(b, a)
>>> d
relativedelta(months=+2, days=+28)
>>> d.months
2
>>> d.days
28
```

## 3.13. Determining Last Friday's Date

### 문제

마지막 금요일 같은 요일의 마지막 발생 날짜를 찾기 위한 일반적인 솔루션이 필요하다.

### 해법

Python의 `datetime` 모듈에는 이러한 계산을 수행하는 데 도움이 되는 유틸리티 함수와 클래스가 있다:

```python
from datetime import datetime, timedelta

weekdays = ['Monday', 'Tuesday', 'Wednesday', 'Thursday',
            'Friday', 'Saturday', 'Sunday']

def get_previous_byday(dayname, start_date=None):
    if start_date is None:
        start_date = datetime.today()
    day_num = start_date.weekday()
    day_num_target = weekdays.index(dayname)
    days_ago = (7 + day_num - day_num_target) % 7
    if days_ago == 0:
        days_ago = 7
    target_date = start_date - timedelta(days=days_ago)
    return target_date
```

사용 예:

```python
>>> datetime.today()
datetime.datetime(2012, 8, 28, 22, 4, 30, 263076)
>>> get_previous_byday('Monday')
datetime.datetime(2012, 8, 27, 22, 3, 57, 29045)
>>> get_previous_byday('Tuesday')  # 오늘이 아닌 이전 주
datetime.datetime(2012, 8, 21, 22, 4, 12, 629771)
>>> get_previous_byday('Friday')
datetime.datetime(2012, 8, 24, 22, 5, 9, 911393)
```

### 논의

이 레시피는 시작 날짜와 목표 날짜를 주의 숫자 위치(월요일을 0일로)에 매핑하여 작동한다. 그런 다음 모듈러 산술을 사용하여 목표 날짜가 마지막으로 발생한 날이 며칠 전인지 알아낸다.

많은 날짜 계산을 수행하는 경우, 대신 `python-dateutil` 패키지를 설치하는 것이 더 나을 수 있다:

```python
>>> from dateutil.relativedelta import relativedelta
>>> from dateutil.rrule import *
>>> d = datetime.now()
>>> print(d + relativedelta(weekday=FR))  # 다음 금요일
2012-12-28 16:31:52.718111
>>> print(d + relativedelta(weekday=FR(-1)))  # 지난 금요일
2012-12-21 16:31:52.718111
```

## 3.14. Finding the Date Range for the Current Month

### 문제

현재 월의 각 날짜를 반복하는 코드가 있고, 해당 날짜 범위를 계산하는 효율적인 방법이 필요하다.

### 해법

날짜를 반복하는 것은 모든 날짜의 리스트를 미리 작성할 필요가 없다. 범위의 시작 및 종료 날짜를 계산한 다음 `datetime.timedelta` 객체를 사용하여 날짜를 증가시키면 된다:

```python
from datetime import datetime, date, timedelta
import calendar

def get_month_range(start_date=None):
    if start_date is None:
        start_date = date.today().replace(day=1)
    _, days_in_month = calendar.monthrange(start_date.year, start_date.month)
    end_date = start_date + timedelta(days=days_in_month)
    return (start_date, end_date)
```

날짜 범위를 반복하기 쉽다:

```python
>>> a_day = timedelta(days=1)
>>> first_day, last_day = get_month_range()
>>> while first_day < last_day:
...     print(first_day)
...     first_day += a_day
2012-08-01
2012-08-02
2012-08-03
...
```

### 논의

이 레시피는 먼저 월의 첫째 날에 해당하는 날짜를 계산하여 작동한다. 이를 수행하는 빠른 방법은 date 또는 datetime 객체의 `replace()` 메서드를 사용하여 days 속성을 1로 설정하는 것이다.

`calendar.monthrange()` 함수는 해당 월의 일수를 알아내는 데 사용된다. 종료 날짜는 시작 날짜에 적절한 `timedelta`를 더하여 계산된다. 미묘하지만 중요한 점은 종료 날짜가 범위에 포함되지
않는다는 것이다(실제로는 다음 달의 첫째 날이다).

내장 `range()` 함수처럼 작동하지만 날짜용인 함수를 만들 수 있다면 좋을 것이다. 다행히 generator를 사용하여 매우 쉽게 구현할 수 있다:

```python
def date_range(start, stop, step):
    while start < stop:
        yield start
        start += step

>>> for d in date_range(datetime(2012, 9, 1), datetime(2012,10,1),
...                      timedelta(hours=6)):
...     print(d)
2012-09-01 00:00:00
2012-09-01 06:00:00
2012-09-01 12:00:00
...
```

## 3.15. Converting Strings into Datetimes

### 문제

애플리케이션이 문자열 형식의 시간 데이터를 수신하지만, 이러한 문자열을 `datetime` 객체로 변환하여 비문자열 연산을 수행하고 싶다.

### 해법

Python의 표준 `datetime` 모듈이 일반적으로 쉬운 솔루션이다:

```python
>>> from datetime import datetime
>>> text = '2012-09-20'
>>> y = datetime.strptime(text, '%Y-%m-%d')
>>> z = datetime.now()
>>> diff = z - y
>>> diff
datetime.timedelta(3, 77824, 177393)
```

### 논의

`datetime.strptime()` 메서드는 4자리 연도를 위한 `%Y`, 2자리 월을 위한 `%m` 같은 많은 포맷 코드를 지원한다. 이러한 포맷 플레이스홀더는 역방향으로도 작동한다:

```python
>>> z
datetime.datetime(2012, 9, 23, 21, 37, 4, 177393)
>>> nice_z = datetime.strftime(z, '%A %B %d, %Y')
>>> nice_z
'Sunday September 23, 2012'
```

`strptime()`의 성능은 순수 Python으로 작성되었고 모든 종류의 시스템 locale 설정을 처리해야 하기 때문에 기대하는 것보다 훨씬 나쁠 수 있다. 코드에서 많은 날짜를 파싱하고 정확한 형식을 알고
있다면, 맞춤형 솔루션을 만들어 훨씬 더 나은 성능을 얻을 수 있다:

```python
from datetime import datetime
def parse_ymd(s):
    year_s, mon_s, day_s = s.split('-')
    return datetime(int(year_s), int(mon_s), int(day_s))
```

테스트했을 때 이 함수는 `datetime.strptime()`보다 7배 이상 빠르게 실행된다.

## 3.16. Manipulating Dates Involving Time Zones

### 문제

2012년 12월 21일 오전 9시 30분(시카고)에 회의 전화가 예정되어 있었다. 인도 방갈로르의 친구는 몇 시 현지 시간에 참석해야 했는가?

### 해법

시간대와 관련된 거의 모든 문제에는 `pytz` 모듈을 사용해야 한다. 이 패키지는 많은 언어와 운영 체제에서 찾을 수 있는 시간대 정보의 사실상 표준인 Olson time zone database를 제공한다.

`pytz`의 주요 용도는 `datetime` 라이브러리로 생성된 단순 날짜를 현지화하는 것이다:

```python
>>> from datetime import datetime
>>> from pytz import timezone
>>> d = datetime(2012, 12, 21, 9, 30, 0)
>>> print(d)
2012-12-21 09:30:00

>>> central = timezone('US/Central')  # 시카고 시간으로 현지화
>>> loc_d = central.localize(d)
>>> print(loc_d)
2012-12-21 09:30:00-06:00
```

날짜가 현지화되면 다른 시간대로 변환할 수 있다. 방갈로르의 동일한 시간을 찾으려면:

```python
>>> bang_d = loc_d.astimezone(timezone('Asia/Kolkata'))
>>> print(bang_d)
2012-12-21 21:00:00+05:30
```

현지화된 날짜로 산술을 수행하려면 일광 절약 시간 전환 및 기타 세부 사항을 특히 알고 있어야 한다. 단순한 산술은 잘못된 결과를 제공한다:

```python
>>> d = datetime(2013, 3, 10, 1, 45)
>>> loc_d = central.localize(d)
>>> later = loc_d + timedelta(minutes=30)
>>> print(later)
2013-03-10 02:15:00-06:00  # WRONG!
```

이를 수정하려면 시간대의 `normalize()` 메서드를 사용한다:

```python
>>> from datetime import timedelta
>>> later = central.normalize(loc_d + timedelta(minutes=30))
>>> print(later)
2013-03-10 03:15:00-05:00
```

### 논의

현지화된 날짜 처리를 위한 일반적인 전략은 모든 날짜를 UTC 시간으로 변환하고 모든 내부 저장 및 조작에 이를 사용하는 것이다:

```python
>>> print(loc_d)
2013-03-10 01:45:00-06:00
>>> utc_d = loc_d.astimezone(pytz.utc)
>>> print(utc_d)
2013-03-10 07:45:00+00:00
```

UTC로 변환하면 일광 절약 시간과 관련된 문제에 대해 걱정할 필요가 없다. 현지화된 시간으로 날짜를 출력하려면 나중에 적절한 시간대로 변환하면 된다:

```python
>>> later_utc = utc_d + timedelta(minutes=30)
>>> print(later_utc.astimezone(central))
2013-03-10 03:15:00-05:00
```

시간대 이름을 알아내려면 ISO 3166 국가 코드를 키로 사용하여 `pytz.country_timezones` dictionary를 참조할 수 있다:

```python
>>> pytz.country_timezones['IN']
['Asia/Kolkata']
```