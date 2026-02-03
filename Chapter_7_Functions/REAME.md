# CHAPTER 7 Functions

이 챕터는 def 문을 사용한 함수 정의를 넘어서 더 고급스럽고 특별한 함수 정의 및 사용 패턴을 다룬다. 주요 주제는 기본 인자, 가변 인자 함수, keyword-only 인자, 어노테이션, 클로저를 포함한다.
또한 콜백 함수와 관련된 까다로운 제어 흐름 및 데이터 전달 문제도 다룬다.

## 7.1. 임의 개수의 인자를 받는 함수 작성

임의 개수의 positional 인자를 받으려면 `*` 인자를 사용한다.

```python
def avg(first, *rest):
    return (first + sum(rest)) / (1 + len(rest))

# 사용 예
avg(1, 2)       # 1.5
avg(1, 2, 3, 4) # 2.5
```

이 예시에서 `rest`는 전달된 모든 추가 positional 인자의 튜플이다.

임의 개수의 keyword 인자를 받으려면 `**`로 시작하는 인자를 사용한다.

```python
import html

def make_element(name, value, **attrs):
    keyvals = [' %s="%s"' % item for item in attrs.items()]
    attr_str = ''.join(keyvals)
    element = '<{name}{attrs}>{value}</{name}>'.format(
        name=name,
        attrs=attr_str,
        value=html.escape(value))
    return element

# 예시
make_element('item', 'Albatross', size='large', quantity=6)
# '<item size="large" quantity="6">Albatross</item>' 생성
```

여기서 `attrs`는 전달된 keyword 인자를 담는 딕셔너리다.

positional과 keyword 인자를 모두 받으려면 `*`와 `**`를 함께 사용한다.

```python
def anyargs(*args, **kwargs):
    print(args)   # 튜플
    print(kwargs) # 딕셔너리
```

`*` 인자는 함수 정의에서 마지막 positional 인자로만 나타날 수 있다. `**` 인자는 마지막 인자로만 나타날 수 있다. `*` 인자 뒤에도 인자가 나올 수 있는데, 이는 keyword-only 인자라
한다(Recipe 7.2 참조).

## 7.2. Keyword 인자만 받는 함수 작성

특정 인자를 keyword로만 받으려면 `*` 인자나 단독 `*` 뒤에 keyword 인자를 배치한다.

```python
def recv(maxsize, *, block):
    'Receives a message'
    pass

recv(1024, True)       # TypeError
recv(1024, block=True) # Ok
```

가변 개수의 positional 인자를 받는 함수에서도 이 기법을 사용할 수 있다.

```python
def minimum(*values, clip=None):
    m = min(values)
    if clip is not None:
        m = clip if clip > m else m
    return m

minimum(1, 5, 2, -5, 10)         # -5 반환
minimum(1, 5, 2, -5, 10, clip=0) # 0 반환
```

Keyword-only 인자는 선택적 함수 인자를 지정할 때 코드의 명확성을 높이는 좋은 방법이다. 예를 들어:

```python
msg = recv(1024, False)  # False가 무엇을 의미하는지 불명확
msg = recv(1024, block=False)  # 훨씬 명확함
```

Keyword-only 인자는 `**kwargs` 트릭보다 선호되는데, help를 요청할 때 제대로 표시되기 때문이다.

```python
>>> help(recv)
Help on function recv in module __main__:
recv(maxsize, *, block)
    Receives a message
```

## 7.3. 함수 인자에 메타데이터 첨부

함수 인자 어노테이션은 함수 사용 방법에 대한 힌트를 제공하는 유용한 방법이다.

```python
def add(x:int, y:int) -> int:
    return x + y
```

Python 인터프리터는 어노테이션에 의미를 부여하지 않는다. 타입 체크를 하지 않으며, Python의 동작을 변경하지도 않는다. 그러나 소스 코드를 읽는 다른 사람에게 유용한 힌트를 줄 수 있다. 서드파티 도구와
프레임워크도 어노테이션에 의미를 부여할 수 있다. 또한 문서에도 나타난다.

```python
>>> help(add)
Help on function add in module __main__:
add(x: int, y: int) -> int
```

어떤 종류의 객체든 어노테이션으로 첨부할 수 있지만, 클래스나 문자열이 가장 의미 있는 경우가 많다.

함수 어노테이션은 함수의 `__annotations__` 속성에 저장된다.

```python
>>> add.__annotations__
{'y': <class 'int'>, 'return': <class 'int'>, 'x': <class 'int'>}
```

어노테이션의 주요 용도는 문서화다. Python에는 타입 선언이 없기 때문에, 소스 코드만 읽고는 함수에 무엇을 전달해야 하는지 알기 어려울 수 있다. 어노테이션은 더 많은 힌트를 제공한다.

## 7.4. 함수에서 여러 값 반환

함수에서 여러 값을 반환하려면 단순히 튜플을 반환하면 된다.

```python
>>> def myfun():
...     return 1, 2, 3
...
>>> a, b, c = myfun()
>>> a
1
>>> b
2
>>> c
3
```

`myfun()`이 여러 값을 반환하는 것처럼 보이지만, 실제로는 튜플이 생성되는 것이다. 튜플을 형성하는 것은 괄호가 아니라 쉼표다.

```python
>>> a = (1, 2)  # 괄호 사용
>>> a
(1, 2)
>>> b = 1, 2    # 괄호 없이
>>> b
(1, 2)
```

튜플을 반환하는 함수를 호출할 때는 결과를 여러 변수에 할당하는 것이 일반적이다. 이는 단순히 튜플 언패킹이다(Recipe 1.1). 반환 값을 단일 변수에 할당할 수도 있다.

```python
>>> x = myfun()
>>> x
(1, 2, 3)
```

## 7.5. 기본 인자를 가진 함수 정의

선택적 인자가 있고 기본값을 가진 함수를 정의하려면, 정의 시 값을 할당하고 기본 인자가 마지막에 오도록 한다.

```python
def spam(a, b=42):
    print(a, b)

spam(1)    # Ok. a=1, b=42
spam(1, 2) # Ok. a=1, b=2
```

기본값이 mutable 컨테이너(리스트, 집합, 딕셔너리)인 경우, `None`을 기본값으로 사용하고 다음과 같이 코드를 작성한다.

```python
def spam(a, b=None):
    if b is None:
        b = []
    ...
```

선택적 인자에 흥미로운 값이 주어졌는지 테스트하려면 다음 관용구를 사용한다.

```python
_no_value = object()

def spam(a, b=_no_value):
    if b is _no_value:
        print('No b value supplied')
    ...
```

사용 예:

```python
>>> spam(1)
No b value supplied
>>> spam(1, 2)   # b = 2
>>> spam(1, None) # b = None
```

값을 전혀 전달하지 않는 것과 `None` 값을 전달하는 것 사이에는 차이가 있다는 점을 주의해야 한다.

### Discussion

기본값으로 할당된 값은 함수 정의 시점에 한 번만 바인딩된다.

```python
>>> x = 42
>>> def spam(a, b=x):
...     print(a, b)
...
>>> spam(1)
1 42
>>> x = 23  # 영향 없음
>>> spam(1)
1 42
```

변수 `x`를 변경해도 기본값에는 영향이 없다. 기본값은 함수 정의 시점에 고정되기 때문이다.

기본값으로 할당되는 값은 항상 `None`, `True`, `False`, 숫자, 문자열과 같은 불변 객체여야 한다. 특히 다음과 같은 코드는 절대 작성하지 말아야 한다.

```python
def spam(a, b=[]):  # 안됨!
    ...
```

이렇게 하면 기본값이 함수를 벗어나 수정될 경우 문제가 발생할 수 있다. 이러한 변경은 향후 함수 호출에서 기본값을 영구적으로 변경한다.

```python
>>> def spam(a, b=[]):
...     print(b)
...     return b
...
>>> x = spam(1)
>>> x
[]
>>> x.append(99)
>>> x.append('Yow!')
>>> x
[99, 'Yow!']
>>> spam(1)  # 수정된 리스트가 반환됨!
[99, 'Yow!']
```

이를 피하려면 기본값으로 `None`을 할당하고 함수 내부에서 체크를 추가하는 것이 좋다.

`None`을 테스트할 때 `is` 연산자를 사용하는 것이 중요하다. 다음과 같은 실수를 하기 쉽다.

```python
def spam(a, b=None):
    if not b:  # 안됨! 'b is None' 사용해야 함
        b = []
    ...
```

`None`이 `False`로 평가되지만, 다른 많은 객체(길이가 0인 문자열, 리스트, 튜플, 딕셔너리 등)도 마찬가지다. 따라서 이 테스트는 특정 입력을 잘못 누락된 것으로 처리할 수 있다.

```python
>>> spam(1)      # OK
>>> x = []
>>> spam(1, x)   # 조용한 오류. x 값이 기본값으로 덮어써짐
>>> spam(1, 0)   # 조용한 오류. 0 무시됨
>>> spam(1, '')  # 조용한 오류. '' 무시됨
```

마지막 부분은 미묘한데, 선택적 인자에 값(어떤 값이든)이 제공되었는지 테스트하는 함수다. 기본값으로 `None`, `0`, `False`를 사용해서는 사용자 제공 인자의 존재를 테스트할 수 없다(이들 모두 사용자가
제공할 수 있는 유효한 값이기 때문). 따라서 테스트할 다른 것이 필요하다.

이 문제를 해결하기 위해 솔루션에 표시된 것처럼 `object`의 고유한 private 인스턴스를 생성할 수 있다(`_no_value` 변수). 함수에서 이 특수 값에 대해 제공된 인자의 identity를 체크하여
인자가 제공되었는지 확인한다. 사용자가 `_no_value` 인스턴스를 입력 값으로 전달할 가능성은 극히 낮다. 따라서 인자가 제공되었는지 확인할 때 체크할 안전한 값이 된다.

`object()` 사용이 이상해 보일 수 있다. `object`는 Python의 거의 모든 객체의 공통 기본 클래스 역할을 하는 클래스다. `object`의 인스턴스를 생성할 수 있지만, 주목할 만한 메서드도 없고
인스턴스 데이터도 없어서 완전히 흥미롭지 않다(하부 인스턴스 딕셔너리가 없기 때문에 속성도 설정할 수 없다). 할 수 있는 유일한 것은 identity 테스트다. 이것이 솔루션에서 보여준 것처럼 특수 값으로 유용하게
만든다.

## 7.6. 익명 또는 인라인 함수 정의

표현식을 평가하는 것 외에는 아무것도 하지 않는 간단한 함수는 lambda 표현식으로 대체할 수 있다.

```python
>>> add = lambda x, y: x + y
>>> add(2,3)
5
>>> add('hello', 'world')
'helloworld'
```

lambda 사용은 다음과 같이 타이핑한 것과 동일하다.

```python
>>> def add(x, y):
...     return x + y
...
>>> add(2,3)
5
```

일반적으로 lambda는 정렬이나 데이터 축소와 같은 다른 연산의 맥락에서 사용된다.

```python
>>> names = ['David Beazley', 'Brian Jones',
...          'Raymond Hettinger', 'Ned Batchelder']
>>> sorted(names, key=lambda name: name.split()[-1].lower())
['Ned Batchelder', 'David Beazley', 'Raymond Hettinger', 'Brian Jones']
```

lambda는 간단한 함수를 정의할 수 있지만, 사용이 매우 제한적이다. 특히 단일 표현식만 지정할 수 있으며, 그 결과가 반환 값이 된다. 이는 여러 문, 조건문, 반복, 예외 처리를 포함한 다른 언어 기능을
포함할 수 없음을 의미한다.

lambda를 전혀 사용하지 않고도 많은 Python 코드를 작성할 수 있다. 그러나 다양한 표현식을 평가하는 많은 작은 함수를 작성하는 프로그램이나 사용자가 콜백 함수를 제공해야 하는 프로그램에서 가끔 발견할 수
있다.

## 7.7. 익명 함수에서 변수 캡처

lambda로 정의한 익명 함수에서 정의 시점의 특정 변수 값을 캡처해야 할 때가 있다.

다음 코드의 동작을 고려해보자.

```python
>>> x = 10
>>> a = lambda y: x + y
>>> x = 20
>>> b = lambda y: x + y
```

`a(10)`과 `b(10)`의 값은 무엇일까? 20과 30이라고 생각한다면 틀렸다.

```python
>>> a(10)
30
>>> b(10)
30
```

문제는 lambda 표현식에서 사용된 `x` 값이 정의 시점이 아니라 런타임에 바인딩되는 자유 변수라는 것이다. 따라서 lambda 표현식의 `x` 값은 실행 시점의 `x` 변수 값이다.

```python
>>> x = 15
>>> a(10)
25
>>> x = 3
>>> a(10)
13
```

익명 함수가 정의 시점의 값을 캡처하고 유지하도록 하려면, 값을 기본값으로 포함시킨다.

```python
>>> x = 10
>>> a = lambda y, x=x: x + y
>>> x = 20
>>> b = lambda y, x=x: x + y
>>> a(10)
20
>>> b(10)
30
```

### Discussion

이 레시피에서 다룬 문제는 list comprehension이나 루프에서 lambda 표현식 리스트를 생성하고 lambda 함수가 정의 시점의 반복 변수를 기억하기를 기대하는 코드에서 발생하는 경향이 있다.

```python
>>> funcs = [lambda x: x+n for n in range(5)]
>>> for f in funcs:
...     print(f(0))
...
4
4
4
4
4
```

모든 함수가 반복 중 `n`의 마지막 값을 생각하는 것을 주목하라. 이제 다음과 비교해보자.

```python
>>> funcs = [lambda x, n=n: x+n for n in range(5)]
>>> for f in funcs:
...     print(f(0))
...
0
1
2
3
4
```

보다시피 함수들은 이제 정의 시점의 `n` 값을 캡처한다.

## 7.8. N개 인자를 받는 Callable을 더 적은 인자로 작동하게 만들기

함수의 인자 수를 줄여야 한다면 `functools.partial()`을 사용해야 한다. `partial()` 함수는 하나 이상의 인자에 고정 값을 할당하여 후속 호출에 제공해야 하는 인자 수를 줄일 수 있다.

```python
def spam(a, b, c, d):
    print(a, b, c, d)
```

`partial()`을 사용하여 특정 인자 값을 고정하는 예:

```python
>>> from functools import partial
>>> s1 = partial(spam, 1)  # a = 1
>>> s1(2, 3, 4)
1 2 3 4
>>> s1(4, 5, 6)
1 4 5 6
>>> s2 = partial(spam, d=42)  # d = 42
>>> s2(1, 2, 3)
1 2 3 42
>>> s2(4, 5, 5)
4 5 5 42
>>> s3 = partial(spam, 1, 2, d=42)  # a = 1, b = 2, d = 42
>>> s3(3)
1 2 3 42
>>> s3(4)
1 2 4 42
>>> s3(5)
1 2 5 42
```

`partial()`은 특정 인자의 값을 고정하고 결과로 새로운 callable을 반환한다. 이 새로운 callable은 아직 할당되지 않은 인자를 받아, `partial()`에 주어진 인자와 결합하여 모든 것을
원래 함수에 전달한다.

### Discussion

이 레시피는 호환되지 않는 코드를 함께 작동시키는 문제와 관련이 있다. 일련의 예제가 설명에 도움이 될 것이다.

첫 번째 예로, (x,y) 좌표의 튜플로 표현된 점 리스트가 있다고 가정하자. 다음 함수를 사용하여 두 점 사이의 거리를 계산할 수 있다.

```python
points = [(1, 2), (3, 4), (5, 6), (7, 8)]
import math

def distance(p1, p2):
    x1, y1 = p1
    x2, y2 = p2
    return math.hypot(x2 - x1, y2 - y1)
```

다른 점으로부터의 거리에 따라 모든 점을 정렬하고 싶다고 가정하자. 리스트의 `sort()` 메서드는 정렬을 커스터마이즈하는 데 사용할 수 있는 key 인자를 받지만, 단일 인자를 받는 함수에서만 작동한다(따라서
`distance()`는 적합하지 않다). `partial()`을 사용하여 수정하는 방법은 다음과 같다.

```python
>>> pt = (4, 3)
>>> points.sort(key=partial(distance, pt))
>>> points
[(3, 4), (1, 2), (5, 6), (7, 8)]
```

이 아이디어의 확장으로, `partial()`은 다른 라이브러리에서 사용되는 콜백 함수의 인자 시그니처를 조정하는 데 자주 사용될 수 있다. 예를 들어, multiprocessing을 사용하여 비동기적으로 결과를
계산하고 결과와 선택적 로깅 인자를 받는 콜백 함수에 전달하는 코드:

```python
def output_result(result, log=None):
    if log is not None:
        log.debug('Got: %r', result)

def add(x, y):
    return x + y

if __name__ == '__main__':
    import logging
    from multiprocessing import Pool
    from functools import partial
    
    logging.basicConfig(level=logging.DEBUG)
    log = logging.getLogger('test')
    
    p = Pool()
    p.apply_async(add, (3, 4), callback=partial(output_result, log=log))
    p.close()
    p.join()
```

`apply_async()`를 사용하여 콜백 함수를 제공할 때, 추가 로깅 인자는 `partial()`을 사용하여 제공된다. multiprocessing은 이에 대해 전혀 알지 못하며, 단순히 단일 값으로 콜백
함수를 호출한다.

비슷한 예로, 네트워크 서버 작성 문제를 고려하자. socketserver 모듈은 이를 비교적 쉽게 만든다. 예를 들어, 간단한 echo 서버:

```python
from socketserver import StreamRequestHandler, TCPServer

class EchoHandler(StreamRequestHandler):
    def handle(self):
        for line in self.rfile:
            self.wfile.write(b'GOT:' + line)

serv = TCPServer(('', 15000), EchoHandler)
serv.serve_forever()
```

`EchoHandler` 클래스에 추가 구성 인자를 받는 `__init__()` 메서드를 제공하고 싶다고 가정하자.

```python
class EchoHandler(StreamRequestHandler):
    def __init__(self, *args, ack, **kwargs):
        self.ack = ack
        super().__init__(*args, **kwargs)
    
    def handle(self):
        for line in self.rfile:
            self.wfile.write(self.ack + line)
```

이 변경을 하면 `TCPServer` 클래스에 연결할 명확한 방법이 없다는 것을 알게 된다. 실제로 코드는 다음과 같은 예외를 생성하기 시작한다.

```
TypeError: __init__() missing 1 required keyword-only argument: 'ack'
```

그러나 `partial()`을 사용하면 쉽게 해결할 수 있다.

```python
from functools import partial
serv = TCPServer(('', 15000), partial(EchoHandler, ack=b'RECEIVED:'))
serv.serve_forever()
```

`partial()`의 기능은 때때로 lambda 표현식으로 대체되기도 한다. 예를 들어, 이전 예제는 다음과 같은 문을 사용할 수 있다.

```python
points.sort(key=lambda p: distance(pt, p))
p.apply_async(add, (3, 4), callback=lambda result: output_result(result, log))
serv = TCPServer(('', 15000),
                 lambda *args, **kwargs: EchoHandler(*args,
                                                      ack=b'RECEIVED:',
                                                      **kwargs))
```

이 코드는 작동하지만, 더 장황하고 읽는 사람에게 잠재적으로 훨씬 더 혼란스러울 수 있다. `partial()` 사용은 의도(일부 인자에 값 제공)에 대해 좀 더 명시적이다.

## 7.9. 단일 메서드 클래스를 함수로 대체

`__init__()` 외에 단일 메서드만 정의하는 클래스가 있는 경우, 코드를 단순화하기 위해 간단한 함수를 선호할 수 있다.

많은 경우 단일 메서드 클래스는 클로저를 사용하여 함수로 전환할 수 있다. 예를 들어, 다음 클래스는 사용자가 템플릿 스킴을 사용하여 URL을 가져올 수 있게 한다.

```python
from urllib.request import urlopen

class UrlTemplate:
    def __init__(self, template):
        self.template = template
    
    def open(self, **kwargs):
        return urlopen(self.template.format_map(kwargs))

# 사용 예
yahoo = UrlTemplate('http://finance.yahoo.com/d/quotes.csv?s={names}&f={fields}')
for line in yahoo.open(names='IBM,AAPL,FB', fields='sl1c1v'):
    print(line.decode('utf-8'))
```

클래스를 훨씬 간단한 함수로 대체할 수 있다.

```python
def urltemplate(template):
    def opener(**kwargs):
        return urlopen(template.format_map(kwargs))
    return opener

# 사용 예
yahoo = urltemplate('http://finance.yahoo.com/d/quotes.csv?s={names}&f={fields}')
for line in yahoo(names='IBM,AAPL,FB', fields='sl1c1v'):
    print(line.decode('utf-8'))
```

### Discussion

많은 경우 단일 메서드 클래스를 가지는 유일한 이유는 메서드에서 사용할 추가 상태를 저장하기 위해서다. 예를 들어, `UrlTemplate` 클래스의 유일한 목적은 `open()` 메서드에서 사용할 수 있도록
template 값을 어딘가에 보관하는 것이다.

솔루션에 표시된 것처럼 내부 함수나 클로저를 사용하는 것이 종종 더 우아하다. 간단히 말해, 클로저는 단지 함수일 뿐이지만 함수 내부에서 사용되는 변수의 추가 환경을 가지고 있다. 클로저의 핵심 특징은 정의된 환경을
기억한다는 것이다. 따라서 솔루션에서 `opener()` 함수는 template 인자의 값을 기억하고 후속 호출에서 사용한다.

코드를 작성할 때 함수에 추가 상태를 첨부하는 문제에 직면하면 클로저를 생각하라. 클로저는 함수를 본격적인 클래스로 전환하는 대안보다 더 최소화되고 우아한 솔루션인 경우가 많다.

## 7.10. 콜백 함수와 함께 추가 상태 전달

콜백 함수(이벤트 핸들러, completion 콜백 등)를 사용하는 코드를 작성할 때, 콜백 함수 내부에서 사용할 추가 상태를 콜백 함수가 전달하도록 하고 싶을 때가 있다.

다음 함수는 콜백을 호출한다고 가정하자.

```python
def apply_async(func, args, *, callback):
    # 결과 계산
    result = func(*args)
    # 결과와 함께 콜백 호출
    callback(result)
```

사용 예:

```python
>>> def print_result(result):
...     print('Got:', result)
...
>>> def add(x, y):
...     return x + y
...
>>> apply_async(add, (2, 3), callback=print_result)
Got: 5
>>> apply_async(add, ('hello', 'world'), callback=print_result)
Got: helloworld
```

`print_result()` 함수는 단일 인자(결과)만 받는다. 다른 정보는 전달되지 않는다. 콜백이 다른 변수나 환경의 일부와 상호 작용하도록 하고 싶을 때 이러한 정보 부족이 문제가 될 수 있다.

콜백에 추가 정보를 전달하는 한 가지 방법은 간단한 함수 대신 bound 메서드를 사용하는 것이다. 예를 들어, 이 클래스는 결과를 받을 때마다 증가하는 내부 시퀀스 번호를 유지한다.

```python
class ResultHandler:
    def __init__(self):
        self.sequence = 0
    
    def handler(self, result):
        self.sequence += 1
        print('[{}] Got: {}'.format(self.sequence, result))
```

사용:

```python
>>> r = ResultHandler()
>>> apply_async(add, (2, 3), callback=r.handler)
[1] Got: 5
>>> apply_async(add, ('hello', 'world'), callback=r.handler)
[2] Got: helloworld
```

클래스의 대안으로 클로저를 사용하여 상태를 캡처할 수도 있다.

```python
def make_handler():
    sequence = 0
    def handler(result):
        nonlocal sequence
        sequence += 1
        print('[{}] Got: {}'.format(sequence, result))
    return handler
```

사용 예:

```python
>>> handler = make_handler()
>>> apply_async(add, (2, 3), callback=handler)
[1] Got: 5
>>> apply_async(add, ('hello', 'world'), callback=handler)
[2] Got: helloworld
```

또 다른 변형으로 coroutine을 사용하여 같은 것을 수행할 수 있다.

```python
def make_handler():
    sequence = 0
    while True:
        result = yield
        sequence += 1
        print('[{}] Got: {}'.format(sequence, result))
```

coroutine의 경우 `send()` 메서드를 콜백으로 사용한다.

```python
>>> handler = make_handler()
>>> next(handler)  # yield로 진행
>>> apply_async(add, (2, 3), callback=handler.send)
[1] Got: 5
>>> apply_async(add, ('hello', 'world'), callback=handler.send)
[2] Got: helloworld
```

마지막으로 추가 인자와 partial 함수 적용을 사용하여 콜백에 상태를 전달할 수도 있다.

```python
>>> class SequenceNo:
...     def __init__(self):
...         self.sequence = 0
...
>>> def handler(result, seq):
...     seq.sequence += 1
...     print('[{}] Got: {}'.format(seq.sequence, result))
...
>>> seq = SequenceNo()
>>> from functools import partial
>>> apply_async(add, (2, 3), callback=partial(handler, seq=seq))
[1] Got: 5
>>> apply_async(add, ('hello', 'world'), callback=partial(handler, seq=seq))
[2] Got: helloworld
```

### Discussion

콜백 함수를 기반으로 한 소프트웨어는 거대하게 얽힌 혼란으로 변할 위험이 있다. 문제의 일부는 콜백 함수가 콜백 실행으로 이어진 초기 요청을 만든 코드와 종종 분리되어 있다는 것이다. 따라서 요청을 만들고 결과를
처리하는 것 사이의 실행 환경이 효과적으로 손실된다. 콜백 함수가 여러 단계를 포함하는 절차를 계속하도록 하려면 관련 상태를 저장하고 복원하는 방법을 알아내야 한다.

상태를 캡처하고 전달하는 데 유용한 두 가지 주요 접근 방식이 있다. 인스턴스에 상태를 전달하거나(bound 메서드에 첨부) 클로저(내부 함수)에 전달할 수 있다. 두 가지 기법 중 클로저가 아마도 좀 더 가볍고
자연스러운데, 단순히 함수로 구축되기 때문이다. 또한 사용되는 모든 변수를 자동으로 캡처한다. 따라서 저장해야 하는 정확한 상태에 대해 걱정할 필요가 없다(코드에서 자동으로 결정된다).

클로저를 사용하는 경우 가변 변수에 주의를 기울여야 한다. 솔루션에서 `nonlocal` 선언은 콜백 내에서 sequence 변수가 수정되고 있음을 나타내는 데 사용된다. 이 선언이 없으면 오류가 발생한다.

콜백 핸들러로서의 coroutine 사용은 클로저 접근 방식과 밀접하게 관련되어 있다는 점에서 흥미롭다. 어떤 의미에서는 단일 함수만 있기 때문에 더 깔끔하다. 또한 변수는 `nonlocal` 선언 걱정 없이
자유롭게 수정할 수 있다. 잠재적인 단점은 coroutine이 Python의 다른 부분만큼 잘 이해되지 않는다는 것이다. coroutine을 사용하기 전에 `next()`를 호출해야 하는 것과 같은 몇 가지 까다로운
부분도 있다. 이것은 실제로 잊기 쉬운 것이다. 그럼에도 불구하고 coroutine은 인라인 콜백 정의와 같은 다른 잠재적 용도가 있다(다음 레시피에서 다룬다).

`partial()`을 포함하는 마지막 기법은 콜백에 추가 값을 전달하기만 하면 되는 경우에 유용하다. `partial()` 대신 때때로 lambda를 사용하여 같은 것을 수행하는 것을 볼 수 있다.

```python
>>> apply_async(add, (2, 3), callback=lambda r: handler(r, seq))
[1] Got: 5
```

더 많은 예제는 Recipe 7.8을 참조하라. 이는 `partial()`을 사용하여 인자 시그니처를 변경하는 방법을 보여준다.

## 7.11. 콜백 함수 인라인화

콜백 함수를 사용하는 코드를 작성하지만, 작은 함수의 확산과 당황스러운 제어 흐름에 대해 우려하고 있다. 코드가 일반적인 절차적 단계 시퀀스처럼 보이게 만드는 방법을 원한다.

콜백 함수는 generator와 coroutine을 사용하여 함수에 인라인화할 수 있다. 다음과 같이 작업을 수행하고 콜백을 호출하는 함수가 있다고 가정하자(Recipe 7.10 참조).

```python
def apply_async(func, args, *, callback):
    # 결과 계산
    result = func(*args)
    # 결과와 함께 콜백 호출
    callback(result)
```

이제 `Async` 클래스와 `inlined_async` 데코레이터를 포함하는 다음 지원 코드를 살펴보자.

```python
from queue import Queue
from functools import wraps

class Async:
    def __init__(self, func, args):
        self.func = func
        self.args = args

def inlined_async(func):
    @wraps(func)
    def wrapper(*args):
        f = func(*args)
        result_queue = Queue()
        result_queue.put(None)
        while True:
            result = result_queue.get()
            try:
                a = f.send(result)
                apply_async(a.func, a.args, callback=result_queue.put)
            except StopIteration:
                break
    return wrapper
```

이 두 코드 조각을 사용하면 yield 문을 사용하여 콜백 단계를 인라인화할 수 있다.

```python
def add(x, y):
    return x + y

@inlined_async
def test():
    r = yield Async(add, (2, 3))
    print(r)
    r = yield Async(add, ('hello', 'world'))
    print(r)
    for n in range(10):
        r = yield Async(add, (n, n))
        print(r)
    print('Goodbye')
```

`test()`를 호출하면 다음과 같은 출력을 얻는다.

```
5
helloworld
0
2
4
6
8
10
12
14
16
18
Goodbye
```

특수 데코레이터와 yield 사용을 제외하면, 콜백 함수가 어디에도 나타나지 않는다는 것을 알 수 있다(무대 뒤에서 제외).

### Discussion

이 레시피는 콜백 함수, generator, 제어 흐름에 대한 지식을 실제로 테스트할 것이다.

첫째, 콜백을 포함하는 코드에서 전체 요점은 현재 계산이 중단되고 나중에 어떤 시점에 재개된다는 것이다(예: 비동기적으로). 계산이 재개될 때, 콜백이 실행되어 처리를 계속한다. `apply_async()` 함수는
콜백 실행의 필수 부분을 보여주지만, 실제로는 훨씬 더 복잡할 수 있다(스레드, 프로세스, 이벤트 핸들러 등 포함).

계산이 중단되고 재개된다는 아이디어는 자연스럽게 generator 함수의 실행 모델에 매핑된다. 특히 yield 연산은 generator 함수가 값을 emit하고 중단되도록 한다. `__next__()` 또는
`send()` 메서드에 대한 후속 호출은 다시 시작한다.

이를 염두에 두고, 이 레시피의 핵심은 `inline_async()` 데코레이터 함수에 있다. 핵심 아이디어는 데코레이터가 generator 함수를 모든 yield 문을 통해 한 번에 하나씩 단계별로 진행한다는
것이다. 이를 위해 결과 큐가 생성되고 초기에 `None` 값으로 채워진다. 그런 다음 루프가 시작되어 결과가 큐에서 팝되고 generator로 전송된다. 이것은 다음 yield로 진행하고, 이 시점에서
`Async`의 인스턴스가 수신된다. 루프는 함수와 인자를 보고 비동기 계산 `apply_async()`를 시작한다. 그러나 이 계산의 가장 교활한 부분은 일반 콜백 함수를 사용하는 대신 콜백이 큐의 `put()`
메서드로 설정된다는 것이다.

이 시점에서 정확히 무슨 일이 일어나는지는 다소 열려 있다. 메인 루프는 즉시 맨 위로 돌아가 큐에서 `get()` 연산을 실행한다. 데이터가 있으면 `put()` 콜백에 의해 거기에 배치된 결과여야 한다. 아무것도
없으면 연산은 블록되어 미래의 어느 시점에 결과가 도착하기를 기다린다. 어떻게 일어날지는 `apply_async()` 함수의 정확한 구현에 달려 있다.

이렇게 미친 것이 작동할지 의심스럽다면, multiprocessing 라이브러리로 시도하고 별도의 프로세스에서 비동기 연산을 실행할 수 있다.

```python
if __name__ == '__main__':
    import multiprocessing
    pool = multiprocessing.Pool()
    apply_async = pool.apply_async
    
    # 테스트 함수 실행
    test()
```

실제로 작동하는 것을 발견할 것이지만, 제어 흐름을 풀어내려면 더 많은 커피가 필요할 수 있다.

generator 함수 뒤에 까다로운 제어 흐름을 숨기는 것은 표준 라이브러리와 서드파티 패키지의 다른 곳에서도 발견된다. 예를 들어, contextlib의 `@contextmanager` 데코레이터는 yield
문을 통해 context manager에서 진입과 종료를 결합하는 유사한 마음을 구부리는 트릭을 수행한다. 인기 있는 Twisted 패키지에도 유사한 인라인 콜백이 있다.

## 7.12. 클로저 내부에 정의된 변수 접근

클로저를 내부 변수에 접근하고 수정할 수 있는 함수로 확장하고 싶다.

일반적으로 클로저의 내부 변수는 외부 세계에 완전히 숨겨져 있다. 그러나 accessor 함수를 작성하고 함수 속성으로 클로저에 첨부하여 접근을 제공할 수 있다.

```python
def sample():
    n = 0
    # 클로저 함수
    def func():
        print('n=', n)
    
    # n에 대한 accessor 메서드
    def get_n():
        return n
    
    def set_n(value):
        nonlocal n
        n = value
    
    # 함수 속성으로 첨부
    func.get_n = get_n
    func.set_n = set_n
    return func
```

사용 예:

```python
>>> f = sample()
>>> f()
n= 0
>>> f.set_n(10)
>>> f()
n= 10
>>> f.get_n()
10
```

### Discussion

이 레시피를 작동시키는 두 가지 주요 기능이 있다. 첫째, `nonlocal` 선언은 내부 변수를 변경하는 함수를 작성할 수 있게 한다. 둘째, 함수 속성은 accessor 메서드가 간단한 방식으로 클로저 함수에
첨부되어 인스턴스 메서드처럼 작동할 수 있게 한다(클래스가 관여하지 않더라도).

이 레시피에 대한 약간의 확장은 클로저가 클래스의 인스턴스를 에뮬레이트하도록 만들 수 있다. 내부 함수를 인스턴스의 딕셔너리에 복사하고 반환하기만 하면 된다.

```python
import sys

class ClosureInstance:
    def __init__(self, locals=None):
        if locals is None:
            locals = sys._getframe(1).f_locals
        
        # callable로 인스턴스 딕셔너리 업데이트
        self.__dict__.update((key, value) for key, value in locals.items()
                             if callable(value))
    
    # 특수 메서드 리디렉션
    def __len__(self):
        return self.__dict__['__len__']()

# 사용 예
def Stack():
    items = []
    
    def push(item):
        items.append(item)
    
    def pop():
        return items.pop()
    
    def __len__():
        return len(items)
    
    return ClosureInstance()
```

실제로 작동하는 대화형 세션:

```python
>>> s = Stack()
>>> s
<__main__.ClosureInstance object at 0x10069ed10>
>>> s.push(10)
>>> s.push(20)
>>> s.push('Hello')
>>> len(s)
3
>>> s.pop()
'Hello'
>>> s.pop()
20
>>> s.pop()
10
```

흥미롭게도 이 코드는 일반 클래스 정의를 사용하는 것보다 약간 빠르게 실행된다. 예를 들어, 다음과 같은 클래스에 대해 성능을 테스트하고 싶을 수 있다.

```python
class Stack2:
    def __init__(self):
        self.items = []
    
    def push(self, item):
        self.items.append(item)
    
    def pop(self):
        return self.items.pop()
    
    def __len__(self):
        return len(self.items)
```

테스트하면 다음과 유사한 결과를 얻는다.

```python
>>> from timeit import timeit
>>> # 클로저를 포함하는 테스트
>>> s = Stack()
>>> timeit('s.push(1);s.pop()', 'from __main__ import s')
0.9874754269840196
>>> # 클래스를 포함하는 테스트
>>> s = Stack2()
>>> timeit('s.push(1);s.pop()', 'from __main__ import s')
1.0707052160287276
```

보다시피 클로저 버전은 약 8% 더 빠르게 실행된다. 대부분은 인스턴스 변수에 대한 간소화된 접근에서 나온다. 클로저는 추가 self 변수가 관여하지 않기 때문에 더 빠르다.

Raymond Hettinger는 이 아이디어의 훨씬 더 사악한 변형을 고안했다. 그러나 코드에서 이와 같은 것을 하려는 경향이 있다면, 이것은 여전히 실제 클래스에 대한 다소 이상한 대체물이라는 것을 알아야 한다.
예를 들어, 상속, 프로퍼티, 디스크립터, 클래스 메서드와 같은 주요 기능은 작동하지 않는다. 특수 메서드가 작동하도록 하려면 몇 가지 트릭을 사용해야 한다(예: `ClosureInstance`의
`__len__()` 구현 참조).

마지막으로, 코드를 읽고 왜 일반 클래스 정의처럼 보이지 않는지 궁금해하는 사람들을 혼란스럽게 할 위험이 있다(물론 그들은 왜 더 빠른지도 궁금해할 것이다). 그럼에도 불구하고, 클로저의 내부에 접근을 제공함으로써
무엇을 할 수 있는지에 대한 흥미로운 예다.

큰 그림에서, 클로저에 메서드를 추가하는 것은 내부 상태를 재설정하거나, 버퍼를 플러시하거나, 캐시를 지우거나, 어떤 종류의 피드백 메커니즘을 갖고 싶은 설정에서 더 많은 유용성을 가질 수 있다.